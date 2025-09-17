pipeline {
    agent any
    environment {
        DOCKER_USER = ''
        DOCKER_PASS = ''
        KUBECONFIG = "${env.WORKSPACE}/kubeconfig"
    }
    stages {
        stage('Prepare Kubeconfig') {
            steps {
                script {
                    // Copy the kubeconfig file from credentials
                    withCredentials([file(credentialsId: 'kubeconfig-file', variable: 'KUBECONFIG_FILE')]) {
                        sh '''
                            cp "$KUBECONFIG_FILE" "$KUBECONFIG"
                            chmod 600 "$KUBECONFIG"
                            # Test if the kubeconfig is valid
                            kubectl cluster-info --request-timeout=5s
                        '''
                    }
                }
            }
        }
        stage('Setup Kubernetes') {
            steps {
                script {
                    // Create namespaces
                    sh '''
                        kubectl create namespace dev --dry-run=client -o yaml | kubectl apply -f -
                        kubectl create namespace qa --dry-run=client -o yaml | kubectl apply -f -
                        kubectl create namespace staging --dry-run=client -o yaml | kubectl apply -f -
                        kubectl create namespace prod --dry-run=client -o yaml | kubectl apply -f -
                    '''
                    
                    // Create DockerHub secret in all namespaces
                    withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                        sh """
                            kubectl create secret docker-registry regcred \
                                --docker-server=https://index.docker.io/v1/ \
                                --docker-username=${DOCKER_USER} \
                                --docker-password=${DOCKER_PASS} \
                                --dry-run=client -o yaml | kubectl apply -n dev --validate=false -f -
                            kubectl create secret docker-registry regcred \
                                --docker-server=https://index.docker.io/v1/ \
                                --docker-username=${DOCKER_USER} \
                                --docker-password=${DOCKER_PASS} \
                                --dry-run=client -o yaml | kubectl apply -n qa --validate=false -f -
                            kubectl create secret docker-registry regcred \
                                --docker-server=https://index.docker.io/v1/ \
                                --docker-username=${DOCKER_USER} \
                                --docker-password=${DOCKER_PASS} \
                                --dry-run=client -o yaml | kubectl apply -n staging --validate=false -f -
                            kubectl create secret docker-registry regcred \
                                --docker-server=https://index.docker.io/v1/ \
                                --docker-username=${DOCKER_USER} \
                                --docker-password=${DOCKER_PASS} \
                                --dry-run=client -o yaml | kubectl apply -n prod --validate=false -f -
                        """
                    }
                }
            }
        }
        stage('Build & Push Cast Service') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    sh """
                        docker build -t ${DOCKER_USER}/cast-service:test ./cast-service
                        echo ${DOCKER_PASS} | docker login -u ${DOCKER_USER} --password-stdin
                        
                        # Check if image already exists in DockerHub before pushing
                        if docker pull ${DOCKER_USER}/cast-service:test 2>/dev/null; then
                            echo "Image ${DOCKER_USER}/cast-service:test already exists in DockerHub, skipping push"
                        else
                            echo "Pushing ${DOCKER_USER}/cast-service:test to DockerHub"
                            docker push ${DOCKER_USER}/cast-service:test || echo "Push failed, but continuing pipeline"
                        fi
                    """
                }
            }
        }
        stage('Build & Push Movie Service') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    sh """
                        docker build -t ${DOCKER_USER}/movie-service:test ./movie-service
                        
                        # Check if image already exists in DockerHub before pushing
                        if docker pull ${DOCKER_USER}/movie-service:test 2>/dev/null; then
                            echo "Image ${DOCKER_USER}/movie-service:test already exists in DockerHub, skipping push"
                        else
                            echo "Pushing ${DOCKER_USER}/movie-service:test to DockerHub"
                            docker push ${DOCKER_USER}/movie-service:test || echo "Push failed, but continuing pipeline"
                        fi
                    """
                }
            }
        }
        stage('Test Deployment') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    sh """
                        helm upgrade --install test-app ./charts -n dev \
                            --set movie_service.image.repository=${DOCKER_USER}/movie-service \
                            --set movie_service.image.tag=test \
                            --set cast_service.image.repository=${DOCKER_USER}/cast-service \
                            --set cast_service.image.tag=test
                    """
                }
            }
        }
    }
    post {
        always {
            sh 'kubectl get pods -n dev || true'
            sh 'rm -f ${HOME}/.docker/config.json'
            sh 'rm -f ${KUBECONFIG}'
        }
    }
}