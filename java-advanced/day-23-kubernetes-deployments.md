# Day 23: Kubernetes Deployments & Strategies

## Mục tiêu học tập
- Hiểu sâu về Deployment strategies trong Kubernetes
- Implement Rolling Updates, Blue-Green, và Canary deployments
- Cấu hình rollback và revision history
- Sử dụng Helm charts cho package management
- Implement GitOps với ArgoCD

## 1. Deployment Strategies

### 1.1 Strategy Overview

```
Kubernetes Deployment Strategies
════════════════════════════════

1. Rolling Update (Default)
   ┌─────────────────────────────────────────────────────────────┐
   │ Time 0:  [v1] [v1] [v1] [v1]                                │
   │ Time 1:  [v1] [v1] [v1] [v2]  ← New pod created            │
   │ Time 2:  [v1] [v1] [v2] [v2]  ← Old pod terminated         │
   │ Time 3:  [v1] [v2] [v2] [v2]                                │
   │ Time 4:  [v2] [v2] [v2] [v2]  ← Complete                   │
   │                                                             │
   │ ✓ Zero downtime                                             │
   │ ✓ Gradual rollout                                           │
   │ ✗ Mixed versions during update                              │
   └─────────────────────────────────────────────────────────────┘

2. Blue-Green Deployment
   ┌─────────────────────────────────────────────────────────────┐
   │               Service                                       │
   │                  │                                          │
   │        ┌─────────┴─────────┐                               │
   │        ▼                   ▼                                │
   │   Blue (v1)           Green (v2)                           │
   │   [v1][v1][v1]        [v2][v2][v2]                         │
   │   (inactive)          (active)                              │
   │                                                             │
   │ ✓ Instant rollback                                          │
   │ ✓ No mixed versions                                         │
   │ ✗ Double resource usage                                     │
   └─────────────────────────────────────────────────────────────┘

3. Canary Deployment
   ┌─────────────────────────────────────────────────────────────┐
   │               Service (weighted)                            │
   │                  │                                          │
   │        ┌─────────┴─────────┐                               │
   │        │                   │                                │
   │    90% ▼               10% ▼                                │
   │   Stable (v1)         Canary (v2)                          │
   │   [v1][v1][v1][v1]    [v2]                                 │
   │                                                             │
   │ ✓ Test with real traffic                                    │
   │ ✓ Gradual risk exposure                                     │
   │ ✗ Complex traffic routing                                   │
   └─────────────────────────────────────────────────────────────┘
```

## 2. Rolling Update Configuration

### 2.1 Deployment with Rolling Update

```yaml
# rolling-update-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
  namespace: production
  annotations:
    kubernetes.io/change-cause: "Update to version 2.0.0"
spec:
  replicas: 4
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app: order-service
  strategy:
    type: RollingUpdate
    rollingUpdate:
      # Max pods that can be created over desired replicas
      maxSurge: 1         # or "25%"
      # Max pods that can be unavailable
      maxUnavailable: 0   # or "25%"
  template:
    metadata:
      labels:
        app: order-service
        version: v2.0.0
    spec:
      containers:
        - name: order-service
          image: registry.example.com/order-service:2.0.0
          ports:
            - containerPort: 8080
          readinessProbe:
            httpGet:
              path: /actuator/health/readiness
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 5
            successThreshold: 1
            failureThreshold: 3
          livenessProbe:
            httpGet:
              path: /actuator/health/liveness
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 10
          resources:
            requests:
              memory: "512Mi"
              cpu: "250m"
            limits:
              memory: "1Gi"
              cpu: "1000m"
      # Wait for pod to be fully ready before continuing
      minReadySeconds: 10
      # Graceful shutdown
      terminationGracePeriodSeconds: 30
```

### 2.2 Rolling Update Commands

```bash
# Apply deployment update
kubectl apply -f rolling-update-deployment.yaml

# Watch the rollout
kubectl rollout status deployment/order-service -n production

# Check rollout history
kubectl rollout history deployment/order-service -n production

# View specific revision details
kubectl rollout history deployment/order-service --revision=3

# Pause rollout (for canary-like behavior)
kubectl rollout pause deployment/order-service

# Resume rollout
kubectl rollout resume deployment/order-service

# Undo to previous version
kubectl rollout undo deployment/order-service

# Undo to specific revision
kubectl rollout undo deployment/order-service --to-revision=2

# Annotate deployment for change tracking
kubectl annotate deployment/order-service \
  kubernetes.io/change-cause="Hotfix for order validation bug"
```

## 3. Blue-Green Deployment

### 3.1 Blue-Green Configuration

```yaml
# blue-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service-blue
  namespace: production
  labels:
    app: order-service
    version: blue
spec:
  replicas: 3
  selector:
    matchLabels:
      app: order-service
      version: blue
  template:
    metadata:
      labels:
        app: order-service
        version: blue
    spec:
      containers:
        - name: order-service
          image: registry.example.com/order-service:1.0.0
          ports:
            - containerPort: 8080
          readinessProbe:
            httpGet:
              path: /actuator/health/readiness
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 5
---
# green-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service-green
  namespace: production
  labels:
    app: order-service
    version: green
spec:
  replicas: 3
  selector:
    matchLabels:
      app: order-service
      version: green
  template:
    metadata:
      labels:
        app: order-service
        version: green
    spec:
      containers:
        - name: order-service
          image: registry.example.com/order-service:2.0.0
          ports:
            - containerPort: 8080
          readinessProbe:
            httpGet:
              path: /actuator/health/readiness
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 5
---
# Service that routes to active deployment
apiVersion: v1
kind: Service
metadata:
  name: order-service
  namespace: production
spec:
  type: ClusterIP
  ports:
    - port: 80
      targetPort: 8080
  # Selector points to active deployment
  selector:
    app: order-service
    version: green  # Switch between blue/green
```

### 3.2 Blue-Green Switch Script

```bash
#!/bin/bash
# blue-green-switch.sh

NAMESPACE="production"
SERVICE_NAME="order-service"
CURRENT_VERSION=$(kubectl get svc $SERVICE_NAME -n $NAMESPACE -o jsonpath='{.spec.selector.version}')

if [ "$CURRENT_VERSION" == "blue" ]; then
    NEW_VERSION="green"
else
    NEW_VERSION="blue"
fi

echo "Current version: $CURRENT_VERSION"
echo "Switching to: $NEW_VERSION"

# Verify new deployment is ready
READY_REPLICAS=$(kubectl get deployment ${SERVICE_NAME}-${NEW_VERSION} -n $NAMESPACE \
    -o jsonpath='{.status.readyReplicas}')
DESIRED_REPLICAS=$(kubectl get deployment ${SERVICE_NAME}-${NEW_VERSION} -n $NAMESPACE \
    -o jsonpath='{.spec.replicas}')

if [ "$READY_REPLICAS" != "$DESIRED_REPLICAS" ]; then
    echo "Error: $NEW_VERSION deployment not ready ($READY_REPLICAS/$DESIRED_REPLICAS)"
    exit 1
fi

# Switch service selector
kubectl patch service $SERVICE_NAME -n $NAMESPACE \
    -p "{\"spec\":{\"selector\":{\"version\":\"$NEW_VERSION\"}}}"

echo "Switched to $NEW_VERSION successfully!"

# Verify the switch
kubectl get endpoints $SERVICE_NAME -n $NAMESPACE
```

## 4. Canary Deployment

### 4.1 Basic Canary with Multiple Deployments

```yaml
# canary-stable.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service-stable
  namespace: production
spec:
  replicas: 9  # 90% of traffic
  selector:
    matchLabels:
      app: order-service
      track: stable
  template:
    metadata:
      labels:
        app: order-service
        track: stable
    spec:
      containers:
        - name: order-service
          image: registry.example.com/order-service:1.0.0
          ports:
            - containerPort: 8080
---
# canary-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service-canary
  namespace: production
spec:
  replicas: 1  # 10% of traffic
  selector:
    matchLabels:
      app: order-service
      track: canary
  template:
    metadata:
      labels:
        app: order-service
        track: canary
    spec:
      containers:
        - name: order-service
          image: registry.example.com/order-service:2.0.0
          ports:
            - containerPort: 8080
---
# Service selects both stable and canary
apiVersion: v1
kind: Service
metadata:
  name: order-service
  namespace: production
spec:
  type: ClusterIP
  ports:
    - port: 80
      targetPort: 8080
  selector:
    app: order-service  # Matches both stable and canary
```

### 4.2 Istio-Based Canary (Advanced)

```yaml
# istio-virtual-service.yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: order-service
  namespace: production
spec:
  hosts:
    - order-service
  http:
    - match:
        - headers:
            x-canary:
              exact: "true"
      route:
        - destination:
            host: order-service
            subset: canary
    - route:
        - destination:
            host: order-service
            subset: stable
          weight: 90
        - destination:
            host: order-service
            subset: canary
          weight: 10
---
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: order-service
  namespace: production
spec:
  host: order-service
  subsets:
    - name: stable
      labels:
        track: stable
    - name: canary
      labels:
        track: canary
```

### 4.3 Canary Promotion Script

```bash
#!/bin/bash
# canary-promote.sh

NAMESPACE="production"
SERVICE="order-service"
STABLE_DEPLOY="${SERVICE}-stable"
CANARY_DEPLOY="${SERVICE}-canary"

# Get current canary image
CANARY_IMAGE=$(kubectl get deployment $CANARY_DEPLOY -n $NAMESPACE \
    -o jsonpath='{.spec.template.spec.containers[0].image}')

echo "Promoting canary image: $CANARY_IMAGE"

# Update stable deployment with canary image
kubectl set image deployment/$STABLE_DEPLOY \
    order-service=$CANARY_IMAGE -n $NAMESPACE

# Wait for rollout
kubectl rollout status deployment/$STABLE_DEPLOY -n $NAMESPACE

# Scale down canary
kubectl scale deployment/$CANARY_DEPLOY --replicas=0 -n $NAMESPACE

echo "Canary promoted to stable!"
```

## 5. Helm Package Management

### 5.1 Helm Chart Structure

```
order-service-chart/
├── Chart.yaml
├── values.yaml
├── values-dev.yaml
├── values-staging.yaml
├── values-production.yaml
├── templates/
│   ├── _helpers.tpl
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── configmap.yaml
│   ├── secret.yaml
│   ├── hpa.yaml
│   ├── pdb.yaml
│   └── serviceaccount.yaml
└── charts/
    └── (dependencies)
```

### 5.2 Chart.yaml

```yaml
# Chart.yaml
apiVersion: v2
name: order-service
description: Order Service Helm Chart
type: application
version: 1.0.0
appVersion: "2.0.0"
keywords:
  - order
  - microservice
  - java
  - spring-boot
maintainers:
  - name: DevOps Team
    email: devops@example.com
dependencies:
  - name: redis
    version: "17.x.x"
    repository: "https://charts.bitnami.com/bitnami"
    condition: redis.enabled
```

### 5.3 Values.yaml

```yaml
# values.yaml
replicaCount: 3

image:
  repository: registry.example.com/order-service
  tag: "latest"
  pullPolicy: IfNotPresent

imagePullSecrets:
  - name: registry-credentials

nameOverride: ""
fullnameOverride: ""

serviceAccount:
  create: true
  annotations: {}
  name: ""

podAnnotations:
  prometheus.io/scrape: "true"
  prometheus.io/port: "8080"

podSecurityContext:
  runAsNonRoot: true
  runAsUser: 1000

securityContext:
  capabilities:
    drop:
      - ALL
  readOnlyRootFilesystem: true

service:
  type: ClusterIP
  port: 80
  targetPort: 8080

ingress:
  enabled: true
  className: nginx
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
  hosts:
    - host: api.example.com
      paths:
        - path: /orders
          pathType: Prefix
  tls:
    - secretName: api-tls
      hosts:
        - api.example.com

resources:
  requests:
    cpu: 250m
    memory: 512Mi
  limits:
    cpu: 1000m
    memory: 1Gi

autoscaling:
  enabled: true
  minReplicas: 3
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70

nodeSelector: {}

tolerations: []

affinity:
  podAntiAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchExpressions:
              - key: app
                operator: In
                values:
                  - order-service
          topologyKey: kubernetes.io/hostname

# Application configuration
config:
  springProfile: default
  logLevel: INFO

# External configuration
database:
  host: mysql-service
  port: 3306
  name: orders

redis:
  enabled: true
  host: redis-service
  port: 6379

rabbitmq:
  host: rabbitmq-service
  port: 5672

# Health probes
probes:
  liveness:
    path: /actuator/health/liveness
    initialDelaySeconds: 60
    periodSeconds: 10
  readiness:
    path: /actuator/health/readiness
    initialDelaySeconds: 30
    periodSeconds: 5

# Pod disruption budget
pdb:
  enabled: true
  minAvailable: 2
```

### 5.4 Deployment Template

```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "order-service.fullname" . }}
  labels:
    {{- include "order-service.labels" . | nindent 4 }}
spec:
  {{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount }}
  {{- end }}
  selector:
    matchLabels:
      {{- include "order-service.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      annotations:
        checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
        {{- with .Values.podAnnotations }}
        {{- toYaml . | nindent 8 }}
        {{- end }}
      labels:
        {{- include "order-service.selectorLabels" . | nindent 8 }}
    spec:
      {{- with .Values.imagePullSecrets }}
      imagePullSecrets:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      serviceAccountName: {{ include "order-service.serviceAccountName" . }}
      securityContext:
        {{- toYaml .Values.podSecurityContext | nindent 8 }}
      containers:
        - name: {{ .Chart.Name }}
          securityContext:
            {{- toYaml .Values.securityContext | nindent 12 }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - name: http
              containerPort: {{ .Values.service.targetPort }}
              protocol: TCP
          envFrom:
            - configMapRef:
                name: {{ include "order-service.fullname" . }}-config
            - secretRef:
                name: {{ include "order-service.fullname" . }}-secrets
          livenessProbe:
            httpGet:
              path: {{ .Values.probes.liveness.path }}
              port: http
            initialDelaySeconds: {{ .Values.probes.liveness.initialDelaySeconds }}
            periodSeconds: {{ .Values.probes.liveness.periodSeconds }}
          readinessProbe:
            httpGet:
              path: {{ .Values.probes.readiness.path }}
              port: http
            initialDelaySeconds: {{ .Values.probes.readiness.initialDelaySeconds }}
            periodSeconds: {{ .Values.probes.readiness.periodSeconds }}
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
          volumeMounts:
            - name: tmp
              mountPath: /tmp
      volumes:
        - name: tmp
          emptyDir: {}
      {{- with .Values.nodeSelector }}
      nodeSelector:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      {{- with .Values.affinity }}
      affinity:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      {{- with .Values.tolerations }}
      tolerations:
        {{- toYaml . | nindent 8 }}
      {{- end }}
```

### 5.5 Helm Commands

```bash
# Create new chart
helm create order-service

# Lint chart
helm lint ./order-service-chart

# Template rendering (dry-run)
helm template order-service ./order-service-chart -f values-production.yaml

# Install chart
helm install order-service ./order-service-chart \
    -n production \
    -f values-production.yaml

# Upgrade chart
helm upgrade order-service ./order-service-chart \
    -n production \
    -f values-production.yaml \
    --set image.tag=2.0.0

# Upgrade with atomic (rollback on failure)
helm upgrade order-service ./order-service-chart \
    -n production \
    -f values-production.yaml \
    --atomic \
    --timeout 5m

# List releases
helm list -n production

# Get release history
helm history order-service -n production

# Rollback
helm rollback order-service 1 -n production

# Uninstall
helm uninstall order-service -n production
```

## 6. GitOps with ArgoCD

### 6.1 ArgoCD Application

```yaml
# argocd-application.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: order-service
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default

  source:
    repoURL: https://github.com/company/k8s-manifests.git
    targetRevision: main
    path: apps/order-service/overlays/production

  destination:
    server: https://kubernetes.default.svc
    namespace: production

  syncPolicy:
    automated:
      prune: true
      selfHeal: true
      allowEmpty: false
    syncOptions:
      - CreateNamespace=true
      - PrunePropagationPolicy=foreground
      - PruneLast=true
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m

  # Health assessment
  ignoreDifferences:
    - group: apps
      kind: Deployment
      jsonPointers:
        - /spec/replicas  # Ignore HPA-managed replicas
```

### 6.2 ArgoCD Application Set

```yaml
# application-set.yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: microservices
  namespace: argocd
spec:
  generators:
    - list:
        elements:
          - name: order-service
            namespace: production
          - name: payment-service
            namespace: production
          - name: inventory-service
            namespace: production

  template:
    metadata:
      name: '{{name}}'
    spec:
      project: default
      source:
        repoURL: https://github.com/company/k8s-manifests.git
        targetRevision: main
        path: 'apps/{{name}}/overlays/production'
      destination:
        server: https://kubernetes.default.svc
        namespace: '{{namespace}}'
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
```

### 6.3 Progressive Delivery with Argo Rollouts

```yaml
# argo-rollout.yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: order-service
  namespace: production
spec:
  replicas: 5
  revisionHistoryLimit: 3
  selector:
    matchLabels:
      app: order-service
  template:
    metadata:
      labels:
        app: order-service
    spec:
      containers:
        - name: order-service
          image: registry.example.com/order-service:2.0.0
          ports:
            - containerPort: 8080
  strategy:
    canary:
      maxSurge: 1
      maxUnavailable: 0
      steps:
        - setWeight: 10
        - pause: { duration: 1m }
        - setWeight: 30
        - pause: { duration: 2m }
        - setWeight: 50
        - pause: { duration: 3m }
        - setWeight: 80
        - pause: { duration: 5m }
      # Analysis for automatic promotion/rollback
      analysis:
        templates:
          - templateName: success-rate
        startingStep: 2
        args:
          - name: service-name
            value: order-service
---
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: success-rate
  namespace: production
spec:
  args:
    - name: service-name
  metrics:
    - name: success-rate
      interval: 1m
      successCondition: result[0] >= 0.95
      failureLimit: 3
      provider:
        prometheus:
          address: http://prometheus:9090
          query: |
            sum(rate(http_server_requests_seconds_count{
              service="{{args.service-name}}",
              status=~"2.."
            }[1m])) /
            sum(rate(http_server_requests_seconds_count{
              service="{{args.service-name}}"
            }[1m]))
```

## 7. Hands-on Exercises

### Exercise 1: Rolling Update Strategy
Configure and test rolling updates.

```bash
# Requirements:
# 1. Deploy with maxSurge=1, maxUnavailable=0
# 2. Update image and watch rollout
# 3. Pause mid-rollout
# 4. Resume and complete
# 5. Practice rollback
```

### Exercise 2: Blue-Green Deployment
Implement blue-green deployment manually.

```bash
# Requirements:
# 1. Create blue and green deployments
# 2. Service pointing to blue
# 3. Deploy new version to green
# 4. Switch service to green
# 5. Verify and cleanup blue
```

### Exercise 3: Helm Chart Creation
Package application as Helm chart.

```bash
# Requirements:
# 1. Create chart with all K8s resources
# 2. Parameterize with values.yaml
# 3. Create values for dev/staging/prod
# 4. Install and upgrade
# 5. Rollback on failure
```

### Exercise 4: ArgoCD GitOps
Set up GitOps workflow with ArgoCD.

```bash
# Requirements:
# 1. Install ArgoCD
# 2. Connect Git repository
# 3. Create Application
# 4. Enable auto-sync
# 5. Test self-healing
```

## 8. Key Takeaways

### Deployment Strategies
1. **Rolling Update**: Default, gradual, mixed versions
2. **Blue-Green**: Instant switch, double resources
3. **Canary**: Gradual traffic shift, risk mitigation

### Helm Best Practices
1. **Parameterize**: Use values.yaml for configuration
2. **Environment Files**: Separate values per environment
3. **Atomic Upgrades**: Use --atomic for safe rollouts
4. **Version Pinning**: Pin chart and app versions

### GitOps Principles
1. **Declarative**: Define desired state in Git
2. **Versioned**: Git history is deployment history
3. **Automated**: Reconciliation loop maintains state
4. **Self-Healing**: Detect and fix drift automatically

### Rollback Strategies
1. **kubectl rollout undo**: Quick K8s rollback
2. **helm rollback**: Restore previous Helm release
3. **Git revert**: GitOps rollback via source control

---

## Navigation

[← Day 22: Kubernetes Basics](./day-22-kubernetes-basics.md) | [Day 24: Kubernetes Networking →](./day-24-kubernetes-networking.md)

[Back to Overview](./00-overview.md)
