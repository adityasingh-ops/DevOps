# Kubernetes Pods

## What is a Pod?

A **Pod** is the smallest and simplest unit in the Kubernetes object model. It represents a single instance of a running process in your cluster.

### Key Characteristics:
- A Pod can contain one or more containers
- Containers in a Pod share the same network namespace (IP address and port space)
- Containers in a Pod can communicate via localhost
- Pods are ephemeral - they can be created, destroyed, and recreated

## Pod Lifecycle

Pods go through several phases:

1. **Pending**: Pod accepted by Kubernetes but container(s) not yet created
2. **Running**: Pod bound to a node and all containers created
3. **Succeeded**: All containers terminated successfully
4. **Failed**: All containers terminated, at least one failed
5. **Unknown**: State of the Pod cannot be determined

## Creating a Simple Pod

### Basic Pod YAML Structure:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  labels:
    app: nginx
spec:
  containers:
  - name: nginx
    image: nginx:latest
    ports:
    - containerPort: 80
```

### Create the Pod:

```bash
kubectl apply -f pod.yaml
```

## Multi-Container Pod Example

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: multi-container-pod
spec:
  containers:
  - name: nginx
    image: nginx:latest
    ports:
    - containerPort: 80
  - name: sidecar
    image: busybox
    command: ['sh', '-c', 'while true; do echo "Sidecar running"; sleep 10; done']
```

## Essential Pod Commands

### View Pods:
```bash
# List all pods
kubectl get pods

# List pods with more details
kubectl get pods -o wide

# List pods in all namespaces
kubectl get pods --all-namespaces
```

### Describe Pod:
```bash
kubectl describe pod <pod-name>
```

### Pod Logs:
```bash
# View logs of a pod
kubectl logs <pod-name>

# View logs of specific container in multi-container pod
kubectl logs <pod-name> -c <container-name>

# Follow logs (stream)
kubectl logs -f <pod-name>
```

### Execute Commands in Pod:
```bash
# Execute command in pod
kubectl exec <pod-name> -- <command>

# Interactive shell
kubectl exec -it <pod-name> -- /bin/bash
```

### Delete Pod:
```bash
kubectl delete pod <pod-name>
```

## Pod Configuration Options

### Resource Limits and Requests:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: resource-pod
spec:
  containers:
  - name: nginx
    image: nginx
    resources:
      requests:
        memory: "64Mi"
        cpu: "250m"
      limits:
        memory: "128Mi"
        cpu: "500m"
```

### Environment Variables:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: env-pod
spec:
  containers:
  - name: nginx
    image: nginx
    env:
    - name: ENV_NAME
      value: "production"
    - name: DB_HOST
      value: "mysql.example.com"
```

### Volume Mounts:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: volume-pod
spec:
  containers:
  - name: nginx
    image: nginx
    volumeMounts:
    - name: data
      mountPath: /usr/share/nginx/html
  volumes:
  - name: data
    emptyDir: {}
```

## Init Containers

Init containers run before app containers and are used for setup tasks:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: init-pod
spec:
  initContainers:
  - name: init-service
    image: busybox
    command: ['sh', '-c', 'echo "Initializing..."; sleep 5']
  containers:
  - name: main-app
    image: nginx
```

## Pod Networking

- Each Pod gets its own IP address
- Containers within a Pod share the same network namespace
- Containers can communicate with each other via localhost
- Pods can communicate with other Pods directly using Pod IP

## Liveness and Readiness Probes

### Liveness Probe:
Checks if container is alive. If it fails, container is restarted.

```yaml
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 3
  periodSeconds: 3
```

### Readiness Probe:
Checks if container is ready to accept traffic.

```yaml
readinessProbe:
  httpGet:
    path: /ready
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 5
```

## Best Practices

1. **One Process Per Container**: Each container should have a single responsibility
2. **Use Labels**: Always label your pods for organization
3. **Set Resource Limits**: Define CPU and memory limits to prevent resource hogging
4. **Health Checks**: Always implement liveness and readiness probes
5. **Use Declarative Configuration**: Use YAML files instead of imperative commands
6. **Don't Use Naked Pods**: Use higher-level controllers like Deployments

## Common Pod Patterns

### Sidecar Pattern:
Helper container extends functionality of main container

### Ambassador Pattern:
Proxy container that simplifies networking for main container

### Adapter Pattern:
Container that standardizes output from main container

## Troubleshooting

```bash
# Check pod status
kubectl get pod <pod-name>

# Detailed information
kubectl describe pod <pod-name>

# View events
kubectl get events --sort-by=.metadata.creationTimestamp

# Check logs
kubectl logs <pod-name>

# Interactive debugging
kubectl exec -it <pod-name> -- /bin/bash
```

## Summary

- Pods are the fundamental building blocks in Kubernetes
- They can contain one or more tightly coupled containers
- Pods share network and storage resources
- They are ephemeral and should be managed by higher-level controllers
- Understanding Pods is essential for working with Kubernetes
