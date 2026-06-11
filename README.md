# Attendance Tracker System - Kubernetes Demo

A production-ready Kubernetes deployment of an Attendance Tracking application featuring blue-green deployment strategy, TLS encryption, and a complete microservices architecture.

## 📋 Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
- [Project Structure](#project-structure)
- [Deployment](#deployment)
- [Configuration](#configuration)
- [Accessing the Application](#accessing-the-application)
- [Troubleshooting](#troubleshooting)

## 🎯 Overview

This project demonstrates a complete Kubernetes deployment of an Attendance Tracker System with:

- **Multi-tier Architecture**: Frontend, Backend API, and Database layers
- **Blue-Green Deployment**: Zero-downtime deployments with instant rollback capability
- **Secure Access**: HTTPS/TLS encryption with NGINX Ingress
- **Data Persistence**: PostgreSQL StatefulSet with persistent volumes
- **Configuration Management**: ConfigMaps and Secrets for environment configuration
- **Service Discovery**: Internal Kubernetes DNS service discovery
- **Namespace Isolation**: Dedicated `attendance-system` namespace

## 🏗️ Architecture

```
┌─────────────────────────────────────────────┐
│         Users (attendance.local)            │
└────────────────┬────────────────────────────┘
                 │
    ┌────────────▼─────────────┐
    │  NGINX Ingress Controller │
    │      (TLS/HTTPS)          │
    │   attendance.local        │
    └─┬──────────────────────┬──┘
      │                      │
      │ /api                 │ /
      │                      │
  ┌───▼──────────┐   ┌──────▼──────┐
  │   Backend    │   │  Frontend    │
  │   Service    │   │   Service    │
  │  (Node.js)   │   │   (React)    │
  │   Port 3000  │   │   Port 80    │
  └───┬──────────┘   └──────────────┘
      │
      │ DB Connection
      │ Host: postgresql-service
      │ Port: 5432
      │
  ┌───▼──────────────────┐
  │   PostgreSQL         │
  │   StatefulSet        │
  │   attendance_db      │
  └──────────────────────┘
```

## 📦 Components

### Frontend Service
- **Type**: ClusterIP
- **Port**: 80 (HTTP)
- **Location**: Served via Ingress at root path `/`
- **Namespace**: `attendance-system`

### Backend API Service
- **Type**: ClusterIP
- **Port**: 3000 (Node.js)
- **Framework**: Node.js/Express
- **Environment**: Production
- **Location**: Exposed at Ingress path `/api`
- **Namespace**: `attendance-system`
- **Replicas**: 2 (Blue-Green deployment)

### Database Service
- **Type**: StatefulSet (PostgreSQL)
- **Database**: PostgreSQL 16 (Alpine)
- **Port**: 5432
- **Replicas**: 1
- **Persistence**: Enabled via persistent volume
- **Database Name**: `attendance_db`
- **Namespace**: `attendance-system`

### Ingress Configuration
- **Controller**: NGINX
- **Domain**: `attendance.local`
- **TLS**: Enabled with self-signed certificate
- **Routes**:
  - `/api` → Backend Service (port 3000)
  - `/` → Frontend Service (port 80)

### Blue-Green Deployment Strategy
- **Approach**: Two identical production environments (Blue and Green)
- **Active Environment**: Service routes to one environment at a time
- **Zero Downtime**: Switch traffic instantly between versions
- **Quick Rollback**: Revert to previous version in seconds
- **Testing**: Full smoke tests on inactive environment before switch

## 🚀 Prerequisites

- Kubernetes cluster (v1.20+)
- `kubectl` CLI tool configured
- NGINX Ingress Controller installed

### Local Testing with minikube/Docker Desktop

```bash
# Enable ingress addon (minikube)
minikube addons enable ingress

# Update /etc/hosts (or C:\Windows\System32\drivers\etc\hosts on Windows)
127.0.0.1 attendance.local
```

## 📁 Project Structure

```
k8s-Demo/
├── namespace.yaml              # Namespace creation
├── LICENSE
├── README.md                   # This file
│
├── backend/
│   ├── configmap.yaml          # Backend environment configuration
│   ├── secret.yaml             # Database credentials
│   ├── service.yaml            # Backend API service
│   ├── deployment-blue.yaml    # Blue deployment (v1)
│   └── deployment-green.yaml   # Green deployment (v2)
│
├── database/
│   ├── configmap.yaml          # PostgreSQL configuration
│   ├── secret.yaml             # PostgreSQL admin credentials
│   ├── service.yaml            # PostgreSQL service
│   └── statefulset.yaml        # PostgreSQL StatefulSet definition
│
├── frontend/
│   ├── configmap.yaml          # Frontend configuration
│   ├── service.yaml            # Frontend service
│   ├── deployment-blue.yaml    # Blue deployment (v1)
│   └── deployment-green.yaml   # Green deployment (v2)
│
└── ingress/
    ├── ingress.yaml            # NGINX Ingress routing rules
    ├── nginx-ingress-controller.yaml  # NGINX controller setup
    └── tls-secret.yaml         # TLS certificate secret
```

## 🔧 Deployment

For Minikube:
minikube start --cpus=4 --memory=6144 --driver=docker
minikube addons enable ingress
kubectl auth can-i get pods --as=system:serviceaccount:ingress-nginx:ingress-nginx -n ingress-nginx
kubectl describe clusterrolebinding ingress-nginx

for Argo if using:
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

### 1. Create Namespace

```bash
kubectl apply -f namespace.yaml
```

### 2. Install NGINX Ingress Controller

```bash
kubectl apply -f ingress/nginx-ingress-controller.yaml
```

### 3. Deploy Database

```bash
kubectl apply -f database/
```

Wait for PostgreSQL to be ready:

```bash
kubectl wait --for=condition=ready pod -l app=postgresql -n attendance-system --timeout=300s
```

### 5. Deploy Backend

```bash
kubectl apply -f backend/
kubectl apply -f https://github.com/argoproj/argo-rollouts/releases/latest/download/install.yaml
kubectl apply -f rollouts/backend-rollout.yaml
```

### 6. Deploy Frontend

```bash
kubectl apply -f frontend/
kubectl apply -f rollouts/frontend-rollout.yaml
```

### 6. Configure Ingress and TLS

```bash
kubectl apply -f ingress/tls-secret.yaml
kubectl apply -f ingress/ingress.yaml
```

### Complete Deployment Script

```bash
#!/bin/bash
set -e

echo "Creating namespace..."
kubectl apply -f namespace.yaml

echo "Installing NGINX Ingress..."
kubectl apply -f ingress/nginx-ingress-controller.yaml

echo "Deploying database..."
kubectl apply -f database/
kubectl wait --for=condition=ready pod -l app=postgresql -n attendance-system --timeout=300s

echo "Deploying backend (Blue)..."
kubectl apply -f backend/configmap.yaml
kubectl apply -f backend/secret.yaml
kubectl apply -f backend/deployment-blue.yaml
kubectl apply -f backend/service.yaml

echo "Deploying frontend (Blue)..."
kubectl apply -f frontend/configmap.yaml
kubectl apply -f frontend/deployment-blue.yaml
kubectl apply -f frontend/service.yaml

echo "Configuring ingress and TLS..."
kubectl apply -f ingress/tls-secret.yaml
kubectl apply -f ingress/ingress.yaml

echo "Deployment complete!"
```

## 🔄 Blue-Green Deployment Workflow

### Initial Setup

The initial deployment uses the **Blue** environment:

```bash
# Deploy Blue environment (initial production)
kubectl apply -f backend/deployment-blue.yaml
kubectl apply -f frontend/deployment-blue.yaml

# The service routes to Blue by default
kubectl get deployment -n attendance-system
```

### Deploying a New Version (Switch to Green)

When you have a new version ready:

```bash
# 1. Deploy the new version to Green environment
kubectl apply -f backend/deployment-green.yaml
kubectl apply -f frontend/deployment-green.yaml

# 2. Wait for Green pods to be ready
kubectl wait --for=condition=ready pod -l version=green -n attendance-system --timeout=300s

# 3. Run smoke tests on Green environment
kubectl port-forward svc/backend-green 3000:3000 -n attendance-system &
curl http://localhost:3000/health
kill %1

# 4. If tests pass, switch traffic to Green
# Update the service selector to point to Green
kubectl patch service backend-service -n attendance-system \
  -p '{"spec":{"selector":{"version":"green"}}}'
kubectl patch service frontend-service -n attendance-system \
  -p '{"spec":{"selector":{"version":"green"}}}'

# 5. Monitor the new version
kubectl logs -f deployment/backend-green -n attendance-system
```

### Rollback to Previous Version

If issues are detected:

```bash
# Instantly switch traffic back to Blue
kubectl patch service backend-service -n attendance-system \
  -p '{"spec":{"selector":{"version":"blue"}}}'
kubectl patch service frontend-service -n attendance-system \
  -p '{"spec":{"selector":{"version":"blue"}}}'

echo "Rolled back to Blue environment"
```

### Preparing Next Deployment

Once Blue is stable as the production version:

```bash
# Delete old Green deployment
kubectl delete deployment backend-green frontend-green -n attendance-system

# Blue becomes the new baseline
# Prepare new Green deployment with next version
kubectl apply -f backend/deployment-green.yaml
kubectl apply -f frontend/deployment-green.yaml
```

## 📊 Blue-Green Deployment States

| State | Blue | Green | Production |
|-------|------|-------|------------|
| Initial | v1 ✅ Active | - | Blue (v1) |
| Staging v2 | v1 ✅ Active | v2 Testing | Blue (v1) |
| After Promotion | v1 | v2 ✅ Active | Green (v2) |
| On Rollback | v1 ✅ Active | v2 Idle | Blue (v1) |
| Cleanup | v2 ✅ Active | - | Blue (v2) |

## ⚙️ Configuration

### Backend Configuration (ConfigMap)

The backend service is configured via `backend/configmap.yaml`:

| Variable | Value | Description |
|----------|-------|-------------|
| `NODE_ENV` | `production` | Node.js environment mode |
| `API_PORT` | `3000` | API server port |
| `DB_HOST` | `postgresql-service` | Database service hostname |
| `DB_PORT` | `5432` | PostgreSQL port |
| `DB_NAME` | `attendance_db` | Database name |

### Database Credentials (Secret)

Configured in `backend/secret.yaml` and `database/secret.yaml`:

| Variable | Default | Notes |
|----------|---------|-------|
| `DB_USER` | `admin` | PostgreSQL admin user |
| `DB_PASSWORD` | `SecureP@ssw0rd123` | ⚠️ Change in production |
| `POSTGRES_DB` | `attendance_db` | Database name |

⚠️ **Security Note**: Update these credentials in production!

### Environment Access

```bash
# Get backend configuration
kubectl get configmap backend-config -n attendance-system -o yaml

# Get database credentials
kubectl get secret backend-secret -n attendance-system -o yaml | grep -A 2 DB_

# Get PostgreSQL service
kubectl get service postgresql-service -n attendance-system
```

## 🌐 Accessing the Application

### Prerequisites

1. **Update hosts file:**

   **Linux/macOS**: `/etc/hosts`
   ```
   127.0.0.1 attendance.local
   ```

   **Windows**: `C:\Windows\System32\drivers\etc\hosts`
   ```
   127.0.0.1 attendance.local
   ```

2. **Port forwarding (if not using LoadBalancer):**

   ```bash
   kubectl port-forward -n ingress-nginx svc/ingress-nginx 443:443
   ```

### Access the Application

- **Frontend**: https://attendance.local/
- **API**: https://attendance.local/api
- **Backend Service (internal)**: http://backend-service.attendance-system.svc.cluster.local:3000

### Health Checks

```bash
# Check ingress status
kubectl get ingress -n attendance-system

# Check service endpoints
kubectl get endpoints -n attendance-system

# Check pod status
kubectl get pods -n attendance-system -o wide

# Check which version is active
kubectl get service backend-service -n attendance-system -o jsonpath='{.spec.selector}'
```

## 🔍 Monitoring & Debugging

### View Logs

```bash
# Backend Blue logs
kubectl logs -f deployment/backend-blue -n attendance-system

# Backend Green logs
kubectl logs -f deployment/backend-green -n attendance-system

# Database logs
kubectl logs -f statefulset/postgresql -n attendance-system

# Frontend Blue logs
kubectl logs -f deployment/frontend-blue -n attendance-system

# Frontend Green logs
kubectl logs -f deployment/frontend-green -n attendance-system

# Ingress logs
kubectl logs -f -n ingress-nginx
```

### Check Pod Status

```bash
# All pods in namespace
kubectl get pods -n attendance-system -o wide

# Detailed pod information
kubectl describe pod <pod-name> -n attendance-system
```

### Database Access

```bash
# Connect to PostgreSQL pod
kubectl exec -it postgresql-0 -n attendance-system -- psql -U admin -d attendance_db

# Common queries
\dt                    # List tables
\l                     # List databases
SELECT * FROM users;   # View data
```

### Blue-Green Deployment Management

```bash
# Check current active version
kubectl get svc backend-service -n attendance-system -o jsonpath='{.spec.selector.version}'

# View deployments and their status
kubectl get deployments -n attendance-system -L version
kubectl describe deployment backend-blue -n attendance-system
kubectl describe deployment backend-green -n attendance-system

# Check pod replicas
kubectl get pods -n attendance-system -L version

# Compare resources between Blue and Green
kubectl top pods -n attendance-system -l version=blue
kubectl top pods -n attendance-system -l version=green
```

## 🛠️ Troubleshooting

### Service Discovery Issues

```bash
# Test DNS resolution within cluster
kubectl run -it --rm debug --image=busybox --restart=Never -- nslookup backend-service.attendance-system.svc.cluster.local

# Test connectivity
kubectl run -it --rm debug --image=curlimages/curl --restart=Never -- curl http://backend-service.attendance-system.svc.cluster.local:3000
```

### Database Connection Errors

```bash
# Check PostgreSQL service
kubectl get svc -n attendance-system

# Verify credentials
kubectl get secret backend-secret -n attendance-system -o jsonpath='{.data.DB_USER}' | base64 -d

# Check PVC status
kubectl get pvc -n attendance-system
```

### Ingress Not Working

```bash
# Verify NGINX controller is running
kubectl get pods -n ingress-nginx

# Check ingress configuration
kubectl get ingress -n attendance-system -o yaml

# Check ingress controller logs
kubectl logs -n ingress-nginx -l app.kubernetes.io/name=ingress-nginx
```

### SSL/TLS Certificate Issues

```bash
# View certificate details
kubectl get secret attendance-tls -n attendance-system -o jsonpath='{.data.tls\.crt}' | base64 -d | openssl x509 -text -noout

# Regenerate TLS secret if needed
kubectl delete secret attendance-tls -n attendance-system
kubectl apply -f ingress/tls-secret.yaml
```

## 📊 Resource Requirements

**Recommended minimum cluster resources:**

- **CPU**: 2 cores
- **Memory**: 4 GB RAM
- **Storage**: 10 GB (for PostgreSQL PVC)

## 🔐 Security Considerations

1. **Secrets Management**: Use external secret management (Vault, AWS Secrets Manager) in production
2. **RBAC**: Review and enforce RBAC policies
3. **Network Policies**: Implement network policies for pod-to-pod communication
4. **TLS Certificates**: Use proper signed certificates instead of self-signed
5. **Database Backups**: Implement automated backup strategy
6. **Resource Limits**: Set CPU/memory limits for all containers

## 📝 License

See LICENSE file for details.

## 🤝 Contributing

For improvements or issues, please refer to the project guidelines.

---

**Last Updated**: 2024
**Kubernetes Version**: 1.20+
**Tested On**: minikube, Docker Desktop K8s, EKS 
