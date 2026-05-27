# 3-Tier User Platform Project Setup Guide

This guide explains how to set up an Amazon EKS cluster, configure GitHub OIDC authentication, and deploy AWS Load Balancer Controller for a production-ready Kubernetes environment on AWS. 

---

# Prerequisites

Before starting, make sure you have:

* An AWS account
* AWS CLI configured
* IAM permissions for EKS, IAM, EC2, and VPC
* Ubuntu/Linux system (recommended)

---

# 1. Install AWS CLI

Download and install AWS CLI:

```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
```

Verify installation:

```bash
aws --version
```

Configure AWS credentials:

```bash
aws configure
```

---

# 2. Install eksctl

Download latest eksctl binary:

```bash
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" -o eksctl.tar.gz
```

Extract the archive:

```bash
tar -xvzf eksctl.tar.gz
```

Move binary to system path:

```bash
sudo mv eksctl /bin
```

Verify installation:

```bash
eksctl version
```

---

# 3. Install kubectl

Install kubectl using official Kubernetes documentation.

Verify installation:

```bash
kubectl version --client
```

---

# 4. Install Helm

Download Helm installation script:

```bash
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-4
```

Give execute permission:

```bash
chmod 700 get_helm.sh
```

Install Helm:

```bash
./get_helm.sh
```

Verify installation:

```bash
helm version
```

---

# Verify All Installed Tools

Run the following commands one by one:

```bash
aws --version
eksctl version
kubectl version --client
helm version
```

All commands should work successfully.

---

# Create Amazon EKS Cluster

## Step 1 — Create EKS Cluster Without Node Group

```bash
eksctl create cluster \
  --name <cluster-name> \
  --region <region-name> \
  --without-nodegroup
```

---

## Step 2 — Verify Cluster Creation

```bash
eksctl get clusters --region <region-name>
```

---

## Step 3 — Associate IAM OIDC Provider

```bash
eksctl utils associate-iam-oidc-provider \
  --region=<region-name> \
  --cluster=<cluster-name> \
  --approve
```

---

## Step 4 — Create Managed Node Group

```bash
eksctl create nodegroup \
  --cluster=<cluster-name> \
  --region=<region-name> \
  --name=<nodegroup-name> \
  --node-type=t3.medium \
  --nodes=2 \
  --nodes-min=1 \
  --nodes-max=3 \
  --node-volume-size=20 \
  --managed
```

---

## Step 5 — Configure kubectl Access

### Update kubeconfig

```bash
aws eks update-kubeconfig \
  --region <region-name> \
  --name <cluster-name>
```

### Verify Worker Nodes

```bash
kubectl get nodes
```

---

## Delete EKS Cluster (Optional)

```bash
eksctl delete cluster \
  --name <cluster-name> \
  --region <region-name>
```

---

# Verify AWS & EKS Prerequisites

## Check AWS Identity

```bash
aws sts get-caller-identity
```

Expected output:

```json
{
  "UserId": "AIDAXXXXXXXXXXXXXXX",
  "Account": "585593375370",
  "Arn": "arn:aws:iam::585593375370:user/your-admin-user"
}
```

---

## Verify EKS Cluster Status

```bash
aws eks describe-cluster \
  --name my-cluster \
  --region ap-south-1 \
  --query "{status:cluster.status,authMode:cluster.accessConfig.authenticationMode}" \
  --output json
```

Expected output:

```json
{
  "status": "ACTIVE",
  "authMode": "API_AND_CONFIG_MAP"
}
```

---

# Configure GitHub OIDC Authentication

This section allows GitHub Actions to securely access AWS without storing AWS access keys.

---

# Step 1 — Create GitHub OIDC Provider

## Check Existing OIDC Provider

```bash
aws iam list-open-id-connect-providers \
  --query "OpenIDConnectProviderList[?contains(Arn,'token.actions.githubusercontent.com')].Arn" \
  --output text
```

---

## Delete Existing Provider (Optional)

```bash
aws iam delete-open-id-connect-provider \
  --open-id-connect-provider-arn arn:aws:iam::<account-id>:oidc-provider/token.actions.githubusercontent.com
```

---

## Create New OIDC Provider

```bash
aws iam create-open-id-connect-provider \
  --url https://token.actions.githubusercontent.com \
  --client-id-list sts.amazonaws.com \
  --thumbprint-list 6938fd4d98bab03faadb97b34396831e3780aea1
```

---

# Step 2 — Create Trust Policy File

Create file:

```bash
cat <<EOF > github-oidc-trust.json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::<account-id>:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
        },
        "StringLike": {
          "token.actions.githubusercontent.com:sub": "repo:<github-username>/<repo-name>:ref:refs/heads/main"
        }
      }
    }
  ]
}
EOF
```

---

# Step 3 — Create IAM Role

```bash
aws iam create-role \
  --role-name GitHubActionsEKSDeployRole \
  --assume-role-policy-document file://github-oidc-trust.json
```

Verify role:

```bash
aws iam get-role \
  --role-name GitHubActionsEKSDeployRole
```

---

# Step 4 — Create IAM Policy

Create policy file:

```bash
cat <<EOF > eks-describe-cluster-policy.json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "EKSDescribeCluster",
    "Effect": "Allow",
    "Action": ["eks:DescribeCluster"],
    "Resource": "arn:aws:eks:ap-south-1:<account-id>:cluster/my-cluster"
  }]
}
EOF
```

---

## Create IAM Policy

```bash
aws iam create-policy \
  --policy-name GitHubActionsEKSDescribeClusterPolicy \
  --policy-document file://eks-describe-cluster-policy.json
```

---

## Attach Policy to IAM Role

```bash
aws iam attach-role-policy \
  --role-name GitHubActionsEKSDeployRole \
  --policy-arn arn:aws:iam::<account-id>:policy/GitHubActionsEKSDescribeClusterPolicy
```

---

# Step 5 — Enable EKS Access Entries

Check authentication mode:

```bash
aws eks describe-cluster \
  --name my-cluster \
  --region ap-south-1 \
  --query "cluster.accessConfig.authenticationMode" \
  --output text
```

If output is `CONFIG_MAP`, update it:

```bash
aws eks update-cluster-config \
  --name my-cluster \
  --region ap-south-1 \
  --access-config authenticationMode=API_AND_CONFIG_MAP
```

Wait until cluster status becomes `ACTIVE`.

---

# Step 6 — Create EKS Access Entry

## Create Access Entry

```bash
aws eks create-access-entry \
  --cluster-name my-cluster \
  --region ap-south-1 \
  --principal-arn arn:aws:iam::<account-id>:role/GitHubActionsEKSDeployRole
```

---

## Verify Access Entry

```bash
aws eks describe-access-entry \
  --cluster-name my-cluster \
  --region ap-south-1 \
  --principal-arn arn:aws:iam::<account-id>:role/GitHubActionsEKSDeployRole
```

---

# Step 7 — Associate Access Policy

Grant namespace-level edit access:

```bash
aws eks associate-access-policy \
  --cluster-name my-cluster \
  --region ap-south-1 \
  --principal-arn arn:aws:iam::<account-id>:role/GitHubActionsEKSDeployRole \
  --policy-arn arn:aws:eks::aws:cluster-access-policy/AmazonEKSEditPolicy \
  --access-scope type=namespace,namespaces=qa
```

Verify association:

```bash
aws eks list-associated-access-policies \
  --cluster-name my-cluster \
  --region ap-south-1 \
  --principal-arn arn:aws:iam::<account-id>:role/GitHubActionsEKSDeployRole
```

---

# Step 8 — Add GitHub Actions Secret

Go to:

```text
GitHub Repository → Settings → Secrets and variables → Actions
```

Create secret:

| Secret Name        | Value                                                     |
| ------------------ | --------------------------------------------------------- |
| AWS_ROLE_TO_ASSUME | arn:aws:iam::<account-id>:role/GitHubActionsEKSDeployRole |

---

# Install AWS Load Balancer Controller

AWS Load Balancer Controller is used to create Application Load Balancers (ALB) for Kubernetes Ingress resources.

---

# Step 1 — Verify Public Subnet Tags

Check subnet tags:

```bash
aws ec2 describe-subnets \
  --region ap-south-1 \
  --filters "Name=tag:kubernetes.io/role/elb,Values=1" \
  --query "Subnets[*].{ID:SubnetId,AZ:AvailabilityZone}" \
  --output table
```

---

# Step 2 — Create IAM Policy for ALB Controller

Download IAM policy:

```bash
wget https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/main/docs/install/iam_policy.json
```

Create IAM policy:

```bash
aws iam create-policy \
  --policy-name AWSLoadBalancerControllerIAMPolicy \
  --policy-document file://iam_policy.json
```

---

# Step 3 — Associate IAM OIDC Provider

```bash
eksctl utils associate-iam-oidc-provider \
  --region ap-south-1 \
  --cluster my-cluster \
  --approve
```

---

# Step 4 — Create IAM Service Account

```bash
eksctl create iamserviceaccount \
  --cluster=my-cluster \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --attach-policy-arn=arn:aws:iam::<account-id>:policy/AWSLoadBalancerControllerIAMPolicy \
  --override-existing-serviceaccounts \
  --region ap-south-1 \
  --approve
```

This command creates:

* IAM Role
* Trust policy
* Kubernetes service account

---

# Step 5 — Install AWS Load Balancer Controller Using Helm

Add Helm repository:

```bash
helm repo add eks https://aws.github.io/eks-charts
helm repo update
```

Install controller:

```bash
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=my-cluster \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --version 1.14.0
```

---

# Step 6 — Verify Controller Status

Check deployment:

```bash
kubectl get deployment -n kube-system aws-load-balancer-controller
```

Check pods:

```bash
kubectl get pods -n kube-system | grep aws-load-balancer-controller
```

Expected status:

```text
Running
```

---

# Final Verification Checklist

Make sure all components are working:

* EKS cluster is ACTIVE
* Worker nodes are Ready
* kubectl access works
* GitHub OIDC authentication works
* IAM role is attached properly
* AWS Load Balancer Controller is Running
* ALB subnets are tagged correctly

---

# Useful Commands

## View Nodes

```bash
kubectl get nodes
```

## View All Pods

```bash
kubectl get pods -A
```

## Check Cluster Info

```bash
kubectl cluster-info
```

## View Ingress Resources

```bash
kubectl get ingress -A
```

## Check AWS Load Balancer Controller Logs

```bash
kubectl logs -n kube-system deployment/aws-load-balancer-controller
```

---

# Notes

* Replace placeholders like `<cluster-name>`, `<region-name>`, and `<account-id>` with actual values.
* Use least-privilege IAM permissions in production.
* Store sensitive information securely using GitHub Secrets or AWS Secrets Manager.
* Always verify AWS billing before creating large clusters or load balancers.

---

Project setup completed successfully.
