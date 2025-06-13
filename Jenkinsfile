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
                }
            }
        }
        stage('Prepare test environment') {
            steps {
                sh 'mkdir reports'
                sh 'chmod 777 reports'
                sh '''
                    if [ $(docker ps -q -f name=juice-shop) ]; then
                        echo "Stopping existing juice-shop container"
                        docker stop juice-shop || true
                    fi
                    '''
                sh '''
                    if [ $(docker ps -a -q -f name=zap) ]; then
                        echo "Stopping and removing existing zap container"
                        docker stop zap || true
                        docker rm zap || true
                    fi
                    '''
            }
        }
        
        stage('Scan package-lock.json file') {
            steps {
                sh 'osv-scanner --format json --output reports/osv_json_report.json --lockfile package-lock.json || true'
            }
        }
        stage('Trufflehog Scan') {
            steps {
                sh 'trufflehog git file://$PWD --branch main --json > reports/trufflehog_json_report.json'
            }
        }
        stage('Semgrep Scan') {
            steps {
                sh 'semgrep scan --config auto --json-output=reports/semgrep_json_report.json'
            }
        }
        stage('ZAP Scan') {
            steps {
                sh '''
                    docker run --name juice-shop -d --rm \
                        -p 3000:3000 \
                        bkimminich/juice-shop
                    sleep 5
                '''
                timeout(time: 3, unit: 'MINUTES') {
                sh '''
                    docker rm -f zap || true
                    docker run --name zap \
                        --add-host=host.docker.internal:host-gateway \
                        -v /home/kali/abcd-student/.zap:/zap/wrk/:rw \
                        -t ghcr.io/zaproxy/zaproxy:stable bash -c \
                        "ls -l /zap/wrk/ && zap.sh -cmd -addonupdate; zap.sh -cmd -addoninstall communityScripts -addoninstall pscanrulesAlpha -addoninstall pscanrulesBeta -autorun /zap/wrk/passive.yaml" \
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
            sh '''
                        docker stop zap juice-shop
                        docker rm zap
                    '''
            archiveArtifacts artifacts: 'reports/**/*.*', fingerprint: true
        }
    }
}
