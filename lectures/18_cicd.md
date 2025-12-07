# CI/CD: Continuous Integration and Continuous Deployment

## What is CI/CD?

**CI/CD** is a software development practice that automates the integration, testing, and deployment of code changes.

### Continuous Integration (CI)
Automatically building and testing code changes when developers commit to version control.

### Continuous Delivery (CD)
Automatically preparing code changes for release to production (manual approval).

### Continuous Deployment (CD)
Automatically deploying code changes to production (fully automated).

## Why CI/CD?

1. **Faster Time to Market**: Automated workflows accelerate releases
2. **Reduced Risk**: Small, frequent changes are easier to debug
3. **Better Quality**: Automated testing catches bugs early
4. **Developer Productivity**: Less time on manual processes
5. **Consistent Deployments**: Standardized deployment process
6. **Rapid Feedback**: Quick identification of issues

## CI/CD Pipeline Stages

```
Code Commit
    ↓
Build
    ↓
Unit Tests
    ↓
Integration Tests
    ↓
Security Scan
    ↓
Deploy to Staging
    ↓
End-to-End Tests
    ↓
Deploy to Production
    ↓
Monitoring
```

## Popular CI/CD Tools

### 1. Jenkins
- Open-source automation server
- Highly customizable with plugins
- Self-hosted

### 2. GitHub Actions
- Native to GitHub
- YAML-based workflows
- Free for public repos

### 3. GitLab CI/CD
- Integrated with GitLab
- Auto DevOps feature
- Built-in container registry

### 4. CircleCI
- Cloud-based
- Fast builds with caching
- Docker support

### 5. Travis CI
- Simple configuration
- GitHub integration
- Free for open-source

### 6. Azure Pipelines
- Microsoft's CI/CD service
- Multi-platform support
- Free tier available

### 7. ArgoCD
- GitOps for Kubernetes
- Declarative continuous deployment
- Automatic sync

## GitHub Actions

### Basic Workflow Structure:

```yaml
name: CI/CD Pipeline

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  build:
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Set up Node.js
      uses: actions/setup-node@v3
      with:
        node-version: '18'
    
    - name: Install dependencies
      run: npm install
    
    - name: Run tests
      run: npm test
    
    - name: Build
      run: npm run build
```

### Complete CI/CD Example:

```yaml
name: Complete CI/CD

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

env:
  DOCKER_IMAGE: myapp
  REGISTRY: ghcr.io

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    
    - name: Set up Node.js
      uses: actions/setup-node@v3
      with:
        node-version: '18'
        cache: 'npm'
    
    - name: Install dependencies
      run: npm ci
    
    - name: Run linter
      run: npm run lint
    
    - name: Run tests
      run: npm test
    
    - name: Run coverage
      run: npm run coverage
    
    - name: Upload coverage
      uses: codecov/codecov-action@v3
  
  build:
    needs: test
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    
    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v2
    
    - name: Log in to Container Registry
      uses: docker/login-action@v2
      with:
        registry: ${{ env.REGISTRY }}
        username: ${{ github.actor }}
        password: ${{ secrets.GITHUB_TOKEN }}
    
    - name: Build and push Docker image
      uses: docker/build-push-action@v4
      with:
        context: .
        push: true
        tags: |
          ${{ env.REGISTRY }}/${{ github.repository }}/${{ env.DOCKER_IMAGE }}:latest
          ${{ env.REGISTRY }}/${{ github.repository }}/${{ env.DOCKER_IMAGE }}:${{ github.sha }}
        cache-from: type=gha
        cache-to: type=gha,mode=max
  
  deploy-staging:
    needs: build
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/develop'
    steps:
    - name: Deploy to Staging
      run: |
        echo "Deploying to staging environment"
        # Add deployment commands here
  
  deploy-production:
    needs: build
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    environment:
      name: production
      url: https://myapp.com
    steps:
    - name: Deploy to Production
      run: |
        echo "Deploying to production"
        # Add deployment commands here
```

## Jenkins Pipeline

### Declarative Pipeline:

```groovy
pipeline {
    agent any
    
    environment {
        DOCKER_IMAGE = 'myapp'
        REGISTRY = 'docker.io'
    }
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        
        stage('Build') {
            steps {
                sh 'npm install'
                sh 'npm run build'
            }
        }
        
        stage('Test') {
            steps {
                sh 'npm test'
                sh 'npm run lint'
            }
        }
        
        stage('Security Scan') {
            steps {
                sh 'npm audit'
            }
        }
        
        stage('Docker Build') {
            steps {
                script {
                    docker.build("${DOCKER_IMAGE}:${BUILD_NUMBER}")
                }
            }
        }
        
        stage('Push to Registry') {
            steps {
                script {
                    docker.withRegistry("https://${REGISTRY}", 'docker-credentials') {
                        docker.image("${DOCKER_IMAGE}:${BUILD_NUMBER}").push()
                        docker.image("${DOCKER_IMAGE}:${BUILD_NUMBER}").push('latest')
                    }
                }
            }
        }
        
        stage('Deploy to Staging') {
            steps {
                sh 'kubectl set image deployment/myapp myapp=${DOCKER_IMAGE}:${BUILD_NUMBER} -n staging'
            }
        }
        
        stage('Deploy to Production') {
            when {
                branch 'main'
            }
            steps {
                input message: 'Deploy to Production?', ok: 'Deploy'
                sh 'kubectl set image deployment/myapp myapp=${DOCKER_IMAGE}:${BUILD_NUMBER} -n production'
            }
        }
    }
    
    post {
        success {
            echo 'Pipeline succeeded!'
            // Send notification
        }
        failure {
            echo 'Pipeline failed!'
            // Send notification
        }
        always {
            cleanWs()
        }
    }
}
```

## GitLab CI/CD

### .gitlab-ci.yml:

```yaml
stages:
  - build
  - test
  - security
  - deploy-staging
  - deploy-production

variables:
  DOCKER_IMAGE: $CI_REGISTRY_IMAGE
  DOCKER_TAG: $CI_COMMIT_SHORT_SHA

before_script:
  - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY

build:
  stage: build
  script:
    - docker build -t $DOCKER_IMAGE:$DOCKER_TAG .
    - docker push $DOCKER_IMAGE:$DOCKER_TAG
  only:
    - main
    - develop

test:
  stage: test
  script:
    - npm install
    - npm test
    - npm run lint
  coverage: '/Lines\s*:\s*(\d+\.\d+)%/'
  artifacts:
    reports:
      coverage_report:
        coverage_format: cobertura
        path: coverage/cobertura-coverage.xml

security-scan:
  stage: security
  script:
    - npm audit
    - docker scan $DOCKER_IMAGE:$DOCKER_TAG
  allow_failure: true

deploy-staging:
  stage: deploy-staging
  script:
    - kubectl config use-context staging
    - kubectl set image deployment/myapp myapp=$DOCKER_IMAGE:$DOCKER_TAG -n staging
    - kubectl rollout status deployment/myapp -n staging
  environment:
    name: staging
    url: https://staging.myapp.com
  only:
    - develop

deploy-production:
  stage: deploy-production
  script:
    - kubectl config use-context production
    - kubectl set image deployment/myapp myapp=$DOCKER_IMAGE:$DOCKER_TAG -n production
    - kubectl rollout status deployment/myapp -n production
  environment:
    name: production
    url: https://myapp.com
  when: manual
  only:
    - main
```

## Docker in CI/CD

### Dockerfile for CI/CD:

```dockerfile
# Build stage
FROM node:18-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .
RUN npm run build

# Production stage
FROM node:18-alpine
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/package.json ./
EXPOSE 3000
USER node
CMD ["npm", "start"]
```

## Kubernetes Deployment with CI/CD

### Deployment YAML:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: myapp
        image: myregistry.io/myapp:TAG_PLACEHOLDER
        ports:
        - containerPort: 3000
        env:
        - name: NODE_ENV
          value: "production"
        livenessProbe:
          httpGet:
            path: /health
            port: 3000
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 3000
          initialDelaySeconds: 10
          periodSeconds: 5
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "200m"
```

### Update Image in Pipeline:

```bash
# Replace tag in YAML
sed -i "s/TAG_PLACEHOLDER/${BUILD_TAG}/g" deployment.yaml

# Apply to cluster
kubectl apply -f deployment.yaml

# Or use kubectl set image
kubectl set image deployment/myapp myapp=myregistry.io/myapp:${BUILD_TAG} -n production
```

## ArgoCD (GitOps)

### Application YAML:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/myorg/myapp
    targetRevision: main
    path: k8s
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
```

### GitOps Workflow:
1. Developer pushes code to Git
2. CI pipeline builds and pushes Docker image
3. CI updates image tag in Kubernetes manifests
4. ArgoCD detects changes in Git
5. ArgoCD syncs changes to cluster
6. Application updated automatically

## Best Practices

### 1. Version Control Everything
```
- Application code
- Infrastructure as Code
- CI/CD configurations
- Kubernetes manifests
```

### 2. Automated Testing
```yaml
- Unit tests
- Integration tests
- End-to-end tests
- Security scans
- Performance tests
```

### 3. Build Once, Deploy Many
```
Build → Docker Image → Deploy to Dev → Staging → Production
(Same artifact throughout)
```

### 4. Environment Parity
```
Development ≈ Staging ≈ Production
(Minimize differences)
```

### 5. Secrets Management
```yaml
# Use secret management tools
- Kubernetes Secrets
- HashiCorp Vault
- AWS Secrets Manager
- Azure Key Vault
```

### 6. Rollback Strategy
```yaml
# Always have rollback capability
- Keep previous versions
- Use blue-green deployments
- Implement canary releases
```

### 7. Monitoring and Alerts
```yaml
- Monitor pipeline success/failure
- Track deployment metrics
- Set up alerts for failures
- Log everything
```

### 8. Security Scanning
```yaml
stages:
  - SAST (Static Application Security Testing)
  - Dependency scanning
  - Container scanning
  - DAST (Dynamic Application Security Testing)
```

### 9. Fast Feedback
```yaml
- Run fast tests first
- Parallelize when possible
- Fail fast on errors
- Clear error messages
```

### 10. Documentation
```yaml
- Document pipeline stages
- Explain deployment process
- Maintain runbooks
- Keep diagrams updated
```

## Deployment Strategies

### 1. Rolling Update
```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1
    maxUnavailable: 0
```

### 2. Blue-Green Deployment
```bash
# Deploy new version (green)
kubectl apply -f deployment-green.yaml

# Test green environment
curl http://green.example.com

# Switch traffic to green
kubectl patch service myapp -p '{"spec":{"selector":{"version":"green"}}}'

# Keep blue for rollback
```

### 3. Canary Deployment
```yaml
# Canary (10% traffic)
replicas: 1
version: v2

# Stable (90% traffic)
replicas: 9
version: v1
```

### 4. A/B Testing
```yaml
# Route based on headers/cookies
nginx.ingress.kubernetes.io/canary: "true"
nginx.ingress.kubernetes.io/canary-by-header: "version"
nginx.ingress.kubernetes.io/canary-by-header-value: "v2"
```

## Complete Example: Node.js App

### GitHub Actions Workflow:

```yaml
name: Node.js CI/CD

on:
  push:
    branches: [ main ]

env:
  NODE_VERSION: '18'
  DOCKER_IMAGE: ghcr.io/${{ github.repository }}/myapp

jobs:
  test-and-build:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    
    - name: Setup Node.js
      uses: actions/setup-node@v3
      with:
        node-version: ${{ env.NODE_VERSION }}
        cache: 'npm'
    
    - name: Install dependencies
      run: npm ci
    
    - name: Run linter
      run: npm run lint
    
    - name: Run tests
      run: npm test
    
    - name: Run security audit
      run: npm audit --audit-level=moderate
    
    - name: Build Docker image
      run: docker build -t ${{ env.DOCKER_IMAGE }}:${{ github.sha }} .
    
    - name: Log in to GitHub Container Registry
      uses: docker/login-action@v2
      with:
        registry: ghcr.io
        username: ${{ github.actor }}
        password: ${{ secrets.GITHUB_TOKEN }}
    
    - name: Push Docker image
      run: |
        docker push ${{ env.DOCKER_IMAGE }}:${{ github.sha }}
        docker tag ${{ env.DOCKER_IMAGE }}:${{ github.sha }} ${{ env.DOCKER_IMAGE }}:latest
        docker push ${{ env.DOCKER_IMAGE }}:latest
  
  deploy:
    needs: test-and-build
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    
    - name: Configure kubectl
      uses: azure/k8s-set-context@v3
      with:
        method: kubeconfig
        kubeconfig: ${{ secrets.KUBE_CONFIG }}
    
    - name: Deploy to Kubernetes
      run: |
        kubectl set image deployment/myapp \
          myapp=${{ env.DOCKER_IMAGE }}:${{ github.sha }} \
          -n production
        
        kubectl rollout status deployment/myapp -n production
    
    - name: Verify deployment
      run: |
        kubectl get pods -n production
        kubectl get svc -n production
```

## Troubleshooting CI/CD

### Pipeline Fails at Build
```bash
# Check logs
# Verify dependencies
# Ensure correct versions
# Check environment variables
```

### Tests Failing
```bash
# Run tests locally
# Check test environment
# Verify test data
# Review recent changes
```

### Deployment Fails
```bash
# Check Kubernetes cluster status
kubectl get nodes
kubectl get pods -n production

# Review deployment events
kubectl describe deployment myapp -n production

# Check logs
kubectl logs deployment/myapp -n production
```

### Image Pull Errors
```bash
# Verify image exists in registry
# Check registry credentials
# Verify imagePullSecrets
kubectl get secret -n production
```

## Summary

- CI/CD automates build, test, and deployment processes
- Reduces manual errors and speeds up delivery
- Multiple tools available (Jenkins, GitHub Actions, GitLab CI, etc.)
- Essential components: build, test, security scan, deploy
- Follow best practices: version control, automated testing, secrets management
- Use appropriate deployment strategies (rolling, blue-green, canary)
- GitOps with ArgoCD for Kubernetes deployments
- Monitor pipelines and set up alerts
- Document processes and maintain runbooks
- Critical for modern software development and DevOps
