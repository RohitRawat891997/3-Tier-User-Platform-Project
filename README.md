# ELK Flow Diagram

![ELK Flow Diagram](https://raw.githubusercontent.com/RohitRawat891997/3-Tier-User-Platform-Project/qa/EKS-Flow-Diagram-DevOpsShack.png)

## MySQL on Amazon EKS with AWS Secrets Manager, External Secrets Operator (ESO), and EBS Storage

This guide explains how to:

* Store MySQL credentials securely in AWS Secrets Manager
* Sync secrets into Kubernetes using External Secrets Operator (ESO)
* Configure secure AWS authentication using IRSA
* Deploy MySQL on Amazon EKS using StatefulSet
* Persist MySQL data using Amazon EBS volumes
* Convert all PowerShell commands into Linux terminal commands

---

# Architecture Flow

```text
AWS Secrets Manager
        ↓
External Secrets Operator (ESO)
        ↓
Kubernetes Secret (mysql-secret)
        ↓
MySQL StatefulSet
        ↓
Amazon EBS Persistent Volume
```

---

# Prerequisites

Before starting, ensure you have:

* An active AWS account
* An EKS cluster already created
* kubectl installed
* eksctl installed
* AWS CLI configured
* Helm installed

Verify:

```bash
aws sts get-caller-identity
kubectl get nodes
helm version
eksctl version
```

---

# Step 1 — Create the QA Namespace

Create a dedicated namespace for QA resources.

## Create Namespace

```bash
kubectl create namespace qa
```

## Verify

```bash
kubectl get ns qa
```

---

# Step 2 — Store MySQL Secrets in AWS Secrets Manager

Instead of manually creating Kubernetes secrets, store all credentials securely inside AWS Secrets Manager.

## Delete Existing Secret (Optional)

```bash
aws secretsmanager delete-secret \
  --secret-id qa/mysql-secret \
  --force-delete-without-recovery \
  --region ap-south-1
```

---

## Create JSON Secret File

Create a file named:

```text
mysql-secret.json
```

Add the following content:

```json
{
  "MYSQL_ROOT_PASSWORD": "rootpass",
  "MYSQL_DATABASE": "test_db",
  "MYSQL_USER": "appuser",
  "MYSQL_PASSWORD": "apppass",
  "DATABASE_URL": "mysql://appuser:apppass@mysql:3306/test_db"
}
```

---

## Create Secret in AWS Secrets Manager

```bash
aws secretsmanager create-secret \
  --region ap-south-1 \
  --name qa/mysql-secret \
  --description "MySQL credentials for qa namespace" \
  --secret-string file://mysql-secret.json
```

---

## Verify Secret

```bash
aws secretsmanager get-secret-value \
  --region ap-south-1 \
  --secret-id qa/mysql-secret \
  --query SecretString \
  --output text
```

---

# Why This Step Matters

* AWS becomes the single source of truth for secrets
* Avoids storing credentials directly inside Kubernetes YAML
* Makes secret rotation easier
* Improves security and auditing

---

# Step 3 — Configure OIDC Provider for EKS

IRSA (IAM Roles for Service Accounts) requires an OIDC provider.

Without OIDC:

* Kubernetes service accounts cannot assume IAM roles
* Pods cannot securely access AWS APIs

---

## Set Variables

```bash
export CLUSTER_NAME="my-cluster"
export AWS_REGION="ap-south-1"
```

---

## Verify OIDC Issuer

```bash
aws eks describe-cluster \
  --name $CLUSTER_NAME \
  --region $AWS_REGION \
  --query "cluster.identity.oidc.issuer" \
  --output text
```

---

## Associate OIDC Provider

```bash
eksctl utils associate-iam-oidc-provider \
  --cluster $CLUSTER_NAME \
  --region $AWS_REGION \
  --approve
```

This command is safe even if OIDC is already associated.

---

# Step 4 — Install External Secrets Operator (ESO)

ESO runs inside Kubernetes and continuously syncs secrets from AWS Secrets Manager into Kubernetes Secrets.

---

## Add Helm Repository

```bash
helm repo add external-secrets https://charts.external-secrets.io
```

---

## Update Helm Repositories

```bash
helm repo update
```

---

## Install ESO

```bash
helm install external-secrets external-secrets/external-secrets \
  -n external-secrets \
  --create-namespace
```

---

## Verify Installation

```bash
kubectl get pods -n external-secrets
```

```bash
kubectl get crds | grep external-secrets
```

Expected:

* controller pod running
* webhook pod running
* cert-controller pod running
* external-secrets CRDs available

---

# Step 5 — Create IAM Policy for Secrets Access

ESO needs AWS permission to read secrets.

Create a policy that only allows access to:

```text
qa/mysql-secret
```

---

## Create Policy File

Create:

```text
eso-secrets-policy.json
```

Add:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "secretsmanager:GetSecretValue",
        "secretsmanager:DescribeSecret"
      ],
      "Resource": "arn:aws:secretsmanager:ap-south-1:551322107788:secret:qa/mysql-secret*"
    }
  ]
}
```

---

## Create IAM Policy

```bash
aws iam create-policy \
  --policy-name ESOSecretsManagerQAReadPolicy \
  --policy-document file://eso-secrets-policy.json
```

---

## Save Policy ARN

```bash
export POLICY_ARN=$(aws iam list-policies \
  --scope Local \
  --query "Policies[?PolicyName=='ESOSecretsManagerQAReadPolicy'].Arn" \
  --output text)
```

---

## Verify

```bash
echo $POLICY_ARN
```

---

# Step 6 — Create IAM Service Account Using IRSA

This step creates:

* Kubernetes ServiceAccount
* IAM Role
* IAM Role ↔ Kubernetes ServiceAccount mapping

This allows ESO to securely access AWS Secrets Manager.

---

## Create IAM Service Account

```bash
eksctl create iamserviceaccount \
  --name eso-qa-sa \
  --namespace qa \
  --cluster $CLUSTER_NAME \
  --region $AWS_REGION \
  --role-name eso-qa-secrets-role \
  --attach-policy-arn $POLICY_ARN \
  --approve
```

---

## Verify

```bash
kubectl get sa eso-qa-sa -n qa -o yaml
```

Expected annotation:

```yaml
eks.amazonaws.com/role-arn: arn:aws:iam::<ACCOUNT_ID>:role/eso-qa-secrets-role
```

---

# Step 7 — Create SecretStore

SecretStore tells ESO:

* Which provider to use
* Which region to use
* Which authentication method to use

---

## Create secretstore.yaml

```yaml
apiVersion: external-secrets.io/v1
kind: SecretStore
metadata:
  name: aws-secretsmanager
  namespace: qa
spec:
  provider:
    aws:
      service: SecretsManager
      region: ap-south-1
      auth:
        jwt:
          serviceAccountRef:
            name: eso-qa-sa
```

---

## Apply SecretStore

```bash
kubectl apply -f secretstore.yaml
```

---

## Verify

```bash
kubectl get secretstore -n qa
```

```bash
kubectl describe secretstore aws-secretsmanager -n qa
```

Expected:

```text
READY: True
store validated
```

---

# Step 8 — Create ExternalSecret

ExternalSecret defines:

* Which AWS secret to fetch
* Which keys to extract
* Which Kubernetes secret to create

---

## Create externalsecret.yaml

```yaml
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: mysql-external-secret
  namespace: qa
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secretsmanager
    kind: SecretStore
  target:
    name: mysql-secret
    creationPolicy: Owner
  data:
    - secretKey: MYSQL_ROOT_PASSWORD
      remoteRef:
        key: qa/mysql-secret
        property: MYSQL_ROOT_PASSWORD

    - secretKey: MYSQL_DATABASE
      remoteRef:
        key: qa/mysql-secret
        property: MYSQL_DATABASE

    - secretKey: MYSQL_USER
      remoteRef:
        key: qa/mysql-secret
        property: MYSQL_USER

    - secretKey: MYSQL_PASSWORD
      remoteRef:
        key: qa/mysql-secret
        property: MYSQL_PASSWORD

    - secretKey: DATABASE_URL
      remoteRef:
        key: qa/mysql-secret
        property: DATABASE_URL
```

---

## Apply ExternalSecret

```bash
kubectl apply -f externalsecret.yaml
```

---

## Verify ExternalSecret

```bash
kubectl get externalsecret -n qa
```

```bash
kubectl describe externalsecret mysql-external-secret -n qa
```

---

## Verify Kubernetes Secret Creation

```bash
kubectl get secret mysql-secret -n qa
```

```bash
kubectl describe secret mysql-secret -n qa
```

Expected:

```text
READY: True
SecretSynced
```

---

## Force Secret Refresh (Optional)

```bash
kubectl annotate externalsecret mysql-external-secret \
  -n qa \
  force-sync="$(date +%s)" \
  --overwrite
```

---

# Step 9 — Configure Amazon EBS CSI Driver

MySQL requires persistent storage.

Amazon EBS CSI Driver allows Kubernetes to dynamically create EBS volumes.

---

# Components Overview

## 1. OIDC Provider

Purpose:

* Enables secure authentication between Kubernetes and AWS
* Required for IRSA

---

## 2. IAM Role

Purpose:

* Grants permission to manage EBS volumes
* Uses AmazonEBSCSIDriverPolicy

---

## 3. EBS CSI Driver

Purpose:

* Creates and attaches EBS volumes dynamically
* Runs as pods inside Kubernetes

---

## 4. StorageClass

Purpose:

* Defines how storage is provisioned
* Uses AWS EBS backend

---

# Step 10 — Create IAM Role for EBS CSI Driver

## Create IAM Service Account

```bash
eksctl create iamserviceaccount \
  --name ebs-csi-controller-sa \
  --namespace kube-system \
  --cluster $CLUSTER_NAME \
  --region $AWS_REGION \
  --role-name AmazonEKS_EBS_CSI_DriverRole \
  --role-only \
  --attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy \
  --approve
```

---

# Step 11 — Install EBS CSI Add-on

## Create Add-on

```bash
aws eks create-addon \
  --cluster-name $CLUSTER_NAME \
  --addon-name aws-ebs-csi-driver \
  --region $AWS_REGION \
  --service-account-role-arn arn:aws:iam::551322107788:role/AmazonEKS_EBS_CSI_DriverRole \
  --resolve-conflicts OVERWRITE
```

---

## Update Existing Add-on

```bash
aws eks update-addon \
  --cluster-name $CLUSTER_NAME \
  --addon-name aws-ebs-csi-driver \
  --region $AWS_REGION \
  --service-account-role-arn arn:aws:iam::551322107788:role/AmazonEKS_EBS_CSI_DriverRole \
  --resolve-conflicts OVERWRITE
```

---

# Step 12 — Verify EBS CSI Driver

```bash
kubectl get pods -n kube-system | grep ebs
```

Expected:

* ebs-csi-controller pods running
* ebs-csi-node pods running

---

# Step 13 — Create StorageClass

## Create storageclass-ebs.yaml

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ebs-sc
provisioner: ebs.csi.aws.com
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
parameters:
  type: gp3
  fsType: ext4
```

---

## Apply StorageClass

```bash
kubectl apply -f storageclass-ebs.yaml
```

---

# Why This StorageClass Works

* `ebs.csi.aws.com` uses AWS EBS CSI Driver
* `WaitForFirstConsumer` delays provisioning until scheduling
* `gp3` provides SSD-backed storage
* `allowVolumeExpansion` supports resizing volumes

---

# Step 14 — Deploy MySQL StatefulSet

## Create mysql-statefulset.yaml

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
  namespace: qa
spec:
  serviceName: mysql
  replicas: 1
  selector:
    matchLabels:
      app: mysql

  template:
    metadata:
      labels:
        app: mysql

    spec:
      containers:
        - name: mysql
          image: mysql:8.0

          args:
            - "--default-authentication-plugin=mysql_native_password"

          ports:
            - containerPort: 3306

          envFrom:
            - secretRef:
                name: mysql-secret

          volumeMounts:
            - name: mysql-storage
              mountPath: /var/lib/mysql

  volumeClaimTemplates:
    - metadata:
        name: mysql-storage

      spec:
        storageClassName: ebs-sc

        accessModes:
          - ReadWriteOnce

        resources:
          requests:
            storage: 5Gi
```

---

## Apply StatefulSet

```bash
kubectl apply -f mysql-statefulset.yaml
```

---

# Step 15 — Verify MySQL Deployment

## Check Pods

```bash
kubectl get pods -n qa
```

---

## Check PVC

```bash
kubectl get pvc -n qa
```

---

## Check StatefulSet

```bash
kubectl get statefulset -n qa
```

---

# Final Flow Summary

```text
1. Store secrets in AWS Secrets Manager
2. ESO runs inside EKS
3. ESO uses IRSA to access AWS securely
4. SecretStore defines AWS connection settings
5. ExternalSecret fetches qa/mysql-secret
6. ESO creates mysql-secret inside Kubernetes
7. MySQL reads mysql-secret using envFrom
8. EBS CSI Driver dynamically provisions storage
9. MySQL data persists on Amazon EBS
```

---

# Important Notes

* Never store database passwords directly inside Kubernetes YAML
* Use IRSA instead of static AWS credentials
* Always use StatefulSet for databases
* Use EBS volumes for persistent MySQL data
* Use gp3 volumes for better performance and lower cost

---

# Useful Troubleshooting Commands

## Check ESO Logs

```bash
kubectl logs -n external-secrets deploy/external-secrets
```

---

## Check ExternalSecret Status

```bash
kubectl describe externalsecret mysql-external-secret -n qa
```

---

## Check SecretStore Status

```bash
kubectl describe secretstore aws-secretsmanager -n qa
```

---

## Check PVC Events

```bash
kubectl describe pvc -n qa
```

---

## Check MySQL Pod Logs

```bash
kubectl logs -n qa mysql-0
```

---

# Cleanup Commands

## Delete StatefulSet

```bash
kubectl delete statefulset mysql -n qa
```

---

## Delete ExternalSecret

```bash
kubectl delete externalsecret mysql-external-secret -n qa
```

---

## Delete SecretStore

```bash
kubectl delete secretstore aws-secretsmanager -n qa
```

---

## Delete Namespace

```bash
kubectl delete namespace qa
```

---

# Conclusion

You now have:

* Secure secret management using AWS Secrets Manager
* Automatic Kubernetes secret synchronization using ESO
* Secure AWS authentication using IRSA
* Persistent MySQL storage using Amazon EBS
* A production-style MySQL deployment on Amazon EKS
