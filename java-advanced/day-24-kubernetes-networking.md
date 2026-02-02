# Day 24: Kubernetes Networking

## Mục tiêu học tập
- Hiểu Kubernetes networking model
- Cấu hình Services: ClusterIP, NodePort, LoadBalancer
- Implement Ingress controllers và routing
- Sử dụng Network Policies cho security
- Service mesh basics với Istio

## 1. Kubernetes Networking Model

### 1.1 Network Architecture

```
Kubernetes Networking Overview
══════════════════════════════

┌─────────────────────────────────────────────────────────────────────┐
│                        EXTERNAL TRAFFIC                              │
│                              │                                       │
│                              ▼                                       │
│                    ┌─────────────────┐                              │
│                    │  Load Balancer  │                              │
│                    │   (External)    │                              │
│                    └────────┬────────┘                              │
│                              │                                       │
│                              ▼                                       │
│  ┌───────────────────────────────────────────────────────────────┐ │
│  │                    INGRESS CONTROLLER                          │ │
│  │  ┌─────────────────────────────────────────────────────────┐  │ │
│  │  │  /api/orders → order-service:80                         │  │ │
│  │  │  /api/payments → payment-service:80                     │  │ │
│  │  │  /api/users → user-service:80                           │  │ │
│  │  └─────────────────────────────────────────────────────────┘  │ │
│  └───────────────────────────────────────────────────────────────┘ │
│                              │                                       │
│                              ▼                                       │
│  ┌───────────────────────────────────────────────────────────────┐ │
│  │                      SERVICES (ClusterIP)                      │ │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐           │ │
│  │  │order-service│  │payment-svc  │  │ user-service│           │ │
│  │  │10.96.0.10   │  │10.96.0.11   │  │ 10.96.0.12  │           │ │
│  │  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘           │ │
│  └─────────│────────────────│────────────────│───────────────────┘ │
│            │                │                │                      │
│            ▼                ▼                ▼                      │
│  ┌───────────────────────────────────────────────────────────────┐ │
│  │                         PODS                                   │ │
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐         │ │
│  │  │ Pod 1   │  │ Pod 2   │  │ Pod 3   │  │ Pod 4   │         │ │
│  │  │10.1.1.1 │  │10.1.1.2 │  │10.1.2.1 │  │10.1.2.2 │         │ │
│  │  └─────────┘  └─────────┘  └─────────┘  └─────────┘         │ │
│  │       Node 1                    Node 2                        │ │
│  └───────────────────────────────────────────────────────────────┘ │
│                                                                      │
│  Network Requirements:                                               │
│  1. All pods can communicate without NAT                            │
│  2. All nodes can communicate with all pods                         │
│  3. The IP a pod sees itself as is the same IP others see it as    │
└─────────────────────────────────────────────────────────────────────┘
```

### 1.2 DNS Resolution

```
Kubernetes DNS
══════════════

Service DNS Names:
  <service-name>.<namespace>.svc.cluster.local

Examples:
  order-service.production.svc.cluster.local
  mysql.database.svc.cluster.local
  redis-master.cache.svc.cluster.local

Short Names (within same namespace):
  order-service
  mysql
  redis-master

Pod DNS Names (Headless Service):
  <pod-name>.<service-name>.<namespace>.svc.cluster.local
  order-service-0.order-service-headless.production.svc.cluster.local

DNS Resolution Flow:
┌────────────┐     DNS Query      ┌─────────────┐
│    Pod     │ ──────────────────▶│  CoreDNS    │
│            │                     │             │
│            │ ◀────────────────── │  10.96.0.10 │
│            │     IP Address      └─────────────┘
└────────────┘
```

## 2. Service Types

### 2.1 ClusterIP Service

```yaml
# ClusterIP (Default) - Internal access only
apiVersion: v1
kind: Service
metadata:
  name: order-service
  namespace: production
  labels:
    app: order-service
spec:
  type: ClusterIP
  selector:
    app: order-service
  ports:
    - name: http
      protocol: TCP
      port: 80          # Service port
      targetPort: 8080  # Container port
    - name: grpc
      protocol: TCP
      port: 9090
      targetPort: 9090
---
# Headless Service (for StatefulSets or direct pod access)
apiVersion: v1
kind: Service
metadata:
  name: order-service-headless
  namespace: production
spec:
  type: ClusterIP
  clusterIP: None  # Headless - returns pod IPs directly
  selector:
    app: order-service
  ports:
    - name: http
      port: 8080
```

### 2.2 NodePort Service

```yaml
# NodePort - Exposes on each node's IP
apiVersion: v1
kind: Service
metadata:
  name: order-service-nodeport
  namespace: production
spec:
  type: NodePort
  selector:
    app: order-service
  ports:
    - name: http
      protocol: TCP
      port: 80
      targetPort: 8080
      nodePort: 30080  # Range: 30000-32767, auto-assigned if not specified
```

### 2.3 LoadBalancer Service

```yaml
# LoadBalancer - External access via cloud provider LB
apiVersion: v1
kind: Service
metadata:
  name: order-service-lb
  namespace: production
  annotations:
    # AWS-specific annotations
    service.beta.kubernetes.io/aws-load-balancer-type: nlb
    service.beta.kubernetes.io/aws-load-balancer-internal: "false"
    service.beta.kubernetes.io/aws-load-balancer-cross-zone-load-balancing-enabled: "true"
spec:
  type: LoadBalancer
  selector:
    app: order-service
  ports:
    - name: http
      protocol: TCP
      port: 80
      targetPort: 8080
  # Optional: restrict source IPs
  loadBalancerSourceRanges:
    - 10.0.0.0/8
    - 192.168.0.0/16
```

### 2.4 ExternalName Service

```yaml
# ExternalName - Alias for external service
apiVersion: v1
kind: Service
metadata:
  name: external-database
  namespace: production
spec:
  type: ExternalName
  externalName: db.external-provider.com
  # No selector - points to external DNS name
---
# Endpoints for external IP addresses
apiVersion: v1
kind: Service
metadata:
  name: external-api
  namespace: production
spec:
  type: ClusterIP
  ports:
    - port: 443
      targetPort: 443
---
apiVersion: v1
kind: Endpoints
metadata:
  name: external-api  # Must match service name
  namespace: production
subsets:
  - addresses:
      - ip: 203.0.113.10
      - ip: 203.0.113.11
    ports:
      - port: 443
```

## 3. Ingress Configuration

### 3.1 Basic Ingress

```yaml
# ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-ingress
  namespace: production
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  tls:
    - hosts:
        - api.example.com
      secretName: api-tls-secret
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /orders
            pathType: Prefix
            backend:
              service:
                name: order-service
                port:
                  number: 80
          - path: /payments
            pathType: Prefix
            backend:
              service:
                name: payment-service
                port:
                  number: 80
          - path: /users
            pathType: Prefix
            backend:
              service:
                name: user-service
                port:
                  number: 80
```

### 3.2 Advanced Ingress with Annotations

```yaml
# advanced-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-ingress-advanced
  namespace: production
  annotations:
    # Ingress class
    kubernetes.io/ingress.class: nginx

    # SSL/TLS
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
    cert-manager.io/cluster-issuer: "letsencrypt-prod"

    # Proxy settings
    nginx.ingress.kubernetes.io/proxy-body-size: "50m"
    nginx.ingress.kubernetes.io/proxy-connect-timeout: "30"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "300"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "300"
    nginx.ingress.kubernetes.io/proxy-buffer-size: "8k"

    # Rate limiting
    nginx.ingress.kubernetes.io/limit-connections: "10"
    nginx.ingress.kubernetes.io/limit-rps: "100"
    nginx.ingress.kubernetes.io/limit-rpm: "3000"

    # CORS
    nginx.ingress.kubernetes.io/enable-cors: "true"
    nginx.ingress.kubernetes.io/cors-allow-origin: "https://app.example.com"
    nginx.ingress.kubernetes.io/cors-allow-methods: "GET, POST, PUT, DELETE, OPTIONS"
    nginx.ingress.kubernetes.io/cors-allow-headers: "DNT,User-Agent,X-Requested-With,If-Modified-Since,Cache-Control,Content-Type,Range,Authorization"

    # Security headers
    nginx.ingress.kubernetes.io/configuration-snippet: |
      add_header X-Frame-Options "SAMEORIGIN" always;
      add_header X-Content-Type-Options "nosniff" always;
      add_header X-XSS-Protection "1; mode=block" always;
      add_header Referrer-Policy "strict-origin-when-cross-origin" always;

    # WebSocket support
    nginx.ingress.kubernetes.io/proxy-http-version: "1.1"
    nginx.ingress.kubernetes.io/upstream-hash-by: "$request_uri"

    # Session affinity (sticky sessions)
    nginx.ingress.kubernetes.io/affinity: "cookie"
    nginx.ingress.kubernetes.io/session-cookie-name: "route"
    nginx.ingress.kubernetes.io/session-cookie-expires: "172800"
    nginx.ingress.kubernetes.io/session-cookie-max-age: "172800"

spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - api.example.com
        - admin.example.com
      secretName: api-tls-secret
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: api-gateway
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
                  number: 80
```

### 3.3 Path-Based Routing with Rewrite

```yaml
# rewrite-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-rewrite-ingress
  namespace: production
  annotations:
    nginx.ingress.kubernetes.io/use-regex: "true"
    nginx.ingress.kubernetes.io/rewrite-target: /$2
spec:
  rules:
    - host: api.example.com
      http:
        paths:
          # /api/v1/orders/123 → /123 on order-service
          - path: /api/v1/orders(/|$)(.*)
            pathType: Prefix
            backend:
              service:
                name: order-service
                port:
                  number: 80
          # /api/v1/payments/xyz → /xyz on payment-service
          - path: /api/v1/payments(/|$)(.*)
            pathType: Prefix
            backend:
              service:
                name: payment-service
                port:
                  number: 80
```

## 4. Network Policies

### 4.1 Default Deny All

```yaml
# default-deny-all.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production
spec:
  podSelector: {}  # Applies to all pods
  policyTypes:
    - Ingress
    - Egress
  # No ingress/egress rules = deny all
```

### 4.2 Allow Specific Traffic

```yaml
# order-service-network-policy.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: order-service-policy
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: order-service
  policyTypes:
    - Ingress
    - Egress

  ingress:
    # Allow from API Gateway
    - from:
        - podSelector:
            matchLabels:
              app: api-gateway
      ports:
        - protocol: TCP
          port: 8080

    # Allow from same namespace monitoring
    - from:
        - namespaceSelector:
            matchLabels:
              name: monitoring
          podSelector:
            matchLabels:
              app: prometheus
      ports:
        - protocol: TCP
          port: 8080

  egress:
    # Allow to database
    - to:
        - podSelector:
            matchLabels:
              app: mysql
      ports:
        - protocol: TCP
          port: 3306

    # Allow to Redis
    - to:
        - podSelector:
            matchLabels:
              app: redis
      ports:
        - protocol: TCP
          port: 6379

    # Allow to RabbitMQ
    - to:
        - podSelector:
            matchLabels:
              app: rabbitmq
      ports:
        - protocol: TCP
          port: 5672

    # Allow DNS
    - to:
        - namespaceSelector: {}
          podSelector:
            matchLabels:
              k8s-app: kube-dns
      ports:
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53

    # Allow external HTTPS (for external APIs)
    - to:
        - ipBlock:
            cidr: 0.0.0.0/0
            except:
              - 10.0.0.0/8
              - 172.16.0.0/12
              - 192.168.0.0/16
      ports:
        - protocol: TCP
          port: 443
```

### 4.3 Namespace Isolation

```yaml
# namespace-isolation.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: namespace-isolation
  namespace: production
spec:
  podSelector: {}
  policyTypes:
    - Ingress
  ingress:
    # Only allow from same namespace
    - from:
        - podSelector: {}
    # Allow from ingress controller namespace
    - from:
        - namespaceSelector:
            matchLabels:
              name: ingress-nginx
```

## 5. Service Mesh with Istio

### 5.1 Istio Architecture

```
Istio Service Mesh Architecture
═══════════════════════════════

┌─────────────────────────────────────────────────────────────────┐
│                       CONTROL PLANE                              │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                         istiod                           │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐     │   │
│  │  │   Pilot     │  │   Citadel   │  │   Galley    │     │   │
│  │  │ (Discovery) │  │ (Security)  │  │  (Config)   │     │   │
│  │  └─────────────┘  └─────────────┘  └─────────────┘     │   │
│  └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
                              │
                              │ Configuration
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                        DATA PLANE                                │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │                         Pod                                │ │
│  │  ┌─────────────┐     ┌─────────────────────────────────┐ │ │
│  │  │    App      │◀───▶│     Envoy Sidecar Proxy         │ │ │
│  │  │  Container  │     │  - Traffic routing               │ │ │
│  │  │             │     │  - Load balancing                │ │ │
│  │  │             │     │  - mTLS encryption               │ │ │
│  │  │             │     │  - Observability                 │ │ │
│  │  └─────────────┘     └─────────────────────────────────┘ │ │
│  └───────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

### 5.2 Istio Virtual Service

```yaml
# virtual-service.yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: order-service
  namespace: production
spec:
  hosts:
    - order-service
  http:
    # Route based on headers
    - match:
        - headers:
            x-version:
              exact: "v2"
      route:
        - destination:
            host: order-service
            subset: v2

    # Canary routing
    - route:
        - destination:
            host: order-service
            subset: v1
          weight: 90
        - destination:
            host: order-service
            subset: v2
          weight: 10

    # Retry policy
    retries:
      attempts: 3
      perTryTimeout: 2s
      retryOn: gateway-error,connect-failure,refused-stream

    # Timeout
    timeout: 30s
---
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: order-service
  namespace: production
spec:
  host: order-service
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 100
      http:
        h2UpgradePolicy: UPGRADE
        http1MaxPendingRequests: 100
        http2MaxRequests: 1000
    loadBalancer:
      simple: ROUND_ROBIN
    outlierDetection:
      consecutive5xxErrors: 5
      interval: 30s
      baseEjectionTime: 30s
      maxEjectionPercent: 50
  subsets:
    - name: v1
      labels:
        version: v1
    - name: v2
      labels:
        version: v2
```

### 5.3 Istio Gateway

```yaml
# istio-gateway.yaml
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: api-gateway
  namespace: production
spec:
  selector:
    istio: ingressgateway
  servers:
    - port:
        number: 80
        name: http
        protocol: HTTP
      hosts:
        - api.example.com
      tls:
        httpsRedirect: true
    - port:
        number: 443
        name: https
        protocol: HTTPS
      hosts:
        - api.example.com
      tls:
        mode: SIMPLE
        credentialName: api-tls-secret
---
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: api-routes
  namespace: production
spec:
  hosts:
    - api.example.com
  gateways:
    - api-gateway
  http:
    - match:
        - uri:
            prefix: /api/orders
      route:
        - destination:
            host: order-service
            port:
              number: 80
    - match:
        - uri:
            prefix: /api/payments
      route:
        - destination:
            host: payment-service
            port:
              number: 80
```

### 5.4 mTLS Configuration

```yaml
# peer-authentication.yaml
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: production
spec:
  mtls:
    mode: STRICT  # or PERMISSIVE for gradual migration
---
# authorization-policy.yaml
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: order-service-policy
  namespace: production
spec:
  selector:
    matchLabels:
      app: order-service
  action: ALLOW
  rules:
    - from:
        - source:
            principals:
              - cluster.local/ns/production/sa/api-gateway
              - cluster.local/ns/production/sa/admin-service
      to:
        - operation:
            methods: ["GET", "POST", "PUT", "DELETE"]
            paths: ["/api/*"]
```

## 6. Debugging Network Issues

### 6.1 Debugging Commands

```bash
# DNS resolution test
kubectl run -it --rm debug --image=busybox --restart=Never -- nslookup order-service

# Network connectivity test
kubectl run -it --rm debug --image=busybox --restart=Never -- wget -qO- http://order-service:80/actuator/health

# Check service endpoints
kubectl get endpoints order-service -n production

# View service details
kubectl describe service order-service -n production

# Check pod networking
kubectl exec -it order-service-xxx -- ip addr
kubectl exec -it order-service-xxx -- netstat -tlnp

# Check network policies
kubectl get networkpolicies -n production
kubectl describe networkpolicy order-service-policy -n production

# Port forwarding for testing
kubectl port-forward svc/order-service 8080:80 -n production

# Check ingress
kubectl get ingress -n production
kubectl describe ingress api-ingress -n production

# Check ingress controller logs
kubectl logs -n ingress-nginx -l app.kubernetes.io/name=ingress-nginx

# Istio debugging
istioctl analyze -n production
istioctl proxy-config cluster order-service-xxx -n production
istioctl proxy-config routes order-service-xxx -n production
```

### 6.2 Network Debug Pod

```yaml
# network-debug-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: network-debug
  namespace: production
spec:
  containers:
    - name: debug
      image: nicolaka/netshoot
      command: ["sleep", "3600"]
      securityContext:
        capabilities:
          add: ["NET_ADMIN", "NET_RAW"]
```

```bash
# Use debug pod
kubectl exec -it network-debug -n production -- bash

# Inside debug pod:
# DNS lookup
nslookup order-service.production.svc.cluster.local

# HTTP request
curl -v http://order-service:80/actuator/health

# TCP connectivity
nc -zv order-service 80

# Trace route
traceroute order-service

# tcpdump (if needed)
tcpdump -i any port 80
```

## 7. Hands-on Exercises

### Exercise 1: Service Configuration
Create services for a microservices application.

```bash
# Requirements:
# 1. ClusterIP for internal services
# 2. Headless service for StatefulSet
# 3. Test DNS resolution between services
# 4. Configure readiness probes
```

### Exercise 2: Ingress Setup
Configure ingress with TLS and multiple services.

```bash
# Requirements:
# 1. Install nginx ingress controller
# 2. Create TLS secret
# 3. Configure path-based routing
# 4. Add rate limiting
# 5. Enable CORS
```

### Exercise 3: Network Policies
Implement network segmentation.

```bash
# Requirements:
# 1. Default deny all in namespace
# 2. Allow specific pod-to-pod traffic
# 3. Allow database access only from app pods
# 4. Allow DNS traffic
# 5. Test and verify policies
```

### Exercise 4: Service Mesh Basics
Set up Istio traffic management.

```bash
# Requirements:
# 1. Install Istio
# 2. Enable sidecar injection
# 3. Configure VirtualService for canary
# 4. Set up mTLS
# 5. Create authorization policies
```

## 8. Key Takeaways

### Service Types
1. **ClusterIP**: Internal communication, default type
2. **NodePort**: External access via node ports
3. **LoadBalancer**: Cloud provider load balancer
4. **ExternalName**: DNS alias for external services

### Ingress Best Practices
1. **TLS Termination**: Always use HTTPS in production
2. **Rate Limiting**: Protect against abuse
3. **Health Checks**: Configure backend health checks
4. **Path Rewriting**: Handle API versioning

### Network Policies
1. **Default Deny**: Start with deny-all policy
2. **Least Privilege**: Allow only necessary traffic
3. **Namespace Isolation**: Separate environments
4. **DNS Access**: Always allow DNS queries

### Service Mesh Benefits
1. **Observability**: Distributed tracing, metrics
2. **Security**: mTLS, authorization policies
3. **Traffic Management**: Canary, circuit breaker
4. **Resilience**: Retries, timeouts

---

## Navigation

[← Day 23: Kubernetes Deployments](./day-23-kubernetes-deployments.md) | [Day 25: Kubernetes Storage →](./day-25-kubernetes-storage.md)

[Back to Overview](./00-overview.md)
