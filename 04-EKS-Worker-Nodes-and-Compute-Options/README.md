# EKS Worker Nodes and Compute Options

## Table of Contents

- [Introduction](#introduction)
- [Comparing Managed Node Groups and Fargate Profiles](#comparing-managed-node-groups-and-fargate-profiles)
- [Creating Managed Node Groups](#creating-managed-node-groups)
- [Creating IAM Role for Fargate Profile](#creating-iam-role-for-fargate-profile)
- [Adding a Fargate Profile to EKS](#adding-a-fargate-profile-to-eks)
- [Deploying Pods to Fargate](#deploying-pods-to-fargate)
- [Adding aws-logging ConfigMap](#adding-aws-logging-configmap)
- [Summary](#summary)

---

## Introduction

This section covers how EKS runs workloads using two primary compute options:
- **Managed Node Groups**
- **Fargate Profiles**

### Key Topics Covered
- Comparison between Node Groups and Fargate
- Creating and configuring Managed Node Groups
- Setting up Fargate Profiles
- Deploying pods on both compute options
- Enabling logging for Fargate pods using AWS logging ConfigMap

---

## Comparing Managed Node Groups and Fargate Profiles

### Compute & Scaling

**Standard Mode (Node Groups)**
- Launches EC2 instances using launch templates and auto scaling groups
- Shared compute environment (multiple pods per instance)
- Nodes added based on scaling rules

**Fargate Mode**
- No EC2 instances to manage
- AWS creates a lightweight VM (node) per pod
- Serverless compute provisioning

### Infrastructure & Responsibility

**Node Groups**
- Full control over instance type, updates, OS patching, and security
- More flexible but requires more management
- You handle infrastructure

**Fargate**
- AWS manages everything below the pod level
- Define CPU and memory requirements
- Focus on applications, not infrastructure

### Workload Compatibility

**Node Groups Support:**
- Complex and demanding workloads
- DaemonSets
- Custom AMIs
- EBS volumes
- GPU-based nodes
- Long-lived workloads

**Fargate Support:**
- Stateless applications
- Microservices
- Batch jobs
- EFS (supported)

**Fargate Limitations:**
- No EBS support
- No GPU support
- No DaemonSets
- Only private subnets

### Security & IAM Roles

**Node Groups**
- You manage EC2 instance security
- Manage IAM roles for nodes
- Node instance role connects to cluster, pulls images, accesses AWS services

**Fargate**
- AWS handles host security and isolation
- Pod-level IAM permissions only
- Uses pod execution role

### Cost Model

**Node Groups**
- Charged for entire EC2 instance (regardless of utilization)
- Cost-effective for long-lived, high-density workloads
- No extra charges for EKS managed node groups
- Pay for: EC2, EBS, Networking

**Fargate**
- Pay-per-pod model
- Billed based on CPU and memory requested in pod spec
- Per-second billing with 1-minute minimum
- Ideal for short-lived or bursty workloads

### Networking & Subnets

**Node Groups**
- Pods inherit EC2 network settings
- Public subnets: pods get public IPs, direct internet access
- Private subnets: internet via NAT gateway

**Fargate**
- Only supports private subnets
- No subnets with direct route to internet gateway
- Pods do NOT receive public IPs
- All outbound traffic through NAT gateway

### When to Use What?

**Use Node Groups When:**
- Need custom workloads
- Require GPUs
- Need DaemonSets
- Need EBS volumes
- Want more control over infrastructure

**Use Fargate When:**
- Want simplicity and speed
- Don't want to manage EC2
- Running stateless or short-lived apps
- Need on-demand scaling

**Hybrid Approach:** Many teams use both - Node Groups for heavy/specialized workloads, Fargate for lightweight services.

---

## Creating Managed Node Groups

### Architecture Overview

When creating a node group, EKS automatically sets up:
1. **Launch Template** - Defines EC2 instance configuration (AMI, instance type, disk size)
2. **Auto Scaling Group** - Manages scaling, instance replacement, lifecycle

### Configuration Settings

**AMI Type:** Amazon Linux (EKS optimized)
- Pre-installed: kubelet, container runtime, EKS networking plugins

**Instance Type:** t3.medium

**Capacity Type:** On-Demand (or Spot)

**Disk Size:** 20 GB (default)

**Scaling Configuration:**
- Desired capacity: 2 nodes
- Minimum: 2 nodes
- Maximum: 5 nodes

**Additional Features:**
- Auto-repair enabled (unhealthy nodes automatically replaced)
- Launched into public subnets (for dev environments)

### Demonstration Steps

1. Navigate to EKS cluster → Compute tab
2. Click "Add Node Group"
3. Name: `demo-node-group-1`
4. Select IAM role (created earlier)
5. Add node label: `node-group=demo-node-group-1`
6. No taints (allows scheduling of core pods)
7. Add tag: `Name=EKS`
8. Configure compute settings
9. Set scaling configuration
10. Choose update strategy (number-based: 1 node)
11. Enable auto-repair
12. Select public subnets
13. Create node group

### Auto-Created Resources

After node group creation, automatically created:
- Launch template (visible in EC2 dashboard)
- Auto Scaling Group (configured with desired/min/max)
- EC2 instances (joined to cluster)

### Verification Commands

```bash
# Check nodes
kubectl get nodes

# Check node labels
kubectl get nodes --show-labels

# Check pending pods
kubectl get pods -A | grep Pending

# Describe pod to see scheduling issue
kubectl describe pod <pod-name> -n <namespace>

# Check all pods status
kubectl get pods -A
```

### System Pods Deployed

Once nodes are ready, these DaemonSets run automatically:
- **aws-node** - VPC CNI networking
- **eks-node-monitoring-agent** - Monitoring capabilities
- **eks-pod-identity-agent** - IAM roles for service accounts

---

## Creating IAM Role for Fargate Profile

### Purpose
Fargate profile requires a dedicated IAM role separate from node group role.

### Steps to Create Role

1. Go to IAM → Roles → Create role
2. **Trusted Entity Type:** AWS Service
3. **Use Case:** EKS → EKS Fargate pod
4. Click Next
5. **Default Policy Added:** `AmazonEKSFargatePodExecutionRolePolicy`
6. **Additional Policy:** `CloudWatchLogsFullAccess` (for pod logging)
7. **Role Name:** `demo-eks-fargate-role`
8. **Tag:** `Name=EKS`
9. Create role

### Attached Policies

1. **AmazonEKSFargatePodExecutionRolePolicy** (default)
   - Allows Fargate to pull container images
   - Manages pod lifecycle

2. **CloudWatchLogsFullAccess** (added)
   - Enables Fargate pods to write logs to CloudWatch

---

## Adding a Fargate Profile to EKS

### Key Concepts

**Fargate Profile:** Tells EKS which pods should be scheduled on Fargate using selectors.

**Selectors Include:**
- Namespace (required)
- Labels (optional)

### Configuration Requirements

- **Subnet Type:** Private subnets ONLY
- **Cannot be edited:** After creation, must create new profile for changes
- **Maximum Selectors:** 5 per profile

### Demonstration Steps

1. Navigate to EKS cluster → Compute tab
2. Click "Add Fargate Profile"
3. **Name:** `demo-eks-fargate`
4. **Pod Execution Role:** Select `demo-eks-fargate-role`
5. **Subnets:** Select private subnets (1, 2, 3)
6. **Tag:** `Name=EKS`

### Pod Selector Configuration

**Selector 1:**
- **Namespace:** `demo-ns`
- **Label:** `type=fargate`
- Only pods with this label in `demo-ns` scheduled on Fargate

**Selector 2:**
- **Namespace:** `database-ns`
- **No label criteria**
- ALL pods in this namespace scheduled on Fargate

### Important Notes

- Namespace must exist before profile applies
- No label selector = all pods in namespace use Fargate
- Specific labels = only matching pods use Fargate

---

## Deploying Pods to Fargate

### Current State Check

```bash
# Check existing nodes (should show node group nodes only)
kubectl get nodes

# Create required namespace
kubectl create namespace demo-ns
```

### Test 1: Pod Without Label

```bash
# Create pod without label (schedules on node group)
kubectl run nginx --image=nginx:latest -n demo-ns

# Check pod placement
kubectl get pods -n demo-ns -o wide
```

**Result:** Pod runs on existing worker nodes (node group)

### Test 2: Pod With Fargate Label

```bash
# Create pod with matching label
kubectl run httpd --image=httpd:latest -n demo-ns --labels=type=fargate

# Monitor pod creation
kubectl get pods -n demo-ns -o wide
```

**Observation:**
- Pod shows "Nominated Node" (Fargate provisioning)
- Status: Pending (while Fargate node creates)
- New Fargate node appears in node list
- Pod transitions to Running

### Verify Fargate Node

```bash
# Check nodes (should show new Fargate node)
kubectl get nodes

# Describe Fargate node
kubectl describe node <fargate-node-name>
```

**Fargate Node Characteristics:**
- Unique name and IP
- Managed by Fargate infrastructure
- Architecture: AMD64
- OS: Linux
- Compute Type: Fargate

### Test 3: Scale with Another Pod

```bash
# Create another pod with Fargate label
kubectl run httpd-1 --image=httpd:latest -n demo-ns --labels=type=fargate

# Watch nodes
kubectl get nodes -w

# Watch pods
kubectl get pods -n demo-ns -o wide
```

**Result:** Another Fargate node provisioned automatically

### Default Fargate Resources

**Per Pod:**
- **vCPU:** 0.25
- **Memory:** 0.5 GB

Can be overridden in pod spec if needed.

### Verification in Console

1. Navigate to EKS cluster → Compute tab
2. See all nodes (node group + Fargate)
3. Click Resources tab
4. Select namespace to view pods
5. Check pod details for compute configuration

---

## Adding aws-logging ConfigMap

### Problem Statement

Fargate pods show message: **"LoggingDisabled: LOGGING_CONFIGMAP_NOT_FOUND"**

```bash
# Check pod events
kubectl describe pod httpd -n demo-ns
```

### Solution Overview

**Built-in Log Router:** Fargate has Fluent Bit built-in (no sidecar needed)

**Requirement:** Create ConfigMap to configure log routing to CloudWatch

### ConfigMap Requirements

- **Name:** `aws-logging` (exactly)
- **Namespace:** `aws-observability` (exactly)
- **Size Limit:** Maximum 5,300 characters

### Step 1: Create Namespace

```yaml
# aws-observability-namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: aws-observability
  labels:
    name: aws-observability: enabled
```

```bash
kubectl apply -f aws-observability-namespace.yaml
```

### Step 2: Create ConfigMap

```yaml
# aws-logging-cloudwatch-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: aws-logging
  namespace: aws-observability
data:
  flb_log_cw: "true"  # Enable CloudWatch logging
  filters.conf: |
    [FILTER]
        Name parser
        Match *
        Key_name log
        Parser crio
    [FILTER]
        Name kubernetes
        Match kube.*
        Merge_Log On
        Keep_Log Off
        Buffer_Size 0
        Kube_Meta_Cache_TTL 300s
  output.conf: |
    [OUTPUT]
        Name cloudwatch_logs
        Match kube.*
        region us-east-2  # Update to your region
        log_group_name demo-eks-cluster-application-logs
        log_stream_prefix from-fluent-bit-
        auto_create_group true
        log_retention_days 60
  parsers.conf: |
    [PARSER]
        Name        crio
        Format      Regex
        Regex       ^(?<time>[^ ]+) (?<stream>stdout|stderr) (?<logtag>P|F) (?<log>.*)$
        Time_Key    time
        Time_Format %Y-%m-%dT%H:%M:%S.%L%z
```

### Key Configuration Parameters

- **flb_log_cw:** `true` (enables log shipping)
- **region:** Match your EKS cluster region
- **log_group_name:** Custom name for application logs
- **log_stream_prefix:** Organize log streams
- **auto_create_group:** `true` (creates log group automatically)
- **log_retention_days:** 60 days

### Apply ConfigMap

```bash
# Apply the ConfigMap
kubectl apply -f aws-logging-cloudwatch-configmap.yaml

# Verify ConfigMap created
kubectl get configmap -n aws-observability
# Output: NAME          DATA   AGE
#         aws-logging   4      10s
```

### Test Logging

```bash
# Delete existing pod
kubectl delete pod httpd -n demo-ns

# Recreate with Fargate label
kubectl run httpd --image=httpd:latest -n demo-ns --labels=type=fargate

# Describe pod to verify logging enabled
kubectl describe pod httpd -n demo-ns
```

**Expected Event:** "Successfully enabled logging for pod"

### Verify in CloudWatch

1. Go to CloudWatch → Log Groups
2. Find log group: `demo-eks-cluster-application-logs`
3. Open log streams
4. See entries: `from-fluent-bit-kube.var.log.containers.<namespace>.<pod-name>...`
5. View actual log data

### Control Plane vs Application Logs

**Control Plane Logs:** `/aws/eks/demo-eks-cluster`
- API server
- Authenticator
- Controller manager
- Scheduler
- Audit logs

**Application Logs:** `demo-eks-cluster-application-logs`
- Pod logs from Fargate
- Managed by Fluent Bit
- Custom log group

### Additional Testing

```bash
# Create nginx pod with logging
kubectl run nginx --image=nginx:latest -n demo-ns --labels=type=fargate

# Verify logging enabled
kubectl describe pod nginx -n demo-ns

# Check CloudWatch for new log stream
# New stream will appear for nginx pod
```

### Benefits

- Logs accessible via `kubectl logs` AND CloudWatch
- Longer retention in CloudWatch
- Centralized visibility
- Essential for production workloads

---

## Summary

### Key Learnings

This section covered two key compute options in Amazon EKS:

**1. Managed Node Groups**
- Set up EC2-based worker nodes
- Configure auto-scaling groups
- Full control over infrastructure
- Suitable for complex workloads

**2. Fargate Profiles**
- Run pods without managing infrastructure
- Serverless compute provisioning
- Ideal for stateless applications
- Dynamic resource allocation

**3. Workload Deployment**
- Deploy pods on both compute types
- Use labels and namespaces for scheduling
- Understand resource allocation differences

**4. Logging Configuration**
- Configure pod-level logging for Fargate
- Use AWS logging ConfigMap in `aws-observability` namespace
- Enable Fluent Bit log routing to CloudWatch
- Maintain centralized log visibility

---

## Important Commands Reference

```bash
# Node and Pod Management
kubectl get nodes
kubectl get pods -A
kubectl get pods -n <namespace> -o wide
kubectl describe pod <pod-name> -n <namespace>

# Namespace Operations
kubectl create namespace <namespace-name>
kubectl get namespaces

# ConfigMap Operations
kubectl get configmap -n aws-observability
kubectl apply -f <configmap-file.yaml>

# Pod Creation
kubectl run <pod-name> --image=<image:tag> -n <namespace>
kubectl run <pod-name> --image=<image:tag> -n <namespace> --labels=key=value

# Pod Deletion
kubectl delete pod <pod-name> -n <namespace>

# Watch Resources
kubectl get nodes -w
kubectl get pods -n <namespace> -w
```
