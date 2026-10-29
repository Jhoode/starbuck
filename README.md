![Starbucks Clone Deployment](https://github.com/user-attachments/assets/6b654f47-9537-4b88-9584-41c760fc49ac)

# Deploy Starbucks Clone Application AWS using DevSecOps Approach
https://app.eraser.io/workspace/59NJfCay26dUMl5YAlFl?origin=share

> **Demo setup:** see [`DEMO-DEPLOY.md`](DEMO-DEPLOY.md) for the full step-by-step runbook (EC2 sizing, ports, Jenkins plugins/tools/credentials, SonarQube webhook, demo tips).

# **Quick Start (local Docker)**
```
docker build -t starbucks .
docker run -d --name starbucks -p 3000:3000 starbucks
```
Open http://localhost:3000

The `Dockerfile` is multi-stage: it builds the React app and serves it with nginx on port 3000 (`nginx.conf` handles React Router paths like `/dashboard`). The original dev-server Dockerfile is kept as `Dockerfile.dev`.

# **Install AWS CLI**
```
sudo apt install unzip -y
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
```
 
# **Install Jenkins on Ubuntu:**

```
#!/bin/bash
sudo apt update -y
wget -O - https://packages.adoptium.net/artifactory/api/gpg/key/public | sudo tee /etc/apt/keyrings/adoptium.asc
echo "deb [signed-by=/etc/apt/keyrings/adoptium.asc] https://packages.adoptium.net/artifactory/deb $(awk -F= '/^VERSION_CODENAME/{print$2}' /etc/os-release) main" | sudo tee /etc/apt/sources.list.d/adoptium.list
sudo apt update -y
sudo apt install temurin-21-jdk -y
/usr/bin/java --version
curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io-2026.key | sudo tee /usr/share/keyrings/jenkins-keyring.asc > /dev/null
echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/ | sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt-get update -y
sudo apt-get install jenkins -y
sudo systemctl start jenkins
sudo systemctl status jenkins
```


# **Install Docker on Ubuntu:**
```
# Add Docker's official GPG key:
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
# Add the repository to Apt sources:
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y
sudo usermod -aG docker ubuntu
sudo chmod 777 /var/run/docker.sock
newgrp docker
sudo systemctl status docker
```

# **Install Trivy on Ubuntu:**

Reference Doc: https://aquasecurity.github.io/trivy/v0.55/getting-started/installation/
```
sudo apt-get install wget apt-transport-https gnupg
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | gpg --dearmor | sudo tee /usr/share/keyrings/trivy.gpg > /dev/null
echo "deb [signed-by=/usr/share/keyrings/trivy.gpg] https://aquasecurity.github.io/trivy-repo/deb generic main" | sudo tee -a /etc/apt/sources.list.d/trivy.list
sudo apt-get update
sudo apt-get install trivy
```


# **Install Docker Scout:**
```
docker login       `Give Dockerhub credentials here`
```
```
curl -sSfL https://raw.githubusercontent.com/docker/scout-cli/main/install.sh | sudo sh -s -- -b /usr/local/bin
```
# Deployment Stages:
<img width="966" alt="Screenshot 2024-09-15 at 7 20 49 AM" src="https://github.com/user-attachments/assets/ddb5e618-79ab-49b3-8f13-b5114824eec3">

# **Run SonarQube**
```
docker run -d --name sonar -p 9000:9000 sonarqube:lts-community
```
Open http://<EC2-IP>:9000 (admin/admin) and generate a token. Then add a webhook to `http://<EC2-IP>:8080/sonarqube-webhook/` so the Quality Gate stage works.

# **Allow Jenkins to use Docker**
```
sudo usermod -aG docker jenkins
sudo systemctl restart jenkins
```

# Jenkins Complete pipeline
This is the same pipeline as the [`Jenkinsfile`](Jenkinsfile) in this repo. Before running it, set `GIT_REPO` and `IMAGE` to your own GitHub repo and Docker Hub image.

Jenkins names that must match: tools `jdk17`, `node16`, `sonar-scanner`, `DP-Check`, `docker`; credentials `Sonar-token`, `docker`; SonarQube server `sonar-server`.

```groovy
pipeline {
    agent any
    tools {
        jdk 'jdk17'
        nodejs 'node16'
    }
    environment {
        SCANNER_HOME = tool 'sonar-scanner'
        // >>> CHANGE THESE TWO <<<
        GIT_REPO   = 'https://github.com/<your-github-user>/starbucks.git'
        IMAGE      = '<your-dockerhub-user>/starbucks:latest'
    }
    stages {
        stage('Clean workspace') { steps { cleanWs() } }

        stage('Git checkout') {
            steps { git branch: 'main', url: "${GIT_REPO}" }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh '$SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=starbucks -Dsonar.projectKey=starbucks'
                }
            }
        }

        stage('Quality Gate') {
            steps {
                script { waitForQualityGate abortPipeline: false, credentialsId: 'Sonar-token' }
            }
        }

        stage('Install NPM Dependencies') { steps { sh 'npm ci' } }

        stage('OWASP FS Scan') {
            steps {
                dependencyCheck additionalArguments: '--scan ./ --disableYarnAudit --disableNodeAudit', odcInstallation: 'DP-Check'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }

        stage('Trivy File Scan') { steps { sh 'trivy fs . > trivy.txt' } }

        stage('Build Docker Image') { steps { sh 'docker build -t starbucks .' } }

        stage('Tag & Push to DockerHub') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker') {
                        sh "docker tag starbucks ${IMAGE}"
                        sh "docker push ${IMAGE}"
                    }
                }
            }
        }

        stage('Docker Scout Image') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker', toolName: 'docker') {
                        sh "docker-scout quickview ${IMAGE}"
                        sh "docker-scout cves ${IMAGE}"
                        sh "docker-scout recommendations ${IMAGE}"
                    }
                }
            }
        }

        stage('Deploy to Container') {
            steps {
                // remove the previous run so re-deploys don't fail on the name clash
                sh 'docker rm -f starbucks || true'
                sh "docker run -d --name starbucks -p 3000:3000 ${IMAGE}"
            }
        }
    }
    post {
        always {
            emailext attachLog: true,
                subject: "'${currentBuild.result}'",
                body: "Project: ${env.JOB_NAME}<br/>Build: ${env.BUILD_NUMBER}<br/>URL: ${env.BUILD_URL}",
                to: 'ojugo.jude@gmail.com',
                mimeType: 'text/html',
                attachmentsPattern: 'trivy.txt'
        }
    }
}
```

# **Clean up**
Terminate the EC2 instance after the demo so it stops billing.
