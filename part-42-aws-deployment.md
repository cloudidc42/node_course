# Part 42 | ขั้นตอนที่ 741-760 จาก 1000

## AWS Deployment - การ Deploy Node.js บน Amazon Web Services

---

## สารบัญ

1. [AWS Overview สำหรับ Node.js](#aws-overview-สำหรับ-nodejs)
2. [EC2 - Virtual Servers](#ec2---virtual-servers)
3. [RDS - Managed Database](#rds---managed-database)
4. [S3 - Object Storage](#s3---object-storage)
5. [Lambda - Serverless Functions](#lambda---serverless-functions)
6. [Elastic Beanstalk](#elastic-beanstalk)
7. [ECS - Container Service](#ecs---container-service)
8. [CloudFront - CDN](#cloudfront---cdn)
9. [IAM - Identity and Access Management](#iam---identity-and-access-management)
10. [แบบฝึกหัด](#แบบฝึกหัด)

---

## AWS Overview สำหรับ Node.js

### ขั้นตอนที่ 741: AWS Services ที่ใช้กับ Node.js

```
AWS Services สำคัญสำหรับ Node.js Developer:

Compute:
  EC2      → Virtual machines (เต็มรูปแบบ)
  Lambda   → Serverless functions
  ECS/EKS  → Container orchestration
  Beanstalk → Managed app platform

Database:
  RDS      → Relational databases (PostgreSQL, MySQL)
  DynamoDB → NoSQL database
  ElastiCache → Redis/Memcached

Storage:
  S3       → Object storage
  EFS      → Elastic File System

Networking:
  VPC      → Virtual Private Cloud
  ALB      → Application Load Balancer
  Route 53 → DNS management
  CloudFront → CDN

Security:
  IAM      → Identity and Access Management
  Secrets Manager → Secret storage
  Certificate Manager → SSL certificates

Monitoring:
  CloudWatch → Logs, metrics, alarms
  X-Ray    → Distributed tracing
```

### ขั้นตอนที่ 742: ติดตั้ง AWS CLI และ SDK

```bash
# ติดตั้ง AWS CLI
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

# Configure credentials
aws configure
# AWS Access Key ID: your-access-key
# AWS Secret Access Key: your-secret-key
# Default region: ap-southeast-1
# Default output format: json

# ติดตั้ง AWS SDK สำหรับ Node.js
npm install @aws-sdk/client-s3 @aws-sdk/client-ec2 @aws-sdk/client-rds
```

---

## EC2 - Virtual Servers

### ขั้นตอนที่ 743: สร้างและตั้งค่า EC2 Instance

```bash
# สร้าง Key Pair
aws ec2 create-key-pair \
  --key-name myapp-key \
  --query 'KeyMaterial' \
  --output text > myapp-key.pem
chmod 400 myapp-key.pem

# สร้าง Security Group
aws ec2 create-security-group \
  --group-name myapp-sg \
  --description "Security group for Node.js app"

# เปิด ports
aws ec2 authorize-security-group-ingress \
  --group-name myapp-sg \
  --protocol tcp --port 22 --cidr 0.0.0.0/0   # SSH

aws ec2 authorize-security-group-ingress \
  --group-name myapp-sg \
  --protocol tcp --port 80 --cidr 0.0.0.0/0   # HTTP

aws ec2 authorize-security-group-ingress \
  --group-name myapp-sg \
  --protocol tcp --port 443 --cidr 0.0.0.0/0  # HTTPS

# Launch EC2 instance
aws ec2 run-instances \
  --image-id ami-0df7a207adb9748c7 \
  --instance-type t3.small \
  --key-name myapp-key \
  --security-groups myapp-sg \
  --user-data file://user-data.sh \
  --count 1 \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=myapp-server}]'
```

```bash
#!/bin/bash
# user-data.sh - ติดตั้ง server เมื่อ launch

# อัปเดต system
yum update -y

# ติดตั้ง Node.js 18
curl -fsSL https://rpm.nodesource.com/setup_18.x | bash -
yum install -y nodejs

# ติดตั้ง PM2
npm install -g pm2

# ติดตั้ง Git
yum install -y git

# ติดตั้ง Nginx
yum install -y nginx
systemctl start nginx
systemctl enable nginx

# Clone application
cd /home/ec2-user
git clone https://github.com/username/myapp.git
cd myapp
npm ci --only=production

# Start application
pm2 start ecosystem.config.js
pm2 startup
pm2 save

echo "Server setup complete!"
```

### ขั้นตอนที่ 744: PM2 Configuration

```javascript
// ecosystem.config.js
module.exports = {
  apps: [{
    name: 'myapp',
    script: 'src/index.js',
    instances: 'max',    // ใช้ทุก CPU cores
    exec_mode: 'cluster',
    watch: false,

    env: {
      NODE_ENV: 'development',
    },
    env_production: {
      NODE_ENV: 'production',
      PORT: 3000,
    },

    // Logs
    log_date_format: 'YYYY-MM-DD HH:mm:ss',
    error_file: '/var/log/myapp/error.log',
    out_file: '/var/log/myapp/out.log',
    merge_logs: true,

    // Restart policy
    max_memory_restart: '512M',
    restart_delay: 3000,
    max_restarts: 10,

    // Health monitoring
    min_uptime: 5000,
  }],
};
```

### ขั้นตอนที่ 745: Auto Scaling Group

```bash
# สร้าง Launch Template
aws ec2 create-launch-template \
  --launch-template-name myapp-template \
  --launch-template-data '{
    "ImageId": "ami-0df7a207adb9748c7",
    "InstanceType": "t3.small",
    "KeyName": "myapp-key",
    "SecurityGroupIds": ["sg-xxxxx"],
    "UserData": "...",
    "IamInstanceProfile": {
      "Name": "myapp-instance-profile"
    }
  }'

# สร้าง Auto Scaling Group
aws autoscaling create-auto-scaling-group \
  --auto-scaling-group-name myapp-asg \
  --launch-template LaunchTemplateName=myapp-template,Version='$Latest' \
  --min-size 2 \
  --max-size 10 \
  --desired-capacity 2 \
  --vpc-zone-identifier "subnet-xxx,subnet-yyy" \
  --target-group-arns "arn:aws:elasticloadbalancing:..."

# สร้าง Scaling Policies
aws autoscaling put-scaling-policy \
  --auto-scaling-group-name myapp-asg \
  --policy-name scale-up \
  --scaling-adjustment 2 \
  --adjustment-type ChangeInCapacity \
  --cooldown 300
```

---

## RDS - Managed Database

### ขั้นตอนที่ 746: สร้าง RDS Instance

```bash
# สร้าง DB Subnet Group
aws rds create-db-subnet-group \
  --db-subnet-group-name myapp-subnet \
  --db-subnet-group-description "MyApp DB Subnet" \
  --subnet-ids subnet-xxx subnet-yyy

# สร้าง RDS PostgreSQL
aws rds create-db-instance \
  --db-instance-identifier myapp-db \
  --db-instance-class db.t3.small \
  --engine postgres \
  --engine-version 15.3 \
  --master-username dbadmin \
  --master-user-password ${DB_PASSWORD} \
  --db-name myappdb \
  --db-subnet-group-name myapp-subnet \
  --vpc-security-group-ids sg-xxxxx \
  --multi-az \
  --storage-type gp3 \
  --allocated-storage 20 \
  --backup-retention-period 7 \
  --deletion-protection \
  --enable-cloudwatch-logs-exports postgresql
```

### ขั้นตอนที่ 747: เชื่อมต่อ Node.js กับ RDS

```javascript
// src/config/database.js
const { Sequelize } = require('sequelize');

const sequelize = new Sequelize({
  host: process.env.RDS_HOST,
  port: parseInt(process.env.RDS_PORT) || 5432,
  database: process.env.RDS_DATABASE,
  username: process.env.RDS_USERNAME,
  password: process.env.RDS_PASSWORD,
  dialect: 'postgres',

  pool: {
    max: 10,
    min: 2,
    acquire: 30000,
    idle: 10000,
  },

  // SSL สำหรับ RDS
  dialectOptions: {
    ssl: {
      require: true,
      rejectUnauthorized: false,
      // ใช้ RDS Certificate ใน production
      // ca: fs.readFileSync('/app/certs/rds-ca.pem').toString(),
    },
  },

  logging: process.env.NODE_ENV === 'development' ? console.log : false,
});

// Connection pool monitoring
sequelize.pool.on('acquire', () => {
  console.log('Connection acquired from pool');
});

module.exports = sequelize;
```

---

## S3 - Object Storage

### ขั้นตอนที่ 748: ตั้งค่า S3 Bucket

```javascript
// src/config/s3.js
const { S3Client } = require('@aws-sdk/client-s3');

const s3Client = new S3Client({
  region: process.env.AWS_REGION || 'ap-southeast-1',
  credentials: {
    accessKeyId: process.env.AWS_ACCESS_KEY_ID,
    secretAccessKey: process.env.AWS_SECRET_ACCESS_KEY,
  },
});

module.exports = s3Client;
```

### ขั้นตอนที่ 749: File Upload ไปยัง S3

```javascript
// src/services/s3Service.js
const {
  S3Client,
  PutObjectCommand,
  GetObjectCommand,
  DeleteObjectCommand,
  ListObjectsV2Command,
  CreatePresignedUrlCommand,
} = require('@aws-sdk/client-s3');
const { getSignedUrl } = require('@aws-sdk/s3-request-presigner');
const { v4: uuidv4 } = require('uuid');
const path = require('path');

const s3Client = require('../config/s3');
const BUCKET_NAME = process.env.S3_BUCKET_NAME;

class S3Service {
  // Upload file
  async uploadFile(file, folder = 'uploads') {
    const fileExt = path.extname(file.originalname);
    const fileName = `${folder}/${uuidv4()}${fileExt}`;

    const command = new PutObjectCommand({
      Bucket: BUCKET_NAME,
      Key: fileName,
      Body: file.buffer,
      ContentType: file.mimetype,
      // Metadata
      Metadata: {
        originalName: file.originalname,
        uploadedAt: new Date().toISOString(),
      },
      // ตั้งค่า caching headers
      CacheControl: 'max-age=31536000',
      // ServerSideEncryption: 'AES256',  // Encrypt at rest
    });

    await s3Client.send(command);

    return {
      key: fileName,
      url: `https://${BUCKET_NAME}.s3.${process.env.AWS_REGION}.amazonaws.com/${fileName}`,
    };
  }

  // Upload หลายไฟล์พร้อมกัน
  async uploadMultipleFiles(files, folder = 'uploads') {
    const uploads = files.map(file => this.uploadFile(file, folder));
    return Promise.all(uploads);
  }

  // สร้าง presigned URL สำหรับ direct upload จาก browser
  async getUploadPresignedUrl(fileName, contentType, expiresIn = 300) {
    const key = `uploads/${uuidv4()}-${fileName}`;

    const command = new PutObjectCommand({
      Bucket: BUCKET_NAME,
      Key: key,
      ContentType: contentType,
    });

    const url = await getSignedUrl(s3Client, command, { expiresIn });

    return { url, key };
  }

  // สร้าง presigned URL สำหรับ download
  async getDownloadPresignedUrl(key, expiresIn = 3600) {
    const command = new GetObjectCommand({
      Bucket: BUCKET_NAME,
      Key: key,
    });

    return getSignedUrl(s3Client, command, { expiresIn });
  }

  // ลบไฟล์
  async deleteFile(key) {
    const command = new DeleteObjectCommand({
      Bucket: BUCKET_NAME,
      Key: key,
    });

    await s3Client.send(command);
  }

  // List files
  async listFiles(prefix = '', maxKeys = 100) {
    const command = new ListObjectsV2Command({
      Bucket: BUCKET_NAME,
      Prefix: prefix,
      MaxKeys: maxKeys,
    });

    const response = await s3Client.send(command);
    return response.Contents || [];
  }

  // Copy file
  async copyFile(sourceKey, destinationKey) {
    const { CopyObjectCommand } = require('@aws-sdk/client-s3');
    const command = new CopyObjectCommand({
      Bucket: BUCKET_NAME,
      CopySource: `${BUCKET_NAME}/${sourceKey}`,
      Key: destinationKey,
    });

    await s3Client.send(command);
  }
}

module.exports = new S3Service();
```

### ขั้นตอนที่ 750: File Upload Route

```javascript
// src/routes/upload.js
const express = require('express');
const multer = require('multer');
const s3Service = require('../services/s3Service');

const router = express.Router();

// ใช้ memory storage (เก็บใน RAM ก่อน upload S3)
const upload = multer({
  storage: multer.memoryStorage(),
  limits: {
    fileSize: 5 * 1024 * 1024,  // 5MB
  },
  fileFilter: (req, file, cb) => {
    const allowedTypes = ['image/jpeg', 'image/png', 'image/gif', 'image/webp'];
    if (allowedTypes.includes(file.mimetype)) {
      cb(null, true);
    } else {
      cb(new Error('Invalid file type'));
    }
  },
});

// Upload single file
router.post('/upload', requireAuth, upload.single('file'), async (req, res) => {
  try {
    if (!req.file) {
      return res.status(400).json({ error: 'No file provided' });
    }

    const result = await s3Service.uploadFile(req.file, 'images');

    res.json({
      success: true,
      url: result.url,
      key: result.key,
    });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Upload หลายไฟล์
router.post('/upload-multiple', requireAuth, upload.array('files', 10), async (req, res) => {
  try {
    if (!req.files || req.files.length === 0) {
      return res.status(400).json({ error: 'No files provided' });
    }

    const results = await s3Service.uploadMultipleFiles(req.files);

    res.json({ success: true, files: results });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Get presigned URL สำหรับ direct upload
router.post('/presigned-url', requireAuth, async (req, res) => {
  try {
    const { fileName, contentType } = req.body;
    const result = await s3Service.getUploadPresignedUrl(fileName, contentType);

    res.json({
      uploadUrl: result.url,
      key: result.key,
      expiresIn: 300,
    });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

module.exports = router;
```

---

## Lambda - Serverless Functions

### ขั้นตอนที่ 751: สร้าง Lambda Function

```javascript
// lambda/image-processor/index.js
const { S3Client, GetObjectCommand, PutObjectCommand } = require('@aws-sdk/client-s3');
const sharp = require('sharp');
const path = require('path');

const s3Client = new S3Client({ region: process.env.AWS_REGION });

// Handler สำหรับ S3 trigger
exports.handler = async (event) => {
  console.log('Event:', JSON.stringify(event, null, 2));

  try {
    for (const record of event.Records) {
      await processImage(record.s3);
    }

    return { statusCode: 200, body: 'Images processed successfully' };
  } catch (error) {
    console.error('Error:', error);
    throw error;
  }
};

async function processImage(s3Event) {
  const sourceBucket = s3Event.bucket.name;
  const sourceKey = decodeURIComponent(s3Event.object.key.replace(/\+/g, ' '));

  // ดึง original image จาก S3
  const getCommand = new GetObjectCommand({
    Bucket: sourceBucket,
    Key: sourceKey,
  });

  const { Body, ContentType } = await s3Client.send(getCommand);
  const imageBuffer = await streamToBuffer(Body);

  // Resize เป็นหลายขนาด
  const sizes = [
    { name: 'thumbnail', width: 150, height: 150 },
    { name: 'small', width: 320, height: 240 },
    { name: 'medium', width: 640, height: 480 },
    { name: 'large', width: 1280, height: 960 },
  ];

  for (const size of sizes) {
    const resized = await sharp(imageBuffer)
      .resize(size.width, size.height, {
        fit: 'cover',
        position: 'center',
      })
      .webp({ quality: 80 })
      .toBuffer();

    const destKey = sourceKey.replace(
      path.extname(sourceKey),
      `_${size.name}.webp`
    );

    await s3Client.send(new PutObjectCommand({
      Bucket: process.env.DEST_BUCKET || sourceBucket,
      Key: `processed/${destKey}`,
      Body: resized,
      ContentType: 'image/webp',
    }));

    console.log(`Created ${size.name} version: ${destKey}`);
  }
}

function streamToBuffer(stream) {
  return new Promise((resolve, reject) => {
    const chunks = [];
    stream.on('data', chunk => chunks.push(chunk));
    stream.on('error', reject);
    stream.on('end', () => resolve(Buffer.concat(chunks)));
  });
}
```

### ขั้นตอนที่ 752: API Gateway + Lambda

```javascript
// lambda/api/handler.js
const serverless = require('serverless-http');
const express = require('express');

const app = express();
app.use(express.json());

// Routes เหมือน Express ปกติ
app.get('/hello', (req, res) => {
  res.json({ message: 'Hello from Lambda!', time: new Date() });
});

app.post('/process', async (req, res) => {
  const { data } = req.body;
  const result = await processData(data);
  res.json({ result });
});

// Export handler
module.exports.handler = serverless(app);
```

```yaml
# serverless.yml (Serverless Framework)
service: myapp-api

provider:
  name: aws
  runtime: nodejs18.x
  region: ap-southeast-1
  memorySize: 256
  timeout: 30

  environment:
    MONGODB_URI: ${ssm:/myapp/mongodb-uri}
    JWT_SECRET: ${ssm:/myapp/jwt-secret}

  iam:
    role:
      statements:
        - Effect: Allow
          Action:
            - s3:GetObject
            - s3:PutObject
          Resource: !Sub 'arn:aws:s3:::${BucketName}/*'

functions:
  api:
    handler: handler.handler
    events:
      - http:
          path: /
          method: ANY
          cors: true
      - http:
          path: /{proxy+}
          method: ANY
          cors: true

  imageProcessor:
    handler: image-processor.handler
    events:
      - s3:
          bucket: myapp-images
          event: s3:ObjectCreated:*
          rules:
            - prefix: uploads/
    layers:
      - !Ref SharpLambdaLayer

  scheduler:
    handler: scheduler.handler
    events:
      - schedule: rate(1 hour)
```

---

## Elastic Beanstalk

### ขั้นตอนที่ 753: Deploy ด้วย Elastic Beanstalk

```bash
# ติดตั้ง EB CLI
pip install awsebcli

# Initialize application
eb init myapp --region ap-southeast-1 --platform node.js-18

# สร้าง environment
eb create myapp-production \
  --instance_type t3.small \
  --min-instances 2 \
  --max-instances 10 \
  --database \
  --envvars NODE_ENV=production

# Deploy
eb deploy

# ดู logs
eb logs

# SSH เข้า instance
eb ssh
```

```json
// .elasticbeanstalk/config.yml
branch-defaults:
  main:
    environment: myapp-production
    group_suffix: null

deploy:
  artifact: deploy.zip

global:
  application_name: myapp
  default_ec2_keyname: myapp-key
  default_platform: Node.js 18 running on 64bit Amazon Linux 2
  default_region: ap-southeast-1
  include_git_submodules: true
  instance_profile: null
  platform_name: null
  platform_version: null
  profile: null
  sc: git
  workspace_type: Application
```

```json
// .ebextensions/nodecommand.config
option_settings:
  aws:elasticbeanstalk:application:environment:
    NODE_ENV: production
    PORT: 8080
  aws:elasticbeanstalk:container:nodejs:
    NodeCommand: "npm start"
    NodeVersion: 18
  aws:elasticbeanstalk:environment:proxy:
    ProxyServer: nginx
```

---

## ECS - Container Service

### ขั้นตอนที่ 754: สร้าง ECS Cluster

```bash
# สร้าง ECS Cluster
aws ecs create-cluster \
  --cluster-name myapp-cluster \
  --capacity-providers FARGATE FARGATE_SPOT \
  --default-capacity-provider-strategy \
    capacityProvider=FARGATE,weight=1 \
    capacityProvider=FARGATE_SPOT,weight=4

# สร้าง Task Definition
aws ecs register-task-definition \
  --cli-input-json file://task-definition.json
```

```json
// task-definition.json
{
  "family": "myapp",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "256",
  "memory": "512",
  "executionRoleArn": "arn:aws:iam::123456789:role/ecsTaskExecutionRole",
  "taskRoleArn": "arn:aws:iam::123456789:role/myappTaskRole",
  "containerDefinitions": [
    {
      "name": "myapp",
      "image": "123456789.dkr.ecr.ap-southeast-1.amazonaws.com/myapp:latest",
      "portMappings": [
        {
          "containerPort": 3000,
          "protocol": "tcp"
        }
      ],
      "environment": [
        { "name": "NODE_ENV", "value": "production" },
        { "name": "PORT", "value": "3000" }
      ],
      "secrets": [
        {
          "name": "MONGODB_URI",
          "valueFrom": "arn:aws:secretsmanager:ap-southeast-1:123456789:secret:myapp/mongodb-uri"
        }
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/myapp",
          "awslogs-region": "ap-southeast-1",
          "awslogs-stream-prefix": "ecs"
        }
      },
      "healthCheck": {
        "command": ["CMD-SHELL", "curl -f http://localhost:3000/health || exit 1"],
        "interval": 30,
        "timeout": 5,
        "retries": 3,
        "startPeriod": 30
      }
    }
  ]
}
```

---

## CloudFront - CDN

### ขั้นตอนที่ 755: ตั้งค่า CloudFront

```javascript
// src/config/cloudfront.js
const { CloudFrontClient, CreateInvalidationCommand } = require('@aws-sdk/client-cloudfront');
const { v4: uuidv4 } = require('uuid');

const cloudfront = new CloudFrontClient({ region: 'us-east-1' });
const DISTRIBUTION_ID = process.env.CLOUDFRONT_DISTRIBUTION_ID;

// Invalidate cache
async function invalidateCache(paths = ['/*']) {
  const command = new CreateInvalidationCommand({
    DistributionId: DISTRIBUTION_ID,
    InvalidationBatch: {
      CallerReference: uuidv4(),
      Paths: {
        Quantity: paths.length,
        Items: paths,
      },
    },
  });

  const result = await cloudfront.send(command);
  console.log(`Cache invalidated: ${result.Invalidation.Id}`);
  return result.Invalidation;
}

// Generate signed URL (สำหรับ private content)
async function getSignedUrl(resourcePath, expiresIn = 3600) {
  const { getSignedUrl } = require('@aws-sdk/cloudfront-signer');

  const url = `https://${process.env.CLOUDFRONT_DOMAIN}${resourcePath}`;
  const expiresAt = Math.floor(Date.now() / 1000) + expiresIn;

  return getSignedUrl({
    url,
    keyPairId: process.env.CLOUDFRONT_KEY_PAIR_ID,
    dateLessThan: new Date(expiresAt * 1000).toISOString(),
    privateKey: process.env.CLOUDFRONT_PRIVATE_KEY,
  });
}

module.exports = { invalidateCache, getSignedUrl };
```

---

## IAM - Identity and Access Management

### ขั้นตอนที่ 756: IAM Roles สำหรับ Node.js

```json
// iam-policy.json - Least privilege principle
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:DeleteObject"
      ],
      "Resource": "arn:aws:s3:::myapp-bucket/*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:ListBucket"
      ],
      "Resource": "arn:aws:s3:::myapp-bucket"
    },
    {
      "Effect": "Allow",
      "Action": [
        "secretsmanager:GetSecretValue"
      ],
      "Resource": "arn:aws:secretsmanager:*:*:secret:myapp/*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "sqs:SendMessage",
        "sqs:ReceiveMessage",
        "sqs:DeleteMessage"
      ],
      "Resource": "arn:aws:sqs:ap-southeast-1:*:myapp-*"
    }
  ]
}
```

### ขั้นตอนที่ 757: Secrets Manager Integration

```javascript
// src/config/secrets.js
const { SecretsManagerClient, GetSecretValueCommand } = require('@aws-sdk/client-secrets-manager');

const client = new SecretsManagerClient({ region: process.env.AWS_REGION });

// Cache secrets
const secretsCache = new Map();

async function getSecret(secretName) {
  // ตรวจสอบ cache ก่อน
  if (secretsCache.has(secretName)) {
    const { value, expiry } = secretsCache.get(secretName);
    if (Date.now() < expiry) {
      return value;
    }
  }

  try {
    const command = new GetSecretValueCommand({ SecretId: secretName });
    const response = await client.send(command);

    let secret;
    if (response.SecretString) {
      try {
        secret = JSON.parse(response.SecretString);
      } catch {
        secret = response.SecretString;
      }
    } else {
      secret = Buffer.from(response.SecretBinary, 'base64').toString('utf8');
    }

    // Cache เป็นเวลา 5 นาที
    secretsCache.set(secretName, {
      value: secret,
      expiry: Date.now() + 5 * 60 * 1000,
    });

    return secret;
  } catch (error) {
    throw new Error(`Failed to get secret ${secretName}: ${error.message}`);
  }
}

// โหลด secrets เมื่อเริ่ม app
async function loadSecrets() {
  const secrets = await getSecret('myapp/production');

  process.env.MONGODB_URI = secrets.MONGODB_URI;
  process.env.JWT_SECRET = secrets.JWT_SECRET;
  process.env.REDIS_PASSWORD = secrets.REDIS_PASSWORD;

  console.log('✅ Secrets loaded from AWS Secrets Manager');
}

module.exports = { getSecret, loadSecrets };
```

---

## CloudWatch Monitoring

### ขั้นตอนที่ 758: CloudWatch Logging

```javascript
// src/config/cloudwatch.js
const winston = require('winston');
const WinstonCloudWatch = require('winston-cloudwatch');

const logger = winston.createLogger({
  level: 'info',
  format: winston.format.combine(
    winston.format.timestamp(),
    winston.format.json()
  ),
  transports: [
    new winston.transports.Console(),

    // ส่ง logs ไปยัง CloudWatch
    new WinstonCloudWatch({
      logGroupName: `/myapp/${process.env.NODE_ENV}`,
      logStreamName: `${process.env.HOSTNAME || 'app'}-${new Date().toISOString().slice(0, 10)}`,
      awsRegion: process.env.AWS_REGION,
      messageFormatter: (item) => JSON.stringify({
        level: item.level,
        message: item.message,
        ...item,
      }),
    }),
  ],
});

module.exports = logger;
```

### ขั้นตอนที่ 759: Custom CloudWatch Metrics

```javascript
// src/monitoring/cloudwatchMetrics.js
const { CloudWatchClient, PutMetricDataCommand } = require('@aws-sdk/client-cloudwatch');

const cloudwatch = new CloudWatchClient({ region: process.env.AWS_REGION });

async function putMetric(metricName, value, unit = 'Count', dimensions = []) {
  try {
    const command = new PutMetricDataCommand({
      Namespace: 'MyApp/API',
      MetricData: [
        {
          MetricName: metricName,
          Dimensions: [
            { Name: 'Environment', Value: process.env.NODE_ENV },
            ...dimensions,
          ],
          Value: value,
          Unit: unit,
          Timestamp: new Date(),
        },
      ],
    });

    await cloudwatch.send(command);
  } catch (error) {
    console.error('Failed to put metric:', error);
  }
}

// Middleware สำหรับ API metrics
function apiMetricsMiddleware(req, res, next) {
  const start = Date.now();

  res.on('finish', async () => {
    const duration = Date.now() - start;

    // ส่ง metrics
    await Promise.all([
      putMetric('APICallCount', 1, 'Count', [
        { Name: 'Endpoint', Value: req.path },
        { Name: 'Method', Value: req.method },
      ]),
      putMetric('APILatency', duration, 'Milliseconds', [
        { Name: 'Endpoint', Value: req.path },
      ]),
      res.statusCode >= 400 && putMetric('APIErrors', 1, 'Count', [
        { Name: 'StatusCode', Value: String(res.statusCode) },
      ]),
    ].filter(Boolean));
  });

  next();
}

module.exports = { putMetric, apiMetricsMiddleware };
```

---

## SQS - Simple Queue Service

### ขั้นตอนที่ 760: SQS Integration

```javascript
// src/services/sqsService.js
const {
  SQSClient,
  SendMessageCommand,
  ReceiveMessageCommand,
  DeleteMessageCommand,
  SendMessageBatchCommand,
} = require('@aws-sdk/client-sqs');

const sqs = new SQSClient({ region: process.env.AWS_REGION });
const QUEUE_URL = process.env.SQS_QUEUE_URL;

class SQSService {
  async sendMessage(messageBody, options = {}) {
    const command = new SendMessageCommand({
      QueueUrl: QUEUE_URL,
      MessageBody: JSON.stringify(messageBody),
      DelaySeconds: options.delay || 0,
      MessageGroupId: options.groupId,  // สำหรับ FIFO queues
      MessageDeduplicationId: options.deduplicationId,
    });

    const result = await sqs.send(command);
    return result.MessageId;
  }

  async sendBatch(messages) {
    const entries = messages.map((msg, index) => ({
      Id: String(index),
      MessageBody: JSON.stringify(msg.body),
      DelaySeconds: msg.delay || 0,
    }));

    const command = new SendMessageBatchCommand({
      QueueUrl: QUEUE_URL,
      Entries: entries,
    });

    return sqs.send(command);
  }

  async receiveMessages(maxMessages = 10) {
    const command = new ReceiveMessageCommand({
      QueueUrl: QUEUE_URL,
      MaxNumberOfMessages: maxMessages,
      WaitTimeSeconds: 20,  // Long polling
      MessageAttributeNames: ['All'],
      VisibilityTimeout: 30,
    });

    const result = await sqs.send(command);
    return result.Messages || [];
  }

  async deleteMessage(receiptHandle) {
    const command = new DeleteMessageCommand({
      QueueUrl: QUEUE_URL,
      ReceiptHandle: receiptHandle,
    });

    await sqs.send(command);
  }

  // Worker loop
  async startWorker(processFn) {
    console.log('🔄 SQS Worker started');

    while (true) {
      try {
        const messages = await this.receiveMessages(10);

        await Promise.all(messages.map(async (message) => {
          try {
            const body = JSON.parse(message.Body);
            await processFn(body);
            await this.deleteMessage(message.ReceiptHandle);
          } catch (error) {
            console.error('Failed to process message:', error);
            // Message จะกลับมาใน queue หลัง VisibilityTimeout
          }
        }));
      } catch (error) {
        console.error('Worker error:', error);
        await new Promise(resolve => setTimeout(resolve, 5000));
      }
    }
  }
}

module.exports = new SQSService();
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Deploy Node.js API บน EC2

```bash
# TODO: ทำตามขั้นตอน:
# 1. สร้าง EC2 instance
# 2. ติดตั้ง Node.js, PM2, Nginx
# 3. Deploy Node.js API
# 4. ตั้งค่า SSL ด้วย Let's Encrypt
# 5. ตั้งค่า Auto Scaling
```

### แบบฝึกหัดที่ 2: Serverless File Processor

```javascript
// TODO: สร้าง Lambda function ที่:
// - Trigger เมื่อมีไฟล์ upload ไปยัง S3
// - อ่านไฟล์ CSV
// - ประมวลผลข้อมูล
// - บันทึกผลลัพธ์ลง DynamoDB
// - ส่ง notification ผ่าน SNS
```

### แบบฝึกหัดที่ 3: Multi-Region Deployment

```yaml
# TODO: ออกแบบ multi-region architecture:
# - Primary: ap-southeast-1 (Singapore)
# - Secondary: ap-southeast-2 (Sydney)
# - Route 53 health checks
# - Database replication
# - S3 replication
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **EC2** - virtual servers, auto scaling, load balancing
2. **RDS** - managed databases, connection pooling
3. **S3** - file storage, presigned URLs, CloudFront CDN
4. **Lambda** - serverless functions, event triggers
5. **ECS** - container orchestration ด้วย Fargate
6. **Elastic Beanstalk** - managed app platform
7. **IAM** - roles, policies, least privilege
8. **Secrets Manager** - secrets storage และ rotation
9. **CloudWatch** - logging, metrics, monitoring
10. **SQS** - message queuing

ในบทถัดไปเราจะเรียนรู้ **Monitoring และ Performance** - PM2, Prometheus, Grafana, profiling
