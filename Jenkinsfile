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
                    env.SHORT_COMMIT_HASH = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
                    echo "Short Commit Hash: ${env.SHORT_COMMIT_HASH}"
                }
            }
        }

        stage('Docker Login') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'fastapi-dockerhub-login', usernameVariable: 'DOCKERHUB_USERNAME', passwordVariable: 'DOCKERHUB_PASSWORD')]) {
                        sh(script: 'echo "$DOCKERHUB_PASSWORD" | docker login -u "$DOCKERHUB_USERNAME" --password-stdin')
                    }
                }
            }
        }

        stage('Build and Push Docker Images') {
            parallel {
                stage('Build Docker Image and Push to Registry - Backend') {
                    steps {
                        script {
                            withCredentials([ string(credentialsId: 'DOCKER_REGISTRY_FASTAPI_PROJECT', variable: 'DOCKER_REGISTRY'), 
                                            string(credentialsId: 'DOCKER_IMAGE_BACKEND_FASTAPI_PROJECT', variable: 'DOCKER_IMAGE_BACKEND') ]) {
                                sh '''
                                    set -e

                                    export TAG=${SHORT_COMMIT_HASH}

                                    docker compose -f docker-compose-build.yml build backend

                                    REPO="${DOCKER_REGISTRY}/${DOCKER_IMAGE_BACKEND}"
                                    IMAGE="${REPO}:${TAG}"

                                    new_id=$(docker images --quiet "$IMAGE")

                                    if [ -z "$new_id" ]; then
                                        echo "ERROR: No found built image: $IMAGE"
                                        exit 0
                                    fi

                                    docker images --format '{{.Repository}}:{{.Tag}} {{.ID}}

                                    dup_count=$(docker images --format '{{.Repository}}:{{.Tag}} {{.ID}}' | awk -v repo="$REPO" -v id="$new_id" '$1 ~ "^"repo":" && $2 == id { print $1 }' | wc -l)

                                    if [ "$dup_count" -gt 1 ]; then
                                        echo "Found $dup_count tags in $REPO with the same image ID $new_id."
                                        echo "=> Please check the Dockerfile and rebuild the image to ensure unique image IDs for each build."

                                        docker rmi "$IMAGE" || true
                                        exit 0
                                    fi

                                    docker compose -f docker-compose-build.yml push backend

                                    touch BACKEND_IMAGE_BUILDED
                                '''
                            }
                        }
                    }
                }

                stage('Build Docker Image and Push to Registry - Frontend') {
                    steps {
                        script {
                            withCredentials([ string(credentialsId: 'DOCKER_REGISTRY_FASTAPI_PROJECT', variable: 'DOCKER_REGISTRY'),
                                            string(credentialsId: 'DOCKER_IMAGE_FRONTEND_FASTAPI_PROJECT', variable: 'DOCKER_IMAGE_FRONTEND') ]) {
                                sh '''
                                    set -e

                                    export TAG=${SHORT_COMMIT_HASH}

                                    docker compose -f docker-compose-build.yml build frontend

                                    REPO="${DOCKER_REGISTRY}/${DOCKER_IMAGE_FRONTEND}"
                                    IMAGE="${REPO}:${TAG}"

                                    new_id=$(docker images --quiet "$IMAGE")

                                    if [ -z "$new_id" ]; then
                                        echo "ERROR: No found built image: $IMAGE"
                                        exit 0
                                    fi

                                    docker images --format '{{.Repository}}:{{.Tag}} {{.ID}}

                                    dup_count=$(docker images --format '{{.Repository}}:{{.Tag}} {{.ID}}' | awk -v repo="$REPO" -v id="$new_id" '$1 ~ "^"repo":" && $2 == id { print $1 }' | wc -l)

                                    if [ "$dup_count" -gt 1 ]; then
                                        echo "Found $dup_count tags in $REPO with the same image ID $new_id."
                                        echo "=> Please check the Dockerfile and rebuild the image to ensure unique image IDs for each build."

                                        docker rmi "$IMAGE" || true
                                        exit 0
                                    fi

                                    docker compose -f docker-compose-build.yml push frontend

                                    touch FRONTEND_IMAGE_BUILDED
                                '''
                            }
                        }
                    }
                }
            }
        }

        stage('First build check') {
            steps {
                script {
                    withCredentials([file(credentialsId: 'vmware-kubeconfig', variable: 'KUBECONFIG_FILE')]) {

                        def exists = sh(
                            script: 'export KUBECONFIG=$KUBECONFIG_FILE && kubectl -n fastapi-backend get deployment fastapi-backend-deployment >/dev/null 2>&1',
                            returnStatus: true
                        )

                        if (exists != 0) {
                            sh """
                                set -e
                                export KUBECONFIG=${KUBECONFIG_FILE}
                                echo 'Deployment not found. Running initial apply...'
                                kubectl apply -f k8s/fastapi-backend/fastapi-backend.yaml
                                kubectl apply -f k8s/fastapi-frontend/fastapi-frontend.yaml
                            """
                        } else {
                            echo 'Deployment already exists. Skipping apply.'
                        }
                    }
                }
            }
        }

        stage('Reapply the new image') {
            steps {
                script {
                    withCredentials([file(credentialsId: 'vmware-kubeconfig', variable: 'KUBECONFIG_FILE'), 
                                    string(credentialsId: 'DOCKER_REGISTRY_FASTAPI_PROJECT', variable: 'DOCKER_REGISTRY'), 
                                    string(credentialsId: 'DOCKER_IMAGE_BACKEND_FASTAPI_PROJECT', variable: 'DOCKER_IMAGE_BACKEND'), 
                                    string(credentialsId: 'DOCKER_IMAGE_FRONTEND_FASTAPI_PROJECT', variable: 'DOCKER_IMAGE_FRONTEND') 
                                    ]) {

                        sh '''
                            set -e

                            export KUBECONFIG="$KUBECONFIG_FILE"
                            export TAG="$SHORT_COMMIT_HASH"

                            if [ -f BACKEND_IMAGE_BUILDED ]; then
                                kubectl -n fastapi-backend set image deployment/fastapi-backend-deployment fastapi-backend=${DOCKER_REGISTRY}/${DOCKER_IMAGE_BACKEND}:${TAG}
                            fi

                            if [ -f FRONTEND_IMAGE_BUILDED ]; then
                                kubectl -n fastapi-frontend set image deployment/fastapi-frontend-deployment fastapi-frontend=${DOCKER_REGISTRY}/${DOCKER_IMAGE_FRONTEND}:${TAG}
                            fi
                        '''
                    }
                }
            }
        }

        stage('Cleanup Docker Images') {
            steps {
                sh '''
                    set -e

                    docker image prune -f
                '''
            }
        }
    }

    post {
        always {
            sh 'docker logout || true'
        }
        failure {
            echo 'Pipeline failed! Check logs above for details.'
        }
        success {
            echo 'Pipeline completed successfully!'
        }
    }
}