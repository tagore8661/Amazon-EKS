# Creating an EKS Cluster

## Table of Contents
- [Introduction](#introduction)
- [Creating EKS Cluster via AWS Console](#creating-eks-cluster-via-aws-console)
- [EKS UI Walkthrough](#eks-ui-walkthrough)
- [Upgrading EKS Cluster Version](#upgrading-eks-cluster-version)
- [Accessing EKS Using kubectl and AWS CLI](#accessing-eks-using-kubectl-and-aws-cli)
- [Summary](#summary)

---

## Introduction

### Key Topics Covered
- Creating a cluster using AWS console
- Exploring the EKS control plane
- Navigating the ECS user interface
- Upgrading cluster version
- Accessing cluster using kubectl and AWS CLI

---

## Creating EKS Cluster via AWS Console

### Prerequisites
- AWS user with necessary permissions (Admin or specific IAM policies)
- Access to AWS Console

### Step 1: Initial Setup
1. Navigate to **Elastic Kubernetes Service** in AWS Console
2. Click **Create Cluster**

### Step 2: Configuration Options
**Two options available:**
- **Quick Configuration (EKS Auto Mode)**: AWS handles most settings automatically
- **Custom Configuration**: Manual control over settings (Used in this demo)

### Step 3: Cluster Configuration

#### Basic Settings
- **Cluster Name**: `demo-eks-cluster`
- **IAM Role**: `demo-eks-cluster-role` (pre-created with required permissions)
- **Kubernetes Version**: `1.31` (instead of latest 1.32 for upgrade demo purposes)

#### Support Information
- **Standard Support Duration**: 14 months from release date
- **Version 1.31 Support**: Until November 26, 2025
- Extended support available at additional cost after standard period

### Step 4: Cluster Access Configuration

#### Administrator Access
- **Option**: Allow cluster administrator access
- **Effect**: Grants current IAM user (Cherry) admin privileges
- **Alternative**: Disable for no default user access

#### Authentication Mode
- **Mode**: EKS API
- **Mechanism**: Authentication managed through AWS IAM
- Can provide access to other users (e.g., `Tagore`) later

#### Encryption
- **Default**: AWS managed keys
- **Optional**: Customer managed KMS keys for more control
- **Demo Setting**: Disabled (using AWS managed keys)

#### Additional Settings
- **Zonal Shift**: Disabled
- **Tags**: Added tag with key `Name` and value `EKS`

### Step 5: Networking Configuration

#### VPC and Subnets
- **VPC**: Default VPC in region
- **Subnets**: Public subnets (allows worker nodes internet access)

#### Security Groups
- **Cluster Security Group**: Auto-created by EKS for control plane-worker node communication
- **Additional Security Group**: Default VPC security group (permits port 443 access)
- **External Access**: Requires port 443 open for kubectl access from your machine

#### IP Addressing
- **Address Family**: IPv4
- **Service IP Range**: Custom CIDR `10.0.0.0/16`

#### Hybrid Networking
- **Status**: Disabled (not using hybrid EKS nodes)

### Step 6: Cluster Endpoint Access

#### Three Options Available:

**1. Public (Not Recommended)**
- Endpoint accessible over internet
- Worker node traffic leaves VPC
- Security concern

**2. Public and Private (Recommended - Selected)**
- Endpoint accessible from internet
- Worker node traffic stays within VPC
- Provides external access for administration
- Maintains secure internal communication

**3. Private**
- Endpoint accessible only within VPC
- Worker node traffic stays internal
- Highest isolation but difficult external access

#### SSH Access
- Port 22 can be opened later for SSH to worker nodes
- Should be done carefully for security reasons

### Step 7: Observability Configuration

#### Prometheus Metrics
- **Option**: Send metrics to Amazon Managed Service for Prometheus
- **Functionality**: Auto-configures Prometheus monitoring
- **Configuration**: Can create new or use existing workspace
- **Demo Setting**: Kept default settings

#### CloudWatch Telemetry
- **Add-on**: Amazon CloudWatch Observability
- **Functionality**: Sends application and infrastructure telemetry
- **Demo Setting**: Disabled (will be covere later)

#### Control Plane Logging
- **Status**: Enabled
- **Destination**: Amazon CloudWatch Logs
- **Purpose**: Monitor, audit, and troubleshoot EKS control plane

### Step 8: EKS Add-ons Configuration

#### Default Add-ons (Always Installed)

***AWS add-ons***

1. **VPC CNI**: Handles pod networking
2. **CoreDNS**: Provides internal domain name resolution
3. **Kube-proxy**: Manages network rules for service discovery and routing
4. **Node Monitoring**: Detects node health issues (must-have)

5. **Pod Identity**
- Assigns IAM permissions to pods
- Maps Kubernetes service accounts to IAM roles
- Allows secure API calls without hardcoded credentials

***Community add-ons***

1. **External DNS**
- Manages DNS records in Route 53 based on Kubernetes resources

2. **Metrics Server**
- Collects CPU and memory usage from nodes and pods

#### Add-ons Summary
- **Total**: 5 AWS-specific add-ons + 2 community add-ons
- **Community Add-ons**: External DNS and Metrics Server
- **Demo Setting**: Bare minimum setup, will enable more later

### Step 9: Review and Create
- All configurations reviewed
- Default settings kept for add-ons
- IAM roles for Pod Identity to be added when needed
- Cluster creation initiated (takes several minutes)

---

## EKS UI Walkthrough

### Cluster Overview Page
- **Cluster Name**: demo-eks-cluster
- **Status**: Active
- **Version**: Kubernetes 1.31
- **Support Until**: November 26, 2025
- **Upgrade Policy**: Standard (can switch to Extended after support end date)
- **Upgrade Available**: Yes (notification shown)

### Cluster Details Dashboard

#### Status Information
- **Upgrade Notification**: Newer version available but issues need resolution
- **Standard Support**: Ends November 26, 2025
- **Cluster Type**: EKS provisioned
- **Health Status**: No current issues
- **Compatibility**: Some checks needed for upgrades

#### Upgrade Issues
- **Issue Type**: EKS add-on version compatibility
- **Impact**: Doesn't affect current version but must be resolved before upgrading

### Key Cluster Information

### Overview Section

#### API and Security
- **API Server Endpoint**: URL for Kubernetes API server communication
- **Requirement**: kubeconfig file needed for authentication/authorization
- **Certificate Authority**: CA for secure API communications
- **OIDC Provider URL**: For identity and access management

#### Metadata
- **Created**: Timestamp of cluster provisioning
- **IAM Role**: Associated role with EKS cluster
- **Cluster URN**: AWS resource metadata
- **Platform Version**: AWS resource metadata

#### Configuration Status
- **EKS Auto Mode**: Disabled
- **Upgrade Type**: Standard
- **Zonal Shift**: Disabled

### Resources Section

#### Available Kubernetes Objects
- Pod Templates
- Pods
- Replica Sets
- Deployments
- Nodes
- Namespaces
- API Services
- Leases
- Runtime Classes
- ConfigMaps
- Secrets
- Services
- Endpoints
- Ingresses
- Authentication resources
- Storage resources

#### Current Pod Status
- **CoreDNS Pod**: Pending state
- **Metrics Server Pod**: Pending state
- **Reason**: No compute nodes available for scheduling

### Compute Section

#### Current State
- **Nodes**: None provisioned
- **Node Groups**: None defined (Auto Scaling Groups in backend)
- **Fargate Profiles**: None

#### Fargate Option
- Serverless option for running pods
- No EC2 node management required
- AWS provisions and manages infrastructure
- Billing based on pod-level usage

### Networking Section

#### Configuration Details
- **VPC**: Cluster deployment VPC
- **IP Address Family**: IPv4
- **Service IPv4 Range**: 10.0.0.0/16 (internal service communication)
- **Subnets**: Cluster deployment subnets
- **Primary Cluster Security Group**: Auto-created
- **Additional Security Group**: For external network access

#### Endpoint Access
- **API Server Endpoint Access**: Public and Private
- **Public Access Allowed**: 0.0.0.0/0 (open to all traffic)
- **Note**: Applies only to API endpoint, not worker nodes

### Add-ons Section

#### Installed Add-ons
- **Total**: 7 EKS add-ons
- **Status Issues**: CoreDNS and External DNS showing degraded
- **Compatibility**: Kube-proxy requires update for future versions

### Access Section

#### Current Access Grants
1. **Service Role**: Auto-granted to EKS system components
2. **Prometheus Scraper**: Can collect cluster metrics
3. **IAM User (kulbhushan)**: Admin permissions
4. **Cluster Insight Policy**: Limited access
5. **Amazon Prometheus Scraper Policy**: Metrics collection access

#### Planned Addition
- `Tagore` IAM user to be added for IAM access configuration

### Observability Section
- **Prometheus Scraper**: Configured and active
- **Function**: Fetches metrics and forwards to Amazon Managed Prometheus
- **Purpose**: Detailed monitoring and visualization

### Update History Section
- **Current State**: Empty (no version upgrades performed yet)

### Tags Section
- Custom tags applied to cluster visible here

---

## Upgrading EKS Cluster Version

### Pre-Upgrade Requirements

#### Version Information
- **Current Version**: 1.31
- **Target Version**: 1.32

#### Compatibility Issues to Resolve
Add-ons not compatible with version 1.32:
1. CoreDNS
2. Kube-proxy
3. VPC CNI Plugin

### Step 1: Update CoreDNS Add-on
1. Navigate to add-ons section
2. Select CoreDNS (current version: 1.11.3-eks-build.1)
3. Click **Edit** or **Update Now**
4. Select latest available version from dropdown
5. Save changes
6. Update starts in background

### Step 2: Update Kube-proxy Add-on
1. Select Kube-proxy add-on
2. Update to latest version
3. Save changes
4. Verification: Updates to latest version successfully

### Step 3: Update VPC CNI Add-on
1. Select VPC CNI add-on
2. Update to latest version
3. Save changes
4. Verification: Updates to latest version successfully

### Add-on Update Status

#### Issues Encountered
- **CoreDNS**: Stuck in "updating" state
- **Reason**: No worker nodes available for pod scheduling
- **Impact**: New CoreDNS pods cannot be scheduled

#### Successful Updates
- **Kube-proxy**: Updated successfully
- **VPC CNI**: Updated successfully

### Step 4: Initiate Cluster Upgrade

#### Process
1. Click **Upgrade** button on cluster dashboard
2. Select target version (1.32)
3. Type "confirm" to proceed
4. Monitor progress in cluster dashboard

#### Important Considerations
**Real-World Scenario Checklist:**
- Ensure applications are compatible with new Kubernetes version
- Check for deprecated features
- Test deployments before upgrading
- Review breaking changes in release notes

### Upgrade Completion

#### Results
- **Status**: Successfully upgraded to version 1.32
- **New Support End Date**: March 21, 2026 (Standard support)
- **Control Plane**: Upgraded successfully

#### Remaining Issues
- Some add-ons still in degraded state
- **Reason**: Absence of worker nodes
- **Resolution**: Issues will resolve after node creation in next section

---

## Accessing EKS Using kubectl and AWS CLI

### Initial Setup

#### Client Machine
- Pre-configured with kubectl and AWS CLI
- IAM user: `Tagore`
- Cluster name: `demo-eks-cluster`

### Step 1: Initial Connection Attempt

#### Command
```bash
kubectl version
```

#### Error Encountered
```
The connection to the server localhost:8080 was refused - did you specify the right host or port?
```

#### Root Cause
- kubeconfig file not configured on system
- kubectl doesn't know which cluster to connect to

### Step 2: Configure kubeconfig File

#### Command
```bash
aws eks update-kubeconfig --name demo-eks-cluster --region <region-name>
```

#### Result
- Command runs successfully
- Updates/creates kubeconfig file
- Sets new context pointing to cluster
- Success message: "Added new context <context-name> to <path>"

#### kubeconfig Location
```
/home/ubuntu/.kube/config
```

### Step 3: Verify kubeconfig File

#### Check File Creation
```bash
ls -ltr /home/ubuntu/.kube/config
```

#### File Contents
- Kubernetes API service endpoint (matches EKS console)
- User section for authentication
- Credentials managed by AWS
- Stored in `~/.aws/` directory

### Step 4: Second Connection Attempt

#### Command
```bash
kubectl version
```

#### New Error
```
You must be logged into the server (the server has asked for the client to provide credentials)
```

#### Analysis
- kubeconfig file is correct
- Successfully trying to connect to API server
- Current IAM user (`Tagore`) doesn't have cluster access

### Step 5: Grant IAM User Access

#### Check Current Access
On EKS console, Access tab shows:
- Original admin user
- Service role
- Prometheus user
- `Tagore` NOT listed

#### Grant Access via Console
1. Navigate to cluster in EKS console
2. Open **Access** tab
3. Scroll to **IAM Access Entries**
4. Click **Create Access Entry**
5. Select IAM principal ARN: `Tagore`
6. Type: **Standard**
7. Access level Policy Name: **Cluster Admin Policy**
8. Can grant namespace-level access if needed
9. Click **Add Policy**
10. Can add multiple policies if required
11. Click **Next**, then **Create**

### Step 6: Verify Access

#### Command
```bash
kubectl version
```

#### Result
- Both client and server versions displayed
- **Client Version**: v1.33.1
- **Server Version**: v1.32.5-eks
- Successfully connected to cluster

### Step 7: Check Cluster Resources

#### View Pods
```bash
kubectl get pods -A
```

#### Observation
- Several pods visible
- All pods in **Pending** state

#### Investigate Pod Status
```bash
kubectl describe pod <pod-name> -n <namespace>
```

#### Finding
- Message: "No nodes are available to schedule pods"

#### Verify Nodes
```bash
kubectl get nodes
```

#### Result
- No worker nodes attached to cluster
- Pods cannot run until worker nodes are added

### Connection Success Summary

#### Achievements
- Client machine fully configured
- `Tagore` IAM identity has cluster access
- kubeconfig file correctly set up using AWS CLI
- kubectl successfully communicates with EKS cluster

#### Pending Tasks
- Worker nodes need to be added
- Pods can only run after node creation

---

## Summary

### What We Accomplished

1. **Created EKS Cluster**
   - Used AWS Console with custom configuration
   - Configured networking, security, and observability
   - Set up add-ons and access policies

2. **Explored Control Plane**
   - Reviewed cluster details and metadata
   - Examined networking and security configurations
   - Checked add-ons and access policies

3. **Navigated EKS UI**
   - Explored all sections of cluster dashboard
   - Understood resources, compute, and networking tabs
   - Reviewed observability and access configurations

4. **Upgraded Cluster Version**
   - Updated incompatible add-ons (CoreDNS, Kube-proxy, VPC CNI)
   - Successfully upgraded from Kubernetes 1.31 to 1.32
   - Monitored upgrade process and verified completion

5. **Accessed Cluster**
   - Configured kubeconfig file using AWS CLI
   - Granted IAM user access via EKS console
   - Successfully connected using kubectl
   - Verified cluster communication

### Next Steps
- Create worker nodes/node groups
- Deploy applications
- Enable additional add-ons as needed
- Configure namespace-level access controls

---

## Important Commands Reference

### AWS CLI Commands
```bash
# Update kubeconfig
aws eks update-kubeconfig --name <cluster-name> --region <region>
```

### kubectl Commands
```bash
# Check version
kubectl version

# Get all pods across namespaces
kubectl get pods -A

# Describe a pod
kubectl describe pod <pod-name> -n <namespace>

# Get nodes
kubectl get nodes
```

### File Locations
```
kubeconfig: /home/ubuntu/.kube/config
AWS credentials: ~/.aws/
```
