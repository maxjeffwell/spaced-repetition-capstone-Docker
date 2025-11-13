# Kubernetes Deployment Guide

This guide covers deploying the Spaced Repetition Capstone application to Kubernetes, specifically targeting Linode Kubernetes Engine (LKE).

## Table of Contents

- [Prerequisites](#prerequisites)
- [Architecture Overview](#architecture-overview)
- [Database Setup](#database-setup)
- [Cluster Setup](#cluster-setup)
- [Deployment](#deployment)
- [SSL/TLS Configuration](#ssltls-configuration)
- [Monitoring and Scaling](#monitoring-and-scaling)
- [Troubleshooting](#troubleshooting)
- [Cost Optimization](#cost-optimization)

## Prerequisites

- kubectl CLI installed
- Linode account (or other Kubernetes provider)
- Domain name configured
- MongoDB Atlas account
- Docker images pushed to Docker Hub

## Architecture Overview

### Components

```
┌─────────────────────────────────────────┐
│              Internet                    │
└────────────────┬────────────────────────┘
                 │
          ┌──────▼──────┐
          │   Ingress   │ (nginx + cert-manager)
          └──────┬──────┘
                 │
        ┌────────┴────────┐
        │                 │
   ┌────▼────┐      ┌────▼────┐
   │ Client  │      │ Server  │
   │ Service │      │ Service │
   └────┬────┘      └────┬────┘
        │                │
   ┌────▼────┐      ┌────▼────┐
   │ Client  │      │ Server  │
   │  Pods   │      │  Pods   │
   │ (2-8)   │      │ (2-10)  │
   └─────────┘      └────┬────┘
                         │
                   ┌─────▼──────┐
                   │  MongoDB   │
                   │   Atlas    │
                   │ (External) │
                   └────────────┘
```

### Features

- **Auto-scaling**: HPA scales 2-10 server pods, 2-8 client pods
- **High Availability**: Multiple replicas across nodes
- **Security**: TLS/SSL with Let's Encrypt, security hardening
- **External Database**: MongoDB Atlas (managed service)
- **Resource Management**: CPU/memory limits and requests
- **Health Checks**: Liveness and readiness probes

## Database Setup

### MongoDB Atlas

The application uses external MongoDB Atlas (not in-cluster):

1. **Create Cluster**:
   ```
   Atlas Console → Build a Database → Shared (Free M0 or Dedicated)
   ```

2. **Database Access**:
   ```
   Security → Database Access → Add New Database User
   - Username: spaced_repetition_prod
   - Password: <strong-random-password>
   - Role: Read and write to any database
   ```

3. **Network Access**:
   ```
   Security → Network Access → Add IP Address
   - For production: Add your Kubernetes cluster's external IPs
   - Or use 0.0.0.0/0 (allow from anywhere) with strong auth
   ```

4. **Get Connection String**:
   ```
   Deployment → Connect → Connect your application
   Format: mongodb+srv://username:password@cluster.mongodb.net/spaced_repetition?appName=spaced-repetition
   ```

### Why External Database?

- **Managed Service**: Automated backups, monitoring, updates
- **High Availability**: Built-in replication and failover
- **Scalability**: Easy to scale without managing infrastructure
- **Cost-Effective**: Free tier available, pay-as-you-grow
- **Security**: Atlas provides enterprise-grade security

## Cluster Setup

### Linode Kubernetes Engine (LKE)

#### 1. Create Cluster

```bash
# Via Linode CLI
linode-cli lke cluster-create \
  --label spaced-repetition-prod \
  --region us-east \
  --k8s_version 1.28 \
  --node_pools.type g6-standard-2 \
  --node_pools.count 3

# Or use Linode Cloud Manager UI:
# Kubernetes → Create Cluster
# - Label: spaced-repetition-prod
# - Region: Choose closest to users
# - Kubernetes Version: 1.28+
# - Node Pool: 3x Linode 4GB (g6-standard-2) or higher
```

#### 2. Configure kubectl

```bash
# Download kubeconfig from Linode
# Save to ~/.kube/spaced-repetition-prod-kubeconfig.yaml

export KUBECONFIG=~/.kube/spaced-repetition-prod-kubeconfig.yaml

# Verify connection
kubectl get nodes
kubectl cluster-info
```

### Alternative Providers

#### DigitalOcean Kubernetes (DOKS)

```bash
doctl kubernetes cluster create spaced-repetition-prod \
  --region nyc1 \
  --version 1.28.2-do.0 \
  --node-pool "name=worker;size=s-2vcpu-4gb;count=3"
```

#### AWS EKS

```bash
eksctl create cluster \
  --name spaced-repetition-prod \
  --region us-east-1 \
  --nodegroup-name standard-workers \
  --node-type t3.medium \
  --nodes 3
```

## Deployment

### 1. Create Namespace

```bash
cd k8s/
kubectl apply -f namespace.yaml
```

### 2. Configure Secrets

```bash
# Create secrets from your values
kubectl create secret generic spaced-repetition-secrets \
  --from-literal=MONGODB_URI='mongodb+srv://user:pass@cluster.mongodb.net/spaced_repetition?appName=spaced-repetition' \
  --from-literal=JWT_SECRET='your-super-secret-jwt-key-min-32-chars' \
  --namespace=spaced-repetition

# Or use the example file
cp secrets.yaml.example secrets.yaml
# Edit secrets.yaml with base64 encoded values
kubectl apply -f secrets.yaml

# Verify
kubectl get secrets -n spaced-repetition
```

**Generate base64 values:**

```bash
echo -n "mongodb+srv://user:pass@cluster.mongodb.net/spaced_repetition" | base64
echo -n "your-jwt-secret-here" | base64
```

### 3. Apply ConfigMap

```bash
# Update configmap.yaml with your domain
sed -i 's/yourdomain.com/your-actual-domain.com/g' configmap.yaml

kubectl apply -f configmap.yaml
```

### 4. Deploy Services

```bash
# Server
kubectl apply -f server-deployment.yaml
kubectl apply -f server-service.yaml

# Client
kubectl apply -f client-deployment.yaml
kubectl apply -f client-service.yaml

# Verify
kubectl get pods -n spaced-repetition
kubectl get services -n spaced-repetition
```

### 5. Install Ingress Controller

```bash
# Install nginx ingress controller
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.8.2/deploy/static/provider/cloud/deploy.yaml

# Wait for LoadBalancer IP
kubectl get services -n ingress-nginx -w
```

### 6. Configure DNS

```bash
# Get LoadBalancer external IP
kubectl get service -n ingress-nginx ingress-nginx-controller

# Add DNS A records:
# yourdomain.com     →  <LoadBalancer-IP>
# www.yourdomain.com →  <LoadBalancer-IP>
```

### 7. Install cert-manager (SSL/TLS)

```bash
# Install cert-manager
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.13.0/cert-manager.yaml

# Create ClusterIssuer for Let's Encrypt
cat <<EOF | kubectl apply -f -
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: your-email@example.com  # Update this
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
    - http01:
        ingress:
          class: nginx
EOF
```

### 8. Deploy Ingress

```bash
# Update ingress.yaml with your domain
sed -i 's/yourdomain.com/your-actual-domain.com/g' ingress.yaml

# Apply ingress
kubectl apply -f ingress.yaml

# Verify cert-manager issues certificate
kubectl get certificate -n spaced-repetition -w
```

### 9. Enable Auto-scaling

```bash
# Deploy HPA (requires metrics-server)
kubectl apply -f hpa.yaml

# Verify
kubectl get hpa -n spaced-repetition
```

## SSL/TLS Configuration

### Let's Encrypt Certificates

cert-manager automatically provisions and renews SSL certificates:

```bash
# Check certificate status
kubectl describe certificate spaced-repetition-tls -n spaced-repetition

# Check certificate secret
kubectl get secret spaced-repetition-tls -n spaced-repetition

# View certificate details
kubectl get certificate spaced-repetition-tls -n spaced-repetition -o yaml
```

### Manual Certificate

If using your own certificate:

```bash
kubectl create secret tls spaced-repetition-tls \
  --cert=path/to/tls.crt \
  --key=path/to/tls.key \
  --namespace=spaced-repetition
```

## Monitoring and Scaling

### View Application Status

```bash
# All resources
kubectl get all -n spaced-repetition

# Pods with more details
kubectl get pods -n spaced-repetition -o wide

# Describe pod
kubectl describe pod <pod-name> -n spaced-repetition

# Logs
kubectl logs -f <pod-name> -n spaced-repetition

# Multiple pods
kubectl logs -f -l app=spaced-repetition-server -n spaced-repetition
```

### Horizontal Pod Autoscaler (HPA)

```bash
# View HPA status
kubectl get hpa -n spaced-repetition

# Detailed HPA metrics
kubectl describe hpa spaced-repetition-server-hpa -n spaced-repetition

# Watch scaling
kubectl get hpa -n spaced-repetition -w
```

**HPA Configuration:**

- **Server**: Scales 2-10 pods based on 70% CPU, 80% memory
- **Client**: Scales 2-8 pods based on 70% CPU, 80% memory
- **Scale Up**: Aggressive (50% or 2 pods per 60s)
- **Scale Down**: Conservative (25% or 1 pod per 120s, 5min stabilization)

### Manual Scaling

```bash
# Scale server
kubectl scale deployment spaced-repetition-server --replicas=5 -n spaced-repetition

# Scale client
kubectl scale deployment spaced-repetition-client --replicas=3 -n spaced-repetition
```

### Resource Monitoring

```bash
# Node resource usage
kubectl top nodes

# Pod resource usage
kubectl top pods -n spaced-repetition

# Install metrics-server if not present
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

## Troubleshooting

### Pods Not Starting

```bash
# Check pod status
kubectl get pods -n spaced-repetition

# Describe pod for events
kubectl describe pod <pod-name> -n spaced-repetition

# View logs
kubectl logs <pod-name> -n spaced-repetition

# Previous container logs (if crashed)
kubectl logs <pod-name> -n spaced-repetition --previous
```

### MongoDB Connection Issues

```bash
# Test from within pod
kubectl exec -it <server-pod> -n spaced-repetition -- sh
echo $MONGODB_URI
npm install -g mongosh
mongosh "$MONGODB_URI"

# Check secret
kubectl get secret spaced-repetition-secrets -n spaced-repetition -o yaml
echo "<base64-value>" | base64 -d
```

### Ingress Not Working

```bash
# Check ingress
kubectl get ingress -n spaced-repetition
kubectl describe ingress spaced-repetition-ingress -n spaced-repetition

# Check ingress controller
kubectl get pods -n ingress-nginx

# Check ingress controller logs
kubectl logs -n ingress-nginx -l app.kubernetes.io/name=ingress-nginx
```

### SSL Certificate Issues

```bash
# Check certificate
kubectl get certificate -n spaced-repetition
kubectl describe certificate spaced-repetition-tls -n spaced-repetition

# Check cert-manager logs
kubectl logs -n cert-manager -l app=cert-manager

# Check challenge
kubectl get challenge -n spaced-repetition
```

### Application Errors

```bash
# Server logs
kubectl logs -f -l app=spaced-repetition-server -n spaced-repetition

# Client logs
kubectl logs -f -l app=spaced-repetition-client -n spaced-repetition

# Exec into pod
kubectl exec -it <pod-name> -n spaced-repetition -- sh
```

## Cost Optimization

### Linode LKE Pricing (Estimate)

**Basic Setup (3 nodes):**
- 3x Linode 4GB (g6-standard-2): $36/month each = **$108/month**
- LoadBalancer: **$10/month**
- **Total: ~$118/month**

**With Auto-scaling (Max):**
- If scaled to 10 server + 8 client pods across 5 nodes
- 5x Linode 4GB: **$180/month**

### Cost Reduction Tips

1. **Right-size Nodes**: Start with 2GB nodes if traffic is low
2. **Use Fewer Nodes**: 2 nodes minimum for HA
3. **Auto-scaling**: Automatically scales down during low traffic
4. **MongoDB Atlas**: Use free tier (M0) or shared cluster (M2/M5)
5. **Resource Limits**: Prevent overprovisioning
6. **Spot Instances**: Use spot/preemptible nodes (50-80% discount) if supported

### Shared Cluster (Development)

For development/staging, use smaller cluster:

```bash
# 2 nodes, smaller size
linode-cli lke cluster-create \
  --label spaced-repetition-dev \
  --region us-east \
  --k8s_version 1.28 \
  --node_pools.type g6-nanode-1 \  # 1GB, $5/month
  --node_pools.count 2

# Total: ~$20/month
```

## Updates and Rollbacks

### Rolling Updates

```bash
# Update server image
kubectl set image deployment/spaced-repetition-server \
  server=maxjeffwell/spaced-repetition-capstone-server:v1.1.0 \
  -n spaced-repetition

# Watch rollout
kubectl rollout status deployment/spaced-repetition-server -n spaced-repetition

# Update client
kubectl set image deployment/spaced-repetition-client \
  client=maxjeffwell/spaced-repetition-capstone-client:v1.1.0 \
  -n spaced-repetition
```

### Rollback

```bash
# View rollout history
kubectl rollout history deployment/spaced-repetition-server -n spaced-repetition

# Rollback to previous version
kubectl rollout undo deployment/spaced-repetition-server -n spaced-repetition

# Rollback to specific revision
kubectl rollout undo deployment/spaced-repetition-server --to-revision=2 -n spaced-repetition
```

## Backup and Disaster Recovery

### MongoDB Atlas Backups

Atlas handles automated backups:
- Continuous snapshots
- Point-in-time recovery
- Configurable retention

### Kubernetes Resources

```bash
# Export all resources
kubectl get all -n spaced-repetition -o yaml > backup.yaml

# Backup specific resources
kubectl get deployment,service,ingress -n spaced-repetition -o yaml > resources-backup.yaml

# Backup secrets (careful with this!)
kubectl get secrets -n spaced-repetition -o yaml > secrets-backup.yaml
```

## Next Steps

- [Docker Guide](DOCKER.md) - Local development with Docker
- [CI/CD Guide](CICD.md) - Automated deployments
- [README](README.md) - Project overview

## Additional Resources

- [Linode Kubernetes Engine Docs](https://www.linode.com/docs/guides/deploy-and-manage-lke-cluster/)
- [kubectl Cheat Sheet](https://kubernetes.io/docs/reference/kubectl/cheatsheet/)
- [cert-manager Documentation](https://cert-manager.io/docs/)
- [MongoDB Atlas Documentation](https://docs.atlas.mongodb.com/)
