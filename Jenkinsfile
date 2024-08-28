i want to build on it egent call sonar agent useing this pipline how to do it?

node("sonar") {
  stage('SCM') {
    checkout scm
  }
 stage('Initialize'){
     dir('src')
     {
         sh "npm install"
     }
 }
  stage('SonarQube Analysis') {
    def scannerHome = tool 'SonarScanner';
    withSonarQubeEnv() {
      sh "${scannerHome}/bin/sonar-scanner"
    }
  }
}
