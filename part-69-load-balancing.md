# Part 69: Load Balancing
## ขั้นตอนที่ 681-690 จาก 1000

---

## Load Balancing คืออะไร?

Load Balancing คือการกระจาย incoming requests ไปยัง server instances หลายตัว เพื่อป้องกันไม่ให้ instance ใด instance หนึ่งรับ load มากเกินไป และเพิ่ม availability ของระบบ

---

## 1. Load Balancing Algorithms

### Round Robin

```
Request 1 → Server A
Request 2 → Server B
Request 3 → Server C
Request 4 → Server A  (วนซ้ำ)
```

### Least Connections

```
Server A: 10 connections
Server B: 5 connections  ← ส่งไปที่นี่
Server C: 8 connections
```

### IP Hash (Sticky Sessions)

```
Client IP 192.168.1.1 → Server A (เสมอ)
Client IP 192.168.1.2 → Server B (เสมอ)
```

### Weighted Round Robin

```
Server A (weight: 3): รับ 3 requests
Server B (weight: 2): รับ 2 requests
Server C (weight: 1): รับ 1 request
```

---

## 2. Nginx Load Balancer

### Basic Nginx Configuration

```nginx
# /etc/nginx/conf.d/app.conf

upstream nodejs_backend {
    # Round Robin (default)
    server 127.0.0.1:3001;
    server 127.0.0.1:3002;
    server 127.0.0.1:3003;
    
    # Optional: keepalive connections
    keepalive 32;
}

server {
    listen 80;
    server_name example.com;

    location / {
        proxy_pass http://nodejs_backend;
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
}
```

### Weighted Load Balancing

```nginx
upstream nodejs_backend {
    server 127.0.0.1:3001 weight=3;   # รับ 3/6 ของ requests
    server 127.0.0.1:3002 weight=2;   # รับ 2/6
    server 127.0.0.1:3003 weight=1;   # รับ 1/6
    
    # Health check
    server 127.0.0.1:3004 backup;     # ใช้เมื่อ servers อื่น down
}
```

### Least Connections

```nginx
upstream nodejs_backend {
    least_conn;
    server 127.0.0.1:3001;
    server 127.0.0.1:3002;
    server 127.0.0.1:3003;
}
```

### Nginx with SSL Termination

```nginx
server {
    listen 443 ssl http2;
    server_name example.com;
    
    ssl_certificate /etc/nginx/ssl/cert.pem;
    ssl_certificate_key /etc/nginx/ssl/key.pem;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers on;
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 10m;
    
    # HSTS
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;

    location / {
        proxy_pass http://nodejs_backend;
        proxy_http_version 1.1;
        proxy_set_header X-Forwarded-Proto https;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}

# Redirect HTTP to HTTPS
server {
    listen 80;
    server_name example.com;
    return 301 https://$server_name$request_uri;
}
```

### Nginx Rate Limiting

```nginx
http {
    # Define rate limit zones
    limit_req_zone $binary_remote_addr zone=api:10m rate=100r/m;
    limit_req_zone $binary_remote_addr zone=login:10m rate=5r/m;

    server {
        location /api/ {
            limit_req zone=api burst=20 nodelay;
            limit_req_status 429;
            proxy_pass http://nodejs_backend;
        }

        location /auth/login {
            limit_req zone=login burst=3 nodelay;
            limit_req_status 429;
            proxy_pass http://nodejs_backend;
        }
    }
}
```

---

## 3. Node.js Cluster Module

### Basic Cluster

```javascript
// cluster.js
const cluster = require('cluster');
const os = require('os');
const process = require('process');

const numCPUs = os.cpus().length;

if (cluster.isPrimary) {
  console.log(`Primary ${process.pid} is running`);
  console.log(`Starting ${numCPUs} workers...`);

  // Fork workers
  for (let i = 0; i < numCPUs; i++) {
    cluster.fork();
  }

  cluster.on('exit', (worker, code, signal) => {
    console.log(`Worker ${worker.process.pid} died (code: ${code}, signal: ${signal})`);
    
    // Auto restart worker
    if (code !== 0) {
      console.log('Restarting worker...');
      cluster.fork();
    }
  });

  cluster.on('online', (worker) => {
    console.log(`Worker ${worker.process.pid} is online`);
  });

} else {
  // Worker process
  const app = require('./app');
  
  app.listen(process.env.PORT || 3000, () => {
    console.log(`Worker ${process.pid} started on port ${process.env.PORT || 3000}`);
  });
}
```

### Advanced Cluster with Graceful Shutdown

```javascript
// cluster-manager.js
const cluster = require('cluster');
const os = require('os');
const EventEmitter = require('events');

class ClusterManager extends EventEmitter {
  constructor(options = {}) {
    super();
    this.numWorkers = options.numWorkers || os.cpus().length;
    this.restartDelay = options.restartDelay || 1000;
    this.maxRestarts = options.maxRestarts || 10;
    this.workers = new Map();
    this.restartCounts = new Map();
  }

  start(workerScript) {
    if (!cluster.isPrimary) {
      return require(workerScript);
    }

    console.log(`Starting ${this.numWorkers} workers`);
    
    cluster.setupPrimary({ exec: workerScript });

    for (let i = 0; i < this.numWorkers; i++) {
      this.spawnWorker();
    }

    // Handle signals
    process.on('SIGTERM', () => this.gracefulShutdown());
    process.on('SIGINT', () => this.gracefulShutdown());

    cluster.on('exit', (worker, code, signal) => {
      this.workers.delete(worker.id);
      this.emit('workerExit', { pid: worker.process.pid, code, signal });

      if (!worker.exitedAfterDisconnect) {
        this.handleWorkerCrash(worker);
      }
    });

    cluster.on('message', (worker, message) => {
      this.emit('message', { worker, message });
    });
  }

  spawnWorker() {
    const worker = cluster.fork();
    this.workers.set(worker.id, worker);
    
    worker.on('message', (msg) => {
      if (msg.type === 'ready') {
        console.log(`Worker ${worker.process.pid} ready`);
      }
    });

    return worker;
  }

  handleWorkerCrash(deadWorker) {
    const restarts = (this.restartCounts.get(deadWorker.id) || 0) + 1;
    
    if (restarts > this.maxRestarts) {
      console.error(`Worker crashed too many times. Not restarting.`);
      if (this.workers.size === 0) {
        process.exit(1);
      }
      return;
    }

    console.log(`Worker died. Restarting in ${this.restartDelay}ms...`);
    
    setTimeout(() => {
      const newWorker = this.spawnWorker();
      this.restartCounts.set(newWorker.id, restarts);
    }, this.restartDelay);
  }

  async gracefulShutdown() {
    console.log('Graceful shutdown initiated...');
    
    const shutdownPromises = Array.from(this.workers.values()).map(worker => {
      return new Promise((resolve) => {
        worker.send({ type: 'shutdown' });
        
        const timeout = setTimeout(() => {
          worker.kill();
          resolve();
        }, 30000);

        worker.once('exit', () => {
          clearTimeout(timeout);
          resolve();
        });
      });
    });

    await Promise.all(shutdownPromises);
    console.log('All workers stopped. Exiting.');
    process.exit(0);
  }

  broadcastMessage(message) {
    this.workers.forEach(worker => {
      worker.send(message);
    });
  }

  getStats() {
    return {
      numWorkers: this.workers.size,
      workerPids: Array.from(this.workers.values()).map(w => w.process.pid)
    };
  }
}

module.exports = ClusterManager;
```

---

## 4. Sticky Sessions

### ปัญหาของ Stateful Sessions

```
ปัญหา: ถ้า user login ที่ Server A แต่ request ถัดไปไปที่ Server B
Server B ไม่รู้ว่า user login แล้ว เพราะ session อยู่ที่ Server A
```

### Solution 1: Centralized Session Store

```javascript
// session-store-redis.js
const session = require('express-session');
const RedisStore = require('connect-redis').default;
const Redis = require('ioredis');

const redisClient = new Redis({
  host: process.env.REDIS_HOST,
  port: process.env.REDIS_PORT,
  password: process.env.REDIS_PASSWORD,
  retryDelayOnFailover: 100,
  maxRetriesPerRequest: 3
});

const sessionMiddleware = session({
  store: new RedisStore({ client: redisClient }),
  secret: process.env.SESSION_SECRET,
  resave: false,
  saveUninitialized: false,
  cookie: {
    secure: process.env.NODE_ENV === 'production',
    httpOnly: true,
    maxAge: 24 * 60 * 60 * 1000
  }
});

module.exports = sessionMiddleware;
```

### Solution 2: Nginx IP Hash

```nginx
upstream nodejs_backend {
    ip_hash;  # Sticky sessions
    server 127.0.0.1:3001;
    server 127.0.0.1:3002;
    server 127.0.0.1:3003;
}
```

### Solution 3: JWT (Stateless)

```javascript
// ใช้ JWT แทน sessions - ไม่ต้องการ sticky sessions
// เพราะ JWT verify ได้บน server ใดก็ได้

const jwt = require('jsonwebtoken');

app.use((req, res, next) => {
  const token = req.headers.authorization?.replace('Bearer ', '');
  if (token) {
    req.user = jwt.verify(token, process.env.JWT_SECRET);
  }
  next();
});
```

---

## 5. Load Balancer Health Checks

### Active Health Checks (Nginx Plus)

```nginx
upstream backend {
    zone backend 64k;
    server 127.0.0.1:3001;
    server 127.0.0.1:3002;
    
    health_check interval=5s fails=3 passes=2 uri=/health;
}
```

### Passive Health Checks

```nginx
upstream backend {
    server 127.0.0.1:3001 max_fails=3 fail_timeout=30s;
    server 127.0.0.1:3002 max_fails=3 fail_timeout=30s;
}
```

---

## Docker Compose Multi-instance

```yaml
# docker-compose.yml
version: '3.8'

services:
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
    volumes:
      - ./nginx.conf:/etc/nginx/conf.d/default.conf
    depends_on:
      - app1
      - app2
      - app3

  app1:
    build: .
    environment:
      - NODE_ENV=production
      - INSTANCE_ID=1
      - REDIS_URL=redis://redis:6379
    depends_on:
      - redis

  app2:
    build: .
    environment:
      - NODE_ENV=production
      - INSTANCE_ID=2
      - REDIS_URL=redis://redis:6379
    depends_on:
      - redis

  app3:
    build: .
    environment:
      - NODE_ENV=production
      - INSTANCE_ID=3
      - REDIS_URL=redis://redis:6379
    depends_on:
      - redis

  redis:
    image: redis:alpine
    volumes:
      - redis_data:/data

volumes:
  redis_data:
```

---

## แบบฝึกหัด

### ระดับ 1: พื้นฐาน
Setup load balancing:
- 3 Node.js instances
- Nginx round-robin
- ทดสอบด้วย Apache Bench

### ระดับ 2: กลาง
เพิ่ม:
- Redis session store
- Health checks
- Graceful shutdown

### ระดับ 3: ขั้นสูง
Production setup:
- Node.js Cluster
- Nginx + SSL
- Zero-downtime deployment

---

## สรุป

Load Balancing เป็นส่วนสำคัญของ scaling infrastructure สำหรับ stateless applications (JWT) ใช้ Round Robin ง่ายที่สุด สำหรับ stateful apps ควรใช้ centralized session store (Redis) แทน sticky sessions เพื่อ availability ที่ดีกว่า

> ขั้นตอนต่อไป: Part 70 - Kubernetes
