# Kubernetes on Raspberry Pi - Complete Visual Guide
## For Single Node K3s Setup

---

## Table of Contents
1. Architecture Overview
2. Understanding Kubernetes Components
3. K3s vs Docker Desktop
4. How to Use K3s (YAML vs CLI)
5. Complete Setup Guide
6. Practical Examples

---

## 1. Architecture Overview

### Multi-Node vs Your Single-Node

**Article's Setup (4 Pis):**
```
┌─────────────────────────────────────────────────────────┐
│                    HOME NETWORK                         │
│                                                         │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐ │
│  │   Pi 1       │  │   Pi 2       │  │   Pi 3       │ │
│  │   MASTER     │  │   WORKER     │  │   WORKER     │ │
│  │  (Manager)   │  │   (Runner)   │  │   (Runner)   │ │
│  └──────────────┘  └──────────────┘  └──────────────┘ │
│         │                 │                 │          │
│         └─────────────────┴─────────────────┘          │
│                           │                            │
│                  ┌──────────────┐                      │
│                  │   Pi 4       │                      │
│                  │   WORKER     │                      │
│                  │   (Runner)   │                      │
│                  └──────────────┘                      │
└─────────────────────────────────────────────────────────┘
```

**YOUR Setup (1 Pi):**
```
┌─────────────────────────────────────────────────────────┐
│                    HOME NETWORK                         │
│                                                         │
│              ┌──────────────────────┐                  │
│              │  Raspberry Pi 5      │                  │
│              │                      │                  │
│              │  ┌────────────────┐  │                  │
│              │  │  MASTER        │  │ ← Controls       │
│              │  │  (Control      │  │   everything     │
│              │  │   Plane)       │  │                  │
│              │  └────────────────┘  │                  │
│              │                      │                  │
│              │  ┌────────────────┐  │                  │
│              │  │  WORKER        │  │ ← Runs your      │
│              │  │  (Same Pi!)    │  │   apps           │
│              │  └────────────────┘  │                  │
│              │                      │                  │
│              └──────────────────────┘                  │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

**Key Point:** Your single Pi acts as BOTH master and worker!

---

## 2. Understanding Kubernetes Components

### What is Kubernetes? (Port Analogy)

```
┌────────────────────────────────────────────────────────┐
│               KUBERNETES = SHIPPING PORT               │
│                                                        │
│  ┌──────────────────────────────────────────────────┐ │
│  │  HARBOR MASTER (Control Plane)                   │ │
│  │  • Tracks all ships (pods)                       │ │
│  │  • Assigns docks (nodes)                         │ │
│  │  • Handles scheduling                            │ │
│  └──────────────────────────────────────────────────┘ │
│                          │                            │
│                          ↓                            │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  │
│  │   DOCK 1    │  │   DOCK 2    │  │   DOCK 3    │  │
│  │  (Node 1)   │  │  (Node 2)   │  │  (Node 3)   │  │
│  │             │  │             │  │             │  │
│  │  🚢 🚢      │  │  🚢 🚢 🚢   │  │  🚢         │  │
│  │  (Pods)     │  │  (Pods)     │  │  (Pods)     │  │
│  └─────────────┘  └─────────────┘  └─────────────┘  │
│                                                        │
└────────────────────────────────────────────────────────┘

In your case: Only 1 dock, but it works the same way!
```

### Component 1: Pod

```
┌─────────────────────────────────┐
│          POD                    │
│  ┌───────────────────────────┐  │
│  │     CONTAINER 1           │  │ ← Your app
│  │   (e.g., ArgoCD)          │  │   (Docker container)
│  │   IP: 10.42.0.16          │  │
│  │   Port: 8080              │  │
│  └───────────────────────────┘  │
│                                 │
│  Shared storage & network       │
└─────────────────────────────────┘

Pod = The smallest unit in Kubernetes
    = A box that holds one or more containers
```

### Component 2: Deployment

```
┌──────────────────────────────────────────┐
│         DEPLOYMENT                       │
│   "Run 3 copies of my app"               │
│                                          │
│   ┌──────┐  ┌──────┐  ┌──────┐         │
│   │ Pod 1│  │ Pod 2│  │ Pod 3│         │
│   │ App  │  │ App  │  │ App  │         │
│   └──────┘  └──────┘  └──────┘         │
│                                          │
│   If Pod 2 crashes → Auto-restart!      │
│   If you scale → Creates more pods!     │
└──────────────────────────────────────────┘

Deployment = Instructions for how to run your app
           = Manages pods
           = Handles scaling & updates
```

### Component 3: Service

```
Problem: Pods get random IPs that change when they restart!
Pod 1: 10.42.0.15 → Crashes → New pod: 10.42.0.23

Solution: Service = Permanent internal address

┌──────────────────────────────────────────┐
│          SERVICE                         │
│   Name: my-app-service                   │
│   ClusterIP: 10.43.85.210 (permanent!)   │
│                                          │
│         Routes traffic to:               │
│                 ↓                        │
│   ┌──────┐  ┌──────┐  ┌──────┐         │
│   │ Pod 1│  │ Pod 2│  │ Pod 3│         │
│   │10.42 │  │10.42 │  │10.42 │         │
│   │.0.15 │  │.0.16 │  │.0.17 │         │
│   └──────┘  └──────┘  └──────┘         │
└──────────────────────────────────────────┘

Service = A permanent phone number
        = Load balancer between pods
        = DNS name inside cluster
```

### Component 4: Ingress

```
┌────────────────────────────────────────────────────┐
│            NGINX INGRESS                           │
│         (Listens on port 80/443)                   │
│                                                    │
│  Request comes in: "argocd.homelab.local"         │
│                                                    │
│  Checks rules:                                     │
│  ┌──────────────────────────────────────────┐     │
│  │ argocd.homelab.local → ArgoCD Service    │     │
│  │ app.homelab.local    → App Service       │     │
│  │ api.homelab.local    → API Service       │     │
│  └──────────────────────────────────────────┘     │
│                                                    │
│  Routes to correct service ✓                      │
└────────────────────────────────────────────────────┘

Ingress = Smart receptionist
        = Routes external traffic
        = Based on hostname/path
```

---

## 3. Complete Request Flow

### User Visits Your Website - Step by Step

```
STEP 1: User Types URL
┌─────────────────────┐
│  Your Laptop        │
│  Browser:           │
│  "myapp.homelab.    │
│   local"            │
└──────────┬──────────┘
           │
           ↓

STEP 2: DNS Resolution
┌─────────────────────┐
│  /etc/hosts         │
│  192.168.88.3       │
│  myapp.homelab.     │
│  local              │
└──────────┬──────────┘
           │
           ↓

STEP 3: Request Hits Pi
┌─────────────────────┐
│  Raspberry Pi       │
│  IP: 192.168.88.3   │
│  Port 80            │
└──────────┬──────────┘
           │
           ↓

STEP 4: Nginx Ingress Receives
┌───────────────────────────────┐
│  Nginx Ingress Controller     │
│  "Looking for myapp.homelab.  │
│   local..."                   │
│  Checks Ingress rules...      │
└──────────┬────────────────────┘
           │
           ↓

STEP 5: Routes to Service
┌───────────────────────────────┐
│  Service: myapp-service       │
│  ClusterIP: 10.43.x.x         │
│  "Which pod should I use?"    │
└──────────┬────────────────────┘
           │
           ↓

STEP 6: Forwards to Pod
┌───────────────────────────────┐
│  Pod: myapp-pod               │
│  Container: Your App          │
│  Port: 3000                   │
│  "Processing request..."      │
└──────────┬────────────────────┘
           │
           ↓

STEP 7: Response Back
┌───────────────────────────────┐
│  Your App Returns HTML        │
│  ← Through Service            │
│  ← Through Ingress            │
│  ← To Browser                 │
│  User sees your website! 🎉   │
└───────────────────────────────┘
```

---

## 4. ArgoCD - The Auto-Deployer

### GitOps Workflow

```
┌──────────────────────────────────────────────────────┐
│                   GITOPS WORKFLOW                    │
│                                                      │
│  STEP 1: You Code                                    │
│  ┌────────────┐                                     │
│  │ Your Laptop│                                     │
│  │ VS Code    │                                     │
│  └─────┬──────┘                                     │
│        │ git push                                    │
│        ↓                                            │
│                                                      │
│  STEP 2: GitHub Stores                              │
│  ┌────────────────────────────┐                    │
│  │  GitHub Repository         │                    │
│  │  ├── src/                  │                    │
│  │  ├── Dockerfile            │                    │
│  │  └── k8s/                  │                    │
│  │      ├── deployment.yaml   │                    │
│  │      ├── service.yaml      │                    │
│  │      └── ingress.yaml      │                    │
│  └────────────┬───────────────┘                    │
│               │                                     │
│               │                                     │
│  STEP 3: ArgoCD Watches                            │
│  ┌────────────▼───────────────┐                    │
│  │  ArgoCD (Running on Pi)    │                    │
│  │  • Polls GitHub every 3min │                    │
│  │  • Detects changes         │                    │
│  │  • "Oh! deployment.yaml    │                    │
│  │    changed!"               │                    │
│  └────────────┬───────────────┘                    │
│               │                                     │
│               ↓                                     │
│                                                      │
│  STEP 4: Applies to Cluster                        │
│  ┌─────────────────────────────┐                   │
│  │  Kubernetes Cluster (Pi)    │                   │
│  │  kubectl apply -f ...       │                   │
│  │  ✓ Deployment updated       │                   │
│  │  ✓ Pods recreated           │                   │
│  │  ✓ App updated!             │                   │
│  └─────────────────────────────┘                   │
│                                                      │
└──────────────────────────────────────────────────────┘

Result: Code changes automatically deploy! 🚀
```

---

## 5. Port Mapping - Layer by Layer

```
┌─────────────────────────────────────────────────────────────┐
│            HOW PORTS WORK - LAYER BY LAYER                  │
│                                                             │
│  EXTERNAL (Your Laptop)                                     │
│  http://myapp.homelab.local:80                             │
│           │                                                 │
│           ↓                                                 │
│  ┌────────────────────────────────────────────────────┐    │
│  │  Pi's Network Interface (192.168.88.3:80)          │    │
│  └───────────────────┬────────────────────────────────┘    │
│                      │                                      │
│                      ↓                                      │
│  ┌────────────────────────────────────────────────────┐    │
│  │  Nginx Ingress Service (Port 80)                   │    │
│  │  Type: LoadBalancer                                │    │
│  │  ExternalIP: 192.168.88.3                          │    │
│  └───────────────────┬────────────────────────────────┘    │
│                      │                                      │
│                      ↓                                      │
│  ┌────────────────────────────────────────────────────┐    │
│  │  Nginx Ingress Pod (Container Port 80/443)         │    │
│  │  Reads Ingress rules                               │    │
│  │  Routes based on hostname                          │    │
│  └───────────────────┬────────────────────────────────┘    │
│                      │                                      │
│                      ↓                                      │
│  ┌────────────────────────────────────────────────────┐    │
│  │  Your App Service (ClusterIP)                      │    │
│  │  Port: 80 → forwards to →                          │    │
│  └───────────────────┬────────────────────────────────┘    │
│                      │                                      │
│                      ↓                                      │
│  ┌────────────────────────────────────────────────────┐    │
│  │  Your App Pod                                      │    │
│  │  Container Port: 3000                              │    │
│  │  Your Node.js app actually listens here           │    │
│  └────────────────────────────────────────────────────┘    │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 6. Your Complete Architecture

```
┌──────────────────────────────────────────────────────────────┐
│                    RASPBERRY PI 5                            │
│                   (192.168.88.3)                             │
│                                                              │
│  ┌────────────────────────────────────────────────────────┐ │
│  │              KUBERNETES (K3s)                          │ │
│  │                                                        │ │
│  │  ┌──────────────────────────────────────────────────┐ │ │
│  │  │  Namespace: ingress-nginx                        │ │ │
│  │  │  ┌────────────────────────────────────────────┐  │ │ │
│  │  │  │  Pod: Nginx Ingress Controller           │  │ │ │
│  │  │  │  • Listens on port 80/443                │  │ │ │
│  │  │  │  • Routes traffic                        │  │ │ │
│  │  │  └────────────────────────────────────────────┘  │ │ │
│  │  └──────────────────────────────────────────────────┘ │ │
│  │                                                        │ │
│  │  ┌──────────────────────────────────────────────────┐ │ │
│  │  │  Namespace: argocd                               │ │ │
│  │  │  ┌────────────────────────────────────────────┐  │ │ │
│  │  │  │  Pod: ArgoCD Server                      │  │ │ │
│  │  │  │  • Web UI                                │  │ │ │
│  │  │  │  • Watches GitHub                        │  │ │ │
│  │  │  │  • Syncs apps                            │  │ │ │
│  │  │  └────────────────────────────────────────────┘  │ │ │
│  │  └──────────────────────────────────────────────────┘ │ │
│  │                                                        │ │
│  │  ┌──────────────────────────────────────────────────┐ │ │
│  │  │  Namespace: default                              │ │ │
│  │  │  ┌────────────────────────────────────────────┐  │ │ │
│  │  │  │  Pod: Your App 1                         │  │ │ │
│  │  │  │  Container: my-aduanku:v1.0              │  │ │ │
│  │  │  └────────────────────────────────────────────┘  │ │ │
│  │  │  ┌────────────────────────────────────────────┐  │ │ │
│  │  │  │  Pod: Your App 2                         │  │ │ │
│  │  │  │  Container: tax-tracker:v1.0             │  │ │ │
│  │  │  └────────────────────────────────────────────┘  │ │ │
│  │  └──────────────────────────────────────────────────┘ │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 7. K3s vs Docker Desktop - Key Differences

### Docker Desktop (What You Know)

```
┌─────────────────────────────────┐
│       DOCKER DESKTOP            │
│                                 │
│  docker run nginx               │
│       ↓                         │
│  Creates 1 container            │
│       ↓                         │
│  Running on your laptop         │
│                                 │
│  • Simple                       │
│  • Single machine               │
│  • Manual management            │
│  • No orchestration             │
└─────────────────────────────────┘
```

### Kubernetes/K3s (What You're Learning)

```
┌─────────────────────────────────┐
│       KUBERNETES (K3s)          │
│                                 │
│  kubectl apply -f app.yaml      │
│       ↓                         │
│  Creates Deployment             │
│       ↓                         │
│  Deployment creates 3 Pods      │
│       ↓                         │
│  Each Pod has containers        │
│       ↓                         │
│  Automatic:                     │
│  • Restart if crash             │
│  • Load balancing               │
│  • Scaling                      │
│  • Updates                      │
│  • Self-healing                 │
└─────────────────────────────────┘
```

### Key Differences

| Feature | Docker Desktop | Kubernetes (K3s) |
|---------|---------------|------------------|
| **Complexity** | Simple | More complex |
| **Scale** | 1 machine | Multiple machines |
| **Containers** | Run directly | Wrapped in Pods |
| **Management** | Manual | Automated |
| **High Availability** | No | Yes |
| **Load Balancing** | No | Yes |
| **Auto-restart** | No | Yes |
| **Ideal For** | Development | Production |

---

## 8. How to Use K3s - YAML vs CLI

### The Two Ways to Control Kubernetes

#### Method 1: CLI (Command Line - Quick & Dirty)

```bash
# Create a deployment directly
kubectl create deployment nginx --image=nginx

# Expose it as a service
kubectl expose deployment nginx --port=80

# Scale it
kubectl scale deployment nginx --replicas=3

# Delete it
kubectl delete deployment nginx
```

**Pros:**
- ✅ Fast for testing
- ✅ Good for learning
- ✅ Quick experiments

**Cons:**
- ❌ Not reproducible
- ❌ Hard to track changes
- ❌ Not GitOps-friendly
- ❌ Manual work

#### Method 2: YAML Files (Declarative - Professional)

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 3
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
        image: nginx:latest
        ports:
        - containerPort: 80
```

```bash
# Apply the YAML
kubectl apply -f deployment.yaml

# To delete
kubectl delete -f deployment.yaml
```

**Pros:**
- ✅ Reproducible
- ✅ Version controlled (Git)
- ✅ GitOps-ready
- ✅ Self-documenting
- ✅ Easy to review

**Cons:**
- ❌ More initial work
- ❌ Need to learn YAML syntax

### Which Method Should You Use?

**For Learning:** Start with CLI to understand concepts
**For Real Projects:** Use YAML files (always!)

---

## 9. YAML File Structure Explained

### Anatomy of a Kubernetes YAML File

```yaml
# Every Kubernetes YAML has these 4 parts:

# 1. API Version - Which Kubernetes API to use
apiVersion: apps/v1

# 2. Kind - What type of object is this?
kind: Deployment

# 3. Metadata - Information about the object
metadata:
  name: my-app              # Name of this deployment
  namespace: default        # Which namespace (folder)
  labels:                   # Tags for organization
    app: my-app
    version: v1.0

# 4. Spec - The actual configuration
spec:
  replicas: 3               # How many copies?
  selector:                 # How to find pods?
    matchLabels:
      app: my-app
  template:                 # Pod template
    metadata:
      labels:
        app: my-app
    spec:
      containers:           # List of containers
      - name: my-app
        image: my-app:v1.0
        ports:
        - containerPort: 3000
```

### Common Kubernetes Objects (Kinds)

1. **Pod** - Single container or group
2. **Deployment** - Manages pods
3. **Service** - Network access to pods
4. **Ingress** - External HTTP(S) access
5. **ConfigMap** - Configuration data
6. **Secret** - Sensitive data
7. **Namespace** - Virtual cluster separation

---

## 10. Practical Example - Deploy Hello World

### Step-by-Step with YAML

#### File 1: deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-world
  namespace: default
spec:
  replicas: 2
  selector:
    matchLabels:
      app: hello-world
  template:
    metadata:
      labels:
        app: hello-world
    spec:
      containers:
      - name: hello
        image: nginxdemos/hello:latest
        ports:
        - containerPort: 80
```

#### File 2: service.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: hello-world-service
  namespace: default
spec:
  selector:
    app: hello-world
  ports:
  - port: 80
    targetPort: 80
  type: ClusterIP
```

#### File 3: ingress.yaml

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: hello-world-ingress
  namespace: default
spec:
  ingressClassName: nginx
  rules:
  - host: hello.homelab.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: hello-world-service
            port:
              number: 80
```

#### Deploy Commands

```bash
# Apply all files
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
kubectl apply -f ingress.yaml

# Or apply all at once
kubectl apply -f .

# Check status
kubectl get deployments
kubectl get pods
kubectl get services
kubectl get ingress

# Access it
# Add to /etc/hosts: 192.168.88.3 hello.homelab.local
# Visit: http://hello.homelab.local
```

---

## 11. Essential kubectl Commands

### Viewing Resources

```bash
# Get all pods
kubectl get pods

# Get all pods in all namespaces
kubectl get pods -A

# Get detailed info about a pod
kubectl describe pod <pod-name>

# Get pod logs
kubectl logs <pod-name>

# Get pod logs (follow mode)
kubectl logs -f <pod-name>

# Get all deployments
kubectl get deployments

# Get all services
kubectl get services

# Get all ingress
kubectl get ingress

# Get everything
kubectl get all
```

### Creating Resources

```bash
# From YAML file
kubectl apply -f myapp.yaml

# From directory of YAML files
kubectl apply -f ./k8s/

# From URL
kubectl apply -f https://example.com/app.yaml
```

### Deleting Resources

```bash
# Delete from YAML
kubectl delete -f myapp.yaml

# Delete by name
kubectl delete deployment my-app
kubectl delete service my-app
kubectl delete pod my-pod

# Delete everything in namespace
kubectl delete all --all -n my-namespace
```

### Debugging

```bash
# Execute command in pod
kubectl exec -it <pod-name> -- /bin/sh

# Port forward to access pod directly
kubectl port-forward <pod-name> 8080:80

# Get events (troubleshooting)
kubectl get events --sort-by=.metadata.creationTimestamp

# Describe for detailed info
kubectl describe pod <pod-name>
kubectl describe deployment <deployment-name>
```

---

## 12. Complete Setup From Scratch

### Phase 1: Fresh Install (30 mins)

#### 1. Flash Ubuntu Server
```
1. Download Raspberry Pi Imager
2. Choose: Ubuntu Server 24.04 LTS (64-bit)
3. Settings (⚙️):
   - Set hostname: homelab-pi
   - Enable SSH
   - Set username: haziqazli
   - Set password: [your password]
   - Configure WiFi (if needed)
4. Flash to SD card
5. Boot Pi
```

#### 2. First Login
```bash
# SSH into Pi
ssh haziqazli@homelab-pi.local
# or
ssh haziqazli@192.168.88.X  # Use IP if .local doesn't work

# Update system
sudo apt update && sudo apt upgrade -y
```

#### 3. Set Static IP
```bash
# Find current IP
hostname -I

# Edit netplan
sudo nano /etc/netplan/50-cloud-init.yaml
```

**Add this:**
```yaml
network:
  version: 2
  wifis:
    wlan0:
      dhcp4: no
      addresses:
        - 192.168.88.3/24  # Your static IP
      routes:
        - to: default
          via: 192.168.88.1  # Your router
      nameservers:
        addresses: [8.8.8.8, 1.1.1.1]
      access-points:
        "YourWiFiName":
          password: "YourWiFiPassword"
```

```bash
# Apply
sudo netplan apply

# Reboot
sudo reboot
```

### Phase 2: Install K3s (10 mins)

```bash
# SSH back in
ssh haziqazli@192.168.88.3

# Install K3s (single command!)
curl -sfL https://get.k3s.io | sh -

# Wait for it to finish (~2 minutes)

# Check status
sudo systemctl status k3s

# Setup kubectl access
mkdir -p ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown $USER:$USER ~/.kube/config
chmod 600 ~/.kube/config

# Test it works
kubectl get nodes
# Should show: homelab-pi   Ready   control-plane,master

kubectl get pods -A
# Should show system pods running
```

### Phase 3: Install Nginx Ingress (5 mins)

```bash
# Install Helm
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Add nginx repo
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

# Install nginx ingress
helm install nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace \
  --set controller.service.type=LoadBalancer \
  --set controller.service.externalIPs[0]=192.168.88.3

# Wait for ready
kubectl wait --for=condition=Ready pods --all -n ingress-nginx --timeout=300s

# Verify
kubectl get pods -n ingress-nginx
# Should show nginx pod Running
```

### Phase 4: Install ArgoCD (5 mins)

```bash
# Create namespace
kubectl create namespace argocd

# Install ArgoCD
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Wait for ready
kubectl wait --for=condition=Ready pods --all -n argocd --timeout=300s

# Change to NodePort for access
kubectl patch svc argocd-server -n argocd -p '{"spec":{"type":"NodePort"}}'

# Get the port
kubectl get svc argocd-server -n argocd -o jsonpath='{.spec.ports[?(@.name=="https")].nodePort}'; echo

# Get password
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d; echo

# Access at: https://192.168.88.3:[PORT]
# Username: admin
# Password: [from command above]
```

---

## 13. Next Steps

### Option A: Deploy Hello World (15 mins)
Learn the basics with a simple app

### Option B: Deploy Your Real App (45 mins)
My Aduanku or Tax Tracker with full CI/CD

### Option C: Add Your Domain (60 mins)
Configure webwork.my with SSL

---

## 14. Useful Resources

- **K3s Documentation**: https://docs.k3s.io
- **Kubernetes Docs**: https://kubernetes.io/docs/
- **ArgoCD Docs**: https://argo-cd.readthedocs.io
- **Helm Charts**: https://artifacthub.io
- **kubectl Cheat Sheet**: https://kubernetes.io/docs/reference/kubectl/cheatsheet/

---

## 15. Troubleshooting

### Common Issues

**Pod stuck in Pending:**
```bash
kubectl describe pod <pod-name>
# Look for events at bottom
```

**Can't access service:**
```bash
# Check service exists
kubectl get svc

# Check endpoints
kubectl get endpoints

# Check ingress
kubectl get ingress
```

**K3s not starting:**
```bash
sudo systemctl status k3s
sudo journalctl -u k3s -f
```

---

## Key Takeaways

1. **Kubernetes = Container orchestration platform**
2. **K3s = Lightweight Kubernetes for small devices**
3. **YAML files = Professional way to manage resources**
4. **CLI = Quick testing and debugging**
5. **ArgoCD = GitOps auto-deployment**
6. **Single Pi = Both master and worker**

---

**Created for Haziq's Raspberry Pi Homelab Journey**
Date: November 2025
