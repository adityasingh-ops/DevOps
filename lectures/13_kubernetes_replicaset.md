# Kubernetes ReplicaSet

## What is a ReplicaSet?

A **ReplicaSet** is a Kubernetes controller that ensures a specified number of pod replicas are running at any given time. It maintains the desired state by creating or deleting pods as needed.

### Key Points:
- Ensures high availability of applications
- Maintains the desired number of pod replicas
- Automatically replaces failed pods
- Successor to ReplicationController (older API)
- Usually managed by Deployments (don't create ReplicaSets directly)

## Why ReplicaSets?

1. **High Availability**: Ensures your application stays running
2. **Load Balancing**: Multiple pods can handle more requests
3. **Self-Healing**: Automatically replaces failed pods
4. **Scalability**: Easy to scale up or down

## ReplicaSet Architecture

```
ReplicaSet Controller
    ↓
Monitors Pods with matching labels
    ↓
Creates/Deletes Pods to match desired count
```

## Basic ReplicaSet YAML

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: nginx-replicaset
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
        image: nginx:latest
        ports:
        - containerPort: 80
```

### Key Components:

1. **replicas**: Number of pod copies to maintain
2. **selector**: How ReplicaSet finds which pods to manage
3. **template**: Pod template used to create new pods

## Creating and Managing ReplicaSets

### Create ReplicaSet:
```bash
kubectl apply -f replicaset.yaml
```

### View ReplicaSets:
```bash
# List all ReplicaSets
kubectl get replicaset

# Short form
kubectl get rs

# Detailed view
kubectl get rs -o wide

# Watch in real-time
kubectl get rs --watch
```

### Describe ReplicaSet:
```bash
kubectl describe rs <replicaset-name>
```

### Delete ReplicaSet:
```bash
# Delete ReplicaSet and its pods
kubectl delete rs <replicaset-name>

# Delete ReplicaSet but keep pods
kubectl delete rs <replicaset-name> --cascade=orphan
```

## Scaling ReplicaSets

### Method 1: Using kubectl scale
```bash
# Scale to 5 replicas
kubectl scale rs <replicaset-name> --replicas=5

# Scale multiple ReplicaSets
kubectl scale rs <rs1> <rs2> --replicas=3
```

### Method 2: Edit the ReplicaSet
```bash
kubectl edit rs <replicaset-name>
# Change replicas value and save
```

### Method 3: Update YAML file
```bash
# Update replicas in YAML file, then:
kubectl apply -f replicaset.yaml
```

## Label Selectors

ReplicaSets use labels to identify which pods to manage:

### Equality-based Selector:
```yaml
selector:
  matchLabels:
    app: nginx
    tier: frontend
```

### Set-based Selector:
```yaml
selector:
  matchExpressions:
  - key: app
    operator: In
    values:
    - nginx
    - apache
  - key: tier
    operator: NotIn
    values:
    - backend
```

## ReplicaSet Examples

### Example 1: Simple Web Application
```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: webapp-rs
spec:
  replicas: 4
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
        image: nginx:alpine
        ports:
        - containerPort: 80
```

### Example 2: With Resource Limits
```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: resource-limited-rs
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
        image: nginx
        resources:
          requests:
            memory: "64Mi"
            cpu: "250m"
          limits:
            memory: "128Mi"
            cpu: "500m"
```

### Example 3: With Environment Variables
```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: env-rs
spec:
  replicas: 2
  selector:
    matchLabels:
      app: envapp
  template:
    metadata:
      labels:
        app: envapp
    spec:
      containers:
      - name: app
        image: busybox
        command: ['sh', '-c', 'echo "Environment: $ENV_NAME" && sleep 3600']
        env:
        - name: ENV_NAME
          value: "production"
        - name: APP_VERSION
          value: "1.0.0"
```

## How ReplicaSet Works

1. **Continuous Monitoring**: ReplicaSet controller constantly watches pods
2. **Label Matching**: Identifies pods based on label selectors
3. **Reconciliation Loop**: Compares current state with desired state
4. **Action**: Creates new pods if count is low, deletes if count is high

## Pod Acquisition

ReplicaSet can "adopt" existing pods that match its selector:

```bash
# Create standalone pods with matching labels
kubectl run nginx-1 --image=nginx --labels="app=nginx"
kubectl run nginx-2 --image=nginx --labels="app=nginx"

# Create ReplicaSet with replicas=3 and selector matching "app=nginx"
# ReplicaSet will adopt the 2 existing pods and create 1 more
```

## Troubleshooting ReplicaSets

### Check ReplicaSet Status:
```bash
kubectl get rs <replicaset-name>
```

Output columns:
- **DESIRED**: Number of replicas specified
- **CURRENT**: Number of replicas currently running
- **READY**: Number of replicas ready to serve traffic

### Common Issues:

#### Issue 1: Pods Not Starting
```bash
# Check ReplicaSet events
kubectl describe rs <replicaset-name>

# Check pod status
kubectl get pods -l app=<label>

# Check specific pod details
kubectl describe pod <pod-name>
```

#### Issue 2: Image Pull Errors
```bash
# Check pod events
kubectl describe pod <pod-name>

# Look for ImagePullBackOff or ErrImagePull
```

#### Issue 3: Resource Constraints
```bash
# Check node resources
kubectl describe nodes

# Check if pods are in Pending state due to insufficient resources
```

## ReplicaSet vs Deployment

| Feature | ReplicaSet | Deployment |
|---------|-----------|------------|
| Rolling Updates | ❌ No | ✅ Yes |
| Rollback | ❌ No | ✅ Yes |
| Update Strategy | Manual | Automated |
| Use Case | Rarely used directly | Recommended |
| Revision History | ❌ No | ✅ Yes |

**Best Practice**: Use Deployments instead of ReplicaSets directly. Deployments manage ReplicaSets and provide additional features.

## Monitoring Commands

```bash
# Watch pods in real-time
kubectl get pods -l app=<label> --watch

# Check ReplicaSet status
kubectl get rs <replicaset-name> -o yaml

# View events
kubectl get events --field-selector involvedObject.kind=ReplicaSet

# Check logs from all pods
kubectl logs -l app=<label> --all-containers=true
```

## Best Practices

1. **Don't Create ReplicaSets Directly**: Use Deployments instead
2. **Unique Labels**: Ensure pod template has unique labels
3. **Resource Limits**: Always set resource requests and limits
4. **Health Checks**: Include liveness and readiness probes
5. **Meaningful Names**: Use descriptive names for debugging
6. **Version Tags**: Use specific image tags, not `latest`
7. **Label Strategy**: Use consistent labeling convention

## Hands-On Exercise

Create a ReplicaSet and observe its behavior:

```bash
# 1. Create ReplicaSet with 3 replicas
kubectl apply -f replicaset.yaml

# 2. Verify pods created
kubectl get pods -l app=nginx

# 3. Delete one pod
kubectl delete pod <pod-name>

# 4. Watch ReplicaSet recreate the pod
kubectl get pods -l app=nginx --watch

# 5. Scale up to 5 replicas
kubectl scale rs nginx-replicaset --replicas=5

# 6. Verify scaling
kubectl get rs nginx-replicaset

# 7. Clean up
kubectl delete rs nginx-replicaset
```

## Summary

- ReplicaSets ensure a specified number of pod replicas are running
- They provide high availability and self-healing capabilities
- Use label selectors to identify which pods to manage
- Can be scaled up or down easily
- In practice, use Deployments which manage ReplicaSets
- Essential for understanding Kubernetes workload management
