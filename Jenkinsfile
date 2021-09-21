pipeline {
  agent any
  stages {
    stage('build') {
      steps {
        echo 'building the application'
        sh 'go version'
      }
    }

  }
  tools {
    go 'Go'
  }
  environment {
    GO111MODULE = 'on'
  }
}