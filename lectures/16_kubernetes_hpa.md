# Kubernetes Horizontal Pod Autoscaler (HPA)

## What is HPA?

**Horizontal Pod Autoscaler (HPA)** automatically scales the number of Pods in a Deployment, ReplicaSet, or StatefulSet based on observed metrics like CPU utilization, memory usage, or custom metrics.

### Key Concepts:
- **Horizontal Scaling**: Adding/removing Pod replicas
- **Automatic**: Scales without manual intervention
- **Metric-based**: Decisions based on resource metrics
- **Target-based**: Maintains target metric value

## How HPA Works

```
HPA Controller (checks every 15s by default)
    ↓
Queries Metrics Server
    ↓
Compares current vs target metrics
    ↓
Calculates desired replica count
    ↓
Updates Deployment/ReplicaSet
    ↓
Pods scaled up/down
```

### Formula:
```
desiredReplicas = ceil[currentReplicas * (currentMetricValue / targetMetricValue)]
```

## Prerequisites

### 1. Metrics Server
HPA requires Metrics Server to collect resource metrics:

```bash
# Check if Metrics Server is running
kubectl get deployment metrics-server -n kube-system

# Install Metrics Server (if not present)
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# For minikube
minikube addons enable metrics-server

# Verify metrics are available
kubectl top nodes
kubectl top pods
```

### 2. Resource Requests
Pods must have resource requests defined:

```yaml
resources:
  requests:
    cpu: 200m
    memory: 128Mi
```

## Basic HPA Example

### 1. Create Deployment:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: php-apache
spec:
  replicas: 1
  selector:
    matchLabels:
      app: php-apache
  template:
    metadata:
      labels:
        app: php-apache
    spec:
      containers:
      - name: php-apache
        image: registry.k8s.io/hpa-example
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: 200m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 256Mi
```

### 2. Create HPA:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: php-apache-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: php-apache
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
```

## HPA Management Commands

### Create HPA:

```bash
# Imperative (simple CPU-based)
kubectl autoscale deployment php-apache --cpu-percent=50 --min=1 --max=10

# Declarative (YAML)
kubectl apply -f hpa.yaml
```

### View HPA:

```bash
# List all HPAs
kubectl get hpa

# Watch HPA in real-time
kubectl get hpa --watch

# Detailed view
kubectl describe hpa <hpa-name>
```

### Delete HPA:

```bash
kubectl delete hpa <hpa-name>
```

## HPA Metrics Types

### 1. Resource Metrics (CPU/Memory)

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: cpu-memory-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
```

### 2. Custom Metrics

Using custom application metrics:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: custom-metric-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  minReplicas: 2
  maxReplicas: 20
  metrics:
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second
      target:
        type: AverageValue
        averageValue: "1000"
```

### 3. External Metrics

Using metrics from external systems (e.g., Prometheus):

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: external-metric-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: External
    external:
      metric:
        name: queue_messages_ready
        selector:
          matchLabels:
            queue: worker_tasks
      target:
        type: AverageValue
        averageValue: "30"
```

## HPA Behavior Configuration

Control scaling behavior (velocity and stabilization):

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: behavior-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  minReplicas: 2
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300  # 5 minutes
      policies:
      - type: Percent
        value: 50
        periodSeconds: 60
      - type: Pods
        value: 2
        periodSeconds: 60
      selectPolicy: Min
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
      - type: Percent
        value: 100
        periodSeconds: 30
      - type: Pods
        value: 4
        periodSeconds: 30
      selectPolicy: Max
```

### Behavior Parameters:

- **stabilizationWindowSeconds**: How long to wait before scaling
- **policies**: Rate of scaling (Pods or Percent per period)
- **selectPolicy**: Which policy to use (Min, Max, Disabled)

## Scaling Policies

### Conservative Scaling:
```yaml
behavior:
  scaleDown:
    stabilizationWindowSeconds: 300  # Wait 5 min before scaling down
    policies:
    - type: Pods
      value: 1
      periodSeconds: 60  # Max 1 pod per minute
```

### Aggressive Scaling:
```yaml
behavior:
  scaleUp:
    stabilizationWindowSeconds: 0  # Scale up immediately
    policies:
    - type: Percent
      value: 100  # Double pods
      periodSeconds: 15
```

## Testing HPA

### Load Test Example:

```bash
# 1. Create deployment and HPA
kubectl apply -f deployment.yaml
kubectl apply -f hpa.yaml

# 2. Expose service
kubectl expose deployment php-apache --port=80 --type=ClusterIP

# 3. Watch HPA
kubectl get hpa --watch

# 4. Generate load (in another terminal)
kubectl run load-generator --image=busybox -it --rm -- /bin/sh -c "while true; do wget -q -O- http://php-apache; done"

# 5. Observe scaling
kubectl get hpa
kubectl get pods

# 6. Stop load (Ctrl+C in load generator terminal)

# 7. Watch scale down
kubectl get hpa --watch
```

## HPA Status

### Check HPA Status:
```bash
kubectl get hpa php-apache-hpa

# Output:
# NAME              REFERENCE              TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
# php-apache-hpa    Deployment/php-apache  45%/50%   1         10        3          5m
```

### Status Fields:
- **TARGETS**: current/target metric value
- **REPLICAS**: current number of replicas
- **MINPODS/MAXPODS**: scaling boundaries

### Describe HPA:
```bash
kubectl describe hpa php-apache-hpa
```

Shows:
- Current metrics
- Scaling events
- Conditions
- Last scale time

## Multiple Metrics

HPA can use multiple metrics simultaneously:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: multi-metric-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  minReplicas: 2
  maxReplicas: 15
  metrics:
  # CPU metric
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  # Memory metric
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  # Custom metric
  - type: Pods
    pods:
      metric:
        name: http_requests
      target:
        type: AverageValue
        averageValue: "1000"
```

**Behavior**: HPA calculates replicas for each metric and uses the **highest** value.

## HPA Limitations

1. **No Scale to Zero**: minReplicas must be ≥ 1
2. **Flapping**: Rapid scaling up/down (use stabilization windows)
3. **Metric Delay**: Metrics may lag (15-30s default)
4. **Resource Requests Required**: Pods must define resource requests
5. **Cold Start**: New pods take time to become ready

## Best Practices

1. **Set Appropriate Limits**:
   ```yaml
   minReplicas: 2  # For high availability
   maxReplicas: 50 # Prevent runaway scaling
   ```

2. **Use Multiple Metrics**: CPU + Memory + Custom metrics

3. **Resource Requests**: Always define accurate requests
   ```yaml
   resources:
     requests:
       cpu: 100m
       memory: 128Mi
   ```

4. **Stabilization Windows**: Prevent flapping
   ```yaml
   behavior:
     scaleDown:
       stabilizationWindowSeconds: 300
   ```

5. **Conservative Scale Down**: Fast scale up, slow scale down
   ```yaml
   scaleDown:
     policies:
     - type: Pods
       value: 1
       periodSeconds: 60
   ```

6. **Readiness Probes**: Ensure pods are ready before receiving traffic

7. **Test Under Load**: Validate HPA behavior with realistic load

8. **Monitor**: Track scaling events and metrics

9. **Cost Awareness**: Set reasonable max replicas

10. **PDB**: Use Pod Disruption Budgets with HPA

## Troubleshooting HPA

### Issue 1: HPA Not Scaling

```bash
# Check HPA status
kubectl describe hpa <hpa-name>

# Check if Metrics Server is running
kubectl get deployment metrics-server -n kube-system

# Check if metrics are available
kubectl top pods

# Verify resource requests are set
kubectl describe deployment <deployment-name>
```

### Issue 2: Unknown Metrics

```bash
# Check HPA conditions
kubectl describe hpa <hpa-name>

# Look for error messages like:
# "unable to get metrics for resource cpu"

# Ensure Metrics Server is installed
kubectl get apiservice v1beta1.metrics.k8s.io
```

### Issue 3: Flapping

```bash
# Increase stabilization window
behavior:
  scaleDown:
    stabilizationWindowSeconds: 300

# Adjust target thresholds
# Increase buffer between scale up/down points
```

### Issue 4: Slow Scaling

```bash
# Check metric update interval
kubectl describe hpa <hpa-name>

# Adjust scaling policies
behavior:
  scaleUp:
    policies:
    - type: Percent
      value: 100
      periodSeconds: 30
```

## HPA with VPA

**Vertical Pod Autoscaler (VPA)** adjusts CPU/memory requests/limits.

**Caution**: Don't use HPA and VPA on the same metrics (CPU/memory) for the same deployment.

**Recommended**:
- HPA: Scale on CPU/memory utilization
- VPA: Adjust resource requests for pods

## Advanced Example

Complete production-ready HPA:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: production-app-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: production-app
  minReplicas: 3
  maxReplicas: 50
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second
      target:
        type: AverageValue
        averageValue: "1000"
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Percent
        value: 50
        periodSeconds: 60
      - type: Pods
        value: 2
        periodSeconds: 60
      selectPolicy: Min
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
      - type: Percent
        value: 100
        periodSeconds: 30
      - type: Pods
        value: 4
        periodSeconds: 30
      selectPolicy: Max
```

## Summary

- HPA automatically scales pods based on metrics
- Requires Metrics Server and resource requests
- Supports CPU, memory, custom, and external metrics
- Use stabilization windows to prevent flapping
- Configure behavior for controlled scaling
- Essential for handling variable workloads
- Combine with resource limits and PDBs
- Test thoroughly under realistic load conditions
- Monitor scaling events and adjust thresholds
- Key component of Kubernetes autoscaling strategy
