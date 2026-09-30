# Part 40 | ขั้นตอนที่ 701-720 จาก 1000

## Docker และ Node.js - การ Containerize แอปพลิเคชัน

---

## สารบัญ

1. [Docker พื้นฐาน](#docker-พื้นฐาน)
2. [Dockerfile สำหรับ Node.js](#dockerfile-สำหรับ-nodejs)
3. [Docker Compose](#docker-compose)
4. [Multi-stage Builds](#multi-stage-builds)
5. [Container Networking](#container-networking)
6. [Docker Volumes](#docker-volumes)
7. [Security Best Practices](#security-best-practices)
8. [Production Deployment](#production-deployment)
9. [แบบฝึกหัด](#แบบฝึกหัด)

---

## Docker พื้นฐาน

### ขั้นตอนที่ 701: Docker คืออะไร

```
ปัญหา: "Works on my machine" syndrome
  Dev environment: Ubuntu 22.04, Node 18, PostgreSQL 15
  Production: CentOS 7, Node 16, PostgreSQL 13
  → แอปทำงานผิดปกติใน production

Docker แก้ปัญหา:
  Container = App + Dependencies + Configuration
  ทำงานเหมือนกันทุก environment
```

```
Docker Architecture:
  ┌─────────────────────────────┐
  │         Docker Host         │
  │  ┌──────────────────────┐   │
  │  │    Docker Engine     │   │
  │  │  ┌──────────────┐   │   │
  │  │  │  Container 1  │   │   │
  │  │  │  (Node App)   │   │   │
  │  │  └──────────────┘   │   │
  │  │  ┌──────────────┐   │   │
  │  │  │  Container 2  │   │   │
  │  │  │  (MongoDB)    │   │   │
  │  │  └──────────────┘   │   │
  │  └──────────────────────┘   │
  └─────────────────────────────┘
```

### ขั้นตอนที่ 702: คำสั่ง Docker พื้นฐาน

```bash
# ดู images ที่มีอยู่
docker images

# Pull image จาก Docker Hub
docker pull node:18-alpine

# Run container
docker run -d \
  --name my-app \
  -p 3000:3000 \
  -e NODE_ENV=production \
  node:18-alpine

# ดู containers ที่กำลังทำงาน
docker ps

# ดู container logs
docker logs my-app
docker logs -f my-app  # Follow logs

# เข้าไปใน container
docker exec -it my-app sh

# หยุด/ลบ container
docker stop my-app
docker rm my-app

# Build image
docker build -t my-app:1.0 .

# รัน container จาก image ที่ build
docker run -d -p 3000:3000 my-app:1.0

# ลบ images ที่ไม่ใช้
docker image prune

# ดู resource usage
docker stats
```

---

## Dockerfile สำหรับ Node.js

### ขั้นตอนที่ 703: Dockerfile พื้นฐาน

```dockerfile
# Dockerfile
# Base image
FROM node:18-alpine

# ตั้ง working directory
WORKDIR /app

# Copy package files ก่อน (leverage layer caching)
COPY package*.json ./

# ติดตั้ง dependencies
RUN npm ci --only=production

# Copy source code
COPY . .

# Expose port
EXPOSE 3000

# Start command
CMD ["node", "src/index.js"]
```

### ขั้นตอนที่ 704: .dockerignore

```
# .dockerignore
node_modules
npm-debug.log
.git
.gitignore
.env
.env.*
README.md
*.test.js
__tests__/
coverage/
.nyc_output
dist/
build/
*.log
.DS_Store
Thumbs.db
```

### ขั้นตอนที่ 705: Optimized Dockerfile

```dockerfile
# Dockerfile - Optimized version
FROM node:18-alpine AS base

# ติดตั้ง dependencies ที่จำเป็นสำหรับ build
RUN apk add --no-cache \
    python3 \
    make \
    g++

# ตั้ง non-root user (security)
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nextjs -u 1001

WORKDIR /app

# ==========================================
# Dependencies stage
# ==========================================
FROM base AS deps

COPY package*.json ./

# ติดตั้ง production dependencies เท่านั้น
RUN npm ci --only=production && npm cache clean --force

# ==========================================
# Development stage
# ==========================================
FROM base AS development

COPY package*.json ./
RUN npm ci

COPY --chown=nodejs:nodejs . .

USER nodejs
EXPOSE 3000

CMD ["npm", "run", "dev"]

# ==========================================
# Production stage
# ==========================================
FROM base AS production

# Copy production dependencies
COPY --from=deps --chown=nodejs:nodejs /app/node_modules ./node_modules

# Copy source code
COPY --chown=nodejs:nodejs . .

# เปลี่ยนเป็น non-root user
USER nodejs

EXPOSE 3000

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD wget --no-verbose --tries=1 --spider http://localhost:3000/health || exit 1

CMD ["node", "src/index.js"]
```

### ขั้นตอนที่ 706: Build และ Run

```bash
# Build image
docker build -t myapp:latest .

# Build สำหรับ production (multi-stage)
docker build --target production -t myapp:prod .

# Build สำหรับ development
docker build --target development -t myapp:dev .

# Run พร้อม environment variables
docker run -d \
  --name myapp \
  -p 3000:3000 \
  --env-file .env \
  myapp:prod

# Tag และ push ไปยัง registry
docker tag myapp:latest registry.example.com/myapp:v1.0.0
docker push registry.example.com/myapp:v1.0.0
```

---

## Docker Compose

### ขั้นตอนที่ 707: docker-compose.yml พื้นฐาน

```yaml
# docker-compose.yml
version: '3.8'

services:
  # Node.js Application
  app:
    build:
      context: .
      dockerfile: Dockerfile
      target: production
    container_name: myapp
    restart: unless-stopped
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=production
      - PORT=3000
      - MONGODB_URI=mongodb://mongodb:27017/mydb
      - REDIS_URL=redis://redis:6379
    env_file:
      - .env
    depends_on:
      mongodb:
        condition: service_healthy
      redis:
        condition: service_started
    networks:
      - app-network
    volumes:
      - uploads:/app/uploads
      - logs:/app/logs

  # MongoDB
  mongodb:
    image: mongo:6.0
    container_name: mongodb
    restart: unless-stopped
    ports:
      - "27017:27017"  # เปิดใน development เท่านั้น
    environment:
      MONGO_INITDB_ROOT_USERNAME: ${MONGO_ROOT_USER}
      MONGO_INITDB_ROOT_PASSWORD: ${MONGO_ROOT_PASSWORD}
      MONGO_INITDB_DATABASE: mydb
    volumes:
      - mongodb-data:/data/db
      - ./mongo-init.js:/docker-entrypoint-initdb.d/mongo-init.js:ro
    networks:
      - app-network
    healthcheck:
      test: ["CMD", "mongosh", "--eval", "db.adminCommand('ping')"]
      interval: 10s
      timeout: 5s
      retries: 5

  # Redis
  redis:
    image: redis:7-alpine
    container_name: redis
    restart: unless-stopped
    ports:
      - "6379:6379"
    command: redis-server --requirepass ${REDIS_PASSWORD}
    volumes:
      - redis-data:/data
    networks:
      - app-network

  # Nginx Reverse Proxy
  nginx:
    image: nginx:alpine
    container_name: nginx
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./nginx/ssl:/etc/nginx/ssl:ro
      - ./nginx/html:/usr/share/nginx/html:ro
    depends_on:
      - app
    networks:
      - app-network

networks:
  app-network:
    driver: bridge

volumes:
  mongodb-data:
  redis-data:
  uploads:
  logs:
```

### ขั้นตอนที่ 708: docker-compose.dev.yml

```yaml
# docker-compose.dev.yml
version: '3.8'

services:
  app:
    build:
      target: development
    volumes:
      # Mount source code สำหรับ hot reload
      - .:/app
      - /app/node_modules  # ไม่ mount node_modules จาก host
    environment:
      - NODE_ENV=development
    command: npm run dev
    ports:
      - "3000:3000"
      - "9229:9229"  # Debug port

  # Development tools
  mongo-express:
    image: mongo-express
    restart: always
    ports:
      - "8081:8081"
    environment:
      ME_CONFIG_MONGODB_SERVER: mongodb
      ME_CONFIG_MONGODB_ADMINUSERNAME: ${MONGO_ROOT_USER}
      ME_CONFIG_MONGODB_ADMINPASSWORD: ${MONGO_ROOT_PASSWORD}
    depends_on:
      - mongodb
    networks:
      - app-network

  redis-commander:
    image: rediscommander/redis-commander:latest
    environment:
      - REDIS_HOSTS=local:redis:6379
    ports:
      - "8082:8081"
    depends_on:
      - redis
    networks:
      - app-network
```

```bash
# Development
docker-compose -f docker-compose.yml -f docker-compose.dev.yml up

# Production
docker-compose up -d

# Rebuild
docker-compose build --no-cache

# ดู logs
docker-compose logs -f app

# Scale workers
docker-compose up -d --scale worker=3
```

---

## Multi-stage Builds

### ขั้นตอนที่ 709: TypeScript Build

```dockerfile
# Dockerfile สำหรับ TypeScript project
FROM node:18-alpine AS base
WORKDIR /app

# ==========================================
# Dependencies
# ==========================================
FROM base AS deps
COPY package*.json tsconfig.json ./
RUN npm ci

# ==========================================
# Builder
# ==========================================
FROM deps AS builder
COPY src/ ./src/
# Build TypeScript
RUN npm run build
# ลบ dev dependencies
RUN npm ci --only=production

# ==========================================
# Production
# ==========================================
FROM base AS production

RUN addgroup -g 1001 -S nodejs && \
    adduser -S app -u 1001

# Copy built files
COPY --from=builder --chown=app:nodejs /app/dist ./dist
COPY --from=builder --chown=app:nodejs /app/node_modules ./node_modules
COPY --chown=app:nodejs package.json ./

USER app

HEALTHCHECK --interval=30s --timeout=3s \
  CMD wget --no-verbose --tries=1 --spider http://localhost:3000/health || exit 1

CMD ["node", "dist/index.js"]
```

---

## Container Networking

### ขั้นตอนที่ 710: Docker Networks

```bash
# ดู networks
docker network ls

# สร้าง network
docker network create --driver bridge mynetwork

# เชื่อมต่อ container กับ network
docker network connect mynetwork container1
docker network connect mynetwork container2

# ตรวจสอบ network
docker network inspect mynetwork
```

```yaml
# docker-compose.yml - Network configuration
services:
  app:
    networks:
      - frontend
      - backend

  database:
    networks:
      - backend  # เฉพาะ backend network (ไม่ expose ถึง frontend)

  nginx:
    networks:
      - frontend

networks:
  frontend:
    driver: bridge
  backend:
    driver: bridge
    internal: true  # ไม่สามารถเข้าถึงจาก outside
```

### ขั้นตอนที่ 711: Nginx Reverse Proxy

```nginx
# nginx/nginx.conf
worker_processes auto;

events {
    worker_connections 1024;
}

http {
    # Gzip compression
    gzip on;
    gzip_types text/plain application/json application/javascript text/css;

    # Rate limiting
    limit_req_zone $binary_remote_addr zone=api:10m rate=10r/s;

    upstream nodejs {
        # Load balancing ระหว่าง Node.js instances
        server app:3000;
        # server app2:3000;  # Scale เพิ่ม instances
        keepalive 32;
    }

    server {
        listen 80;
        server_name example.com www.example.com;

        # Redirect HTTP → HTTPS
        return 301 https://$server_name$request_uri;
    }

    server {
        listen 443 ssl http2;
        server_name example.com www.example.com;

        # SSL Configuration
        ssl_certificate /etc/nginx/ssl/fullchain.pem;
        ssl_certificate_key /etc/nginx/ssl/privkey.pem;
        ssl_protocols TLSv1.2 TLSv1.3;
        ssl_ciphers HIGH:!aNULL:!MD5;

        # Security headers
        add_header X-Frame-Options DENY;
        add_header X-Content-Type-Options nosniff;
        add_header X-XSS-Protection "1; mode=block";
        add_header Strict-Transport-Security "max-age=31536000; includeSubDomains";

        # Static files
        location /static {
            root /usr/share/nginx/html;
            expires 1y;
            add_header Cache-Control "public, immutable";
        }

        # API proxy
        location /api {
            limit_req zone=api burst=20 nodelay;

            proxy_pass http://nodejs;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection 'upgrade';
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            proxy_cache_bypass $http_upgrade;

            # Timeouts
            proxy_connect_timeout 60s;
            proxy_send_timeout 60s;
            proxy_read_timeout 60s;
        }

        # WebSocket
        location /socket.io {
            proxy_pass http://nodejs;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection "upgrade";
            proxy_read_timeout 3600;
        }

        # Health check
        location /health {
            proxy_pass http://nodejs;
            access_log off;
        }
    }
}
```

---

## Docker Volumes

### ขั้นตอนที่ 712: Volume Types

```bash
# Named volumes (managed by Docker)
docker volume create mydata
docker run -v mydata:/app/data myimage

# Bind mounts (mount host directory)
docker run -v $(pwd)/data:/app/data myimage

# tmpfs mounts (in-memory, no persistence)
docker run --tmpfs /tmp myimage
```

```yaml
# docker-compose.yml - Volumes
services:
  app:
    volumes:
      # Named volume
      - app-data:/app/data
      # Bind mount (development)
      - ./config:/app/config:ro  # read-only
      # เฉพาะ directory เดิม (ไม่ sync)
      - /app/node_modules

volumes:
  app-data:
    driver: local
    driver_opts:
      type: none
      o: bind
      device: /data/myapp
```

---

## Security Best Practices

### ขั้นตอนที่ 713: Container Security

```dockerfile
# Security-hardened Dockerfile
FROM node:18-alpine

# อัปเดต packages
RUN apk update && apk upgrade && apk add --no-cache \
    curl \
    && rm -rf /var/cache/apk/*

# สร้าง non-root user
RUN addgroup -g 1001 -S app && \
    adduser -S app -u 1001 -G app

WORKDIR /app

# เปลี่ยน ownership
COPY --chown=app:app package*.json ./

RUN npm ci --only=production && \
    npm cache clean --force && \
    # ลบ npm
    rm -rf /usr/local/lib/node_modules/npm \
    /usr/local/bin/npm \
    /usr/local/bin/npx

COPY --chown=app:app . .

# ไม่ใช้ root user
USER app

# ไม่เปิด privileged ports (<1024)
EXPOSE 3000

# Read-only filesystem (ถ้าเป็นไปได้)
# สามารถ override บาง directories ด้วย tmpfs

CMD ["node", "--max-old-space-size=512", "src/index.js"]
```

```yaml
# docker-compose.yml - Security settings
services:
  app:
    security_opt:
      - no-new-privileges:true
    read_only: true      # Read-only filesystem
    tmpfs:
      - /tmp
      - /app/temp
    cap_drop:
      - ALL              # ลบ capabilities ทั้งหมด
    cap_add:
      - NET_BIND_SERVICE  # เพิ่มเฉพาะที่จำเป็น
    ulimits:
      nproc: 65535
      nofile:
        soft: 20000
        hard: 40000
    deploy:
      resources:
        limits:
          cpus: '0.5'
          memory: 512M
        reservations:
          memory: 256M
```

### ขั้นตอนที่ 714: Secret Management

```yaml
# docker-compose.yml - Secrets
services:
  app:
    environment:
      - NODE_ENV=production
    secrets:
      - db_password
      - jwt_secret
      - api_key

secrets:
  db_password:
    file: ./secrets/db_password.txt
  jwt_secret:
    external: true  # ใช้ Docker Swarm secret
  api_key:
    environment: "API_KEY"  # จาก environment variable
```

```javascript
// อ่าน secrets จาก Docker secrets
const fs = require('fs');

function readSecret(secretName) {
  try {
    // Docker secrets จะถูก mount ที่ /run/secrets/
    return fs.readFileSync(`/run/secrets/${secretName}`, 'utf8').trim();
  } catch (error) {
    // Fallback ไปยัง environment variable
    return process.env[secretName.toUpperCase()];
  }
}

const dbPassword = readSecret('db_password');
const jwtSecret = readSecret('jwt_secret');
```

---

## Production Deployment

### ขั้นตอนที่ 715: Health Checks

```javascript
// src/health.js
const mongoose = require('mongoose');
const redis = require('./config/redis');

async function checkHealth() {
  const checks = {};

  // MongoDB health
  try {
    await mongoose.connection.db.admin().ping();
    checks.mongodb = { status: 'healthy' };
  } catch (error) {
    checks.mongodb = { status: 'unhealthy', error: error.message };
  }

  // Redis health
  try {
    await redis.ping();
    checks.redis = { status: 'healthy' };
  } catch (error) {
    checks.redis = { status: 'unhealthy', error: error.message };
  }

  // Memory check
  const memUsage = process.memoryUsage();
  checks.memory = {
    status: memUsage.heapUsed < 500 * 1024 * 1024 ? 'healthy' : 'warning',
    heapUsed: `${Math.round(memUsage.heapUsed / 1024 / 1024)}MB`,
    heapTotal: `${Math.round(memUsage.heapTotal / 1024 / 1024)}MB`,
  };

  const isHealthy = Object.values(checks).every(c => c.status !== 'unhealthy');

  return {
    status: isHealthy ? 'healthy' : 'unhealthy',
    checks,
    uptime: process.uptime(),
    timestamp: new Date().toISOString(),
    version: process.env.APP_VERSION || '1.0.0',
  };
}

module.exports = { checkHealth };
```

```javascript
// Health endpoint
app.get('/health', async (req, res) => {
  const health = await checkHealth();
  res.status(health.status === 'healthy' ? 200 : 503).json(health);
});

// Readiness endpoint (สำหรับ Kubernetes)
app.get('/ready', async (req, res) => {
  try {
    await mongoose.connection.db.admin().ping();
    res.status(200).json({ status: 'ready' });
  } catch {
    res.status(503).json({ status: 'not ready' });
  }
});

// Liveness endpoint (สำหรับ Kubernetes)
app.get('/live', (req, res) => {
  res.status(200).json({ status: 'alive' });
});
```

### ขั้นตอนที่ 716: Graceful Shutdown

```javascript
// src/gracefulShutdown.js
const mongoose = require('mongoose');
const redis = require('./config/redis');

function setupGracefulShutdown(server) {
  let isShuttingDown = false;

  async function shutdown(signal) {
    if (isShuttingDown) return;
    isShuttingDown = true;

    console.log(`\n📴 Received ${signal}. Graceful shutdown...`);

    // หยุดรับ new requests
    server.close(async () => {
      console.log('✅ HTTP server closed');

      try {
        // ปิด database connections
        await mongoose.connection.close();
        console.log('✅ MongoDB disconnected');

        await redis.quit();
        console.log('✅ Redis disconnected');

        // รอ pending operations เสร็จ (max 30 วินาที)
        await waitForPendingOps();

        console.log('✅ Graceful shutdown complete');
        process.exit(0);
      } catch (error) {
        console.error('Error during shutdown:', error);
        process.exit(1);
      }
    });

    // Force shutdown หลัง 30 วินาที
    setTimeout(() => {
      console.error('❌ Forced shutdown after timeout');
      process.exit(1);
    }, 30000);
  }

  process.on('SIGTERM', () => shutdown('SIGTERM'));
  process.on('SIGINT', () => shutdown('SIGINT'));

  // Handle uncaught exceptions
  process.on('uncaughtException', (error) => {
    console.error('Uncaught Exception:', error);
    shutdown('uncaughtException');
  });

  process.on('unhandledRejection', (reason, promise) => {
    console.error('Unhandled Rejection at:', promise, 'reason:', reason);
    shutdown('unhandledRejection');
  });
}

async function waitForPendingOps() {
  // รอ pending database operations
  return new Promise(resolve => setTimeout(resolve, 1000));
}

module.exports = { setupGracefulShutdown };
```

---

## Docker Swarm

### ขั้นตอนที่ 717: Docker Swarm Mode

```bash
# Initialize Swarm
docker swarm init --advertise-addr <MANAGER-IP>

# Join nodes
docker swarm join --token <TOKEN> <MANAGER-IP>:2377

# Deploy stack
docker stack deploy -c docker-compose.yml myapp

# ดู services
docker service ls

# Scale service
docker service scale myapp_app=3

# Update service (rolling update)
docker service update \
  --image myapp:v2.0 \
  --update-parallelism 1 \
  --update-delay 30s \
  myapp_app

# Rollback
docker service rollback myapp_app
```

```yaml
# docker-compose.swarm.yml
version: '3.8'

services:
  app:
    image: registry.example.com/myapp:latest
    deploy:
      replicas: 3
      restart_policy:
        condition: on-failure
        max_attempts: 3
      update_config:
        parallelism: 1
        delay: 10s
        failure_action: rollback
        order: start-first
      rollback_config:
        parallelism: 1
        delay: 10s
      resources:
        limits:
          cpus: '0.5'
          memory: 512M
    networks:
      - webnet

networks:
  webnet:
    driver: overlay
```

---

## Monitoring Containers

### ขั้นตอนที่ 718: Container Monitoring

```yaml
# docker-compose.monitoring.yml
version: '3.8'

services:
  # Prometheus
  prometheus:
    image: prom/prometheus:latest
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus-data:/prometheus

  # Grafana
  grafana:
    image: grafana/grafana:latest
    ports:
      - "3001:3000"
    environment:
      GF_SECURITY_ADMIN_PASSWORD: ${GRAFANA_PASSWORD}
    volumes:
      - grafana-data:/var/lib/grafana

  # Node Exporter (system metrics)
  node-exporter:
    image: prom/node-exporter:latest
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro

  # cAdvisor (container metrics)
  cadvisor:
    image: gcr.io/cadvisor/cadvisor:latest
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:ro
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro

volumes:
  prometheus-data:
  grafana-data:
```

```yaml
# prometheus.yml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'nodejs'
    static_configs:
      - targets: ['app:9100']

  - job_name: 'mongodb'
    static_configs:
      - targets: ['mongodb-exporter:9216']

  - job_name: 'redis'
    static_configs:
      - targets: ['redis-exporter:9121']
```

---

## Kubernetes Basics

### ขั้นตอนที่ 719: Kubernetes Deployment

```yaml
# k8s/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  labels:
    app: myapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
        - name: myapp
          image: registry.example.com/myapp:v1.0.0
          ports:
            - containerPort: 3000
          env:
            - name: NODE_ENV
              value: "production"
            - name: MONGODB_URI
              valueFrom:
                secretKeyRef:
                  name: app-secrets
                  key: mongodb-uri
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "500m"
              memory: "512Mi"
          readinessProbe:
            httpGet:
              path: /ready
              port: 3000
            initialDelaySeconds: 5
            periodSeconds: 10
          livenessProbe:
            httpGet:
              path: /live
              port: 3000
            initialDelaySeconds: 15
            periodSeconds: 20
---
apiVersion: v1
kind: Service
metadata:
  name: myapp-service
spec:
  selector:
    app: myapp
  ports:
    - protocol: TCP
      port: 80
      targetPort: 3000
  type: ClusterIP
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
    - host: example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: myapp-service
                port:
                  number: 80
```

### ขั้นตอนที่ 720: Kubernetes Commands

```bash
# Apply configurations
kubectl apply -f k8s/

# ดู resources
kubectl get pods
kubectl get services
kubectl get deployments

# ดู logs
kubectl logs -f deployment/myapp

# Scale
kubectl scale deployment/myapp --replicas=5

# Update image (rolling update)
kubectl set image deployment/myapp myapp=registry.example.com/myapp:v2.0

# Rollback
kubectl rollout undo deployment/myapp

# ดู rollout status
kubectl rollout status deployment/myapp

# เข้าไปใน pod
kubectl exec -it <pod-name> -- sh

# Port forward (development)
kubectl port-forward service/myapp-service 3000:80
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Build Optimized Image

```dockerfile
# TODO: สร้าง Dockerfile ที่:
# 1. ใช้ multi-stage build
# 2. ขนาด image น้อยกว่า 100MB
# 3. Run เป็น non-root user
# 4. มี health check
# 5. Handle graceful shutdown
```

### แบบฝึกหัดที่ 2: Full Stack Docker Compose

```yaml
# TODO: สร้าง docker-compose.yml สำหรับ full-stack app:
# - Node.js API
# - React Frontend
# - MongoDB
# - Redis
# - Nginx reverse proxy
# - Monitor ด้วย Prometheus + Grafana
```

### แบบฝึกหัดที่ 3: CI/CD Pipeline

```bash
# TODO: สร้าง script สำหรับ:
# 1. Build Docker image
# 2. Run tests ใน container
# 3. Push ไปยัง registry
# 4. Deploy ด้วย docker-compose
# ทั้งหมดใน single script
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **Docker Basics** - images, containers, commands
2. **Dockerfile** - best practices สำหรับ Node.js
3. **docker-compose** - multi-container applications
4. **Multi-stage Builds** - optimize image size
5. **Networking** - container communication, Nginx
6. **Security** - non-root user, secrets, capabilities
7. **Production** - health checks, graceful shutdown
8. **Kubernetes** - deployment, scaling, rolling updates

ในบทถัดไปเราจะเรียนรู้ **CI/CD กับ GitHub Actions** - automated testing และ deployment
