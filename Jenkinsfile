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
            sh 'whoami; id; hostname; date'
        }
        
        stage('pwd') {
            sh 'pwd'
        }
    }
}