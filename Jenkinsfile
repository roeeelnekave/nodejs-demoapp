node('sonar') {
  stage('SCM') {
    checkout scm
  }
  stage('Initialize'){
    dir('src') {
      sh "npm install"
    }
  }
  stage('test'){
    dir('src') {
      sh "npm run test-report"
      sh "npm run test"
      def postmanResult = sh(script: "npm run test-postman", returnStatus: true)
      if (postmanResult != 0) {
        echo "Postman tests failed with status ${postmanResult}. Continuing the build..."
      }
    }
  }
  stage('fix'){
    dir('src') {
      sh "npm run lint"
      sh "npm run lint-fix"
    }
  }
  stage('SonarQube Analysis') {
    def scannerHome = tool 'SonarScanner'
    withSonarQubeEnv() {
      sh "${scannerHome}/bin/sonar-scanner"
    }
  }
    stage('deploy'){
    dir('src') {
      sh "node --expose_gc server.mjs"
      sh "node server.mjs &"
    }
  }
}
