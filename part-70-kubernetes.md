# Part 70: Kubernetes
## ขั้นตอนที่ 691-700 จาก 1000

---

## Kubernetes คืออะไร?

Kubernetes (k8s) เป็น container orchestration platform ที่ช่วย deploy, scale, และ manage containerized applications อัตโนมัติ พัฒนาโดย Google

---

## 1. Kubernetes Concepts

### Cluster Components

```
Master Node (Control Plane):
  - API Server: จุดรับ requests ทั้งหมด
  - Scheduler: ตัดสินใจว่า Pod จะ run ที่ node ไหน
  - Controller Manager: ดูแล desired state
  - etcd: distributed key-value store

Worker Node:
  - kubelet: agent ที่ manage Pods
  - kube-proxy: network proxy
  - Container Runtime (Docker/containerd)
```

### Core Objects

```
Pod: smallest deployable unit (1 หรือ หลาย containers)
Service: network access ไปยัง Pods
Deployment: manage Pod replicas
ConfigMap: เก็บ configuration
Secret: เก็บ sensitive data
Namespace: แยก resources
```

---

## 2. Deployments

### Dockerfile

```dockerfile
# Dockerfile
FROM node:18-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production

FROM node:18-alpine
RUN addgroup -g 1001 -S nodejs && adduser -S nodejs -u 1001
WORKDIR /app
COPY --from=builder /app/node_modules ./node_modules
COPY --chown=nodejs:nodejs . .
USER nodejs
EXPOSE 3000
CMD ["node", "src/index.js"]
```

### Deployment YAML

```yaml
# k8s/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nodejs-app
  namespace: production
  labels:
    app: nodejs-app
    version: v1.0.0
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nodejs-app
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0  # Zero-downtime deployment
  template:
    metadata:
      labels:
        app: nodejs-app
        version: v1.0.0
    spec:
      containers:
        - name: nodejs-app
          image: myregistry/nodejs-app:1.0.0
          ports:
            - containerPort: 3000
          
          env:
            - name: NODE_ENV
              value: production
            - name: PORT
              value: "3000"
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: app-secrets
                  key: database-url
            - name: APP_CONFIG
              valueFrom:
                configMapKeyRef:
                  name: app-config
                  key: config.json
          
          resources:
            requests:
              memory: "128Mi"
              cpu: "100m"
            limits:
              memory: "512Mi"
              cpu: "500m"
          
          livenessProbe:
            httpGet:
              path: /health
              port: 3000
            initialDelaySeconds: 30
            periodSeconds: 10
            failureThreshold: 3
          
          readinessProbe:
            httpGet:
              path: /ready
              port: 3000
            initialDelaySeconds: 5
            periodSeconds: 5
            successThreshold: 2
          
          lifecycle:
            preStop:
              exec:
                command: ["/bin/sh", "-c", "sleep 5"]
      
      terminationGracePeriodSeconds: 60
      
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchExpressions:
                  - key: app
                    operator: In
                    values: [nodejs-app]
              topologyKey: kubernetes.io/hostname
```

### Horizontal Pod Autoscaler

```yaml
# k8s/hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: nodejs-app-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: nodejs-app
  minReplicas: 3
  maxReplicas: 20
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
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
        - type: Percent
          value: 20
          periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
        - type: Percent
          value: 100
          periodSeconds: 30
```

---

## 3. Services

### ClusterIP Service

```yaml
# k8s/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: nodejs-app-service
  namespace: production
spec:
  selector:
    app: nodejs-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 3000
  type: ClusterIP
```

### LoadBalancer Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nodejs-app-lb
  namespace: production
  annotations:
    # AWS specific
    service.beta.kubernetes.io/aws-load-balancer-type: nlb
spec:
  selector:
    app: nodejs-app
  ports:
    - port: 80
      targetPort: 3000
  type: LoadBalancer
```

### Ingress

```yaml
# k8s/ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nodejs-app-ingress
  namespace: production
  annotations:
    kubernetes.io/ingress.class: nginx
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/rate-limit: "100"
    nginx.ingress.kubernetes.io/proxy-body-size: "50m"
spec:
  tls:
    - hosts:
        - api.example.com
      secretName: api-tls-secret
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: nodejs-app-service
                port:
                  number: 80
```

---

## 4. ConfigMaps and Secrets

### ConfigMap

```yaml
# k8s/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: production
data:
  NODE_ENV: production
  LOG_LEVEL: info
  MAX_UPLOAD_SIZE: "50mb"
  RATE_LIMIT_WINDOW: "900000"
  RATE_LIMIT_MAX: "100"
  config.json: |
    {
      "features": {
        "emailVerification": true,
        "twoFactorAuth": true
      },
      "pagination": {
        "defaultLimit": 20,
        "maxLimit": 100
      }
    }
```

### Secret

```yaml
# k8s/secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
  namespace: production
type: Opaque
data:
  # Values ต้อง base64 encoded
  # echo -n "value" | base64
  database-url: bW9uZ29kYjovL3VzZXI6cGFzc0BtZWdhdGV4LmNvbTo1NDM3L2Ri
  jwt-secret: c3VwZXItc2VjcmV0LWp3dC1rZXktMTIzNDU2Nzg5MA==
  redis-url: cmVkaXM6Ly86cGFzc3dvcmRAcmVkaXM6NjM3OQ==
```

### ใช้ External Secrets (แนะนำสำหรับ Production)

```yaml
# k8s/external-secret.yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: app-secrets
  namespace: production
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secretsmanager
    kind: ClusterSecretStore
  target:
    name: app-secrets
    creationPolicy: Owner
  data:
    - secretKey: database-url
      remoteRef:
        key: myapp/production
        property: database_url
    - secretKey: jwt-secret
      remoteRef:
        key: myapp/production
        property: jwt_secret
```

---

## 5. Helm Charts

### Chart Structure

```
myapp/
├── Chart.yaml
├── values.yaml
├── templates/
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── configmap.yaml
│   ├── secret.yaml
│   └── hpa.yaml
```

### Chart.yaml

```yaml
# Chart.yaml
apiVersion: v2
name: myapp
description: My Node.js Application
type: application
version: 1.0.0
appVersion: "1.0.0"
keywords:
  - nodejs
  - express
  - api
```

### values.yaml

```yaml
# values.yaml
replicaCount: 3

image:
  repository: myregistry/nodejs-app
  pullPolicy: IfNotPresent
  tag: "1.0.0"

service:
  type: ClusterIP
  port: 80
  targetPort: 3000

ingress:
  enabled: true
  className: nginx
  hosts:
    - host: api.example.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: api-tls-secret
      hosts:
        - api.example.com

resources:
  requests:
    memory: 128Mi
    cpu: 100m
  limits:
    memory: 512Mi
    cpu: 500m

autoscaling:
  enabled: true
  minReplicas: 3
  maxReplicas: 20
  targetCPUUtilizationPercentage: 70

env:
  NODE_ENV: production
  LOG_LEVEL: info

secrets:
  existingSecret: app-secrets

livenessProbe:
  httpGet:
    path: /health
    port: 3000
  initialDelaySeconds: 30
  periodSeconds: 10

readinessProbe:
  httpGet:
    path: /ready
    port: 3000
  initialDelaySeconds: 5
  periodSeconds: 5
```

### Deployment Template

```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "myapp.fullname" . }}
  labels:
    {{- include "myapp.labels" . | nindent 4 }}
spec:
  {{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount }}
  {{- end }}
  selector:
    matchLabels:
      {{- include "myapp.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "myapp.selectorLabels" . | nindent 8 }}
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - containerPort: {{ .Values.service.targetPort }}
          env:
            {{- range $key, $value := .Values.env }}
            - name: {{ $key }}
              value: {{ $value | quote }}
            {{- end }}
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
          livenessProbe:
            {{- toYaml .Values.livenessProbe | nindent 12 }}
          readinessProbe:
            {{- toYaml .Values.readinessProbe | nindent 12 }}
```

### Helm Commands

```bash
# ติดตั้ง chart
helm install myapp ./myapp -n production --create-namespace

# Upgrade
helm upgrade myapp ./myapp -n production \
  --set image.tag=1.1.0 \
  --set replicaCount=5

# แสดง values ที่ใช้งาน
helm get values myapp -n production

# Rollback
helm rollback myapp 1 -n production

# Uninstall
helm uninstall myapp -n production

# Template rendering (dry run)
helm template myapp ./myapp -n production
```

---

## 6. Kubernetes คำสั่งสำคัญ

```bash
# ดู Pods
kubectl get pods -n production
kubectl get pods -n production -w  # watch

# ดู logs
kubectl logs -n production deployment/nodejs-app
kubectl logs -n production pod/nodejs-app-xxx --previous

# Describe resource
kubectl describe pod -n production nodejs-app-xxx

# Execute ใน Pod
kubectl exec -it -n production nodejs-app-xxx -- /bin/sh

# Scale deployment
kubectl scale deployment nodejs-app -n production --replicas=5

# Rolling restart
kubectl rollout restart deployment nodejs-app -n production

# ดู rollout status
kubectl rollout status deployment nodejs-app -n production

# Undo rollout
kubectl rollout undo deployment nodejs-app -n production

# Port forward (สำหรับ debug)
kubectl port-forward -n production service/nodejs-app-service 3000:80

# Apply/Delete resources
kubectl apply -f k8s/
kubectl delete -f k8s/

# ดู resource usage
kubectl top pods -n production
kubectl top nodes
```

---

## แบบฝึกหัด

### ระดับ 1: พื้นฐาน
Deploy app ไป Kubernetes:
- Dockerfile
- Deployment
- Service
- ConfigMap/Secret

### ระดับ 2: กลาง
เพิ่ม:
- HPA
- Ingress + TLS
- Health checks
- Resource limits

### ระดับ 3: ขั้นสูง
สร้าง Helm chart:
- Parameterizable values
- Multiple environments
- Helm hooks

---

## สรุป

Kubernetes เป็น industry standard สำหรับ container orchestration ที่ scale ได้ดี ควรเริ่มจาก Deployment + Service + ConfigMap แล้วค่อยเพิ่ม HPA และ Ingress Helm charts ช่วยจัดการ configurations สำหรับ multiple environments

> ขั้นตอนต่อไป: Part 71 - Serverless
