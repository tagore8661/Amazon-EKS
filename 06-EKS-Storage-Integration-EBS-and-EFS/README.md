# EKS Storage Integration EBS and EFS

## Table of Contents
1. [Introduction](#introduction)
2. [Amazon EBS (Elastic Block Store)](#amazon-ebs-elastic-block-store)
   - [Overview & How It Works](#overview--how-it-works)
   - [IAM Configurations](#iam-configurations)
   - [EBS CSI Driver Installation](#ebs-csi-driver-installation)
   - [Persistent Storage with PVC](#persistent-storage-with-pvc)
   - [StatefulSets with ClaimTemplates](#statefulsets-with-claimtemplates)
3. [Amazon EFS (Elastic File System)](#amazon-efs-elastic-file-system)
   - [Overview & Shared Access](#overview--shared-access)
   - [EFS Configuration Setup](#efs-configuration-setup)
   - [Using EFS with Multiple Pods](#using-efs-with-multiple-pods)
4. [Summary](#summary)

---

## Introduction

### What You'll Learn
- How to use **Amazon EBS** and **EFS** for persistent storage in EKS
- EBS for **single pod storage** using PVCs and StatefulSets
- EFS for **shared storage** across multiple pods and Fargate

### The Storage Challenge
**Question:** What happens when your application needs to store data?
- Persist user information in a database
- Retain uploaded files after pod termination
- Maintain state across pod restarts

### Kubernetes Storage Types
1. **Ephemeral Storage** - Data lost when pod stops
2. **Persistent Storage** - Data survives beyond pod lifecycle

---

## Amazon EBS (Elastic Block Store)

### Overview & How It Works

#### What is EBS?
Amazon EBS provides **block-level storage** ideal for:
- Databases (MySQL, PostgreSQL)
- Applications needing high IOPS and throughput
- Random read/write operations

#### Key Components

**1. Amazon EBS CSI Driver**
- CSI = Container Storage Interface
- Acts as a bridge between Kubernetes and AWS EBS
- Allows dynamic provisioning and attachment of EBS volumes to pods

**Driver Functions:**
- Dynamically provision and attach EBS volumes to pods
- Integrate with native Kubernetes objects (PV, PVC, Storage Classes)
- Manage storage volume lifecycle automatically

**2. IAM Roles for Service Accounts (IRSA)**
- CSI driver needs permissions to make AWS API calls
- Uses OIDC identity provider configured in EKS cluster
- Enables secure, least-privileged access

**How IRSA Works:**
1. Create IAM role with required EBS permissions
2. Define custom trust policy for Kubernetes service account
3. Use OIDC identity provider from EKS cluster
4. Annotate service account to associate with IAM role

#### StatefulSets for Stateful Applications

**When to Use StatefulSets:**
- Databases
- Message queues
- Caching systems

**StatefulSet Features:**
- Stable, persistent identity for each pod
- Dedicated PVC for each pod using volume claim templates
- Easier to match existing volumes to replacement pods
- Ideal for stable network identity and dedicated storage per pod

#### Important PVC Concepts

**Key Points:**
- Volume and PVC remain intact even after pod deletion
- Default binding mode: **Wait for First Consumer**
  - Volume not provisioned until pod is scheduled
- Access mode: **ReadWriteOnce** (RWO)
  - Volume mounted by only one node at a time

---

### IAM Configurations

#### Step 1: Create IAM Role

**Custom Trust Policy Requirements:**
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {
      "Federated": "arn:aws:iam::{ACCOUNT_ID}:oidc-provider/oidc.eks.{REGION}.amazonaws.com/id/{OIDC_ID}"
    },
    "Action": "sts:AssumeRoleWithWebIdentity",
    "Condition": {
      "StringEquals": {
        "oidc.eks.{REGION}.amazonaws.com/id/{OIDC_ID}:aud": "sts.amazonaws.com",
        "oidc.eks.{REGION}.amazonaws.com/id/{OIDC_ID}:sub": "system:serviceaccount:kube-system:ebs-csi-controller-sa"
      }
    }
  }]
}
```

**Replace:**
- `{ACCOUNT_ID}` - Your AWS Account ID
- `{REGION}` - Your AWS region (e.g., us-east-2)
- `{OIDC_ID}` - OIDC provider ID from EKS cluster

**Policy to Attach:**
- `AmazonEBSCSIDriverPolicy` (AWS Managed Policy)

**Recommended Role Name:**
- `AmazonEKS_EBS_CSI_DriverRole`

**Add tags:**
- Key: `Name`
- Value: `EKS`

#### Step 2: Add Identity Provider

**Configuration:**
- Provider Type: **OpenID Connect**
- Provider URL: OIDC provider URL from EKS cluster
- Audience: `sts.amazonaws.com`
- Tags:
  - Key: `Name`
  - Value: `EKS`

**Purpose:**
- Authenticates service account in EKS cluster
- Allows service account to assume IAM role

---

### EBS CSI Driver Installation

#### Installation Steps

**1. Install Add-on**
```bash
# Navigate to EKS Cluster → Add-ons → Get More Add-ons
# Search for: Amazon EBS CSI Driver
# Select IAM role created earlier
```

**2. Verify Service Account**
```bash
kubectl get serviceaccounts -n kube-system | grep -i ebs
```
Expected output: `ebs-csi-controller-sa` `ebs-csi-node-sa`

**3. Annotate Service Account**
```bash
kubectl annotate serviceaccount -n kube-system ebs-csi-controller-sa \
  eks.amazonaws.com/role-arn=arn:aws:iam::{ACCOUNT_ID}:role/{ROLE_NAME}
```

**4. Verify Annotation**
```bash
kubectl describe serviceaccount -n kube-system ebs-csi-controller-sa
```
Check for the IAM role ARN in annotations field.

---

### Persistent Storage with PVC

#### PVC Manifest Structure

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: demo-pvc
spec:
  storageClassName: gp2
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

**Key Fields:**
- `storageClassName: gp2` - Auto-created by AWS
- `accessModes: ReadWriteOnce` - Single node access
- `storage: 1Gi` - Requested volume size

#### Storage Class Verification

```bash
kubectl get storageclasses.storage.k8s.io
```

**Important Notes:**
- EBS CSI driver **cannot be used with Fargate**
- Must use **managed node groups**
- Binding mode: **WaitForFirstConsumer**
- Reclaim policy: **Delete** (PV and EBS volume deleted with PVC)

#### Pod Using PVC

```yaml
# pod-with-pvc.yaml
apiVersion: v1
kind: Pod
metadata:
  name: demo-pod
spec:
  containers:
    - name: app
      image: nginx
      volumeMounts:
        - name: storage
          mountPath: /opt
  volumes:
    - name: storage
      persistentVolumeClaim:
        claimName: demo-pvc
```

#### Workflow

1. **Apply PVC**
   ```bash
   kubectl apply -f pvc-csi-ebs.yaml
   kubectl get pvc
   ```
   Status: **Pending** (waiting for consumer)

2. **Create Pod**
   ```bash
   kubectl apply -f pod-with-pvc.yaml
   ```

3. **PVC Status Changes to Bound**
   ```bash
   kubectl get pvc
   kubectl get pv
   
   # Describe PV to see details
   kubectl describe pv <pv-name>
   ```

4. **Verify in AWS Console**
   - EC2 → EBS → Volumes
   - 1 GB volume created and attached to node

5. **Cleanup Behavior**
   - Delete pod: Pod will be removed but PVC remains
   - Delete PVC: PV and EBS volume automatically deleted

---

### StatefulSets with ClaimTemplates

#### Why Volume Claim Templates?

**Scenario:** Multiple pods, each needing dedicated storage
- Manual PVC creation is inefficient
- Common with databases, message queues
- Each pod maintains its own persistent data

**Solution:** StatefulSet with `volumeClaimTemplates`

#### StatefulSet Manifest

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  serviceName: "nginx"
  replicas: 1
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
          image: nginx
          ports:
            - containerPort: 80
              name: web
          volumeMounts:
            - name: www
              mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
    - metadata:
        name: www
      spec:
        storageClassName: gp2
        accessModes:
          - ReadWriteOnce
        resources:
          requests:
            storage: 1Gi
```

#### Key Advantages

- **Automatic PVC creation** for each pod
- **Stable, persistent identity** per pod
- **Dedicated EBS volume** per pod
- **Dynamic provisioning** on scale-up

#### Scaling Behavior

**Scale to 2 replicas:**
```bash
kubectl scale statefulset web --replicas=2
```
- New pod created on different node
- New PVC automatically created
- New EBS volume provisioned and attached

**Scale to 3 replicas:**
```bash
kubectl scale statefulset web --replicas=3
```
- Third pod may share node with first pod
- Third EBS volume attached to that node

#### Deletion Behavior

**Important:** PVCs are NOT automatically deleted

```bash
# Delete StatefulSet
kubectl delete statefulset web

# Pods are gone
kubectl get pods

# But PVCs remain
kubectl get pvc

# Manual cleanup required
kubectl delete pvc --all
```

**Why?** PVC = Persistent Volume Claim - designed to retain data

---

## Amazon EFS (Elastic File System)

### Overview & Shared Access

#### Limitations of EBS
- **One volume per node** at a time
- **No shared access** across multiple pods
- **Not supported on Fargate**

#### What is EFS?

Amazon EFS is a **fully managed, serverless, elastic file system**

**Key Benefits:**
- Mounted by multiple pods simultaneously
- Accessible across different nodes
- Pay only for storage used
- No capacity planning needed

#### EFS vs EBS Comparison

| Feature | EBS | EFS |
|---------|-----|-----|
| **Type** | Block Storage | File Storage |
| **Access** | One pod at a time | Multiple pods simultaneously |
| **Access Mode** | ReadWriteOnce (RWO) | ReadWriteMany (RWX) |
| **Compute Support** | Node Groups only | Node Groups + Fargate |
| **Use Cases** | Databases, single-instance apps | Shared workloads, CMS, multi-pod apps |

#### Perfect Use Cases for EFS

- Content Management Systems (WordPress, Drupal)
- Developer Tools (Jira, Git repositories)
- Shared Notebook Systems (Jupyter)
- Home directories for multiple users
- Any workload requiring concurrent multi-pod access

#### How do we Use EFS in K8s?
```bash
IAM Role -> Pod Identity -> Amazon EFS CSI driver add-on -> EFS file system -> StorageClass -> PersistentVolumeClaim (ReadWriteMany)
```

---

### EFS Configuration Setup

#### Key Differences from EBS

**Authentication:**
- EBS uses **IRSA** (IAM Roles for Service Accounts)
- EFS uses **Pod Identity** (newer, simpler method)

**File System:**
- EBS: Volumes auto-created on demand
- EFS: **File system must exist beforehand**

#### Setup Steps

**1. Create IAM Role**
- Trusted Entity: **AWS Service → EKS**
- Authentication: **EKS Pod Identity** (not OIDC)
- Policy: `AmazonEFSCSIDriverPolicy`
- Role Name: `AmazonEKS_EFS_CSI_DriverRole`
- Add tages: `Name = EKS`

**2. Install EFS CSI Driver**
```bash
# EKS Console → Cluster → Add-ons → Get More Add-ons
# Search: Amazon EFS CSI Driver
# Add-on access (Authentication): EKS Pod Identity 
# Select IAM role created earlier
```

**3. Verify Service Accounts**
```bash
kubectl get serviceaccount -n kube-system
```
Two new service accounts created for EFS CSI driver

**EFS CSI Driver Components:**
- `efs-csi-controller-sa` - Manages EFS file systems
- `efs-csi-node-sa` - Mounts EFS on nodes

**4. Create EFS File System**

**Prerequisites:**
- Must be in same **VPC** as EKS cluster
- Accessible from **subnets** used by Node Groups/Fargate
- Security group allows inbound on **port 2049** (NFS)

**Mount Targets:**
- Configure in required subnets
- Ensure network connectivity from EKS nodes

---

### Using EFS with Multiple Pods

#### Step 1: Create Storage Class

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: efs-sc
provisioner: efs.csi.aws.com
volumeBindingMode: Immediate
```

**Key Difference:**
- Binding Mode: **Immediate** (not WaitForFirstConsumer)
- PVC bound as soon as created

```bash
kubectl apply -f storage-class-efs.yaml
kubectl get storageclasses.storage.k8s.io
```

#### Step 2: Create Persistent Volume

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: efs-pv
spec:
  capacity:
    storage: 5Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  storageClassName: efs-sc
  csi:
    driver: efs.csi.aws.com
    volumeHandle: {FILE_SYSTEM_ID}
```

**Important Fields:**
- `accessModes: ReadWriteMany` - Multi-pod access and simultaneous read/write operations
- `volumeHandle` - EFS File System ID
- `reclaimPolicy: Retain` - Volume not auto-deleted

```bash
kubectl apply -f persistent-volume-efs.yaml
kubectl get pv
```

#### Step 3: Create PVC

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: efs-claim
spec:
  storageClassName: efs-sc
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 5Gi
```

```bash
kubectl apply -f pvc-efs.yaml
kubectl get pvc
```
Status: **Bound** (immediately)

#### Step 4: Deploy Workload on Node Groups

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: deployment-efs
spec:
  replicas: 3
  selector:
    matchLabels:
      app: demo
  template:
    metadata:
      labels:
        app: demo
    spec:
      containers:
        - name: app
          image: busybox
          command: ["/bin/sh"]
          args:
            - "-c"
            - "while true; do echo $(date) - $(hostname) >> /data/out1.txt; sleep 5; done"
          volumeMounts:
            - name: persistent-storage
              mountPath: /data
      volumes:
        - name: persistent-storage
          persistentVolumeClaim:
            claimName: efs-claim
```

**What Happens:**
- 3 pods created across nodes
- All write to `/data/out1.txt`
- Data from all 3 pods in single shared file

**Verification:**
```bash
kubectl apply -f deployment-efs.yaml
kubectl get pods
kubectl exec -it {POD_NAME} -- /bin/sh
cat /data/out1.txt
```
You'll see entries from all 3 pods!

#### Step 5: Scale Up

```bash
kubectl scale deployment deployment-efs --replicas=5
```
- 2 additional pods start
- All 5 pods write to same file
- Simultaneous multi-pod access confirmed

#### Step 6: Deploy on Fargate

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: deployment-fargate-efs
spec:
  replicas: 3
  selector:
    matchLabels:
      app: fargate-demo
  template:
    metadata:
      labels:
        app: fargate-demo
        type: fargate
    spec:
      containers:
        - name: app
          image: busybox
          command: ["/bin/sh"]
          args:
            - "-c"
            - "while true; do echo $(date) - FARGATE-$(hostname) >> /data/out1.txt; sleep 5; done"
          volumeMounts:
            - name: persistent-storage
              mountPath: /data
      volumes:
        - name: persistent-storage
          persistentVolumeClaim:
            claimName: efs-claim
```

**Result:**
- Fargate pods dynamically provisioned
- Same EFS volume mounted
- Both Node Group and Fargate pods writing to same file!

**Verification:**
```bash
kubectl apply -f deployment-fargate-efs.yaml
kubectl exec -it {FARGATE_POD_NAME} -- /bin/sh
cat /data/out1.txt
```
Entries from both deployments visible!

#### Cleanup

```bash
# Delete deployments
kubectl delete deployment deployment-efs
kubectl delete deployment deployment-fargate-efs

# Verify pods terminating
kubectl get pods

# Delete PVC
kubectl delete pvc efs-claim

# Delete PV
kubectl delete pv efs-pv

# Storage class remains
# EFS file system NOT deleted
```

---

## Summary

### Amazon EBS
**Block-level storage** for individual pods
Ideal for **databases** and high-performance workloads
Access Mode: **ReadWriteOnce** (single node)
Configured using **PVCs** and **StatefulSets**
Supports **Node Groups only** (not Fargate)
Volumes **auto-provisioned** dynamically

### Amazon EFS
**Shared file storage** across multiple pods
Access Mode: **ReadWriteMany** (multi-node)
Supports **Node Groups + Fargate**
Perfect for **CMS, shared workloads, collaborative tools**
File system must **pre-exist**
Volume binding: **Immediate**
