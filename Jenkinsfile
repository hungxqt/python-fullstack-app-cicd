pipeline {
    agent {
        label 'hungqt-server'
    }

    stages {
        stage('info') {
            sh 'whoami; id; hostname; date'
        }
        stage('pwd') {
            sh 'pwd'
        }
    }
}