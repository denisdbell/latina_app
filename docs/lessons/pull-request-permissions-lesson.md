# Pull Request Permissions & Policies Lesson
## Focus: Frontend Service Deployment for Latina App
**Duration: 2 Hours**

---

## Overview

This lesson covers how to implement pull request (PR) permissions and policies in Azure DevOps to secure your frontend service deployment pipeline. You'll learn to enforce code quality, control deployments across environments, and establish proper approval workflows.

---

## Learning Objectives

By the end of this lesson, you will be able to:
1. Configure branch protection policies in Azure DevOps
2. Set up required reviewers and code review workflows
3. Implement build validation policies for frontend changes
4. Configure environment-specific approval gates (dev → test → prod)
5. Establish security roles and permissions for PR workflows

---

## Prerequisites

- Azure DevOps project with `latina_app` repository
- Frontend service pipeline configured (see `azure/pipelines/frontend-service.yml`)
- Basic understanding of Git workflows

---

## Part 1: Understanding the Deployment Flow (15 minutes)

### Current Pipeline Architecture

The frontend service pipeline (`azure/pipelines/frontend-service.yml`) follows this flow:

```
┌─────────┐     ┌────────────┐     ┌───────────────┐     ┌────────────┐     ┌───────────────┐     ┌────────────┐
│  Build  │────▶│ Deploy Dev │────▶│ Approve Test  │────▶│ Deploy Test│────▶│ Approve Prod  │────▶│ Deploy Prod│
│         │     │            │     │ (Manual Gate) │     │            │     │ (Manual Gate) │     │            │
└─────────┘     └────────────┘     └───────────────┘     └────────────┘     └───────────────┘     └────────────┘
```

### Trigger Configuration

The pipeline is triggered by changes to the `main` branch with path filters:

```yaml
trigger:
  branches:
    include:
      - main
  paths:
    include:
      - frontend-service/**
      - pom.xml
      - azure/pipelines/frontend-service.yml
```

**Key Point:** Any PR merging into `main` that touches frontend files will trigger the full deployment pipeline.

---

## Part 2: Branch Protection Policies (25 minutes)

### 2.1 Configuring Branch Policies in Azure DevOps

Navigate to: **Repos → Branches → main → Branch Policies**

### Required Policies to Enable

| Policy | Purpose | Recommended Settings |
|--------|---------|---------------------|
| **Require minimum reviewers** | Ensures code review | 2 reviewers minimum |
| **Require reviewers** | Specific team approval | Frontend team lead required |
| **Check for linked work items** | Traceability | Required |
| **Build validation** | Automated testing | Frontend build pipeline |
| **Require merge strategy** | Commit history control | Squash merge |

### 2.2 Exercise: Configure Branch Policies

**Step 1:** Navigate to your Azure DevOps project

```
Project Settings → Repositories → latina_app → Branches → main → Policies
```

**Step 2:** Enable the following policies:

#### A. Minimum Reviewers

1. Toggle **"Require a minimum number of reviewers"** to ON
2. Set minimum reviewers to **2**
3. Check **"Allow requesters to approve their own changes"** to OFF
4. Check **"Allow completion even if some reviewers vote to wait or reject"** to OFF

#### B. Required Reviewers for Frontend

1. Toggle **"Require reviewers"** to ON
2. Add path filter: `/frontend-service/**`
3. Add required reviewer group: **Frontend Team** or specific user
4. Set to **Required** (not Optional)

#### C. Build Validation

1. Toggle **"Build validation"** to ON
2. Select build pipeline: `frontend-service` pipeline
3. Set trigger: **Run on every update**
4. Enable **"Require successful build completion"**

**Step 3:** Save policies

---

## Part 3: Repository Permissions (20 minutes)

### 3.1 Understanding Permission Levels

| Role | Read | Contribute | Create Branch | Force Push | Bypass Policies |
|------|------|-----------|---------------|------------|-----------------|
| Reader | ✅ | ❌ | ❌ | ❌ | ❌ |
| Contributor | ✅ | ✅ | ✅ | ❌ | ❌ |
| Administrator | ✅ | ✅ | ✅ | ✅ | ✅ |

### 3.2 Exercise: Set Up Permission Structure

Navigate to: **Project Settings → Repositories → latina_app → Security**

#### Recommended Permission Structure for Frontend:

```
latina_app Repository
├── Readers (Project Team)
│   └── Read: Allow
│
├── Contributors (Developers)
│   ├── Read: Allow
│   ├── Contribute: Allow
│   ├── Create Branch: Allow
│   └── Force Push: Deny
│
├── Frontend Team
│   ├── All Contributor permissions
│   └── Manage permissions for /frontend-service/**
│
├── Build Service
│   ├── Read: Allow
│   ├── Contribute: Allow
│   └── Create Branch: Allow (for tags)
│
└── Project Administrators
    └── All permissions: Allow
```

#### Configure Frontend Team Permissions:

1. Create a **Frontend Team** group
2. Grant **Contribute** permission at repository level
3. Add members who will work on frontend service

---

## Part 4: Environment Approvals and Checks (30 minutes)

### 4.1 Understanding Environment-Based Gates

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

### 4.2 Exercise: Configure Environment Approvals

**Step 1:** Navigate to Environments

```
Pipelines → Environments
```

You should see (or create):
- `dev`
- `testing`
- `prod`

**Step 2:** Configure approvals for each environment

#### For `prod` environment:

1. Click on **prod** environment
2. Click **"..."** → **Approvals and checks**
3. Add **Approvals**:
   - Approvers: Add production team lead or operations team
   - Instructions: "Approve deployment to production environment"
   - Timeout: 7 days

4. Add **Branch Control** check:
   - Allowed branches: `refs/heads/main`
   - Required: Yes

5. Add **Azure Kubernetes Service** check (if using AKS):
   - Service connection: `aks-service-connection`
   - Namespace: `prod`
   - Check: Verify deployment status

#### For `testing` environment:

1. Approvers: QA team or frontend team lead
2. Branch control: `refs/heads/main` or `refs/heads/release/*`

#### For `dev` environment:

1. No approval required (automatic deployment)
2. Branch control: Any branch (for development testing)

### 4.3 Advanced: Service Connection Security

Navigate to: **Project Settings → Service connections**

#### Configure Service Connection Approvals:

| Service Connection | Environment | Approval Required |
|-------------------|-------------|-------------------|
| `dev-acr-service-connection` | dev | No |
| `test-acr-service-connection` | testing | Yes - QA Team |
| `prod-acr-service-connection` | prod | Yes - Ops Team |
| `aks-service-connection` | all | Yes - Approvers per namespace |

---

## Part 5: PR Workflow Best Practices (20 minutes)

### 5.1 Recommended PR Workflow for Frontend

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
│   3. Developer creates PR targeting main                                                │
│                                                                                         │
│   4. Build validation runs automatically                                                │
│      ├── Builds frontend-service                                                        │
│      ├── Runs unit tests                                                                │
│      └── Status reported to PR                                                          │
│                                                                                         │
│   5. Reviewers assigned automatically (based on /frontend-service/** path)              │
│                                                                                         │
│   6. Required reviewers approve                                                         │
│                                                                                         │
│   7. PR is completed (squash merge)                                                      │
│      └── Main branch updated → Pipeline triggered                                       │
│                                                                                         │
│   8. Pipeline deploys to dev automatically                                               │
│                                                                                         │
│   9. Manual approval for test environment                                               │
│                                                                                         │
│   10. Manual approval for prod environment                                               │
│                                                                                         │
└─────────────────────────────────────────────────────────────────────────────────────────┘
```

### 5.2 PR Template for Frontend Changes

Create file: `.azuredevops/pull_request_templates/frontend.md`

```markdown
## Frontend Service Change

### Description
<!-- Describe your changes to the frontend service -->

### Type of Change
- [ ] Bug fix (non-breaking change)
- [ ] New feature (non-breaking change)
- [ ] Breaking change
- [ ] UI/UX update
- [ ] Performance improvement

### Frontend Files Changed
- [ ] `frontend-service/src/main/resources/static/index.html`
- [ ] `frontend-service/src/main/resources/static/app.js`
- [ ] `frontend-service/src/main/resources/static/style.css`
- [ ] `frontend-service/src/main/java/`
- [ ] `frontend-service/pom.xml`
- [ ] `frontend-service/k8s/`

### Testing
- [ ] Unit tests pass locally
- [ ] Manual testing completed
- [ ] Browser compatibility tested (Chrome, Firefox, Safari)

### Screenshots (if UI changes)
<!-- Add screenshots here -->

### Checklist
- [ ] Code follows project style guidelines
- [ ] Self-review completed
- [ ] Comments added for complex logic
- [ ] Documentation updated if needed
- [ ] No new warnings introduced
```

### 5.3 Exercise: Create PR Template

1. Create directory: `.azuredevops/pull_request_templates/`
2. Create template file for frontend changes
3. Test by creating a new PR

---

## Part 6: Security Scenarios (10 minutes)

### 6.1 Scenario: Preventing Unauthorized Deployments

**Problem:** Developer tries to deploy unapproved frontend changes to production.

**Solution:**
```yaml
# Environment checks prevent this
- stage: DeployProd
  dependsOn: ApproveProd
  condition: succeeded()  # Only runs after approval
  jobs:
  - deployment: DeployProd
    environment: prod  # Requires approval from designated approvers
```

### 6.2 Scenario: Bypass Protection for Hotfixes

**Problem:** Critical production issue needs immediate fix.

**Solution:**
1. Create **Hotfix Branch Policy**:
   - Branch filter: `hotfix/*`
   - Reduced reviewers: 1 (instead of 2)
   - Still require build validation

2. Or: Grant **Bypass policies** permission to:
   - Project Administrators
   - Release Managers

### 6.3 Scenario: External Contributor

**Problem:** External developer wants to contribute to frontend.

**Solution:**
1. Fork workflow:
   - External developer forks repository
   - Creates PR from fork
   - Project member reviews and approves
   - Project member completes PR

2. Configure **Fork policies**:
   - Navigate to: Project Settings → Repositories → latina_app → Policies
   - Enable **"Contribution from forks"** with restrictions
   - Require **"Secret variable access"** to OFF

---

## Part 7: Hands-On Lab (20 minutes)

### Lab Exercise: Set Up Complete PR Policy Chain

#### Task 1: Create Branch Policies (5 minutes)

1. Navigate to your Azure DevOps project
2. Go to **Repos → Branches**
3. Click **"..."** on `main` branch → **Branch policies**
4. Enable:
   - Minimum reviewers: 2
   - Required reviewers for `/frontend-service/**`: Frontend Team
   - Build validation: frontend-service pipeline
   - Work item linking: Required

#### Task 2: Configure Environment Approvals (10 minutes)

1. Go to **Pipelines → Environments**
2. Create `dev`, `testing`, `prod` environments if not exists
3. For `prod`:
   - Add approvers: Your team lead
   - Add branch control: `refs/heads/main`
4. For `testing`:
   - Add approvers: QA team
   - Add branch control: `refs/heads/main` or `refs/heads/release/*`

#### Task 3: Test the Workflow (5 minutes)

1. Create a test branch:
   ```bash
   git checkout -b test/pr-policy-test
   echo "// Test comment" >> frontend-service/src/main/resources/static/index.html
   git add .
   git commit -m "Test PR policy workflow"
   git push origin test/pr-policy-test
   ```

2. Create PR targeting `main`
3. Observe:
   - Build validation triggered
   - Reviewers automatically assigned
   - PR cannot be completed without approvals

---

## Summary

### Key Takeaways

| Concept | Implementation |
|---------|----------------|
| Branch Protection | Policies on `main` branch |
| Code Review | Minimum 2 reviewers, required for frontend paths |
| Build Validation | Automatic pipeline run on PR creation |
| Environment Security | Manual approval gates for test/prod |
| Permission Levels | Role-based access to repository and environments |

### Security Checklist

- [ ] Branch policies configured on `main` branch
- [ ] Minimum reviewers requirement set to 2+
- [ ] Required reviewers assigned for `/frontend-service/**`
- [ ] Build validation enabled
- [ ] Environment approvals configured (test, prod)
- [ ] Service connections secured with approvals
- [ ] PR templates created for frontend changes
- [ ] Work item linking required
- [ ] Merge strategy enforced (squash)

---

## Additional Resources

### Azure DevOps Documentation
- [Branch Policies](https://learn.microsoft.com/en-us/azure/devops/repos/git/branch-policies)
- [Environment Approvals](https://learn.microsoft.com/en-us/azure/devops/pipelines/process/environments)
- [Repository Security](https://learn.microsoft.com/en-us/azure/devops/repos/git/security)

### Project Files Reference

| File | Purpose |
|------|---------|
| `azure/pipelines/frontend-service.yml` | Main pipeline with approval gates |
| `azure/templates/deploy.yaml` | Deployment template |
| `frontend-service/k8s/deployment.yaml` | Kubernetes deployment manifest |

---

## Q&A

### Common Questions

**Q: Can I skip approvals for development environment?**
A: Yes, the current pipeline deploys to dev automatically. Only test and prod require manual approval.

**Q: How do I add temporary approvers for a specific release?**
A: Navigate to the environment and add temporary approvers in the "Approvals and checks" section.

**Q: What happens if a build fails during validation?**
A: The PR cannot be completed. The build must pass before merge is allowed.

**Q: Can I configure different policies for different branches?**
A: Yes, you can configure separate policies for `main`, `release/*`, `hotfix/*`, etc.

---

*This lesson is designed for the Latina App frontend service deployment pipeline.*