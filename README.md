# 🚀 Django Notes App — CI/CD Pipeline with Jenkins & Docker

A real-world CI/CD pipeline for a Django Notes App using Jenkins, Docker, and GitHub.
Automatically deploys to **Dev** and **Staging** environments on a single EC2 instance.

---

## 🏢 Project Overview

| Tool | Purpose |
|------|---------|
| EC2 (Ubuntu 22.04) | Jenkins Host & Docker Runtime |
| GitHub | Source Code |
| Jenkins | CI/CD Pipeline Orchestrator |
| Docker | Containerization |
| Docker Hub | Image Registry |
| Jenkins Secrets | Secure Credential Management |

---

## 📦 Port Mapping

| Environment | Container Port | Host Port |
|-------------|---------------|-----------|
| Dev | 8000 | 8001 |
| Staging | 8000 | 8002 |

---

## 🛠 Step-by-Step Setup

### ✅ Step 1: GitHub Repo Clone karo

```bash
git clone https://github.com/nasirbloch323/django-notes-app-Docker.git
cd django-notes-app-Docker
```

---

### ✅ Step 2: EC2 Instance Launch karo

- OS: Ubuntu 22.04
- Instance Type: t2.micro
- Inbound Ports Open karo:

| Port | Purpose |
|------|---------|
| 22 | SSH |
| 8080 | Jenkins |
| 8001 | Dev Environment |
| 8002 | Staging Environment |

SSH karo:
```bash
ssh -i your-key.pem ubuntu@<EC2_PUBLIC_IP>
```

---

### ✅ Step 3: Jenkins Install karo

```bash
sudo apt update
sudo apt install -y openjdk-17-jdk
wget -q -O - https://pkg.jenkins.io/debian-stable/jenkins.io.key | \
sudo tee /usr/share/keyrings/jenkins-keyring.asc
echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
https://pkg.jenkins.io/debian-stable binary/ | \
sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt update
sudo apt install -y jenkins
sudo systemctl enable jenkins
sudo systemctl start jenkins
```

---

### ✅ Step 4: Docker Install karo

```bash
sudo apt install -y docker.io

# Jenkins ko Docker permission do
sudo usermod -aG docker jenkins
sudo chmod 666 /var/run/docker.sock
sudo systemctl restart jenkins

sudo rm -f /usr/share/keyrings/jenkins-keyring.* /etc/apt/sources.list.d/jenkins.list && sudo mkdir -p /etc/apt/keyrings && curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io-2026.key | sudo tee /etc/apt/keyrings/jenkins-keyring.asc > /dev/null && echo "deb [signed-by=/etc/apt/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/" | sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null && sudo apt update




```

Verify karo:
```bash
groups jenkins
# Output mein "docker" hona chahiye
```

---

### ✅ Step 5: Jenkins Setup karo

Browser mein kholo:
```
http://<EC2_PUBLIC_IP>:8080
```

Initial password lo:
```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

Plugins install karo:
- Docker Pipeline
- GitHub Integration
- Credentials Binding

---

### ✅ Step 6: Docker Hub Access Token banao

1. Jao: https://hub.docker.com/settings/security
2. **New Access Token** click karo
3. Name: `jenkins-token`
4. Permissions: **Read & Write** ✅
5. Token copy karo — ek baar hi dikhega!

---

### ✅ Step 7: Jenkins mein Docker Hub Credentials add karo

```
http://<EC2_PUBLIC_IP>:8080/manage/credentials/store/system/domain/_/newCredentials
```

| Field | Value |
|-------|-------|
| Kind | Username with password |
| Scope | **Global** ✅ |
| Username | `nasirbloch323` |
| Password | `docker hub access token` |
| ID | `dockerhub` |
| Description | Docker Hub Creds |

> ⚠️ Scope **Global** hona zaroori hai — System scope se error aata hai!

---

### ✅ Step 8: GitHub Webhook Setup karo

GitHub pe jao:
```
https://github.com/nasirbloch323/django-notes-app-Docker/settings/hooks
```

Add Webhook:

| Field | Value |
|-------|-------|
| Payload URL | `http://<EC2_IP>:8080/github-webhook/` |
| Content Type | `application/json` |
| Events | Just the push event |

Jenkins mein:
```
Job → Configure → Build Triggers
→ ✅ "GitHub hook trigger for GITScm polling" check karo
→ Save
```

---

### ✅ Step 9: Jenkins Pipeline Job banao

```
Jenkins Dashboard → New Item → "django_notes_app" → Pipeline
```

Jenkinsfile paste karo:

```groovy
pipeline {
    agent any

    environment {
        IMAGE_NAME = 'nasirbloch323/django_notes_app'
        DOCKER_CREDENTIALS_ID = 'dockerhub'
    }

    stages {
        stage('Clone Repo') {
            steps {
                git branch: 'main', url: 'https://github.com/nasirbloch323/django-notes-app-Docker.git'
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    env.IMAGE_TAG = "${IMAGE_NAME}:${env.BUILD_NUMBER}"
                    sh "docker build -t ${env.IMAGE_TAG} ."
                }
            }
        }

        stage('Login to Docker Hub') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: "${DOCKER_CREDENTIALS_ID}",
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh "echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin"
                }
            }
        }

        stage('Push Image to Docker Hub') {
            steps {
                sh "docker push ${env.IMAGE_TAG}"
                sh "docker tag ${env.IMAGE_TAG} ${IMAGE_NAME}:latest"
                sh "docker push ${IMAGE_NAME}:latest"
            }
        }

        stage('Deploy Dev and Staging') {
            steps {
                script {
                    def containers = ['dev': 8001, 'staging': 8002]
                    containers.each { envName, port ->
                        def containerName = "django-notes-${envName}"
                        sh """
                            docker rm -f ${containerName} || true
                            docker run -d --name ${containerName} \
                            -p ${port}:8000 ${env.IMAGE_TAG}
                        """
                    }
                }
            }
        }
    }

    post {
        always {
            sh "docker logout"
            sh "docker image prune -f"
        }
        success {
            echo "✅ Dev: http://<EC2_IP>:8001 | Staging: http://<EC2_IP>:8002"
        }
        failure {
            echo "❌ Pipeline failed!"
        }
    }
}
```

---

### ✅ Step 10: Deployment Trigger karo

```bash
git checkout main
echo "### Trigger Jenkins deployment" >> README.md
git add .
git commit -m "Trigger Jenkins"
git push origin main
```

---

### ✅ Step 11: Containers Verify karo

```bash
docker ps
```

Expected Output:
```
CONTAINER ID   IMAGE                              PORTS                    NAMES
xxx            nasirbloch323/django_notes_app:8   0.0.0.0:8001->8000/tcp   django-notes-dev
yyy            nasirbloch323/django_notes_app:8   0.0.0.0:8002->8000/tcp   django-notes-staging
```

---

### ✅ Step 12: Browser mein Access karo

```
Dev:      http://54.211.121.19:8001
Staging:  http://54.211.121.19:8002
```

---

## ⚠️ Common Errors & Fixes

| Error | Fix |
|-------|-----|
| `permission denied /var/run/docker.sock` | `sudo usermod -aG docker jenkins && sudo chmod 666 /var/run/docker.sock && sudo systemctl restart jenkins` |
| `authentication required - insufficient scopes` | Docker Hub pe Access Token banao Read & Write permissions ke sath |
| `Could not find credentials entry with ID 'dockerhub'` | Jenkins credentials mein Scope **Global** rakho, System nahi |

---

## 🎯 Pipeline Flow

```
GitHub Push
    ↓
Jenkins Webhook Trigger
    ↓
Clone Repo
    ↓
Build Docker Image
    ↓
Push to Docker Hub
    ↓
Deploy Dev  (port 8001)
Deploy Staging (port 8002)
```

---

## 👨‍💻 Author

**Nasir Bloch**
GitHub: [@nasirbloch323](https://github.com/nasirbloch323)
Docker Hub: [nasirbloch323](https://hub.docker.com/u/nasirbloch323)
