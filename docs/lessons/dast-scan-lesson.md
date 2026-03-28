# DAST Security Scan Lesson
## Focus: Frontend Service Deployment for Latina App
**Duration: 1 Hour**

---

## Overview

This lesson covers how to implement Dynamic Application Security Testing (DAST) in your Azure DevOps pipeline for the frontend service. You'll learn to add OWASP ZAP scanning to detect security vulnerabilities in your running application.

---

## Learning Objectives

By the end of this lesson, you will be able to:
1. Understand what DAST is and why it's important
2. Add a DAST scan stage to your deployment pipeline
3. Configure dynamic URL discovery from AKS
4. Review and interpret DAST scan results
5. Integrate DAST findings into your security workflow

---

## Prerequisites

- Completed the Pull Request Permissions lesson
- Successfully deployed frontend service to dev environment
- Azure DevOps pipeline running successfully on `dast-scan` branch
- Frontend service deployed with LoadBalancer type in AKS
- Azure CLI access to AKS cluster
- Access to GitHub repositories:
  - https://github.com/denisdbell/latina_app
  - https://github.com/denisdbell/latina_app_template
- **Publish HTML Reports extension** installed in your Azure DevOps organization (see below)

### Required Extension: Publish HTML Reports

The DAST scan produces an HTML report that is published as a pipeline artifact. To view this report directly within Azure DevOps as a separate tab on your pipeline run, you must install the **Publish HTML Reports** extension.

**Install the extension:**

1. Navigate to: **Organization Settings → Extensions → Browse marketplace**  
   (or visit the [Visual Studio Marketplace](https://marketplace.visualstudio.com/) directly)
2. Search for **"Publish HTML Reports"** by **LakshayKaushik**
3. Click **"Get it free"** and install it to your Azure DevOps organization

| Detail | Value |
|--------|-------|
| Extension name | Publish HTML Reports |
| Publisher | LakshayKaushik |
| Version | 1.2.1 (Latest) |
| Purpose | Visualize HTML reports (e.g., OWASP ZAP, JMeter) as a separate tab in Azure DevOps pipeline runs |

> **Note:** Without this extension, you can still download the ZAP report artifact manually, but you will not be able to view it inline within Azure DevOps.


## Part 0: Understand DAST (10 minutes)

### 0.1 What is DAST?

Dynamic Application Security Testing (DAST) analyzes running applications by simulating real-world attacks. Unlike SAST (Static Application Security Testing) which analyzes source code, DAST tests the application from the outside-in.

### 0.2 DAST vs SAST

| Aspect | DAST | SAST |
|--------|------|------|
| When to run | After deployment | During build |
| What it tests | Running application | Source code |
| Findings | Runtime vulnerabilities | Code vulnerabilities |
| False positives | Lower | Higher |
| Requires | Deployed app | Source code only |

### 0.3 OWASP ZAP

OWASP ZAP (Zed Attack Proxy) is a free, open-source DAST tool:
- **Baseline Scan**: Quick scan for low-hanging vulnerabilities
- **Full Scan**: Comprehensive scan including active attacks
- **API Scan**: Tests REST/API endpoints

### 0.4 Where DAST Fits in the Pipeline

```
Build → Deploy to Dev → DAST Scan → Manual Approval → Deploy to Test → Deploy to Prod
```

---

## Step 0: Deploy Azure Infrastructure (30 minutes)

Before configuring Azure DevOps, you need to deploy the Azure infrastructure that the pipelines will deploy to.

### 0.1 Download the main.json

```bash
curl -fsSL https://raw.githubusercontent.com/denisdbell/latina_app/refs/heads/dast-scan/azure/arm/main.json -o main.json
```

### 0.2 Create a resource group

```bash
az group create --name rg-latina --location westus3
```

### 0.3 Deploy the ARM template

```bash
az deployment group create \
  --resource-group rg-latina \
  --template-file main.json
```

### 0.4 Note the deployment outputs

```bash
az deployment group show \
  --resource-group rg-latina \
  --name main \
  --query properties.outputs
```

Keep a note of `acrLoginServer` and `aksName` — you will need them in later steps.

### 0.5 Connect kubectl to the cluster

```bash
az aks get-credentials --resource-group rg-latina --name aks-latina-shared
```

### 0.6 Create Kubernetes namespaces

```bash
curl -fsSL https://raw.githubusercontent.com/denisdbell/latina_app/refs/heads/dast-scan/azure/arm/namespaces.yaml -o namespaces.yaml

kubectl apply -f namespaces.yaml
```

This creates the `dev`, `test`, and `prod` namespaces with network isolation policies and resource quotas.

---

## Step 1: Import Repositories (15 minutes)

### 1.1 Import latina_app Repository

Navigate to: **Repos → Files → Import a repository**

**Step 1:** Click **"Import"** or **"Import a repository"**

**Step 2:** Enter the clone URL:
```
https://github.com/denisdbell/latina_app
```

**Step 3:** Name the repository: `latina_app`

**Step 4:** Click **"Import"** and wait for completion

### 1.2 Import latina_app_template Repository

**Step 1:** Click **"Import a repository"** again

**Step 2:** Enter the clone URL:
```
https://github.com/denisdbell/latina_app_template
```

**Step 3:** Name the repository: `latina_app_template`

**Step 4:** Click **"Import"** and wait for completion

### 1.3 Verify Repository Import

After importing both repositories, verify they appear in your project:

```
Repos → Files
├── latina_app
│   ├── frontend-service/
│   ├── azure/
│   │   ├── pipelines/
│   │   │   └── frontend-service.yml
│   │   └── arm/
│   │       └── main.json
│   └── ...
│
└── latina_app_template
    ├── build.yaml
    ├── deploy.yaml
    └── ...
```

### 1.4 Clone Repository Locally

After importing to Azure DevOps, clone the repository locally to make changes:

**Step 1:** Get the ADO repository URL:

Navigate to: **Repos → Files → Clone**

Copy the HTTPS URL (e.g., `https://dev.azure.com/org/project/_git/latina_app`)

**Step 2:** Clone locally:

```bash
# Clone the repository
git clone https://dev.azure.com/denisemmanuel/LatinaApp/_git/latina_app

# Navigate to the repository
cd latina_app
```

### 1.5 Checkout the dast-scan Branch

**Step 1:** Fetch and checkout the branch:

```bash
# Fetch all remote branches
git fetch --all

# List remote branches to verify dast-scan exists
git branch -r

# Checkout the dast-scan branch
git checkout dast-scan

# Pull latest changes
git pull origin dast-scan
```

**Step 2:** If the branch doesn't exist locally, create it from remote:

```bash
# Create local branch tracking remote
git checkout -b dast-scan origin/dast-scan
```

### 1.6 Add GitHub as Additional Remote (Optional)

If you want to push changes to GitHub as well:

```bash
# Add GitHub as a remote named 'github'
git remote add github https://github.com/denisdbell/latina_app.git

# Verify remotes
git remote -v

# Push to both remotes
git push origin dast-scan
git push github dast-scan
```

---

## Part 1: Create Service Connections (20 minutes)

The DAST stage needs access to AKS to discover the frontend service URL, and the pipeline needs ACR access for container images.

### 1.1 Understanding Required Service Connections

| Service Connection | Type | Purpose |
|-------------------|------|---------|
| `aks-service-connection` | Kubernetes | Deploy to AKS cluster |
| `acr-shared-service-connection` | Azure Container Registry | Push/pull container images |
| `arm-service-connection` | Azure Resource Manager | Get AKS credentials for URL discovery |

### 1.2 Create AKS Service Connection

Navigate to: **Project Settings → Service connections → New service connection**

**Step 1:** Select **"Kubernetes"** → **"Azure Kubernetes Service"**

**Step 2:** Configure:
- Connection name: `aks-service-connection`
- Subscription: Select your Azure subscription
- Cluster: `aks-latina-shared`
- Namespace: Leave empty (will be specified per deployment)

**Step 3:** Click **"Verify and Save"**

### 1.3 Create ACR Service Connection

**Step 1:** Click **"New service connection"** → Select **"Docker Registry"** → **"Azure Container Registry"**

**Step 2:** Configure:
- Connection name: `acr-shared-service-connection`
- Subscription: Select your Azure subscription
- Container registry: `acrlatinashared<uniqueSuffix>` (from ARM deployment)
- Grant access to all pipelines: **Yes**

**Step 3:** Click **"Verify and Save"**

### 1.4 Create ARM Service Connection

**Step 1:** Click **"New service connection"** → Select **"Azure Resource Manager"**

**Step 2:** Select **"Service principal (automatic)"**

**Step 3:** Configure:
- Scope level: **Subscription**
- Subscription: Select your Azure subscription
- Resource Group: Leave empty (access to all resources)
- Connection name: `arm-service-connection`
- Grant access permission to all pipelines: **Checked**

**Step 4:** Click **"Save"**

### 1.5 Verify Service Connections

Navigate to: **Project Settings → Service connections**

Ensure you see:
```
Service connections
├── aks-service-connection (Kubernetes)
├── acr-shared-service-connection (Azure Container Registry)
└── arm-service-connection (Azure Resource Manager)
```

---

## Part 2: Create and Configure Pipeline (25 minutes)

### 2.1 Review the Pipeline YAML

The `dast-scan` branch contains the DAST stage in `azure/pipelines/frontend-service.yml`. Review locally:

```bash
cat azure/pipelines/frontend-service.yml
```

Key sections to verify:

| Section | Purpose |
|---------|---------|
| `trigger: branches: include: - dast-scan` | Pipeline triggers on dast-scan branch |
| `KubectlInstaller@0` | Installs kubectl on agent |
| `AzureCLI@2` | Gets frontend URL from AKS |
| `docker run` | Runs OWASP ZAP scan |
| `PublishBuildArtifacts@1` | Publishes scan report |

### 2.2 Verify Pipeline Variables

The pipeline includes these variables:

```yaml
variables:
  aksResourceGroup: 'rg-latina'
  aksClusterName: 'aks-latina-shared'
  armServiceConnection: 'arm-service-connection'
```

Ensure these match your Azure deployment.

### 2.3 Create the Pipeline in Azure DevOps

Navigate to: **Pipelines → New pipeline**

**Step 1:** Select **"Azure Repos Git"**

**Step 2:** Select the `latina_app` repository

**Step 3:** Select **"Existing Azure Pipelines YAML file"**

**Step 4:** Configure:
- Branch: `dast-scan`
- Path: `/azure/pipelines/frontend-service.yml`

**Step 5:** Click **"Continue"**

**Step 6:** Review the YAML and click **"Save"** (do not run yet)

### 2.4 Populate Pipeline Variables

Before running the pipeline, set the `uniqueSuffix` variable:

Navigate to: **Pipelines → frontend-service → Variables**

**Step 1:** Retrieve the `uniqueSuffix` from your ARM deployment:
```bash
az deployment group show --resource-group rg-latina --name main --query properties.outputs.uniqueSuffix.value
```

**Step 2:** Add the variable:

| Variable | Value | Notes |
|----------|-------|-------|
| `uniqueSuffix` | `<from ARM deployment>` | e.g., `rs25m4je` |

### 2.5 Verify Service Connection Access

Navigate to: **Project Settings → Service connections → arm-service-connection**

**Step 1:** Click **"..."** → **"Security"**

**Step 2:** Ensure **"Grant access permission to all pipelines"** is checked

**Step 3:** If prompted, approve access for the `frontend-service` pipeline

### 2.6 Push Changes to Trigger Pipeline

If you made any local changes, push to the `dast-scan` branch:

```bash
# Push to ADO
git push origin dast-scan

# Optionally push to GitHub as well
git push github dast-scan
```

---

## Part 3: Trigger and Monitor DAST Scan (15 minutes)

### 3.1 Run the Pipeline

Navigate to: **Pipelines → frontend-service**

**Step 1:** Click **"Run pipeline"**

**Step 2:** Select branch: `dast-scan`

**Step 3:** Click **"Run"**

### 3.2 Monitor Pipeline Execution

The pipeline will execute in order:

```
1. Build Stage
   └── Compiles and packages frontend service

2. Deploy to Dev
   └── Deploys to development environment

3. DAST Security Scan
   ├── Installs kubectl
   ├── Discovers frontend URL from AKS
   ├── Pulls OWASP ZAP Docker image
   ├── Runs baseline scan
   └── Publishes security report

4. Manual Approval
   └── Review DAST results and approve

5. Deploy to Test
   └── Promotes to testing environment

6. Deploy to Prod
   └── Promotes to production environment
```

### 3.3 Exercise: Monitor DAST Stage

**Step 1:** Click on the **"DAST Security Scan"** stage when it starts

**Step 2:** Monitor the tasks:
- **Install Kubectl** - Should complete in seconds
- **Get Frontend Service URL from AKS** - Should show the discovered URL
- **Run OWASP ZAP DAST Scan** - Takes 2-5 minutes
- **Publish DAST Report** - Uploads the report artifact

**Step 3:** Watch for the URL discovery output:
```
Frontend Service URL: http://<EXTERNAL_IP>
```

**Step 4:** If URL discovery fails, check:
- ARM service connection has correct permissions
- Frontend service is deployed in `dev` namespace
- Service has LoadBalancer type with external IP

### 3.4 Review DAST Results

After the DAST scan completes:

**Step 1:** Click on the **"DASTScan"** stage

**Step 2:** Click on the **"Publish DAST Report"** task

**Step 3:** Navigate to the **"Artifacts"** section

**Step 4:** Download **"zap-security-report"** artifact

**Step 5:** Extract and open `zap-report.html` in a browser

### 3.5 Common DAST Findings

| Finding | Severity | Mitigation |
|---------|----------|-------------|
| Missing X-Frame-Options | Medium | Add security headers |
| Cookie without HttpOnly | Medium | Set HttpOnly flag |
| Cookie without Secure | High | Use Secure flag |
| Cross-Domain JavaScript | Low | Review external scripts |
| Information Disclosure | Informational | Remove sensitive info |

---

## Part 4: Manual Approval and Promotion (10 minutes)

### 4.1 Approve Deployment to Test

After DAST scan completes, the pipeline pauses at **"Manual Approval - Promote to Testing"**

**Step 1:** Navigate to: **Pipelines → frontend-service → [Current Run]**

**Step 2:** Click on the **"ApproveTesting"** stage

**Step 3:** Review:
- DAST scan completed successfully
- Check the published artifact for any critical findings
- Verify no high-severity vulnerabilities

**Step 4:** Click **"Resume"**

**Step 5:** Add comments (optional): "DAST scan reviewed, no critical findings"

### 4.2 Approve Deployment to Prod

After Test deployment succeeds:

**Step 1:** Click on the **"ApproveProd"** stage

**Step 2:** Review test deployment status

**Step 3:** Click **"Resume"**

---

## Part 5: Advanced DAST Configuration (Optional)

### 5.1 Full Scan vs Baseline Scan

The default uses baseline scan. For more thorough testing:

Edit `azure/pipelines/frontend-service.yml` and replace the scan script:

```yaml
- script: |
    mkdir -p $(Pipeline.Workspace)/zap-reports
    chmod 777 $(Pipeline.Workspace)/zap-reports

    docker run --rm \
      -v $(Pipeline.Workspace)/zap-reports:/zap/wrk:rw \
      ghcr.io/zaproxy/zaproxy:stable \
      zap-full-scan.py \
        -t "$(dastTargetUrl)" \
        -r zap-report.html
  displayName: 'Run OWASP ZAP Full Scan'
```

### 5.2 Exclude Paths from Scan

To skip certain endpoints (like logout):

```yaml
- script: |
    docker run --rm \
      -v $(Pipeline.Workspace)/zap-reports:/zap/wrk:rw \
      ghcr.io/zaproxy/zaproxy:stable \
      zap-baseline.py \
        -t "$(dastTargetUrl)" \
        -r zap-report.html \
        -I \
        -e "/logout,/health"
  displayName: 'Run OWASP ZAP DAST Scan'
```

### 5.3 Configure Branch Trigger

To run DAST on a different branch, update the trigger and conditions:

```yaml
trigger:
  branches:
    include:
      - main  # or your production branch

# Update all stage conditions:
condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main'))
```

---

## Summary

### Pre-Setup Checklist

Before starting, ensure the following are completed in order:

```
1. [ ] Deploy Azure infrastructure (Step 0)
2. [ ] Import latina_app repository from GitHub
3. [ ] Import latina_app_template repository from GitHub
4. [ ] Clone repository locally and checkout dast-scan branch
5. [ ] Create aks-service-connection (Kubernetes)
6. [ ] Create acr-shared-service-connection (Azure Container Registry)
7. [ ] Create arm-service-connection (Azure Resource Manager)
8. [ ] Create pipeline from azure/pipelines/frontend-service.yml
9. [ ] Set uniqueSuffix variable from ARM deployment
10. [ ] Verify pipeline references latina_app_template repository
11. [ ] Frontend service deployed to AKS dev namespace
12. [ ] Frontend service has LoadBalancer type with external IP
13. [ ] Pipeline triggered and monitored
```

### Pipeline Flow with DAST

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                           Frontend Pipeline with DAST                                    │
├─────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                         │
│   1. Build Stage                                                                        │
│      └── Compiles and packages frontend service                                         │
│                                                                                         │
│   2. Deploy to Dev                                                                      │
│      └── Deploys to development environment                                             │
│                                                                                         │
│   3. DAST Security Scan                                                                 │
│      ├── Installs kubectl                                                               │
│      ├── Discovers frontend URL from AKS                                                │
│      ├── Pulls OWASP ZAP Docker image                                                   │
│      ├── Runs baseline scan against running application                                 │
│      └── Publishes security report as pipeline artifact                                │
│                                                                                         │
│   4. Manual Approval                                                                    │
│      └── Review DAST results and approve                                                 │
│                                                                                         │
│   5. Deploy to Test                                                                     │
│      └── Promotes to testing environment                                                │
│                                                                                         │
│   6. Deploy to Prod                                                                     │
│      └── Promotes to production environment                                             │
│                                                                                         │
└─────────────────────────────────────────────────────────────────────────────────────────┘
```

### Key Takeaways

| Concept | Implementation |
|---------|----------------|
| DAST Tool | OWASP ZAP (Docker-based) |
| Scan Type | Baseline scan (quick, non-intrusive) |
| Timing | After dev deployment, before test promotion |
| URL Discovery | Dynamic from AKS LoadBalancer service |
| Report | Pipeline artifact (zap-security-report) |
| Branch Trigger | `dast-scan` branch |

### Security Checklist

- [ ] latina_app repository imported from GitHub
- [ ] latina_app_template repository imported from GitHub
- [ ] aks-service-connection created and verified
- [ ] acr-shared-service-connection created and verified
- [ ] arm-service-connection created and verified
- [ ] Pipeline created from YAML file
- [ ] uniqueSuffix variable populated from ARM deployment
- [ ] Pipeline triggered on `dast-scan` branch
- [ ] Frontend service deployed with LoadBalancer type
- [ ] DAST stage completes successfully
- [ ] DAST report reviewed for vulnerabilities
- [ ] Manual approval references DAST results
- [ ] Security findings addressed before production

---

## Additional Resources

### Azure DevOps Documentation
- [Azure CLI task](https://learn.microsoft.com/en-us/azure/devops/pipelines/tasks/deploy/azure-cli)
- [Publish Build Artifacts](https://learn.microsoft.com/en-us/azure/devops/pipelines/artifacts/build-artifacts)
- [Pipeline conditions](https://learn.microsoft.com/en-us/azure/devops/pipelines/process/conditions)

### OWASP ZAP Documentation
- [OWASP ZAP Getting Started](https://www.zaproxy.org/getting-started/)
- [ZAP Baseline Scan](https://www.zaproxy.org/docs/docker/baseline-scan/)
- [ZAP Full Scan](https://www.zaproxy.org/docs/docker/full-scan/)

### Security Best Practices
- [OWASP Top 10](https://owasp.org/Top10/)
- [Web Security Guidelines](https://cheatsheetseries.owasp.org/)

---

## Q&A

### Common Questions

**Q: Why does DAST need to discover the URL dynamically?**
A: In Kubernetes environments, LoadBalancer IPs can change. Dynamic discovery ensures the scan always targets the correct endpoint.

**Q: What's the difference between baseline and full scan?**
A: Baseline scan is non-intrusive and quick (minutes). Full scan includes active attacks and may modify data. Use baseline for CI/CD, full for scheduled security audits.

**Q: Can DAST scan authenticated pages?**
A: Yes, but requires additional configuration with authentication headers or recorded login scripts.

**Q: Should the pipeline fail on DAST findings?**
A: Initially, use `-I` flag to ignore warnings. Later, configure stricter gates based on your security requirements.

**Q: How do I test the DAST stage locally?**
A: Run the Docker command locally:
```bash
docker run --rm -v $(pwd)/reports:/zap/wrk:rw \
  ghcr.io/zaproxy/zaproxy:stable \
  zap-baseline.py -t "http://your-service-url" -r report.html -I
```

---

## Troubleshooting

### Common Issues

| Issue | Solution |
|-------|----------|
| Docker pull fails | Ensure pipeline agent has Docker installed and network access |
| Cannot resolve service URL | Verify frontend-service has LoadBalancer type: `kubectl get svc frontend-service -n dev` |
| AKS credentials fail | Check `armServiceConnection` has subscription-level access |
| Pipeline runs on wrong branch | Verify trigger and condition match `dast-scan` branch |
| No external IP | Service may still be provisioning; wait and retry |
| Permission denied on reports | Ensure `chmod 777` is run on zap-reports directory |

### Debugging Commands

If the frontend URL cannot be discovered, run locally:

```bash
# Check service exists
kubectl get svc -n dev

# Check service type
kubectl get svc frontend-service -n dev -o jsonpath='{.spec.type}'

# Check LoadBalancer status
kubectl get svc frontend-service -n dev -o jsonpath='{.status.loadBalancer}'

# Test connectivity from inside cluster
kubectl run curl-test --image=curlimages/curl -it --rm -- curl http://frontend-service.dev.svc.cluster.local:80
```

---

*This lesson extends the Latina App frontend service deployment pipeline with DAST security scanning capabilities.*
