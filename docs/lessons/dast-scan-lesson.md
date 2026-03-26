# DAST Security Scan Lesson
## Focus: Frontend Service Deployment for Latina App
**Duration: 1 Hour**

---

## Overview

This lesson covers how to implement Dynamic Application Security Testing (DAST) in your Azure DevOps pipeline for the frontend service. DAST scans running applications to identify security vulnerabilities such as cross-site scripting (XSS), SQL injection, broken authentication, and other OWASP Top 10 vulnerabilities.

---

## Learning Objectives

By the end of this lesson, you will be able to:
1. Understand what DAST is and why it's important
2. Configure DAST scan variables in your pipeline
3. Add a DAST scan stage to your deployment pipeline
4. Review and interpret DAST scan results
5. Integrate DAST findings into your security workflow

---

## Prerequisites

- Completed the Pull Request Permissions lesson
- Successfully deployed frontend service to dev environment
- Azure DevOps pipeline running successfully
- Access to the deployed frontend service URL

---

## Part 1: Understanding DAST (15 minutes)

### 1.1 What is DAST?

Dynamic Application Security Testing (DAST) is a security testing methodology that analyzes running applications by simulating real-world attacks. Unlike SAST (Static Application Security Testing) which analyzes source code, DAST tests the application from the outside-in.

### 1.2 DAST vs SAST

| Aspect | DAST | SAST |
|--------|------|------|
| When to run | After deployment | During build |
| What it tests | Running application | Source code |
| Findings | Runtime vulnerabilities | Code vulnerabilities |
| False positives | Lower | Higher |
| Requires | Deployed app | Source code only |

### 1.3 OWASP ZAP

OWASP ZAP (Zed Attack Proxy) is a free, open-source DAST tool that helps find security vulnerabilities in web applications. It provides:

- **Baseline Scan**: Quick scan for low-hanging vulnerabilities
- **Full Scan**: Comprehensive scan including active attacks
- **API Scan**: Tests REST/API endpoints
- **Report Generation**: HTML, JSON, and other formats

### 1.4 Where DAST Fits in the Pipeline

```
Build → Deploy to Dev → DAST Scan → Manual Approval → Deploy to Test → Deploy to Prod
```

DAST runs after deployment to dev, catching vulnerabilities before promoting to higher environments.

---

## Part 2: Pipeline Configuration (20 minutes)

### 2.1 Understanding the DAST Stage

The DAST stage was added to `azure/pipelines/frontend-service.yml`:

```yaml
- stage: DASTScan
  displayName: 'DAST Security Scan'
  dependsOn: DeployDev
  condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/pull-request-permission'))
  jobs:
  - job: DASTScanJob
    steps:
    - script: |
        echo "Running OWASP ZAP DAST scan against deployed application"
        # Install OWASP ZAP
        sudo apt-get update
        sudo apt-get install -y zaproxy
      displayName: 'Install OWASP ZAP'
    - script: |
        # Run ZAP baseline scan against the frontend service
        zap-baseline.py -t "$(dastTargetUrl)" -r baseline-report.html || true
      displayName: 'Run DAST Baseline Scan'
    - task: PublishHtmlReport@1
      displayName: 'Publish DAST Report'
      inputs:
        reportDir: 'baseline-report.html'
        reportName: 'DAST Security Report'
        reportTitle: 'OWASP ZAP DAST Scan Results'
        failOnWarning: false
```

### 2.2 Key Configuration Variables

| Variable | Purpose | Example Value |
|----------|---------|---------------|
| `dastTargetUrl` | Target URL for DAST scan | `https://frontend-dev.azurewebsites.net` |

### 2.3 DAST Stage Dependencies

```
DeployDev → DASTScan → ApproveTesting → DeployTest → ApproveProd → DeployProd
```

The DAST scan:
- Runs after successful dev deployment
- Must complete before manual approval for test promotion
- Does not block pipeline on warnings (configurable)

---

## Part 3: Configure DAST Variables (10 minutes)

### 3.1 Identify Your Frontend Service URL

After deploying to dev environment, identify the frontend service URL:

**Option A: Azure App Service**
```
https://<app-name>.azurewebsites.net
```

**Option B: AKS with Ingress**
```
kubectl get ingress -n dev
```

**Option C: Kubernetes Service**
```
kubectl get service frontend-service -n dev
```

### 3.2 Update Pipeline Variables

Navigate to: **Pipelines → frontend-service → Variables**

Add or update the following variable:

| Variable Name | Value | Notes |
|---------------|-------|-------|
| `dastTargetUrl` | `https://your-frontend-dev-url` | Your deployed frontend URL |

### 3.3 Alternative: Set URL in YAML

You can also set the URL directly in the pipeline file:

```yaml
variables:
  # ... existing variables ...

  # DAST Scan Configuration
  dastTargetUrl: 'https://frontend-service-dev.yourdomain.com'
```

---

## Part 4: Running and Reviewing DAST Scans (15 minutes)

### 4.1 Trigger a Pipeline Run

Push changes to the `pull-request-permission` branch:

```bash
git checkout pull-request-permission
git add .
git commit -m "Trigger DAST scan"
git push origin pull-request-permission
```

### 4.2 Monitor Pipeline Execution

Navigate to: **Pipelines → frontend-service → Latest run**

The pipeline will execute:
1. Build stage
2. Deploy to Dev stage
3. **DAST Security Scan stage** ← New
4. Manual Approval stage
5. Deploy to Test stage
6. Deploy to Prod stage

### 4.3 Review DAST Results

After the DAST scan completes:

**Step 1:** Click on the **DASTScan** stage

**Step 2:** Click on the **Publish DAST Report** task

**Step 3:** Download or view the HTML report

**Step 4:** Review findings categorized by severity:
- **High**: Critical vulnerabilities requiring immediate attention
- **Medium**: Significant issues to address before production
- **Low**: Minor issues or informational findings
- **Informational**: Best practice recommendations

### 4.4 Common DAST Findings

| Finding | Severity | Mitigation |
|---------|----------|-------------|
| Missing X-Frame-Options | Medium | Add security headers |
| Cookie without HttpOnly | Medium | Set HttpOnly flag |
| Cookie without Secure | High | Use Secure flag |
| Cross-Domain JavaScript | Low | Review external scripts |
| Information Disclosure | Informational | Remove sensitive info |

---

## Part 5: Advanced DAST Configuration (Optional)

### 5.1 Full Scan vs Baseline Scan

For more thorough testing, use a full scan:

```yaml
- script: |
    # Full scan includes active attacks (use with caution)
    zap-full-scan.py -t "$(dastTargetUrl)" -r full-report.html
  displayName: 'Run DAST Full Scan'
  env:
    ZAP_AUTH_HEADER: $(zapAuthHeader)  # Optional: for authenticated scans
```

### 5.2 Authenticated Scans

For applications requiring authentication:

```yaml
- script: |
    # Configure authentication
    zap-baseline.py \
      -t "$(dastTargetUrl)" \
      -r baseline-report.html \
      -z "-config authentication.type=httpHeader" \
      -z "-config authentication.httpHeader.authHeader=$(authToken)"
  displayName: 'Run Authenticated DAST Scan'
```

### 5.3 Custom ZAP Configuration

Create a `zap.conf` file in your repository:

```properties
# zap.conf
connection.timeoutInSecs=60
scanner.maxChildren=10
scanner.injectable=3
```

Reference in pipeline:

```yaml
- script: |
    zap-baseline.py \
      -t "$(dastTargetUrl)" \
      -r baseline-report.html \
      -c $(Build.SourcesDirectory)/zap.conf
  displayName: 'Run DAST with Custom Config'
```

### 5.4 Scan Context Files

Define scan scope and exclude paths:

```yaml
# zap-context.json
{
  "name": "frontend-scan",
  "includeRegexes": ["https://frontend-dev\\.azurewebsites\\.net/.*"],
  "excludeRegexes": ["https://frontend-dev\\.azurewebsites\\.net/logout.*"]
}
```

---

## Part 6: Integrating with Security Workflow

### 6.1 Pipeline Gate Configuration

Configure DAST results as a quality gate:

```yaml
- task: PublishHtmlReport@1
  displayName: 'Publish DAST Report'
  inputs:
    reportDir: 'baseline-report.html'
    reportName: 'DAST Security Report'
    failOnWarning: true  # Fail pipeline on warnings
```

### 6.2 Create Security Review Task

After DAST scan, create a work item for review:

```yaml
- task: CreateWorkItem@1
  displayName: 'Create Security Review Task'
  condition: failed()
  inputs:
    workItemType: 'Task'
    title: 'Review DAST Findings'
    description: 'Review DAST report and address findings'
```

### 6.3 Notify Security Team

Add email notification for findings:

```yaml
- task: SendEmail@1
  displayName: 'Notify Security Team'
  condition: failed()
  inputs:
    to: 'security-team@company.com'
    subject: 'DAST Scan Findings - Frontend Service'
    body: 'DAST scan completed with findings. Review the pipeline run.'
```

---

## Summary

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
│   3. DAST Security Scan ← NEW                                                          │
│      ├── Installs OWASP ZAP                                                             │
│      ├── Scans running application for vulnerabilities                                  │
│      └── Publishes security report                                                       │
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
| DAST Tool | OWASP ZAP (open source) |
| Scan Type | Baseline scan (quick, non-intrusive) |
| Timing | After dev deployment, before test promotion |
| Report | HTML report published to Azure DevOps |
| Gate | Can block on warnings (configurable) |

### Security Checklist

- [ ] `dastTargetUrl` variable configured with correct URL
- [ ] DAST stage added after DeployDev
- [ ] OWASP ZAP installed in pipeline agent
- [ ] Report publishing configured
- [ ] Manual approval references DAST results
- [ ] Security team notified of findings workflow established

---

## Additional Resources

### OWASP ZAP Documentation
- [OWASP ZAP Getting Started](https://www.zaproxy.org/getting-started/)
- [ZAP Baseline Scan](https://www.zaproxy.org/docs/docker/baseline-scan/)
- [ZAP Full Scan](https://www.zaproxy.org/docs/docker/full-scan/)

### Azure DevOps Integration
- [Publish HTML Report Task](https://marketplace.visualstudio.com/items?itemName=JakubIshamowicz.PublishHtmlReport)
- [Customize Pipeline Conditions](https://learn.microsoft.com/en-us/azure/devops/pipelines/process/conditions)

### Security Best Practices
- [OWASP Top 10](https://owasp.org/Top10/)
- [Web Security Guidelines](https://cheatsheetseries.owasp.org/)

---

## Q&A

### Common Questions

**Q: Why run DAST after dev deployment instead of during build?**
A: DAST requires a running application to test. Static analysis (SAST) runs during build, but DAST must test the live endpoints.

**Q: What's the difference between baseline and full scan?**
A: Baseline scan is non-intrusive and quick (minutes). Full scan includes active attacks and may modify data (use cautiously).

**Q: Can DAST scan authenticated pages?**
A: Yes, ZAP supports authentication. Configure auth headers or use recorded login scripts.

**Q: Should the pipeline fail on DAST findings?**
A: It depends on your maturity level. Initially, set `failOnWarning: false` to gather findings. Later, enforce stricter gates.

**Q: How often should I run DAST scans?**
A: Run DAST on every deployment to dev, and periodically against running environments for regression detection.

---

## Troubleshooting

### Common Issues

| Issue | Solution |
|-------|----------|
| ZAP fails to connect | Verify target URL is accessible from pipeline agent |
| Scan timeout | Increase timeout or use baseline scan instead of full |
| Too many false positives | Configure exclude paths in context file |
| Pipeline fails on warnings | Set `failOnWarning: false` during initial tuning |
| SSL certificate errors | Add `-z "-config network.connection.sslInsecure=true"` |

### Useful ZAP Flags

```bash
# Ignore SSL errors
zap-baseline.py -t "https://url" -r report.html -z "-config network.connection.sslInsecure=true"

# Set timeout
zap-baseline.py -t "https://url" -r report.html -z "-config connection.timeoutInSecs=120"

# Exclude paths
zap-baseline.py -t "https://url" -r report.html -e "/logout,/health"
```

---

*This lesson extends the Latina App frontend service deployment pipeline with DAST security scanning capabilities.*