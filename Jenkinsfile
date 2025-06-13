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
                }
            }
        }
       //  stage('Start Juice Shop'){
         //   steps{
           //     sh '''
             //       docker run --name juice-shop -d -p 3000:3000 bkimminich/juice-shop:latest
               // '''
          //  }
       // }
        stage('OSV-Scanner Package-lock.json scan') {
            steps {
                script{
                    sh 'osv-scanner --lockfile package-lock.json --format json --output=${WORKSPACE}/results/osv-report.json || true'
                }
            }
        }
        stage('Example') {
            steps {
                echo 'Hello!'
                sh 'ls -la'
            }
        }
    }
}
