# STEPS

## 1. Start Minikube cluster with Docker driver:
```bash
minikube start --driver=docker
```

## 2. Run Jenkins in a Docker container with host network (if doesn't already exist):
```bash
docker run -d --name jenkins --network=host -v jenkins_home:/var/jenkins_home -v /var/run/docker.sock:/var/run/docker.sock -p 8080:8080 -p 50000:50000 jenkins/jenkins:lts
```

## 3. Start Jenkins in Docker container (if already exists):
```bash
docker start jenkins
```

## 4. Install the Docker CLI inside the Jenkins container permanently (if not already installed):
```bash
docker exec -u root jenkins /bin/bash -c "apt-get update && apt-get install -y apt-transport-https ca-certificates curl gnupg lsb-release"
docker exec -u root jenkins /bin/bash -c "curl -fsSL https://download.docker.com/linux/debian/gpg | gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg"
docker exec -u root jenkins /bin/bash -c "echo 'deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/debian $(lsb_release -cs) stable' | tee /etc/apt/sources.list.d/docker.list > /dev/null"
docker exec -u root jenkins /bin/bash -c "apt-get update && apt-get install -y docker-ce-cli"
```

## 5. Update package list and install curl in Jenkins:
```bash
docker exec -u root jenkins /bin/bash -c "apt-get update && apt-get install -y curl"
```

## 6. Install kubectl in Jenkins container permanently (if not already installed):
```bash
docker exec -u root jenkins /bin/bash -c "curl -LO https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl && chmod +x kubectl && mv kubectl /usr/local/bin/"
```

## 7. Install Helm in Jenkins container permanently (if not already installed):
```bash
docker exec -u root jenkins /bin/bash -c "curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash"
```

## 8. Verify installations:
```bash
docker exec jenkins docker --version
```
```bash
docker exec jenkins kubectl version --client
```
```bash
docker exec jenkins helm version
```

## 9. Find out the User ID (UID) and Group ID (GID) of the Jenkins user in the container:
```bash
docker exec jenkins id -u jenkins
```
```bash
docker exec jenkins id -g jenkins
```
- The likeliest output for both commands is:
```bash
1000
```

## 10. Change the ownership of the /var/run/docker.sock to match the UID and GID of the Jenkins user in the container:
```bash
sudo chown 1000:1000 /var/run/docker.sock
```

## 11. Open the Docker Daemon Configuration File (or create it if doesn't exist):
```bash
sudo nano /etc/docker/daemon.json
```

## 12. Disable IPv6 for the Docker daemon to avoid network connectivity problem:
- Paste the following content into the file:
```bash
{
  "ipv6": false
}
```

## 13. Restart Docker, Minikube and Jenkins:
```bash
sudo systemctl restart docker
minikube start --driver=docker
docker start jenkins
```

## 14. Get Jenkins admin password:
```bash
docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword
```

## 15. Access Jenkins in browser:
'http://localhost:8080'

## 16. Check logs:
```bash
docker logs jenkins
```

## 17. Create the README.txt and dockerhub.txt files:
```bash
echo "GitHub Repository: https://github.com/SamiraJouini/jenkins_devops_exam" > README.txt
```
```bash
echo "DockerHub: https://hub.docker.com/u/jouinis" > dockerhub.txt
```