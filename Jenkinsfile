pipeline {
	agent any
	tools {
        go 'Go'
    }
    environment {
        GO111MODULE = 'on'
    }
	stages {
		stage("build") {
			steps {
			echo "building the application"
			sh 'go version'
}
}
}
}
