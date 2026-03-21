# Pull Request Permissions & Policies Lesson
## Focus: Frontend Service Deployment for Latina App
**Duration: 2 Hours**

---

## Overview

This lesson covers how to implement pull request (PR) permissions and policies in Azure DevOps to secure your frontend service deployment pipeline. You'll learn to enforce code quality, control deployments across environments, and establish proper approval workflows.

---

## Learning Objectives

By the end of this lesson, you will be able to:
1. Import repositories into Azure DevOps
2. Create required service connections for pipeline deployment
3. Configure and populate pipeline variables
4. Configure branch protection policies in Azure DevOps
5. Set up required reviewers and code review workflows
6. Implement build validation policies for frontend changes

---

## Prerequisites

- Azure DevOps project created
- Access to GitHub repositories:
  - https://github.com/denisdbell/latina_app
  - https://github.com/denisdbell/latina_app_template
- Basic understanding of Git workflows

---

## Step 0: Deploy Azure Infrastructure (30 minutes)

Before configuring Azure DevOps, you need to deploy the Azure infrastructure that the pipelines will deploy to.

### 0.1 Download the main.json

```bash
curl -fsSL https://raw.githubusercontent.com/denisdbell/latina_app/refs/heads/pull-request-permissions-lesson.md/azure/arm/main.json -o main.json
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
curl -fsSL https://raw.githubusercontent.com/denisdbell/latina_app/refs/heads/pull-request-permissions-lesson.md/azure/arm/namespaces.yaml -o namespaces.yaml

kubectl apply -f namespaces.yaml
```

This creates the `dev`, `test`, and `prod` namespaces with network isolation policies and resource quotas.

---

## Part 1: Import Repositories (15 minutes)

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

---

## Part 2: Create Service Connections (20 minutes)

### 2.1 Understanding Required Service Connections

Based on the `azure/arm/main.json` and pipeline configuration, you need to create the following service connections:

| Service Connection | Type | Purpose | Used In Environments |
|-------------------|------|---------|---------------------|
| `aks-service-connection` | Azure Kubernetes Service | Deploy to AKS cluster | dev, test, prod |
| `acr-shared-service-connection` | Azure Container Registry | Push/pull container images | dev, test, prod |

### 2.2 Create AKS Service Connection

Navigate to: **Project Settings → Service connections → New service connection**

**Step 1:** Select **"Kubernetes"** → **"Azure Kubernetes Service"**

**Step 2:** Configure:
- Connection name: `aks-service-connection`
- Subscription: Select your Azure subscription
- Cluster: `aks-latina-shared`
- Namespace: Leave empty (will be specified per deployment)

**Step 3:** Click **"Verify and Save"**

### 2.3 Create ACR Service Connection

**Step 1:** Click **"New service connection"** → Select **"Docker Registry"** → **"Azure Container Registry"**

**Step 2:** Configure:
- Connection name: `acr-shared-service-connection`
- Subscription: Select your Azure subscription
- Container registry: `acrlatinashared<uniqueSuffix>` (from ARM deployment)
- Grant access to all pipelines: **Yes**

**Step 3:** Click **"Verify and Save"**

### 2.4 Verify Service Connections

Ensure both service connections appear in your list:

```
Project Settings → Service connections
├── aks-service-connection (Kubernetes)
└── acr-shared-service-connection (Azure Container Registry)
```

---

## Part 3: Create and Configure Pipeline (25 minutes)

### 3.1 Create the Pipeline

Navigate to: **Pipelines → New pipeline**

**Step 1:** Select **"Azure Repos Git"**

**Step 2:** Select the `latina_app` repository

**Step 3:** Select **"Existing Azure Pipelines YAML file"**

**Step 4:** Configure:
- Branch: `pull-request-permission`
- Path: `/azure/pipelines/frontend-service.yml`

**Step 5:** Click **"Continue"**

### 3.2 Populate Pipeline Variables

Before saving the pipeline, you need to update the variables based on your ARM deployment outputs.

**Step 1:** Retrieve the `uniqueSuffix` from your ARM deployment:
```
az deployment group show --resource-group <your-rg> --name <deployment-name> --query properties.outputs.uniqueSuffix.value
```

Or from the Azure portal, check the deployment outputs.

**Step 2:** Update pipeline variables:

| Variable | Value | Notes |
|----------|-------|-------|
| `uniqueSuffix` | `<from ARM deployment>` | e.g., `rs25m4je` |
| `aksServiceConnection` | `aks-service-connection` | Already set |
| `imageRepository` | `frontend-service` | Already set |

**Step 3:** Save the pipeline:
- Click **"Save"** (do not run yet)
- The pipeline must exist before branch policies can reference it

### 3.3 Verify Pipeline References Template Repository

The pipeline references templates from `latina_app_template`:

```yaml
resources:
  repositories:
    - repository: templates
      type: git
      name: LatinaApp/latina_app_template
      ref: main
```

Ensure `latina_app_template` repository is imported and accessible.

### 3.4 Pipeline Variable Configuration Summary

Based on `azure/arm/main.json` outputs and pipeline requirements:

```yaml
# Required Pipeline Variables
uniqueSuffix: '<from-deployment>'        # ARM output
aksServiceConnection: 'aks-service-connection'
devAcrConnection: 'acr-shared-service-connection'
testAcrConnection: 'acr-shared-service-connection'
prodAcrConnection: 'acr-shared-service-connection'

# Derived Values (automatically computed)
devAcrLoginServer: 'acrlatinashared$(uniqueSuffix).azurecr.io'
testAcrLoginServer: 'acrlatinashared$(uniqueSuffix).azurecr.io'
prodAcrLoginServer: 'acrlatinashared$(uniqueSuffix).azurecr.io'
```

---

## Part 4: Branch Protection Policies (30 minutes)

### 4.1 Setting pull-request-permission as Main Branch

Before configuring policies, ensure `pull-request-permission` is set as the main branch.

**Step 1:** Navigate to: **Project Settings → Repositories → latina_app**

**Step 2:** Click **"Settings"** or **"Branch policies"**

**Step 3:** Set default branch to `pull-request-permission`:
- If needed, go to **Repos → Branches**
- Hover over `pull-request-permission` branch
- Click **"..."** → **"Set as default branch"**

### 4.2 Configure Branch Policies

Navigate to: **Repos → Branches → pull-request-permission → Branch Policies**

### Required Policies to Enable

| Policy | Purpose | Settings |
|--------|---------|----------|
| **Minimum reviewers** | Ensures code review | 1 reviewer minimum |
| **Require merge strategy** | Commit history control | All strategies enabled |
| **Build validation** | Automated testing | Frontend pipeline |
| **Check for linked work items** | Traceability | Required |

### 4.3 Exercise: Configure Branch Policies

#### A. Minimum Reviewers

1. Toggle **"Require a minimum number of reviewers"** to ON
2. Set minimum reviewers to **1**
3. **IMPORTANT:** Check **"Allow requesters to approve their own changes"** to ON
   - This allows a single user to create and approve their own PR
4. Check **"Allow completion even if some reviewers vote to wait or reject"** to OFF

#### B. Merge Strategies

1. Toggle **"Require a merge strategy"** to ON
2. **Enable ALL merge strategies:**
   - Squash merge
   - Merge commit
   - Rebase and fast-forward
3. This ensures flexibility in how changes are integrated

#### C. Build Validation

1. Toggle **"Build validation"** to ON
2. Select build pipeline: `frontend-service` pipeline (created in Part 3)
3. Set trigger: **Run on every update**
4. Enable **"Require successful build completion"**

#### D. Work Item Linking

1. Toggle **"Check for linked work items"** to ON
2. Set to **Required**

### 4.4 Adding a Single User as Required Reviewer

**Step 1:** Navigate to branch policies (same screen)

**Step 2:** Under **"Require reviewers"** section:
1. Click **"Add reviewers"**
2. In the search box, type the user's name or email
3. Select the specific user (not a group)
4. Set to **Required** (not Optional)
5. Click **"Add"**

**Step 3:** Save policies

---

## Part 5: Repository Permissions (15 minutes)

### 5.1 Understanding Permission Levels

| Role | Read | Contribute | Create Branch | Force Push | Bypass Policies |
|------|------|-----------|---------------|------------|-----------------|
| Reader | Yes | No | No | No | No |
| Contributor | Yes | Yes | Yes | No | No |
| Administrator | Yes | Yes | Yes | Yes | Yes |

### 5.2 Exercise: Configure User Permissions

Navigate to: **Project Settings → Repositories → latina_app → Security**

#### Configure Single User Permissions:

1. Find the user in the list or use search
2. Grant the following permissions:

```
User Permissions for latina_app:
├── Read: Allow
├── Contribute: Allow
├── Create Branch: Allow
├── Create Tag: Allow
├── Manage Note: Not Set
├── Force Push: Deny
├── Bypass Policies: Not Set
└── Create Repository: Not Set
```

3. Click **"Save changes"**

---

## Part 6: Environment Approvals and Checks (15 minutes)

### 6.1 Understanding Environment-Based Gates

The pipeline uses **Manual Validation** tasks for approvals between environments:

```yaml
- stage: ApproveTesting
  displayName: 'Manual Approval - Promote to Testing'
  dependsOn: DeployDev
  condition: succeeded()
  jobs:
  - job: WaitForApprovalTesting
    pool: server
    timeoutInMinutes: 1440
    steps:
    - task: ManualValidation@0
      inputs:
        instructions: 'Please validate Dev deployment and approve promotion to Testing.'
```

### 6.2 Configure Environment Approvals

**Step 1:** Navigate to Environments

```
Pipelines → Environments
```

Create (if not exists):
- `dev`
- `testing`
- `prod`

**Step 2:** Configure approvals for each environment

#### For `prod` environment:

1. Click on **prod** environment
2. Click **"..."** → **Approvals and checks**
3. Add **Approvals**:
   - Approvers: Add the single designated user
   - Instructions: "Approve deployment to production environment"
   - Timeout: 7 days

4. Add **Branch Control** check:
   - Allowed branches: `refs/heads/pull-request-permission`
   - Required: Yes

#### For `testing` environment:

1. Approvers: Same designated user
2. Branch control: `refs/heads/pull-request-permission`

#### For `dev` environment:

1. No approval required (automatic deployment)
2. Branch control: `refs/heads/pull-request-permission`

---

## Part 7: Complete Workflow Summary

### Pre-Setup Checklist

Before starting, ensure the following are completed in order:

```
1. [ ] Import latina_app repository from GitHub
2. [ ] Import latina_app_template repository from GitHub
3. [ ] Create aks-service-connection (Kubernetes)
4. [ ] Create acr-shared-service-connection (Azure Container Registry)
5. [ ] Create pipeline from azure/pipelines/frontend-service.yml
6. [ ] Set uniqueSuffix variable from ARM deployment
7. [ ] Verify pipeline references latina_app_template repository
8. [ ] Set pull-request-permission as default branch
9. [ ] Configure branch policies
10. [ ] Add single user as required reviewer
11. [ ] Configure environment approvals
```

### PR Workflow

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                           Frontend PR Workflow                                          │
├─────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                         │
│   1. Developer creates feature branch:                                                  │
│      git checkout -b feature/frontend-update                                            │
│                                                                                         │
│   2. Developer pushes changes:                                                          │
│      git push origin feature/frontend-update                                            │
│                                                                                         │
│   3. Developer creates PR targeting pull-request-permission                            │
│                                                                                         │
│   4. Build validation runs automatically                                                │
│      ├── Builds frontend-service                                                        │
│      ├── Runs unit tests                                                                │
│      └── Status reported to PR                                                          │
│                                                                                         │
│   5. Single reviewer approves (can be self-approval if enabled)                         │
│                                                                                         │
│   6. PR is completed (any merge strategy)                                               │
│      └── pull-request-permission branch updated → Pipeline triggered                   │
│                                                                                         │
│   7. Pipeline deploys to dev automatically                                               │
│                                                                                         │
│   8. Manual approval for test environment (same user)                                   │
│                                                                                         │
│   9. Manual approval for prod environment (same user)                                   │
│                                                                                         │
└─────────────────────────────────────────────────────────────────────────────────────────┘
```

---

## Summary

### Key Takeaways

| Concept | Implementation |
|---------|----------------|
| Repository Import | Import latina_app and latina_app_template from GitHub |
| Service Connections | AKS and ACR connections required |
| Pipeline Setup | Create from YAML, set uniqueSuffix variable |
| Branch Protection | Policies on `pull-request-permission` branch |
| Code Review | 1 reviewer, self-approval allowed |
| Merge Strategies | All strategies enabled (squash, merge, rebase) |
| Build Validation | Automatic pipeline run on PR creation |
| Environment Security | Manual approval gates for test/prod |

### Security Checklist

- [ ] latina_app repository imported from GitHub
- [ ] latina_app_template repository imported from GitHub
- [ ] aks-service-connection created and verified
- [ ] acr-shared-service-connection created and verified
- [ ] Pipeline created from YAML file
- [ ] uniqueSuffix variable populated from ARM deployment
- [ ] Branch policies configured on `pull-request-permission` branch
- [ ] Minimum reviewers set to 1
- [ ] Self-approval option enabled
- [ ] All merge strategies enabled
- [ ] Single user added as required reviewer
- [ ] Build validation enabled
- [ ] Environment approvals configured (dev, test, prod)

---

## Additional Resources

### Azure DevOps Documentation
- [Import repositories](https://learn.microsoft.com/en-us/azure/devops/repos/git/import-git-repository)
- [Service connections](https://learn.microsoft.com/en-us/azure/devops/pipelines/library/service-endpoints)
- [Branch Policies](https://learn.microsoft.com/en-us/azure/devops/repos/git/branch-policies)
- [Environment Approvals](https://learn.microsoft.com/en-us/azure/devops/pipelines/process/environments)

### Project Files Reference

| File | Purpose |
|------|---------|
| `azure/pipelines/frontend-service.yml` | Main pipeline with approval gates |
| `azure/arm/main.json` | ARM template with infrastructure outputs |
| `azure/templates/deploy.yaml` | Deployment template (in latina_app_template) |

---

## Q&A

### Common Questions

**Q: Why do I need to import both repositories?**
A: The pipeline in latina_app references templates from latina_app_template. Both must be imported for the pipeline to work correctly.

**Q: Where do I get the uniqueSuffix value?**
A: Run the ARM deployment first, then retrieve the uniqueSuffix from deployment outputs. This is used to construct the ACR login server URL.

**Q: Why set pull-request-permission as the main branch?**
A: This allows you to test PR policies without affecting your actual main branch. It creates an isolated workflow for learning.

**Q: Can a user approve their own PR?**
A: Yes, when "Allow requesters to approve their own changes" is enabled in branch policies. This is useful for solo development or testing workflows.

**Q: What if I need different reviewers for different paths?**
A: Add multiple "Require reviewers" policies with different path filters. Each can have different reviewers.

---

*This lesson is designed for the Latina App frontend service deployment pipeline.*
