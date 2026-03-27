# Complete Production-Grade Deployment Guide: ML/DL Project on AWS using ECR, EC2, and GitHub Actions

## Table of Contents
1. [Architecture Overview](#architecture-overview)
2. [Prerequisites](#prerequisites)
3. [Step 1: Project Setup and Dockerization](#step-1-project-setup-and-dockerization)
4. [Step 2: GitHub Repository Setup](#step-2-github-repository-setup)
5. [Step 3: GitHub Actions CI/CD Pipeline](#step-3-github-actions-cicd-pipeline)
6. [Step 4: AWS IAM User Configuration](#step-4-aws-iam-user-configuration)
7. [Step 5: Amazon ECR Repository Creation](#step-5-amazon-ecr-repository-creation)
8. [Step 6: EC2 Instance Setup](#step-6-ec2-instance-setup)
9. [Step 7: Docker Installation on EC2](#step-7-docker-installation-on-ec2)
10. [Step 8: GitHub Self-Hosted Runner Configuration](#step-8-github-self-hosted-runner-configuration)
11. [Step 9: GitHub Secrets Configuration](#step-9-github-secrets-configuration)
12. [Step 10: Deployment and Testing](#step-10-deployment-and-testing)
13. [Step 11: Troubleshooting Common Issues](#step-11-troubleshooting-common-issues)
14. [Step 12: Cleanup and Cost Management](#step-12-cleanup-and-cost-management)
15. [Step 13: Security Best Practices](#step-13-security-best-practices)
16. [Step 14: Scaling Considerations](#step-14-scaling-considerations)
17. [Step 15: Monitoring and Logging](#step-15-monitoring-and-logging)

---

## Architecture Overview

### System Architecture Diagram

```
┌─────────────────┐     ┌──────────────────┐     ┌─────────────────┐
│                 │     │                  │     │                 │
│  Local Machine  │────▶│   GitHub Repo    │────▶│  GitHub Actions │
│  (Development)  │     │   (Source Code)  │     │   (CI/CD)       │
│                 │     │                  │     │                 │
└─────────────────┘     └──────────────────┘     └────────┬────────┘
                                                          │
                                                          ▼
┌─────────────────┐     ┌──────────────────┐     ┌─────────────────┐
│                 │     │                  │     │                 │
│   End Users     │────▶│   AWS EC2        │◀────│   AWS ECR       │
│   (HTTP/HTTPS)  │     │   (Production)   │     │   (Image Store) │
│                 │     │                  │     │                 │
└─────────────────┘     └──────────────────┘     └─────────────────┘
```

### Deployment Flow

1. **Development Phase**
   - Local development of ML/DL project
   - Dockerfile creation for containerization
   - Local testing with Docker

2. **CI/CD Pipeline (GitHub Actions)**
   - **Continuous Integration (CI)**: Code linting, unit testing
   - **Build Phase**: Docker image creation
   - **Push Phase**: Image upload to AWS ECR
   - **Deployment Phase**: Pull and run on EC2 via self-hosted runner

3. **Production Infrastructure**
   - **AWS ECR**: Private Docker image registry
   - **AWS EC2**: Production server running containers
   - **Self-Hosted Runner**: GitHub Actions runner on EC2

### Technology Stack

| Component | Technology | Purpose |
|-----------|------------|---------|
| Containerization | Docker | Application packaging and isolation |
| Source Control | GitHub | Version control and CI/CD trigger |
| CI/CD | GitHub Actions | Automated build and deployment |
| Image Registry | AWS ECR | Private Docker image storage |
| Compute | AWS EC2 | Production server |
| Operating System | Ubuntu 22.04 LTS | Server OS |
| Application | Python 3.8/3.11 | ML/DL application runtime |

---

## Prerequisites

### Required Accounts and Tools

| Resource | Purpose | Link |
|----------|---------|------|
| AWS Account | Cloud infrastructure | [aws.amazon.com](https://aws.amazon.com) |
| GitHub Account | Source code and CI/CD | [github.com](https://github.com) |
| Docker Desktop | Local container testing | [docker.com](https://docker.com) |
| Git | Version control | [git-scm.com](https://git-scm.com) |
| Python 3.8+ | Local development | [python.org](https://python.org) |

### Knowledge Prerequisites

- Basic understanding of Docker concepts (images, containers, Dockerfiles)
- Familiarity with Git commands (commit, push, pull)
- Basic Linux command line knowledge
- Understanding of AWS services (EC2, ECR, IAM)

### Estimated Costs (Monthly)

| Service | t3.micro | t3.medium | t3.xlarge |
|---------|----------|-----------|-----------|
| EC2 (on-demand) | ~$8.50 | ~$34.00 | ~$136.00 |
| ECR Storage (per GB) | $0.10 | $0.10 | $0.10 |
| Data Transfer (first 1GB) | Free | Free | Free |
| **Total Estimate** | **~$10-15** | **~$35-40** | **~$140-150** |

---

## Step 1: Project Setup and Dockerization

### 1.1 Project Structure

Before creating the Dockerfile, ensure your ML/DL project has a standard structure:

```
student-performance-prediction/
├── app.py                 # Main application entry point
├── requirements.txt       # Python dependencies
├── Dockerfile            # Container configuration
├── .gitignore           # Git ignore rules
├── README.md            # Project documentation
├── models/              # Trained ML models
│   └── model.pkl
├── data/                # Data files
│   └── processed_data.csv
├── notebooks/           # Jupyter notebooks (optional)
├── src/                 # Source code modules
│   ├── __init__.py
│   ├── preprocessing.py
│   └── predict.py
└── tests/               # Unit tests
    └── test_app.py
```

### 1.2 Create Dockerfile

Create a `Dockerfile` in your project root with the following content:

```dockerfile
# Stage 1: Base image
# Using Python 3.8-slim for smaller image size
FROM python:3.8-slim-buster

# Set environment variables
ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    PIP_NO_CACHE_DIR=1 \
    PIP_DISABLE_PIP_VERSION_CHECK=1

# Set working directory
WORKDIR /app

# Install system dependencies
RUN apt-get update && apt-get install -y \
    gcc \
    g++ \
    && rm -rf /var/lib/apt/lists/*

# Copy requirements first (for better caching)
COPY requirements.txt .

# Install Python dependencies
RUN pip install --no-cache-dir -r requirements.txt

# Copy project files
COPY . .

# Expose application port
EXPOSE 8080

# Run the application
CMD ["python3", "app.py"]
```

### 1.3 Create requirements.txt

```txt
# Core dependencies
numpy==1.24.3
pandas==2.0.3
scikit-learn==1.3.0
flask==2.3.3
gunicorn==21.2.0

# Optional: For deep learning projects
# torch==2.0.1
# tensorflow==2.13.0
# transformers==4.31.0
```

### 1.4 Sample app.py

```python
from flask import Flask, request, jsonify, render_template
import numpy as np
import pandas as pd
import pickle
import os

app = Flask(__name__)

# Load model (adjust path as needed)
MODEL_PATH = os.environ.get('MODEL_PATH', 'models/model.pkl')

try:
    with open(MODEL_PATH, 'rb') as f:
        model = pickle.load(f)
    print("Model loaded successfully")
except:
    print("Model not found, using dummy model")
    model = None

@app.route('/')
def home():
    return render_template('index.html')

@app.route('/predict', methods=['POST'])
def predict():
    try:
        # Get data from request
        data = request.get_json()
        
        # Convert to DataFrame
        df = pd.DataFrame([data])
        
        # Make prediction
        if model:
            prediction = model.predict(df)
            result = float(prediction[0])
        else:
            # Dummy prediction
            result = 44.875
        
        return jsonify({
            'status': 'success',
            'prediction': result
        })
    except Exception as e:
        return jsonify({
            'status': 'error',
            'message': str(e)
        }), 400

if __name__ == '__main__':
    port = int(os.environ.get('PORT', 8080))
    app.run(host='0.0.0.0', port=port, debug=False)
```

### 1.5 Local Docker Testing

Build and test the Docker image locally:

```bash
# Build the Docker image
docker build -t student-performance:latest .

# List images to verify
docker images

# Run the container locally
docker run -d -p 8080:8080 --name student-performance student-performance:latest

# Test the application
curl http://localhost:8080

# View logs
docker logs student-performance

# Stop and remove container
docker stop student-performance
docker rm student-performance
```

---

## Step 2: GitHub Repository Setup

### 2.1 Create GitHub Repository

1. Log in to your GitHub account
2. Click the **+** icon in the top-right corner
3. Select **New repository**
4. Configure repository settings:

| Setting | Value |
|---------|-------|
| Repository name | `student-performance-prediction` |
| Description | ML project for student performance prediction with CI/CD |
| Visibility | Public or Private (choose based on your needs) |
| Initialize with README | Yes |
| .gitignore | Python |
| License | MIT (or your preferred license) |

### 2.2 Push Code to GitHub

```bash
# Initialize git repository (if not already)
git init

# Add remote origin
git remote add origin https://github.com/your-username/student-performance-prediction.git

# Add all files
git add .

# Commit changes
git commit -m "Initial commit: Add ML project with Dockerfile"

# Push to main branch
git branch -M main
git push -u origin main
```

### 2.3 Create .gitignore

```gitignore
# Python
__pycache__/
*.py[cod]
*$py.class
*.so
.Python
env/
venv/
ENV/
build/
develop-eggs/
dist/
downloads/
eggs/
.eggs/
lib/
lib64/
parts/
sdist/
var/
wheels/
*.egg-info/
.installed.cfg
*.egg

# Virtual Environment
venv/
.env
.venv

# IDE
.vscode/
.idea/
*.swp
*.swo
*~

# OS
.DS_Store
.DS_Store?
._*
.Spotlight-V100
.Trashes
ehthumbs.db
Thumbs.db

# Project specific
models/*.pkl
data/raw/
*.log
*.db
```

---

## Step 3: GitHub Actions CI/CD Pipeline

### 3.1 Create Workflow Directory

```bash
mkdir -p .github/workflows
```

### 3.2 Create main.yaml Workflow File

Create `.github/workflows/main.yaml`:

```yaml
name: CI/CD Pipeline for ML Deployment

on:
  push:
    branches: [ "main" ]
    paths-ignore:
      - 'README.md'
      - '.gitignore'
  pull_request:
    branches: [ "main" ]

# Environment variables
env:
  AWS_REGION: ${{ secrets.AWS_REGION }}
  ECR_REGISTRY: ${{ secrets.ECR_REGISTRY }}
  ECR_REPOSITORY: ${{ secrets.ECR_REPOSITORY }}
  CONTAINER_NAME: student-performance

jobs:
  # Job 1: Continuous Integration
  continuous-integration:
    name: 🔍 Continuous Integration
    runs-on: ubuntu-latest
    
    steps:
      - name: 📥 Checkout Code
        uses: actions/checkout@v3
      
      - name: 🐍 Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.8'
      
      - name: 📦 Install Dependencies
        run: |
          python -m pip install --upgrade pip
          pip install -r requirements.txt
          pip install pytest flake8 black
      
      - name: 🔍 Lint Code with flake8
        run: |
          # Stop the build if there are Python syntax errors or undefined names
          flake8 . --count --select=E9,F63,F7,F82 --show-source --statistics
          # Exit-zero treats all errors as warnings
          flake8 . --count --exit-zero --max-complexity=10 --max-line-length=127 --statistics
      
      - name: 🎨 Check Code Formatting with Black
        run: |
          black --check --diff .
      
      - name: 🧪 Run Unit Tests
        run: |
          pytest tests/ -v --tb=short
      
      - name: ✅ Integration Complete
        run: echo "✅ Continuous Integration completed successfully!"

  # Job 2: Build and Push to ECR
  build-and-push-ecr:
    name: 🏗️ Build & Push to AWS ECR
    needs: continuous-integration
    runs-on: ubuntu-latest
    
    steps:
      - name: 📥 Checkout Code
        uses: actions/checkout@v3
      
      - name: 🔧 Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v1
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ secrets.AWS_REGION }}
      
      - name: 🔑 Login to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v1
      
      - name: 🏗️ Build Docker Image
        run: |
          docker build -t $ECR_REPOSITORY:latest .
          docker tag $ECR_REPOSITORY:latest $ECR_REGISTRY/$ECR_REPOSITORY:latest
          docker tag $ECR_REPOSITORY:latest $ECR_REGISTRY/$ECR_REPOSITORY:${GITHUB_SHA}
      
      - name: 📤 Push Docker Image to ECR
        run: |
          docker push $ECR_REGISTRY/$ECR_REPOSITORY:latest
          docker push $ECR_REGISTRY/$ECR_REPOSITORY:${GITHUB_SHA}
      
      - name: 📝 Output Image Info
        run: |
          echo "✅ Image pushed successfully!"
          echo "Image: $ECR_REGISTRY/$ECR_REPOSITORY:latest"
          echo "Image with SHA: $ECR_REGISTRY/$ECR_REPOSITORY:${GITHUB_SHA}"

  # Job 3: Deploy to EC2
  deploy-to-ec2:
    name: 🚀 Deploy to EC2
    needs: build-and-push-ecr
    runs-on: self-hosted
    
    steps:
      - name: 📥 Pull Latest Image from ECR
        run: |
          # Login to ECR
          aws ecr get-login-password --region $AWS_REGION | docker login --username AWS --password-stdin $ECR_REGISTRY
          
          # Pull the latest image
          docker pull $ECR_REGISTRY/$ECR_REPOSITORY:latest
      
      - name: 🛑 Stop and Remove Existing Container
        run: |
          # Stop running container if exists
          docker stop $CONTAINER_NAME || true
          docker rm $CONTAINER_NAME || true
      
      - name: 🚀 Run New Container
        run: |
          docker run -d \
            --name $CONTAINER_NAME \
            --restart unless-stopped \
            -p 8080:8080 \
            -e PORT=8080 \
            $ECR_REGISTRY/$ECR_REPOSITORY:latest
      
      - name: 🧹 Clean Up Old Images
        run: |
          # Remove unused images
          docker image prune -f
      
      - name: ✅ Verify Deployment
        run: |
          # Wait for container to start
          sleep 10
          
          # Check if container is running
          if docker ps | grep -q $CONTAINER_NAME; then
            echo "✅ Container is running successfully!"
          else
            echo "❌ Container failed to start"
            docker logs $CONTAINER_NAME
            exit 1
          fi
```

### 3.3 Create Unit Tests

Create `tests/test_app.py`:

```python
import pytest
import sys
import os

# Add project root to path
sys.path.insert(0, os.path.abspath(os.path.join(os.path.dirname(__file__), '..')))

def test_imports():
    """Test that required modules can be imported"""
    try:
        import flask
        import numpy
        import pandas
        import sklearn
        assert True
    except ImportError as e:
        assert False, f"Import failed: {e}"

def test_app_exists():
    """Test that app.py exists"""
    assert os.path.exists('app.py'), "app.py not found"

def test_requirements_exist():
    """Test that requirements.txt exists"""
    assert os.path.exists('requirements.txt'), "requirements.txt not found"
    
    # Check if requirements.txt has content
    with open('requirements.txt', 'r') as f:
        content = f.read()
        assert len(content) > 0, "requirements.txt is empty"

def test_dockerfile_exists():
    """Test that Dockerfile exists"""
    assert os.path.exists('Dockerfile'), "Dockerfile not found"
    
    # Check if Dockerfile has content
    with open('Dockerfile', 'r') as f:
        content = f.read()
        assert len(content) > 0, "Dockerfile is empty"
        assert "FROM" in content, "Dockerfile missing FROM instruction"
        assert "WORKDIR" in content, "Dockerfile missing WORKDIR instruction"
        assert "CMD" in content, "Dockerfile missing CMD instruction"
```

---

## Step 4: AWS IAM User Configuration

### 4.1 Create IAM User

1. Log in to AWS Console
2. Navigate to **IAM** (Identity and Access Management)
3. Click **Users** → **Create user**

### 4.2 Configure IAM User

| Setting | Value |
|---------|-------|
| User name | `github-actions-deploy` |
| Provide user access to AWS Management Console | ❌ Uncheck |
| Create IAM user | ✅ Check |

### 4.3 Set Permissions

1. Click **Attach policies directly**
2. Search and select:

| Policy Name | Purpose |
|-------------|---------|
| `AmazonEC2FullAccess` | Manage EC2 instances |
| `AmazonEC2ContainerRegistryFullAccess` | Manage ECR repositories |
| `AmazonECS_FullAccess` | (Optional) For ECS integration |

**Alternative: Create Custom Policy**

For least-privilege access, create a custom policy:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "ecr:GetAuthorizationToken",
                "ecr:BatchCheckLayerAvailability",
                "ecr:GetDownloadUrlForLayer",
                "ecr:GetRepositoryPolicy",
                "ecr:DescribeRepositories",
                "ecr:ListImages",
                "ecr:DescribeImages",
                "ecr:BatchGetImage",
                "ecr:InitiateLayerUpload",
                "ecr:UploadLayerPart",
                "ecr:CompleteLayerUpload",
                "ecr:PutImage"
            ],
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "ec2:DescribeInstances",
                "ec2:StartInstances",
                "ec2:StopInstances",
                "ec2:RebootInstances"
            ],
            "Resource": "*"
        }
    ]
}
```

### 4.4 Generate Access Keys

1. Click on the created user
2. Go to **Security credentials** tab
3. Click **Create access key**
4. Select **Command Line Interface (CLI)**
5. Click **Create access key**
6. **Download the CSV file** containing:
   - Access Key ID
   - Secret Access Key

> ⚠️ **IMPORTANT**: Save these credentials securely. You cannot retrieve the secret access key again.

---

## Step 5: Amazon ECR Repository Creation

### 5.1 Create ECR Repository

1. Navigate to **ECR** in AWS Console
2. Click **Create repository**
3. Configure repository:

| Setting | Value |
|---------|-------|
| Visibility setting | Private |
| Repository name | `student-performance` |
| Tag immutability | Disabled (or Enabled for production) |
| Scan on push | ✅ Enable (security best practice) |
| KMS encryption | AWS managed key (default) |

### 5.2 Note Repository Details

After creation, note the following:

- **Repository URI**: `{account-id}.dkr.ecr.{region}.amazonaws.com/student-performance`
- Example: `123456789012.dkr.ecr.us-east-1.amazonaws.com/student-performance`

### 5.3 Test ECR Access Locally (Optional)

```bash
# Login to ECR
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin {account-id}.dkr.ecr.us-east-1.amazonaws.com

# Build and push test image
docker tag student-performance:latest {account-id}.dkr.ecr.us-east-1.amazonaws.com/student-performance:latest
docker push {account-id}.dkr.ecr.us-east-1.amazonaws.com/student-performance:latest

# Verify image is pushed
aws ecr list-images --repository-name student-performance
```

---

## Step 6: EC2 Instance Setup

### 6.1 Launch EC2 Instance

1. Navigate to **EC2** → **Instances**
2. Click **Launch instance**
3. Configure instance:

| Setting | Value | Notes |
|---------|-------|-------|
| Name | `student-performance-prod` | Descriptive name |
| AMI | Ubuntu Server 22.04 LTS (HVM), SSD Volume Type | 64-bit (x86) |
| Instance type | t3.medium (or t2.medium for lower cost) | Choose based on workload |
| Key pair | Create new: `student-performance-key` | RSA, .pem format |
| VPC | Default VPC | Or your custom VPC |
| Auto-assign public IP | Enable | For external access |
| Security group | Create new: `student-performance-sg` | Configure rules below |

### 6.2 Configure Security Group

Add the following inbound rules:

| Type | Protocol | Port | Source | Description |
|------|----------|------|--------|-------------|
| SSH | TCP | 22 | My IP | Secure shell access |
| HTTP | TCP | 80 | 0.0.0.0/0 | Web traffic |
| HTTPS | TCP | 443 | 0.0.0.0/0 | Secure web traffic |
| Custom TCP | TCP | 8080 | 0.0.0.0/0 | Application port |
| Custom TCP | TCP | 3000-4000 | 0.0.0.0/0 | Additional services |

### 6.3 Configure Storage

| Setting | Recommended | Notes |
|---------|-------------|-------|
| Root volume size | 30 GB | General purpose SSD (gp3) |
| Delete on termination | ✅ Enable | Save costs |
| Additional volumes | 50 GB (optional) | For data persistence |

### 6.4 Advanced Details (Optional)

Add user data script for automatic setup:

```bash
#!/bin/bash
# User data script for EC2 initialization
apt-get update
apt-get upgrade -y
apt-get install -y curl git
curl -fsSL https://get.docker.com -o get-docker.sh
sh get-docker.sh
usermod -aG docker ubuntu
```

### 6.5 Record EC2 Details

After instance is running:

- **Public IP**: `54.123.45.67`
- **Public DNS**: `ec2-54-123-45-67.compute-1.amazonaws.com`
- **Instance ID**: `i-1234567890abcdef0`

---

## Step 7: Docker Installation on EC2

### 7.1 Connect to EC2

```bash
# Set proper key permissions
chmod 400 ~/.ssh/student-performance-key.pem

# Connect via SSH
ssh -i "~/.ssh/student-performance-key.pem" ubuntu@<EC2_PUBLIC_IP>
```

### 7.2 Update System

```bash
# Update package index
sudo apt update -y

# Upgrade all packages
sudo apt upgrade -y

# Install prerequisite packages
sudo apt install -y \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg \
    lsb-release \
    git \
    wget \
    htop
```

### 7.3 Install Docker

```bash
# Method 1: Using official Docker installation script
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

# Method 2: Manual installation (if script fails)
# Add Docker's official GPG key
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

# Add Docker repository
echo "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list

# Install Docker Engine
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin
```

### 7.4 Configure Docker

```bash
# Verify Docker installation
sudo docker --version

# Add ubuntu user to docker group (avoid using sudo)
sudo usermod -aG docker ubuntu

# Apply group changes
newgrp docker

# Verify docker without sudo
docker --version

# Enable Docker to start on boot
sudo systemctl enable docker
sudo systemctl start docker

# Check Docker status
sudo systemctl status docker
```

### 7.5 Install AWS CLI

```bash
# Install AWS CLI v2
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

# Verify installation
aws --version

# Configure AWS credentials (for ECR access)
aws configure
# Enter:
# AWS Access Key ID: [from IAM user]
# AWS Secret Access Key: [from IAM user]
# Default region: [us-east-1]
# Default output format: json
```

### 7.6 Test Docker with Simple Container

```bash
# Run a test container
docker run hello-world

# Should see confirmation message
# If successful, Docker is working correctly
```

---

## Step 8: GitHub Self-Hosted Runner Configuration

### 8.1 Add Self-Hosted Runner to GitHub

1. Go to your GitHub repository
2. Navigate to **Settings** → **Actions** → **Runners**
3. Click **New self-hosted runner**
4. Select **Linux** as operating system
5. Follow the commands shown

### 8.2 Install Runner on EC2

Connect to EC2 and execute the provided commands:

```bash
# Create runner directory
mkdir actions-runner && cd actions-runner

# Download the runner package (URL from GitHub)
curl -o actions-runner-linux-x64-2.308.0.tar.gz -L https://github.com/actions/runner/releases/download/v2.308.0/actions-runner-linux-x64-2.308.0.tar.gz

# Extract the package
tar xzf ./actions-runner-linux-x64-2.308.0.tar.gz

# Configure the runner
./config.sh --url https://github.com/your-username/student-performance-prediction \
            --token YOUR_GENERATED_TOKEN \
            --name ec2-runner \
            --labels self-hosted \
            --unattended

# Run the runner (for testing)
./run.sh

# To run in background, use:
nohup ./run.sh & > runner.log 2>&1 &
```

### 8.3 Set Up Runner as a Service

For production, set up the runner as a system service:

```bash
# Install the runner as a service
sudo ./svc.sh install

# Start the service
sudo ./svc.sh start

# Check service status
sudo ./svc.sh status

# View logs
sudo ./svc.sh logs
```

### 8.4 Verify Runner Connection

1. Go back to GitHub repository
2. Navigate to **Settings** → **Actions** → **Runners**
3. You should see your runner with status **Idle** (green dot)

### 8.5 Test Runner with Simple Workflow

Create a test workflow to verify runner works:

```yaml
# .github/workflows/test-runner.yaml
name: Test Self-Hosted Runner

on:
  workflow_dispatch

jobs:
  test-runner:
    runs-on: self-hosted
    steps:
      - name: Test Runner
        run: |
          echo "Runner is working!"
          docker --version
          aws --version
```

Trigger this workflow manually and verify it runs on your EC2 instance.

---

## Step 9: GitHub Secrets Configuration

### 9.1 Add Secrets to Repository

Go to your GitHub repository → **Settings** → **Secrets and variables** → **Actions** → **New repository secret**

Add the following secrets:

| Secret Name | Value | Description |
|-------------|-------|-------------|
| `AWS_ACCESS_KEY_ID` | `AKIA...` | IAM user access key |
| `AWS_SECRET_ACCESS_KEY` | `...` | IAM user secret key |
| `AWS_REGION` | `us-east-1` | Your AWS region |
| `ECR_REGISTRY` | `123456789012.dkr.ecr.us-east-1.amazonaws.com` | ECR registry URL |
| `ECR_REPOSITORY` | `student-performance` | ECR repository name |

### 9.2 Verify Secrets

To verify secrets are correctly set, create a test workflow:

```yaml
name: Test Secrets

on:
  workflow_dispatch

jobs:
  test-secrets:
    runs-on: ubuntu-latest
    steps:
      - name: Check Secrets
        run: |
          echo "AWS Region: ${{ secrets.AWS_REGION }}"
          echo "ECR Registry: ${{ secrets.ECR_REGISTRY }}"
          echo "ECR Repository: ${{ secrets.ECR_REPOSITORY }}"
```

### 9.3 Configure AWS Credentials on EC2

For the self-hosted runner, configure AWS credentials:

```bash
# On EC2 instance
aws configure

# Enter:
AWS Access Key ID: [from GitHub secrets]
AWS Secret Access Key: [from GitHub secrets]
Default region: us-east-1
Default output format: json
```

Alternatively, use IAM roles (more secure):

```bash
# Create IAM role for EC2 with required permissions
# Attach to EC2 instance
# Then no credentials needed on EC2
```

---

## Step 10: Deployment and Testing

### 10.1 Trigger First Deployment

Make a code change to trigger the CI/CD pipeline:

```bash
# Create or modify a file
echo "# Updated deployment" >> README.md

# Commit and push
git add .
git commit -m "Trigger deployment: Add README updates"
git push origin main
```

### 10.2 Monitor Deployment

1. Go to GitHub repository → **Actions** tab
2. See the workflow running
3. Monitor each job:
   - Continuous Integration
   - Build and Push to ECR
   - Deploy to EC2

### 10.3 Verify ECR Image

Check if image was pushed to ECR:

```bash
# List images in ECR
aws ecr list-images --repository-name student-performance

# Describe image
aws ecr describe-images --repository-name student-performance --image-ids imageTag=latest
```

### 10.4 Verify EC2 Deployment

On EC2 instance:

```bash
# Check running containers
docker ps

# Should see:
# CONTAINER ID   IMAGE                    COMMAND      STATUS        PORTS
# xxxxx          student-performance:latest   "python3 app.py"   Up 10 seconds   0.0.0.0:8080->8080/tcp

# Check container logs
docker logs student-performance

# Test locally on EC2
curl http://localhost:8080
```

### 10.5 Test Application

Access your application:

```
http://<EC2_PUBLIC_IP>:8080
```

Test prediction endpoint:

```bash
# Test with curl
curl -X POST http://<EC2_PUBLIC_IP>:8080/predict \
  -H "Content-Type: application/json" \
  -d '{
    "gender": "female",
    "race_ethnicity": "group B",
    "parental_level_of_education": "bachelors degree",
    "lunch": "standard",
    "test_preparation_course": "none",
    "reading_score": 72,
    "writing_score": 74
  }'
```

### 10.6 Test with Postman

1. Open Postman
2. Create new request:
   - Method: POST
   - URL: `http://<EC2_PUBLIC_IP>:8080/predict`
   - Headers: `Content-Type: application/json`
   - Body (raw JSON):

```json
{
    "gender": "female",
    "race_ethnicity": "group B",
    "parental_level_of_education": "bachelors degree",
    "lunch": "standard",
    "test_preparation_course": "none",
    "reading_score": 72,
    "writing_score": 74
}
```

---

## Step 11: Troubleshooting Common Issues

### 11.1 ECR Push Fails

**Error**: `denied: User not authorized to perform action`

**Solutions**:
```bash
# Check IAM permissions
aws iam list-attached-user-policies --user-name github-actions-deploy

# Verify ECR repository exists
aws ecr describe-repositories --repository-names student-performance

# Test login manually
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin $ECR_REGISTRY
```

### 11.2 Container Fails to Start on EC2

**Error**: `container exited with code 1`

**Solutions**:
```bash
# Check logs
docker logs student-performance

# Run container in interactive mode for debugging
docker run -it --rm $ECR_REGISTRY/$ECR_REPOSITORY:latest /bin/bash

# Check if port is already in use
sudo lsof -i :8080

# Check disk space
df -h
```

### 11.3 Application Not Accessible

**Error**: `Connection refused` or `Timeout`

**Solutions**:
```bash
# Check if container is running
docker ps | grep student-performance

# Check security group rules
aws ec2 describe-security-groups --group-ids sg-xxxxx

# Check if application is listening on correct interface
docker exec student-performance netstat -tlnp

# Check EC2 firewall (UFW)
sudo ufw status
```

### 11.4 Self-Hosted Runner Issues

**Error**: `Runner not connecting`

**Solutions**:
```bash
# Check runner service status
sudo ./svc.sh status

# View runner logs
tail -f /home/ubuntu/actions-runner/_diag/Runner_*.log

# Restart runner
sudo ./svc.sh restart

# Re-configure runner
./config.sh --unconfigure
./config.sh --url https://github.com/your-username/repo --token TOKEN
```

### 11.5 Docker Build Fails

**Error**: `Failed to build Docker image`

**Solutions**:
```bash
# Check Dockerfile syntax
docker build --no-cache -t test .

# Verify requirements.txt exists and is valid
pip install -r requirements.txt

# Check disk space on runner
df -h

# Clear Docker cache
docker system prune -a -f
```

### 11.6 AWS Credentials Issues

**Error**: `Unable to locate credentials`

**Solutions**:
```bash
# Check AWS credentials on EC2
aws sts get-caller-identity

# Verify IAM role is attached (if using instance profile)
curl http://169.254.169.254/latest/meta-data/iam/security-credentials/

# Check environment variables
env | grep AWS
```

---

## Step 12: Cleanup and Cost Management

### 12.1 Stop Services

```bash
# Stop Docker container
docker stop student-performance
docker rm student-performance

# Stop self-hosted runner service
cd ~/actions-runner
sudo ./svc.sh stop
sudo ./svc.sh uninstall
```

### 12.2 Terminate EC2 Instance

1. Go to EC2 Console → Instances
2. Select your instance
3. Click **Instance State** → **Terminate**
4. Confirm termination

### 12.3 Delete ECR Repository

```bash
# Delete ECR repository
aws ecr delete-repository --repository-name student-performance --force

# Or via AWS Console:
# ECR → Repositories → student-performance → Delete
```

### 12.4 Delete IAM User

1. Go to IAM Console → Users
2. Select `github-actions-deploy`
3. Click **Delete user**
4. Confirm deletion

### 12.5 Delete Security Group

1. Go to EC2 Console → Security Groups
2. Select `student-performance-sg`
3. Click **Actions** → **Delete**

### 12.6 Remove GitHub Secrets and Runner

```bash
# Remove runner from GitHub
# Settings → Actions → Runners → Remove runner

# Delete secrets from GitHub
# Settings → Secrets → Delete each secret
```

### 12.7 Cost Verification

Check AWS Cost Explorer to ensure no ongoing charges:

```bash
# Check current month's costs
aws ce get-cost-and-usage --time-period Start=$(date +%Y-%m-01),End=$(date +%Y-%m-%d) --granularity MONTHLY --metrics "BlendedCost"
```

---

## Step 13: Security Best Practices

### 13.1 IAM Security

**Principle of Least Privilege**:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "ecr:GetAuthorizationToken",
                "ecr:PutImage",
                "ecr:InitiateLayerUpload",
                "ecr:UploadLayerPart",
                "ecr:CompleteLayerUpload"
            ],
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "ec2:DescribeInstances"
            ],
            "Resource": "*"
        }
    ]
}
```

### 13.2 EC2 Security

**Use IAM Roles instead of Access Keys**:

```bash
# Create IAM role for EC2
aws iam create-role --role-name EC2-GitHubRunner-Role \
    --assume-role-policy-document file://trust-policy.json

# Attach policies
aws iam attach-role-policy --role-name EC2-GitHubRunner-Role \
    --policy-arn arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly

# Attach role to EC2 instance
aws ec2 associate-iam-instance-profile \
    --instance-id i-xxxxx \
    --iam-instance-profile Name=EC2-GitHubRunner-Role
```

### 13.3 Network Security

**Restrict Security Group Rules**:

```bash
# Create security group with strict rules
aws ec2 authorize-security-group-ingress \
    --group-id sg-xxxxx \
    --protocol tcp \
    --port 22 \
    --cidr YOUR_HOME_IP/32

# For production, use a load balancer instead of direct access
```

### 13.4 Docker Security

```dockerfile
# Use non-root user in Dockerfile
FROM python:3.8-slim

RUN useradd -m -u 1000 appuser
WORKDIR /app
COPY . .
RUN chown -R appuser:appuser /app
USER appuser

CMD ["python3", "app.py"]
```

### 13.5 Secret Management

**Use AWS Secrets Manager for sensitive data**:

```bash
# Store database credentials
aws secretsmanager create-secret \
    --name db-credentials \
    --secret-string '{"username":"dbuser","password":"securepass"}'

# Retrieve in application
import boto3
client = boto3.client('secretsmanager')
response = client.get_secret_value(SecretId='db-credentials')
```

### 13.6 Enable AWS CloudTrail

```bash
# Enable CloudTrail for audit logging
aws cloudtrail create-trail \
    --name student-performance-trail \
    --s3-bucket-name your-audit-bucket

aws cloudtrail start-logging --name student-performance-trail
```

---

## Step 14: Scaling Considerations

### 14.1 Horizontal Scaling with Load Balancer

**Create Application Load Balancer**:

1. Create target group for EC2 instances
2. Register EC2 instances
3. Create load balancer with:
   - Listener on port 80/443
   - Target group with health checks on port 8080

**Auto Scaling Group**:

```bash
# Create launch template
aws ec2 create-launch-template \
    --launch-template-name student-performance-template \
    --image-id ami-xxxxx \
    --instance-type t3.medium \
    --user-data file://user-data.sh

# Create auto scaling group
aws autoscaling create-auto-scaling-group \
    --auto-scaling-group-name student-performance-asg \
    --launch-template LaunchTemplateName=student-performance-template \
    --min-size 2 \
    --max-size 10 \
    --desired-capacity 2 \
    --vpc-zone-identifier subnet-xxxxx,subnet-yyyyy
```

### 14.2 Container Orchestration with ECS

**Migrate to Amazon ECS for better orchestration**:

```yaml
# task-definition.json
{
    "family": "student-performance",
    "taskRoleArn": "arn:aws:iam::xxx:role/ecsTaskRole",
    "executionRoleArn": "arn:aws:iam::xxx:role/ecsTaskExecutionRole",
    "networkMode": "awsvpc",
    "containerDefinitions": [
        {
            "name": "student-performance",
            "image": "${ECR_REGISTRY}/${ECR_REPOSITORY}:latest",
            "memory": 512,
            "cpu": 256,
            "essential": true,
            "portMappings": [
                {
                    "containerPort": 8080,
                    "protocol": "tcp"
                }
            ]
        }
    ],
    "requiresCompatibilities": ["FARGATE"],
    "cpu": "256",
    "memory": "512"
}
```

### 14.3 Database Scaling

**Use Amazon RDS for database**:

```bash
# Create RDS PostgreSQL instance
aws rds create-db-instance \
    --db-instance-identifier student-performance-db \
    --db-instance-class db.t3.medium \
    --engine postgres \
    --allocated-storage 20 \
    --master-username dbadmin \
    --master-user-password SecurePass123
```

### 14.4 Monitoring and Alerting

**Set up CloudWatch Alarms**:

```bash
# CPU utilization alarm
aws cloudwatch put-metric-alarm \
    --alarm-name high-cpu-utilization \
    --alarm-description "CPU > 80% for 5 minutes" \
    --metric-name CPUUtilization \
    --namespace AWS/EC2 \
    --statistic Average \
    --period 300 \
    --threshold 80 \
    --comparison-operator GreaterThanThreshold \
    --dimensions Name=InstanceId,Value=i-xxxxx \
    --evaluation-periods 2 \
    --alarm-actions arn:aws:sns:region:account:topic
```

---

## Step 15: Monitoring and Logging

### 15.1 Centralized Logging with CloudWatch

**Install CloudWatch Agent on EC2**:

```bash
# Download CloudWatch agent
wget https://s3.amazonaws.com/amazoncloudwatch-agent/ubuntu/amd64/latest/amazon-cloudwatch-agent.deb
sudo dpkg -i -E ./amazon-cloudwatch-agent.deb

# Configure agent
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-config-wizard

# Start agent
sudo systemctl start amazon-cloudwatch-agent
```

### 15.2 Application Logging

```python
import logging
import boto3
import os

# Configure logging
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s',
    handlers=[
        logging.FileHandler('/var/log/app.log'),
        logging.StreamHandler()
    ]
)

logger = logging.getLogger(__name__)

# Example usage
@app.route('/predict', methods=['POST'])
def predict():
    logger.info(f"Received prediction request: {request.get_json()}")
    try:
        # Processing
        logger.info("Prediction successful")
    except Exception as e:
        logger.error(f"Prediction failed: {e}")
```

### 15.3 Container Logs

```bash
# View container logs
docker logs student-performance

# Follow logs in real-time
docker logs -f student-performance

# Send logs to CloudWatch
docker logs student-performance | aws logs put-log-events \
    --log-group-name /ecs/student-performance \
    --log-stream-name container-logs
```

### 15.4 Health Checks

**Create health check endpoint**:

```python
@app.route('/health')
def health():
    return jsonify({
        'status': 'healthy',
        'timestamp': datetime.utcnow().isoformat(),
        'version': os.environ.get('APP_VERSION', '1.0.0')
    })
```

**Configure health check in load balancer**:

```bash
aws elbv2 create-target-group \
    --name student-performance-tg \
    --protocol HTTP \
    --port 8080 \
    --health-check-path /health \
    --health-check-interval-seconds 30 \
    --healthy-threshold-count 2 \
    --unhealthy-threshold-count 3
```

### 15.5 Performance Monitoring

**Install Prometheus and Grafana**:

```bash
# Run Prometheus container
docker run -d \
    --name prometheus \
    -p 9090:9090 \
    -v /prometheus.yml:/etc/prometheus/prometheus.yml \
    prom/prometheus

# Run Grafana container
docker run -d \
    --name grafana \
    -p 3000:3000 \
    grafana/grafana
```

### 15.6 Cost Optimization Monitoring

**Set up AWS Budget Alerts**:

```bash
# Create budget
aws budgets create-budget \
    --account-id 123456789012 \
    --budget file://budget.json \
    --notifications-with-subscribers file://notifications.json
```

**budget.json**:
```json
{
    "BudgetName": "Monthly Cost Budget",
    "BudgetLimit": {
        "Amount": "100",
        "Unit": "USD"
    },
    "TimeUnit": "MONTHLY",
    "BudgetType": "COST"
}
```

---

## Conclusion

### Key Achievements

✅ **Dockerized Application**: Created a containerized ML application with proper Dockerfile
✅ **CI/CD Pipeline**: Implemented automated build and deployment with GitHub Actions
✅ **AWS Infrastructure**: Set up ECR for private image storage and EC2 for hosting
✅ **Self-Hosted Runner**: Configured GitHub Actions runner on EC2 for direct deployment
✅ **Security**: Implemented IAM best practices and security group rules
✅ **Monitoring**: Set up logging and health checks for production

### Best Practices Summary

| Area | Best Practice | Implementation |
|------|--------------|----------------|
| Security | Least privilege IAM | Custom policies, instance profiles |
| Reliability | Health checks | /health endpoint, container restart policy |
| Scalability | Horizontal scaling | Load balancer, auto-scaling groups |
| Monitoring | Centralized logging | CloudWatch, Docker logs |
| Cost | Cleanup automation | Scheduled termination, budget alerts |
| Versioning | Image tagging | Git SHA tags, latest tags |

### Next Steps

1. **Enhance Monitoring**: Integrate Prometheus and Grafana for detailed metrics
2. **Improve Security**: Add SSL/TLS certificates with Let's Encrypt
3. **Database Integration**: Connect to managed RDS PostgreSQL
4. **Blue-Green Deployment**: Implement zero-downtime deployments
5. **Infrastructure as Code**: Migrate to Terraform or CloudFormation
6. **Multi-Region**: Deploy to multiple regions for high availability
7. **Container Orchestration**: Migrate to ECS/EKS for better management

### Resources

- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [AWS ECR Documentation](https://docs.aws.amazon.com/ecr/)
- [AWS EC2 Documentation](https://docs.aws.amazon.com/ec2/)
- [Docker Documentation](https://docs.docker.com/)
- [PostgreSQL Documentation](https://www.postgresql.org/docs/)
- [Flask Documentation](https://flask.palletsprojects.com/)

---

## Appendix

### A. Useful Commands Reference

```bash
# Docker Commands
docker build -t <image>:<tag> .
docker images
docker run -d -p 8080:8080 <image>:<tag>
docker ps
docker logs <container>
docker exec -it <container> /bin/bash

# AWS ECR Commands
aws ecr get-login-password | docker login --username AWS --password-stdin <registry>
aws ecr create-repository --repository-name <name>
aws ecr list-images --repository-name <name>
aws ecr delete-repository --repository-name <name> --force

# EC2 Commands
aws ec2 describe-instances
aws ec2 start-instances --instance-ids <id>
aws ec2 stop-instances --instance-ids <id>
aws ec2 terminate-instances --instance-ids <id>

# GitHub CLI
gh repo view
gh run list
gh run watch
```

### B. Environment Variables Template

```bash
# .env.example
# AWS Configuration
AWS_REGION=us-east-1
AWS_ACCESS_KEY_ID=AKIAXXXXXXXX
AWS_SECRET_ACCESS_KEY=xxxxxxxxxxxx

# ECR Configuration
ECR_REGISTRY=123456789012.dkr.ecr.us-east-1.amazonaws.com
ECR_REPOSITORY=student-performance

# Application Configuration
APP_PORT=8080
APP_ENVIRONMENT=production
LOG_LEVEL=INFO

# Database Configuration (if applicable)
DB_HOST=localhost
DB_PORT=5432
DB_NAME=studentdb
DB_USER=postgres
DB_PASSWORD=securepassword
```

### C. Troubleshooting Quick Reference

| Issue | Quick Fix |
|-------|-----------|
| Container won't start | Check logs: `docker logs <container>` |
| Can't connect to port | Verify security group: port 8080 open |
| Docker build fails | Check Dockerfile syntax and dependencies |
| AWS credentials error | Run `aws configure` and verify |
| Runner offline | Restart runner service: `sudo ./svc.sh restart` |
| No space left | Clean Docker: `docker system prune -a` |

---
