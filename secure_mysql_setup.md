````markdown
# Setup MySQL on Amazon EKS Using AWS Secrets Manager, External Secrets Operator (ESO), IRSA, and EBS CSI Driver

## Architecture

```text
AWS Secrets Manager
        │
        ▼
External Secrets Operator (ESO)
        │
       IRSA
        │
        ▼
SecretStore
        │
        ▼
ExternalSecret
        │
        ▼
Kubernetes Secret (mysql-secret)
        │
        ▼
MySQL StatefulSet
        │
        ▼
Persistent Volume Claim
        │
        ▼
StorageClass (ebs-sc)
        │
        ▼
EBS CSI Driver
        │
        ▼
Amazon EBS Volume
````

---

# Prerequisites

Verify AWS access:

```bash
aws sts get-caller-identity
```

Verify EKS cluster:

```bash
aws eks describe-cluster \
  --name my-cluster \
  --region ap-south-1 \
  --query "cluster.status"
```

Expected:

```text
ACTIVE
```

---

# Step 1: Create Namespace

```bash
kubectl create namespace prod
```

Verify:

```bash
kubectl get ns
```

---

# Step 2: Store MySQL Secret in AWS Secrets Manager

Delete existing secret (optional):

```bash
aws secretsmanager delete-secret \
  --secret-id prod/mysql-secret \
  --force-delete-without-recovery \
  --region ap-south-1
```

Create JSON file:

```bash
cat <<EOF > mysql-secret.json
{
  "MYSQL_ROOT_PASSWORD": "rootpass",
  "MYSQL_DATABASE": "test_db",
  "MYSQL_USER": "appuser",
  "MYSQL_PASSWORD": "apppass",
  "DATABASE_URL": "mysql://appuser:apppass@mysql:3306/test_db"
}
EOF
```

Create secret:

```bash
aws secretsmanager create-secret \
  --region ap-south-1 \
  --name prod/mysql-secret \
  --description "MySQL credentials for prod namespace" \
  --secret-string file://mysql-secret.json
```

Verify:

```bash
aws secretsmanager get-secret-value \
  --region ap-south-1 \
  --secret-id prod/mysql-secret \
  --query SecretString \
  --output text
```

---

# Step 3: Associate EKS OIDC Provider

Set variables:

```bash
export CLUSTER_NAME=my-cluster
export AWS_REGION=ap-south-1
```

Verify OIDC issuer:

```bash
aws eks describe-cluster \
  --name $CLUSTER_NAME \
  --region $AWS_REGION \
  --query "cluster.identity.oidc.issuer" \
  --output text
```

Associate OIDC provider:

```bash
eksctl utils associate-iam-oidc-provider \
  --cluster $CLUSTER_NAME \
  --region $AWS_REGION \
  --approve
```

---

# Step 4: Install External Secrets Operator

Add Helm repository:

```bash
helm repo add external-secrets https://charts.external-secrets.io
helm repo update
```

Install ESO:

```bash
helm install external-secrets \
  external-secrets/external-secrets \
  -n external-secrets \
  --create-namespace
```

Verify:

```bash
kubectl get pods -n external-secrets
```

```bash
kubectl get crds | grep external-secrets
```

---

# Step 5: Create IAM Policy for Secrets Manager

Create policy:

```bash
cat <<EOF > eso-secrets-policy.json
{
  "Version":"2012-10-17",
  "Statement":[
    {
      "Effect":"Allow",
      "Action":[
        "secretsmanager:GetSecretValue",
        "secretsmanager:DescribeSecret"
      ],
      "Resource":"arn:aws:secretsmanager:ap-south-1:551322107788:secret:prod/mysql-secret*"
    }
  ]
}
EOF
```

Create IAM policy:

```bash
aws iam create-policy \
  --policy-name ESOSecretsManagerPRODReadPolicy \
  --policy-document file://eso-secrets-policy.json
```

Get Policy ARN:

```bash
export POLICY_ARN=$(aws iam list-policies \
  --scope Local \
  --query "Policies[?PolicyName=='ESOSecretsManagerPRODReadPolicy'].Arn" \
  --output text)
```

Verify:

```bash
echo $POLICY_ARN
```

---

# Step 6: Create IRSA Service Account

```bash
eksctl create iamserviceaccount \
  --name eso-prod-sa \
  --namespace prod \
  --cluster my-cluster \
  --region ap-south-1 \
  --role-name eso-prod-secrets-role \
  --attach-policy-arn $POLICY_ARN \
  --approve
```

Verify:

```bash
kubectl get sa eso-prod-sa -n prod -o yaml
```

Expected annotation:

```yaml
eks.amazonaws.com/role-arn: arn:aws:iam::<ACCOUNT_ID>:role/eso-prod-secrets-role
```

---

# Step 7: Create SecretStore

Create:

```yaml
apiVersion: external-secrets.io/v1
kind: SecretStore
metadata:
  name: aws-secretsmanager
  namespace: prod
spec:
  provider:
    aws:
      service: SecretsManager
      region: ap-south-1
      auth:
        jwt:
          serviceAccountRef:
            name: eso-prod-sa
```

Apply:

```bash
kubectl apply -f secretstore.yaml
```

Verify:

```bash
kubectl get secretstore -n prod
```

```bash
kubectl describe secretstore aws-secretsmanager -n prod
```

Expected:

```text
READY=True
```

---

# Step 8: Create ExternalSecret

Create:

```yaml
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: mysql-external-secret
  namespace: prod
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
        key: prod/mysql-secret
        property: MYSQL_ROOT_PASSWORD
    - secretKey: MYSQL_DATABASE
      remoteRef:
        key: prod/mysql-secret
        property: MYSQL_DATABASE
    - secretKey: MYSQL_USER
      remoteRef:
        key: prod/mysql-secret
        property: MYSQL_USER
    - secretKey: MYSQL_PASSWORD
      remoteRef:
        key: prod/mysql-secret
        property: MYSQL_PASSWORD
    - secretKey: DATABASE_URL
      remoteRef:
        key: prod/mysql-secret
        property: DATABASE_URL
```

Apply:

```bash
kubectl apply -f externalsecret.yaml
```

Verify:

```bash
kubectl get externalsecret -n prod
```

```bash
kubectl describe externalsecret mysql-external-secret -n prod
```

```bash
kubectl get secret mysql-secret -n prod
```

Force sync:

```bash
kubectl annotate externalsecret mysql-external-secret \
  -n prod \
  force-sync="$(date +%s)" \
  --overwrite
```

---

# Step 9: Create IAM Role for EBS CSI Driver

```bash
eksctl create iamserviceaccount \
  --name ebs-csi-controller-sa \
  --namespace kube-system \
  --cluster my-cluster \
  --region ap-south-1 \
  --role-name AmazonEKS_EBS_CSI_DriverRole \
  --role-only \
  --attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy \
  --approve
```

---

# Step 10: Install EBS CSI Add-on

Install:

```bash
aws eks create-addon \
  --cluster-name my-cluster \
  --addon-name aws-ebs-csi-driver \
  --region ap-south-1 \
  --service-account-role-arn arn:aws:iam::551322107788:role/AmazonEKS_EBS_CSI_DriverRole \
  --resolve-conflicts OVERWRITE
```

If already exists:

```bash
aws eks update-addon \
  --cluster-name my-cluster \
  --addon-name aws-ebs-csi-driver \
  --region ap-south-1 \
  --service-account-role-arn arn:aws:iam::551322107788:role/AmazonEKS_EBS_CSI_DriverRole \
  --resolve-conflicts OVERWRITE
```

Verify:

```bash
kubectl get pods -n kube-system | grep ebs
```

---

# Step 11: Create StorageClass

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

Apply:

```bash
kubectl apply -f storageclass-ebs.yaml
```

Verify:

```bash
kubectl get storageclass
```

---

# Step 12: Deploy MySQL StatefulSet

Apply StatefulSet:

```bash
kubectl apply -f mysql-statefulset.yaml
```

Verify:

```bash
kubectl get pods -n prod
```

```bash
kubectl get pvc -n prod
```

```bash
kubectl get pv
```

---

# End-to-End Flow

```text
1. Store secret in AWS Secrets Manager
2. ESO authenticates using IRSA
3. SecretStore connects to AWS
4. ExternalSecret fetches secret
5. ESO creates mysql-secret
6. MySQL consumes mysql-secret
7. PVC requests storage
8. StorageClass uses EBS CSI Driver
9. EBS Volume created automatically
10. MySQL data persists on Amazon EBS
```

```
```
