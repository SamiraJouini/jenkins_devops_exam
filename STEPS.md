# STEPS

## 1. Start Minikube cluster with Docker driver:
```bash
minikube start --driver=docker
```

## 2. Create a new Docker container with Jenkins (if doesn't exist):
```bash
docker run -d -p 8080:8080 -p 50000:50000 \
  -v jenkins_home:/var/jenkins_home \
  -v /var/run/docker.sock:/var/run/docker.sock \
  --name jenkins \
  jenkins/jenkins:lts
```

## 3. Start Jenkins in Docker container (if already exists):
```bash
docker start jenkins
```

## 4. Install kubectl in Jenkins container:
```bash
docker exec -u root jenkins /bin/bash -c "curl -LO https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl && chmod +x kubectl && mv kubectl /usr/local/bin/"
```

## 5. Install Helm in Jenkins container:
```bash
docker exec -u root jenkins /bin/bash -c "curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash"
```

## 7. Get Jenkins admin password:
```bash
docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword
```

## 8. Access Jenkins in browser:
'http://localhost:8080'

## 9. Check logs:
```bash
docker logs jenkins
```