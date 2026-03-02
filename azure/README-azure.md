# Azure Deployment Guide - Inspire App

This guide covers deploying the three-tier Inspire App to Azure Kubernetes Service (AKS) using ARM templates and Azure DevOps pipelines.

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     Azure Infrastructure                      │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  ┌─────────────────┐    ┌─────────────────────────────────┐ │
│  │ Azure Container │    │   Azure Kubernetes Service      │ │
│  │    Registry     │───▶│         (AKS Cluster)           │ │
│  │     (ACR)       │    │                                 │ │
│  └─────────────────┘    │  ┌───────────────────────────┐  │ │
│                         │  │ inspire namespace         │  │ │
│                         │  │  ├── frontend-service     │  │ │
│                         │  │  ├── image-service        │  │ │
│                         │  │  └── phrase-service       │  │ │
│                         │  └───────────────────────────┘  │ │
│                         └─────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

---

## Prerequisites

- Azure subscription with Owner or Contributor access
- Azure CLI installed (`az`)
- kubectl installed
- Docker installed
- Azure DevOps project with access to create pipelines

---

## Repository Structure

```
azure/
├── arm/
│   ├── main.json                      # Main ARM template
│   ├── main.parameters.dev.json       # Dev environment parameters
│   ├── main.parameters.test.json      # Test environment parameters
│   └── main.parameters.prod.json      # Prod environment parameters
├── pipelines/
│   ├── image-service.yml              # Image service pipeline
│   ├── phrase-service.yml             # Phrase service pipeline
│   └── frontend-service.yml           # Frontend service pipeline
└── templates/
    ├── build.yaml                     # Shared build template
    ├── deploy.yaml                    # Shared deploy template
    └── promote-acr.yaml               # ACR promotion template
```

---

## Step 1: Create Resource Groups

Create a resource group for each environment:

```bash
# Dev
az group create --name rg-inspire-dev --location eastus

# Test
az group create --name rg-inspire-test --location eastus

# Prod
az group create --name rg-inspire-prod --location eastus
```

---

## Step 2: Deploy ARM Templates

### Deploy to Development

```bash
az deployment group create \
  --resource-group rg-inspire-dev \
  --template-file azure/arm/main.json \
  --parameters @azure/arm/main.parameters.dev.json
```

### Deploy to Testing

```bash
az deployment group create \
  --resource-group rg-inspire-test \
  --template-file azure/arm/main.json \
  --parameters @azure/arm/main.parameters.test.json
```

### Deploy to Production

```bash
az deployment group create \
  --resource-group rg-inspire-prod \
  --template-file azure/arm/main.json \
  --parameters @azure/arm/main.parameters.prod.json
```

### Capture Deployment Outputs

After deployment, note the outputs:

```bash
# Get ACR login server
az acr show --name acrinspiredev --query loginServer -o tsv

# Get AKS credentials
az aks get-credentials --resource-group rg-inspire-dev --name aks-inspire-dev
```

---

## Step 3: Grant ACR Pull Access to AKS

The AKS cluster needs permission to pull images from ACR:

```bash
# Get AKS kubelet identity object ID
AKS_OBJECT_ID=$(az aks show --resource-group rg-inspire-dev --name aks-inspire-dev --query identityProfile.kubeletidentity.objectId -o tsv)

# Get ACR resource ID
ACR_ID=$(az acr show --name acrinspiredev --query id -o tsv)

# Assign AcrPull role
az role assignment create --assignee $AKS_OBJECT_ID --role AcrPull --scope $ACR_ID
```

Repeat for test and prod environments.

---

## Step 4: Configure Azure DevOps

### 4.1 Create Service Connections

Create the following service connections in Azure DevOps:

| Name | Type | Purpose |
|------|------|---------|
| `aks-service-connection` | Kubernetes | Connect to AKS cluster |
| `dev-acr-service-connection` | Docker Registry | Dev ACR |
| `test-acr-service-connection` | Docker Registry | Test ACR |
| `prod-acr-service-connection` | Docker Registry | Prod ACR |

#### Create Kubernetes Service Connection

1. Go to Project Settings > Service connections
2. New service connection > Kubernetes
3. Select "Azure Subscription"
4. Choose subscription, cluster, and namespace

#### Create Docker Registry Service Connection

1. Go to Project Settings > Service connections
2. New service connection > Docker Registry
3. Select "Azure Container Registry"
4. Choose subscription and ACR

### 4.2 Create Pipeline Templates Repository

Create a repository named `pipeline-templates` in your Azure DevOps project:

```bash
# Create a new repository
git init pipeline-templates
cd pipeline-templates

# Copy templates
cp -r ../azure/templates/* .

# Push to Azure DevOps
git remote add origin https://dev.azure.com/your-org/your-project/_git/pipeline-templates
git add .
git commit -m "Add pipeline templates"
git push -u origin main
```

### 4.3 Update Pipeline Variables

Edit each pipeline file and update:

```yaml
variables:
  # Update the unique suffix from your ARM deployment
  uniqueSuffix: 'dev001'  # Replace with actual suffix

  # Update service connection names if different
  aksServiceConnection: 'aks-service-connection'
  devAcrConnection: 'dev-acr-service-connection'
```

---

## Step 5: Create Pipelines

### Create Pipeline for Each Service

1. Go to Pipelines > New pipeline
2. Select "Azure Repos Git"
3. Select your repository
4. Select "Existing Azure Pipelines YAML file"
5. Select the pipeline file:
   - `/azure/pipelines/image-service.yml`
   - `/azure/pipelines/phrase-service.yml`
   - `/azure/pipelines/frontend-service.yml`

---

## Step 6: Manual Deployment (Alternative)

If you prefer manual deployment without pipelines:

### Build and Push Images

```bash
# Login to ACR
az acr login --name acrinspiredev

# Set ACR login server
ACR=acrinspiredev.azurecr.io

# Build and push images
docker build -f image-service/Dockerfile -t $ACR/image-service:latest .
docker build -f phrase-service/Dockerfile -t $ACR/phrase-service:latest .
docker build -f frontend-service/Dockerfile -t $ACR/frontend-service:latest .

docker push $ACR/image-service:latest
docker push $ACR/phrase-service:latest
docker push $ACR/frontend-service:latest
```

### Deploy to AKS

```bash
# Get AKS credentials
az aks get-credentials --resource-group rg-inspire-dev --name aks-inspire-dev

# Update deployment manifests with ACR login server
sed -i "s|<ACR_LOGIN_SERVER>|$ACR|g" image-service/k8s/deployment.yaml
sed -i "s|<ACR_LOGIN_SERVER>|$ACR|g" phrase-service/k8s/deployment.yaml
sed -i "s|<ACR_LOGIN_SERVER>|$ACR|g" frontend-service/k8s/deployment.yaml

# Apply Kubernetes manifests
kubectl apply -f k8s/namespace.yaml

kubectl apply -f image-service/k8s/service.yaml
kubectl apply -f image-service/k8s/deployment.yaml

kubectl apply -f phrase-service/k8s/service.yaml
kubectl apply -f phrase-service/k8s/deployment.yaml

kubectl apply -f frontend-service/k8s/service.yaml
kubectl apply -f frontend-service/k8s/deployment.yaml
```

### Verify Deployment

```bash
# Watch pods become ready
kubectl get pods -n inspire --watch

# Get the public IP
kubectl get svc frontend-service -n inspire

# Open http://<EXTERNAL-IP> in your browser
```

---

## Pipeline Flow

```
┌──────────────────────────────────────────────────────────────────────┐
│                         Pipeline Stages                               │
├──────────────────────────────────────────────────────────────────────┤
│                                                                       │
│  ┌─────────┐    ┌──────────┐    ┌─────────┐    ┌──────────┐         │
│  │  Build  │───▶│ DeployDev│───▶│ Approval│───▶│DeployTest│         │
│  │         │    │          │    │ Testing │    │          │         │
│  └─────────┘    └──────────┘    └─────────┘    └──────────┘         │
│                                                       │              │
│                                                       ▼              │
│  ┌─────────┐    ┌──────────┐                           │             │
│  │ Deploy  │◀───│ Approval │◀──────────────────────────┘             │
│  │  Prod   │    │   Prod   │                                         │
│  └─────────┘    └──────────┘                                         │
│                                                                       │
└──────────────────────────────────────────────────────────────────────┘
```

### Stage Descriptions

| Stage | Description |
|-------|-------------|
| Build | Maven build + tests, Docker build & push to Dev ACR |
| DeployDev | Deploy to development AKS namespace |
| ApproveTesting | Manual validation gate |
| DeployTest | Promote image to Test ACR, deploy to test namespace |
| ApproveProd | Manual validation gate |
| DeployProd | Promote image to Prod ACR, deploy to prod namespace |

---

## Environment Variables

| Variable | Service | Default | Description |
|----------|---------|---------|-------------|
| `IMAGE_SERVICE_URL` | frontend-service | `http://image-service:8081` | URL of image-service |
| `PHRASE_SERVICE_URL` | frontend-service | `http://phrase-service:8082` | URL of phrase-service |
| `SERVER_PORT` | all | `8080` | HTTP port |

---

## Scaling

### Manual Scaling

```bash
# Scale a deployment
kubectl scale deployment image-service --replicas=3 -n inspire
```

### Auto-scaling (HPA)

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: image-service-hpa
  namespace: inspire
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: image-service
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

---

## Monitoring

### View Pod Logs

```bash
# Get pod name
kubectl get pods -n inspire

# Stream logs
kubectl logs -f <pod-name> -n inspire
```

### Check Pod Status

```bash
kubectl describe pod <pod-name> -n inspire
```

### View Events

```bash
kubectl get events -n inspire --sort-by='.lastTimestamp'
```

---

## Troubleshooting

### Image Pull Errors

```bash
# Check if ACR credentials are configured
kubectl get secrets -n inspire

# Create image pull secret if needed
kubectl create secret docker-registry acr-secret \
  --docker-server=<acr-login-server> \
  --docker-username=<acr-username> \
  --docker-password=<acr-password> \
  -n inspire
```

### Service Connectivity

```bash
# Test DNS resolution
kubectl run test --rm -it --image=busybox -- nslookup image-service.inspire.svc.cluster.local

# Test service endpoint
kubectl run test --rm -it --image=curlimages/curl -- curl http://image-service.inspire:8081/image
```

---

## Cost Optimization

### Use Spot Instances for Dev/Test

Add to ARM template parameters:

```json
{
  "aksAgentPoolSpotInstances": {
    "value": true
  }
}
```

### Scale Down Outside Business Hours

```bash
# Scale down AKS nodes
az aks scale --resource-group rg-inspire-dev --name aks-inspire-dev --node-count 1
```

---

## Security Recommendations

1. **Use Managed Identity** for AKS to access ACR (already configured)
2. **Enable Azure Policy** for AKS
3. **Use Azure Key Vault** for secrets management
4. **Enable Azure Monitor** for logging and monitoring
5. **Configure Network Policies** to restrict pod-to-pod communication
6. **Use Private ACR** with Private Endpoints for production

---

## Clean Up Resources

```bash
# Delete resource groups (removes all resources)
az group delete --name rg-inspire-dev --yes --no-wait
az group delete --name rg-inspire-test --yes --no-wait
az group delete --name rg-inspire-prod --yes --no-wait
```

---

## Useful Commands Reference

| Task | Command |
|------|---------|
| Get AKS credentials | `az aks get-credentials -g <rg> -n <aks>` |
| Login to ACR | `az acr login --name <acr>` |
| List ACR images | `az acr repository list --name <acr>` |
| View pods | `kubectl get pods -n inspire` |
| View services | `kubectl get svc -n inspire` |
| View deployments | `kubectl get deployments -n inspire` |
| Stream logs | `kubectl logs -f <pod> -n inspire` |
| Describe resource | `kubectl describe <resource> <name> -n inspire` |
| Delete all resources | `kubectl delete all --all -n inspire` |