pipeline {
    agent any
    environment {
        DOCKER_USER = credentials('dockerhub-creds').username
        DOCKER_PASS = credentials('dockerhub-creds').password
        GITHUB_TOKEN = credentials('github-creds').password
        K8S_CONTEXT = "minikube"  // Update to your cluster name
    }
    stages {
        stage('Setup Kubernetes') {
            steps {
                script {
                    // Create namespaces
                    sh """
                    kubectl create namespace dev --dry-run=client -o yaml | kubectl apply -f -
                    kubectl create namespace qa --dry-run=client -o yaml | kubectl apply -f -
                    kubectl create namespace staging --dry-run=client -o yaml | kubectl apply -f -
                    kubectl create namespace prod --dry-run=client -o yaml | kubectl apply -f -
                    """
                    
                    // Create DockerHub secret in all namespaces
                    withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                        sh """
                        kubectl create secret docker-registry regcred \
                            --docker-server=https://index.docker.io/v1/ \
                            --docker-username=${DOCKER_USER} \
                            --docker-password=${DOCKER_PASS} \
                            --dry-run=client -o yaml | kubectl apply -n dev -f -
                        
                        kubectl create secret docker-registry regcred \
                            --docker-server=https://index.docker.io/v1/ \
                            --docker-username=${DOCKER_USER} \
                            --docker-password=${DOCKER_PASS} \
                            --dry-run=client -o yaml | kubectl apply -n qa -f -
                            
                        kubectl create secret docker-registry regcred \
                            --docker-server=https://index.docker.io/v1/ \
                            --docker-username=${DOCKER_USER} \
                            --docker-password=${DOCKER_PASS} \
                            --dry-run=client -o yaml | kubectl apply -n staging -f -
                            
                        kubectl create secret docker-registry regcred \
                            --docker-server=https://index.docker.io/v1/ \
                            --docker-username=${DOCKER_USER} \
                            --docker-password=${DOCKER_PASS} \
                            --dry-run=client -o yaml | kubectl apply -n prod -f -
                        """
                    }
                }
            }
        }
        stage('Build & Push Cast Service') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    sh """
                    docker build -t ${DOCKER_USER}/cast-service:${BUILD_ID} ./cast-service
                    echo ${DOCKER_PASS} | docker login -u ${DOCKER_USER} --password-stdin
                    docker push ${DOCKER_USER}/cast-service:${BUILD_ID}
                    """
                }
            }
        }
        stage('Build & Push Movie Service') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    sh """
                    docker build -t ${DOCKER_USER}/movie-service:${BUILD_ID} ./movie-service
                    docker push ${DOCKER_USER}/movie-service:${BUILD_ID}
                    """
                }
            }
        }
        stage('Deploy to Environment') {
            when {
                anyOf { 
                    branch 'dev'; branch 'qa'; branch 'staging' 
                }
            }
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    sh """
                    helm upgrade --install app-chart ./charts -n ${BRANCH_NAME} \
                        --set movie_service.image.repository=${DOCKER_USER}/movie-service \
                        --set movie_service.image.tag=${BUILD_ID} \
                        --set cast_service.image.repository=${DOCKER_USER}/cast-service \
                        --set cast_service.image.tag=${BUILD_ID}
                    """
                }
            }
        }
        stage('Deploy to Production') {
            when { 
                branch 'master' 
            }
            steps {
                input message: 'Deploy to PROD?', ok: 'Confirm'
                withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    sh """
                    helm upgrade --install app-chart ./charts -n prod \
                        --set movie_service.image.repository=${DOCKER_USER}/movie-service \
                        --set movie_service.image.tag=${BUILD_ID} \
                        --set cast_service.image.repository=${DOCKER_USER}/cast-service \
                        --set cast_service.image.tag=${BUILD_ID}
                    """
                }
            }
        }
    }
    post {
        always {
            // Clean up Docker credentials
            sh 'rm -f ${HOME}/.docker/config.json'
        }
        success {
            // Send notification (optional)
            echo 'Pipeline succeeded!'
        }
        failure {
            // Send notification (optional)
            echo 'Pipeline failed!'
        }
    }
}