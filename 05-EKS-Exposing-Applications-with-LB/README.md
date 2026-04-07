# EKS Exposing Applications with Load Balancers

## Table of Contents

- [Introduction](#introduction)
- [Understanding Load Balancing in EKS](#understanding-load-balancing-in-eks)
- [Expose Application Using Service Type LoadBalancer](#expose-application-using-service-type-loadbalancer)
- [Using Annotations to Create Network Load Balancer](#using-annotations-to-create-network-load-balancer)
- [Cleaning Up Existing Resources](#cleaning-up-existing-resources)
- [Summary](#summary)

---

## Introduction

This section covers how to expose applications in EKS using the Load Balancer service type.

### Key Topics Covered
- Default setup creating Classic Load Balancer (CLB)
- Advanced setup using annotations to create Network Load Balancer (NLB)
- Subnet tagging and configuration
- Health checks and traffic distribution
- Resource cleanup

---

## Understanding Load Balancing in EKS

### The Problem

When applications are deployed inside Kubernetes clusters:
- Pods run in isolated environments
- Internal pod-to-pod communication works within the cluster
- External users on the internet cannot directly reach pods
- **Solution: Use Load Balancers to expose applications**

### How Load Balancers Work

#### Basic Architecture
```
Internet Traffic
        ↓
   Load Balancer
        ↓
    ┌───┴───┐
    ↓       ↓
  Pod 1   Pod 2   (Healthy pods)

  Pod 3 (Unhealthy - traffic not routed)
```

**Load Balancer Service Type:**
- Requests creation of external load balancer from cloud provider (AWS)
- Becomes entry point for application traffic
- Distributes traffic across healthy backend pods
- Automatically detects unhealthy pods and routes around them

**Benefits:**
- Internet accessibility
- High availability and resilience
- Automatic failover
- Traffic distribution

### Load Balancer Types in AWS

#### Classic Load Balancer (CLB)

**Characteristics:**
- AWS default when no annotations specified
- **Status**: Maintenance mode (critical bug fixes only)
- **Functionality**: Basic load balancing
- **Limitation**: Limited feature set
- **Layer**: Works at Layer 4 & 7

**When to Use:**
- Legacy applications
- Simple HTTP/HTTPS traffic
- No specific advanced requirements

**Creation:**
```bash
kubectl expose deployment {APP_NAME} --type=load-balancer
```

---

#### Network Load Balancer (NLB)

**Characteristics:**
- **Modern Generation**: Newer, performance-focused
- **Layer**: Layer 4 (Transport Layer)
- **Protocol**: TCP/UDP based applications
- **Performance**: Ultra high-performance
- **Use Case**: Extreme performance requirements

**Benefits Over Classic:**
- Higher throughput
- Lower latency
- Better for real-time applications
- More control and customization

**Creation:**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: app-service-nlb
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: nlb
    service.beta.kubernetes.io/aws-load-balancer-scheme: internet-facing
spec:
  type: LoadBalancer
  selector:
    app: app-name
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
```

---

#### Application Load Balancer (ALB)

**Characteristics:**
- **Layer**: Layer 7 (Application Layer)
- **Protocol**: HTTP/HTTPS
- **Routing**: Advanced path-based and host-based routing
- **Not covered in this class** but available option

---

### Subnet Tagging Requirement

**Why Tagging is Essential:**
- AWS needs to know which subnets to use for load balancer deployment
- Load balancers must be in public subnets for internet-facing traffic

**Tag Format:**
```
Key: kubernetes.io/cluster/<cluster-name>
Value: shared (or owned)
```
- `shared`: Subnet can be used by multiple clusters
- `owned`: Subnet owned exclusively by this cluster
- Either value works - indicates subnet belongs to cluster

**Example:**
```
Key: kubernetes.io/cluster/demo-eks-cluster
Value: shared
```

**Application:**
- Public subnets: For internet-facing load balancers
- Private subnets: For internal load balancers
- Tag all three public subnets for high availability

### Automatic Resource Management

**When You Delete a Service:**
- Kubernetes automatically deletes associated AWS load balancer
- No manual cleanup needed
- Managed by EKS cluster IAM role

**IAM Requirement:**
- Cluster IAM role must have `AmazonEKSVPCResourceController` policy
- Enables automatic load balancer creation and deletion

### Health Checks

**Dual Layer Protection:**
1. **Kubernetes Health Checks:** Pod-level monitoring
2. **AWS ELB Health Checks:** Load balancer validation
   - Second layer of protection
   - Ensures traffic only to healthy pods
   - Catches issues Kubernetes hasn't detected yet
   - Configurable for specific needs

### Architecture Overview

**Without Load Balancer:**
- Worker nodes (EC2)
- Pods inside nodes
- No external access

**With Load Balancer:**
- Internet → Load Balancer (AWS provisioned)
- Load Balancer → Healthy Pods
- Automatic traffic distribution
- Redundancy across pods

---

## Expose Application Using Service Type LoadBalancer

### Prerequisites

**Clean State:** Delete any existing resources
```bash
# Check existing resources
kubectl get all -n demo-ns

# Delete all pods
kubectl delete pod --all -n demo-ns
```

### Step 1: Deploy Application

**Deployment Manifest Content:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demo-welcome-app
  namespace: demo-ns
  labels:
    app: welcome-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: welcome-app
  template:
    metadata:
      labels:
        app: welcome-app
    spec:
      containers:
      - name: java-welcome-app
        image: ghcr.io/kul-samples/javawelcomeapp:v3
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 512Mi
        ports:
          - name: tomcat
            containerPort: 8080
```

**Deploy Application:**
```bash
# Apply deployment
kubectl apply -f deployment-welcome-app.yaml

# Verify deployment
kubectl get all -n demo-ns
```

**Verification:**
- Deployment created
- 2 replicas specified
- 2 pods running and ready
- Pods labeled with `app: welcome-app`

### Step 2: Create LoadBalancer Service (Default - Classic LB)

**Command:**
```bash
kubectl expose deployment demo-welcome-app \
  --name=welcome-app-service \
  --port=8080 \
  --type=LoadBalancer \
  -n demo-ns
```

**What This Does:**
- Creates Kubernetes service
- Type: LoadBalancer
- Targets port 8080 on pods
- Requests AWS to provision load balancer

### Step 3: Check Service Status

```bash
# Check service
kubectl get service -n demo-ns

# Watch for external IP assignment
kubectl get service -w -n demo-ns
```

**Initial State:**
- Cluster IP: Assigned
- External IP: **Pending** (waiting for load balancer provisioning)
- Type: LoadBalancer

```bash
# Describe service to see events
kubectl describe service welcome-app-service -n demo-ns
```
**Problem:** External IP remains pending with error
```
Error syncing load balancer: failed to ensure load balancer
```

**Cause:** EKS doesn't know which subnets to use for load balancer

### Step 4: Tag Public Subnets

**Required for All Public Subnets (3 recommended for HA):**

1. Go to VPC Console → Subnets
2. Identify public subnets (typically have route to IGW)
3. For each subnet:
   - Select subnet
   - Click "Manage Tags"
   - Add tag:
     - **Key:** `kubernetes.io/cluster/demo-eks-cluster`
     - **Value:** `shared`
4. Save tags

**Why All Three Subnets:**
- High availability
- Ensures load balancer can be placed optimally
- Redundancy if one subnet becomes unavailable

### Step 5: Verify Service External IP

```bash
# Check service status
kubectl get service -n demo-ns
```

**After Tagging:**
- External IP: **Assigned** (DNS name of load balancer)
- Status: Ready to receive traffic

**Example Output:**
```
NAME                    TYPE           CLUSTER-IP      EXTERNAL-IP
welcome-app-service    LoadBalancer   10.100.x.x      <load-balancer-dns>:8080
```

### Step 6: Verify Load Balancer in AWS Console

**EC2 Dashboard → Load Balancers:**
- New classic load balancer appears
- Type: Classic
- Status: InService
- Registered instances: EKS worker nodes (2)
- Listener: Port 8080 → Backend port 3684

**Target Instances Tab:**
- Shows registered nodes
- Health check status: Healthy
- Ready to receive traffic

### Step 7: Access Application

**Get Load Balancer DNS:**
```bash
# From kubectl output
kubectl get service -n demo-ns

# Copy EXTERNAL-IP value (DNS name)
```

**Initial Access:**
```
http://<load-balancer-dns>:8080/welcome
```

**Result:** 404 Error (incorrect path)

**Check Pod Logs:**
```bash
kubectl logs <pod-name> -n demo-ns
```

**Finding:** Application uses context path `/kmayer` not `/welcome`

**Correct Access:**
```
http://<load-balancer-dns>:8080/kmayer
```

**Result:** Application loads successfully
```
Welcome message from host pod
Pod Name: demo-welcome-app-xxxxx
Pod IP: 10.x.x.152
```

### Step 8: Verify Load Balancing

**Check Pod IPs:**
```bash
kubectl get pods -n demo-ns -o wide
```

**Test Load Distribution:**
1. Refresh browser multiple times
2. Pod IP in response changes between requests
3. Confirms traffic rotating between pods

**Example Results:**
- Request 1 → Pod IP: 10.x.x.152
- Request 2 → Pod IP: 10.x.x.153
- Request 3 → Pod IP: 10.x.x.152

### Summary of Classic Load Balancer Setup

**Advantages:**
- Simple, minimal configuration
- Works out of the box with proper tagging
- Suitable for basic applications

**Limitations:**
- Older generation load balancer
- Maintenance mode only
- Limited advanced features

---

## Using Annotations to Create Network Load Balancer

### Why Use Network Load Balancer?

- Modern alternative to Classic LB
- Better performance
- Layer 4 (TCP/UDP) optimization
- Suitable for high-performance applications

### Step 1: Prepare Service Manifest with Annotations

**Service Manifest (NLB):**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: welcome-app-service-nlb
  namespace: demo-ns
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
    service.beta.kubernetes.io/aws-load-balancer-scheme: "internet-facing"
    service.beta.kubernetes.io/aws-load-balancer-target-type: "ip"
spec:
  type: LoadBalancer
  selector:
    app: welcome-app
  ports:
  - port: 80              # LB listens on port 80
    targetPort: 8080      # Routes to pod port 8080
    protocol: TCP
```

### Key Annotations Explained

**1. Load Balancer Type:**
```yaml
service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
```
- Values: `"nlb"`, `"alb"`, or default (classic)
- Default: Classic Load Balancer
- Our choice: NLB (Network Load Balancer)

**2. Load Balancer Scheme:**
```yaml
service.beta.kubernetes.io/aws-load-balancer-scheme: "internet-facing"
```
- `"internet-facing"`: Public, accessible from internet
- `"internal"`: Private, accessible only within VPC
- Uses public/private subnets accordingly

**3. Target Type:**
```yaml
service.beta.kubernetes.io/aws-load-balancer-target-type: "ip"
```
- `"ip"`: Route directly to pod IPs (recommended for Fargate)
- `"instance"`: Route to nodes first, then pods (default)
- `"ip"` provides lower latency and direct pod targeting

### Additional Annotations Reference

**Optional Annotations:**
```yaml
# Health check configuration
service.beta.kubernetes.io/aws-load-balancer-healthcheck-protocol: "HTTP"
service.beta.kubernetes.io/aws-load-balancer-healthcheck-path: "/health"
service.beta.kubernetes.io/aws-load-balancer-healthcheck-port: "8080"

# Specific subnets
service.beta.kubernetes.io/aws-load-balancer-subnets: "subnet-xxx,subnet-yyy"

# SSL/TLS
service.beta.kubernetes.io/aws-load-balancer-backend-protocol: "https"
service.beta.kubernetes.io/aws-load-balancer-ssl-ports: "443"
```

### Step 2: Deploy NLB Service

```bash
# Apply the service manifest
kubectl apply -f welcome-app-service-nlb.yaml

# Verify service created
kubectl get service -n demo-ns
```

**Expected Output:**
```
NAME                        TYPE           CLUSTER-IP      EXTERNAL-IP
welcome-app-service-nlb     LoadBalancer   10.100.x.x      <nlb-dns>
```

**Differences from Classic LB:**
- Service port: 80 (not 8080)
- Node port: Assigned automatically
- New DNS name (NLB endpoint)

### Step 3: Verify NLB in AWS Console

**EC2 Dashboard → Load Balancers:**

**Load Balancer Details:**
- Type: **Network Load Balancer** (not Classic)
- Scheme: Internet-facing
- Listener: Port 80 → Target group
- Status: Provisioning

**Target Group Details:**
1. Click on target group
2. View targets (initially initializing)
3. Targets: Pod IPs (because we used IP target type)
4. Health check: HTTP on port 8080

**Health Check Progress:**
- Initially: Initializing
- Wait 1-2 minutes
- Targets: Healthy state
- Ready for traffic

### Step 4: Access NLB Application

```bash
# Get NLB DNS name
kubectl get service -n demo-ns
```

**Access Application:**
```
http://<nlb-dns>/kmayer
```

**Result:**
- Application loads successfully
- Same welcome app as before
- Served through NLB

**Verify Pod Response:**
- Click "Get Hostname"
- Shows pod IP and name
- Confirms NLB routing to pods

### Step 5: Test NLB Load Distribution

**Refresh Browser Multiple Times:**
```
Request 1 → Pod 1 IP
Request 2 → Pod 2 IP
Request 1 → Pod 1 IP
```

**Verification:**
- Responses alternate between pods
- NLB distributes traffic correctly
- Both targets healthy

### Comparison: Classic LB vs NLB

| Feature | Classic LB | NLB |
|---------|-----------|-----|
| Layer | Layer 4/7 | Layer 4 |
| Performance | Standard | Ultra-high |
| Status | Maintenance mode | Current |
| Configuration | Simple | Annotations-based |
| Target Type | Instance only | Instance or IP |
| Health Checks | Basic | Advanced |
| SSL/TLS | Supported | Supported |
| Use Case | Legacy | Modern apps |

### Common Annotation Use Cases

**For Fargate Pods:**
```yaml
annotations:
  service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
  service.beta.kubernetes.io/aws-load-balancer-target-type: "ip"
  # IP target type required for Fargate
```

**For Internal Load Balancer:**
```yaml
annotations:
  service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
  service.beta.kubernetes.io/aws-load-balancer-scheme: "internal"
  # Routes to private subnets only
```

**With Custom Health Checks:**
```yaml
annotations:
  service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
  service.beta.kubernetes.io/aws-load-balancer-healthcheck-protocol: "HTTP"
  service.beta.kubernetes.io/aws-load-balancer-healthcheck-path: "/health"
  service.beta.kubernetes.io/aws-load-balancer-healthcheck-port: "8080"
```

---

## Cleaning Up Existing Resources

### Purpose

Remove all created resources to maintain clean environment and avoid confusion.

### Resources to Clean Up

**Check Current State:**
```bash
kubectl get all -n demo-ns
```

**Existing Resources:**
- 1 Deployment: `demo-welcome-app`
- 2 Services: `welcome-app-service` (Classic), `welcome-app-service-nlb` (NLB)
- 2 Pods: Running from deployment

### Step 1: Delete Deployment

```bash
# Delete using manifest file
kubectl delete -f deployment-welcome-app.yaml

# Verify deletion
kubectl get deployment -n demo-ns
```

**Result:** Deployment removed, replica set removed, pods terminated

### Step 2: Delete Services

**Delete Classic LB Service:**
```bash
# Direct deletion
kubectl delete service welcome-app-service -n demo-ns
```

**Delete NLB Service:**
```bash
# Delete using manifest
kubectl delete -f welcome-service-nlb.yaml

# Alternative: Direct deletion
kubectl delete service welcome-app-service-nlb -n demo-ns
```

**Verification:**
```bash
kubectl get service -n demo-ns
```

**Result:** No services remaining

### Step 3: Verify AWS Cleanup

**EC2 Dashboard → Load Balancers:**

1. Refresh page
2. Both load balancers are **gone**
3. Classic LB: Removed
4. NLB: Removed

**Why Automatic Cleanup?**
- Kubernetes controllers manage AWS resources
- Deleting service → Automatic LB deletion
- Handled by EKS IAM role permissions
- No manual infrastructure cleanup needed

### Step 4: Final Verification

**Check Kubernetes Resources:**
```bash
# Pods in namespace
kubectl get pods -n demo-ns
# Result: No pods

# Services in namespace
kubectl get svc -n demo-ns
# Result: No services

# All resources
kubectl get all -n demo-ns
# Result: Empty namespace
```

---

## Summary

This section covered:

1. **Load Balancer Fundamentals**
   - Why load balancers are needed
   - How EKS integrates with AWS LBs
   - Subnet tagging requirements

2. **Classic Load Balancer Deployment**
   - Creating service with type LoadBalancer
   - Tagging public subnets
   - Accessing application via DNS
   - Verifying traffic distribution

3. **Network Load Balancer Deployment**
   - Using annotations for NLB
   - Configuring target type (IP vs Instance)
   - Scheme configuration (internet-facing vs internal)
   - Performance benefits over Classic LB

4. **Resource Cleanup**
   - Deleting services and deployments
   - Automatic AWS load balancer cleanup
   - Maintaining clean environment

5. **Best Practices**
   - Use NLB over Classic LB for new applications
   - Use IP target type for Fargate pods
   - Tag all public subnets for high availability
   - Document custom annotations for team use

