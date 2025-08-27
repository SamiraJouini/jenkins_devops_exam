pipeline {
    agent any
    environment {
        // These will be automatically populated from Jenkins credentials
        DOCKER_USER = credentials('dockerhub-creds').username
        DOCKER_PASS = credentials('dockerhub-creds').password
        GITHUB_USER = credentials('github-creds').username
        GITHUB_TOKEN = credentials('github-creds').password
    }
    stages {
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
                    
                    // Create DockerHub secret
                    sh """
                    kubectl create secret docker-registry regcred \
                        --docker-server=https://index.docker.io/v1/ \
                        --docker-username=${DOCKER_USER} \
                        --docker-password=${DOCKER_PASS} \
                        --dry-run=client -o yaml | kubectl apply -n dev -f -
                    """
                }
            }
        }
        stage('Build & Push Cast Service') {
            steps {
                sh """
                docker build -t ${DOCKER_USER}/cast-service:test ./cast-service
                echo ${DOCKER_PASS} | docker login -u ${DOCKER_USER} --password-stdin
                docker push ${DOCKER_USER}/cast-service:test
                """
            }
        }
        stage('Build & Push Movie Service') {
            steps {
                sh """
                docker build -t ${DOCKER_USER}/movie-service:test ./movie-service
                docker push ${DOCKER_USER}/movie-service:test
                """
            }
        }
        stage('Test Deployment') {
            steps {
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
    post {
        always {
            sh 'rm -f ${HOME}/.docker/config.json'
            sh 'kubectl get pods -n dev'
        }
    }
}