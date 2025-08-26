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
docker jenkins
```

## 4. Get Jenkins admin password:
```bash
docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword
```

## 5. Access Jenkins in browser:
'http://localhost:8080'

