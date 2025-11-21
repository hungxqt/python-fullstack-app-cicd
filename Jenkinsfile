pipeline {
    agent {
        label 'hungqt-server'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('info') {
            steps {
                sh 'whoami; id; hostname; date'
            }
        }

        stage('pwd') {
            steps {
                sh 'pwd; ls -al'
            }
        }
    }
}