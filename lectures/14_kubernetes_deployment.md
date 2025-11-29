# Kubernetes Deployment

## What is a Deployment?

A **Deployment** is a Kubernetes resource that provides declarative updates for Pods and ReplicaSets. It's the most commonly used controller for managing stateless applications in Kubernetes.

### Key Features:
- Declarative updates for Pods and ReplicaSets
- Rolling updates with zero downtime
- Rollback capability to previous versions
- Scaling applications up or down
- Pause and resume deployments
- Self-healing capabilities

## Why Use Deployments?

1. **Zero-Downtime Updates**: Rolling updates ensure application availability
2. **Easy Rollbacks**: Revert to previous version if issues occur
3. **Version Control**: Track deployment history
4. **Declarative Configuration**: Define desired state, Kubernetes handles the rest
5. **Automated Management**: Handles ReplicaSet creation and updates

## Deployment Architecture

```
Deployment
    ↓
ReplicaSet (v1) → ReplicaSet (v2) → ReplicaSet (v3)
    ↓                ↓                  ↓
Pods (v1)        Pods (v2)          Pods (v3)
```

## Basic Deployment YAML

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
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
        image: nginx:1.21
        ports:
        - containerPort: 80
```

## Creating and Managing Deployments

### Create Deployment:
```bash
# From YAML file
kubectl apply -f deployment.yaml

# Imperative command
kubectl create deployment nginx --image=nginx:1.21 --replicas=3
```

### View Deployments:
```bash
# List all deployments
kubectl get deployments

# Short form
kubectl get deploy

# Detailed view
kubectl get deploy -o wide

# Watch in real-time
kubectl get deploy --watch
```

### Describe Deployment:
```bash
kubectl describe deployment <deployment-name>
```

### Delete Deployment:
```bash
kubectl delete deployment <deployment-name>
```

## Scaling Deployments

### Scale Up/Down:
```bash
# Scale to 5 replicas
kubectl scale deployment <deployment-name> --replicas=5

# Scale multiple deployments
kubectl scale deployment <deploy1> <deploy2> --replicas=3
```

### Autoscaling:
```bash
# Create Horizontal Pod Autoscaler
kubectl autoscale deployment <deployment-name> --min=2 --max=10 --cpu-percent=80
```

## Update Strategies

### 1. Rolling Update (Default)

Gradually replaces old pods with new ones:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: rolling-deployment
spec:
  replicas: 5
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1        # Max pods above desired count
      maxUnavailable: 1  # Max pods unavailable during update
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
        image: nginx:1.21
```

**Parameters:**
- `maxSurge`: Maximum number of extra pods created during update
- `maxUnavailable`: Maximum number of pods that can be unavailable

### 2. Recreate Strategy

Terminates all old pods before creating new ones:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: recreate-deployment
spec:
  replicas: 3
  strategy:
    type: Recreate
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
        image: nginx:1.21
```

**Use Case**: When you can't have multiple versions running simultaneously

## Updating Deployments

### Method 1: Update Image
```bash
# Update container image
kubectl set image deployment/<deployment-name> <container-name>=<new-image>:<tag>

# Example
kubectl set image deployment/nginx-deployment nginx=nginx:1.22
```

### Method 2: Edit Deployment
```bash
kubectl edit deployment <deployment-name>
```

### Method 3: Apply Updated YAML
```bash
# Update YAML file, then apply
kubectl apply -f deployment.yaml
```

### Method 4: Patch Deployment
```bash
kubectl patch deployment <deployment-name> -p '{"spec":{"replicas":5}}'
```

## Rollout Management

### Check Rollout Status:
```bash
kubectl rollout status deployment/<deployment-name>
```

### View Rollout History:
```bash
# View revision history
kubectl rollout history deployment/<deployment-name>

# View specific revision details
kubectl rollout history deployment/<deployment-name> --revision=2
```

### Pause Rollout:
```bash
kubectl rollout pause deployment/<deployment-name>

# Make multiple changes while paused
kubectl set image deployment/<deployment-name> nginx=nginx:1.22
kubectl set resources deployment/<deployment-name> -c=nginx --limits=cpu=200m,memory=512Mi

# Resume rollout
kubectl rollout resume deployment/<deployment-name>
```

### Rollback:
```bash
# Rollback to previous version
kubectl rollout undo deployment/<deployment-name>

# Rollback to specific revision
kubectl rollout undo deployment/<deployment-name> --to-revision=2
```

## Deployment Examples

### Example 1: Web Application with Health Checks

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: webapp
  template:
    metadata:
      labels:
        app: webapp
    spec:
      containers:
      - name: webapp
        image: nginx:1.21
        ports:
        - containerPort: 80
        livenessProbe:
          httpGet:
            path: /healthz
            port: 80
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
        resources:
          requests:
            memory: "64Mi"
            cpu: "250m"
          limits:
            memory: "128Mi"
            cpu: "500m"
```

### Example 2: Application with Environment Variables

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: env-deployment
spec:
  replicas: 2
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
        image: myapp:1.0
        env:
        - name: DATABASE_URL
          value: "postgres://db:5432/mydb"
        - name: ENVIRONMENT
          value: "production"
        - name: SECRET_KEY
          valueFrom:
            secretKeyRef:
              name: app-secrets
              key: secret-key
```

### Example 3: Multi-Container Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: multi-container-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: multiapp
  template:
    metadata:
      labels:
        app: multiapp
    spec:
      containers:
      - name: nginx
        image: nginx:alpine
        ports:
        - containerPort: 80
        volumeMounts:
        - name: shared-data
          mountPath: /usr/share/nginx/html
      - name: content-updater
        image: busybox
        command: ['sh', '-c', 'while true; do echo "Updated: $(date)" > /data/index.html; sleep 10; done']
        volumeMounts:
        - name: shared-data
          mountPath: /data
      volumes:
      - name: shared-data
        emptyDir: {}
```

## Deployment Status

### Understanding Status:
```bash
kubectl get deployment <deployment-name>
```

Output columns:
- **READY**: Number of ready replicas / Desired replicas
- **UP-TO-DATE**: Number of replicas updated to latest version
- **AVAILABLE**: Number of available replicas
- **AGE**: Time since deployment creation

### Conditions:
```bash
kubectl describe deployment <deployment-name>
```

Look for conditions:
- **Available**: Deployment has minimum availability
- **Progressing**: Deployment is making progress
- **ReplicaFailure**: ReplicaSet cannot create pods

## Monitoring and Troubleshooting

### Check Deployment Events:
```bash
kubectl describe deployment <deployment-name>
```

### View Associated ReplicaSets:
```bash
kubectl get rs -l app=<label>
```

### View Pods:
```bash
kubectl get pods -l app=<label>
```

### Check Logs:
```bash
# Logs from all pods in deployment
kubectl logs -l app=<label> --all-containers=true

# Follow logs
kubectl logs -l app=<label> -f
```

### Common Issues:

#### Issue 1: ImagePullBackOff
```bash
# Check pod events
kubectl describe pod <pod-name>

# Verify image name and tag
# Check image pull secrets if using private registry
```

#### Issue 2: CrashLoopBackOff
```bash
# Check pod logs
kubectl logs <pod-name>

# Check previous container logs
kubectl logs <pod-name> --previous
```

#### Issue 3: Insufficient Resources
```bash
# Check node resources
kubectl describe nodes

# Adjust resource requests/limits in deployment
```

## Best Practices

1. **Use Specific Image Tags**: Avoid `latest` tag for production
2. **Set Resource Limits**: Define CPU and memory limits
3. **Health Checks**: Always implement liveness and readiness probes
4. **Rolling Updates**: Use rolling update strategy for zero downtime
5. **Revision History**: Keep `revisionHistoryLimit` reasonable (default is 10)
6. **Labels**: Use meaningful labels for organization
7. **Namespace**: Use namespaces to separate environments
8. **Test Updates**: Test in staging before production
9. **Gradual Rollout**: Use maxSurge and maxUnavailable wisely
10. **Monitor**: Watch rollout status during updates

## Advanced Features

### Revision History Limit:
```yaml
spec:
  revisionHistoryLimit: 5  # Keep only last 5 ReplicaSets
```

### Progress Deadline:
```yaml
spec:
  progressDeadlineSeconds: 600  # Fail if deployment takes more than 10 minutes
```

### Min Ready Seconds:
```yaml
spec:
  minReadySeconds: 30  # Wait 30 seconds before considering pod available
```

## Complete Deployment Example

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: production-app
  namespace: production
  labels:
    app: myapp
    environment: production
spec:
  replicas: 5
  revisionHistoryLimit: 5
  progressDeadlineSeconds: 600
  minReadySeconds: 10
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
        version: v1.0.0
    spec:
      containers:
      - name: myapp
        image: myapp:1.0.0
        ports:
        - containerPort: 8080
        env:
        - name: ENVIRONMENT
          value: "production"
        resources:
          requests:
            memory: "128Mi"
            cpu: "250m"
          limits:
            memory: "256Mi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 3
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 5
          timeoutSeconds: 3
          failureThreshold: 3
```

## Hands-On Exercise

Practice deployment lifecycle:

```bash
# 1. Create deployment
kubectl create deployment nginx --image=nginx:1.21 --replicas=3

# 2. Check status
kubectl get deployment nginx
kubectl rollout status deployment/nginx

# 3. Update image
kubectl set image deployment/nginx nginx=nginx:1.22

# 4. Watch rollout
kubectl rollout status deployment/nginx

# 5. Check history
kubectl rollout history deployment/nginx

# 6. Rollback
kubectl rollout undo deployment/nginx

# 7. Scale
kubectl scale deployment nginx --replicas=5

# 8. Clean up
kubectl delete deployment nginx
```

## Summary

- Deployments are the standard way to manage stateless applications in Kubernetes
- They provide rolling updates, rollbacks, and scaling capabilities
- Use rolling update strategy for zero-downtime deployments
- Always implement health checks and resource limits
- Monitor rollout status and keep revision history
- Deployments manage ReplicaSets, which manage Pods
- Essential for production Kubernetes workloads
