# Day 26: Kubernetes Security

## Mục tiêu học tập
- Hiểu Kubernetes security model
- Cấu hình RBAC (Role-Based Access Control)
- Implement Pod Security Standards
- Sử dụng Service Accounts và Secrets management
- Apply security best practices cho Java applications

## 1. Kubernetes Security Model

### 1.1 Security Layers

```
Kubernetes Security Layers
══════════════════════════

┌─────────────────────────────────────────────────────────────────────┐
│                    4. RUNTIME SECURITY                              │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │  - Container isolation                                       │   │
│  │  - Resource limits                                           │   │
│  │  - Security contexts                                         │   │
│  │  - Pod security policies/standards                           │   │
│  └─────────────────────────────────────────────────────────────┘   │
├─────────────────────────────────────────────────────────────────────┤
│                    3. NETWORK SECURITY                              │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │  - Network policies                                          │   │
│  │  - Service mesh (mTLS)                                       │   │
│  │  - Ingress TLS                                               │   │
│  └─────────────────────────────────────────────────────────────┘   │
├─────────────────────────────────────────────────────────────────────┤
│                    2. AUTHORIZATION (RBAC)                          │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │  - Roles / ClusterRoles                                      │   │
│  │  - RoleBindings / ClusterRoleBindings                        │   │
│  │  - Service accounts                                          │   │
│  └─────────────────────────────────────────────────────────────┘   │
├─────────────────────────────────────────────────────────────────────┤
│                    1. AUTHENTICATION                                │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │  - X.509 certificates                                        │   │
│  │  - Bearer tokens                                             │   │
│  │  - OIDC providers                                            │   │
│  │  - Service account tokens                                    │   │
│  └─────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
```

### 1.2 API Request Flow

```
API Request Flow
════════════════

┌──────────┐    ┌─────────────────────────────────────────────────┐
│  Client  │───▶│                 API Server                      │
│ (kubectl,│    │                                                 │
│  pod)    │    │  ┌─────────────┐  ┌───────────┐  ┌──────────┐ │
└──────────┘    │  │ Authenticate│─▶│ Authorize │─▶│ Admission│ │
                │  │             │  │  (RBAC)   │  │ Control  │ │
                │  │ Who are you?│  │ Can you?  │  │ Mutate/  │ │
                │  └─────────────┘  └───────────┘  │ Validate │ │
                │                                  └──────────┘ │
                │                                       │       │
                │                                       ▼       │
                │                               ┌──────────┐    │
                │                               │  etcd    │    │
                │                               └──────────┘    │
                └─────────────────────────────────────────────────┘
```

## 2. RBAC Configuration

### 2.1 Roles and ClusterRoles

```yaml
# role.yaml - Namespace-scoped permissions
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: production
rules:
  - apiGroups: [""]  # Core API group
    resources: ["pods"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["pods/log"]
    verbs: ["get"]
---
# Developer role with deployment permissions
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer
  namespace: production
rules:
  - apiGroups: [""]
    resources: ["pods", "services", "configmaps", "secrets"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["apps"]
    resources: ["deployments", "replicasets"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
  - apiGroups: [""]
    resources: ["pods/exec", "pods/log"]
    verbs: ["get", "create"]
---
# ClusterRole - Cluster-wide permissions
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cluster-admin-readonly
rules:
  - apiGroups: ["*"]
    resources: ["*"]
    verbs: ["get", "list", "watch"]
---
# ClusterRole for specific resources across namespaces
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: secret-reader
rules:
  - apiGroups: [""]
    resources: ["secrets"]
    verbs: ["get", "list"]
    # Can limit to specific resource names
    # resourceNames: ["my-secret"]
```

### 2.2 RoleBindings and ClusterRoleBindings

```yaml
# rolebinding.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: developer-binding
  namespace: production
subjects:
  - kind: User
    name: john@example.com
    apiGroup: rbac.authorization.k8s.io
  - kind: Group
    name: developers
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: developer
  apiGroup: rbac.authorization.k8s.io
---
# Bind ClusterRole to namespace (scoped permissions)
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: admin-view-production
  namespace: production
subjects:
  - kind: User
    name: admin@example.com
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: view  # Built-in ClusterRole
  apiGroup: rbac.authorization.k8s.io
---
# ClusterRoleBinding - Cluster-wide binding
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: cluster-admins
subjects:
  - kind: Group
    name: cluster-admins
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
```

### 2.3 Service Account RBAC

```yaml
# service-account.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: order-service
  namespace: production
  annotations:
    # For AWS IAM roles for service accounts (IRSA)
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789:role/order-service-role
---
# Role for the service account
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: order-service-role
  namespace: production
rules:
  - apiGroups: [""]
    resources: ["configmaps"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["secrets"]
    resourceNames: ["order-service-secrets"]
    verbs: ["get"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: order-service-binding
  namespace: production
subjects:
  - kind: ServiceAccount
    name: order-service
    namespace: production
roleRef:
  kind: Role
  name: order-service-role
  apiGroup: rbac.authorization.k8s.io
```

## 3. Pod Security

### 3.1 Pod Security Context

```yaml
# secure-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-app
  namespace: production
spec:
  # Pod-level security context
  securityContext:
    runAsUser: 1000
    runAsGroup: 1000
    fsGroup: 1000
    runAsNonRoot: true
    seccompProfile:
      type: RuntimeDefault

  containers:
    - name: app
      image: order-service:1.0.0
      # Container-level security context
      securityContext:
        allowPrivilegeEscalation: false
        readOnlyRootFilesystem: true
        runAsNonRoot: true
        runAsUser: 1000
        capabilities:
          drop:
            - ALL
          # Add only needed capabilities
          # add:
          #   - NET_BIND_SERVICE

      volumeMounts:
        - name: tmp
          mountPath: /tmp
        - name: cache
          mountPath: /app/.cache

  volumes:
    - name: tmp
      emptyDir: {}
    - name: cache
      emptyDir: {}
```

### 3.2 Pod Security Standards

```yaml
# namespace-with-pod-security.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    # Pod Security Standards enforcement
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/enforce-version: latest
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/audit-version: latest
    pod-security.kubernetes.io/warn: restricted
    pod-security.kubernetes.io/warn-version: latest
```

```yaml
# deployment-meeting-restricted-standard.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: order-service
  template:
    metadata:
      labels:
        app: order-service
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        runAsGroup: 1000
        fsGroup: 1000
        seccompProfile:
          type: RuntimeDefault

      serviceAccountName: order-service
      automountServiceAccountToken: false  # Disable if not needed

      containers:
        - name: order-service
          image: registry.example.com/order-service:1.0.0
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities:
              drop:
                - ALL
          ports:
            - containerPort: 8080
          resources:
            requests:
              memory: "512Mi"
              cpu: "250m"
            limits:
              memory: "1Gi"
              cpu: "1000m"
          volumeMounts:
            - name: tmp
              mountPath: /tmp
            - name: config
              mountPath: /app/config
              readOnly: true

      volumes:
        - name: tmp
          emptyDir:
            sizeLimit: 100Mi
        - name: config
          configMap:
            name: order-service-config
```

## 4. Secrets Management

### 4.1 Kubernetes Secrets

```yaml
# secrets.yaml
apiVersion: v1
kind: Secret
metadata:
  name: database-credentials
  namespace: production
type: Opaque
stringData:
  username: db_user
  password: supersecretpassword
---
# TLS Secret
apiVersion: v1
kind: Secret
metadata:
  name: tls-secret
  namespace: production
type: kubernetes.io/tls
data:
  tls.crt: <base64-encoded-cert>
  tls.key: <base64-encoded-key>
---
# Docker registry secret
apiVersion: v1
kind: Secret
metadata:
  name: registry-credentials
  namespace: production
type: kubernetes.io/dockerconfigjson
data:
  .dockerconfigjson: <base64-encoded-docker-config>
```

### 4.2 External Secrets Operator

```yaml
# external-secret.yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: database-credentials
  namespace: production
spec:
  refreshInterval: 1h

  secretStoreRef:
    kind: ClusterSecretStore
    name: aws-secrets-manager

  target:
    name: database-credentials
    creationPolicy: Owner
    template:
      type: Opaque
      data:
        username: "{{ .username }}"
        password: "{{ .password }}"
        connection-string: "jdbc:mysql://{{ .host }}:{{ .port }}/{{ .database }}"

  data:
    - secretKey: username
      remoteRef:
        key: production/database
        property: username

    - secretKey: password
      remoteRef:
        key: production/database
        property: password

    - secretKey: host
      remoteRef:
        key: production/database
        property: host

    - secretKey: port
      remoteRef:
        key: production/database
        property: port

    - secretKey: database
      remoteRef:
        key: production/database
        property: database
---
# ClusterSecretStore for AWS Secrets Manager
apiVersion: external-secrets.io/v1beta1
kind: ClusterSecretStore
metadata:
  name: aws-secrets-manager
spec:
  provider:
    aws:
      service: SecretsManager
      region: us-east-1
      auth:
        jwt:
          serviceAccountRef:
            name: external-secrets
            namespace: external-secrets
```

### 4.3 HashiCorp Vault Integration

```yaml
# vault-secret.yaml
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: vault-backend
  namespace: production
spec:
  provider:
    vault:
      server: "https://vault.example.com:8200"
      path: "secret"
      version: "v2"
      auth:
        kubernetes:
          mountPath: "kubernetes"
          role: "order-service"
          serviceAccountRef:
            name: "order-service"
---
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: vault-secrets
  namespace: production
spec:
  refreshInterval: 15m
  secretStoreRef:
    kind: SecretStore
    name: vault-backend
  target:
    name: order-service-secrets
  data:
    - secretKey: db-password
      remoteRef:
        key: secret/data/production/database
        property: password
    - secretKey: api-key
      remoteRef:
        key: secret/data/production/api
        property: key
```

## 5. Network Security

### 5.1 Secure Ingress with TLS

```yaml
# secure-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: secure-api
  namespace: production
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
    cert-manager.io/cluster-issuer: "letsencrypt-prod"

    # Security headers
    nginx.ingress.kubernetes.io/configuration-snippet: |
      more_set_headers "X-Frame-Options: DENY";
      more_set_headers "X-Content-Type-Options: nosniff";
      more_set_headers "X-XSS-Protection: 1; mode=block";
      more_set_headers "Strict-Transport-Security: max-age=31536000; includeSubDomains";
      more_set_headers "Content-Security-Policy: default-src 'self'";

    # Rate limiting
    nginx.ingress.kubernetes.io/limit-rps: "100"
    nginx.ingress.kubernetes.io/limit-connections: "10"

    # WAF rules (if using ModSecurity)
    nginx.ingress.kubernetes.io/enable-modsecurity: "true"
    nginx.ingress.kubernetes.io/modsecurity-snippet: |
      SecRuleEngine On
      SecRule ARGS "@contains script" "id:1,deny,status:403"

spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - api.example.com
      secretName: api-tls
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
```

### 5.2 Network Policy for Zero Trust

```yaml
# zero-trust-network-policy.yaml
# Default deny all
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
  namespace: production
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
---
# Allow specific traffic for order-service
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
    - from:
        - podSelector:
            matchLabels:
              app: api-gateway
      ports:
        - protocol: TCP
          port: 8080
  egress:
    - to:
        - podSelector:
            matchLabels:
              app: mysql
      ports:
        - protocol: TCP
          port: 3306
    - to:
        - podSelector:
            matchLabels:
              app: redis
      ports:
        - protocol: TCP
          port: 6379
    # Allow DNS
    - to:
        - namespaceSelector: {}
          podSelector:
            matchLabels:
              k8s-app: kube-dns
      ports:
        - protocol: UDP
          port: 53
```

## 6. Security Scanning and Compliance

### 6.1 Image Scanning with Trivy

```yaml
# trivy-scan-job.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: image-scan
  namespace: security
spec:
  template:
    spec:
      containers:
        - name: trivy
          image: aquasec/trivy:latest
          args:
            - image
            - --severity
            - "HIGH,CRITICAL"
            - --exit-code
            - "1"
            - registry.example.com/order-service:1.0.0
          env:
            - name: TRIVY_USERNAME
              valueFrom:
                secretKeyRef:
                  name: registry-credentials
                  key: username
            - name: TRIVY_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: registry-credentials
                  key: password
      restartPolicy: Never
  backoffLimit: 1
```

### 6.2 OPA Gatekeeper Constraints

```yaml
# require-labels-constraint.yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8srequiredlabels
spec:
  crd:
    spec:
      names:
        kind: K8sRequiredLabels
      validation:
        openAPIV3Schema:
          type: object
          properties:
            labels:
              type: array
              items:
                type: string
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8srequiredlabels

        violation[{"msg": msg, "details": {"missing_labels": missing}}] {
          provided := {label | input.review.object.metadata.labels[label]}
          required := {label | label := input.parameters.labels[_]}
          missing := required - provided
          count(missing) > 0
          msg := sprintf("Missing required labels: %v", [missing])
        }
---
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredLabels
metadata:
  name: require-app-labels
spec:
  match:
    kinds:
      - apiGroups: ["apps"]
        kinds: ["Deployment"]
    namespaces: ["production"]
  parameters:
    labels:
      - "app"
      - "version"
      - "owner"
```

## 7. Java Application Security

### 7.1 Secure Spring Boot Configuration

```yaml
# application-kubernetes.yml
spring:
  security:
    # Disable sensitive endpoints
    user:
      name: ${ADMIN_USER:admin}
      password: ${ADMIN_PASSWORD}

management:
  endpoints:
    web:
      exposure:
        include: health,info,prometheus
  endpoint:
    health:
      probes:
        enabled: true
      show-details: when_authorized

server:
  # Security headers
  error:
    include-stacktrace: never
    include-message: never

  # SSL configuration if needed
  ssl:
    enabled: false  # TLS termination at ingress

logging:
  level:
    org.springframework.security: INFO
  # Don't log sensitive data
  pattern:
    console: "%d{yyyy-MM-dd HH:mm:ss} - %msg%n"
```

### 7.2 Secure Dockerfile

```dockerfile
# Dockerfile with security best practices
FROM eclipse-temurin:21-jre-alpine AS runtime

# Create non-root user
RUN addgroup -g 1000 appgroup && \
    adduser -u 1000 -G appgroup -H -D appuser

# Install security updates
RUN apk update && apk upgrade && \
    rm -rf /var/cache/apk/*

WORKDIR /app

# Copy application with proper ownership
COPY --chown=appuser:appgroup target/app.jar /app/app.jar

# Remove unnecessary tools
RUN rm -rf /usr/bin/wget /usr/bin/curl 2>/dev/null || true

# Set file permissions
RUN chmod 400 /app/app.jar

# Switch to non-root user
USER appuser

# No shell access
SHELL ["/bin/false"]

EXPOSE 8080

ENTRYPOINT ["java", \
    "-XX:+UseContainerSupport", \
    "-Djava.security.egd=file:/dev/./urandom", \
    "-jar", "app.jar"]
```

## 8. Hands-on Exercises

### Exercise 1: RBAC Setup
Configure RBAC for a development team.

```bash
# Requirements:
# 1. Create developer Role with limited permissions
# 2. Create RoleBinding for team members
# 3. Test access with kubectl auth can-i
# 4. Create read-only ClusterRole for monitoring
```

### Exercise 2: Pod Security
Implement Pod Security Standards.

```bash
# Requirements:
# 1. Enable restricted PSS on namespace
# 2. Deploy compliant application
# 3. Test non-compliant deployment (should fail)
# 4. Fix and redeploy
```

### Exercise 3: Secrets Management
Set up external secrets management.

```bash
# Requirements:
# 1. Install External Secrets Operator
# 2. Configure AWS Secrets Manager store
# 3. Create ExternalSecret
# 4. Verify secret synchronization
```

### Exercise 4: Security Audit
Perform security audit of cluster.

```bash
# Requirements:
# 1. Run kube-bench for CIS benchmarks
# 2. Scan images with Trivy
# 3. Check RBAC with rbac-police
# 4. Generate compliance report
```

## 9. Key Takeaways

### RBAC Principles
1. **Least Privilege**: Grant minimum required permissions
2. **Namespace Scoping**: Use Roles when possible
3. **Service Accounts**: Dedicated accounts per application
4. **Regular Audit**: Review permissions periodically

### Pod Security
1. **Non-root**: Always run as non-root user
2. **Read-only FS**: Use read-only root filesystem
3. **Drop Capabilities**: Remove all unnecessary capabilities
4. **Security Context**: Set at both pod and container level

### Secrets Management
1. **External Stores**: Use Vault, AWS Secrets Manager
2. **Rotation**: Implement automatic secret rotation
3. **Encryption**: Enable etcd encryption at rest
4. **Access Control**: Limit secret access via RBAC

### Network Security
1. **Default Deny**: Start with deny-all policies
2. **TLS Everywhere**: Encrypt all traffic
3. **Zero Trust**: Verify every connection
4. **Ingress Protection**: WAF, rate limiting

---

## Navigation

[← Day 25: Kubernetes Storage](./day-25-kubernetes-storage.md) | [Day 27: Kubernetes Observability →](./day-27-kubernetes-observability.md)

[Back to Overview](./00-overview.md)
