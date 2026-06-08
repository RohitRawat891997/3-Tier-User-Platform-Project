# GitHub OIDC-Based Secure Deployment Architecture for Amazon EKS (PROD)

## Overview

This guide implements a secure GitHub Actions → AWS EKS deployment architecture using OpenID Connect (OIDC).

### Authentication Flow

```text
GitHub Actions
      │
      ▼
Requests OIDC Token from GitHub
      │
      ▼
AWS IAM Trusts GitHub OIDC Provider
      │
      ▼
Assume IAM Role (Short-Lived Credentials)
      │
      ▼
EKS Access Entry Authentication
      │
      ▼
aws eks update-kubeconfig
      │
      ▼
kubectl deploys to namespace: prod
```

## Benefits

* No long-lived AWS Access Keys
* No kubeconfig stored in GitHub Secrets
* No Kubernetes ServiceAccount tokens stored in GitHub
* Uses short-lived credentials
* AWS recommended approach
* GitHub recommended approach
* Namespace-scoped access control

---

# Prerequisites

Verify AWS CLI authentication:

```bash
aws sts get-caller-identity
```

Verify EKS cluster status and authentication mode:

```bash
aws eks describe-cluster \
  --name my-cluster \
  --region ap-south-1 \
  --query "{status:cluster.status,authMode:cluster.accessConfig.authenticationMode}" \
  --output json
```

Expected Output:

```json
{
  "status": "ACTIVE",
  "authMode": "API_AND_CONFIG_MAP"
}
```

Supported authentication modes:

* API
* API_AND_CONFIG_MAP

---

# Step 1: Create GitHub OIDC Provider

## Check Existing Provider

```bash
aws iam list-open-id-connect-providers \
  --query "OpenIDConnectProviderList[?contains(Arn, 'token.actions.githubusercontent.com')].Arn" \
  --output text
```

## Delete Existing Provider (Optional)

```bash
aws iam delete-open-id-connect-provider \
  --open-id-connect-provider-arn arn:aws:iam::551322107788:oidc-provider/token.actions.githubusercontent.com
```

## Create GitHub OIDC Provider

```bash
aws iam create-open-id-connect-provider \
  --url https://token.actions.githubusercontent.com \
  --client-id-list sts.amazonaws.com \
  --thumbprint-list 6938fd4d98bab03faadb97b34396831e3780aea1
```

### What Does This Do?

Creates an IAM OIDC Identity Provider that allows AWS to trust GitHub-issued OIDC tokens.

Used by:

```text
GitHub Actions
      ↓
sts:AssumeRoleWithWebIdentity
      ↓
AWS IAM Role
```

---

# Step 2: Create IAM Trust Policy

Create trust policy file:

```bash
cat <<EOF > github-oidc-trust.json
{
  "Version":"2012-10-17",
  "Statement":[
    {
      "Effect":"Allow",
      "Principal":{
        "Federated":"arn:aws:iam::551322107788:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action":"sts:AssumeRoleWithWebIdentity",
      "Condition":{
        "StringEquals":{
          "token.actions.githubusercontent.com:aud":"sts.amazonaws.com"
        },
        "StringLike":{
          "token.actions.githubusercontent.com:sub":"repo:jaiswaladi246/3-Tier-User-Platform-Project:ref:refs/heads/main"
        }
      }
    }
  ]
}
EOF
```

Verify:

```bash
cat github-oidc-trust.json
```

### Security Controls

The trust policy restricts access to:

* Specific GitHub repository
* Specific branch (main)
* GitHub OIDC provider only
* AWS STS audience only

---

# Step 3: Create IAM Role

```bash
aws iam create-role \
  --role-name GitHubActionsEKSDeployRolePROD \
  --assume-role-policy-document file://github-oidc-trust.json
```

Verify:

```bash
aws iam get-role \
  --role-name GitHubActionsEKSDeployRolePROD
```

---

# Step 4: Create IAM Policy for EKS Access

`aws eks update-kubeconfig` requires:

```text
eks:DescribeCluster
```

Create policy file:

```bash
cat <<EOF > eks-describe-cluster-policy.json
{
  "Version":"2012-10-17",
  "Statement":[
    {
      "Sid":"EKSDescribeCluster",
      "Effect":"Allow",
      "Action":[
        "eks:DescribeCluster"
      ],
      "Resource":"arn:aws:eks:ap-south-1:551322107788:cluster/my-cluster"
    }
  ]
}
EOF
```

Verify:

```bash
cat eks-describe-cluster-policy.json
```

Create policy:

```bash
aws iam create-policy \
  --policy-name GitHubActionsEKSDescribeClusterPolicy \
  --policy-document file://eks-describe-cluster-policy.json
```

---

# Step 5: Attach Policy to IAM Role

```bash
POLICY_ARN="arn:aws:iam::551322107788:policy/GitHubActionsEKSDescribeClusterPolicy"

aws iam attach-role-policy \
  --role-name GitHubActionsEKSDeployRolePROD \
  --policy-arn $POLICY_ARN
```

Verify:

```bash
aws iam list-attached-role-policies \
  --role-name GitHubActionsEKSDeployRolePROD
```

---

# Step 6: Verify EKS Authentication Mode

Check:

```bash
aws eks describe-cluster \
  --name my-cluster \
  --region ap-south-1 \
  --query "cluster.accessConfig.authenticationMode" \
  --output text
```

If output is:

```text
CONFIG_MAP
```

Update:

```bash
aws eks update-cluster-config \
  --name my-cluster \
  --region ap-south-1 \
  --access-config authenticationMode=API_AND_CONFIG_MAP
```

Wait for cluster to become active:

```bash
aws eks describe-cluster \
  --name my-cluster \
  --region ap-south-1 \
  --query "cluster.status" \
  --output text
```

Expected:

```text
ACTIVE
```

---

# Step 7: Create EKS Access Entry

Create access entry:

```bash
aws eks create-access-entry \
  --cluster-name my-cluster \
  --region ap-south-1 \
  --principal-arn arn:aws:iam::551322107788:role/GitHubActionsEKSDeployRolePROD
```

Verify:

```bash
aws eks describe-access-entry \
  --cluster-name my-cluster \
  --region ap-south-1 \
  --principal-arn arn:aws:iam::551322107788:role/GitHubActionsEKSDeployRolePROD
```

### Why Access Entries?

Access Entries allow IAM principals to authenticate directly to EKS without manually editing:

```text
aws-auth ConfigMap
```

---

# Step 8: Associate Namespace-Scoped Access Policy

Grant deployment access only to namespace:

```text
prod
```

Associate policy:

```bash
aws eks associate-access-policy \
  --cluster-name my-cluster \
  --region ap-south-1 \
  --principal-arn arn:aws:iam::551322107788:role/GitHubActionsEKSDeployRolePROD \
  --policy-arn arn:aws:eks::aws:cluster-access-policy/AmazonEKSEditPolicy \
  --access-scope type=namespace,namespaces=prod
```

Verify:

```bash
aws eks list-associated-access-policies \
  --cluster-name my-cluster \
  --region ap-south-1 \
  --principal-arn arn:aws:iam::551322107788:role/GitHubActionsEKSDeployRolePROD
```

Expected:

```text
Policy      : AmazonEKSEditPolicy
Scope Type  : Namespace
Namespace   : prod
```

### Security Benefit

The GitHub Actions role can:

✅ Deploy applications

✅ Update deployments

✅ Manage services

Inside:

```text
prod namespace only
```

Cannot manage cluster-wide resources.

---

# Step 9: Configure GitHub Repository Secret

Navigate to:

```text
GitHub Repository
 └── Settings
      └── Secrets and Variables
           └── Actions
```

Create Secret:

```text
AWS_ROLE_TO_ASSUME1
```

Value:

```text
arn:aws:iam::551322107788:role/GitHubActionsEKSDeployRolePROD
```

---

# Step 10: Remove Old Kubernetes Secrets

The following GitHub Secrets are no longer required:

```text
KUBE_CONFIG_B64
K8S_SERVER
K8S_CA_CERT
K8S_TOKEN
```

OIDC completely replaces them.

---

# Step 11: Create Docker Registry Secret in Kubernetes

```bash
kubectl create secret docker-registry regcred \
  --docker-username=devopsshack \
  --docker-password='Dockerhub@888' \
  --docker-email=devopsshack2025@gmail.com \
  -n prod
```

Verify:

```bash
kubectl get secret regcred -n prod
```

---

# Final Deployment Flow

```text
GitHub Push
     │
     ▼
GitHub Actions Workflow
     │
     ▼
OIDC Token Issued
     │
     ▼
Assume IAM Role
     │
     ▼
aws eks update-kubeconfig
     │
     ▼
Authenticate using EKS Access Entry
     │
     ▼
Deploy to prod Namespace
     │
     ▼
Application Running on EKS
```

## Architecture Summary

| Component            | Purpose                 |
| -------------------- | ----------------------- |
| GitHub OIDC Provider | Trust GitHub Tokens     |
| IAM Role             | Temporary AWS Access    |
| Trust Policy         | Restrict Repo & Branch  |
| EKS Access Entry     | Authenticate to Cluster |
| AmazonEKSEditPolicy  | Kubernetes Permissions  |
| Namespace Scope      | Least Privilege Access  |
| GitHub Secret        | IAM Role ARN Only       |

This setup follows AWS and GitHub best practices for secure, short-lived, OIDC-based deployments to Amazon EKS.
