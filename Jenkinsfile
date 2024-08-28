pipeline {
    agent { label 'sonar' }

    stages {
        stage('SCM') {
            steps {
                checkout scm
            }
        }

        stage('Install Dependencies') {
            steps {
                 dir('src'){
                sh 'npm install'
                 }
            }
        }

        stage('Run Tests') {
            steps {
                dir('src'){
                sh 'npm test'  // Make sure your tests generate a coverage report
                }
            }
        }

        stage('SonarQube Analysis') {
            steps {
                script {
                    def scannerHome = tool 'SonarScanner';
                    withSonarQubeEnv('SonarQube') { // Replace 'SonarQube' with your SonarQube server name in Jenkins
                        sh "${scannerHome}/bin/sonar-scanner"
                    }
                }
            }
        }
    }

    post {
        always {
            junit '**/test-results/*.xml' // Archive test results if available
        }
    }
}
