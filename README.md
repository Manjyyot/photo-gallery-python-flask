# Flask App Deployment on EC2 with Docker and GitHub Actions

## Overview
This guide explains how to deploy a Flask application on an **Ubuntu EC2 instance** using **Docker** and **GitHub Actions** for continuous deployment.

---
## 1. Forking and Cloning the Repository
1. Fork the original repository on GitHub.
2. Clone your forked repository:
   ```sh
   git clone https://github.com/your-username/photo-gallery-python-flask.git
   ```
3. Navigate into the project directory:
   ```sh
   cd photo-gallery-python-flask
   ```

---
## 2. Setting Up an EC2 Instance
1. **Launch an EC2 instance** (Ubuntu 20.04/22.04) and connect via SSH:
   ```sh
   ssh -i your-key.pem ubuntu@your-ec2-ip
   ```
2. **Update and install Docker**:
   ```sh
   sudo apt update -y
   sudo apt install -y docker.io
   sudo systemctl start docker
   sudo systemctl enable docker
   ```
3. **Grant Docker permissions** (logout and log back in after running this):
   ```sh
   sudo usermod -aG docker $USER
   ```

---
## 3. Configuring GitHub Actions Secrets
Navigate to **GitHub repository > Settings > Secrets and variables > Actions** and add:

- `DOCKER_HUB_USERNAME` - Your Docker Hub username
- `DOCKER_HUB_PASSWORD` - Your Docker Hub password
- `EC2_HOST` - Your EC2 instance IP
- `EC2_USER` - `ubuntu`
- `EC2_SSH_KEY` - Your private SSH key
- `STAGING_SERVER_IP` - EC2 IP for staging environment

---
## 4. GitHub Actions Workflow (CI/CD Pipeline)
Create a `.github/workflows/deploy.yml` file with the following content:

```yaml
name: CI/CD Pipeline for Flask App with Docker

on:
  push:
    branches:
      - staging
      - master

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout Code
        uses: actions/checkout@v3

      - name: Login to Docker Hub
        uses: docker/login-action@v2
        with:
          username: ${{ secrets.DOCKER_HUB_USERNAME }}
          password: ${{ secrets.DOCKER_HUB_PASSWORD }}

      - name: Build and Push Docker Image
        uses: docker/build-push-action@v4
        with:
          context: .
          file: ./Dockerfile
          push: true
          tags: |
            ${{ secrets.DOCKER_HUB_USERNAME }}/flask-app:latest
            ${{ secrets.DOCKER_HUB_USERNAME }}/flask-app:${{ github.sha }}

  deploy:
    needs: build
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/master'
    steps:
      - name: Deploy to EC2
        uses: appleboy/ssh-action@master
        with:
          host: ${{ secrets.EC2_HOST }}
          username: ${{ secrets.EC2_USER }}
          key: ${{ secrets.EC2_SSH_KEY }}
          script: |
            sudo docker stop flask-app || true
            sudo docker rm flask-app || true
            sudo docker pull ${{ secrets.DOCKER_HUB_USERNAME }}/flask-app:latest
            sudo docker run -d --name flask-app -p 5000:5000 \
              -e ENV=production \
              --restart unless-stopped \
              ${{ secrets.DOCKER_HUB_USERNAME }}/flask-app:latest
```

---
## 5. Setting Up Nginx as a Reverse Proxy
1. **Install Nginx**:
   ```sh
   sudo apt install -y nginx
   ```
2. **Edit Nginx configuration**:
   ```sh
   sudo nano /etc/nginx/sites-available/default
   ```
   Replace contents with:
   ```nginx
   server {
       listen 80;
       server_name your-ec2-ip;

       location / {
           proxy_pass http://127.0.0.1:5000;
           proxy_set_header Host $host;
           proxy_set_header X-Real-IP $remote_addr;
           proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
       }
   }
   ```
3. **Restart Nginx**:
   ```sh
   sudo systemctl restart nginx
   ```

---
## 6. Testing Deployment
1. Push changes to GitHub (`master` branch triggers deployment):
   ```sh
   git add .
   git commit -m "Deploying Flask app"
   git push origin master
   ```
2. Check logs on EC2:
   ```sh
   docker logs flask-app
   ```
3. Open in browser:
   ```sh
   http://your-ec2-ip
   ```

---
## 7. Troubleshooting
- **Check if the container is running**:
  ```sh
  docker ps
  ```
- **Check Nginx logs**:
  ```sh
  sudo journalctl -u nginx --no-pager | tail -n 50
  ```
- **Check application logs**:
  ```sh
  docker logs flask-app
  ```

---
## Conclusion
This guide sets up a fully automated **CI/CD pipeline** using **GitHub Actions, Docker, and Nginx** to deploy a Flask app on an EC2 instance.
