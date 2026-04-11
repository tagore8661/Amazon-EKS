# Private Registry - ECR (Elastic Container Registry)

---

## Table of Contents

- [Introduction](#introduction)
- [Creating and Managing Amazon ECR Repositories](#creating-and-managing-amazon-ecr-repositories)
- [Authenticating on Ubuntu Machine for ECR](#authenticating-on-ubuntu-machine-for-ecr)
- [Pushing a Docker Image to ECR and Deploying it on EKS](#pushing-a-docker-image-to-ecr-and-deploying-it-on-eks)
- [Summary](#summary)

---

## Introduction

- Learn how to **create and manage ECR repositories**.
- Understand ECR's role in your **Kubernetes workflow**.
### Key Topics Covered
  - Create and manage container images in ECR.
  - Authenticate from a client machine.
  - Push Docker images to ECR.
  - Deploy those images in EKS using **IAM role-based access**.

---

## Creating and Managing Amazon ECR Repositories

### ECR Hierarchy

| Term | Definition |
|---|---|
| **ECR** | The registry service itself (Elastic Container Registry) |
| **Repository** | Storage location within ECR for container images of a specific application |
| **Image** | The actual container image stored in a repository |
| **Tag** | Version identifier for an image (e.g., `v1`, `v2`, `latest`) |

### Creating a Repository

#### Step 1 - Navigate to ECR

- AWS Console → Elastic Container Registry (ECR)
- Click **Create repository**

#### Step 2 - Configure Repository

| Setting | Options | Recommended |
|---|---|---|
| **Repository Name** | Simple: `myapp` OR Namespaced: `demo/welcomeapp` | Namespaced (better organization) |
| **Tag Mutability** | **Mutable:** Overwrite existing tags | **Immutable** (enforce version control) |
| | **Immutable:** Block overwriting existing tags | ✓ Best Practice |
| **Encryption** | AES-256 (default) OR AWS KMS | AES-256 (industry standard) |
| **Image Scanning** | Deprecated feature | Skip for now |

#### Repository URI Format

```
<registry-url>/<namespace>/<repository>:<tag>
```

Example:
```
123456789012.dkr.ecr.us-east-1.amazonaws.com/demo/welcomeapp:v1
```

Breakdown:
- `123456789012.dkr.ecr.us-east-1.amazonaws.com` = Registry URL (FQDN)
- `demo` = Namespace
- `welcomeapp` = Repository name
- `v1` = Image tag

### Repository Management

Once created, you can:

| Action | Purpose |
|---|---|
| View Push Commands | Get CLI commands to push/pull images |
| View Summary Images | List all images in the repository |
| Manage Permissions | Control who can access this repository |
| Edit Repository Settings | Change mutability, encryption, scanning |
| Delete Repository | Remove the repository and all images |

### Private vs Public Repositories

| Type | Use Case |
|---|---|
| **Private** (default) | Enterprise, secure, limited access — **recommended** |
| **Public** | Open-source, publicly available images |

---

## Authenticating on Ubuntu Machine for ECR

### Prerequisites

- **AWS CLI** installed on the client machine.
- **Docker** installed on the client machine.
- **IAM permissions** for the user to access ECR.

### Step 1 - Grant IAM Permissions

If the IAM user lacks ECR permissions:

1. Go to **IAM Console** → **Users**
2. Select the user → **Add permissions**
3. Choose: **Attach policies directly**
4. Search for: `container registry` or `AmazonEC2ContainerRegistryFullAccess`
5. Select permission level:
   - `FullAccess` - Full read/write to ECR
   - `PowerUser` - Most operations
   - `PullOnly` - Download images only
   - `ReadOnly` - Inspect images only

6. Click **Add permissions**

### Step 2 - Login to ECR from Client Machine

#### Command Structure

```bash
aws ecr get-login-password --region <region> | docker login --username AWS --password-stdin <registry-url>
```

#### Breakdown

| Component | Meaning |
|---|---|
| `aws ecr get-login-password` | Fetches a **temporary authentication token** (not a static password) |
| `--region <region>` | AWS region where ECR repository is located |
| `docker login` | Authenticates Docker daemon with ECR registry |
| `--username AWS` | Standard username for ECR (always `AWS`) |
| `--password-stdin <registry-url>` | Passes token from AWS CLI to Docker login |

#### Example

```bash
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin 123456789012.dkr.ecr.us-east-1.amazonaws.com
```

**Success Output:**
```
Login Succeeded
```

### Step 3 - Pull Docker Images

#### From Public Registry

```bash
docker pull ghcr.io/sample/welcomeapp:v1
docker pull ghcr.io/sample/welcomeapp:v2
docker pull ghcr.io/sample/welcomeapp:v3
```

> No authentication required for public images.

#### Verify Images

```bash
docker image list
```

---

## Pushing a Docker Image to ECR and Deploying it on EKS

### Step 1 - Tag Images for ECR

Before pushing, tag the images with the ECR URI format.

#### Command

```bash
docker image tag <source-image>:<tag> <ecr-registry>/<namespace>/<repository>:<tag>
```

#### Examples

```bash
docker image tag ghcr.io/sample/welcomeapp:v1 123456789012.dkr.ecr.us-east-1.amazonaws.com/demo/welcomeapp:v1
docker image tag ghcr.io/sample/welcomeapp:v2 123456789012.dkr.ecr.us-east-1.amazonaws.com/demo/welcomeapp:v2
docker image tag ghcr.io/sample/welcomeapp:v3 123456789012.dkr.ecr.us-east-1.amazonaws.com/demo/welcomeapp:v3
docker image tag ghcr.io/sample/welcomeapp:v3 123456789012.dkr.ecr.us-east-1.amazonaws.com/demo/welcomeapp:latest
```

### Step 2 - Push Images to ECR

#### Login Again (if needed)

```bash
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin 123456789012.dkr.ecr.us-east-1.amazonaws.com
```

#### Push Commands

```bash
docker push 123456789012.dkr.ecr.us-east-1.amazonaws.com/demo/welcomeapp:v1
docker push 123456789012.dkr.ecr.us-east-1.amazonaws.com/demo/welcomeapp:v2
docker push 123456789012.dkr.ecr.us-east-1.amazonaws.com/demo/welcomeapp:v3
docker push 123456789012.dkr.ecr.us-east-1.amazonaws.com/demo/welcomeapp:latest
```

> **Note:** Only changes (layers) are pushed; cached layers are skipped for faster uploads.

### Step 3 - Verify Images in ECR Console

- Go to ECR Console → Select Repository
- You should see all pushed versions listed with:
  - Image size
  - Artifact type
  - Push timestamp
  - Image URI

### Step 4 - Scan Images (Optional)

You can manually scan images for vulnerabilities:

1. Select an image in ECR Console
2. Click **Scan**
3. Results show vulnerabilities by severity:
   - Critical
   - High
   - Medium
   - Low
   - Informational

### Step 5 -Deploy from ECR to EKS

#### Create Kubernetes Manifest

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: welcomeapp-ecr
  namespace: demo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: welcomeapp
  template:
    metadata:
      labels:
        app: welcomeapp
    spec:
      containers:
      - name: welcomeapp
        image: 123456789012.dkr.ecr.us-east-1.amazonaws.com/demo/welcomeapp:v3
        ports:
        - containerPort: 8080
```

#### Create Namespace (if needed)

```bash
kubectl create namespace demo
```

#### Apply Deployment

```bash
kubectl apply -f deployment-welcomeapp-ecr.yaml
```

#### Verify Deployment

```bash
kubectl get pods -n demo
kubectl describe pod <pod-name> -n demo
```

---

### IAM Role-Based ECR Access (No Image Pull Secrets Needed)

#### Why No Image Pull Secrets?

Normally, to pull from **private registries**, you'd need to:
1. Create Docker credentials secret
2. Add `imagePullSecrets` to manifest

**With ECR + IAM Roles:** This is **automatic and seamless**.

#### How It Works

| Component | Function |
|---|---|
| **EKS Node** | Runs as EC2 instance |
| **IAM Role** | Attached to EC2 instance |
| **ECR Policy** | Grants read access to ECR (attached to IAM role) |
| **Pod** | Uses node's IAM role to pull from ECR |

#### Required IAM Policy

Attached to the **EKS Node Group IAM Role**:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ecr:GetAuthorizationToken",
        "ecr:BatchGetImage",
        "ecr:GetDownloadUrlForLayer"
      ],
      "Resource": "*"
    }
  ]
}
```

#### Verify in AWS Console

1. Go to **EC2 Console**
2. Select an instance from your EKS node group
3. Check the **IAM role** attached
4. Verify the role has ECR read-only policy

#### Deployment Manifest (Simplified)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: welcomeapp
spec:
  replicas: 2
  template:
    spec:
      containers:
      - name: app
        image: 123456789012.dkr.ecr.us-east-1.amazonaws.com/demo/welcomeapp:v3
```

> **No `imagePullSecrets` needed!** IAM role handles authentication automatically.

---

## Summary

| Task | Key Points |
|---|---|
| **Create ECR Repository** | Use namespaced format (`demo/welcomeapp`), set tag mutability to immutable |
| **Grant IAM Permissions** | Attach ECR policy to the IAM user before accessing ECR |
| **Authenticate from Client** | Use `aws ecr get-login-password \| docker login` (temporary tokens, not static passwords) |
| **Tag Images** | Format: `<ecr-registry>/<namespace>/<repository>:<tag>` |
| **Push Images** | Use `docker push` after tagging; only changes are uploaded |
| **Scan Images** | Optional: manually scan for vulnerabilities in ECR Console |
| **Deploy to EKS** | Update image field in manifest to ECR URI |
| **IAM Role Access** | EKS nodes pull from ECR using attached IAM role — no secrets needed |

