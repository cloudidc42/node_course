# Part 73 | ขั้นตอนที่ 1261-1280 จาก 1000+

# Service Mesh Concepts

---

## 🎯 เป้าหมายของ Part นี้

หลังจากเรียนจบ Part นี้ คุณจะสามารถ:
1. เข้าใจ Service Mesh architecture
2. ติดตั้ง Istio บน Kubernetes
3. implement Traffic Management
4. ใช้ mTLS สำหรับ service communication
5. ตั้งค่า Observability ด้วย Service Mesh
6. implement resilience patterns

---

## ขั้นตอนที่ 1261: Service Mesh คืออะไร?

Service Mesh เป็น infrastructure layer สำหรับจัดการ service-to-service communication โดยใช้ sidecar proxies

```
Service Mesh Architecture:
┌─────────────────────────────────────────────────────┐
│  Pod A                    Pod B                      │
│  ┌─────────────────────┐  ┌─────────────────────┐   │
│  │ App Container    │  │  │ App Container    │  │   │
│  │    [Service A]   │  │  │    [Service B]   │  │   │
│  └──────────────────┘  │  └──────────────────┘  │   │
│  ┌──────────────────┐  │  ┌──────────────────┐  │   │
│  │ Sidecar Proxy    │  │  │ Sidecar Proxy    │  │   │
│  │    [Envoy]       │◄─┼──►    [Envoy]       │  │   │
│  └──────────────────┘  │  └──────────────────┘  │   │
│  └─────────────────────┘  └─────────────────────┘   │
│                                                      │
│  ┌──────────────────────────────────────────────┐   │
│  │           Control Plane (Istiod)              │   │
│  │  Config  |  Service Discovery  |  Certificates│   │
│  └──────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────┘
```

---

## ขั้นตอนที่ 1262: Service Mesh Features

```
Service Mesh Capabilities:
                                                    
Traffic Management:                                 
  - Load balancing                                  
  - Traffic routing                                 
  - Canary deployments                              
  - A/B testing                                     
  - Circuit breaking                                
  - Retries and timeouts                            
                                                    
Security:                                           
  - mTLS (mutual TLS)                               
  - Certificate management                          
  - Authorization policies                          
  - Traffic encryption                              
                                                    
Observability:                                      
  - Distributed tracing                             
  - Metrics collection                              
  - Access logs                                     
  - Service topology                                
```

---

## ขั้นตอนที่ 1263: ติดตั้ง Istio

```bash
# Download Istio
curl -L https://istio.io/downloadIstio | sh -
cd istio-1.x.x
export PATH=$PWD/bin:$PATH

# Install Istio in Kubernetes
istioctl install --set profile=demo -y

# Enable automatic sidecar injection for namespace
kubectl label namespace default istio-injection=enabled

# Verify installation
kubectl get pods -n istio-system

# Install addons (Prometheus, Grafana, Jaeger, Kiali)
kubectl apply -f samples/addons/

# Access Kiali dashboard
istioctl dashboard kiali

# Access Grafana
istioctl dashboard grafana

# Access Jaeger
istioctl dashboard jaeger
```

---

## ขั้นตอนที่ 1264: Virtual Services

```yaml
# virtual-service.yaml - Traffic routing rules
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: product-service-vs
spec:
  hosts:
    - product-service
  http:
    # Route 90% traffic to v1, 10% to v2 (canary)
    - match:
        - headers:
            x-canary:
              exact: "true"
      route:
        - destination:
            host: product-service
            subset: v2
    - route:
        - destination:
            host: product-service
            subset: v1
          weight: 90
        - destination:
            host: product-service
            subset: v2
          weight: 10
    
    # Retry configuration
    retries:
      attempts: 3
      perTryTimeout: 2s
      retryOn: "5xx,reset,connect-failure"
    
    # Timeout
    timeout: 5s
    
    # Fault injection for testing
    fault:
      abort:
        httpStatus: 500
        percentage:
          value: 5  # 5% of requests return 500
      delay:
        fixedDelay: 2s
        percentage:
          value: 10 # 10% of requests delayed
```

---

## ขั้นตอนที่ 1265: Destination Rules

```yaml
# destination-rule.yaml - Load balancing and circuit breaking
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: product-service-dr
spec:
  host: product-service
  
  trafficPolicy:
    # Load balancing
    loadBalancer:
      simple: LEAST_REQUEST
    
    # Connection pool settings
    connectionPool:
      tcp:
        maxConnections: 100
        connectTimeout: 3s
      http:
        h2UpgradePolicy: UPGRADE
        http1MaxPendingRequests: 1000
        http2MaxRequests: 1000
    
    # Circuit breaker (outlier detection)
    outlierDetection:
      consecutiveGatewayErrors: 5
      interval: 30s
      baseEjectionTime: 30s
      maxEjectionPercent: 50
  
  # Version subsets
  subsets:
    - name: v1
      labels:
        version: v1
      trafficPolicy:
        loadBalancer:
          simple: RANDOM
    - name: v2
      labels:
        version: v2
```

---

## ขั้นตอนที่ 1266: mTLS Configuration

```yaml
# peer-authentication.yaml - Enable mTLS
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: production
spec:
  mtls:
    mode: STRICT  # Require mTLS for all services
```

```yaml
# Allow specific service without mTLS (legacy)
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: legacy-service-policy
  namespace: default
spec:
  selector:
    matchLabels:
      app: legacy-service
  mtls:
    mode: PERMISSIVE  # Allow both TLS and plain text
```

```yaml
# Authorization policy
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: order-service-policy
  namespace: default
spec:
  selector:
    matchLabels:
      app: order-service
  rules:
    # Allow user-service to call GET/POST
    - from:
        - source:
            principals:
              - "cluster.local/ns/default/sa/user-service"
      to:
        - operation:
            methods: ["GET", "POST"]
            paths: ["/api/orders/*"]
    
    # Allow admin-service all operations
    - from:
        - source:
            principals:
              - "cluster.local/ns/default/sa/admin-service"
    
    # Deny all others by default (implicit)
```

---

## ขั้นตอนที่ 1267: Traffic Management - Canary Deployments

```yaml
# canary-deployment.yaml

# Deployment v1 (stable)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: product-service-v1
spec:
  replicas: 3
  selector:
    matchLabels:
      app: product-service
      version: v1
  template:
    metadata:
      labels:
        app: product-service
        version: v1
    spec:
      containers:
        - name: product-service
          image: my-registry/product-service:v1
          ports:
            - containerPort: 3000

---
# Deployment v2 (canary)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: product-service-v2
spec:
  replicas: 1
  selector:
    matchLabels:
      app: product-service
      version: v2
  template:
    metadata:
      labels:
        app: product-service
        version: v2
    spec:
      containers:
        - name: product-service
          image: my-registry/product-service:v2
          ports:
            - containerPort: 3000

---
# Service (routes to both versions)
apiVersion: v1
kind: Service
metadata:
  name: product-service
spec:
  selector:
    app: product-service  # No version - routes to both
  ports:
    - port: 80
      targetPort: 3000
```

---

## ขั้นตอนที่ 1268: Sidecar Proxy กับ Node.js

```typescript
// src/service-mesh/health-check.ts
// Service mesh typically handles retries and circuit breaking
// But Node.js app should still have proper health endpoints

import express from "express";

const app = express();

// Liveness probe - is the app alive?
app.get("/health/live", (req, res) => {
  res.json({ status: "alive" });
});

// Readiness probe - is the app ready to serve traffic?
app.get("/health/ready", async (req, res) => {
  try {
    // Check critical dependencies
    await checkDatabase();
    await checkCache();
    
    res.json({ status: "ready" });
  } catch (error) {
    res.status(503).json({ 
      status: "not ready",
      reason: (error as Error).message
    });
  }
});

// Startup probe - has the app finished initializing?
app.get("/health/startup", (req, res) => {
  if (isInitialized) {
    res.json({ status: "started" });
  } else {
    res.status(503).json({ status: "starting" });
  }
});

let isInitialized = false;

async function initialize() {
  await connectDatabase();
  await loadConfigurations();
  isInitialized = true;
}

async function checkDatabase(): Promise<void> { }
async function checkCache(): Promise<void> { }
async function connectDatabase(): Promise<void> { }
async function loadConfigurations(): Promise<void> { }

initialize();
```

---

## ขั้นตอนที่ 1269: Service Communication Patterns

```typescript
// src/services/base-client.ts
// When using service mesh, the sidecar handles retries and circuit breaking
// But we still need to handle application-level errors

import axios, { AxiosInstance, AxiosRequestConfig } from "axios";
import { Logger } from "@nestjs/common";

export class ServiceClient {
  protected readonly logger = new Logger(ServiceClient.name);
  protected readonly http: AxiosInstance;

  constructor(
    protected readonly serviceName: string,
    baseURL: string
  ) {
    this.http = axios.create({
      baseURL,
      timeout: 5000,  // Service mesh handles retries, but we set app-level timeout
      headers: {
        "Content-Type": "application/json"
      }
    });

    // Add request logging
    this.http.interceptors.request.use((config) => {
      this.logger.debug(`→ ${config.method?.toUpperCase()} ${config.url}`);
      return config;
    });

    // Add response logging
    this.http.interceptors.response.use(
      (response) => {
        this.logger.debug(`← ${response.status} ${response.config.url}`);
        return response;
      },
      (error) => {
        this.logger.error(
          `← Error ${error.response?.status} ${error.config?.url}: ${error.message}`
        );
        return Promise.reject(error);
      }
    );
  }

  protected async get<T>(path: string, config?: AxiosRequestConfig): Promise<T> {
    const response = await this.http.get<T>(path, config);
    return response.data;
  }

  protected async post<T>(path: string, data?: any): Promise<T> {
    const response = await this.http.post<T>(path, data);
    return response.data;
  }
}

// Specific service client
export class OrderServiceClient extends ServiceClient {
  constructor() {
    super("order-service", process.env.ORDER_SERVICE_URL ?? "http://order-service");
  }

  async getOrder(orderId: string) {
    return this.get<any>(`/orders/${orderId}`);
  }

  async createOrder(data: any) {
    return this.post<any>("/orders", data);
  }
}
```

---

## ขั้นตอนที่ 1270: Observability with Istio

```yaml
# Telemetry configuration
apiVersion: telemetry.istio.io/v1alpha1
kind: Telemetry
metadata:
  name: default
  namespace: default
spec:
  # Access logging
  accessLogging:
    - providers:
        - name: envoy
      filter:
        expression: "response.code >= 400"  # Log only errors

  # Tracing
  tracing:
    - providers:
        - name: zipkin
      randomSamplingPercentage: 10  # Sample 10% of requests
      customTags:
        environment:
          literal:
            value: production
        user-agent:
          header:
            name: user-agent
  
  # Metrics
  metrics:
    - providers:
        - name: prometheus
      overrides:
        - match:
            mode: CLIENT_AND_SERVER
          tagOverrides:
            destination_service_name:
              operation: UPSERT
              value: "destination.service.name"
```

---

## ขั้นตอนที่ 1271: Gateway API

```yaml
# gateway.yaml - Ingress configuration
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: my-gateway
spec:
  selector:
    istio: ingressgateway
  servers:
    - port:
        number: 80
        name: http
        protocol: HTTP
      hosts:
        - "api.example.com"
      tls:
        httpsRedirect: true  # Redirect HTTP to HTTPS
    
    - port:
        number: 443
        name: https
        protocol: HTTPS
      tls:
        mode: SIMPLE
        credentialName: api-tls-secret
      hosts:
        - "api.example.com"

---
# Virtual Service for the gateway
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: api-gateway-vs
spec:
  hosts:
    - "api.example.com"
  gateways:
    - my-gateway
  http:
    - match:
        - uri:
            prefix: /api/v1/users
      route:
        - destination:
            host: user-service
            port:
              number: 80
    
    - match:
        - uri:
            prefix: /api/v1/orders
      route:
        - destination:
            host: order-service
            port:
              number: 80
```

---

## ขั้นตอนที่ 1272: Service Mesh Monitoring

```typescript
// src/monitoring/service-mesh-metrics.ts
// These metrics are automatically collected by Istio/Envoy
// But we can add application-level metrics too

import * as promClient from "prom-client";

// Business metrics (not covered by service mesh)
const orderRevenue = new promClient.Gauge({
  name: "business_order_revenue_total",
  help: "Total order revenue",
  labelNames: ["currency", "status"]
});

const activeUsers = new promClient.Gauge({
  name: "business_active_users",
  help: "Number of active users"
});

const productViewCount = new promClient.Counter({
  name: "business_product_views_total",
  help: "Total product view count",
  labelNames: ["product_id", "category"]
});

// Service mesh collects these automatically:
// - istio_requests_total
// - istio_request_duration_milliseconds
// - istio_request_bytes
// - istio_response_bytes
// - istio_tcp_connections_opened_total
// - istio_tcp_connections_closed_total
```

---

## ขั้นตอนที่ 1273: Traffic Splitting

```yaml
# Progressive delivery - split traffic gradually
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: progressive-delivery
spec:
  hosts:
    - product-service
  http:
    # Phase 1: 5% to v2
    - match:
        - headers:
            x-user-id:
              regex: ".*[0-4]$"  # Users whose ID ends in 0-4
      route:
        - destination:
            host: product-service
            subset: v2
    # Default: v1
    - route:
        - destination:
            host: product-service
            subset: v1
```

---

## ขั้นตอนที่ 1274: Service Entry

```yaml
# service-entry.yaml - Register external services
apiVersion: networking.istio.io/v1alpha3
kind: ServiceEntry
metadata:
  name: external-payment-api
spec:
  hosts:
    - payment-api.external.com
  ports:
    - number: 443
      name: https
      protocol: HTTPS
  location: MESH_EXTERNAL
  resolution: DNS

---
# Allow egress to external service
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: payment-api-vs
spec:
  hosts:
    - payment-api.external.com
  http:
    - timeout: 10s
      retries:
        attempts: 3
        perTryTimeout: 3s
      route:
        - destination:
            host: payment-api.external.com
            port:
              number: 443
```

---

## ขั้นตอนที่ 1275: Linkerd Alternative

```bash
# ติดตั้ง Linkerd (lightweight alternative to Istio)
curl --proto '=https' --tlsv1.2 -sSfL https://run.linkerd.io/install | sh
export PATH=$PATH:$HOME/.linkerd2/bin

# Check pre-conditions
linkerd check --pre

# Install Linkerd
linkerd install --crds | kubectl apply -f -
linkerd install | kubectl apply -f -

# Verify
linkerd check

# Add viz extension
linkerd viz install | kubectl apply -f -
linkerd viz check

# Open dashboard
linkerd viz dashboard
```

```yaml
# Enable Linkerd for a deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
  annotations:
    linkerd.io/inject: enabled
spec:
  template:
    metadata:
      annotations:
        linkerd.io/inject: enabled
```

---

## ขั้นตอนที่ 1276: Debugging Service Mesh

```bash
# Check Istio proxy status
istioctl proxy-status

# Check config for a pod
istioctl proxy-config all <pod-name> -n default

# Check routes
istioctl proxy-config routes <pod-name> -n default --name 80

# Check clusters
istioctl proxy-config clusters <pod-name> -n default

# Check listeners
istioctl proxy-config listeners <pod-name> -n default

# Analyze configuration
istioctl analyze

# Get Envoy access logs
kubectl logs <pod-name> -c istio-proxy

# Trace a request
kubectl exec -it <pod-name> -c istio-proxy -- curl http://product-service/health

# Check mTLS status
istioctl x check-inject -n default
```

---

## ขั้นตอนที่ 1277: Chaos Testing with Service Mesh

```yaml
# Inject faults for testing
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: chaos-test
spec:
  hosts:
    - payment-service
  http:
    - fault:
        # Abort 10% of requests with 503
        abort:
          httpStatus: 503
          percentage:
            value: 10
        # Add 2s delay to 20% of requests
        delay:
          fixedDelay: 2s
          percentage:
            value: 20
      route:
        - destination:
            host: payment-service
```

---

## ขั้นตอนที่ 1278: Service Mesh with Node.js Graceful Shutdown

```typescript
// src/server.ts - Proper shutdown for service mesh

import express from "express";

const app = express();
let isShuttingDown = false;

// Readiness probe - return 503 when shutting down
app.get("/health/ready", (req, res) => {
  if (isShuttingDown) {
    return res.status(503).json({ status: "shutting_down" });
  }
  res.json({ status: "ready" });
});

const server = app.listen(3000);

async function gracefulShutdown(signal: string) {
  console.log(`Received ${signal}, starting graceful shutdown...`);
  
  // Signal readiness probe to return 503
  // Service mesh will stop routing traffic here
  isShuttingDown = true;
  
  // Wait for ongoing requests to complete (service mesh drain time)
  // Istio default preStop sleep is 5s
  await new Promise(r => setTimeout(r, 5000));
  
  // Stop accepting new connections
  server.close(async () => {
    console.log("HTTP server closed");
    
    // Cleanup resources
    await disconnectDatabase();
    
    console.log("Shutdown complete");
    process.exit(0);
  });
  
  // Force close after 30s
  setTimeout(() => {
    console.error("Forced shutdown");
    process.exit(1);
  }, 30000);
}

process.on("SIGTERM", () => gracefulShutdown("SIGTERM"));
process.on("SIGINT", () => gracefulShutdown("SIGINT"));

async function disconnectDatabase(): Promise<void> {}
```

---

## ขั้นตอนที่ 1279: Kubernetes Deployment with Istio

```yaml
# deployment.yaml - Kubernetes deployment for service mesh
apiVersion: apps/v1
kind: Deployment
metadata:
  name: product-service
  labels:
    app: product-service
    version: v1
spec:
  replicas: 3
  selector:
    matchLabels:
      app: product-service
      version: v1
  template:
    metadata:
      labels:
        app: product-service
        version: v1
      annotations:
        # Istio annotations
        sidecar.istio.io/proxyCPU: "100m"
        sidecar.istio.io/proxyMemory: "128Mi"
        # Exclude health check ports from Istio proxy
        traffic.sidecar.istio.io/excludeInboundPorts: "9090"
    spec:
      serviceAccountName: product-service
      containers:
        - name: product-service
          image: my-registry/product-service:v1
          ports:
            - name: http
              containerPort: 3000
            - name: metrics
              containerPort: 9090
          env:
            - name: NODE_ENV
              value: production
          resources:
            requests:
              cpu: "250m"
              memory: "256Mi"
            limits:
              cpu: "500m"
              memory: "512Mi"
          livenessProbe:
            httpGet:
              path: /health/live
              port: 3000
            initialDelaySeconds: 30
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /health/ready
              port: 3000
            initialDelaySeconds: 5
            periodSeconds: 5
          lifecycle:
            preStop:
              exec:
                command: ["/bin/sleep", "5"]  # Wait for Istio to drain
```

---

## ขั้นตอนที่ 1280: Service Mesh vs Direct Communication

```typescript
// Direct service communication (without service mesh)
// Pros: Simpler, less overhead
// Cons: Manual retry logic, no encryption, manual observability

@Injectable()
export class DirectOrderService {
  async getOrder(orderId: string) {
    // Manual retry logic needed
    for (let i = 0; i < 3; i++) {
      try {
        const response = await axios.get(
          `http://order-service/orders/${orderId}`,
          { timeout: 5000 }
        );
        return response.data;
      } catch (error) {
        if (i === 2) throw error;
        await new Promise(r => setTimeout(r, 1000 * (i + 1)));
      }
    }
  }
}

// With service mesh (recommended for production)
// Pros: Automatic retry, mTLS, observability, circuit breaking
// Cons: Added complexity, resource overhead

@Injectable()
export class MeshOrderService {
  async getOrder(orderId: string) {
    // No retry logic needed - Istio handles it
    // No TLS cert management - Istio handles mTLS
    // Automatically observed - Istio collects metrics
    const response = await axios.get(
      `http://order-service/orders/${orderId}`,
      { timeout: 5000 }  // App-level timeout
    );
    return response.data;
  }
}
```

---

## 🏋️ แบบฝึกหัด

### แบบฝึกหัดที่ 1: Istio Setup
1. ติดตั้ง Minikube และ Istio
2. Deploy 2 services
3. กำหนดค่า Virtual Service

### แบบฝึกหัดที่ 2: Canary Deployment
1. Deploy v1 และ v2 ของ service
2. Route 10% traffic ไปยัง v2
3. Monitor metrics

### แบบฝึกหัดที่ 3: Security
1. Enable mTLS ระหว่าง services
2. สร้าง Authorization Policy
3. ทดสอบ policy enforcement

---

## 📚 สรุป

ใน Part นี้เราได้เรียนรู้:
- Service Mesh architecture
- Istio components
- Virtual Services และ Destination Rules
- mTLS สำหรับ service security
- Traffic management: canary, A/B testing
- Observability ด้วย Istio
- Chaos testing กับ service mesh
- Graceful shutdown

**Part ถัดไป**: Chaos Engineering
