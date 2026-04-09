# Managing Networking and Ingress

## Table of Contents

- [Introduction](#introduction)
- [Understanding Ingress Controllers and ALB Setup](#understanding-ingress-controllers-and-alb-setup)
- [Creating IAM Policy & Role](#creating-iam-policy--role)
- [Deploying ALB Ingress Controller Resources](#deploying-alb-ingress-controller-resources)
- [Deploying ALB Ingress Controller to Route External Traffic](#deploying-alb-ingress-controller-to-route-external-traffic)
- [Summary](#summary)

---

## Introduction

- Setting up the **AWS Load Balancer Controller** in EKS to automatically provision and manage an **Application Load Balancer (ALB)**

### Key Topics Covered
  - Controller's role in Kubernetes
  - Creating required **IAM resources**
  - Deploying the controller using **Helm**
  - Applying **CRDs (Custom Resource Definitions)**
  - How external traffic is routed to services inside the cluster

---

## Understanding Ingress Controllers and ALB Setup

### What is the Application Load Balancer (ALB)?

- Operates at **Layer 7** of the OSI model (Application Layer)
- Ideal for routing web traffic based on:
  - URLs
  - Hostnames
  - Paths

### AWS Load Balancer Controller

- A **Kubernetes controller** developed by AWS
- Automates provisioning and configuration of AWS Elastic Load Balancers (ALBs and NLBs)
- Based on Kubernetes resources like **Ingress** and **Service**
- Originally started as the **AWS ALB Ingress Controller** by Ticketmaster and CoreOS
- Later donated to the **Kubernetes SIG AWS** community

### How It Works

When you define an Ingress resource with correct annotations, the AWS Load Balancer Controller:

1. Watches the Kubernetes API for the new Ingress resource
2. Automatically provisions a new **Application Load Balancer**
3. Configures **listeners** (typically on port 80 and 443)
4. Creates **target groups** for backend services
5. Sets up **routing rules** based on hostnames or paths defined in the Ingress spec

> The ALB can route HTTP/HTTPS traffic to pods running on EC2 worker nodes or AWS Fargate.
> No manual ALB creation is required.

### ALB Deployment Modes

| Mode | Description | Use Case |
|------|-------------|----------|
| **Public Subnet** | Internet-facing ALB | Apps accessible from the internet |
| **Private Subnet** | Internal ALB | Apps accessible only within the VPC |

### Demo Application - Lucky Draw

- Two services exposed through one ALB:
  - `/participate` - routes to the **Participate** backend (submit entries)
  - `/submissions` - routes to the **Submissions** backend (view entries)
- One ALB with a single DNS entry serves multiple routes in the same cluster

### ALB Traffic Modes

| Mode | How It Works | Best For |
|------|-------------|----------|
| **Instance Mode** | Traffic forwarded to NodePort on EC2 workers, then to pods | EC2-based workloads (default) |
| **IP Mode** | Traffic routed directly to pod IPs, bypassing NodePort | AWS Fargate or hybrid infra (more efficient) |

> IP Mode requires annotating the Ingress resource accordingly.

### Steps Overview (Hands-On)

1. Create a **custom IAM policy and role**
   - Role trusts the OIDC provider linked to the EKS cluster
   - Trust relationship allows only the Load Balancer Controller's service account in `kube-system` namespace
2. Create the **Kubernetes Service Account** and annotate it with the IAM role
3. Add the **AWS EKS Helm chart repository** and install the AWS Load Balancer Controller
4. Apply required **CRDs** (TargetGroupBinding and IngressClassParams)
5. Deploy services and define an **Ingress resource** with ALB-specific annotations:
   - Schema: internet-facing
   - Specific subnet IDs
   - Target type: IP
   - Backend protocol: HTTP

---

## Creating IAM Policy & Role

### Step 1 - Create the IAM Policy

1. Navigate to **AWS Console > IAM > Policies**
2. Click **Create Policy**
3. Switch to the **JSON** tab in the policy editor
4. Paste the predefined JSON with all necessary ALB Ingress Controller permissions
5. Click **Next**
6. Name the policy: `AWSLoadBalancerControllerIAMPolicy`
7. Optionally add a description and tag (`name: eks`)
8. Click **Create Policy**

> The policy is created as a **Customer Managed Policy**.

### Step 2 - Create the IAM Role

1. Navigate to **IAM > Roles > Create Role**
2. Trusted entity type: **Web Identity**
3. Identity Provider: Select the **OIDC provider** created earlier
4. Audience: `sts.amazonaws.com`
5. Attach the policy: `AWSLoadBalancerControllerIAMPolicy`
6. Name the role: `AmazonEKSLoadBalancerControllerRole`
7. Add tag (`Name: EKS`)
8. Click **Create Role**

### Step 3 - Edit the Trust Relationship

1. Go to **Role Details > Trust Relationships**
2. Click **Edit Trust Policy**
3. Add a **condition for the `sub` field** to restrict which service account can assume this role:

```json
"Condition": {
  "StringEquals": {
    "oidc.eks.<region>.amazonaws.com/id/<OIDC_ID>:aud": "sts.amazonaws.com",
    "oidc.eks.<region>.amazonaws.com/id/<OIDC_ID>:sub": "system:serviceaccount:kube-system:aws-load-balancer-controller"
  }
}
```

4. Click **Update Policy**

> This ensures the role can **only be assumed** by the `aws-load-balancer-controller` service account in the `kube-system` namespace.

---

## Deploying ALB Ingress Controller Resources

### Step 1 - Create the Kubernetes Service Account

```bash
kubectl create serviceaccount aws-load-balancer-controller -n kube-system
```

### Step 2 - Annotate the Service Account with IAM Role ARN

```bash
kubectl annotate serviceaccount aws-load-balancer-controller \
  -n kube-system \
  eks.amazonaws.com/role-arn=<ROLE_ARN>
```

Verify the annotation:

```bash
kubectl describe serviceaccounts -n kube-system aws-load-balancer-controller
```

### Step 3 - Deploy Using Helm

Add the EKS Helm repository:

```bash
helm repo add eks https://aws.github.io/eks-charts
helm repo update
```

Install the AWS Load Balancer Controller:

```bash
helm upgrade -i aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=demo-eks-cluster \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=us-east-2 \
  --set vpcId=<VPC_ID>
```

- `upgrade -i` - installs the release if it doesn't exist
- `serviceAccount.create=false` - uses the pre-created service account

Verify the deployment:

```bash
helm list -A

helm list -n kube-system
```

### Step 4 - Apply the CRDs

Download the CRD manifest:

```bash
wget https://raw.githubusercontent.com/aws/eks-charts/master/stable/aws-load-balancer-controller/crds/crds.yaml
```

Apply the CRDs:

```bash
kubectl apply -f crds.yaml
```

This creates two custom resources:
- **TargetGroupBinding**
- **IngressClassParams**

Verify the Ingress Class:

```bash
kubectl get ingressclass
```

> You should see an entry for `alb`.

### Step 5 - Verify the Controller is Running

```bash
kubectl get deployments.apps -n kube-system aws-load-balancer-controller

# Check pods
kubectl get pods -n kube-system | grep -i aws
```

> Expect **2 pods** for the load balancer controller with status `Running`.

---

## Deploying ALB Ingress Controller to Route External Traffic

### Application Architecture

Three services deployed in the cluster:
- **Redis** (StatefulSet) - backend storage
- **Participate** service - listens on port 80, container on port 5000
- **Submissions** service - listens on port 80, container on port 5000

Two services exposed via the ALB Ingress Controller.

### Step 1 - Deploy the Application

```bash
kubectl apply -f manifest.yaml
```

Creates:
- Namespace
- 1 Redis StatefulSet
- 2 Deployments
- 3 ClusterIP Services

### Step 2 - Apply the Ingress Resource

Sample Ingress definition:

```yaml
# alb-ingress-resources.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: demo-alb-ingress
  namespace: <your-namespace>
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/subnets: <public-subnet-ids>
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/backend-protocol: HTTP
spec:
  ingressClassName: alb
  rules:
    - http:
        paths:
          - path: /participate
            pathType: Prefix
            backend:
              service:
                name: participate
                port:
                  number: 80
          - path: /submissions
            pathType: Prefix
            backend:
              service:
                name: submissions
                port:
                  number: 80
```

Verify the Ingress:

```bash
kubectl get ingress -n <namespace>
```

> The `ADDRESS` column shows the **DNS name** of the automatically provisioned ALB.

### Step 3 - Test the Application

- Wait **60-90 seconds** for the ALB to become active
- Access `http://<ALB_DNS>/participate` to submit a lucky draw entry
- Access `http://<ALB_DNS>/submissions` to view all submitted entries

Both routes are served from a **single ALB with one DNS entry** using path-based routing.

### Key Observations

- ALB was **automatically provisioned** when the Ingress resource was applied
- No manual ALB creation required
- Pod names in responses confirm correct routing to backend pods

### Cleanup

```bash
kubectl delete ingress demo-alb-ingress -n <namespace>
```

> Deleting the Ingress triggers **automatic deletion** of the associated ALB from AWS.

```bash
kubectl delete -f manifest.yaml
```

---

## Summary

| Step | Action |
|------|--------|
| 1 | Created **IAM Policy** with all necessary ALB permissions |
| 2 | Created **IAM Role** linked to EKS cluster via OIDC identity provider |
| 3 | Installed **AWS Load Balancer Controller** using Helm |
| 4 | Applied required **CRDs** manually |
| 5 | Deployed Ingress resource - ALB **automatically provisioned** |
| 6 | Confirmed **path-based routing** to correct backend pods |

### Key Takeaways

- The **AWS Load Balancer Controller** acts as a bridge between Kubernetes and AWS Load Balancers
- ALB operates at **Layer 7** - supports URL, hostname, and path-based routing
- **IP mode** is more efficient and required for Fargate workloads
- A **single ALB** with one DNS entry can serve **multiple services** through path-based routing
- The entire ALB lifecycle (creation, configuration, deletion) is managed **automatically** by the controller
- IAM integration via **OIDC + Service Account annotation** ensures secure, scoped access
