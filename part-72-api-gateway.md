# Part 72 | ขั้นตอนที่ 1241-1260 จาก 1000+

# API Gateway Patterns

---

## 🎯 เป้าหมายของ Part นี้

หลังจากเรียนจบ Part นี้ คุณจะสามารถ:
1. เข้าใจ API Gateway patterns
2. ติดตั้งและกำหนดค่า Kong API Gateway
3. ใช้ Nginx เป็น API Gateway
4. implement Request Routing strategies
5. สร้าง custom API Gateway ด้วย Node.js
6. implement Rate Limiting, Authentication, Load Balancing

---

## ขั้นตอนที่ 1241: API Gateway คืออะไร?

API Gateway เป็น single entry point สำหรับ client requests ในระบบ microservices ช่วยจัดการ routing, authentication, rate limiting และ monitoring

```
Without API Gateway:
Client → Service A (port 3001)
Client → Service B (port 3002)
Client → Service C (port 3003)

With API Gateway:
Client → API Gateway (port 80/443)
             ├── /users → User Service
             ├── /orders → Order Service
             ├── /products → Product Service
             └── /payments → Payment Service

Benefits:
- Single entry point
- Cross-cutting concerns
- Service decoupling
- Security enforcement
```

---

## ขั้นตอนที่ 1242: Kong API Gateway

```bash
# ติดตั้ง Kong ด้วย Docker
docker-compose -f docker-compose.kong.yml up -d
```

```yaml
# docker-compose.kong.yml
version: "3"
services:
  kong-db:
    image: postgres:15
    environment:
      POSTGRES_USER: kong
      POSTGRES_DB: kong
      POSTGRES_PASSWORD: kongpass
    
  kong-migration:
    image: kong:3.4
    command: kong migrations bootstrap
    environment:
      KONG_DATABASE: postgres
      KONG_PG_HOST: kong-db
      KONG_PG_USER: kong
      KONG_PG_PASSWORD: kongpass
    depends_on:
      - kong-db

  kong:
    image: kong:3.4
    environment:
      KONG_DATABASE: postgres
      KONG_PG_HOST: kong-db
      KONG_PG_USER: kong
      KONG_PG_PASSWORD: kongpass
      KONG_PROXY_ACCESS_LOG: /dev/stdout
      KONG_ADMIN_ACCESS_LOG: /dev/stdout
      KONG_PROXY_ERROR_LOG: /dev/stderr
      KONG_ADMIN_ERROR_LOG: /dev/stderr
      KONG_ADMIN_LISTEN: 0.0.0.0:8001
    ports:
      - "80:8000"
      - "443:8443"
      - "8001:8001"
      - "8444:8444"
    depends_on:
      - kong-db

  konga:
    image: pantsel/konga
    environment:
      NODE_ENV: development
    ports:
      - "1337:1337"
    depends_on:
      - kong
```

---

## ขั้นตอนที่ 1243: Kong Configuration

```bash
# สร้าง Service ใน Kong
curl -i -X POST http://localhost:8001/services \
  --data "name=user-service" \
  --data "url=http://user-service:3001"

# สร้าง Route
curl -i -X POST http://localhost:8001/services/user-service/routes \
  --data "name=user-routes" \
  --data "paths[]=/users" \
  --data "methods[]=GET" \
  --data "methods[]=POST"

# เพิ่ม Rate Limiting plugin
curl -i -X POST http://localhost:8001/services/user-service/plugins \
  --data "name=rate-limiting" \
  --data "config.minute=100" \
  --data "config.hour=1000" \
  --data "config.policy=local"

# เพิ่ม Authentication plugin (JWT)
curl -i -X POST http://localhost:8001/services/user-service/plugins \
  --data "name=jwt" \
  --data "config.claims_to_verify=exp"

# เพิ่ม CORS plugin
curl -i -X POST http://localhost:8001/services/user-service/plugins \
  --data "name=cors" \
  --data "config.origins=*" \
  --data "config.methods=GET,POST,PUT,DELETE" \
  --data "config.headers=Content-Type,Authorization"

# เพิ่ม Request Logging
curl -i -X POST http://localhost:8001/services/user-service/plugins \
  --data "name=http-log" \
  --data "config.http_endpoint=http://logging-service:3000/logs"
```

---

## ขั้นตอนที่ 1244: Kong Declarative Config

```yaml
# kong.yml - Declarative configuration
_format_version: "3.0"
_transform: true

services:
  - name: user-service
    url: http://user-service:3001
    routes:
      - name: user-routes
        paths:
          - /api/v1/users
        methods:
          - GET
          - POST
          - PUT
          - DELETE
    plugins:
      - name: rate-limiting
        config:
          minute: 100
          policy: local
      - name: jwt
        config:
          claims_to_verify:
            - exp

  - name: order-service
    url: http://order-service:3002
    routes:
      - name: order-routes
        paths:
          - /api/v1/orders
    plugins:
      - name: rate-limiting
        config:
          minute: 50
          policy: local

  - name: product-service
    url: http://product-service:3003
    routes:
      - name: product-routes
        paths:
          - /api/v1/products
        methods:
          - GET    # Public read access

plugins:
  # Global plugins
  - name: correlation-id
    config:
      header_name: X-Correlation-ID
      generator: uuid#4
      echo_downstream: true
  
  - name: response-transformer
    config:
      add:
        headers:
          - "X-Powered-By: Kong"

consumers:
  - username: frontend-app
    jwt_secrets:
      - key: my-frontend-key
        secret: super-secret-key
```

---

## ขั้นตอนที่ 1245: Nginx เป็น API Gateway

```nginx
# nginx.conf
events {
    worker_connections 1024;
}

http {
    # Upstream services
    upstream user_service {
        least_conn;
        server user-service-1:3001;
        server user-service-2:3001;
        keepalive 32;
    }

    upstream order_service {
        server order-service:3002;
    }

    upstream product_service {
        least_conn;
        server product-service-1:3003;
        server product-service-2:3003;
        server product-service-3:3003;
    }

    # Rate limiting zones
    limit_req_zone $binary_remote_addr zone=api_limit:10m rate=100r/m;
    limit_req_zone $http_authorization zone=user_limit:10m rate=1000r/m;

    # Main server
    server {
        listen 80;
        listen 443 ssl;
        server_name api.example.com;

        ssl_certificate /etc/ssl/certs/api.crt;
        ssl_certificate_key /etc/ssl/private/api.key;
        ssl_protocols TLSv1.2 TLSv1.3;

        # Security headers
        add_header X-Frame-Options DENY;
        add_header X-Content-Type-Options nosniff;
        add_header X-XSS-Protection "1; mode=block";
        add_header Strict-Transport-Security "max-age=31536000; includeSubDomains";
        add_header X-Correlation-ID $request_id;

        # Gzip compression
        gzip on;
        gzip_types application/json text/plain;
        gzip_min_length 1000;

        # Rate limiting
        limit_req zone=api_limit burst=20 nodelay;

        # User service routes
        location /api/v1/users {
            # Auth check
            auth_request /auth;
            auth_request_set $auth_status $upstream_status;

            proxy_pass http://user_service;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Request-ID $request_id;

            proxy_connect_timeout 5s;
            proxy_read_timeout 30s;
        }

        # Order service routes (authenticated)
        location /api/v1/orders {
            auth_request /auth;

            proxy_pass http://order_service;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        }

        # Product service (public)
        location /api/v1/products {
            # Higher rate limit for public endpoints
            limit_req zone=api_limit burst=50 nodelay;

            proxy_pass http://product_service;
            proxy_cache product_cache;
            proxy_cache_valid 200 1m;
            proxy_cache_key $uri$is_args$args;
        }

        # Auth verification endpoint
        location = /auth {
            internal;
            proxy_pass http://auth-service:3000/verify;
            proxy_pass_request_body off;
            proxy_set_header Content-Length "";
            proxy_set_header X-Original-URI $request_uri;
        }

        # Health check
        location /health {
            access_log off;
            return 200 "OK\n";
            add_header Content-Type text/plain;
        }

        # WebSocket support
        location /ws {
            proxy_pass http://ws-service:3004;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection "upgrade";
        }
    }

    # Cache zone for products
    proxy_cache_path /tmp/nginx_cache levels=1:2 keys_zone=product_cache:10m inactive=1h;
}
```

---

## ขั้นตอนที่ 1246: Custom API Gateway ด้วย Node.js

```typescript
// src/gateway/gateway.ts
import express, { Request, Response, NextFunction } from "express";
import { createProxyMiddleware } from "http-proxy-middleware";
import rateLimit from "express-rate-limit";
import helmet from "helmet";
import compression from "compression";
import { v4 as uuidv4 } from "uuid";
import jwt from "jsonwebtoken";

const app = express();

// Services registry
const services = new Map([
  ["users", { url: "http://user-service:3001", auth: true }],
  ["orders", { url: "http://order-service:3002", auth: true }],
  ["products", { url: "http://product-service:3003", auth: false }],
  ["payments", { url: "http://payment-service:3004", auth: true }]
]);

// Security middleware
app.use(helmet());
app.use(compression());

// Correlation ID
app.use((req: Request, res: Response, next: NextFunction) => {
  req.headers["x-correlation-id"] = 
    req.headers["x-correlation-id"] as string ?? uuidv4();
  res.setHeader("x-correlation-id", req.headers["x-correlation-id"] as string);
  next();
});

// Rate limiting
const globalLimiter = rateLimit({
  windowMs: 60 * 1000, // 1 minute
  max: 100,
  message: { error: "Too many requests" },
  keyGenerator: (req) => req.ip ?? "unknown"
});

app.use(globalLimiter);

// Request logging
app.use((req: Request, res: Response, next: NextFunction) => {
  const start = Date.now();
  const correlationId = req.headers["x-correlation-id"];
  
  console.log(`[${correlationId}] → ${req.method} ${req.url}`);
  
  res.on("finish", () => {
    const duration = Date.now() - start;
    console.log(`[${correlationId}] ← ${res.statusCode} ${duration}ms`);
  });
  
  next();
});

// Authentication middleware
function authenticate(req: Request, res: Response, next: NextFunction) {
  const token = req.headers.authorization?.split(" ")[1];
  
  if (!token) {
    return res.status(401).json({ error: "No token provided" });
  }
  
  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET!);
    (req as any).user = decoded;
    next();
  } catch {
    res.status(401).json({ error: "Invalid token" });
  }
}

// Dynamic routing
for (const [prefix, service] of services.entries()) {
  const middlewares: any[] = [];
  
  // Add auth if required
  if (service.auth) {
    middlewares.push(authenticate);
  }
  
  // Add proxy
  middlewares.push(createProxyMiddleware({
    target: service.url,
    changeOrigin: true,
    pathRewrite: { [`^/api/${prefix}`]: "" },
    on: {
      error: (err, req, res: any) => {
        console.error(`Proxy error for ${prefix}:`, err);
        res.status(502).json({ error: "Service unavailable" });
      }
    }
  }));
  
  app.use(`/api/${prefix}`, ...middlewares);
}

export default app;
```

---

## ขั้นตอนที่ 1247: Load Balancing Strategies

```typescript
// src/gateway/load-balancer.ts

interface ServiceInstance {
  url: string;
  weight?: number;
  healthy: boolean;
  activeConnections?: number;
  lastResponseTime?: number;
}

// Round Robin
class RoundRobinBalancer {
  private current = 0;

  next(instances: ServiceInstance[]): ServiceInstance {
    const healthy = instances.filter(i => i.healthy);
    if (healthy.length === 0) throw new Error("No healthy instances");
    
    const instance = healthy[this.current % healthy.length];
    this.current++;
    return instance;
  }
}

// Weighted Round Robin
class WeightedRoundRobinBalancer {
  private current = 0;

  next(instances: ServiceInstance[]): ServiceInstance {
    const healthy = instances.filter(i => i.healthy);
    if (healthy.length === 0) throw new Error("No healthy instances");
    
    const totalWeight = healthy.reduce((sum, i) => sum + (i.weight ?? 1), 0);
    let rand = (this.current++ % totalWeight);
    
    for (const instance of healthy) {
      rand -= (instance.weight ?? 1);
      if (rand < 0) return instance;
    }
    
    return healthy[0];
  }
}

// Least Connections
class LeastConnectionsBalancer {
  next(instances: ServiceInstance[]): ServiceInstance {
    const healthy = instances.filter(i => i.healthy);
    if (healthy.length === 0) throw new Error("No healthy instances");
    
    return healthy.reduce((least, current) =>
      (current.activeConnections ?? 0) < (least.activeConnections ?? 0)
        ? current
        : least
    );
  }
}

// Least Response Time
class LeastResponseTimeBalancer {
  next(instances: ServiceInstance[]): ServiceInstance {
    const healthy = instances.filter(i => i.healthy);
    if (healthy.length === 0) throw new Error("No healthy instances");
    
    return healthy.reduce((fastest, current) =>
      (current.lastResponseTime ?? Infinity) < (fastest.lastResponseTime ?? Infinity)
        ? current
        : fastest
    );
  }
}

// Health-check aware load balancer
class SmartLoadBalancer {
  private readonly balancer: RoundRobinBalancer;
  private instances: ServiceInstance[];

  constructor(initialInstances: ServiceInstance[]) {
    this.instances = initialInstances;
    this.balancer = new RoundRobinBalancer();
    this.startHealthChecks();
  }

  next(): ServiceInstance {
    return this.balancer.next(this.instances);
  }

  private startHealthChecks() {
    setInterval(async () => {
      for (const instance of this.instances) {
        instance.healthy = await this.checkHealth(instance.url);
      }
    }, 10000);
  }

  private async checkHealth(url: string): Promise<boolean> {
    try {
      const response = await fetch(`${url}/health`, {
        signal: AbortSignal.timeout(3000)
      });
      return response.ok;
    } catch {
      return false;
    }
  }
}
```

---

## ขั้นตอนที่ 1248: Circuit Breaker Pattern

```typescript
// src/gateway/circuit-breaker.ts

enum CircuitState {
  CLOSED = "CLOSED",    // Normal operation
  OPEN = "OPEN",        // Blocking calls
  HALF_OPEN = "HALF_OPEN" // Testing recovery
}

interface CircuitBreakerConfig {
  failureThreshold: number;    // Number of failures before opening
  successThreshold: number;    // Number of successes to close again
  timeout: number;             // Time to wait before half-open (ms)
}

class CircuitBreaker {
  private state: CircuitState = CircuitState.CLOSED;
  private failures = 0;
  private successes = 0;
  private lastFailureTime?: number;

  constructor(
    private readonly name: string,
    private readonly config: CircuitBreakerConfig
  ) {}

  async call<T>(operation: () => Promise<T>): Promise<T> {
    if (this.state === CircuitState.OPEN) {
      const elapsed = Date.now() - (this.lastFailureTime ?? 0);
      
      if (elapsed >= this.config.timeout) {
        this.state = CircuitState.HALF_OPEN;
        console.log(`Circuit ${this.name}: HALF_OPEN`);
      } else {
        throw new Error(`Circuit ${this.name} is OPEN`);
      }
    }

    try {
      const result = await operation();
      this.onSuccess();
      return result;
    } catch (error) {
      this.onFailure();
      throw error;
    }
  }

  private onSuccess() {
    if (this.state === CircuitState.HALF_OPEN) {
      this.successes++;
      if (this.successes >= this.config.successThreshold) {
        this.state = CircuitState.CLOSED;
        this.failures = 0;
        this.successes = 0;
        console.log(`Circuit ${this.name}: CLOSED`);
      }
    } else {
      this.failures = 0;
    }
  }

  private onFailure() {
    this.failures++;
    this.lastFailureTime = Date.now();
    
    if (this.failures >= this.config.failureThreshold) {
      this.state = CircuitState.OPEN;
      console.error(`Circuit ${this.name}: OPEN after ${this.failures} failures`);
    }
  }

  getState(): CircuitState {
    return this.state;
  }
}

// Usage in gateway
const orderServiceBreaker = new CircuitBreaker("order-service", {
  failureThreshold: 5,
  successThreshold: 2,
  timeout: 30000 // 30 seconds
});

app.use("/api/orders", async (req: Request, res: Response) => {
  try {
    await orderServiceBreaker.call(async () => {
      const response = await proxyRequest(req);
      res.json(response);
    });
  } catch (error) {
    if (error.message.includes("OPEN")) {
      res.status(503).json({ error: "Service temporarily unavailable" });
    } else {
      res.status(500).json({ error: "Internal error" });
    }
  }
});

function proxyRequest(req: Request): Promise<any> {
  return Promise.resolve({});
}
```

---

## ขั้นตอนที่ 1249: Request/Response Transformation

```typescript
// src/gateway/transformers.ts

// Request transformer
function transformRequest(req: Request, targetService: string) {
  // Add service-specific headers
  req.headers["x-service-name"] = targetService;
  req.headers["x-gateway-version"] = "1.0";
  
  // Transform auth header
  if (req.headers.authorization) {
    const user = (req as any).user;
    req.headers["x-user-id"] = user?.id;
    req.headers["x-user-role"] = user?.role;
  }
  
  // Remove sensitive headers
  delete req.headers["x-forwarded-secret"];
}

// Response transformer
function transformResponse(response: any) {
  // Add standard response wrapper
  if (!response.meta) {
    return {
      data: response,
      meta: {
        timestamp: new Date().toISOString(),
        version: "1.0"
      }
    };
  }
  return response;
}

// Request aggregation (BFF pattern)
app.get("/api/dashboard", authenticate, async (req: Request, res: Response) => {
  try {
    const userId = (req as any).user.id;
    
    // Parallel requests to multiple services
    const [user, orders, notifications] = await Promise.all([
      fetchService(`/users/${userId}`),
      fetchService(`/orders?userId=${userId}&limit=5`),
      fetchService(`/notifications?userId=${userId}&unread=true`)
    ]);
    
    // Aggregate response (BFF - Backend for Frontend)
    res.json({
      user,
      recentOrders: orders,
      unreadNotifications: notifications
    });
  } catch (error) {
    res.status(500).json({ error: "Failed to load dashboard" });
  }
});

async function fetchService(path: string): Promise<any> {
  return {};
}
```

---

## ขั้นตอนที่ 1250: API Versioning

```typescript
// src/gateway/versioning.ts

// Version routing in API Gateway
app.use("/api", (req: Request, res: Response, next: NextFunction) => {
  const version = req.headers["x-api-version"] as string ?? 
    req.path.match(/^\/v(\d+)\//)?.[1] ?? "1";
  
  (req as any).apiVersion = parseInt(version, 10);
  next();
});

// Route to different versions
function createVersionedProxy(
  serviceName: string,
  versionedUrls: Record<number, string>
) {
  return async (req: Request, res: Response, next: NextFunction) => {
    const version = (req as any).apiVersion ?? 1;
    const serviceUrl = versionedUrls[version] ?? versionedUrls[1];
    
    if (!serviceUrl) {
      return res.status(400).json({ error: `API version ${version} not supported` });
    }
    
    // Proxy to versioned service
    createProxyMiddleware({
      target: serviceUrl,
      changeOrigin: true
    })(req, res, next);
  };
}

// Setup versioned routes
app.use("/api/users", createVersionedProxy("users", {
  1: "http://user-service-v1:3001",
  2: "http://user-service-v2:3001"
}));
```

---

## ขั้นตอนที่ 1251: API Documentation Gateway

```typescript
// src/gateway/docs.ts
// Aggregate OpenAPI specs from multiple services

import { OpenAPIObject } from "@nestjs/swagger";

async function aggregateOpenAPISpecs(): Promise<OpenAPIObject> {
  const services = [
    { name: "users", url: "http://user-service:3001/api-json" },
    { name: "orders", url: "http://order-service:3002/api-json" },
    { name: "products", url: "http://product-service:3003/api-json" }
  ];
  
  const specs = await Promise.all(
    services.map(async s => {
      const response = await fetch(s.url);
      const spec = await response.json();
      return { name: s.name, spec };
    })
  );
  
  // Merge specs
  const merged: OpenAPIObject = {
    openapi: "3.0.0",
    info: { title: "API Documentation", version: "1.0.0" },
    paths: {},
    components: { schemas: {}, securitySchemes: {} }
  };
  
  for (const { name, spec } of specs) {
    // Prefix paths with service name
    for (const [path, pathItem] of Object.entries(spec.paths ?? {})) {
      merged.paths[`/${name}${path}`] = pathItem as any;
    }
    
    // Merge schemas with prefix to avoid conflicts
    for (const [schemaName, schema] of Object.entries(spec.components?.schemas ?? {})) {
      merged.components!.schemas![`${name}_${schemaName}`] = schema as any;
    }
  }
  
  return merged;
}

// Serve aggregated docs
app.get("/docs/json", async (req: Request, res: Response) => {
  const spec = await aggregateOpenAPISpecs();
  res.json(spec);
});
```

---

## ขั้นตอนที่ 1252: WebSocket Proxying

```typescript
// src/gateway/ws-proxy.ts
import { createServer } from "http";
import { WebSocketServer, WebSocket } from "ws";

function createWebSocketProxy(targetUrl: string) {
  const wss = new WebSocketServer({ noServer: true });
  
  wss.on("connection", (clientWs, request) => {
    const targetWs = new WebSocket(`${targetUrl}${request.url}`);
    
    // Forward messages from client to target
    clientWs.on("message", (data) => {
      if (targetWs.readyState === WebSocket.OPEN) {
        targetWs.send(data);
      }
    });
    
    // Forward messages from target to client
    targetWs.on("message", (data) => {
      if (clientWs.readyState === WebSocket.OPEN) {
        clientWs.send(data);
      }
    });
    
    // Handle disconnects
    clientWs.on("close", () => targetWs.close());
    targetWs.on("close", () => clientWs.close());
    
    // Handle errors
    clientWs.on("error", (err) => {
      console.error("Client WS error:", err);
      targetWs.close();
    });
    
    targetWs.on("error", (err) => {
      console.error("Target WS error:", err);
      clientWs.close();
    });
  });
  
  return wss;
}

// Setup HTTP server with WebSocket upgrade
const server = createServer(app);
const notificationsWss = createWebSocketProxy("ws://notifications-service:3005");

server.on("upgrade", (request, socket, head) => {
  if (request.url?.startsWith("/ws/notifications")) {
    notificationsWss.handleUpgrade(request, socket as any, head, (ws) => {
      notificationsWss.emit("connection", ws, request);
    });
  }
});
```

---

## ขั้นตอนที่ 1253: API Analytics

```typescript
// src/gateway/analytics.ts

interface RequestMetric {
  timestamp: Date;
  method: string;
  path: string;
  statusCode: number;
  duration: number;
  userId?: string;
  ip: string;
  userAgent?: string;
}

class GatewayAnalytics {
  private metrics: RequestMetric[] = [];
  private readonly maxSize = 10000;

  record(metric: RequestMetric): void {
    this.metrics.push(metric);
    if (this.metrics.length > this.maxSize) {
      this.metrics.shift();
    }
  }

  getStats(windowMs: number = 5 * 60 * 1000) {
    const cutoff = new Date(Date.now() - windowMs);
    const recent = this.metrics.filter(m => m.timestamp > cutoff);
    
    if (recent.length === 0) return null;
    
    const durations = recent.map(m => m.duration);
    const sorted = [...durations].sort((a, b) => a - b);
    
    return {
      requestCount: recent.length,
      errorRate: recent.filter(m => m.statusCode >= 500).length / recent.length,
      avgDuration: durations.reduce((a, b) => a + b, 0) / durations.length,
      p50Duration: sorted[Math.floor(sorted.length * 0.5)],
      p95Duration: sorted[Math.floor(sorted.length * 0.95)],
      p99Duration: sorted[Math.floor(sorted.length * 0.99)],
      byPath: this.groupByPath(recent),
      byStatus: this.groupByStatus(recent)
    };
  }

  private groupByPath(metrics: RequestMetric[]) {
    const groups: Record<string, { count: number; errors: number }> = {};
    for (const m of metrics) {
      const key = `${m.method} ${m.path}`;
      if (!groups[key]) groups[key] = { count: 0, errors: 0 };
      groups[key].count++;
      if (m.statusCode >= 400) groups[key].errors++;
    }
    return groups;
  }

  private groupByStatus(metrics: RequestMetric[]) {
    const groups: Record<number, number> = {};
    for (const m of metrics) {
      groups[m.statusCode] = (groups[m.statusCode] ?? 0) + 1;
    }
    return groups;
  }
}

const analytics = new GatewayAnalytics();

// Analytics middleware
app.use((req: Request, res: Response, next: NextFunction) => {
  const start = Date.now();
  
  res.on("finish", () => {
    analytics.record({
      timestamp: new Date(),
      method: req.method,
      path: req.path,
      statusCode: res.statusCode,
      duration: Date.now() - start,
      userId: (req as any).user?.id,
      ip: req.ip ?? "unknown",
      userAgent: req.headers["user-agent"]
    });
  });
  
  next();
});

// Analytics endpoint
app.get("/gateway/analytics", (req: Request, res: Response) => {
  res.json(analytics.getStats());
});
```

---

## ขั้นตอนที่ 1254: Service Mesh vs API Gateway

```
API Gateway vs Service Mesh:

API Gateway:
- North-South traffic (external → internal)
- Client-facing concerns
- Authentication, rate limiting, routing
- Protocol translation
- API composition

Service Mesh (Istio, Linkerd):
- East-West traffic (service to service)
- Internal concerns
- mTLS, retries, timeouts
- Traffic management
- Observability

Use Both:
External Client → API Gateway → Service A ↔ Service Mesh ↔ Service B
```

---

## ขั้นตอนที่ 1255: Request Caching in Gateway

```typescript
// src/gateway/cache.ts
import { createClient } from "redis";

const redis = createClient({ url: process.env.REDIS_URL });

function cacheMiddleware(ttl: number = 60) {
  return async (req: Request, res: Response, next: NextFunction) => {
    // Only cache GET requests
    if (req.method !== "GET") return next();
    
    const cacheKey = `gateway:${req.url}`;
    
    const cached = await redis.get(cacheKey);
    if (cached) {
      res.setHeader("X-Cache", "HIT");
      res.setHeader("Content-Type", "application/json");
      return res.send(cached);
    }
    
    // Override res.json to cache response
    const originalJson = res.json.bind(res);
    res.json = (data: any) => {
      redis.setEx(cacheKey, ttl, JSON.stringify(data)).catch(console.error);
      res.setHeader("X-Cache", "MISS");
      return originalJson(data);
    };
    
    next();
  };
}

// Apply to specific routes
app.use("/api/products", cacheMiddleware(300)); // Cache 5 minutes
app.use("/api/categories", cacheMiddleware(3600)); // Cache 1 hour
```

---

## ขั้นตอนที่ 1256: Gateway Health Check

```typescript
// src/gateway/health.ts

interface ServiceHealth {
  name: string;
  url: string;
  status: "healthy" | "unhealthy" | "unknown";
  responseTime?: number;
  lastChecked: Date;
  error?: string;
}

class GatewayHealthChecker {
  private serviceHealth = new Map<string, ServiceHealth>();

  async checkService(name: string, url: string): Promise<ServiceHealth> {
    const start = Date.now();
    
    try {
      const response = await fetch(`${url}/health`, {
        signal: AbortSignal.timeout(5000)
      });
      
      const health: ServiceHealth = {
        name,
        url,
        status: response.ok ? "healthy" : "unhealthy",
        responseTime: Date.now() - start,
        lastChecked: new Date()
      };
      
      this.serviceHealth.set(name, health);
      return health;
    } catch (error) {
      const health: ServiceHealth = {
        name,
        url,
        status: "unhealthy",
        lastChecked: new Date(),
        error: (error as Error).message
      };
      
      this.serviceHealth.set(name, health);
      return health;
    }
  }

  async checkAll(services: Record<string, string>): Promise<ServiceHealth[]> {
    return Promise.all(
      Object.entries(services).map(([name, url]) =>
        this.checkService(name, url)
      )
    );
  }

  getHealth(): ServiceHealth[] {
    return Array.from(this.serviceHealth.values());
  }
}

const healthChecker = new GatewayHealthChecker();

// Health endpoint
app.get("/health", async (req: Request, res: Response) => {
  const services = {
    "user-service": "http://user-service:3001",
    "order-service": "http://order-service:3002",
    "product-service": "http://product-service:3003"
  };
  
  const health = await healthChecker.checkAll(services);
  const isHealthy = health.every(h => h.status === "healthy");
  
  res.status(isHealthy ? 200 : 503).json({
    status: isHealthy ? "healthy" : "degraded",
    services: health
  });
});

// Periodic health checks
setInterval(() => {
  healthChecker.checkAll({
    "user-service": "http://user-service:3001",
    "order-service": "http://order-service:3002"
  });
}, 30000);
```

---

## ขั้นตอนที่ 1257: GraphQL Federation Gateway

```typescript
// npm install @apollo/gateway @apollo/server graphql

// src/gateway/graphql-gateway.ts
import { ApolloGateway, IntrospectAndCompose } from "@apollo/gateway";
import { ApolloServer } from "@apollo/server";
import { expressMiddleware } from "@apollo/server/express4";

const gateway = new ApolloGateway({
  supergraphSdl: new IntrospectAndCompose({
    subgraphs: [
      { name: "users", url: "http://user-service:3001/graphql" },
      { name: "products", url: "http://product-service:3003/graphql" },
      { name: "orders", url: "http://order-service:3002/graphql" }
    ]
  })
});

const server = new ApolloServer({ gateway });

async function setupGraphQL() {
  await server.start();
  
  app.use(
    "/graphql",
    expressMiddleware(server, {
      context: async ({ req }) => ({
        user: (req as any).user,
        token: req.headers.authorization
      })
    })
  );
}

setupGraphQL();
```

---

## ขั้นตอนที่ 1258: API Gateway Testing

```typescript
// tests/gateway.test.ts
import * as request from "supertest";
import app from "../src/gateway/gateway";

describe("API Gateway", () => {
  describe("Rate Limiting", () => {
    it("should allow requests under limit", async () => {
      const response = await request(app)
        .get("/api/products")
        .expect(200);
      
      expect(response.headers["x-ratelimit-remaining"]).toBeDefined();
    });

    it("should block requests over limit", async () => {
      // Send 101 requests
      const requests = Array.from({ length: 101 }, () =>
        request(app).get("/api/products")
      );
      
      const responses = await Promise.all(requests);
      const blocked = responses.filter(r => r.status === 429);
      
      expect(blocked.length).toBeGreaterThan(0);
    });
  });

  describe("Authentication", () => {
    it("should return 401 for protected routes without token", async () => {
      await request(app)
        .get("/api/orders")
        .expect(401);
    });

    it("should allow access with valid token", async () => {
      const token = generateTestToken({ id: "user1", role: "user" });
      
      await request(app)
        .get("/api/orders")
        .set("Authorization", `Bearer ${token}`)
        .expect(200);
    });
  });
});

function generateTestToken(payload: any): string {
  const jwt = require("jsonwebtoken");
  return jwt.sign(payload, process.env.JWT_SECRET!, { expiresIn: "1h" });
}
```

---

## ขั้นตอนที่ 1259: Security Hardening

```typescript
// src/gateway/security.ts

// IP Whitelist/Blacklist
const ipBlacklist = new Set<string>();
const ipWhitelist = new Set<string>();

function ipFilterMiddleware(req: Request, res: Response, next: NextFunction) {
  const ip = req.ip ?? "";
  
  if (ipBlacklist.has(ip)) {
    return res.status(403).json({ error: "Access denied" });
  }
  
  // If whitelist is non-empty, only allow whitelisted IPs for admin
  if (req.path.startsWith("/admin") && ipWhitelist.size > 0) {
    if (!ipWhitelist.has(ip)) {
      return res.status(403).json({ error: "Access denied" });
    }
  }
  
  next();
}

// Request size limits
app.use(express.json({ limit: "10mb" }));
app.use(express.urlencoded({ limit: "10mb", extended: true }));

// SQL injection prevention in query params
function sanitizeQueryParams(req: Request, res: Response, next: NextFunction) {
  const dangerousPatterns = [
    /(\b(SELECT|INSERT|UPDATE|DELETE|DROP|CREATE|ALTER|EXEC)\b)/gi,
    /(--|\/\*|\*\/)/g,
    /(\bOR\b|\bAND\b)\s+\d+\s*=\s*\d+/gi
  ];
  
  const queryStr = JSON.stringify(req.query);
  
  for (const pattern of dangerousPatterns) {
    if (pattern.test(queryStr)) {
      return res.status(400).json({ error: "Invalid request parameters" });
    }
  }
  
  next();
}

app.use(sanitizeQueryParams);

// XSS protection in body
function sanitizeBody(req: Request, res: Response, next: NextFunction) {
  if (req.body && typeof req.body === "object") {
    req.body = sanitizeObject(req.body);
  }
  next();
}

function sanitizeObject(obj: any): any {
  if (typeof obj === "string") {
    return obj
      .replace(/<script[^>]*>.*?<\/script>/gi, "")
      .replace(/javascript:/gi, "");
  }
  if (typeof obj === "object" && obj !== null) {
    const result: any = Array.isArray(obj) ? [] : {};
    for (const [key, value] of Object.entries(obj)) {
      result[key] = sanitizeObject(value);
    }
    return result;
  }
  return obj;
}
```

---

## ขั้นตอนที่ 1260: Production Deployment

```yaml
# kubernetes/api-gateway-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-gateway
spec:
  replicas: 3
  selector:
    matchLabels:
      app: api-gateway
  template:
    metadata:
      labels:
        app: api-gateway
    spec:
      containers:
        - name: api-gateway
          image: my-registry/api-gateway:latest
          ports:
            - containerPort: 3000
          env:
            - name: JWT_SECRET
              valueFrom:
                secretKeyRef:
                  name: jwt-secret
                  key: secret
            - name: REDIS_URL
              valueFrom:
                configMapKeyRef:
                  name: app-config
                  key: redis-url
          resources:
            requests:
              memory: "256Mi"
              cpu: "250m"
            limits:
              memory: "512Mi"
              cpu: "500m"
          livenessProbe:
            httpGet:
              path: /health
              port: 3000
            initialDelaySeconds: 30
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /health
              port: 3000
            initialDelaySeconds: 5
            periodSeconds: 5

---
apiVersion: v1
kind: Service
metadata:
  name: api-gateway-service
spec:
  selector:
    app: api-gateway
  ports:
    - port: 80
      targetPort: 3000
  type: LoadBalancer
```

---

## 🏋️ แบบฝึกหัด

### แบบฝึกหัดที่ 1: Kong Setup
1. ติดตั้ง Kong ด้วย Docker
2. กำหนดค่า services และ routes
3. เพิ่ม JWT authentication plugin

### แบบฝึกหัดที่ 2: Custom Gateway
1. สร้าง Node.js API Gateway
2. implement round-robin load balancing
3. เพิ่ม circuit breaker

### แบบฝึกหัดที่ 3: Nginx Gateway
1. กำหนดค่า Nginx เป็น API Gateway
2. implement rate limiting
3. เพิ่ม response caching

---

## 📚 สรุป

ใน Part นี้เราได้เรียนรู้:
- API Gateway patterns และ benefits
- Kong API Gateway configuration
- Nginx เป็น reverse proxy และ API Gateway
- Custom Node.js Gateway
- Load balancing strategies
- Circuit breaker pattern
- Request/response transformation
- Security hardening

**Part ถัดไป**: Service Mesh
