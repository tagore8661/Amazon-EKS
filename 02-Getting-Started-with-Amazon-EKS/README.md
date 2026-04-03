# Getting Started with Amazon EKS

## Table of Contents
- [Introduction](#introduction)
- [Understanding EKS Architecture](#understanding-eks-architecture)
- [Creating IAM Cluster Role](#creating-iam-cluster-role)
- [Creating IAM NodeGroup Role](#creating-iam-nodegroup-role)
- [Installing CLI Tools and Setting Up EKS Access](#installing-cli-tools-and-setting-up-eks-access)
- [Summary](#summary)

---

## Introduction

Setting up everything needed to start working with Amazon EKS.

**Topics Covered:**
- Brief overview of EKS architecture
- Creating required IAM roles
- Installing tools: `kubectl` and AWS CLI

---

## Understanding EKS Architecture

### Kubernetes Cluster Basics

**Node**: A physical or virtual machine (like an EC2 instance) that runs workloads in the form of pods.

**Kubernetes Cluster**: A group of nodes working together to run applications.

### Types of Nodes

1. **Control Plane** - The brain of the cluster
   - Makes all decisions
   - Handles scheduling, scaling, health checks and more..
   - It tells the Cluster what needs to happen and when.

2. **Worker Nodes** - Actually run the applications

### Control Plane Components

- **API Server**: Main entry point for all commands and communication
- **Controller Manager**: Watches the cluster and maintains desired state
- **Scheduler**: Decides where new pods should run
- **etcd**: Key-value store that tracks entire cluster state

### Node Components

Each node (control plane or worker) includes:
- **Kubelet**: Talks to API server, manages containers, reports back
- **Kube Proxy**: Manages networking rules for pod communication
- **Container Runtime**: (e.g., containerd) Actually runs containers

### Amazon EKS Benefits

**AWS Fully Manages the Control Plane:**
- No need to install, upgrade, or scale
- Automatically distributed across different availability zones
- High availability and fault tolerance
- No single point of failure
- Control plane in private subnets managed by AWS
- Backed by network load balancer

### kubectl Communication Flow

1. `kubectl` reads `kubeconfig` file
2. Locates EKS API server endpoint and cluster CA
3. Generates authentication token
4. Sends secure HTTPS request to Kubernetes API server
5. Request is TLS encrypted
6. Uses CA certificate to verify server identity

### Data Plane (Worker Nodes) Options

#### Standard Mode
**Self-Managed Nodes:**
- You maintain nodes yourself
- Full control over configuration

**Managed Node Groups:**
- AWS helps with scaling and updates
- Pre-configured with EKS optimized AMIs
- Includes Kubelet and container runtime

#### Auto Mode
- AWS manages both control plane and worker nodes
- Automatically provisions EC2 instances
- Handles scaling, patching, and security
- Minimal operational overhead

#### Fargate
- Serverless option for running pods
- No EC2 instance management
- AWS provisions compute per pod
- Pay only for what you use
- **Limitations:**
  - No support for DaemonSets
  - No privileged containers
  - No AWS EBS as PVT

### EKS Architecture Components

#### Region Selection
- Determines physical location of control plane
- Choose based on requirements (e.g., US East 2 - Ohio)

#### IAM Cluster Role
Required policies:
- **Amazon EKS Cluster Policy**: Grants control plane permissions for EC2, networking, autoscaling, load balancers
- **Amazon VPC Resource Controller Policy** (Optional): Manages IP addresses and ENIs automatically

#### VPC and Networking
- Control plane runs in AWS-managed VPC
- Select your VPC and at least 2 subnets in different AZs
- Subnets used for:
  - Launching worker nodes/pods
  - Assigning IP addresses
  - Hosting ENIs for secure control plane connection
- Ensures high availability and service discovery

#### Security Groups
- **Cluster Security Group**: Automatically created for control plane ↔ worker node communication
- **Additional Security Groups** (Optional): Custom access rules for API server
- **Important**: Port 443 must be open for `kubectl` access from laptop

#### API Server Access Options

1. **Public Access**: Reachable over internet (convenient but less secure)
2. **Private Access**: Limited to internal VPC (most secure)
3. **Public and Private** (Recommended):
   - Publicly accessible for admin tasks
   - Worker nodes communicate privately with control plane

### EKS Add-ons

**Core Add-ons:**
- **VPC CNI Plugin**: Pod networking, each pod gets own IP
- **CoreDNS**: Service discovery within cluster
- **Kube Proxy**: Internal service routing
- **Metrics Server**: Resource usage data for horizontal pod autoscaling
- **Pod Identity Agent**: Assigns IAM roles to individual pods (installed externally)

**Additional Tools:**
- **External DNS**: Manages DNS records automatically
- **Node Monitoring Agent**: Tracks node health

### Command Line Tools

#### kubectl
- Standard Kubernetes CLI
- Runs commands like checking pod status, deploying applications
- Sends requests to Kubernetes API server

#### AWS CLI
- Generates `kubeconfig` file
- Contains everything `kubectl` needs:
  - API endpoint
  - Authentication settings
  - Certificates

**IAM User Setup:**
- Using IAM user (e.g., "Think Next user")
- Initially has EKS full access
- Additional permissions granted as needed

### Observability
- Metrics collection
- Control plane logging
- Enable only when necessary (additional costs)

### Access Control

#### Authentication
- Handled using AWS IAM
- IAM access entry required for users
- Verifies if user is allowed to connect

#### Key Components
1. **Kubernetes API Server Endpoint**: Address `kubectl` uses to connect
2. **OIDC Provider URL**:
   - Authenticates pods
   - Allows service accounts to assume IAM roles
   - Secure access to AWS services without storing credentials

---

## Creating IAM Cluster Role

### Steps

1. Navigate to IAM Dashboard → Roles
2. Click **Create Role**
3. Select **AWS Service**
4. Choose **EKS** → **EKS Cluster** option
5. Click **Next**
6. Default policy attached: **Amazon EKS Cluster Policy**
7. Click **Next**
8. Name: `demo-eks-cluster-role`
9. Click **Create Role**

### Add Additional Policy

1. Click on role name
2. Click **Add Permissions** → **Attach Policies**
3. Search: **Amazon EKS VPC Resource Controller**
4. Select and click **Add Permissions**

**Result**: Cluster role ready with both required policies.

---

## Creating IAM NodeGroup Role

### Purpose
- Manages node groups (groups of EC2 instances)
- Uses Auto Scaling Group for scaling worker nodes

### Steps

1. IAM Dashboard → Roles
2. Click **Create Role**
3. Choose **AWS Service**
4. Select **EC2** (role assumed by EC2 worker nodes)
5. Click **Next**

### Attach Three Policies

1. **Amazon EKS CNI Policy**: Allows nodes Configures elastic network interfaces
2. **Amazon EKS Worker Node Policy**: Similar to cluster policy, but for worker nodes
3. **Amazon EC2 Container Registry Read Only**: Pull container images from ECR

6. Click **Next**
7. Name: `demo-eks-nodegroup-role`
8. Click **Create Role**

**Result**: Node group role ready with all three required policies.

---

## Installing CLI Tools and Setting Up EKS Access

### Installing kubectl (Debian-based Linux)

```bash
# Update system and install dependencies
sudo apt-get update
# apt-transport-https may be a dummy package; if so, you can skip that package
sudo apt-get install -y apt-transport-https ca-certificates curl gnupg

# Download GPG key
# If the folder `/etc/apt/keyrings` does not exist, it should be created before the curl command, read the note below.
# sudo mkdir -p -m 755 /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.35/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

sudo chmod 644 /etc/apt/keyrings/kubernetes-apt-keyring.gpg # allow unprivileged APT programs to read this keyring

# Add Kubernetes repository
# This overwrites any existing configuration in /etc/apt/sources.list.d/kubernetes.list
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.35/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo chmod 644 /etc/apt/sources.list.d/kubernetes.list   # helps tools such as command-not-found to work correctly

# Update and install
sudo apt-get update
sudo apt-get install -y kubectl

# Verify installation
kubectl version --client
```

**Version Compatibility**: Using kubectl 1.33.x is compatible with EKS cluster running Kubernetes 1.31.

### Installing AWS CLI (Linux)

```bash
# Download installer
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"

# Unzip (ensure unzip is installed)
unzip awscliv2.zip

# Run installer
sudo ./aws/install

# If updating existing installation
sudo ./aws/install --update

# Verify installation
aws --version
```

### AWS CLI Configuration

```bash
# Configure AWS CLI
aws configure
```

**Required Information:**
- AWS Access Key ID
- AWS Secret Access Key
- Default region: `us-east-2` (Ohio)
- Default output format: `json`

### IAM User Setup

**User**: Tagore

**Permissions**:
- Customer inline policy with `EKS-FullAccess`
- Additional permissions added as needed

**Result**: Both tools installed and ready for use.

---

## Summary

### What We Accomplished

1. **Architecture Understanding**
   - Learned EKS architecture fundamentals
   - Understood control plane and data plane interaction

2. **IAM Cluster Role**
   - Created role for EKS control plane
   - Enables secure management of AWS resources
   - Attached policies:
     - Amazon EKS Cluster Policy
     - Amazon VPC Resource Controller Policy

3. **IAM NodeGroup Role**
   - Created role for EC2 worker nodes
   - Attached policies:
     - Amazon EKS CNI Policy
     - Amazon EKS Worker Node Policy
     - Amazon EC2 Container Registry Read Only

4. **Command Line Tools**
   - Installed `kubectl` for cluster interaction
   - Installed AWS CLI for kubeconfig management
   - Configured AWS CLI with IAM user credentials

### Next Steps
Ready to create and interact with EKS cluster.