# Day 22: Kubernetes Basics

## Mục tiêu học tập
- Hiểu Kubernetes architecture và core concepts
- Deploy Java applications lên Kubernetes
- Cấu hình Services, ConfigMaps, và Secrets
- Implement health checks và resource management
- Sử dụng kubectl để quản lý cluster

## 1. Kubernetes Architecture

### 1.1 Cluster Overview

```
Kubernetes Architecture
═══════════════════════

┌─────────────────────────────────────────────────────────────────────────┐
│                           CONTROL PLANE                                  │
├─────────────────────────────────────────────────────────────────────────┤
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐   │
│  │ API Server  │  │   etcd      │  │ Scheduler   │  │ Controller  │   │
│  │             │  │             │  │             │  │  Manager    │   │
│  │ - REST API  │  │ - Key-Value │  │ - Pod       │  │ - Node      │   │
│  │ - AuthN/Z   │  │ - Cluster   │  │   placement │  │ - Replica   │   │
│  │ - Admission │  │   state     │  │ - Resource  │  │ - Endpoint  │   │
│  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘   │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    │ Network
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                           WORKER NODES                                   │
├─────────────────────────────────────────────────────────────────────────┤
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                          NODE 1                                  │   │
│  │  ┌─────────┐  ┌─────────────────────────────────────────────┐  │   │
│  │  │ kubelet │  │                   PODS                       │  │   │
│  │  │         │  │  ┌─────────┐  ┌─────────┐  ┌─────────┐     │  │   │
│  │  │ - Pod   │  │  │  Pod 1  │  │  Pod 2  │  │  Pod 3  │     │  │   │
│  │  │   mgmt  │  │  │┌───────┐│  │┌───────┐│  │┌───────┐│     │  │   │
│  │  └─────────┘  │  ││ App   ││  ││ App   ││  ││ App   ││     │  │   │
│  │  ┌─────────┐  │  │└───────┘│  │└───────┘│  │└───────┘│     │  │   │
│  │  │kube-    │  │  └─────────┘  └─────────┘  └─────────┘     │  │   │
│  │  │proxy    │  └─────────────────────────────────────────────┘  │   │
│  │  │ - Net   │                                                    │   │
│  │  │   rules │  Container Runtime (containerd/CRI-O)             │   │
│  │  └─────────┘                                                    │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                          NODE 2                                  │   │
│  │  [Similar structure...]                                          │   │
│  └─────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────┘
```

### 1.2 Core Concepts

```
Kubernetes Objects Hierarchy
════════════════════════════

Cluster
├── Namespaces (logical isolation)
│   ├── default
│   ├── kube-system
│   └── production
│       ├── Deployments
│       │   └── order-service (manages ReplicaSets)
│       │       └── ReplicaSet (manages Pods)
│       │           └── Pods (smallest deployable unit)
│       │               └── Containers
│       ├── Services (networking)
│       │   ├── ClusterIP
│       │   ├── NodePort
│       │   └── LoadBalancer
│       ├── ConfigMaps (configuration)
│       ├── Secrets (sensitive data)
│       ├── Ingress (external access)
│       └── PersistentVolumeClaims (storage)

Object Relationships:
┌────────────┐     manages     ┌────────────┐
│ Deployment │ ───────────────▶│ ReplicaSet │
└────────────┘                 └─────┬──────┘
                                     │
                                     │ manages
                                     ▼
                               ┌─────────────┐
                               │    Pods     │
                               │ ┌─────────┐ │
                               │ │Container│ │
                               │ └─────────┘ │
                               └─────────────┘
```

## 2. Deploying Java Applications

### 2.1 Basic Deployment

```yaml
# order-service-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
  namespace: production
  labels:
    app: order-service
    version: v1
spec:
  replicas: 3
  selector:
    matchLabels:
      app: order-service
  template:
    metadata:
      labels:
        app: order-service
        version: v1
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8080"
        prometheus.io/path: "/actuator/prometheus"
    spec:
      containers:
        - name: order-service
          image: registry.example.com/order-service:1.0.0
          imagePullPolicy: Always
          ports:
            - name: http
              containerPort: 8080
              protocol: TCP
          env:
            - name: SPRING_PROFILES_ACTIVE
              value: "kubernetes"
            - name: JAVA_OPTS
              value: "-XX:+UseContainerSupport -XX:MaxRAMPercentage=75.0"
          resources:
            requests:
              memory: "512Mi"
              cpu: "250m"
            limits:
              memory: "1Gi"
              cpu: "1000m"
          livenessProbe:
            httpGet:
              path: /actuator/health/liveness
              port: 8080
            initialDelaySeconds: 60
            periodSeconds: 10
            timeoutSeconds: 5
            failureThreshold: 3
          readinessProbe:
            httpGet:
              path: /actuator/health/readiness
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 5
            timeoutSeconds: 3
            failureThreshold: 3
          lifecycle:
            preStop:
              exec:
                command: ["sh", "-c", "sleep 10"]
      terminationGracePeriodSeconds: 30
      imagePullSecrets:
        - name: registry-credentials
```

### 2.2 Service Definition

```yaml
# order-service-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: order-service
  namespace: production
  labels:
    app: order-service
spec:
  type: ClusterIP
  ports:
    - name: http
      port: 80
      targetPort: 8080
      protocol: TCP
  selector:
    app: order-service
---
# Headless service for StatefulSets or direct pod access
apiVersion: v1
kind: Service
metadata:
  name: order-service-headless
  namespace: production
spec:
  type: ClusterIP
  clusterIP: None
  ports:
    - name: http
      port: 8080
  selector:
    app: order-service
```

### 2.3 Ingress Configuration

```yaml
# order-service-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: order-service-ingress
  namespace: production
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/proxy-body-size: "10m"
    nginx.ingress.kubernetes.io/proxy-connect-timeout: "30"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "300"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "300"
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
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
```

## 3. Configuration Management

### 3.1 ConfigMap

```yaml
# order-service-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: order-service-config
  namespace: production
data:
  # Application properties
  application.yml: |
    spring:
      application:
        name: order-service
      datasource:
        hikari:
          maximum-pool-size: 10
          minimum-idle: 5
      rabbitmq:
        host: rabbitmq-service
        port: 5672
      redis:
        host: redis-service
        port: 6379

    management:
      endpoints:
        web:
          exposure:
            include: health,info,prometheus
      health:
        probes:
          enabled: true

  # Logback configuration
  logback-spring.xml: |
    <?xml version="1.0" encoding="UTF-8"?>
    <configuration>
      <include resource="org/springframework/boot/logging/logback/defaults.xml"/>

      <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
          <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n</pattern>
        </encoder>
      </appender>

      <root level="INFO">
        <appender-ref ref="CONSOLE"/>
      </root>

      <logger name="com.example" level="DEBUG"/>
    </configuration>
```

```yaml
# Using ConfigMap in Deployment
spec:
  containers:
    - name: order-service
      volumeMounts:
        - name: config-volume
          mountPath: /app/config
          readOnly: true
      env:
        - name: SPRING_CONFIG_LOCATION
          value: "file:/app/config/"
  volumes:
    - name: config-volume
      configMap:
        name: order-service-config
```

### 3.2 Secrets

```yaml
# order-service-secrets.yaml
apiVersion: v1
kind: Secret
metadata:
  name: order-service-secrets
  namespace: production
type: Opaque
stringData:
  # Will be base64 encoded automatically
  db-username: "order_user"
  db-password: "secretpassword123"
  rabbitmq-username: "app_user"
  rabbitmq-password: "rabbitpassword"
---
# Using external secret (AWS Secrets Manager, HashiCorp Vault)
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: order-service-external-secrets
  namespace: production
spec:
  refreshInterval: 1h
  secretStoreRef:
    kind: ClusterSecretStore
    name: aws-secrets-manager
  target:
    name: order-service-secrets
    creationPolicy: Owner
  data:
    - secretKey: db-password
      remoteRef:
        key: production/order-service/database
        property: password
    - secretKey: api-key
      remoteRef:
        key: production/order-service/api
        property: key
```

```yaml
# Using Secrets in Deployment
spec:
  containers:
    - name: order-service
      env:
        - name: SPRING_DATASOURCE_USERNAME
          valueFrom:
            secretKeyRef:
              name: order-service-secrets
              key: db-username
        - name: SPRING_DATASOURCE_PASSWORD
          valueFrom:
            secretKeyRef:
              name: order-service-secrets
              key: db-password
      # Or mount as files
      volumeMounts:
        - name: secrets-volume
          mountPath: /app/secrets
          readOnly: true
  volumes:
    - name: secrets-volume
      secret:
        secretName: order-service-secrets
```

## 4. Health Checks & Probes

### 4.1 Spring Boot Actuator Configuration

```yaml
# application.yml
management:
  endpoints:
    web:
      exposure:
        include: health,info,prometheus,metrics
      base-path: /actuator
  endpoint:
    health:
      show-details: always
      probes:
        enabled: true
      group:
        liveness:
          include: livenessState,ping
        readiness:
          include: readinessState,db,redis,rabbit
  health:
    livenessState:
      enabled: true
    readinessState:
      enabled: true
    db:
      enabled: true
    redis:
      enabled: true
    rabbit:
      enabled: true
```

```java
// Custom health indicator
@Component
public class CustomServiceHealthIndicator implements HealthIndicator {

    private final ExternalServiceClient externalService;

    public CustomServiceHealthIndicator(ExternalServiceClient externalService) {
        this.externalService = externalService;
    }

    @Override
    public Health health() {
        try {
            boolean isUp = externalService.healthCheck();

            if (isUp) {
                return Health.up()
                    .withDetail("service", "external-api")
                    .withDetail("status", "reachable")
                    .build();
            } else {
                return Health.down()
                    .withDetail("service", "external-api")
                    .withDetail("status", "unreachable")
                    .build();
            }
        } catch (Exception e) {
            return Health.down()
                .withDetail("service", "external-api")
                .withException(e)
                .build();
        }
    }
}

// Readiness state management
@Component
@Slf4j
public class ApplicationReadiness {

    private final ApplicationEventPublisher eventPublisher;

    public ApplicationReadiness(ApplicationEventPublisher eventPublisher) {
        this.eventPublisher = eventPublisher;
    }

    // Call when application is ready to serve traffic
    public void markReady() {
        log.info("Application is ready to serve traffic");
        AvailabilityChangeEvent.publish(
            eventPublisher,
            this,
            ReadinessState.ACCEPTING_TRAFFIC
        );
    }

    // Call when application needs to stop accepting traffic
    public void markUnready() {
        log.warn("Application is not ready to serve traffic");
        AvailabilityChangeEvent.publish(
            eventPublisher,
            this,
            ReadinessState.REFUSING_TRAFFIC
        );
    }
}
```

### 4.2 Kubernetes Probes Configuration

```yaml
# Detailed probes configuration
spec:
  containers:
    - name: order-service
      # Startup probe - for slow starting apps
      startupProbe:
        httpGet:
          path: /actuator/health/liveness
          port: 8080
        initialDelaySeconds: 10
        periodSeconds: 10
        timeoutSeconds: 5
        failureThreshold: 30  # 30 * 10s = 5 minutes to start

      # Liveness probe - is the app alive?
      livenessProbe:
        httpGet:
          path: /actuator/health/liveness
          port: 8080
        initialDelaySeconds: 0  # Not needed with startup probe
        periodSeconds: 10
        timeoutSeconds: 5
        failureThreshold: 3

      # Readiness probe - can it serve traffic?
      readinessProbe:
        httpGet:
          path: /actuator/health/readiness
          port: 8080
        initialDelaySeconds: 0
        periodSeconds: 5
        timeoutSeconds: 3
        failureThreshold: 3
        successThreshold: 1
```

## 5. Resource Management

### 5.1 Resource Requests and Limits

```yaml
# Resource management for Java apps
spec:
  containers:
    - name: order-service
      resources:
        requests:
          # Guaranteed resources
          memory: "512Mi"  # Minimum memory
          cpu: "250m"      # 0.25 CPU cores
        limits:
          # Maximum resources
          memory: "1Gi"    # Will OOMKill if exceeded
          cpu: "1000m"     # Will throttle if exceeded
      env:
        # JVM heap sizing based on container limits
        - name: JAVA_OPTS
          value: >-
            -XX:+UseContainerSupport
            -XX:InitialRAMPercentage=50.0
            -XX:MaxRAMPercentage=75.0
            -XX:+ExitOnOutOfMemoryError
            -XX:+UseG1GC
            -XX:MaxGCPauseMillis=200
```

### 5.2 Horizontal Pod Autoscaler

```yaml
# order-service-hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: order-service-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: order-service
  minReplicas: 3
  maxReplicas: 10
  metrics:
    # CPU-based scaling
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    # Memory-based scaling
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
    # Custom metrics (requires metrics-server + prometheus-adapter)
    - type: Pods
      pods:
        metric:
          name: http_requests_per_second
        target:
          type: AverageValue
          averageValue: "100"
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
        - type: Percent
          value: 10
          periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
        - type: Percent
          value: 100
          periodSeconds: 15
        - type: Pods
          value: 4
          periodSeconds: 15
      selectPolicy: Max
```

### 5.3 Pod Disruption Budget

```yaml
# order-service-pdb.yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: order-service-pdb
  namespace: production
spec:
  minAvailable: 2
  # Or use maxUnavailable: 1
  selector:
    matchLabels:
      app: order-service
```

## 6. kubectl Commands

### 6.1 Essential Commands

```bash
# Cluster information
kubectl cluster-info
kubectl get nodes
kubectl get namespaces

# Create/Apply resources
kubectl apply -f deployment.yaml
kubectl apply -f . --recursive
kubectl apply -k ./kustomize/overlays/production

# View resources
kubectl get pods -n production
kubectl get pods -o wide
kubectl get pods -w  # Watch mode
kubectl get all -n production
kubectl get deployments
kubectl get services
kubectl get ingress
kubectl get configmaps
kubectl get secrets

# Describe resources
kubectl describe pod order-service-xxx
kubectl describe deployment order-service
kubectl describe service order-service
kubectl describe node worker-1

# Logs
kubectl logs order-service-xxx
kubectl logs order-service-xxx -f  # Follow
kubectl logs order-service-xxx --previous  # Previous container
kubectl logs order-service-xxx -c container-name  # Specific container
kubectl logs -l app=order-service --all-containers

# Execute commands
kubectl exec -it order-service-xxx -- /bin/sh
kubectl exec order-service-xxx -- cat /app/config/application.yml

# Port forwarding
kubectl port-forward pod/order-service-xxx 8080:8080
kubectl port-forward svc/order-service 8080:80

# Scaling
kubectl scale deployment order-service --replicas=5
kubectl autoscale deployment order-service --min=3 --max=10 --cpu-percent=70

# Rolling update
kubectl set image deployment/order-service order-service=registry.example.com/order-service:2.0.0
kubectl rollout status deployment/order-service
kubectl rollout history deployment/order-service
kubectl rollout undo deployment/order-service
kubectl rollout undo deployment/order-service --to-revision=2

# Delete resources
kubectl delete pod order-service-xxx
kubectl delete -f deployment.yaml
kubectl delete deployment order-service

# Resource usage
kubectl top nodes
kubectl top pods -n production

# Debug
kubectl get events -n production --sort-by='.lastTimestamp'
kubectl describe pod order-service-xxx
kubectl logs order-service-xxx --previous
```

### 6.2 Useful Aliases and Scripts

```bash
# .bashrc or .zshrc
alias k='kubectl'
alias kgp='kubectl get pods'
alias kgs='kubectl get services'
alias kgd='kubectl get deployments'
alias kgn='kubectl get nodes'
alias kl='kubectl logs'
alias klf='kubectl logs -f'
alias kex='kubectl exec -it'
alias kpf='kubectl port-forward'
alias kctx='kubectl config use-context'
alias kns='kubectl config set-context --current --namespace'

# Functions
kexec() {
    kubectl exec -it "$1" -- "${2:-/bin/sh}"
}

klogs() {
    kubectl logs -f -l "app=$1" --all-containers
}

krestart() {
    kubectl rollout restart deployment "$1"
}
```

```bash
#!/bin/bash
# k8s-debug.sh - Debug pod issues

POD_NAME=$1
NAMESPACE=${2:-default}

echo "=== Pod Details ==="
kubectl describe pod "$POD_NAME" -n "$NAMESPACE"

echo ""
echo "=== Pod Logs ==="
kubectl logs "$POD_NAME" -n "$NAMESPACE" --tail=100

echo ""
echo "=== Previous Container Logs (if any) ==="
kubectl logs "$POD_NAME" -n "$NAMESPACE" --previous --tail=50 2>/dev/null || echo "No previous container"

echo ""
echo "=== Events ==="
kubectl get events -n "$NAMESPACE" --sort-by='.lastTimestamp' | grep "$POD_NAME"

echo ""
echo "=== Resource Usage ==="
kubectl top pod "$POD_NAME" -n "$NAMESPACE" 2>/dev/null || echo "Metrics not available"
```

## 7. Complete Deployment Example

### 7.1 Kustomize Structure

```
kubernetes/
├── base/
│   ├── kustomization.yaml
│   ├── namespace.yaml
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── configmap.yaml
│   └── hpa.yaml
└── overlays/
    ├── development/
    │   ├── kustomization.yaml
    │   └── patch-replicas.yaml
    ├── staging/
    │   ├── kustomization.yaml
    │   └── patch-resources.yaml
    └── production/
        ├── kustomization.yaml
        ├── patch-replicas.yaml
        ├── patch-resources.yaml
        └── ingress.yaml
```

```yaml
# base/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - namespace.yaml
  - deployment.yaml
  - service.yaml
  - configmap.yaml
  - hpa.yaml

commonLabels:
  app.kubernetes.io/name: order-service
  app.kubernetes.io/managed-by: kustomize
```

```yaml
# overlays/production/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: production

resources:
  - ../../base
  - ingress.yaml

patches:
  - path: patch-replicas.yaml
  - path: patch-resources.yaml

images:
  - name: order-service
    newName: registry.example.com/order-service
    newTag: "1.0.0"

configMapGenerator:
  - name: order-service-config
    behavior: merge
    literals:
      - SPRING_PROFILES_ACTIVE=production

secretGenerator:
  - name: order-service-secrets
    type: Opaque
    literals:
      - DB_PASSWORD=prodpassword
```

```yaml
# overlays/production/patch-replicas.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
spec:
  replicas: 5
```

```yaml
# overlays/production/patch-resources.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
spec:
  template:
    spec:
      containers:
        - name: order-service
          resources:
            requests:
              memory: "1Gi"
              cpu: "500m"
            limits:
              memory: "2Gi"
              cpu: "2000m"
```

### 7.2 Deployment Commands

```bash
# Preview changes
kubectl kustomize kubernetes/overlays/production

# Apply to cluster
kubectl apply -k kubernetes/overlays/production

# Verify deployment
kubectl get pods -n production -l app=order-service
kubectl rollout status deployment/order-service -n production

# Check endpoints
kubectl get endpoints order-service -n production
```

## 8. Hands-on Exercises

### Exercise 1: Basic Deployment
Deploy a Spring Boot application to Kubernetes.

```bash
# Requirements:
# 1. Create namespace 'learning'
# 2. Deploy with 3 replicas
# 3. Add Service (ClusterIP)
# 4. Configure health probes
# 5. Test with port-forward
```

### Exercise 2: Configuration Management
Set up ConfigMaps and Secrets.

```bash
# Requirements:
# 1. ConfigMap with application.yml
# 2. Secret with database credentials
# 3. Mount as volumes and environment variables
# 4. Update config and verify hot-reload
```

### Exercise 3: Autoscaling
Configure horizontal pod autoscaler.

```bash
# Requirements:
# 1. Set up HPA with CPU target 70%
# 2. Min 2, Max 10 replicas
# 3. Generate load and observe scaling
# 4. Configure scale-down behavior
```

### Exercise 4: Troubleshooting
Debug a failing deployment.

```bash
# Requirements:
# 1. Deploy a misconfigured app (wrong port, missing config)
# 2. Use kubectl commands to diagnose
# 3. Check events, logs, describe
# 4. Fix and verify
```

## 9. Key Takeaways

### Core Concepts
1. **Pods**: Smallest deployable unit, contains containers
2. **Deployments**: Manage ReplicaSets for declarative updates
3. **Services**: Stable networking for pods
4. **ConfigMaps/Secrets**: Configuration management

### Java Application Tips
1. **Container Support**: Use `-XX:+UseContainerSupport`
2. **Memory Settings**: Use RAM percentage instead of fixed heap
3. **Health Probes**: Configure separate liveness and readiness
4. **Graceful Shutdown**: Handle SIGTERM properly

### Best Practices
1. **Resource Limits**: Always set requests and limits
2. **Pod Disruption Budget**: Ensure availability during updates
3. **Namespaces**: Isolate environments
4. **Labels**: Use consistently for selection

### kubectl Essentials
- `kubectl get`: List resources
- `kubectl describe`: Detailed info
- `kubectl logs`: Container logs
- `kubectl exec`: Run commands in containers
- `kubectl apply`: Create/update resources

---

## Navigation

[← Day 21: Docker Fundamentals](./day-21-docker-fundamentals.md) | [Day 23: Kubernetes Deployments →](./day-23-kubernetes-deployments.md)

[Back to Overview](./00-overview.md)
