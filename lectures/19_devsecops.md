# DevSecOps: Integrating Security into DevOps

## What is DevSecOps?

**DevSecOps** (Development, Security, and Operations) is the practice of integrating security practices within the DevOps process. It emphasizes that security is everyone's responsibility and should be implemented at every stage of the software development lifecycle.

### Traditional vs DevSecOps:

**Traditional**:
```
Develop → Test → Deploy → Security Audit → Production
(Security at the end)
```

**DevSecOps**:
```
Plan (Security) → Develop (Security) → Build (Security) → 
Test (Security) → Deploy (Security) → Monitor (Security)
(Security throughout)
```

## Why DevSecOps?

1. **Shift Left**: Find security issues early when they're cheaper to fix
2. **Faster Remediation**: Security issues caught in development, not production
3. **Compliance**: Meet security and regulatory requirements
4. **Cost Reduction**: Early detection reduces remediation costs
5. **Better Collaboration**: Shared responsibility between Dev, Sec, and Ops
6. **Reduced Risk**: Proactive security vs reactive

## DevSecOps Principles

### 1. Security as Code
Treat security policies and configurations as code:
```yaml
- Version controlled
- Automated
- Tested
- Reviewed
```

### 2. Shift Left Security
Integrate security early in the development process

### 3. Automation
Automate security testing and compliance checks

### 4. Continuous Monitoring
Monitor security in real-time

### 5. Shared Responsibility
Everyone is responsible for security

### 6. Feedback Loop
Quick feedback on security issues

## DevSecOps Pipeline Stages

### 1. Pre-Commit Stage

**IDE Security Plugins**:
```
- SonarLint
- Snyk
- ESLint security plugins
```

**Pre-commit Hooks**:
```bash
# .git/hooks/pre-commit
#!/bin/bash

# Secret scanning
git diff --cached --name-only | xargs grep -E "(password|api_key|secret)" && exit 1

# Run security linter
npm run security-lint

# Check for hardcoded secrets
detect-secrets scan --baseline .secrets.baseline
```

### 2. Source Code Management (SCM)

**Branch Protection**:
```yaml
- Require pull request reviews
- Require status checks
- Enforce code review from security team
- Block force pushes
```

**Secret Scanning**:
```yaml
tools:
  - GitGuardian
  - Trufflehog
  - GitHub Secret Scanning
  - GitLab Secret Detection
```

### 3. Build Stage

**Dependency Scanning**:
```yaml
# GitHub Actions
- name: Run dependency check
  run: |
    npm audit --audit-level=moderate
    npm audit fix

# Check for known vulnerabilities
- name: Snyk Scan
  uses: snyk/actions/node@master
  env:
    SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
```

**SAST (Static Application Security Testing)**:
```yaml
tools:
  - SonarQube
  - Checkmarx
  - Fortify
  - Semgrep
  - CodeQL
```

**Example - SonarQube**:
```yaml
# .github/workflows/sonarqube.yml
- name: SonarQube Scan
  uses: sonarsource/sonarqube-scan-action@master
  env:
    SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
    SONAR_HOST_URL: ${{ secrets.SONAR_HOST_URL }}
```

### 4. Container Security

**Container Image Scanning**:
```yaml
# Scan Docker image
- name: Trivy Scan
  uses: aquasecurity/trivy-action@master
  with:
    image-ref: 'myapp:${{ github.sha }}'
    format: 'sarif'
    output: 'trivy-results.sarif'
    severity: 'CRITICAL,HIGH'

# Scan with Snyk
- name: Snyk Container Scan
  uses: snyk/actions/docker@master
  with:
    image: myapp:latest
  env:
    SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
```

**Best Practices**:
```dockerfile
# Use official base images
FROM node:18-alpine

# Run as non-root user
USER node

# Copy only necessary files
COPY --chown=node:node package*.json ./
COPY --chown=node:node . .

# Don't expose unnecessary ports
EXPOSE 3000

# Use specific tags, not latest
FROM nginx:1.21-alpine
```

### 5. Infrastructure as Code (IaC) Security

**Scanning Tools**:
```yaml
- Checkov
- tfsec
- Terrascan
- KICS
- CloudFormation Guard
```

**Example - Terraform Scanning**:
```bash
# Install tfsec
brew install tfsec

# Scan Terraform files
tfsec .

# In CI/CD pipeline
- name: tfsec
  uses: aquasecurity/tfsec-action@v1.0.0
  with:
    soft_fail: false
```

**Terraform Security Example**:
```hcl
# Bad: Unrestricted security group
resource "aws_security_group" "bad" {
  ingress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

# Good: Restricted access
resource "aws_security_group" "good" {
  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["10.0.0.0/8"]
  }
}
```

### 6. Kubernetes Security

**Pod Security Standards**:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    fsGroup: 2000
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: app
    image: myapp:1.0
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop:
        - ALL
        add:
        - NET_BIND_SERVICE
    resources:
      limits:
        memory: "128Mi"
        cpu: "500m"
```

**Kubernetes Security Scanning**:
```bash
# Kubesec - security risk analysis
kubesec scan deployment.yaml

# Kube-bench - CIS Kubernetes Benchmark
kube-bench run --targets node,master

# Falco - runtime security
helm install falco falcosecurity/falco
```

**Network Policies**:
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-ingress
spec:
  podSelector: {}
  policyTypes:
  - Ingress
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 8080
```

### 7. Secrets Management

**Tools**:
```yaml
- HashiCorp Vault
- AWS Secrets Manager
- Azure Key Vault
- Google Secret Manager
- Sealed Secrets (Kubernetes)
```

**Kubernetes Secrets Best Practices**:
```yaml
# External Secrets Operator
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: app-secrets
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: vault-backend
    kind: SecretStore
  target:
    name: app-secrets
    creationPolicy: Owner
  data:
  - secretKey: password
    remoteRef:
      key: secret/data/app
      property: password
```

**Vault Integration**:
```bash
# Install Vault
helm install vault hashicorp/vault

# Store secret
vault kv put secret/myapp password="super-secret"

# Access in pod via sidecar
annotations:
  vault.hashicorp.com/agent-inject: "true"
  vault.hashicorp.com/agent-inject-secret-password: "secret/myapp"
  vault.hashicorp.com/role: "myapp-role"
```

### 8. DAST (Dynamic Application Security Testing)

**Tools**:
```yaml
- OWASP ZAP
- Burp Suite
- Acunetix
- Netsparker
```

**OWASP ZAP in CI/CD**:
```yaml
# GitHub Actions
- name: OWASP ZAP Scan
  uses: zaproxy/action-full-scan@v0.4.0
  with:
    target: 'https://staging.myapp.com'
    rules_file_name: '.zap/rules.tsv'
    cmd_options: '-a'
```

### 9. Runtime Security

**Falco - Runtime Threat Detection**:
```yaml
# Install Falco
helm repo add falcosecurity https://falcosecurity.github.io/charts
helm install falco falcosecurity/falco

# Custom rule
- rule: Suspicious Network Activity
  desc: Detect suspicious network connections
  condition: >
    spawned_process and
    proc.name = "nc" and
    (fd.sip = "0.0.0.0" or fd.sip = "::")
  output: >
    Suspicious network activity detected
    (user=%user.name command=%proc.cmdline)
  priority: WARNING
```

## Security Scanning Tools Comparison

| Tool | Type | Focus Area |
|------|------|------------|
| SonarQube | SAST | Code quality & security |
| Snyk | SCA | Dependencies & containers |
| Trivy | Container | Container images |
| Checkmarx | SAST | Source code |
| OWASP ZAP | DAST | Running applications |
| Vault | Secrets | Secret management |
| Falco | Runtime | Kubernetes runtime |
| tfsec | IaC | Terraform |
| Checkov | IaC | Multi-cloud IaC |

## Complete DevSecOps Pipeline

```yaml
name: Complete DevSecOps Pipeline

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  secret-scan:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    - name: Trufflehog Scan
      uses: trufflesecurity/trufflehog@main
      with:
        path: ./
        base: ${{ github.event.repository.default_branch }}
        head: HEAD

  sast:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    - name: Run Semgrep
      uses: returntocorp/semgrep-action@v1
    - name: SonarQube Scan
      uses: sonarsource/sonarqube-scan-action@master
      env:
        SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}

  dependency-check:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    - name: Run npm audit
      run: npm audit --audit-level=moderate
    - name: Snyk Dependency Scan
      uses: snyk/actions/node@master
      env:
        SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}

  build-and-scan:
    needs: [secret-scan, sast, dependency-check]
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    
    - name: Build Docker image
      run: docker build -t myapp:${{ github.sha }} .
    
    - name: Trivy Container Scan
      uses: aquasecurity/trivy-action@master
      with:
        image-ref: myapp:${{ github.sha }}
        format: 'sarif'
        output: 'trivy-results.sarif'
        severity: 'CRITICAL,HIGH'
    
    - name: Upload Trivy results
      uses: github/codeql-action/upload-sarif@v2
      with:
        sarif_file: 'trivy-results.sarif'
    
    - name: Push image
      run: |
        docker tag myapp:${{ github.sha }} ghcr.io/${{ github.repository }}/myapp:${{ github.sha }}
        docker push ghcr.io/${{ github.repository }}/myapp:${{ github.sha }}

  iac-scan:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    - name: tfsec
      uses: aquasecurity/tfsec-action@v1.0.0
    - name: Checkov
      uses: bridgecrewio/checkov-action@master
      with:
        directory: .
        framework: terraform

  deploy-staging:
    needs: build-and-scan
    runs-on: ubuntu-latest
    steps:
    - name: Deploy to Staging
      run: |
        kubectl set image deployment/myapp \
          myapp=ghcr.io/${{ github.repository }}/myapp:${{ github.sha }} \
          -n staging

  dast:
    needs: deploy-staging
    runs-on: ubuntu-latest
    steps:
    - name: OWASP ZAP Scan
      uses: zaproxy/action-full-scan@v0.4.0
      with:
        target: 'https://staging.myapp.com'

  deploy-production:
    needs: [dast]
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    environment:
      name: production
    steps:
    - name: Deploy to Production
      run: |
        kubectl set image deployment/myapp \
          myapp=ghcr.io/${{ github.repository }}/myapp:${{ github.sha }} \
          -n production
```

## Security Metrics

### Key Metrics to Track:
```yaml
1. Mean Time to Detect (MTTD)
   - Time to identify security issues

2. Mean Time to Resolve (MTTR)
   - Time to fix security issues

3. Vulnerability Density
   - Vulnerabilities per 1000 lines of code

4. Security Test Coverage
   - Percentage of code with security tests

5. Failed Security Checks
   - Number of pipeline failures due to security

6. Security Debt
   - Known vulnerabilities not yet fixed

7. Compliance Score
   - Adherence to security standards
```

## Best Practices

### 1. Shift Left
```yaml
- Integrate security from day one
- Use IDE security plugins
- Pre-commit hooks for security checks
- Educate developers on security
```

### 2. Automate Everything
```yaml
- Automated security testing
- Automated compliance checks
- Automated remediation where possible
- Automated reporting
```

### 3. Defense in Depth
```yaml
- Multiple layers of security
- Don't rely on single security measure
- Network segmentation
- Principle of least privilege
```

### 4. Fail Fast
```yaml
- Fail pipeline on critical issues
- Provide clear error messages
- Quick feedback to developers
```

### 5. Continuous Monitoring
```yaml
- Monitor in production
- Set up alerts
- Log everything
- Regular security audits
```

### 6. Security as Code
```yaml
- Version control security policies
- Infrastructure as Code
- Policy as Code
- Immutable infrastructure
```

### 7. Least Privilege
```yaml
- Minimal permissions
- RBAC (Role-Based Access Control)
- Service accounts with specific roles
- Regular access reviews
```

### 8. Zero Trust
```yaml
- Never trust, always verify
- Verify every request
- Micro-segmentation
- Strong authentication
```

## Common Vulnerabilities

### OWASP Top 10:
```yaml
1. Broken Access Control
2. Cryptographic Failures
3. Injection
4. Insecure Design
5. Security Misconfiguration
6. Vulnerable Components
7. Authentication Failures
8. Data Integrity Failures
9. Logging Failures
10. SSRF (Server-Side Request Forgery)
```

## Compliance and Standards

```yaml
Standards:
  - CIS Benchmarks
  - NIST Cybersecurity Framework
  - ISO 27001
  - PCI DSS
  - HIPAA
  - GDPR
  - SOC 2
```

## Summary

- DevSecOps integrates security throughout SDLC
- Shift left: find security issues early
- Automate security testing and scanning
- Use multiple security tools (SAST, DAST, SCA, etc.)
- Container and Kubernetes security are critical
- Secrets management is essential
- Monitor and respond to security events in real-time
- Shared responsibility model
- Continuous improvement and learning
- Security should never slow down delivery - automate it
