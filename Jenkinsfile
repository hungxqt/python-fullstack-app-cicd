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

        stage('Build Docker Image') {
            steps {
                script {
                    withCredentials([ usernamePassword(credentialsId: 'fastapi-dockerhub-login', usernameVariable: 'DOCKERHUB_USERNAME', passwordVariable: 'DOCKERHUB_PASSWORD'),
                                    string(credentialsId: 'DOCKER_REGISTRY_FASTAPI_PROJECT', variable: 'DOCKER_REGISTRY'), 
                                    string(credentialsId: 'DOCKER_IMAGE_BACKEND_FASTAPI_PROJECT', variable: 'DOCKER_IMAGE_BACKEND'), 
                                    string(credentialsId: 'DOCKER_IMAGE_FRONTEND_FASTAPI_PROJECT', variable: 'DOCKER_IMAGE_FRONTEND') 
                                    ]) {
                        sh(script: 'echo "$DOCKERHUB_PASSWORD" | docker login -u "$DOCKERHUB_USERNAME" --password-stdin')

                        sh '''
                            set -e

                            export TAG=${SHORT_COMMIT_HASH}

                            docker compose -f docker-compose-build.yml build

                            REPO="${DOCKER_REGISTRY}/${DOCKER_IMAGE_BACKEND}"
                            IMAGE="${REPO}:${TAG}"

                            new_id=$(docker images --no-trunc --quiet "$IMAGE")

                            if [ -z "$new_id" ]; then
                                echo "ERROR: No found built image: $IMAGE"
                                exit 1
                            fi

                            echo "New backend image: $IMAGE"
                            echo "New backend image ID: $new_id"

                            dup_count=$(docker images --format '{{.Repository}}:{{.Tag}} {{.ID}}' \
                            | awk -v repo="$REPO" -v id="$new_id" '
                                $1 ~ "^"repo":" && $2 == id { print $1 }
                                ' \
                            | wc -l)

                            if [ "$dup_count" -gt 1 ]; then
                                echo "Found $dup_count tags in $REPO with the same image ID $new_id."
                                echo "=> Please check the Dockerfile and rebuild the image to ensure unique image IDs for each build."

                                docker rmi "$IMAGE" || true
                                exit 1
                            fi

                            docker compose -f docker-compose-build.yml push
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
                            script: 'export KUBECONFIG=${KUBECONFIG_FILE} && kubectl -n fastapi-backend get deployment fastapi-backend-deployment >/dev/null 2>&1',
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

                            kubectl -n fastapi-backend set image deployment/fastapi-backend-deployment fastapi-backend=${DOCKER_REGISTRY}/${DOCKER_IMAGE_BACKEND}:${TAG}
                            kubectl -n fastapi-frontend set image deployment/fastapi-frontend-deployment fastapi-frontend=${DOCKER_REGISTRY}/${DOCKER_IMAGE_FRONTEND}:${TAG}
                        '''
                    }
                }
            }
        }
    }
}