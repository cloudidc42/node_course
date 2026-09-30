# Part 59 | ขั้นตอนที่ 1041-1060 จาก 1000

## Kubernetes สำหรับ Node.js Applications

Kubernetes (K8s) เป็น container orchestration platform ที่ช่วยจัดการ, deploy, และ scale containerized applications

---

## ขั้นตอนที่ 1041: Kubernetes Concepts

```
Kubernetes Architecture:

Master Node (Control Plane):
  - API Server: ศูนย์กลางการสื่อสาร
  - etcd: distributed key-value store (cluster state)
  - Scheduler: กำหนดว่า Pod จะ run บน node ไหน
  - Controller Manager: รักษา desired state

Worker Nodes:
  - kubelet: agent บน node
  - kube-proxy: network rules
  - Container Runtime: Docker/containerd

Key Objects:
  - Pod: smallest deployable unit (1+ containers)
  - Service: stable network endpoint
  - Deployment: declares desired state for Pods
  - ConfigMap/Secret: configuration data
  - Ingress: HTTP routing
  - PersistentVolume: storage
```

---

## ขั้นตอนที่ 1042: Dockerfile สำหรับ Node.js

```dockerfile
# Dockerfile (Multi-stage build)

# Stage 1: Build
FROM node:18-alpine AS builder

WORKDIR /app

# Copy package files ก่อน (cache optimization)
COPY package*.json ./
RUN npm ci --only=production

# Stage 2: Production image
FROM node:18-alpine AS production

# Security: สร้าง non-root user
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nodeuser -u 1001 -G nodejs

WORKDIR /app

# Copy from builder
COPY --from=builder --chown=nodeuser:nodejs /app/node_modules ./node_modules
COPY --chown=nodeuser:nodejs . .

# ใช้ non-root user
USER nodeuser

EXPOSE 3000

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=10s --retries=3 \
  CMD wget --quiet --tries=1 --spider http://localhost:3000/health || exit 1

CMD ["node", "server.js"]
```

```javascript
// server.js - Kubernetes-ready Node.js app

const express = require('express');
const app = express();

// Health check endpoint (K8s liveness/readiness probe)
app.get('/health', (req, res) => {
  res.json({
    status: 'ok',
    uptime: process.uptime(),
    timestamp: new Date()
  });
});

// Readiness check (ตรวจว่า app พร้อมรับ traffic)
app.get('/ready', async (req, res) => {
  try {
    // ตรวจสอบ DB connection
    const mongoose = require('mongoose');
    if (mongoose.connection.readyState !== 1) {
      return res.status(503).json({ status: 'not ready', reason: 'db not connected' });
    }
    
    res.json({ status: 'ready' });
  } catch (error) {
    res.status(503).json({ status: 'not ready', error: error.message });
  }
});

// Graceful shutdown
const server = app.listen(3000, () => {
  console.log('Server running on port 3000');
});

process.on('SIGTERM', async () => {
  console.log('SIGTERM received, shutting down gracefully');
  
  server.close(() => {
    console.log('HTTP server closed');
    
    mongoose.connection.close(false, () => {
      console.log('MongoDB connection closed');
      process.exit(0);
    });
  });
  
  // Force close after 30 seconds
  setTimeout(() => {
    console.error('Forced shutdown');
    process.exit(1);
  }, 30000);
});
```

---

## ขั้นตอนที่ 1043: Kubernetes Deployment

```yaml
# k8s/deployment.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: node-api
  namespace: production
  labels:
    app: node-api
    version: v1.0.0

spec:
  replicas: 3              # จำนวน Pod instances
  
  selector:
    matchLabels:
      app: node-api
  
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1          # เพิ่ม 1 Pod ระหว่าง update
      maxUnavailable: 0    # ห้าม Pod ล่มระหว่าง update
  
  template:
    metadata:
      labels:
        app: node-api
        version: v1.0.0
    
    spec:
      containers:
      - name: node-api
        image: registry.example.com/node-api:1.0.0
        
        ports:
        - containerPort: 3000
          name: http
        
        # Environment variables
        env:
        - name: NODE_ENV
          value: "production"
        - name: PORT
          value: "3000"
        - name: MONGODB_URI
          valueFrom:
            secretKeyRef:
              name: app-secrets
              key: mongodb-uri
        - name: JWT_SECRET
          valueFrom:
            secretKeyRef:
              name: app-secrets
              key: jwt-secret
        
        # ConfigMap values
        envFrom:
        - configMapRef:
            name: app-config
        
        # Resource limits (สำคัญมาก!)
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        
        # Health probes
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
          initialDelaySeconds: 10
          periodSeconds: 5
          failureThreshold: 3
        
        # Graceful shutdown
        lifecycle:
          preStop:
            exec:
              command: ["/bin/sh", "-c", "sleep 5"]
      
      # Pod-level settings
      terminationGracePeriodSeconds: 60
      
      # Topology: กระจาย Pods ไปยัง nodes ต่างๆ
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: kubernetes.io/hostname
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
          matchLabels:
            app: node-api
```

---

## ขั้นตอนที่ 1044: Services

```yaml
# k8s/service.yaml

apiVersion: v1
kind: Service
metadata:
  name: node-api-service
  namespace: production

spec:
  selector:
    app: node-api    # เชื่อมกับ Pods ที่มี label นี้
  
  ports:
  - name: http
    port: 80
    targetPort: 3000
    protocol: TCP
  
  type: ClusterIP    # Internal only (ใช้ Ingress สำหรับ external)

---
# NodePort สำหรับ development
apiVersion: v1
kind: Service
metadata:
  name: node-api-nodeport
  namespace: development

spec:
  selector:
    app: node-api
  
  ports:
  - port: 80
    targetPort: 3000
    nodePort: 30080   # Accessible at <NodeIP>:30080
  
  type: NodePort
```

---

## ขั้นตอนที่ 1045: Ingress

```yaml
# k8s/ingress.yaml

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: node-api-ingress
  namespace: production
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/proxy-body-size: "10m"
    cert-manager.io/cluster-issuer: "letsencrypt-prod"

spec:
  tls:
  - hosts:
    - api.example.com
    secretName: api-tls
  
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /api/v1
        pathType: Prefix
        backend:
          service:
            name: node-api-service
            port:
              number: 80
      
      - path: /api/v2
        pathType: Prefix
        backend:
          service:
            name: node-api-v2-service
            port:
              number: 80
```

---

## ขั้นตอนที่ 1046: ConfigMaps และ Secrets

```yaml
# k8s/configmap.yaml

apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: production

data:
  APP_NAME: "My Node API"
  LOG_LEVEL: "info"
  RATE_LIMIT: "100"
  CORS_ORIGIN: "https://app.example.com"

---
# k8s/secret.yaml (values ต้อง base64 encoded)

apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
  namespace: production

type: Opaque

data:
  # echo -n "mongodb://..." | base64
  mongodb-uri: bW9uZ29kYjovL...
  jwt-secret: c2VjcmV0a2V5...
  redis-url: cmVkaXM6Ly8...
```

```bash
# สร้าง Secret จาก command line
kubectl create secret generic app-secrets \
  --from-literal=mongodb-uri='mongodb://...' \
  --from-literal=jwt-secret='mysecret' \
  --namespace=production

# สร้างจาก .env file
kubectl create secret generic app-secrets \
  --from-env-file=.env.production \
  --namespace=production
```

---

## ขั้นตอนที่ 1047: Horizontal Pod Autoscaler (HPA)

```yaml
# k8s/hpa.yaml

apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: node-api-hpa
  namespace: production

spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: node-api
  
  minReplicas: 2      # ขั้นต่ำ
  maxReplicas: 10     # สูงสุด
  
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70    # Scale up เมื่อ CPU > 70%
  
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 30
      policies:
      - type: Pods
        value: 2
        periodSeconds: 60
    
    scaleDown:
      stabilizationWindowSeconds: 300  # รอ 5 นาทีก่อน scale down
      policies:
      - type: Pods
        value: 1
        periodSeconds: 60
```

---

## ขั้นตอนที่ 1048: Persistent Storage สำหรับ MongoDB

```yaml
# k8s/mongodb.yaml

# PersistentVolumeClaim
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mongodb-pvc
  namespace: production

spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: standard
  resources:
    requests:
      storage: 10Gi

---
# MongoDB Deployment
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mongodb
  namespace: production

spec:
  serviceName: mongodb
  replicas: 1
  
  selector:
    matchLabels:
      app: mongodb
  
  template:
    metadata:
      labels:
        app: mongodb
    
    spec:
      containers:
      - name: mongodb
        image: mongo:6.0
        
        ports:
        - containerPort: 27017
        
        env:
        - name: MONGO_INITDB_ROOT_USERNAME
          valueFrom:
            secretKeyRef:
              name: mongodb-secrets
              key: root-username
        - name: MONGO_INITDB_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mongodb-secrets
              key: root-password
        
        volumeMounts:
        - name: mongodb-storage
          mountPath: /data/db
        
        resources:
          requests:
            memory: "512Mi"
            cpu: "250m"
          limits:
            memory: "2Gi"
            cpu: "1"
  
  volumeClaimTemplates:
  - metadata:
      name: mongodb-storage
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 10Gi
```

---

## ขั้นตอนที่ 1049: Namespace Organization

```yaml
# k8s/namespaces.yaml

apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    environment: production

---
apiVersion: v1
kind: Namespace
metadata:
  name: staging
  labels:
    environment: staging

---
apiVersion: v1
kind: Namespace
metadata:
  name: development
  labels:
    environment: development

---
# Resource Quotas สำหรับแต่ละ namespace
apiVersion: v1
kind: ResourceQuota
metadata:
  name: production-quota
  namespace: production

spec:
  hard:
    requests.cpu: "4"
    requests.memory: 8Gi
    limits.cpu: "8"
    limits.memory: 16Gi
    pods: "20"
```

---

## ขั้นตอนที่ 1050: Kubernetes CI/CD Pipeline

```yaml
# .github/workflows/k8s-deploy.yml

name: Deploy to Kubernetes

on:
  push:
    branches: [main]

env:
  REGISTRY: registry.example.com
  IMAGE_NAME: node-api

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Login to registry
      run: echo "${{ secrets.REGISTRY_PASSWORD }}" | docker login $REGISTRY -u ${{ secrets.REGISTRY_USERNAME }} --password-stdin
    
    - name: Build and push
      run: |
        IMAGE_TAG=${REGISTRY}/${IMAGE_NAME}:${GITHUB_SHA::8}
        docker build -t $IMAGE_TAG .
        docker push $IMAGE_TAG
        echo "IMAGE_TAG=$IMAGE_TAG" >> $GITHUB_ENV
    
    - name: Deploy to K8s
      run: |
        # Update image in deployment
        kubectl set image deployment/node-api \
          node-api=${{ env.IMAGE_TAG }} \
          --namespace=production
        
        # Wait for rollout
        kubectl rollout status deployment/node-api \
          --namespace=production \
          --timeout=300s
      
      env:
        KUBECONFIG_DATA: ${{ secrets.KUBECONFIG }}
```

---

## ขั้นตอนที่ 1051: Helm Charts

```yaml
# helm/node-api/Chart.yaml

apiVersion: v2
name: node-api
description: Node.js API Helm chart
version: 1.0.0
appVersion: "1.0.0"
```

```yaml
# helm/node-api/values.yaml

replicaCount: 3

image:
  repository: registry.example.com/node-api
  tag: "latest"
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80
  targetPort: 3000

ingress:
  enabled: true
  className: nginx
  host: api.example.com
  tls: true

resources:
  requests:
    memory: "128Mi"
    cpu: "100m"
  limits:
    memory: "512Mi"
    cpu: "500m"

autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70

env:
  NODE_ENV: production
  LOG_LEVEL: info

secrets:
  mongodbUri: ""  # Set during deploy
  jwtSecret: ""
```

```yaml
# helm/node-api/templates/deployment.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "node-api.fullname" . }}
  labels:
    {{- include "node-api.labels" . | nindent 4 }}

spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      {{- include "node-api.selectorLabels" . | nindent 6 }}
  
  template:
    metadata:
      labels:
        {{- include "node-api.selectorLabels" . | nindent 8 }}
    
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
          {{- toYaml .Values.resources | nindent 10 }}
        
        livenessProbe:
          httpGet:
            path: /health
            port: {{ .Values.service.targetPort }}
          initialDelaySeconds: 30
          periodSeconds: 10
        
        readinessProbe:
          httpGet:
            path: /ready
            port: {{ .Values.service.targetPort }}
          initialDelaySeconds: 10
          periodSeconds: 5
```

---

## ขั้นตอนที่ 1052: Pod Disruption Budget

```yaml
# k8s/pdb.yaml

apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: node-api-pdb
  namespace: production

spec:
  minAvailable: 2          # ต้องมีอย่างน้อย 2 Pods available
  # หรือ: maxUnavailable: 1
  
  selector:
    matchLabels:
      app: node-api
```

---

## ขั้นตอนที่ 1053: Network Policies

```yaml
# k8s/network-policy.yaml

# อนุญาต traffic เฉพาะจาก ingress controller และ monitoring
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: node-api-network-policy
  namespace: production

spec:
  podSelector:
    matchLabels:
      app: node-api
  
  policyTypes:
  - Ingress
  - Egress
  
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: ingress-nginx
    ports:
    - protocol: TCP
      port: 3000
  
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: mongodb
    ports:
    - protocol: TCP
      port: 27017
  
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
    ports:
    - protocol: UDP
      port: 53
```

---

## ขั้นตอนที่ 1054: Monitoring with Prometheus

```javascript
// lib/metrics.js
// Prometheus metrics สำหรับ Node.js

const prometheus = require('prom-client');

// Collect default metrics (CPU, Memory, etc.)
prometheus.collectDefaultMetrics({
  prefix: 'node_api_'
});

// Custom metrics
const httpRequestDuration = new prometheus.Histogram({
  name: 'http_request_duration_seconds',
  help: 'Duration of HTTP requests in seconds',
  labelNames: ['method', 'route', 'status_code'],
  buckets: [0.001, 0.01, 0.05, 0.1, 0.5, 1, 2, 5]
});

const httpRequestTotal = new prometheus.Counter({
  name: 'http_requests_total',
  help: 'Total HTTP requests',
  labelNames: ['method', 'route', 'status_code']
});

const activeConnections = new prometheus.Gauge({
  name: 'active_connections',
  help: 'Number of active connections'
});

// Express middleware
function metricsMiddleware(req, res, next) {
  const start = Date.now();
  
  res.on('finish', () => {
    const duration = (Date.now() - start) / 1000;
    const route = req.route?.path || 'unknown';
    
    httpRequestDuration
      .labels(req.method, route, res.statusCode)
      .observe(duration);
    
    httpRequestTotal
      .labels(req.method, route, res.statusCode)
      .inc();
  });
  
  next();
}

// Metrics endpoint
async function metricsHandler(req, res) {
  res.set('Content-Type', prometheus.register.contentType);
  res.end(await prometheus.register.metrics());
}

module.exports = { metricsMiddleware, metricsHandler, activeConnections };
```

```yaml
# k8s/servicemonitor.yaml (Prometheus Operator)

apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: node-api-monitor
  namespace: monitoring

spec:
  selector:
    matchLabels:
      app: node-api
  
  endpoints:
  - port: http
    path: /metrics
    interval: 30s
```

---

## ขั้นตอนที่ 1055: Logging with EFK Stack

```javascript
// lib/logger.js
// Structured logging สำหรับ Kubernetes

const winston = require('winston');

const logger = winston.createLogger({
  level: process.env.LOG_LEVEL || 'info',
  format: winston.format.combine(
    winston.format.timestamp(),
    winston.format.errors({ stack: true }),
    winston.format.json() // JSON format สำหรับ log aggregation
  ),
  defaultMeta: {
    service: process.env.APP_NAME || 'node-api',
    version: process.env.APP_VERSION || '1.0.0',
    pod: process.env.HOSTNAME // K8s sets HOSTNAME to pod name
  },
  transports: [
    new winston.transports.Console()
  ]
});

// Log middleware
function requestLogger(req, res, next) {
  const startTime = Date.now();
  
  res.on('finish', () => {
    logger.info('HTTP Request', {
      method: req.method,
      path: req.path,
      statusCode: res.statusCode,
      duration: Date.now() - startTime,
      userAgent: req.headers['user-agent'],
      requestId: req.headers['x-request-id']
    });
  });
  
  next();
}

module.exports = { logger, requestLogger };
```

---

## ขั้นตอนที่ 1056: Rolling Updates & Rollbacks

```bash
# Deploy new version
kubectl set image deployment/node-api \
  node-api=registry.example.com/node-api:1.1.0 \
  --namespace=production

# ดู rollout status
kubectl rollout status deployment/node-api --namespace=production

# ดู rollout history
kubectl rollout history deployment/node-api --namespace=production

# Rollback ไปยัง version ก่อนหน้า
kubectl rollout undo deployment/node-api --namespace=production

# Rollback ไปยัง revision เฉพาะ
kubectl rollout undo deployment/node-api \
  --to-revision=2 \
  --namespace=production

# Pause rollout (ระหว่าง canary)
kubectl rollout pause deployment/node-api --namespace=production
kubectl rollout resume deployment/node-api --namespace=production
```

---

## ขั้นตอนที่ 1057: Job และ CronJob

```yaml
# k8s/cronjob.yaml

apiVersion: batch/v1
kind: CronJob
metadata:
  name: daily-report
  namespace: production

spec:
  schedule: "0 8 * * *"   # 8am ทุกวัน
  
  concurrencyPolicy: Forbid       # ไม่ run ซ้อนกัน
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 3
  
  jobTemplate:
    spec:
      backoffLimit: 2    # retry 2 ครั้ง
      
      template:
        spec:
          restartPolicy: OnFailure
          
          containers:
          - name: report-job
            image: registry.example.com/node-api:latest
            
            command: ["node", "scripts/generateReport.js"]
            
            env:
            - name: MONGODB_URI
              valueFrom:
                secretKeyRef:
                  name: app-secrets
                  key: mongodb-uri
            
            resources:
              limits:
                memory: "256Mi"
                cpu: "200m"
```

---

## ขั้นตอนที่ 1058: Init Containers

```yaml
# k8s/deployment-with-init.yaml
# Init containers รัน setup tasks ก่อน main container

spec:
  initContainers:
  
  # รอ MongoDB ก่อน start
  - name: wait-for-mongodb
    image: busybox
    command: ['sh', '-c', 'until nc -z mongodb-service 27017; do echo "Waiting for MongoDB..."; sleep 2; done']
  
  # Run database migrations
  - name: run-migrations
    image: registry.example.com/node-api:latest
    command: ['node', 'scripts/migrate.js']
    env:
    - name: MONGODB_URI
      valueFrom:
        secretKeyRef:
          name: app-secrets
          key: mongodb-uri
  
  containers:
  - name: node-api
    image: registry.example.com/node-api:latest
    # ... main container
```

---

## ขั้นตอนที่ 1059: Service Mesh กับ Istio

```yaml
# istio/virtual-service.yaml
# Canary deployment ด้วย Istio

apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: node-api-vs
  namespace: production

spec:
  hosts:
  - node-api-service
  
  http:
  - match:
    - headers:
        x-canary:
          exact: "true"
    route:
    - destination:
        host: node-api-service
        subset: v2
  
  - route:
    - destination:
        host: node-api-service
        subset: v1
      weight: 90    # 90% ไปยัง v1
    
    - destination:
        host: node-api-service
        subset: v2
      weight: 10    # 10% ไปยัง v2 (canary)

---
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: node-api-dr

spec:
  host: node-api-service
  
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
  
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 100
      http:
        http2MaxRequests: 1000
    
    outlierDetection:
      consecutiveErrors: 5
      interval: 10s
      baseEjectionTime: 30s
```

---

## ขั้นตอนที่ 1060: Kubernetes Production Checklist

```yaml
# production-checklist.yaml
# ตรวจสอบก่อน deploy production

# 1. Resource Limits - ต้องกำหนดเสมอ
resources:
  requests:
    memory: "128Mi"
    cpu: "100m"
  limits:
    memory: "512Mi"
    cpu: "500m"

# 2. Health Probes - liveness และ readiness
livenessProbe: ...
readinessProbe: ...

# 3. Graceful Shutdown
lifecycle:
  preStop:
    exec:
      command: ["/bin/sh", "-c", "sleep 5"]
terminationGracePeriodSeconds: 60

# 4. Pod Disruption Budget
# PDB: minAvailable: 2

# 5. Anti-affinity
affinity:
  podAntiAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
    - weight: 100
      podAffinityTerm:
        topologyKey: kubernetes.io/hostname
        labelSelector:
          matchLabels:
            app: node-api

# 6. Security Context
securityContext:
  runAsNonRoot: true
  runAsUser: 1001
  readOnlyRootFilesystem: true
  allowPrivilegeEscalation: false

# 7. NetworkPolicy - จำกัด traffic

# 8. HPA - auto scaling

# 9. Monitoring - ServiceMonitor, alerts

# 10. Multiple replicas - minReplicas: 2
```

```bash
# Useful kubectl commands
kubectl get pods --namespace=production
kubectl describe pod <pod-name> --namespace=production
kubectl logs <pod-name> --namespace=production --follow
kubectl exec -it <pod-name> -- /bin/sh
kubectl top pods --namespace=production
kubectl get events --namespace=production --sort-by=.metadata.creationTimestamp
```

---

## แบบฝึกหัดท้ายบท

### แบบฝึกหัดที่ 1: Deploy Node.js API
Deploy Node.js API พร้อม MongoDB บน Kubernetes local (minikube)

### แบบฝึกหัดที่ 2: HPA Configuration
ตั้งค่า HPA และทดสอบ auto-scaling ด้วย load testing

### แบบฝึกหัดที่ 3: Rolling Update
ทำ rolling update โดยไม่มี downtime และ rollback เมื่อมีปัญหา

### แบบฝึกหัดที่ 4: Helm Chart
สร้าง Helm chart และ deploy ไปยัง environments ต่างๆ

### แบบฝึกหัดที่ 5: Monitoring Setup
ตั้งค่า Prometheus + Grafana สำหรับ Node.js application monitoring

---

## สรุป

Kubernetes ช่วยจัดการ containerized applications ในระดับ production โดย key concepts คือ Deployments, Services, Ingress, ConfigMaps/Secrets, และ HPA การทำ health checks ที่ถูกต้อง, resource limits, และ graceful shutdown เป็นสิ่งจำเป็นสำหรับ production-ready Node.js applications บน Kubernetes
