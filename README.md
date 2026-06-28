# IAM IRSA Failure Troubleshooting on Amazon EKS

## 📌 Project Overview

This project demonstrates how to troubleshoot an IAM Roles for Service Accounts (IRSA) failure in an Amazon EKS cluster.

The scenario simulates an application that suddenly loses access to an Amazon DynamoDB table because the pod starts using the Node IAM Role instead of the IRSA Role, resulting in an AccessDeniedException.

---

## 🏗️ Architecture

```text
Application Pod
      │
      ▼
ServiceAccount (app-sa)
      │
      ▼
IAM Role (IRSA)
      │
      ▼
Amazon DynamoDB
```

---

## 🚨 Incident

**Application Logs**

```text
botocore.exceptions.ClientError:

An error occurred (AccessDeniedException)
when calling the GetItem operation:

User:
arn:aws:sts::<ACCOUNT_ID>:assumed-role/eks-nodegroup-role

is not authorized to perform:
dynamodb:GetItem
```

---

## 🎯 Objective

- Deploy an Amazon EKS cluster
- Configure IAM Roles for Service Accounts (IRSA)
- Allow a pod to access DynamoDB
- Simulate an IRSA failure
- Identify the root cause
- Fix the issue

---

## 🛠️ AWS Services Used

- Amazon EKS
- Amazon DynamoDB
- IAM
- IAM Roles for Service Accounts (IRSA)
- OIDC Provider
- AWS CLI
- kubectl
- eksctl

---

## 📁 Project Structure

```text
IAM-IRSA-Failure/
│
├── README.md
├── pod.yaml
├── dynamodb-policy.json
├── item.json
└── screenshots/
```

---

## ⚙️ Prerequisites

- AWS Account
- AWS CLI
- kubectl
- eksctl
- IAM User with Administrator permissions

---

# Step 1: Configure AWS CLI

```bash
aws configure
```

Verify:

```bash
aws sts get-caller-identity
```

---

# Step 2: Create EKS Cluster

```bash
eksctl create cluster \
--name irsa-cluster \
--region us-east-1 \
--nodes 2
```

Verify:

```bash
kubectl get nodes
```

---

# Step 3: Create DynamoDB Table

```bash
aws dynamodb create-table \
--table-name customer-data \
--attribute-definitions AttributeName=id,AttributeType=S \
--key-schema AttributeName=id,KeyType=HASH \
--billing-mode PAY_PER_REQUEST \
--region us-east-1
```

---

# Step 4: Insert Sample Data

Create `item.json`

```json
{
  "id": {
    "S": "1"
  },
  "name": {
    "S": "Dharshan"
  }
}
```

Insert the item:

```bash
aws dynamodb put-item \
--table-name customer-data \
--item file://item.json \
--region us-east-1
```

---

# Step 5: Create IAM Policy

Create `dynamodb-policy.json`

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "dynamodb:GetItem"
      ],
      "Resource": "arn:aws:dynamodb:us-east-1:<ACCOUNT_ID>:table/customer-data"
    }
  ]
}
```

Create the policy:

```bash
aws iam create-policy \
--policy-name DynamoDBReadPolicy \
--policy-document file://dynamodb-policy.json
```

---

# Step 6: Associate OIDC Provider

```bash
eksctl utils associate-iam-oidc-provider \
--cluster irsa-cluster \
--region us-east-1 \
--approve
```

---

# Step 7: Create IRSA Service Account

```bash
eksctl create iamserviceaccount \
--cluster irsa-cluster \
--region us-east-1 \
--namespace default \
--name app-sa \
--attach-policy-arn <POLICY_ARN> \
--approve
```

Verify:

```bash
kubectl get sa app-sa -o yaml
```

---

# Step 8: Deploy Test Pod

Create `pod.yaml`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: aws-cli

spec:
  serviceAccountName: app-sa

  containers:
  - name: aws
    image: amazon/aws-cli
    command:
    - sleep
    - "3600"
```

Deploy:

```bash
kubectl apply -f pod.yaml
```

---

# Step 9: Verify IRSA

Enter the pod:

```bash
kubectl exec -it aws-cli -- sh
```

Verify IAM identity:

```bash
aws sts get-caller-identity
```

Expected:

```text
arn:aws:sts::<ACCOUNT_ID>:assumed-role/<IRSA_ROLE_NAME>/...
```

Read from DynamoDB:

```bash
aws dynamodb get-item \
--table-name customer-data \
--key '{"id":{"S":"1"}}' \
--region us-east-1
```

---

# Step 10: Simulate the Failure

Edit `pod.yaml`

Change:

```yaml
serviceAccountName: app-sa
```

to:

```yaml
serviceAccountName: default
```

Delete and recreate the pod:

```bash
kubectl delete pod aws-cli

kubectl apply -f pod.yaml
```

---

# Step 11: Observe the Failure

Verify IAM identity:

```bash
aws sts get-caller-identity
```

Output:

```text
arn:aws:sts::<ACCOUNT_ID>:assumed-role/...NodeInstanceRole
```

Try reading from DynamoDB:

```bash
aws dynamodb get-item \
--table-name customer-data \
--key '{"id":{"S":"1"}}' \
--region us-east-1
```

Result:

```text
AccessDeniedException

User:
arn:aws:sts::<ACCOUNT_ID>:assumed-role/...NodeInstanceRole

is not authorized to perform:
dynamodb:GetItem
```

---

# 🔍 Root Cause Analysis

| Problem | Cause |
|----------|-------|
| Pod used Node IAM Role | IRSA was not used |
| IRSA failed | Pod used the default ServiceAccount instead of `app-sa` |
| AccessDeniedException | Node IAM Role did not have DynamoDB permissions |

---

# ✅ Resolution

Update `pod.yaml`:

```yaml
serviceAccountName: app-sa
```

Recreate the pod:

```bash
kubectl delete pod aws-cli

kubectl apply -f pod.yaml
```

Verify:

```bash
kubectl exec -it aws-cli -- sh

aws sts get-caller-identity
```

Expected:

```text
arn:aws:sts::<ACCOUNT_ID>:assumed-role/<IRSA_ROLE_NAME>/...
```

Verify DynamoDB access:

```bash
aws dynamodb get-item \
--table-name customer-data \
--key '{"id":{"S":"1"}}' \
--region us-east-1
```

The request should now succeed.

---

# 📷 Screenshots

Include screenshots of:

- AWS CLI Configuration
- EKS Cluster Creation
- kubectl get nodes
- DynamoDB Table
- IAM Policy
- OIDC Provider
- IRSA Service Account
- Pod Deployment
- Successful IRSA Verification
- AccessDeniedException
- Successful Fix Verification

---

# 📚 Key Learnings

- Configured IAM Roles for Service Accounts (IRSA) on Amazon EKS.
- Granted least-privilege access to DynamoDB using IAM policies.
- Verified pod identity using AWS STS.
- Simulated an IRSA failure by changing the ServiceAccount.
- Diagnosed the root cause using AWS CLI and Kubernetes.
- Restored IRSA configuration and verified successful access to DynamoDB.

---

## 👨‍💻 Author

**Dharshan R**

B.Tech Information Technology  
Aspiring DevOps & Cloud Engineer
