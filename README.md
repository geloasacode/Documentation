Task Assignment:

You can find below the steps made to install a working node app accessible via nginx proxy: 

### 1. Install Docker 
Just the basic installation and also we add ubuntu user to the docker group so that we don't need to use "sudo" everytime we try to run docker command. Same will be applied to jenkins user.

```bash
#! /bin/bash

set -ex

sudo apt update
sudo apt install -y docker.io
sudo systemctl enable docker
sudo systemctl start docker

sudo usermod -aG docker ubuntu
```
### 2. Install Jenkins 
After Installation, it can be accessed through http://172.188.82.198:8080. Password can be found on /var/lib/jenkins/secrets/initialAdminPassword

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
I forked the repository https://github.com/heroku/node-js-sample since I will be needing to write a Dockerfile that will be used by jenkins later on the process.
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

### Pushed To Repo
Then I pushed the changes to my repository

```bash
git add Dockerfile
git commit -m "Add Dockerfile"
git push
```

### 5. Create Jenkins Pipeline
This will build the app image and will run the container on the VM, noticed that I have referenced the forked repository on my github repository to where Dockerfile is added. We need Dockerfile since this is the blueprint the docker will follow in creating our application image.

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

### 7. Generate SSL Certs
In this section, I used the VM's ip as CNAME. Also this is where we create our private key and TLS crt.

```bash
#! /bin/bash

set -e

sudo mkdir -p /etc/nginx/ssl

sudo openssl req -x509 -nodes -days 365 \
  -newkey rsa:2048 \
  -keyout /etc/nginx/ssl/nodeapp.key \
  -out /etc/nginx/ssl/nodeapp.crt \
  -subj "/CN=172.188.82.198"
```

### 8. Configure Nginx Reverse Proxy
This is where we map our nginx to acts as proxy. So basically it listens on port 443 with the specified server_name and it uses the TLS certificate and key we generated to encrypt traffic.
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
We use "ufw" to whitelist ports 22, 443, 80, and allow any port but 8080 to access localhost
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
