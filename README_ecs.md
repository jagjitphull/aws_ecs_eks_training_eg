# AWS ECS Training Guide: Deploy a Containerized Web Application

## 📋 Table of Contents
- [Overview](#overview)
- [Prerequisites](#prerequisites)
- [Architecture Diagram](#architecture-diagram)
- [Step-by-Step Tutorial](#step-by-step-tutorial)
- [Troubleshooting Guide](#troubleshooting-guide)
- [Cleanup Instructions](#cleanup-instructions)
- [Key Concepts](#key-concepts)
- [Best Practices](#best-practices)
- [Additional Resources](#additional-resources)

---

## 💻 Command Shell Selection

This tutorial provides commands for both **Bash** (Mac/Linux) and **PowerShell** (Windows).

### **Which Should You Use?**

| Operating System | Recommended Shell | Notes |
|-----------------|-------------------|-------|
| **Windows** | PowerShell | Use PowerShell 5.1+ or PowerShell Core 7+ |
| **macOS** | Bash or Zsh | Both work identically for this tutorial |
| **Linux** | Bash | Default on most distributions |

### **Windows Users: Important Notes**

1. **Use PowerShell, not Command Prompt (cmd.exe)**
   - Command Prompt doesn't support multi-line commands well
   - PowerShell is pre-installed on Windows 10/11

2. **Check Your Docker Desktop**
   - Open Docker Desktop
   - Go to Settings → General
   - Ensure "Use WSL 2 based engine" is checked (recommended)
   - Make sure Docker is set to run **Linux containers** (not Windows containers)

3. **Running PowerShell**
   - Press `Win + X` and select "Windows PowerShell" or "Terminal"
   - Or search for "PowerShell" in the Start menu

### **Quick Command Translation Examples**

Throughout this guide, you'll see commands in both formats:

**Bash Example:**
```bash
export VPC_ID="vpc-12345"
aws ec2 describe-vpcs \
  --filters "Name=isDefault,Values=true"
echo $VPC_ID
```

**PowerShell Equivalent:**
```powershell
$VPC_ID = "vpc-12345"
aws ec2 describe-vpcs `
  --filters "Name=isDefault,Values=true"
Write-Host $VPC_ID
```

**Key Differences:**
- Bash uses `\` for line continuation; PowerShell uses `` ` `` (backtick)
- Bash uses `export VAR=value`; PowerShell uses `$VAR = "value"`
- Bash uses `echo`; PowerShell uses `Write-Host` or `echo`

---

## Overview

**What You'll Build:**
A simple Node.js "Hello World" web application deployed on AWS ECS using Fargate (serverless containers).

**What You'll Learn:**
- Containerizing applications with Docker
- Pushing images to Amazon ECR
- Creating ECS clusters and task definitions
- Deploying services with AWS Fargate
- Networking and security group configuration
- Troubleshooting common ECS issues

**Estimated Time:** 45-60 minutes

---

## Prerequisites

### Required Tools
- [ ] AWS Account with admin access
- [ ] AWS CLI installed and configured
- [ ] Docker Desktop installed
- [ ] Text editor (VS Code, Sublime, etc.)
- [ ] Terminal/Command Line access

### Verify Installation

**Bash (Mac/Linux):**
```bash
# Check AWS CLI
aws --version

# Check Docker
docker --version

# Verify AWS credentials
aws sts get-caller-identity
```

**PowerShell (Windows):**
```powershell
# Check AWS CLI
aws --version

# Check Docker
docker --version

# Verify AWS credentials
aws sts get-caller-identity
```

### Set Environment Variables (Optional but Recommended)

**Bash (Mac/Linux):**
```bash
# Replace with your AWS account ID
export AWS_ACCOUNT_ID=123456789012
export AWS_REGION=us-east-1

# Verify
echo $AWS_ACCOUNT_ID
```

**PowerShell (Windows):**
```powershell
# Replace with your AWS account ID
$env:AWS_ACCOUNT_ID="123456789012"
$env:AWS_REGION="us-east-1"

# Verify
echo $env:AWS_ACCOUNT_ID
```

### 💡 Command Syntax Guide

| Feature | Bash | PowerShell |
|---------|------|------------|
| **Variables** | `$VAR_NAME` | `$env:VAR_NAME` or `$VAR_NAME` |
| **Set Variable** | `export VAR=value` | `$VAR="value"` |
| **Line Continuation** | `\` | `` ` `` (backtick) |
| **Command Substitution** | `$(command)` | `$(command)` |
| **Pipe** | `\|` | `\|` |

---

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                        AWS Cloud                             │
│                                                              │
│  ┌────────────────────────────────────────────────────┐    │
│  │              Amazon ECR (Container Registry)        │    │
│  │              ecs-demo-app:latest                    │    │
│  └──────────────────┬─────────────────────────────────┘    │
│                     │                                        │
│                     │ Pull Image                             │
│                     ▼                                        │
│  ┌────────────────────────────────────────────────────┐    │
│  │         ECS Cluster (demo-cluster)                  │    │
│  │                                                     │    │
│  │  ┌──────────────────────────────────────────┐     │    │
│  │  │   ECS Service (demo-service)             │     │    │
│  │  │                                           │     │    │
│  │  │  ┌─────────────────────────────────┐    │     │    │
│  │  │  │  Fargate Task                    │    │     │    │
│  │  │  │  - Container: Node.js App        │    │     │    │
│  │  │  │  - Port: 3000                    │    │     │    │
│  │  │  │  - Public IP: X.X.X.X            │    │     │    │
│  │  │  └─────────────────────────────────┘    │     │    │
│  │  └──────────────────────────────────────────┘     │    │
│  └────────────────────────────────────────────────────┘    │
│                                                              │
│  ┌────────────────────────────────────────────────────┐    │
│  │         VPC & Networking                            │    │
│  │  - Security Group (port 3000)                       │    │
│  │  - Public Subnet                                    │    │
│  └────────────────────────────────────────────────────┘    │
│                                                              │
│  ┌────────────────────────────────────────────────────┐    │
│  │         CloudWatch Logs                             │    │
│  │         /ecs/demo-app                               │    │
│  └────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────┘
                           │
                           ▼
                    Internet User
                http://PUBLIC_IP:3000
```

---

## Step-by-Step Tutorial

### **STEP 1: Create the Application**

#### 1.1 Create Project Directory

**Bash (Mac/Linux):**
```bash
mkdir ecs-demo-app
cd ecs-demo-app
```

**PowerShell (Windows):**
```powershell
New-Item -ItemType Directory -Name ecs-demo-app
Set-Location ecs-demo-app
```

#### 1.2 Create `app.js`
```javascript
const express = require('express');
const app = express();
const PORT = 3000;

app.get('/', (req, res) => {
  res.send('Hello from AWS ECS! 🚀');
});

app.get('/health', (req, res) => {
  res.status(200).json({ status: 'healthy' });
});

app.listen(PORT, '0.0.0.0', () => {
  console.log(`Server running on port ${PORT}`);
});
```

#### 1.3 Create `package.json`
```json
{
  "name": "ecs-demo-app",
  "version": "1.0.0",
  "main": "app.js",
  "dependencies": {
    "express": "^4.18.2"
  },
  "scripts": {
    "start": "node app.js"
  }
}
```

#### 1.4 Verify Files

**Bash (Mac/Linux):**
```bash
ls -la
# Should see: app.js, package.json
```

**PowerShell (Windows):**
```powershell
Get-ChildItem
# Should see: app.js, package.json
```

---

### **STEP 2: Create Dockerfile**

Create a `Dockerfile` in the project root:

```dockerfile
FROM node:18-alpine

WORKDIR /app

COPY package.json ./
RUN npm install

COPY app.js ./

EXPOSE 3000

CMD ["node", "app.js"]
```

**Understanding the Dockerfile:**
- `FROM node:18-alpine`: Base image (lightweight Node.js)
- `WORKDIR /app`: Set working directory in container
- `COPY` and `RUN`: Install dependencies
- `EXPOSE 3000`: Document which port the app uses
- `CMD`: Command to run when container starts

---

### **STEP 3: Build and Test Locally**

#### 3.1 Build the Docker Image

**⚠️ IMPORTANT: Platform Specification**

If you're on **Apple Silicon Mac (M1/M2/M3)**, you MUST specify the platform:
```bash
docker build --platform linux/amd64 -t ecs-demo-app .
```

If you're on **Windows/Linux x86**, you can use:
```bash
docker build -t ecs-demo-app .
```

**Why?** AWS Fargate runs on AMD64/x86_64 architecture, not ARM64.

#### 3.2 Verify Image

**Bash (Mac/Linux):**
```bash
docker images | grep ecs-demo-app
```

**PowerShell (Windows):**
```powershell
docker images | Select-String "ecs-demo-app"
# Or simply
docker images ecs-demo-app
```

#### 3.3 Test Locally
```bash
docker run -p 3000:3000 ecs-demo-app
```

Open browser: `http://localhost:3000`

You should see: **"Hello from AWS ECS! 🚀"**

Stop the container: Press `Ctrl+C`

---

### **STEP 4: Push Image to Amazon ECR**

#### 4.1 Create ECR Repository
```bash
aws ecr create-repository \
  --repository-name ecs-demo-app \
  --region us-east-1
```

**Expected Output:**
```json
{
    "repository": {
        "repositoryArn": "arn:aws:ecr:us-east-1:123456789012:repository/ecs-demo-app",
        "registryId": "123456789012",
        "repositoryName": "ecs-demo-app",
        "repositoryUri": "123456789012.dkr.ecr.us-east-1.amazonaws.com/ecs-demo-app"
    }
}
```

📝 **Note the `repositoryUri` - you'll need it!**

#### 4.2 Authenticate Docker to ECR

**Bash (Mac/Linux):**
```bash
aws ecr get-login-password --region us-east-1 | \
docker login --username AWS --password-stdin \
<YOUR_AWS_ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com
```

**PowerShell (Windows):**
```powershell
(aws ecr get-login-password --region us-east-1) | docker login --username AWS --password-stdin <YOUR_AWS_ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com
```

Or using PowerShell variable:
```powershell
$password = aws ecr get-login-password --region us-east-1
$password | docker login --username AWS --password-stdin <YOUR_AWS_ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com
```

Replace `<YOUR_AWS_ACCOUNT_ID>` with your actual account ID.

**Success message:** `Login Succeeded`

#### 4.3 Tag Your Image
```bash
docker tag ecs-demo-app:latest \
<YOUR_AWS_ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/ecs-demo-app:latest
```

#### 4.4 Push to ECR
```bash
docker push \
<YOUR_AWS_ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/ecs-demo-app:latest
```

**This may take 1-2 minutes.**

#### 4.5 Verify Image in ECR
```bash
aws ecr describe-images \
  --repository-name ecs-demo-app \
  --region us-east-1
```

---

### **STEP 5: Create ECS Cluster**

```bash
aws ecs create-cluster \
  --cluster-name demo-cluster \
  --region us-east-1
```

**Expected Output:**
```json
{
    "cluster": {
        "clusterArn": "arn:aws:ecs:us-east-1:123456789012:cluster/demo-cluster",
        "clusterName": "demo-cluster",
        "status": "ACTIVE"
    }
}
```

#### Verify Cluster
```bash
aws ecs list-clusters --region us-east-1
```

---

### **STEP 6: Create Task Definition**

#### 6.1 Create CloudWatch Log Group First
```bash
aws logs create-log-group \
  --log-group-name /ecs/demo-app \
  --region us-east-1
```

#### 6.2 Create Task Execution Role (If Not Exists)

Check if the role exists:
```bash
aws iam get-role --role-name ecsTaskExecutionRole 2>/dev/null
```

If it doesn't exist, create it:
```bash
aws iam create-role \
  --role-name ecsTaskExecutionRole \
  --assume-role-policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Principal": {"Service": "ecs-tasks.amazonaws.com"},
      "Action": "sts:AssumeRole"
    }]
  }'

aws iam attach-role-policy \
  --role-name ecsTaskExecutionRole \
  --policy-arn arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy
```

#### 6.3 Create `task-definition.json`

Create this file in your project directory:

```json
{
  "family": "ecs-demo-task",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "256",
  "memory": "512",
  "containerDefinitions": [
    {
      "name": "ecs-demo-container",
      "image": "<YOUR_AWS_ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/ecs-demo-app:latest",
      "portMappings": [
        {
          "containerPort": 3000,
          "protocol": "tcp"
        }
      ],
      "essential": true,
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/demo-app",
          "awslogs-region": "us-east-1",
          "awslogs-stream-prefix": "ecs"
        }
      }
    }
  ],
  "executionRoleArn": "arn:aws:iam::<YOUR_AWS_ACCOUNT_ID>:role/ecsTaskExecutionRole"
}
```

**⚠️ Replace `<YOUR_AWS_ACCOUNT_ID>` in TWO places!**

#### 6.4 Register Task Definition
```bash
aws ecs register-task-definition \
  --cli-input-json file://task-definition.json \
  --region us-east-1
```

#### 6.5 Verify Task Definition
```bash
aws ecs list-task-definitions --region us-east-1
```

---

### **STEP 7: Configure Networking**

#### 7.1 Get Default VPC ID

**Bash (Mac/Linux):**
```bash
VPC_ID=$(aws ec2 describe-vpcs \
  --filters "Name=isDefault,Values=true" \
  --query "Vpcs[0].VpcId" \
  --output text)

echo "VPC ID: $VPC_ID"
```

**PowerShell (Windows):**
```powershell
$VPC_ID = aws ec2 describe-vpcs `
  --filters "Name=isDefault,Values=true" `
  --query "Vpcs[0].VpcId" `
  --output text

Write-Host "VPC ID: $VPC_ID"
```

#### 7.2 Create Security Group

**Bash (Mac/Linux):**
```bash
SG_ID=$(aws ec2 create-security-group \
  --group-name ecs-demo-sg \
  --description "Security group for ECS demo" \
  --vpc-id $VPC_ID \
  --query 'GroupId' \
  --output text)

echo "Security Group ID: $SG_ID"
```

**PowerShell (Windows):**
```powershell
$SG_ID = aws ec2 create-security-group `
  --group-name ecs-demo-sg `
  --description "Security group for ECS demo" `
  --vpc-id $VPC_ID `
  --query 'GroupId' `
  --output text

Write-Host "Security Group ID: $SG_ID"
```

**📝 Finding Security Group ID Later:**

**Bash:**
```bash
# If you need to find it again
aws ec2 describe-security-groups \
  --filters "Name=group-name,Values=ecs-demo-sg" \
  --query "SecurityGroups[0].GroupId" \
  --output text
```

**PowerShell:**
```powershell
# If you need to find it again
aws ec2 describe-security-groups `
  --filters "Name=group-name,Values=ecs-demo-sg" `
  --query "SecurityGroups[0].GroupId" `
  --output text
```

#### 7.3 Add Inbound Rule (Port 3000)
```bash
aws ec2 authorize-security-group-ingress \
  --group-id $SG_ID \
  --protocol tcp \
  --port 3000 \
  --cidr 0.0.0.0/0
```

**⚠️ Note:** `0.0.0.0/0` allows access from anywhere. For production, restrict this!

#### 7.4 Get Public Subnet ID

**Bash (Mac/Linux):**
```bash
SUBNET_ID=$(aws ec2 describe-subnets \
  --filters "Name=vpc-id,Values=$VPC_ID" \
  --query "Subnets[0].SubnetId" \
  --output text)

echo "Subnet ID: $SUBNET_ID"
```

**PowerShell (Windows):**
```powershell
$SUBNET_ID = aws ec2 describe-subnets `
  --filters "Name=vpc-id,Values=$VPC_ID" `
  --query "Subnets[0].SubnetId" `
  --output text

Write-Host "Subnet ID: $SUBNET_ID"
```

---

### **STEP 8: Create ECS Service**

#### 8.1 Create the Service

**Note:** We're using the `demo-cluster` that was created in Step 5!

**Bash (Mac/Linux):**
```bash
aws ecs create-service \
  --cluster demo-cluster \
  --service-name demo-service \
  --task-definition ecs-demo-task \
  --desired-count 1 \
  --launch-type FARGATE \
  --network-configuration "awsvpcConfiguration={subnets=[$SUBNET_ID],securityGroups=[$SG_ID],assignPublicIp=ENABLED}" \
  --region us-east-1
```

**PowerShell (Windows):**
```powershell
aws ecs create-service `
  --cluster demo-cluster `
  --service-name demo-service `
  --task-definition ecs-demo-task `
  --desired-count 1 `
  --launch-type FARGATE `
  --network-configuration "awsvpcConfiguration={subnets=[$SUBNET_ID],securityGroups=[$SG_ID],assignPublicIp=ENABLED}" `
  --region us-east-1
```

**This creates a service that maintains 1 running task.**

#### 8.2 Monitor Service Creation
```bash
aws ecs describe-services \
  --cluster demo-cluster \
  --services demo-service \
  --region us-east-1 \
  --query 'services[0].{Status:status,Running:runningCount,Desired:desiredCount}'
```

Wait until `runningCount` equals `desiredCount` (usually 1-2 minutes).

---

### **STEP 9: Access Your Application**

#### 9.1 Get Running Task ARN

**Bash (Mac/Linux):**
```bash
TASK_ARN=$(aws ecs list-tasks \
  --cluster demo-cluster \
  --service-name demo-service \
  --desired-status RUNNING \
  --region us-east-1 \
  --query 'taskArns[0]' \
  --output text)

echo "Task ARN: $TASK_ARN"
```

**PowerShell (Windows):**
```powershell
$TASK_ARN = aws ecs list-tasks `
  --cluster demo-cluster `
  --service-name demo-service `
  --desired-status RUNNING `
  --region us-east-1 `
  --query 'taskArns[0]' `
  --output text

Write-Host "Task ARN: $TASK_ARN"
```

#### 9.2 Get Task Details
```bash
aws ecs describe-tasks \
  --cluster demo-cluster \
  --tasks $TASK_ARN \
  --region us-east-1
```

#### 9.3 Extract Public IP

**Method 1: Multi-step approach (Works on all platforms)**

**Bash (Mac/Linux):**
```bash
# Get ENI ID
ENI_ID=$(aws ecs describe-tasks \
  --cluster demo-cluster \
  --tasks $TASK_ARN \
  --region us-east-1 \
  --query 'tasks[0].attachments[0].details[?name==`networkInterfaceId`].value' \
  --output text)

# Get Public IP from ENI
PUBLIC_IP=$(aws ec2 describe-network-interfaces \
  --network-interface-ids $ENI_ID \
  --query 'NetworkInterfaces[0].Association.PublicIp' \
  --output text)

echo "Public IP: $PUBLIC_IP"
```

**PowerShell (Windows):**
```powershell
# Get ENI ID
$ENI_ID = aws ecs describe-tasks `
  --cluster demo-cluster `
  --tasks $TASK_ARN `
  --region us-east-1 `
  --query 'tasks[0].attachments[0].details[?name==`networkInterfaceId`].value' `
  --output text

# Get Public IP from ENI
$PUBLIC_IP = aws ec2 describe-network-interfaces `
  --network-interface-ids $ENI_ID `
  --query 'NetworkInterfaces[0].Association.PublicIp' `
  --output text

Write-Host "Public IP: $PUBLIC_IP"
```

**Method 2: One-liner (Advanced)**

**Bash (Mac/Linux):**
```bash
PUBLIC_IP=$(aws ecs describe-tasks \
  --cluster demo-cluster \
  --tasks $TASK_ARN \
  --region us-east-1 \
  --query 'tasks[0].attachments[0].details[?name==`networkInterfaceId`].value' \
  --output text | xargs -I {} aws ec2 describe-network-interfaces \
  --network-interface-ids {} \
  --query 'NetworkInterfaces[0].Association.PublicIp' \
  --output text)

echo "Public IP: $PUBLIC_IP"
```

**PowerShell (Windows):**
```powershell
# PowerShell one-liner approach
$PUBLIC_IP = (aws ec2 describe-network-interfaces `
  --network-interface-ids (aws ecs describe-tasks `
    --cluster demo-cluster `
    --tasks $TASK_ARN `
    --region us-east-1 `
    --query 'tasks[0].attachments[0].details[?name==`networkInterfaceId`].value' `
    --output text) `
  --query 'NetworkInterfaces[0].Association.PublicIp' `
  --output text)

Write-Host "Public IP: $PUBLIC_IP"
```

#### 9.4 Test Your Application

**Bash (Mac/Linux):**
```bash
# Using curl
curl http://$PUBLIC_IP:3000

# Or open in browser
echo "Open this URL: http://$PUBLIC_IP:3000"
```

**PowerShell (Windows):**
```powershell
# Using Invoke-WebRequest
Invoke-WebRequest -Uri "http://${PUBLIC_IP}:3000"

# Or using curl alias (if available)
curl http://${PUBLIC_IP}:3000

# Open in default browser
Start-Process "http://${PUBLIC_IP}:3000"

# Display URL
Write-Host "Open this URL: http://${PUBLIC_IP}:3000"
```

**Expected Response:** `Hello from AWS ECS! 🚀`

Test health endpoint:

**Bash:**
```bash
curl http://$PUBLIC_IP:3000/health
```

**PowerShell:**
```powershell
Invoke-WebRequest -Uri "http://${PUBLIC_IP}:3000/health"
# Or
curl http://${PUBLIC_IP}:3000/health
```

---

### **STEP 10: View Logs**

```bash
aws logs tail /ecs/demo-app --follow
```

Press `Ctrl+C` to stop.

---

### **STEP 11: Update Application (Optional)**

#### 11.1 Modify Your Code
Edit `app.js` to change the message:
```javascript
app.get('/', (req, res) => {
  res.send('Hello from AWS ECS - Version 2! 🎉');
});
```

#### 11.2 Rebuild and Push
```bash
# Build with platform flag
docker build --platform linux/amd64 -t ecs-demo-app .

# Tag
docker tag ecs-demo-app:latest \
<YOUR_AWS_ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/ecs-demo-app:latest

# Push
docker push \
<YOUR_AWS_ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/ecs-demo-app:latest
```

#### 11.3 Force New Deployment
```bash
aws ecs update-service \
  --cluster demo-cluster \
  --service demo-service \
  --force-new-deployment \
  --region us-east-1
```

Wait 1-2 minutes, then refresh your browser!

---

## Troubleshooting Guide

### **Issue 1: Task Keeps Stopping - Platform Mismatch**

**Error:**
```
CannotPullContainerError: image Manifest does not contain descriptor matching platform 'linux/amd64'
```

**Solution:**
Rebuild with the correct platform:
```bash
docker build --platform linux/amd64 -t ecs-demo-app .
docker tag ecs-demo-app:latest <ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/ecs-demo-app:latest
docker push <ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/ecs-demo-app:latest
aws ecs update-service --cluster demo-cluster --service demo-service --force-new-deployment
```

---

### **Issue 2: Cannot Access Application**

**Check 1: Is task running?**
```bash
aws ecs list-tasks \
  --cluster demo-cluster \
  --service-name demo-service \
  --desired-status RUNNING
```

**Check 2: Security group rules**
```bash
aws ec2 describe-security-groups \
  --group-ids $SG_ID \
  --query 'SecurityGroups[0].IpPermissions'
```

Should show port 3000 open to 0.0.0.0/0.

**Check 3: Public IP assigned?**
```bash
aws ecs describe-tasks \
  --cluster demo-cluster \
  --tasks $TASK_ARN \
  --query 'tasks[0].attachments[0].details'
```

Look for `publicIPv4Address` or verify in network interface.

---

### **Issue 3: Task Fails to Start - ECR Pull Error**

**Error:**
```
CannotPullContainerError: AccessDeniedException
```

**Solution:**
Verify the task execution role has ECR permissions:
```bash
aws iam list-attached-role-policies --role-name ecsTaskExecutionRole
```

Should include `AmazonECSTaskExecutionRolePolicy`.

---

### **Issue 4: Out of Memory**

**Error in logs:**
```
Container killed due to memory usage
```

**Solution:**
Increase memory in task definition:
```json
"memory": "1024"
```

Re-register task definition and update service.

---

### **Issue 5: Can't Find Resources**

**Find your Security Group:**

**Bash:**
```bash
aws ec2 describe-security-groups \
  --filters "Name=group-name,Values=ecs-demo-sg" \
  --query "SecurityGroups[0].GroupId" \
  --output text
```

**PowerShell:**
```powershell
aws ec2 describe-security-groups `
  --filters "Name=group-name,Values=ecs-demo-sg" `
  --query "SecurityGroups[0].GroupId" `
  --output text
```

**Find your VPC:**

**Bash:**
```bash
aws ec2 describe-vpcs \
  --filters "Name=isDefault,Values=true" \
  --query "Vpcs[0].VpcId" \
  --output text
```

**PowerShell:**
```powershell
aws ec2 describe-vpcs `
  --filters "Name=isDefault,Values=true" `
  --query "Vpcs[0].VpcId" `
  --output text
```

**Find your Subnets:**

**Bash:**
```bash
aws ec2 describe-subnets \
  --filters "Name=vpc-id,Values=<VPC_ID>" \
  --query "Subnets[*].SubnetId" \
  --output table
```

**PowerShell:**
```powershell
aws ec2 describe-subnets `
  --filters "Name=vpc-id,Values=<VPC_ID>" `
  --query "Subnets[*].SubnetId" `
  --output table
```

---

### **Issue 6: Docker Command Not Found (Windows)**

**Error:**
```
docker: command not found
```

**Solution:**

1. Ensure Docker Desktop is installed and running
2. Check if Docker is in your PATH:
   ```powershell
   $env:PATH -split ';' | Select-String "Docker"
   ```

3. Restart PowerShell after installing Docker
4. Verify Docker is running:
   ```powershell
   docker --version
   docker ps
   ```

---

### **Issue 7: WSL 2 Not Enabled (Windows)**

**Error:**
```
Docker Desktop requires WSL 2
```

**Solution:**

1. Open PowerShell as Administrator
2. Enable WSL:
   ```powershell
   wsl --install
   ```
3. Restart your computer
4. Open Docker Desktop and verify WSL 2 backend is enabled

---

## Cleanup Instructions

**⚠️ IMPORTANT: Follow these steps to avoid charges!**

### Step 1: Scale Service to Zero
```bash
aws ecs update-service \
  --cluster demo-cluster \
  --service demo-service \
  --desired-count 0 \
  --region us-east-1
```

Wait 30 seconds for tasks to stop.

### Step 2: Delete Service
```bash
aws ecs delete-service \
  --cluster demo-cluster \
  --service demo-service \
  --region us-east-1
```

### Step 3: Delete Cluster
```bash
aws ecs delete-cluster \
  --cluster demo-cluster \
  --region us-east-1
```

### Step 4: Deregister Task Definition
```bash
aws ecs deregister-task-definition \
  --task-definition ecs-demo-task:1 \
  --region us-east-1
```

### Step 5: Delete ECR Repository
```bash
aws ecr delete-repository \
  --repository-name ecs-demo-app \
  --force \
  --region us-east-1
```

### Step 6: Delete CloudWatch Log Group
```bash
aws logs delete-log-group \
  --log-group-name /ecs/demo-app \
  --region us-east-1
```

### Step 7: Delete Security Group

**Bash (Mac/Linux):**
```bash
# Get SG ID if you don't have it
SG_ID=$(aws ec2 describe-security-groups \
  --filters "Name=group-name,Values=ecs-demo-sg" \
  --query "SecurityGroups[0].GroupId" \
  --output text)

# Delete
aws ec2 delete-security-group --group-id $SG_ID
```

**PowerShell (Windows):**
```powershell
# Get SG ID if you don't have it
$SG_ID = aws ec2 describe-security-groups `
  --filters "Name=group-name,Values=ecs-demo-sg" `
  --query "SecurityGroups[0].GroupId" `
  --output text

# Delete
aws ec2 delete-security-group --group-id $SG_ID
```

### Step 8: Verify Cleanup
```bash
# Check for remaining resources
aws ecs list-clusters
aws ecr describe-repositories
aws ec2 describe-security-groups --filters "Name=group-name,Values=ecs-demo-sg"
```

---

## Key Concepts

### **ECS Components**

| Component | Description | Analogy |
|-----------|-------------|---------|
| **Cluster** | Logical grouping of tasks/services | Factory building |
| **Task Definition** | Blueprint for your application | Recipe |
| **Task** | Running instance of task definition | Dish being cooked |
| **Service** | Maintains desired number of tasks | Restaurant kitchen manager |
| **Container** | Isolated runtime environment | Individual pot/pan |

### **Fargate vs EC2 Launch Type**

| Feature | Fargate | EC2 |
|---------|---------|-----|
| **Server Management** | Serverless | Manage EC2 instances |
| **Pricing** | Pay per task | Pay for EC2 instances |
| **Scaling** | Automatic | Configure Auto Scaling |
| **Best For** | Simplicity, variable workloads | Cost optimization, control |

### **CPU and Memory Options (Fargate)**

| CPU | Memory Options |
|-----|----------------|
| 0.25 vCPU | 512 MB, 1 GB, 2 GB |
| 0.5 vCPU | 1 GB to 4 GB (1 GB increments) |
| 1 vCPU | 2 GB to 8 GB (1 GB increments) |
| 2 vCPU | 4 GB to 16 GB (1 GB increments) |
| 4 vCPU | 8 GB to 30 GB (1 GB increments) |

---

## Best Practices

### **Security**
- ✅ Use least privilege IAM roles
- ✅ Restrict security group to specific IPs in production
- ✅ Enable encryption for ECR repositories
- ✅ Scan images for vulnerabilities
- ✅ Use secrets manager for sensitive data (not environment variables)

### **Performance**
- ✅ Right-size CPU and memory allocations
- ✅ Use health checks
- ✅ Implement graceful shutdown handling
- ✅ Use Application Load Balancer for production
- ✅ Enable CloudWatch Container Insights

### **Cost Optimization**
- ✅ Use Fargate Spot for fault-tolerant workloads
- ✅ Delete unused task definition revisions
- ✅ Set up log retention policies
- ✅ Use ARM-based Graviton processors when possible
- ✅ Monitor with AWS Cost Explorer

### **Development Workflow**
- ✅ Always specify `--platform linux/amd64` when building on Mac M1/M2/M3
- ✅ Test containers locally before pushing to ECR
- ✅ Use image tags (not just `latest`) for version control
- ✅ Automate deployments with CI/CD
- ✅ Use infrastructure as code (CloudFormation, Terraform)

---

## Additional Resources

### **AWS Documentation**
- [ECS Developer Guide](https://docs.aws.amazon.com/ecs/)
- [Fargate User Guide](https://docs.aws.amazon.com/AmazonECS/latest/userguide/what-is-fargate.html)
- [ECR User Guide](https://docs.aws.amazon.com/ecr/)
- [ECS Best Practices Guide](https://docs.aws.amazon.com/AmazonECS/latest/bestpracticesguide/)

### **Tutorials**
- [ECS Workshop](https://ecsworkshop.com/)
- [AWS Containers Roadmap](https://github.com/aws/containers-roadmap)

### **Tools**
- [AWS Copilot CLI](https://aws.github.io/copilot-cli/) - Simplified ECS deployments
- [eksctl](https://eksctl.io/) - For Kubernetes on AWS
- [Docker Compose ECS Integration](https://docs.docker.com/cloud/ecs-integration/)

### **Community**
- [AWS Containers Blog](https://aws.amazon.com/blogs/containers/)
- [r/aws on Reddit](https://reddit.com/r/aws)
- [AWS re:Post](https://repost.aws/)

---

## Quick Reference Commands

### **Check Service Status**

**Bash (Mac/Linux):**
```bash
aws ecs describe-services \
  --cluster demo-cluster \
  --services demo-service \
  --query 'services[0].{Status:status,Running:runningCount,Desired:desiredCount,Events:events[0:3]}'
```

**PowerShell (Windows):**
```powershell
aws ecs describe-services `
  --cluster demo-cluster `
  --services demo-service `
  --query 'services[0].{Status:status,Running:runningCount,Desired:desiredCount,Events:events[0:3]}'
```

### **View Recent Service Events**

**Bash:**
```bash
aws ecs describe-services \
  --cluster demo-cluster \
  --services demo-service \
  --query 'services[0].events[0:5]' \
  --output table
```

**PowerShell:**
```powershell
aws ecs describe-services `
  --cluster demo-cluster `
  --services demo-service `
  --query 'services[0].events[0:5]' `
  --output table
```

### **Scale Service**

**Both platforms (same command):**
```bash
aws ecs update-service \
  --cluster demo-cluster \
  --service demo-service \
  --desired-count 3
```

### **Get Public IP of Running Task**

**Bash (Mac/Linux):**
```bash
TASK_ARN=$(aws ecs list-tasks --cluster demo-cluster --service-name demo-service --desired-status RUNNING --query 'taskArns[0]' --output text)
ENI_ID=$(aws ecs describe-tasks --cluster demo-cluster --tasks $TASK_ARN --query 'tasks[0].attachments[0].details[?name==`networkInterfaceId`].value' --output text)
aws ec2 describe-network-interfaces --network-interface-ids $ENI_ID --query 'NetworkInterfaces[0].Association.PublicIp' --output text
```

**PowerShell (Windows):**
```powershell
$TASK_ARN = aws ecs list-tasks --cluster demo-cluster --service-name demo-service --desired-status RUNNING --query 'taskArns[0]' --output text
$ENI_ID = aws ecs describe-tasks --cluster demo-cluster --tasks $TASK_ARN --query 'tasks[0].attachments[0].details[?name==`networkInterfaceId`].value' --output text
aws ec2 describe-network-interfaces --network-interface-ids $ENI_ID --query 'NetworkInterfaces[0].Association.PublicIp' --output text
```

### **Tail CloudWatch Logs**

**Both platforms (same command):**
```bash
aws logs tail /ecs/demo-app --follow --format short
```

---

## Training Exercise Checklist

- [ ] Created and tested Node.js application locally
- [ ] Built Docker image with correct platform
- [ ] Pushed image to Amazon ECR
- [ ] Created ECS cluster
- [ ] Configured networking (VPC, subnet, security group)
- [ ] Created and registered task definition
- [ ] Deployed ECS service
- [ ] Accessed application via public IP
- [ ] Viewed logs in CloudWatch
- [ ] Updated application and redeployed
- [ ] Cleaned up all resources

---

## Troubleshooting Checklist

When things go wrong, check in this order:

1. ✅ **Task Status**: Is it running or stopped?
   
   **Bash:**
   ```bash
   aws ecs describe-tasks --cluster demo-cluster --tasks $TASK_ARN
   ```
   
   **PowerShell:**
   ```powershell
   aws ecs describe-tasks --cluster demo-cluster --tasks $TASK_ARN
   ```

2. ✅ **CloudWatch Logs**: What do the logs say?
   
   **Both platforms:**
   ```bash
   aws logs tail /ecs/demo-app --follow
   ```

3. ✅ **Security Group**: Is port 3000 open?
   
   **Bash:**
   ```bash
   aws ec2 describe-security-groups --group-ids $SG_ID
   ```
   
   **PowerShell:**
   ```powershell
   aws ec2 describe-security-groups --group-ids $SG_ID
   ```

4. ✅ **Public IP**: Does the task have a public IP?
   
   **Bash:**
   ```bash
   aws ecs describe-tasks --cluster demo-cluster --tasks $TASK_ARN | grep -i public
   ```
   
   **PowerShell:**
   ```powershell
   aws ecs describe-tasks --cluster demo-cluster --tasks $TASK_ARN | Select-String -Pattern "public" -CaseSensitive:$false
   ```

5. ✅ **Image Platform**: Was it built for linux/amd64?
   
   **Bash:**
   ```bash
   docker image inspect <IMAGE_NAME> | grep Architecture
   ```
   
   **PowerShell:**
   ```powershell
   docker image inspect <IMAGE_NAME> | Select-String "Architecture"
   ```

---

## PowerShell-Specific Tips for Windows Users

### **Line Continuation**
PowerShell uses backtick (`` ` ``) for line continuation, not backslash (`\`):

**Bash:**
```bash
aws ec2 describe-vpcs \
  --filters "Name=isDefault,Values=true" \
  --query "Vpcs[0].VpcId"
```

**PowerShell:**
```powershell
aws ec2 describe-vpcs `
  --filters "Name=isDefault,Values=true" `
  --query "Vpcs[0].VpcId"
```

### **Variables**
Use `$` prefix for variables (no `export` needed):

**Bash:**
```bash
export VPC_ID="vpc-12345"
echo $VPC_ID
```

**PowerShell:**
```powershell
$VPC_ID = "vpc-12345"
Write-Host $VPC_ID
# Or
echo $VPC_ID
```

### **Environment Variables**
For environment variables, use `$env:` prefix:

**Bash:**
```bash
export AWS_REGION=us-east-1
echo $AWS_REGION
```

**PowerShell:**
```powershell
$env:AWS_REGION = "us-east-1"
echo $env:AWS_REGION
```

### **String Interpolation**
PowerShell supports variable interpolation in double quotes:

**Bash:**
```bash
echo "VPC ID: $VPC_ID"
```

**PowerShell:**
```powershell
Write-Host "VPC ID: $VPC_ID"
# Or use string interpolation
"VPC ID: ${VPC_ID}"
```

### **Grep Alternative**
Use `Select-String` instead of `grep`:

**Bash:**
```bash
docker images | grep ecs-demo-app
aws ecs describe-tasks --cluster demo-cluster --tasks $TASK_ARN | grep -i public
```

**PowerShell:**
```powershell
docker images | Select-String "ecs-demo-app"
aws ecs describe-tasks --cluster demo-cluster --tasks $TASK_ARN | Select-String -Pattern "public" -CaseSensitive:$false
```

### **Creating Directories**
**Bash:**
```bash
mkdir ecs-demo-app
```

**PowerShell:**
```powershell
New-Item -ItemType Directory -Name ecs-demo-app
# Or use the mkdir alias
mkdir ecs-demo-app
```

### **Listing Files**
**Bash:**
```bash
ls -la
```

**PowerShell:**
```powershell
Get-ChildItem
# Or use the alias
ls
# Or for detailed view
dir
```

### **JSON Handling**
PowerShell has excellent built-in JSON support:

```powershell
# Parse JSON output
$result = aws ecs describe-services --cluster demo-cluster --services demo-service | ConvertFrom-Json
$result.services[0].status

# Create JSON for task definition
$taskDef = @{
    family = "ecs-demo-task"
    cpu = "256"
    memory = "512"
} | ConvertTo-Json
```

### **Running Docker Commands**
Docker commands work identically in PowerShell:

```powershell
docker build --platform linux/amd64 -t ecs-demo-app .
docker images
docker run -p 3000:3000 ecs-demo-app
```

### **AWS CLI Output**
AWS CLI commands work the same way, but capture output like this:

```powershell
# Capture output to variable
$output = aws ecr describe-repositories --repository-name ecs-demo-app

# Or pipe to ConvertFrom-Json for object manipulation
$repos = aws ecr describe-repositories | ConvertFrom-Json
$repos.repositories[0].repositoryUri
```

### **Opening URLs in Browser**
```powershell
# Open in default browser
Start-Process "http://${PUBLIC_IP}:3000"
```

### **PowerShell Execution Policy**
If you encounter script execution errors:

```powershell
# Check current policy
Get-ExecutionPolicy

# Set policy (run as Administrator if needed)
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser
```

---

## Notes for Trainers

### **Common Student Mistakes**
1. Forgetting to specify `--platform linux/amd64` on M1/M2 Macs
2. Not replacing `<YOUR_AWS_ACCOUNT_ID>` in task definition
3. Forgetting to create log group before registering task definition
4. Not waiting for service to stabilize before checking public IP
5. Forgetting cleanup steps and incurring charges
6. **PowerShell-specific**: Using backslash (`\`) instead of backtick (`` ` ``) for line continuation
7. **PowerShell-specific**: Forgetting `$env:` prefix for environment variables
8. **PowerShell-specific**: Using `grep` instead of `Select-String`
9. **Windows-specific**: Docker Desktop not running or not configured for Linux containers

### **Time Estimates**
- Setup and prerequisites: 10 minutes
- Steps 1-4 (App creation and ECR): 15 minutes
- Steps 5-8 (ECS setup): 15 minutes
- Steps 9-11 (Testing and updates): 10 minutes
- Troubleshooting: 10 minutes
- Cleanup: 5 minutes

### **Extension Ideas**
- Add Application Load Balancer
- Implement auto-scaling
- Deploy multi-container application
- Set up CI/CD pipeline
- Add RDS database connection
- Implement blue/green deployment

---

**Last Updated:** January 2026  
**Version:** 1.0  
**Maintained by:** Training Team

For questions or issues, please contact your instructor or open an issue in the training repository.

---

**Happy Learning! 🚀**
