pipeline {
    agent any

    options {
      skipDefaultCheckout(true)
    }

    stages {
        stage('Code checkout from GitHub') {
            steps {
                cleanWs()
                git credentialsId: 'github-pat', url: 'https://github.com/adakoval/abcd-student', branch: 'main'
            }
        }

        stage('[SETUP]') {
            steps {
                sh 'mkdir -p results && chmod -R 777 results'
            }
        }

        stage('Skanowanie równolegle') {
            parallel {
                stage('[OSV-Scanner] Scan') {
                    steps {
                        sh 'osv-scanner --lockfile package-lock.json --format json --output=${WORKSPACE}/results/osv-report.json || true'
                    }
                }
                stage('[TruffleHog] Scan') {
                    steps {
                        sh 'trufflehog git file://. --branch main --json > ${WORKSPACE}/results/trufflehog-report.json || true'
                    }
                }
                stage('[Semgrep] Scan') {
                    steps {
                        sh 'semgrep scan --config auto --json-output=${WORKSPACE}/results/semgrep-report.json || true'
                    }
                }
            }
        }

        stage('ZAP Dynamiczny Skan') {
            steps {
                sh '''
                    docker run --name juice-shop -d --rm \\
                        -p 3000:3000 \\
                        bkimminich/juice-shop
                    sleep 7
                '''
                timeout(time: 3, unit: 'MINUTES') {
                    sh '''
                        docker rm -f zap || true
                        docker run --name zap \\
                            --add-host=host.docker.internal:host-gateway \\
                            -v ${WORKSPACE}/.zap:/zap/wrk/:rw \\
                            -t ghcr.io/zaproxy/zaproxy:stable bash -c \\
                            "zap.sh -cmd -addonupdate && zap.sh -cmd -addoninstall communityScripts pscanrulesAlpha pscanrulesBeta && zap.sh -cmd -autorun /zap/wrk/passive.yaml" \\
                            || true
                    '''
                    sh '''
                        docker cp zap:/zap/wrk/reports/zap_html_report.html ${WORKSPACE}/results/zap_html_report.html
                        docker cp zap:/zap/wrk/reports/zap_xml_report.xml ${WORKSPACE}/results/zap_xml_report.xml
                    '''
                }
            }
            
            post {
                always {
                    //czyszczenie
                    sh '''
                        docker stop zap || true
                        docker rm zap || true
                        docker stop juice-shop || true
                    '''
                    // archiwizacja
                    archiveArtifacts artifacts: 'results/**/*', fingerprint: true, allowEmptyArchive: true
                }
            }
        }
    }
}
