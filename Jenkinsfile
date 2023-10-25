pipeline {
    agent {
        kubernetes {
            defaultContainer 'jdk'
            yaml '''
apiVersion: v1
kind: Pod
spec:
  containers:
    - name: jdk
      image: docker.io/eclipse-temurin:20.0.1_9-jdk
      command:
        - cat
      tty: true
'''
        }
    }

    stages {
        stage('Check environment') {
            steps {
                sh 'java -version'
            }
        }
    }
}
