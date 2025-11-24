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
        stage('kubectl test') {
            steps {
                withCredentials([ file(credentialsId: 'vmware-kubeconfig', variable: 'KUBECONFIG_FILE') ]) {
                    sh '''
                        export KUBECONFIG=${KUBECONFIG_FILE}
                        kubectl get nodes
                        kubectl get pods --all-namespaces
                    '''
                }
            }
        }
    }
}