# Part 52: Docker (การใช้งาน Docker)
## ขั้นตอนที่ 52-52 จาก 1000

---

## บทนำ

Docker เป็น containerization platform ที่ช่วยให้เรา package แอปพลิเคชันพร้อม dependencies ทั้งหมดลงใน container ที่พกพาได้

---

## 52.1 Docker Concepts

### ทำไมต้องใช้ Docker?

**ปัญหาก่อน Docker:**
```
"มันรันได้บน machine ฉันนะ แต่ทำไม production ไม่ทำงาน?"

สาเหตุ:
- Node.js version ต่างกัน (local: 18, server: 14)
- OS แตกต่าง (Mac vs Linux)
- Environment variables ขาดหาย
- Dependencies version conflict
```

**ด้วย Docker:**
```
Container = App + Runtime + Dependencies + Config
→ รันเหมือนกันทุกที่
```

### Docker Architecture

```
┌─────────────────────────────────────┐
│          Docker Host                │
│  ┌──────────┐  ┌──────────┐        │
│  │Container1│  │Container2│        │
│  │  App A   │  │  App B   │        │
│  └──────────┘  └──────────┘        │
│  ┌────────────────────────────┐     │
│  │     Docker Engine          │     │
│  └────────────────────────────┘     │
│  ┌────────────────────────────┐     │
│  │        Host OS             │     │
│  └────────────────────────────┘     │
└─────────────────────────────────────┘
```

### คำศัพท์สำคัญ

| คำศัพท์ | คำอธิบาย |
|---------|---------|
| Image | Blueprint สำหรับสร้าง container |
| Container | Instance ที่ running จาก Image |
| Dockerfile | Script สำหรับสร้าง Image |
| Registry | ที่เก็บ Images (Docker Hub) |
| Volume | Persistent storage |
| Network | การสื่อสารระหว่าง containers |

---

## 52.2 Dockerfile สำหรับ Node.js

### Dockerfile พื้นฐาน

```dockerfile
# Dockerfile
FROM node:20-alpine

# ตั้ง working directory
WORKDIR /app

# Copy package files ก่อน (layer caching)
COPY package*.json ./

# ติดตั้ง dependencies
RUN npm install

# Copy source code
COPY . .

# เปิด port
EXPOSE 3000

# รันแอป
CMD ["node", "src/app.js"]
```

### .dockerignore

```
# .dockerignore
node_modules
npm-debug.log
.git
.gitignore
.env
.env.*
*.md
coverage/
.nyc_output/
dist/
build/
logs/
*.log
```

### Build และ Run

```bash
# Build image
docker build -t myapp:1.0 .

# Run container
docker run -p 3000:3000 myapp:1.0

# Run ใน background
docker run -d -p 3000:3000 --name myapp myapp:1.0

# ดู running containers
docker ps

# ดู logs
docker logs myapp -f

# เข้าไปใน container
docker exec -it myapp sh

# หยุด container
docker stop myapp

# ลบ container
docker rm myapp
```

---

## 52.3 Multi-stage Builds

Multi-stage builds ช่วยลดขนาด final image โดยแยก build stage ออกจาก production stage

### Multi-stage Dockerfile สำหรับ Node.js

```dockerfile
# Dockerfile (multi-stage)

# Stage 1: Builder
FROM node:20-alpine AS builder

WORKDIR /app

COPY package*.json ./
RUN npm ci  # ใช้ ci แทน install เพื่อ reproducibility

COPY . .
RUN npm run build  # compile TypeScript, bundle, etc.

# Stage 2: Production
FROM node:20-alpine AS production

# ติดตั้ง security updates
RUN apk update && apk upgrade && apk add --no-cache dumb-init

# สร้าง non-root user
RUN addgroup -g 1001 nodejs && \
    adduser -S -u 1001 -G nodejs nodeuser

WORKDIR /app

# Copy เฉพาะ production dependencies
COPY package*.json ./
RUN npm ci --only=production && npm cache clean --force

# Copy built files จาก builder stage
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/public ./public

# เปลี่ยน ownership
RUN chown -R nodeuser:nodejs /app

# Switch to non-root user
USER nodeuser

EXPOSE 3000

# ใช้ dumb-init เพื่อ handle signals properly
ENTRYPOINT ["dumb-init", "--"]
CMD ["node", "dist/app.js"]
```

### Dockerfile สำหรับ TypeScript

```dockerfile
# Dockerfile.typescript
FROM node:20-alpine AS builder

WORKDIR /app

COPY package*.json tsconfig*.json ./
RUN npm ci

COPY src ./src
RUN npm run build  # tsc

# Production stage
FROM node:20-alpine AS production

RUN apk add --no-cache dumb-init

WORKDIR /app

COPY package*.json ./
RUN npm ci --only=production && npm cache clean --force

COPY --from=builder /app/dist ./dist

# Environment
ENV NODE_ENV=production
ENV PORT=3000

EXPOSE 3000

CMD ["dumb-init", "node", "dist/app.js"]
```

---

## 52.4 Docker Compose

Docker Compose ช่วยจัดการหลาย containers พร้อมกัน

### docker-compose.yml พื้นฐาน

```yaml
# docker-compose.yml
version: '3.8'

services:
  app:
    build: .
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=development
      - PORT=3000
      - DB_HOST=postgres
      - DB_PORT=5432
      - DB_NAME=myapp
      - DB_USER=admin
      - DB_PASSWORD=secret
      - REDIS_HOST=redis
      - REDIS_PORT=6379
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    volumes:
      - ./src:/app/src  # Hot reload
      - /app/node_modules  # ไม่ mount node_modules จาก host
    networks:
      - app-network

  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: myapp
      POSTGRES_USER: admin
      POSTGRES_PASSWORD: secret
    ports:
      - "5432:5432"
    volumes:
      - postgres-data:/var/lib/postgresql/data
      - ./scripts/init.sql:/docker-entrypoint-initdb.d/init.sql
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U admin -d myapp"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - app-network

  redis:
    image: redis:7-alpine
    command: redis-server --appendonly yes --requirepass redispassword
    ports:
      - "6379:6379"
    volumes:
      - redis-data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - app-network

  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf
      - ./nginx/ssl:/etc/nginx/ssl
    depends_on:
      - app
    networks:
      - app-network

volumes:
  postgres-data:
  redis-data:

networks:
  app-network:
    driver: bridge
```

### Docker Compose Commands

```bash
# Start services
docker-compose up -d

# Build และ start
docker-compose up -d --build

# Stop services
docker-compose down

# Stop และ ลบ volumes
docker-compose down -v

# ดู logs
docker-compose logs -f app

# Exec ใน service
docker-compose exec app sh

# Scale service
docker-compose up -d --scale app=3

# ดู status
docker-compose ps
```

---

## 52.5 Environment Configs

### ใช้ .env files

```bash
# .env.development
NODE_ENV=development
PORT=3000
DB_HOST=localhost
DB_PORT=5432
DB_NAME=myapp_dev
DB_USER=admin
DB_PASSWORD=devpassword
REDIS_URL=redis://localhost:6379
JWT_SECRET=dev-secret-key
LOG_LEVEL=debug

# .env.production
NODE_ENV=production
PORT=3000
DB_HOST=prod-db-host.rds.amazonaws.com
DB_PORT=5432
DB_NAME=myapp_prod
DB_USER=prod_user
DB_PASSWORD=${DB_PASSWORD}  # จาก secrets
REDIS_URL=${REDIS_URL}
JWT_SECRET=${JWT_SECRET}
LOG_LEVEL=warn
```

### docker-compose.override.yml (Development)

```yaml
# docker-compose.override.yml
# Auto-loaded เมื่อรัน docker-compose up
version: '3.8'

services:
  app:
    build:
      context: .
      target: development  # ใช้ stage ที่ชื่อ development
    command: npm run dev  # nodemon
    environment:
      - NODE_ENV=development
      - DEBUG=app:*
    volumes:
      - .:/app
      - /app/node_modules

  # เพิ่ม tools สำหรับ development
  adminer:
    image: adminer
    ports:
      - "8080:8080"
    networks:
      - app-network

  redis-commander:
    image: rediscommander/redis-commander
    environment:
      - REDIS_HOSTS=local:redis:6379
    ports:
      - "8081:8081"
    networks:
      - app-network
```

### docker-compose.prod.yml (Production)

```yaml
# docker-compose.prod.yml
version: '3.8'

services:
  app:
    image: myapp:${VERSION:-latest}
    restart: unless-stopped
    deploy:
      replicas: 3
      resources:
        limits:
          cpus: '0.5'
          memory: 512M
        reservations:
          cpus: '0.25'
          memory: 256M
    environment:
      - NODE_ENV=production
    secrets:
      - db_password
      - jwt_secret

  postgres:
    image: postgres:16-alpine
    restart: unless-stopped
    volumes:
      - postgres-data:/var/lib/postgresql/data
    environment:
      POSTGRES_PASSWORD_FILE: /run/secrets/db_password
    secrets:
      - db_password

secrets:
  db_password:
    external: true
  jwt_secret:
    external: true
```

---

## 52.6 Nginx Configuration

```nginx
# nginx/nginx.conf
upstream nodejs_app {
    least_conn;
    server app:3000;
}

server {
    listen 80;
    server_name example.com;
    
    # Redirect HTTP to HTTPS
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl http2;
    server_name example.com;
    
    ssl_certificate /etc/nginx/ssl/cert.pem;
    ssl_certificate_key /etc/nginx/ssl/key.pem;
    
    # Gzip
    gzip on;
    gzip_types text/plain application/json application/javascript text/css;
    
    # Security headers
    add_header X-Frame-Options DENY;
    add_header X-Content-Type-Options nosniff;
    add_header X-XSS-Protection "1; mode=block";
    
    location / {
        proxy_pass http://nodejs_app;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_cache_bypass $http_upgrade;
        
        # Timeout settings
        proxy_connect_timeout 60s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
    }
    
    # Static files
    location /uploads/ {
        alias /var/www/uploads/;
        expires 30d;
        add_header Cache-Control "public, immutable";
    }
}
```

---

## แบบฝึกหัดที่ 52

### แบบฝึกหัดพื้นฐาน

**1. Dockerize Node.js App**

สร้าง Dockerfile สำหรับ Express.js app:
- Multi-stage build
- Non-root user
- Health check endpoint

**2. Docker Compose Stack**

สร้าง docker-compose.yml ที่มี:
- Node.js app
- PostgreSQL database
- Redis cache
- Nginx reverse proxy

### แบบฝึกหัดขั้นสูง

**3. Development vs Production Config**

สร้าง docker-compose stack ที่แยก config:
- Development: hot reload, debug tools
- Production: optimized, secure
- Environment variables management

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **Docker Concepts** - images, containers, volumes
2. **Dockerfile** - best practices สำหรับ Node.js
3. **Multi-stage Builds** - ลดขนาด production image
4. **Docker Compose** - จัดการหลาย services
5. **Environment Configs** - dev vs prod

**ถัดไป:** Part 53 - CI/CD
