pipeline {
    agent { label 'linux' }
    options {
        buildDiscarder(logRotator(numToKeepStr: '5'))
    }
    environment {
        // Define any environment variables here if needed
        SONARQUBE_INSTALLATION = 'sq1'
    }
    stages {
        stage('Checkout') {
            steps {
                // Clone the GitHub repository
                git url: 'https://github.com/benc-uk/nodejs-demoapp.git'
            }
        }
        stage('Build') {
            steps {
                // Install dependencies and build the application
                sh 'make image'
            }
        }
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv(installationName: "${SONARQUBE_INSTALLATION}") {
                    // Run SonarQube analysis
                    sh './mvnw clean org.sonarsource.scanner.maven:sonar-maven-plugin:3.9.0.2155:sonar'
                }
            }
        }
        stage('Quality Gate') {
            steps {
                timeout(time: 2, unit: 'MINUTES') {
                    // Wait for the SonarQube quality gate
                    waitForQualityGate abortPipeline: true
                }
            }
        }
        stage('Deploy') {
            steps {
                // Deploy the application (example command)
                sh 'make deploy'
            }
        }
    }
    post {
        always {
            // Clean up workspace after the pipeline
            cleanWs()
        }
    }
}
