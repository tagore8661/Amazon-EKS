# Scaling Node Groups - Cluster Autoscaler (Amazon EKS)

---

## Table of Contents

- [Introduction](#introduction)
- [Understanding Cluster Autoscaler for Node Groups](#understanding-cluster-autoscaler-for-node-groups)
- [Created IAM Policy & Role for Cluster Autoscaler](#created-iam-policy--role-for-cluster-autoscaler)
- [Enabling and Observing Cluster Autoscaler in Action](#enabling-and-observing-cluster-autoscaler-in-action)
- [Summary](#summary)

---

## Introduction

- Learn how Kubernetes **scales worker nodes automatically** using the **Cluster Autoscaler** in Amazon EKS.
### Key Topics Covered
  - How Cluster Autoscaler works.
  - Setting up a **secure IAM role**.
  - Deploying using **Auto Discovery** method.
  - Watching nodes **scale up/down** based on demand.

---

## Understanding Cluster Autoscaler for Node Groups

### ASG vs Cluster Autoscaler

| Feature | ASG Alone | With Cluster Autoscaler |
|---|---|---|
| Replace unhealthy instances | Yes | Yes |
| Scale up when pods need resources | No | Yes |
| Scale down idle nodes | No | Yes |

### Node Group Scaling Configuration (Set Earlier)

| Parameter | Value |
|---|---|
| Desired Capacity | 2 nodes |
| Min | 2 nodes |
| Max | 5 nodes |

> **Important:** Configuring ASG min/max/desired alone does **NOT** enable auto scaling based on pod demand. That requires the **Cluster Autoscaler** component.

### How Cluster Autoscaler Works

- Maintained by **SIG Auto Scaling**.
- Runs as a **Deployment** inside the cluster - typically in `kube-system` namespace.
- Uses **leader election** - only one replica acts at a time (not horizontally scalable).
- **Simulates** node additions/removals before applying them.
- Respects ASG **min and max** limits - only adjusts `desired` capacity within those bounds.

#### Scale Up Trigger

- Pods are in **Pending** state due to insufficient resources.
- Autoscaler increases ASG `desired` count to bring in new nodes.

#### Scale Down Trigger

- Nodes are **underutilized** and can be safely drained.
- Autoscaler decreases ASG `desired` count to save costs.

### EKS Managed Node Groups - Key Benefits

- **Automatic tagging** for Cluster Autoscaler discovery.
- Supports **graceful node draining** during scale down.
- Simplifies lifecycle management (graceful termination + in-place upgrades).
- Integrates seamlessly with auto scaling tools.

### Mixed Instance Policy Note

> For mixed instance policies, the autoscaler simulates node resources using the **first instance type** listed in the launch template. Keep this in mind when optimizing for cost or availability.

### Auto Discovery Tags (Applied Automatically by EKS)

| Tag Key | Purpose |
|---|---|
| `k8s.io/cluster-autoscaler/enabled` | Marks ASG as eligible for autoscaling |
| `k8s.io/cluster-autoscaler/<cluster-name>` | Associates ASG with specific EKS cluster |

---

## Created IAM Policy & Role for Cluster Autoscaler

### Step 1 - Create IAM Policy

- Reference: [Official Cluster Autoscaler GitHub Docs (AWS)](https://github.com/kubernetes/autoscaler)
- Two policy options available:
  - **Full Feature Policy** - Recommended (used in this demo)
  - Minimal Permission Policy
- Policy grants access to: **EC2**, **EC2 Auto Scaling**, **Amazon EKS**
- Policy Name: `AWSAutoscalerControllerIAMPolicy`

### Step 2 - Create IAM Role

- Trusted Entity Type: **Web Identity**
- OIDC Provider: The one configured earlier for the EKS cluster
- Audience: `sts.amazonaws.com`
- Attach Policy: `AWSAutoscalerControllerIAMPolicy`
- Role Name: `EKSClusterAutoscaler`

### Step 3 - Edit Trust Relationship Policy

- After role creation, edit the **Trust Policy**.
- Replace `aud` with `sub` in the condition.
- Add the subject value in this format:

```
system:serviceaccount:kube-system:cluster-autoscaler
```

- This ensures **only** the Cluster Autoscaler service account can assume this role.

> **Note:** If you rename the service account in the Kubernetes manifest, update this trust policy accordingly.

---

##  Enabling and Observing Cluster Autoscaler in Action

### Deployment Options Available

| Method | Description |
|---|---|
| **Auto Discovery** (used here) | Automatically detects eligible ASGs via tags |
| Multiple ASGs | Manually define multiple auto scaling groups |
| Single ASG | Manually define a single auto scaling group |

### Step 1 - Download the YAML File

```bash
# Download auto-discovery manifest
curl -o cluster-autoscaler-autodiscover.yaml <raw-github-url>
```

### Step 2 - Edit the YAML File (Two Changes Required)

#### Change 1 - Annotate the Service Account

In the `ServiceAccount` section, add:

```yaml
annotations:
  eks.amazonaws.com/role-arn: <IAM_ROLE_ARN>
```

This instructs the service account to use the IAM role via **IRSA (IAM Roles for Service Accounts)**.

#### Change 2 - Set Cluster Name for Auto Discovery

In the `Deployment` section, under containers command, update:

```yaml
- --node-group-auto-discovery=asg:tag=k8s.io/cluster-autoscaler/enabled,k8s.io/cluster-autoscaler/<YOUR-CLUSTER-NAME>
```

Replace `<YOUR-CLUSTER-NAME>` with your actual EKS cluster name (e.g., `demo-eks-cluster`).

### Step 3 - Apply the Manifest

```bash
kubectl apply -f cluster-autoscaler-autodiscover.yaml
```

Resources created:
- `ServiceAccount`
- `ClusterRole`
- `ClusterRoleBinding`
- `Role`
- `RoleBinding`
- `Deployment`

### Step 4 - Verify Pod is Running

```bash
kubectl get pods -n kube-system
```

```bash
kubectl describe pod <cluster-autoscaler-pod> -n kube-system
```

Check environment variables to confirm:
- AWS Region is set correctly
- IAM Role ARN is present
- Web Identity Tokens are picked up automatically

---

### Scale Out Demo

**Initial state:** 2 nodes, 1 replica

#### Scale to 5 replicas

```bash
kubectl scale deployment cluster-autoscaler-demo --replicas=5
```

- All 5 pods scheduled on existing 2 nodes.

#### Scale to 10 replicas

```bash
kubectl scale deployment cluster-autoscaler-demo --replicas=10
```

- Some pods enter **Pending** state.
- `kubectl describe pod <pending-pod>` shows: `Insufficient CPU`.
- Cluster Autoscaler detects unschedulable pods → triggers **scale up**.
- Node count increases from 2 → 3 to accommodate workload.

#### Scale to 20 replicas

```bash
kubectl scale deployment cluster-autoscaler-demo --replicas=20
```

- More pods enter Pending state.
- Two more nodes are added → total **5 nodes**.
- All 20 pods eventually reach Running state.

```bash
kubectl get nodes -w
```

---

### Scale In Demo

#### Reduce to 2 replicas

```bash
kubectl scale deployment cluster-autoscaler-demo --replicas=2
```

- Extra pods are terminated.
- Nodes do **NOT** get removed immediately (cooldown period applies).

#### Graceful Termination Process

1. **Scheduling Disabled** - No new pods scheduled on idle nodes.
2. **Not Ready** - Node marked as not ready.
3. **Drained** - Running pods evicted.
4. **Removed** - ASG desired count reduced.

```bash
# Monitor node status
kubectl get nodes -w

# Monitor autoscaler logs
kubectl logs -f -n kube-system <cluster-autoscaler-pod>
```

> **Cooldown Period:** Default is **5 to 10 minutes**. Nodes are not removed immediately after scale down to ensure stability.

#### Final State

- Back to **2 nodes** matching the number of replicas.
- Confirms **scale in** works as expected.

---

## Summary

| Step | Action |
|---|---|
| 1 | Created **IAM Policy** with full EC2/ASG/EKS permissions |
| 2 | Created **IAM Role** with OIDC Web Identity trust |
| 3 | Edited **Trust Relationship** to restrict access to `cluster-autoscaler` service account |
| 4 | Deployed Cluster Autoscaler using **Auto Discovery** method |
| 5 | Annotated Service Account with IAM Role ARN |
| 6 | Updated cluster name in the auto discovery command |
| 7 | Observed **scale out** - nodes added when pods were Pending |
| 8 | Observed **scale in** - nodes removed after graceful termination cooldown |
