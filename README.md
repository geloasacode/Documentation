Task Assignment:

Below can be found the steps made to install a working node app accessible via nginx proxy: 
# Node.js CI/CD Pipeline with Jenkins, Docker & Nginx (HTTPS)
---
## Table of Contents

- [Architecture](#architecture)  
- [Prerequisites](#prerequisites)  
- [Setup Steps](#setup-steps)  
  - [1. Install Docker](#1-install-docker)  
  - [2. Install Jenkins](#2-install-jenkins)  
  - [3. Fork & Clone Repo](#3-fork--clone-repo)  
  - [4. Add Dockerfile](#4-add-dockerfile)  
  - [5. Create Jenkins Pipeline](#5-create-jenkins-pipeline)  
  - [6. Install Nginx](#6-install-nginx)  
  - [7. Generate TLS Certificates](#7-generate-tls-certificates)  
  - [8. Configure Nginx Reverse Proxy](#8-configure-nginx-reverse-proxy)  
  - [9. Configure Firewall](#9-configure-firewall)  
- [Access](#access)  
- [Security Notes](#security-notes)  
- [Evaluation Checklist](#evaluation-checklist)  

---

### 1. Install Docker

```bash
#! /bin/bash

set -ex

sudo apt update
sudo apt install -y docker.io
sudo systemctl enable docker
sudo systemctl start docker

sudo usermod -aG docker ubuntu
```
### 2. Install Jenkins - After Installation, it can be accessed through http://172.188.82.198:8080. Password can be found on /var/lib/jenkins/secrets/initialAdminPassword

```bash
#!/bin/bash

set -e

sudo apt install -y openjdk-17-jdk wget gnupg2

sudo mkdir -p /etc/apt/keyrings
sudo wget -O /etc/apt/keyrings/jenkins-keyring.asc https://pkg.jenkins.io/debian-stable/jenkins.io-2026.key

echo "deb [signed-by=/etc/apt/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/" | sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null

sudo apt update
sudo apt install -y jenkins

sudo systemctl enable jenkins.service
sudo systemctl start jenkins.service
sudo usermod -aG docker jenkins
```

### 3. Fork & Clone Repo
I forked the repository https://github.com/heroku/node-js-sample
```bash
git clone https://github.com/geloasacode/node-js-sample.git
cd node-js-sample
```
### 4. Add Dockerfile 

```dockerfile
FROM node:18-alpine
WORKDIR /app
COPY package*.json ./
RUN npm install --production
COPY . .
ENV PORT=3000
EXPOSE 3000
CMD ["npm", "start"]
```

### Then I pushed the changes to my repository

```bash
git add Dockerfile
git commit -m "Add Dockerfile"
git push
```

### 5. Create Jenkins Pipeline - This will build the app image and will run the container on the VM

```jenkinsfile
pipeline {
    agent any

    environment {
        IMAGE_NAME = "node-app"
        CONTAINER_NAME = "node-app-container"
        APP_PORT = "3000"
    }

    stages {

        stage('Checkout') {
            steps {
                git branch: 'master',
                    url: 'https://github.com/geloasacode/node-js-sample.git'
            }
        }

        stage('Build Docker Image') {
            steps {
                sh 'docker rmi -f $IMAGE_NAME || true'
                sh 'docker build -t $IMAGE_NAME .'
            }
        }

        stage('Stop Old Container') {
            steps {
                sh 'docker rm -f $CONTAINER_NAME || true'
            }
        }

        stage('Run Container') {
            steps {
                // Run container in detached mode, only on localhost
                sh '''
                docker run -d \
                  --name $CONTAINER_NAME \
                  -p 127.0.0.1:$APP_PORT:$APP_PORT \
                  --restart unless-stopped \
                  $IMAGE_NAME
                '''
            }
        }
    }

    post {
        always {
            cleanWs()
        }
    }
}

```

### 6. Install NGINX

```bash
#! /bin/bash

set -e

sudo apt install -y nginx
sudo systemctl enable nginx
sudo systemctl start nginx

sudo rm /etc/nginx/sites-enabled/default
```

### 7. Generate SSL Certs - In this section, I used the VM's ip as CNAME

```bash
#1 /bin/bash

set -e

sudo mkdir -p /etc/nginx/ssl

sudo openssl req -x509 -nodes -days 365 \
  -newkey rsa:2048 \
  -keyout /etc/nginx/ssl/nodeapp.key \
  -out /etc/nginx/ssl/nodeapp.crt \
  -subj "/CN=172.188.82.198"
```

### 8. Configure Nginx Reverse Proxy

```nginx
server {
    listen 443 ssl;
    server_name 172.188.82.198;

    ssl_certificate /etc/nginx/ssl/nodeapp.crt;
    ssl_certificate_key /etc/nginx/ssl/nodeapp.key;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;

    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
    }
}
```


### 9. Apply and Enable config: 
```bash
#! /bin/bash

set -e

sudo ln -s /etc/nginx/sites-available/nodeapp /etc/nginx/sites-enabled/

sudo nginx -t
sudo systemctl restart nginx
```

### 10. Configure Firewall

```bash
#! /bin/bash

set -e
sudo ufw allow 22/tcp
sudo ufw allow from 127.0.0.1 to any port 8080 proto tcp
sudo ufw allow 443/tcp
sudo ufw allow 80/tcp
```

Then now you should be able to access the node app on:
https://172.188.82.198
