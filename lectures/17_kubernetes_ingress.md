# Kubernetes Ingress

## What is Ingress?

**Ingress** is a Kubernetes resource that manages external HTTP/HTTPS access to services within a cluster. It provides load balancing, SSL termination, and name-based virtual hosting.

### Why Ingress?

Without Ingress:
- Need LoadBalancer service for each external service (expensive)
- Multiple external IPs to manage
- No centralized SSL/TLS management
- No path-based routing

With Ingress:
- Single entry point for multiple services
- Path and host-based routing
- Centralized SSL/TLS termination
- Cost-effective (single load balancer)

## Ingress Architecture

```
Internet
    ↓
Ingress Controller (nginx, traefik, etc.)
    ↓
Ingress Rules
    ↓
Services → Pods
```

## Components

### 1. Ingress Resource
Defines routing rules (YAML manifest)

### 2. Ingress Controller
Actually implements the Ingress rules (nginx, traefik, HAProxy, etc.)

**Important**: Kubernetes cluster doesn't come with Ingress Controller by default - you must install one!

## Installing Ingress Controller

### NGINX Ingress Controller:

```bash
# Install NGINX Ingress Controller
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.8.1/deploy/static/provider/cloud/deploy.yaml

# For minikube
minikube addons enable ingress

# Verify installation
kubectl get pods -n ingress-nginx
kubectl get svc -n ingress-nginx
```

### Check Ingress Controller:
```bash
kubectl get ingressclass
```

## Basic Ingress Example

### 1. Create Deployments and Services:

```yaml
# App 1: Hello World
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: hello
  template:
    metadata:
      labels:
        app: hello
    spec:
      containers:
      - name: hello
        image: gcr.io/google-samples/hello-app:1.0
        ports:
        - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: hello-service
spec:
  selector:
    app: hello
  ports:
  - port: 8080
    targetPort: 8080
```

### 2. Create Ingress:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: hello-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - host: hello.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: hello-service
            port:
              number: 8080
```

## Path Types

### 1. Prefix
Matches based on URL path prefix split by `/`

```yaml
- path: /app
  pathType: Prefix
```
Matches: `/app`, `/app/`, `/app/page`

### 2. Exact
Exact match only

```yaml
- path: /app
  pathType: Exact
```
Matches: `/app` only (not `/app/` or `/app/page`)

### 3. ImplementationSpecific
Depends on IngressClass

```yaml
- path: /app
  pathType: ImplementationSpecific
```

## Path-Based Routing

Route different paths to different services:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: path-based-ingress
spec:
  ingressClassName: nginx
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 8080
      - path: /web
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
      - path: /admin
        pathType: Prefix
        backend:
          service:
            name: admin-service
            port:
              number: 3000
```

## Host-Based Routing

Route different hosts to different services:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: host-based-ingress
spec:
  ingressClassName: nginx
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 8080
  - host: web.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
  - host: admin.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: admin-service
            port:
              number: 3000
```

## Default Backend

Fallback service for unmatched requests:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-with-default
spec:
  ingressClassName: nginx
  defaultBackend:
    service:
      name: default-service
      port:
        number: 80
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: myapp-service
            port:
              number: 8080
```

## TLS/SSL Configuration

### 1. Create TLS Secret:

```bash
# Generate self-signed certificate (for testing)
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout tls.key -out tls.crt -subj "/CN=myapp.example.com"

# Create secret
kubectl create secret tls myapp-tls --cert=tls.crt --key=tls.key
```

### 2. Configure Ingress with TLS:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tls-ingress
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - myapp.example.com
    secretName: myapp-tls
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: myapp-service
            port:
              number: 8080
```

### Multiple TLS Hosts:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: multi-tls-ingress
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - api.example.com
    secretName: api-tls
  - hosts:
    - web.example.com
    secretName: web-tls
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 8080
  - host: web.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
```

## Ingress Annotations

Annotations configure ingress controller behavior:

### Common NGINX Annotations:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: annotated-ingress
  annotations:
    # Rewrite target
    nginx.ingress.kubernetes.io/rewrite-target: /$2
    
    # SSL Redirect
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    
    # Rate limiting
    nginx.ingress.kubernetes.io/limit-rps: "100"
    
    # CORS
    nginx.ingress.kubernetes.io/enable-cors: "true"
    nginx.ingress.kubernetes.io/cors-allow-methods: "GET, POST, OPTIONS"
    
    # Authentication
    nginx.ingress.kubernetes.io/auth-type: basic
    nginx.ingress.kubernetes.io/auth-secret: basic-auth
    
    # Custom headers
    nginx.ingress.kubernetes.io/configuration-snippet: |
      more_set_headers "X-Custom-Header: value";
    
    # Timeout
    nginx.ingress.kubernetes.io/proxy-connect-timeout: "30"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "60"
    
    # WebSocket support
    nginx.ingress.kubernetes.io/websocket-services: "ws-service"
spec:
  ingressClassName: nginx
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: myapp-service
            port:
              number: 8080
```

## Rewrite Rules

### Example: Remove path prefix

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: rewrite-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$2
spec:
  ingressClassName: nginx
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /api(/|$)(.*)
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 8080
```

**Behavior**:
- Request: `myapp.example.com/api/users`
- Forwarded as: `api-service:8080/users`

## Ingress Management Commands

### Create Ingress:
```bash
kubectl apply -f ingress.yaml
```

### View Ingress:
```bash
# List all ingress
kubectl get ingress
kubectl get ing

# Detailed view
kubectl get ingress -o wide

# Specific namespace
kubectl get ingress -n production
```

### Describe Ingress:
```bash
kubectl describe ingress <ingress-name>
```

### Delete Ingress:
```bash
kubectl delete ingress <ingress-name>
```

### Get Ingress IP/Hostname:
```bash
kubectl get ingress <ingress-name>

# Output shows ADDRESS column
# NAME            CLASS   HOSTS              ADDRESS         PORTS
# hello-ingress   nginx   hello.example.com  192.168.49.2    80
```

## Testing Ingress

### 1. Get Ingress Address:
```bash
kubectl get ingress <ingress-name>
```

### 2. Add to /etc/hosts (for local testing):
```bash
# Get ingress IP
INGRESS_IP=$(kubectl get ingress <ingress-name> -o jsonpath='{.status.loadBalancer.ingress[0].ip}')

# Add to /etc/hosts
echo "$INGRESS_IP myapp.example.com" | sudo tee -a /etc/hosts
```

### 3. Test with curl:
```bash
curl http://myapp.example.com
curl https://myapp.example.com  # For TLS
```

### 4. Minikube Testing:
```bash
# Get minikube IP
minikube ip

# Add to /etc/hosts
echo "$(minikube ip) myapp.example.com" | sudo tee -a /etc/hosts

# Test
curl http://myapp.example.com
```

## Complete Example

### Deploy Application with Ingress:

```yaml
# Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
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
        image: nginx:alpine
        ports:
        - containerPort: 80
---
# Service
apiVersion: v1
kind: Service
metadata:
  name: webapp-service
spec:
  selector:
    app: webapp
  ports:
  - port: 80
    targetPort: 80
---
# Ingress
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: webapp-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - host: webapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: webapp-service
            port:
              number: 80
```

## Ingress Controllers Comparison

| Controller | Features | Use Case |
|------------|----------|----------|
| NGINX | Most popular, feature-rich | General purpose |
| Traefik | Modern, auto-discovery | Cloud-native apps |
| HAProxy | High performance | Enterprise |
| Contour | Built on Envoy | Service mesh integration |
| Kong | API Gateway features | API management |
| AWS ALB | Native AWS integration | AWS EKS |
| GCE | Native GCP integration | GCP GKE |

## Advanced Features

### Canary Deployments:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: canary-ingress
  annotations:
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-weight: "20"
spec:
  ingressClassName: nginx
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: myapp-v2
            port:
              number: 8080
```

### Rate Limiting:

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/limit-rps: "10"
    nginx.ingress.kubernetes.io/limit-connections: "10"
```

### Basic Authentication:

```bash
# Create auth secret
htpasswd -c auth admin
kubectl create secret generic basic-auth --from-file=auth
```

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/auth-type: basic
    nginx.ingress.kubernetes.io/auth-secret: basic-auth
    nginx.ingress.kubernetes.io/auth-realm: 'Authentication Required'
```

## Troubleshooting

### Issue 1: 404 Not Found

```bash
# Check ingress rules
kubectl describe ingress <ingress-name>

# Verify service exists
kubectl get svc <service-name>

# Check service endpoints
kubectl get endpoints <service-name>

# Test service directly
kubectl run test --image=busybox -it --rm -- wget -O- http://<service-name>
```

### Issue 2: 503 Service Unavailable

```bash
# Check if pods are running
kubectl get pods -l app=<label>

# Check pod readiness
kubectl describe pod <pod-name>

# Check service endpoints
kubectl get endpoints <service-name>
```

### Issue 3: TLS Certificate Issues

```bash
# Verify secret exists
kubectl get secret <tls-secret-name>

# Check certificate details
kubectl get secret <tls-secret-name> -o yaml

# Verify certificate
kubectl get secret <tls-secret-name> -o jsonpath='{.data.tls\.crt}' | base64 -d | openssl x509 -text -noout
```

### Issue 4: Ingress Controller Not Running

```bash
# Check ingress controller pods
kubectl get pods -n ingress-nginx

# Check logs
kubectl logs -n ingress-nginx <ingress-controller-pod>

# Restart ingress controller
kubectl rollout restart deployment ingress-nginx-controller -n ingress-nginx
```

## Best Practices

1. **Use TLS**: Always enable HTTPS for production
2. **One Ingress per Application**: Easier to manage
3. **Meaningful Names**: Use descriptive ingress names
4. **Rate Limiting**: Protect against abuse
5. **Timeouts**: Configure appropriate timeouts
6. **Health Checks**: Ensure backend services are healthy
7. **Monitoring**: Monitor ingress metrics
8. **Resource Limits**: Set limits on ingress controller
9. **Security**: Use authentication and authorization
10. **Testing**: Test ingress rules before production

## Summary

- Ingress manages external HTTP/HTTPS access to services
- Requires Ingress Controller to be installed
- Provides path-based and host-based routing
- Supports TLS/SSL termination
- Single entry point for multiple services
- Cost-effective compared to multiple LoadBalancers
- Annotations configure controller-specific behavior
- Essential for exposing applications in Kubernetes
- Supports advanced features like canary deployments and rate limiting
- Choose appropriate ingress controller for your needs
