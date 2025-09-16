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
                        sh """
                            cp '${KUBECONFIG_FILE}' '${env.KUBECONFIG}'
                            chmod 600 '${env.KUBECONFIG}'
                            # Test connection to Kubernetes cluster
                            timeout(time: 10, unit: 'SECONDS') {
                                kubectl cluster-info || true
                            }
                        """
                    }
                }
            }
        }
        stage('Setup Kubernetes') {
            steps {
                script {
                    // Create namespaces
                    sh '''
                        kubectl create namespace dev --dry-run=client -o yaml | kubectl apply -f - || true
                        kubectl create namespace qa --dry-run=client -o yaml | kubectl apply -f - || true
                        kubectl create namespace staging --dry-run=client -o yaml | kubectl apply -f - || true
                        kubectl create namespace prod --dry-run=client -o yaml | kubectl apply -f - || true
                    '''
                    
                    // Create DockerHub secret in all namespaces
                    withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                        sh """
                            kubectl create secret docker-registry regcred \
                                --docker-server=https://index.docker.io/v1/ \
                                --docker-username=${DOCKER_USER} \
                                --docker-password=${DOCKER_PASS} \
                                --dry-run=client -o yaml | kubectl apply -n dev -f - || true
                        """
                    }
                }
            }
        }
        stage('Build & Push Cast Service') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    sh """
                        docker build -t ${DOCKER_USER}/cast-service:test ./cast-service || true
                        echo ${DOCKER_PASS} | docker login -u ${DOCKER_USER} --password-stdin || true
                        docker push ${DOCKER_USER}/cast-service:test || true
                    """
                }
            }
        }
        stage('Build & Push Movie Service') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    sh """
                        docker build -t ${DOCKER_USER}/movie-service:test ./movie-service || true
                        docker push ${DOCKER_USER}/movie-service:test || true
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
                            --set cast_service.image.tag=test || true
                    """
                }
            }
        }
    }
    post {
        always {
            sh 'kubectl get pods -n dev || true'
            sh 'rm -f ${HOME}/.docker/config.json || true'
            sh 'rm -f ${KUBECONFIG} || true'
        }
    }
}