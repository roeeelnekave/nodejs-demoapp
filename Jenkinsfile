node('sonar') {
  stage('SCM') {
    checkout scm
  }
 stage('Initialize'){
     dir('src')
     {
         sh "npm install"
     }
 }
  stage('test'){
     dir('src')
     {
         sh "npm run test-report"
         sh "npm run test"
   
     }
 }
    stage('fix'){
     dir('src')
     {
         sh "npm run lint"
        sh "npm run lint-fix"
       
     }
 }
  stage('SonarQube Analysis') {
    def scannerHome = tool 'SonarScanner';
    withSonarQubeEnv() {
      sh "${scannerHome}/bin/sonar-scanner"
    }
  }
}
