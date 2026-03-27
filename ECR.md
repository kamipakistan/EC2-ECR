# Production Deployment using AWS (ECR + EC2 + GitHub Actions)

## Overview

This guide explains how to deploy a **Machine Learning / Deep Learning project** using:

* Docker
* AWS ECR (Elastic Container Registry)
* AWS EC2
* GitHub Actions (CI/CD Pipeline)

---

## Architecture

```
Local Project → Docker Image → GitHub → CI/CD Pipeline
        ↓
      AWS ECR (Private Image Storage)
        ↓
      EC2 Instance (Deployment Server)
```

---

## Step 1: Create Dockerfile

### `Dockerfile`

```dockerfile
FROM python:3.11-slim

WORKDIR /app

COPY . /app

RUN apt-get update && apt-get install -y \
    && pip install --no-cache-dir -r requirements.txt

CMD ["python3", "app.py"]
```

### Explanation

* `FROM python:3.11-slim` → Base Linux image
* `WORKDIR /app` → Working directory inside container
* `COPY . /app` → Copy project files
* `RUN` → Install dependencies
* `CMD` → Run application

---

## Step 2: Build Docker Image (Local Test)

```bash
docker build -t image-name .
docker images
```

---

## Step 3: Push Code to GitHub

```bash
git init
git add .
git commit -m "Initial commit"
git push origin main
```

---

## Step 4: GitHub Actions CI/CD Pipeline

### `.github/workflows/main.yaml`

```yaml
name: CI-CD Pipeline

on:
  push:
    branches: [ "main" ]

jobs:

  # 🔹 Continuous Integration
  integration:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout Code
        uses: actions/checkout@v2

      - name: Lint / Test
        run: echo "Running basic checks..."

  # 🔹 Build & Push to ECR
  build-and-push-ecr:
    needs: integration
    runs-on: ubuntu-latest

    steps:
      - name: Checkout Code
        uses: actions/checkout@v2

      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v1
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ secrets.AWS_REGION }}

      - name: Login to ECR
        run: |
          aws ecr get-login-password --region $AWS_REGION | \
          docker login --username AWS --password-stdin $ECR_REGISTRY

      - name: Build and Push Docker Image
        run: |
          docker build -t $ECR_REPOSITORY .
          docker tag $ECR_REPOSITORY:latest $ECR_REGISTRY/$ECR_REPOSITORY:latest
          docker push $ECR_REGISTRY/$ECR_REPOSITORY:latest

  # 🔹 Deployment on EC2
  deploy:
    needs: build-and-push-ecr
    runs-on: self-hosted

    steps:
      - name: Pull Docker Image
        run: |
          docker pull $ECR_REGISTRY/$ECR_REPOSITORY:latest

      - name: Run Container
        run: |
          docker run -d -p 8080:8080 $ECR_REGISTRY/$ECR_REPOSITORY:latest
```

---

## Step 5: GitHub Secrets

Go to:

**Repo → Settings → Secrets → Actions**

Add:

```
AWS_ACCESS_KEY_ID
AWS_SECRET_ACCESS_KEY
AWS_REGION = us-east-1
ECR_REGISTRY = <your-ecr-uri>
ECR_REPOSITORY = student-performance
```

---

## Step 6: Create IAM User

Grant permissions:

* AmazonEC2FullAccess
* AmazonEC2ContainerRegistryFullAccess

Generate:

* Access Key
* Secret Key

---

## Step 7: Create ECR Repository

* Name: `student-performance`
* Visibility: Private

---

## 🖥️ Step 8: Launch EC2 Instance

* OS: Ubuntu
* Instance Type: t2.medium (or t3.micro for testing)
* Enable:

  * HTTP (80)
  * HTTPS (443)
  * Custom TCP (8080)

---

## Step 9: Setup Docker in EC2

```bash
sudo apt-get update
sudo apt-get upgrade -y

# Install Docker
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

# Add user to Docker group
sudo usermod -aG docker ubuntu
newgrp docker
```

---

## Step 10: Setup Self-Hosted Runner

In GitHub:

```
Settings → Actions → Runners → New Self Hosted Runner
```

Run commands on EC2:

```bash
mkdir actions-runner && cd actions-runner
curl -o actions-runner.tar.gz -L <runner-url>
tar xzf actions-runner.tar.gz

./config.sh
./run.sh
```

---

## Step 11: Trigger Deployment

Make any code change:

```bash
git commit -am "Trigger deployment"
git push
```

This will:

1. Run CI
2. Build Docker image
3. Push to ECR
4. Deploy on EC2

---

## Step 12: Access Application

```
http://<EC2-PUBLIC-IP>:8080
```

---

## Common Issues

### ECR Push Fails

* Wrong `ECR_REPOSITORY` name
* Incorrect `ECR_REGISTRY`

### App Not Accessible

* Port 8080 not open in security group

---

## Cleanup (Important to Avoid Charges)

* Terminate EC2 instance
* Delete ECR repository
* Remove IAM user
* Remove GitHub runner

---

## Key Takeaways

* Docker ensures environment consistency
* ECR stores **private images**
* EC2 runs your application
* GitHub Actions automates deployment

---
