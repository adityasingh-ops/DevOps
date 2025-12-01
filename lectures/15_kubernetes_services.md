# Kubernetes Services

## What is a Service?

A **Kubernetes Service** is an abstraction that defines a logical set of Pods and a policy to access them. Services enable loose coupling between dependent Pods and provide stable networking for dynamic Pod environments.

### Why Services are Needed:
- Pods are ephemeral and can be replaced at any time
- Pod IPs change when they are recreated
- Multiple Pods need a single access point
- Load balancing across multiple Pods
- Service discovery within the cluster

## Service Types

Kubernetes provides four main types of Services:

### 1. ClusterIP (Default)
- Exposes service on cluster-internal IP
- Only accessible within the cluster
- Used for inter-service communication

### 2. NodePort
- Exposes service on each Node's IP at a static port
- Accessible from outside the cluster via `<NodeIP>:<NodePort>`
- Port range: 30000-32767

### 3. LoadBalancer
- Creates an external load balancer (cloud provider specific)
- Assigns an external IP to the service
- Used for production external access

### 4. ExternalName
- Maps service to DNS name
- Returns CNAME record with the external name
- No proxy or load balancing

## Basic Service YAML

### ClusterIP Service:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  type: ClusterIP
  selector:
    app: myapp
  ports:
  - protocol: TCP
    port: 80          # Service port
    targetPort: 8080  # Container port
```

## Service Discovery

Kubernetes provides two methods for service discovery:

### 1. Environment Variables
When a Pod starts, Kubernetes injects environment variables:
```bash
MY_SERVICE_SERVICE_HOST=10.96.0.1
MY_SERVICE_SERVICE_PORT=80
```

### 2. DNS (Recommended)
Services get a DNS name automatically:
```
<service-name>.<namespace>.svc.cluster.local
```

Example:
```bash
# Within same namespace
curl http://my-service

# From different namespace
curl http://my-service.default.svc.cluster.local
```

## ClusterIP Service

**Default service type** - Only accessible within the cluster.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-service
spec:
  type: ClusterIP
  selector:
    app: backend
  ports:
  - name: http
    protocol: TCP
    port: 80
    targetPort: 8080
```

### Use Cases:
- Internal microservices communication
- Database services
- Backend APIs not exposed externally

### Test ClusterIP:
```bash
# Create service
kubectl apply -f clusterip-service.yaml

# Get service details
kubectl get svc backend-service

# Test from within cluster
kubectl run test-pod --image=busybox -it --rm -- wget -O- http://backend-service
```

## NodePort Service

Exposes service on each Node's IP at a static port.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nodeport-service
spec:
  type: NodePort
  selector:
    app: webapp
  ports:
  - protocol: TCP
    port: 80          # Service port
    targetPort: 8080  # Container port
    nodePort: 30080   # External port (optional, auto-assigned if not specified)
```

### Access Methods:
```bash
# Using Node IP
curl http://<NodeIP>:30080

# Using minikube
minikube service nodeport-service --url
```

### Use Cases:
- Development and testing
- When LoadBalancer is not available
- Direct access to services

## LoadBalancer Service

Creates an external load balancer (requires cloud provider support).

```yaml
apiVersion: v1
kind: Service
metadata:
  name: loadbalancer-service
spec:
  type: LoadBalancer
  selector:
    app: webapp
  ports:
  - protocol: TCP
    port: 80
    targetPort: 8080
```

### Cloud Provider Behavior:
- **AWS**: Creates ELB (Elastic Load Balancer)
- **GCP**: Creates Cloud Load Balancer
- **Azure**: Creates Azure Load Balancer
- **Minikube**: Use `minikube tunnel` to simulate

### Check External IP:
```bash
kubectl get svc loadbalancer-service

# Output shows EXTERNAL-IP
# NAME                    TYPE           EXTERNAL-IP    PORT(S)
# loadbalancer-service    LoadBalancer   35.123.45.67   80:31234/TCP
```

## ExternalName Service

Maps service to external DNS name.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: external-database
spec:
  type: ExternalName
  externalName: database.example.com
```

### Use Case:
```bash
# Pods can access external service using internal name
mysql -h external-database -u user -p
# This resolves to database.example.com
```

## Multi-Port Services

Services can expose multiple ports:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: multi-port-service
spec:
  selector:
    app: myapp
  ports:
  - name: http
    protocol: TCP
    port: 80
    targetPort: 8080
  - name: https
    protocol: TCP
    port: 443
    targetPort: 8443
  - name: metrics
    protocol: TCP
    port: 9090
    targetPort: 9090
```

## Headless Service

Service without cluster IP - used for direct Pod access.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: headless-service
spec:
  clusterIP: None  # This makes it headless
  selector:
    app: myapp
  ports:
  - protocol: TCP
    port: 80
    targetPort: 8080
```

### Use Cases:
- StatefulSets
- Direct Pod-to-Pod communication
- Client-side load balancing
- Service discovery without proxying

### DNS Returns:
- Normal Service: Single IP address
- Headless Service: All Pod IPs

## Service with Session Affinity

Stick client to same Pod:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: sticky-service
spec:
  selector:
    app: myapp
  sessionAffinity: ClientIP
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 10800  # 3 hours
  ports:
  - protocol: TCP
    port: 80
    targetPort: 8080
```

## Complete Example: Deployment + Service

### Deployment:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
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

### ClusterIP Service:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  type: ClusterIP
  selector:
    app: nginx
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
```

### NodePort Service:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-nodeport
spec:
  type: NodePort
  selector:
    app: nginx
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
    nodePort: 30080
```

## Service Management Commands

### Create Service:
```bash
# From YAML
kubectl apply -f service.yaml

# Imperative (ClusterIP)
kubectl expose deployment nginx-deployment --port=80 --target-port=8080

# Imperative (NodePort)
kubectl expose deployment nginx-deployment --type=NodePort --port=80
```

### View Services:
```bash
# List all services
kubectl get services
kubectl get svc

# Detailed view
kubectl get svc -o wide

# Specific service
kubectl get svc <service-name>

# With labels
kubectl get svc -l app=nginx
```

### Describe Service:
```bash
kubectl describe svc <service-name>
```

### Get Endpoints:
```bash
# View service endpoints (Pod IPs)
kubectl get endpoints <service-name>
kubectl get ep <service-name>
```

### Delete Service:
```bash
kubectl delete svc <service-name>
```

## Service Selectors

Services use label selectors to identify Pods:

```yaml
# Service
spec:
  selector:
    app: myapp
    tier: backend

# Pod must have matching labels
metadata:
  labels:
    app: myapp
    tier: backend
```

## Service Without Selector

For services that point to external resources or specific endpoints:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: external-service
spec:
  ports:
  - protocol: TCP
    port: 80
    targetPort: 8080
---
apiVersion: v1
kind: Endpoints
metadata:
  name: external-service  # Must match service name
subsets:
- addresses:
  - ip: 192.168.1.100
  - ip: 192.168.1.101
  ports:
  - port: 8080
```

## Troubleshooting Services

### Issue 1: Service Not Accessible
```bash
# Check service exists
kubectl get svc <service-name>

# Check endpoints (should list Pod IPs)
kubectl get endpoints <service-name>

# Verify Pod labels match selector
kubectl get pods --show-labels
kubectl describe svc <service-name>
```

### Issue 2: No Endpoints
```bash
# Verify Pods are running
kubectl get pods -l app=<label>

# Check Pod labels
kubectl get pods --show-labels

# Verify selector in service
kubectl describe svc <service-name>
```

### Issue 3: Connection Timeout
```bash
# Check if Pods are ready
kubectl get pods

# Check Pod logs
kubectl logs <pod-name>

# Test from within cluster
kubectl run test --image=busybox -it --rm -- wget -O- http://<service-name>
```

### Issue 4: Wrong Port
```bash
# Verify port configuration
kubectl describe svc <service-name>

# Check container port
kubectl describe pod <pod-name>
```

## Best Practices

1. **Use ClusterIP by Default**: For internal services
2. **Meaningful Names**: Use descriptive service names
3. **Label Consistency**: Ensure Pod labels match service selectors
4. **DNS Names**: Use service DNS names, not IPs
5. **Health Checks**: Ensure Pods have readiness probes
6. **Multiple Ports**: Name ports when exposing multiple
7. **LoadBalancer Cost**: Be aware of cloud provider costs
8. **Headless for StatefulSets**: Use headless services with StatefulSets
9. **Session Affinity**: Use only when necessary
10. **Network Policies**: Combine with NetworkPolicies for security

## Service Types Comparison

| Feature | ClusterIP | NodePort | LoadBalancer | ExternalName |
|---------|-----------|----------|--------------|--------------|
| Internal Access | ✅ Yes | ✅ Yes | ✅ Yes | ✅ Yes |
| External Access | ❌ No | ✅ Yes | ✅ Yes | N/A |
| Load Balancing | ✅ Yes | ✅ Yes | ✅ Yes | ❌ No |
| Cloud Provider | ❌ Not required | ❌ Not required | ✅ Required | ❌ Not required |
| Port Range | Any | 30000-32767 | Any | N/A |
| Cost | Free | Free | 💰 Paid | Free |

## Hands-On Exercise

```bash
# 1. Create deployment
kubectl create deployment web --image=nginx --replicas=3

# 2. Expose as ClusterIP
kubectl expose deployment web --port=80 --type=ClusterIP

# 3. Test from within cluster
kubectl run test --image=busybox -it --rm -- wget -O- http://web

# 4. Expose as NodePort
kubectl expose deployment web --port=80 --type=NodePort --name=web-nodeport

# 5. Get NodePort
kubectl get svc web-nodeport

# 6. Access via NodePort (if using minikube)
minikube service web-nodeport --url

# 7. Check endpoints
kubectl get endpoints web

# 8. Scale deployment and watch endpoints
kubectl scale deployment web --replicas=5
kubectl get endpoints web --watch

# 9. Clean up
kubectl delete svc web web-nodeport
kubectl delete deployment web
```

## Summary

- Services provide stable networking for dynamic Pods
- ClusterIP: Internal cluster access (default)
- NodePort: External access via Node IP and port
- LoadBalancer: External access via cloud load balancer
- ExternalName: Maps to external DNS
- Services use label selectors to identify Pods
- DNS-based service discovery is recommended
- Endpoints show which Pods back the service
- Essential for microservices architecture in Kubernetes
