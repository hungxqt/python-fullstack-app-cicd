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

        stage('Get Short Commit Hash') {
            steps {
                script {
                    env.SHORT_COMMIT_HASH = sh(script: "git rev-parse --short HEAD", returnStdout: true).trim()
                    echo "Short Commit Hash: ${env.SHORT_COMMIT_HASH}"
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    withCredentials([ usernamePassword(credentialsId: 'fastapi-dockerhub-login', usernameVariable: 'DOCKERHUB_USERNAME', passwordVariable: 'DOCKERHUB_PASSWORD') ]) {
                        sh '''

                            echo "${DOCKERHUB_PASSWORD}" | docker login -u "$DOCKERHUB_USERNAME" --password-stdin

                            export TAG=${env.SHORT_COMMIT_HASH}

                            docker compose -f docker-compose-build.yml build frontend
                            docker compose -f docker-compose-build.yml push frontend

                            docker compose -f docker-compose-build.yml build backend
                            docker compose -f docker-compose-build.yml push backend

                            rm -rf /tmp/dockercfg
                        '''
                    }
                }
            }
        }

        stage('First build check') {
            steps {
                script {
                    withCredentials([file(credentialsId: 'vmware-kubeconfig', variable: 'KUBECONFIG_FILE')]) {

                        def exists = sh(
                            script: "export KUBECONFIG=${KUBECONFIG_FILE} && kubectl -n fastapi-backend get deployment fastapi-backend >/dev/null 2>&1",
                            returnStatus: true
                        )

                        if (exists != 0) {
                            sh """
                                set -e
                                export KUBECONFIG=${KUBECONFIG_FILE}
                                echo "Deployment not found. Running initial apply..."
                                kubectl apply -f k8s/fastapi-backend/fastapi-backend.yaml
                                kubectl apply -f k8s/fastapi-frontend/fastapi-frontend.yaml
                            """
                        } else {
                            echo "Deployment already exists. Skipping apply."
                        }
                    }
                }
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