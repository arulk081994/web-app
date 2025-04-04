Author: [Arulkumar]
Last Updated: [04.04.2015]
Version: 1.0

1. Overview
This document outlines the steps to automate deployments to an AWS EC2 instance using GitHub Actions (GitOps). Every push to the main branch triggers a CI/CD pipeline that:
✅ Builds a Docker container
✅ Deploys it to EC2 via SSH
✅ Configures Nginx as a reverse proxy

2. Prerequisites
AWS EC2 Instance (Ubuntu/Amazon Linux)

SSH Access (Key pair .pem file)

GitHub Account (Repository for the project)

Basic Terminal Knowledge

3. Setup Steps
3.1. Configure EC2 Instance
3.1.1. Install Required Software
bash
Copy
# For Amazon Linux
sudo yum update -y
sudo yum install -y docker nginx git
sudo systemctl start docker
sudo systemctl enable docker
sudo usermod -aG docker $USER

3.1.2. Allow HTTP Traffic
EC2 Security Group: Open ports 80 (HTTP), 443 (HTTPS), and 22 (SSH).

3.2. Prepare GitHub Repository
3.2.1. Project Structure
Copy
my-webapp/
├── app.js            # Node.js server
├── Dockerfile        # Docker config
├── package.json      # Node dependencies
└── .github/workflows/deploy.yml  # GitHub Actions
3.2.2. Add GitHub Secrets
Go to:
Repo → Settings → Secrets → Actions
Add:

EC2_HOST = EC2 Public IP

EC2_USER = ec2-user (or ubuntu)

SSH_PRIVATE_KEY = Contents of .pem file

3.3. GitHub Actions Workflow
3.3.1. .github/workflows/deploy.yml
yaml
Copy
name: Deploy to EC2

on:
  push:
    branches: [ "main" ]

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Install SSH Key
        uses: shimataro/ssh-key-action@v2
        with:
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          known_hosts: "dummy-value"

      - name: Build Docker Image
        run: docker build -t simple-webapp .

      - name: Save & Deploy Image
        run: |
          docker save simple-webapp -o webapp.tar
          scp -o StrictHostKeyChecking=no webapp.tar ${{ secrets.EC2_USER }}@${{ secrets.EC2_HOST }}:~/  
          ssh -o StrictHostKeyChecking=no ${{ secrets.EC2_USER }}@${{ secrets.EC2_HOST }} "
            docker load -i ~/webapp.tar
            docker stop webapp || true
            docker rm webapp || true
            docker run -d --name webapp -p 3000:3000 simple-webapp
          "
3.4. Configure Nginx (Reverse Proxy)
3.4.1. Create Nginx Config
bash
Copy
sudo nano /etc/nginx/conf.d/webapp.conf
Paste:

nginx
Copy
server {
    listen 80;
    server_name _;

    location / {
        proxy_pass http://localhost:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
3.4.2. Restart Nginx
bash
Copy
sudo nginx -t  # Test config
sudo systemctl restart nginx
4. Testing & Validation
4.1. Trigger Deployment
Push a change to main branch:

bash
Copy
git add .
git commit -m "Test deployment"
git push origin main
Check GitHub Actions tab for logs.

4.2. Verify Deployment
Access Web App:

http://<EC2_IP> (via Nginx)

http://<EC2_IP>:3000 (direct Docker access)


