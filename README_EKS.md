# AWS EKS Training Guide: Step-by-Step Tutorial

## Table of Contents
1. [Overview](#overview)
2. [Prerequisites](#prerequisites)
3. [Step 1: Setup AWS CLI and Tools](#step-1-setup-aws-cli-and-tools)
4. [Step 2: Create IAM Role for EKS](#step-2-create-iam-role-for-eks)
5. [Step 3: Create EKS Cluster](#step-3-create-eks-cluster)
6. [Step 4: Configure kubectl](#step-4-configure-kubectl)
7. [Step 5: Create Node Group](#step-5-create-node-group)
8. [Step 6: Deploy Sample Application](#step-6-deploy-sample-application)
9. [Step 7: Expose Application](#step-7-expose-application)
10. [Step 8: Access and Test](#step-8-access-and-test)
11. [Step 9: Cleanup](#step-9-cleanup)
12. [Troubleshooting](#troubleshooting)

---

## Overview

**What is Amazon EKS?**
Amazon Elastic Kubernetes Service (EKS) is a managed Kubernetes service that makes it easy to run Kubernetes on AWS without needing to install and operate your own Kubernetes control plane.

**Use Case**: Deploy a simple NGINX web server on EKS to understand the basic workflow.

**Estimated Time**: 45-60 minutes

---

## Prerequisites

Before starting, ensure you have:

- **AWS Account** with appropriate permissions
- **AWS CLI** installed (version 2.x recommended)
- **kubectl** installed (Kubernetes command-line tool)
- **eksctl** installed (optional but recommended)
- **Basic knowledge** of:
  - AWS services (EC2, VPC, IAM)
  - Kubernetes concepts (Pods, Services, Deployments)
  - Command-line interface

**Cost Warning**: Running this tutorial will incur AWS charges. Make sure to complete the cleanup step.

---

## Step 1: Setup AWS CLI and Tools

### 1.1 Install AWS CLI

**For Linux/macOS:**
```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
```

**For Windows:**
Download and run the AWS CLI MSI installer from the official AWS website.

### 1.2 Configure AWS CLI

```bash
aws configure
```

Enter your credentials:
- AWS Access Key ID
- AWS Secret Access Key
- Default region (e.g., `us-east-1`)
- Default output format (e.g., `json`)

**Verify configuration:**
```bash
aws sts get-caller-identity
```

### 1.3 Install kubectl

**For Linux:**
```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
```

**For macOS:**
```bash
brew install kubectl
```

**For Windows:**
```powershell
choco install kubernetes-cli
```

**Verify installation:**
```bash
kubectl version --client
```

### 1.4 Install eksctl (Recommended)

**For Linux/macOS:**
```bash
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin
```

**For Windows:**
```powershell
choco install eksctl
```

**Verify installation:**
```bash
eksctl version
```

---

## Step 2: Create IAM Role for EKS

### 2.1 Create EKS Cluster Role

Create a file named `eks-cluster-role-trust-policy.json`:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "eks.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

**Create the IAM role:**
```bash
aws iam create-role \
  --role-name myEKSClusterRole \
  --assume-role-policy-document file://eks-cluster-role-trust-policy.json
```

**Attach required policies:**
```bash
aws iam attach-role-policy \
  --role-name myEKSClusterRole \
  --policy-arn arn:aws:iam::aws:policy/AmazonEKSClusterPolicy
```

### 2.2 Create Node Group Role

Create a file named `node-role-trust-policy.json`:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "ec2.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

**Create the IAM role:**
```bash
aws iam create-role \
  --role-name myEKSNodeRole \
  --assume-role-policy-document file://node-role-trust-policy.json
```

**Attach required policies:**
```bash
aws iam attach-role-policy \
  --role-name myEKSNodeRole \
  --policy-arn arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy

aws iam attach-role-policy \
  --role-name myEKSNodeRole \
  --policy-arn arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy

aws iam attach-role-policy \
  --role-name myEKSNodeRole \
  --policy-arn arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly
```

---

## Step 3: Create EKS Cluster

### Method 1: Using eksctl (Recommended for Beginners)

This is the simplest method:

```bash
eksctl create cluster \
  --name my-training-cluster \
  --region us-east-1 \
  --nodegroup-name my-nodes \
  --node-type t3.medium \
  --nodes 2 \
  --nodes-min 1 \
  --nodes-max 3 \
  --managed
```

**This single command will:**
- Create a VPC with public and private subnets
- Create an EKS cluster
- Create a managed node group
- Configure kubectl automatically

**Wait time**: 15-20 minutes

Skip to Step 6 if using this method.

### Method 2: Using AWS Console (Step-by-Step)

#### 3.1 Create VPC (if you don't have one)

You can use the default VPC or create a new one specifically for EKS.

**Using AWS CloudFormation for EKS VPC:**
```bash
aws cloudformation create-stack \
  --stack-name eks-vpc-stack \
  --template-url https://s3.us-west-2.amazonaws.com/amazon-eks/cloudformation/2020-10-29/amazon-eks-vpc-private-subnets.yaml
```

**Wait for stack creation:**
```bash
aws cloudformation wait stack-create-complete --stack-name eks-vpc-stack
```

**Get VPC details:**
```bash
aws cloudformation describe-stacks --stack-name eks-vpc-stack --query "Stacks[0].Outputs"
```

Note the following values:
- VpcId
- SubnetIds (you'll need at least 2)
- SecurityGroups

#### 3.2 Create EKS Cluster

```bash
aws eks create-cluster \
  --name my-training-cluster \
  --role-arn arn:aws:iam::YOUR_ACCOUNT_ID:role/myEKSClusterRole \
  --resources-vpc-config subnetIds=subnet-xxxxx,subnet-yyyyy,securityGroupIds=sg-xxxxx \
  --region us-east-1
```

**Replace**:
- `YOUR_ACCOUNT_ID` with your AWS account ID
- `subnet-xxxxx,subnet-yyyyy` with your subnet IDs
- `sg-xxxxx` with your security group ID

**Check cluster status:**
```bash
aws eks describe-cluster --name my-training-cluster --query "cluster.status"
```

Wait until status is `ACTIVE` (takes 10-15 minutes).

---

## Step 4: Configure kubectl

### 4.1 Update kubeconfig

```bash
aws eks update-kubeconfig \
  --region us-east-1 \
  --name my-training-cluster
```

### 4.2 Verify Connection

```bash
kubectl get svc
```

You should see the Kubernetes API server:
```
NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   10.100.0.1   <none>        443/TCP   5m
```

---

## Step 5: Create Node Group

### 5.1 Create Node Group (if not using eksctl)

```bash
aws eks create-nodegroup \
  --cluster-name my-training-cluster \
  --nodegroup-name my-nodegroup \
  --scaling-config minSize=1,maxSize=3,desiredSize=2 \
  --subnets subnet-xxxxx subnet-yyyyy \
  --instance-types t3.medium \
  --node-role arn:aws:iam::YOUR_ACCOUNT_ID:role/myEKSNodeRole \
  --region us-east-1
```

**Check node group status:**
```bash
aws eks describe-nodegroup \
  --cluster-name my-training-cluster \
  --nodegroup-name my-nodegroup \
  --query "nodegroup.status"
```

Wait until status is `ACTIVE` (takes 5-10 minutes).

### 5.2 Verify Nodes

```bash
kubectl get nodes
```

You should see your worker nodes:
```
NAME                                           STATUS   ROLES    AGE   VERSION
ip-192-168-10-100.us-east-1.compute.internal   Ready    <none>   2m    v1.28.x
ip-192-168-20-200.us-east-1.compute.internal   Ready    <none>   2m    v1.28.x
```

---

## Step 6: Deploy Sample Application

### 6.1 Create Deployment Manifest

Create a file named `nginx-deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
        resources:
          requests:
            memory: "64Mi"
            cpu: "250m"
          limits:
            memory: "128Mi"
            cpu: "500m"
```

### 6.2 Apply Deployment

```bash
kubectl apply -f nginx-deployment.yaml
```

### 6.3 Verify Deployment

```bash
kubectl get deployments
```

Output:
```
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   3/3     3            3           1m
```

**Check pods:**
```bash
kubectl get pods
```

Output:
```
NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-5d59d67564-7xm2w   1/1     Running   0          1m
nginx-deployment-5d59d67564-9k8pz   1/1     Running   0          1m
nginx-deployment-5d59d67564-xq4rt   1/1     Running   0          1m
```

---

## Step 7: Expose Application

### 7.1 Create Service Manifest

Create a file named `nginx-service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  type: LoadBalancer
  selector:
    app: nginx
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
```

### 7.2 Apply Service

```bash
kubectl apply -f nginx-service.yaml
```

### 7.3 Get Service Details

```bash
kubectl get service nginx-service
```

Wait for the `EXTERNAL-IP` to be assigned (takes 2-3 minutes):
```
NAME            TYPE           CLUSTER-IP       EXTERNAL-IP                                                              PORT(S)        AGE
nginx-service   LoadBalancer   10.100.123.456   a1b2c3d4e5f6g7h8.us-east-1.elb.amazonaws.com   80:31234/TCP   2m
```

---

## Step 8: Access and Test

### 8.1 Get the Load Balancer URL

```bash
kubectl get service nginx-service -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'
```

### 8.2 Test the Application

**Using curl:**
```bash
curl http://YOUR_LOAD_BALANCER_URL
```

**Or open in a browser:**
```
http://YOUR_LOAD_BALANCER_URL
```

You should see the default NGINX welcome page!

### 8.3 Additional Verification Commands

**View pod logs:**
```bash
kubectl logs -l app=nginx
```

**Describe a pod:**
```bash
kubectl describe pod <pod-name>
```

**Get detailed service information:**
```bash
kubectl describe service nginx-service
```

**View all resources:**
```bash
kubectl get all
```

---

## Step 9: Cleanup

**Important**: To avoid ongoing charges, clean up all resources.

### 9.1 Delete Service

```bash
kubectl delete service nginx-service
```

Wait for the Load Balancer to be deleted (check AWS Console).

### 9.2 Delete Deployment

```bash
kubectl delete deployment nginx-deployment
```

### 9.3 Delete Node Group

**If using eksctl:**
```bash
eksctl delete nodegroup \
  --cluster=my-training-cluster \
  --name=my-nodes
```

**If using AWS CLI:**
```bash
aws eks delete-nodegroup \
  --cluster-name my-training-cluster \
  --nodegroup-name my-nodegroup
```

Wait for deletion to complete.

### 9.4 Delete EKS Cluster

**If using eksctl:**
```bash
eksctl delete cluster --name my-training-cluster
```

**If using AWS CLI:**
```bash
aws eks delete-cluster --name my-training-cluster
```

### 9.5 Delete VPC Stack (if created)

```bash
aws cloudformation delete-stack --stack-name eks-vpc-stack
```

### 9.6 Delete IAM Roles

```bash
# Detach policies
aws iam detach-role-policy \
  --role-name myEKSClusterRole \
  --policy-arn arn:aws:iam::aws:policy/AmazonEKSClusterPolicy

aws iam detach-role-policy \
  --role-name myEKSNodeRole \
  --policy-arn arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy

aws iam detach-role-policy \
  --role-name myEKSNodeRole \
  --policy-arn arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy

aws iam detach-role-policy \
  --role-name myEKSNodeRole \
  --policy-arn arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly

# Delete roles
aws iam delete-role --role-name myEKSClusterRole
aws iam delete-role --role-name myEKSNodeRole
```

---

## Troubleshooting

### Issue 1: Nodes Not Joining Cluster

**Check:**
```bash
kubectl get nodes
aws eks describe-nodegroup --cluster-name my-training-cluster --nodegroup-name my-nodegroup
```

**Solution**: Verify the node IAM role has the correct policies attached.

### Issue 2: Pods in Pending State

**Check:**
```bash
kubectl describe pod <pod-name>
```

**Common causes**:
- Insufficient resources
- Node selector mismatch
- Image pull errors

**Solution**: Check node capacity and pod resource requests.

### Issue 3: Cannot Connect to Cluster

**Check kubeconfig:**
```bash
kubectl config current-context
```

**Update kubeconfig:**
```bash
aws eks update-kubeconfig --name my-training-cluster --region us-east-1
```

### Issue 4: Load Balancer Not Created

**Check service:**
```bash
kubectl describe service nginx-service
```

**Check AWS Load Balancer Controller:**
```bash
kubectl get pods -n kube-system
```

**Solution**: Ensure your VPC has appropriate tags and internet gateway configured.

### Useful Commands for Debugging

```bash
# Get cluster information
kubectl cluster-info

# Get all resources in all namespaces
kubectl get all --all-namespaces

# View events
kubectl get events --sort-by=.metadata.creationTimestamp

# Check cluster endpoint
aws eks describe-cluster --name my-training-cluster --query "cluster.endpoint"

# View CloudWatch logs (if configured)
aws logs tail /aws/eks/my-training-cluster/cluster --follow
```

---

## Next Steps

After completing this tutorial, you can explore:

1. **Deploy more complex applications** (multi-tier apps, databases)
2. **Implement Horizontal Pod Autoscaling** (HPA)
3. **Set up Cluster Autoscaler** for automatic node scaling
4. **Configure Ingress** using AWS Load Balancer Controller
5. **Implement monitoring** with Prometheus and Grafana
6. **Set up CI/CD pipelines** with AWS CodePipeline
7. **Implement security best practices** (RBAC, Network Policies)
8. **Use Helm** for package management
9. **Configure persistent storage** with EBS or EFS
10. **Implement logging** with Fluent Bit and CloudWatch

---

## Additional Resources

- **AWS EKS Documentation**: https://docs.aws.amazon.com/eks/
- **Kubernetes Documentation**: https://kubernetes.io/docs/
- **eksctl Documentation**: https://eksctl.io/
- **AWS EKS Best Practices Guide**: https://aws.github.io/aws-eks-best-practices/
- **kubectl Cheat Sheet**: https://kubernetes.io/docs/reference/kubectl/cheatsheet/

---

## Summary

You've successfully:
✅ Set up AWS CLI and Kubernetes tools
✅ Created IAM roles for EKS
✅ Launched an EKS cluster
✅ Deployed worker nodes
✅ Deployed an NGINX application
✅ Exposed the application via LoadBalancer
✅ Tested the deployment
✅ Cleaned up resources

**Congratulations on completing your first AWS EKS deployment!**
