pipeline {
    agent any
    options {
        skipDefaultCheckout(true)
    }
    stages {
        stage('Code checkout from GitHub') {
            steps {
                script {
                    cleanWs()
                    git credentialsId: 'github-pat', url: 'https://github.com/adakoval/abcd-student', branch: 'main'
                    sh 'mkdir results'
                    sh "trufflehog git file://. --json --only-verified > results/trufflehog-output.json"
                }
            }
                post{
                    always{
                        archiveArtifacts artifacts: 'results/trufflehog-output.json', fingerprint: true, allowEmptyArchive: true
                    }
                }
        }
        stage('Semgrep') {
            steps {
                script {
                    sh 'semgrep scan --config=auto --output=results/semgrep.json --json'
                }
            }
                post{
                    always{
                        archiveArtifacts artifacts: 'results/semgrep.json', fingerprint: true, allowEmptyArchive: true
                    }
                }
        }
        
        stage("[omv] SKAN"){
            steps{
                sh 'mkdir results || true'
                sh 'osv-scanner scan --lockfile package-lock.json --format json --output results/sca-osv-scanner.json || true'
            }
            post{
                always{
                    archiveArtifacts artifacts: 'results/**/*', fingerprint: true, allowEmptyArchive: true
                }
            }
        }
        
        stage('[ZAP] Baseline passive-scan') {
            steps {
                sh 'mkdir -p results/'
                sh '''
                    docker run --name juice-shop -d --rm \
                        -p 3000:3000 \
                        bkimminich/juice-shop
                    sleep 5
                '''
                sh '''
                    docker run --name zap \
                        --add-host=host.docker.internal:host-gateway \
                        -v /home/kali/abcd-student/.zap:/zap/wrk/:rw \
                        -t ghcr.io/zaproxy/zaproxy:stable bash -c "\
                        zap.sh -cmd -addonupdate;\
                        zap.sh -cmd -addoninstall communityScripts\
                        -addoninstall pscanrulesAlpha\
                        -addoninstall pscanrulesBeta\
                        -autorun /zap/wrk/passive_scan.yaml" \
                        || true
                '''   
            }
            post {
                always {
                    sh '''
                        docker cp zap:/zap/wrk/reports/zap_html_report.html ${WORKSPACE}/results/zap_html_report.html || true
                        docker cp zap:/zap/wrk/reports/zap_xml_report.xml ${WORKSPACE}/results/zap_xml_report.xml || true
                        docker stop zap juice-shop 
                        docker rm zap 
                    '''
                }
            }
        }
    }
}
