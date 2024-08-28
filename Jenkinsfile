pipeline {
  agent any

  stages {
    stage('SCM') {
      steps {
        checkout scm
      }
    }
    stage('Initialize') {
      steps {
        dir('src') {
          sh 'npm install'
        }
      }
    }
    stage('SonarQube Analysis') {
      steps {
        script {
          def scannerHome = tool 'SonarScanner'
          withSonarQubeEnv('SonarQubeServer') {
            sh "${scannerHome}/bin/sonar-scanner"
          }
        }
      }
    }
  }
}
