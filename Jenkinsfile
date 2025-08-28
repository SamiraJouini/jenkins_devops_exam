pipeline {
    agent any
    environment {
        DOCKER_USER = ''
        DOCKER_PASS = ''
    }
    stages {
        stage('Setup Kubernetes') {
            steps {
                script {
                    // Use kubeconfig for authentication
                    withKubeConfig([credentialsId: 'kubeconfig-file']) {
                        sh '''
                            kubectl create namespace dev --dry-run=client -o yaml | kubectl apply -f -
                            kubectl create namespace qa --dry-run=client -o yaml | kubectl apply -f -
                            kubectl create namespace staging --dry-run=client -o yaml | kubectl apply -f -
                            kubectl create namespace prod --dry-run=client -o yaml | kubectl apply -f -
                        '''
                    }
                    
                    // Create DockerHub secret in all namespaces
                    withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                        withKubeConfig([credentialsId: 'kubeconfig-file']) {
                            sh """
                                kubectl create secret docker-registry regcred \
                                    --docker-server=https://index.docker.io/v1/ \
                                    --docker-username=${DOCKER_USER} \
                                    --docker-password=${DOCKER_PASS} \
                                    --dry-run=client -o yaml | kubectl apply -n dev -f -
                                // Repeat for other namespaces if needed
                            """
                        }
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
                        docker push ${DOCKER_USER}/cast-service:test
                    """
                }
            }
        }
        stage('Build & Push Movie Service') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    sh """
                        docker build -t ${DOCKER_USER}/movie-service:test ./movie-service
                        docker push ${DOCKER_USER}/movie-service:test
                    """
                }
            }
        }
        stage('Test Deployment') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    withKubeConfig([credentialsId: 'kubeconfig-file']) {
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
    }
    post {
        always {
            withKubeConfig([credentialsId: 'kubeconfig-file']) {
                sh 'kubectl get pods -n dev'
            }
            sh 'rm -f ${HOME}/.docker/config.json'
        }
    }
}