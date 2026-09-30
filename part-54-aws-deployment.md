# Part 54: AWS Deployment (การ Deploy บน AWS)
## ขั้นตอนที่ 54-54 จาก 1000

---

## บทนำ

AWS (Amazon Web Services) เป็น cloud platform ที่ใหญ่ที่สุดในโลก บทนี้จะครอบคลุมการ deploy Node.js application บน AWS services หลัก

---

## 54.1 EC2 Deployment

EC2 (Elastic Compute Cloud) คือ virtual server บน AWS

### ตั้งค่า EC2 Instance

```bash
# 1. Launch EC2 Instance (Ubuntu 22.04)
# ผ่าน AWS Console หรือ CLI

# 2. เชื่อมต่อผ่าน SSH
ssh -i my-key.pem ubuntu@<EC2_PUBLIC_IP>

# 3. ติดตั้ง Node.js
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt-get install -y nodejs

# 4. ติดตั้ง PM2
sudo npm install -g pm2

# 5. ติดตั้ง Nginx
sudo apt-get install -y nginx

# 6. Clone application
git clone https://github.com/yourorg/yourapp.git
cd yourapp
npm install --production
```

### PM2 Ecosystem Config

```javascript
// ecosystem.config.js
module.exports = {
  apps: [{
    name: 'myapp',
    script: 'src/app.js',
    instances: 'max',  // ใช้ CPU cores ทั้งหมด
    exec_mode: 'cluster',
    
    env: {
      NODE_ENV: 'development',
      PORT: 3000
    },
    
    env_production: {
      NODE_ENV: 'production',
      PORT: 3000
    },
    
    // Log configuration
    log_file: '/var/log/pm2/myapp.log',
    error_file: '/var/log/pm2/myapp-error.log',
    merge_logs: true,
    
    // Auto restart on file change
    watch: false,
    
    // Memory limit
    max_memory_restart: '1G',
    
    // Graceful shutdown
    kill_timeout: 5000,
    
    // Health monitoring
    min_uptime: '5s',
    max_restarts: 10
  }]
};
```

```bash
# Deploy commands
pm2 start ecosystem.config.js --env production
pm2 save
pm2 startup  # Auto-start on server reboot
```

### Nginx Configuration

```nginx
# /etc/nginx/sites-available/myapp
server {
    listen 80;
    server_name api.example.com;
    
    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_cache_bypass $http_upgrade;
    }
}
```

```bash
# Enable site
sudo ln -s /etc/nginx/sites-available/myapp /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx

# SSL with Let's Encrypt
sudo apt install certbot python3-certbot-nginx
sudo certbot --nginx -d api.example.com
```

---

## 54.2 Elastic Beanstalk

Elastic Beanstalk (EB) ช่วย deploy และ manage applications โดยอัตโนมัติ

### ติดตั้ง EB CLI

```bash
pip install awsebcli
eb init --platform node.js --region ap-southeast-1
```

### Procfile

```
# Procfile
web: node src/app.js
```

### .ebextensions Configuration

```yaml
# .ebextensions/01-environment.config
option_settings:
  aws:elasticbeanstalk:application:environment:
    NODE_ENV: production
    PORT: 8080

  aws:elasticbeanstalk:container:nodejs:
    NodeCommand: "npm start"
    NodeVersion: 20

  aws:autoscaling:asg:
    MinSize: 2
    MaxSize: 5

  aws:elb:loadbalancer:
    LoadBalancerHTTPSPort: 443

  aws:elasticbeanstalk:healthreporting:system:
    SystemType: enhanced
```

```yaml
# .ebextensions/02-install-packages.config
packages:
  yum:
    git: []
    
commands:
  01_install_npm_packages:
    command: "npm install --production"
    cwd: /var/app/current
```

### Deploy Commands

```bash
# สร้าง environment
eb create production --tier web --elb-type application

# Deploy
eb deploy

# ดูสถานะ
eb status

# ดู logs
eb logs

# Scale
eb scale 3  # เพิ่มเป็น 3 instances

# Open app
eb open
```

---

## 54.3 Lambda Functions

AWS Lambda ช่วยรัน code โดยไม่ต้องจัดการ server

### Serverless Node.js Function

```javascript
// handler.js

/**
 * Basic Lambda handler
 */
exports.handler = async (event, context) => {
  console.log('Event:', JSON.stringify(event));
  
  try {
    const result = await processEvent(event);
    
    return {
      statusCode: 200,
      headers: {
        'Content-Type': 'application/json',
        'Access-Control-Allow-Origin': '*'
      },
      body: JSON.stringify(result)
    };
  } catch (error) {
    console.error('Error:', error);
    
    return {
      statusCode: 500,
      body: JSON.stringify({ error: error.message })
    };
  }
};

const processEvent = async (event) => {
  if (event.httpMethod) {
    // API Gateway event
    return handleApiRequest(event);
  }
  
  if (event.Records) {
    // SQS, S3, DynamoDB Stream events
    return handleStreamEvent(event.Records);
  }
  
  return { message: 'Event processed' };
};

const handleApiRequest = async (event) => {
  const { httpMethod, path, body, queryStringParameters } = event;
  
  switch (`${httpMethod} ${path}`) {
    case 'GET /users':
      return await getUsers(queryStringParameters);
    case 'POST /users':
      return await createUser(JSON.parse(body));
    default:
      throw new Error(`Not found: ${httpMethod} ${path}`);
  }
};
```

### Express.js บน Lambda (Serverless)

```javascript
// serverless-app.js
const serverless = require('serverless-http');  // npm install serverless-http
const express = require('express');
const app = express();

app.use(express.json());

app.get('/api/users', async (req, res) => {
  const users = await User.findAll();
  res.json(users);
});

app.post('/api/users', async (req, res) => {
  const user = await User.create(req.body);
  res.status(201).json(user);
});

// Export สำหรับ Lambda
module.exports.handler = serverless(app);
```

### serverless.yml (Serverless Framework)

```yaml
# serverless.yml
service: myapp

frameworkVersion: '3'

provider:
  name: aws
  runtime: nodejs20.x
  region: ap-southeast-1
  stage: ${opt:stage, 'dev'}
  
  environment:
    DB_HOST: ${env:DB_HOST}
    DB_NAME: ${env:DB_NAME}
    JWT_SECRET: ${env:JWT_SECRET}
  
  iam:
    role:
      statements:
        - Effect: Allow
          Action:
            - s3:GetObject
            - s3:PutObject
          Resource: 'arn:aws:s3:::${self:custom.bucketName}/*'

functions:
  api:
    handler: serverless-app.handler
    events:
      - http:
          path: /api/{proxy+}
          method: ANY
          cors: true
    timeout: 30
    memorySize: 512

  processImage:
    handler: handlers/imageProcessor.handler
    events:
      - s3:
          bucket: ${self:custom.bucketName}
          event: s3:ObjectCreated:*
          rules:
            - prefix: uploads/
    timeout: 300
    memorySize: 1024

custom:
  bucketName: myapp-${self:provider.stage}-uploads

plugins:
  - serverless-offline  # Local development
```

---

## 54.4 RDS Database

```javascript
// src/config/rds.js
const { Sequelize } = require('sequelize');

const sequelize = new Sequelize(
  process.env.DB_NAME,
  process.env.DB_USER,
  process.env.DB_PASSWORD,
  {
    host: process.env.DB_HOST,
    port: process.env.DB_PORT || 5432,
    dialect: 'postgres',
    
    // SSL สำหรับ RDS
    dialectOptions: {
      ssl: process.env.NODE_ENV === 'production' ? {
        require: true,
        rejectUnauthorized: false
      } : false
    },
    
    // Connection pooling
    pool: {
      max: 10,
      min: 2,
      acquire: 30000,
      idle: 10000
    },
    
    logging: process.env.NODE_ENV !== 'production',
    
    // Read replicas
    replication: process.env.DB_READ_HOST ? {
      read: [{ host: process.env.DB_READ_HOST }],
      write: { host: process.env.DB_HOST }
    } : undefined
  }
);

module.exports = sequelize;
```

---

## 54.5 S3 Storage

```javascript
// src/services/s3Service.js
const { S3Client, PutObjectCommand, GetObjectCommand, DeleteObjectCommand } = require('@aws-sdk/client-s3');
const { getSignedUrl } = require('@aws-sdk/s3-request-presigner');
const { v4: uuidv4 } = require('uuid');

const s3Client = new S3Client({
  region: process.env.AWS_REGION || 'ap-southeast-1',
  credentials: {
    accessKeyId: process.env.AWS_ACCESS_KEY_ID,
    secretAccessKey: process.env.AWS_SECRET_ACCESS_KEY
  }
});

const BUCKET_NAME = process.env.S3_BUCKET_NAME;

/**
 * Upload file ไปยัง S3
 */
const uploadToS3 = async (fileBuffer, options = {}) => {
  const {
    folder = 'uploads',
    filename = `${uuidv4()}`,
    contentType = 'application/octet-stream',
    isPublic = false
  } = options;
  
  const key = `${folder}/${filename}`;
  
  const command = new PutObjectCommand({
    Bucket: BUCKET_NAME,
    Key: key,
    Body: fileBuffer,
    ContentType: contentType,
    ACL: isPublic ? 'public-read' : 'private',
    CacheControl: 'max-age=31536000'
  });
  
  await s3Client.send(command);
  
  return {
    key,
    url: isPublic 
      ? `https://${BUCKET_NAME}.s3.amazonaws.com/${key}`
      : null
  };
};

/**
 * สร้าง pre-signed URL สำหรับ private files
 */
const getPresignedUrl = async (key, expiresInSeconds = 3600) => {
  const command = new GetObjectCommand({
    Bucket: BUCKET_NAME,
    Key: key
  });
  
  return await getSignedUrl(s3Client, command, {
    expiresIn: expiresInSeconds
  });
};

/**
 * สร้าง pre-signed URL สำหรับ upload โดยตรงจาก client
 */
const getPresignedUploadUrl = async (key, contentType, expiresInSeconds = 300) => {
  const command = new PutObjectCommand({
    Bucket: BUCKET_NAME,
    Key: key,
    ContentType: contentType
  });
  
  const uploadUrl = await getSignedUrl(s3Client, command, {
    expiresIn: expiresInSeconds
  });
  
  return { uploadUrl, key };
};

/**
 * ลบไฟล์จาก S3
 */
const deleteFromS3 = async (key) => {
  const command = new DeleteObjectCommand({
    Bucket: BUCKET_NAME,
    Key: key
  });
  
  await s3Client.send(command);
};

module.exports = { uploadToS3, getPresignedUrl, getPresignedUploadUrl, deleteFromS3 };
```

### Controller สำหรับ S3 Upload

```javascript
// src/controllers/uploadController.js
const multer = require('multer');
const sharp = require('sharp');
const { uploadToS3, getPresignedUrl } = require('../services/s3Service');

const storage = multer.memoryStorage();
const upload = multer({ storage, limits: { fileSize: 10 * 1024 * 1024 } });

const uploadFile = async (req, res) => {
  try {
    if (!req.file) {
      return res.status(400).json({ error: 'No file provided' });
    }
    
    let buffer = req.file.buffer;
    
    // Process image ถ้าเป็น image
    if (req.file.mimetype.startsWith('image/')) {
      buffer = await sharp(buffer)
        .resize(1920, null, { fit: 'inside', withoutEnlargement: true })
        .webp({ quality: 85 })
        .toBuffer();
      req.file.mimetype = 'image/webp';
    }
    
    const { key, url } = await uploadToS3(buffer, {
      folder: 'uploads',
      filename: `${Date.now()}-${req.file.originalname.replace(/[^a-zA-Z0-9.]/g, '_')}`,
      contentType: req.file.mimetype,
      isPublic: false
    });
    
    // สร้าง pre-signed URL สำหรับ access
    const accessUrl = await getPresignedUrl(key, 24 * 3600);
    
    res.json({
      success: true,
      data: {
        key,
        url: accessUrl,
        contentType: req.file.mimetype,
        size: buffer.length
      }
    });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
};

module.exports = { uploadFile, upload };
```

---

## แบบฝึกหัดที่ 54

### แบบฝึกหัดพื้นฐาน

**1. EC2 Deployment**

Deploy Node.js app บน EC2:
- ติดตั้ง Node.js, PM2, Nginx
- Configure HTTPS ด้วย Let's Encrypt
- ตั้งค่า auto-restart

**2. S3 File Upload**

สร้าง API ที่:
- Upload files ไป S3
- สร้าง pre-signed URLs
- Process images ก่อน upload

### แบบฝึกหัดขั้นสูง

**3. Serverless API**

แปลง Express app เป็น Lambda:
- ใช้ serverless-http
- Deploy ด้วย Serverless Framework
- ตั้งค่า API Gateway

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **EC2** - manual deployment, PM2, Nginx
2. **Elastic Beanstalk** - managed deployment
3. **Lambda** - serverless functions
4. **RDS** - managed PostgreSQL
5. **S3** - file storage, pre-signed URLs

**ถัดไป:** Part 55 - Performance Optimization
