# Part 71: Serverless
## ขั้นตอนที่ 701-710 จาก 1000

---

## Serverless คืออะไร?

Serverless คือ execution model ที่ cloud provider จัดการ server infrastructure ให้ทั้งหมด นักพัฒนาต้องเขียนแค่ function logic โดยไม่ต้องสนใจเรื่อง server management

---

## 1. Serverless Concepts

### ข้อดีและข้อเสีย

```
ข้อดี:
- ไม่ต้อง manage servers
- Auto scaling (รวมถึง scale to zero)
- Pay per use (ประหยัดถ้า traffic ไม่สม่ำเสมอ)
- Deployment ง่าย

ข้อเสีย:
- Cold starts (latency สูงครั้งแรก)
- Execution time limit
- Vendor lock-in
- ยาก debug และ test locally
- Stateless (ต้องใช้ external storage)
```

### Use Cases ที่เหมาะ

```
เหมาะกับ:
- API endpoints ที่ traffic ไม่สม่ำเสมอ
- Scheduled tasks (cron jobs)
- Event-driven processing (webhooks, file uploads)
- Background jobs

ไม่เหมาะกับ:
- Long-running processes
- Real-time applications (WebSocket)
- High-performance computing
- Applications ที่ต้องการ consistent latency
```

---

## 2. AWS Lambda

### Basic Lambda Function

```javascript
// handler.js
exports.handler = async (event, context) => {
  console.log('Event:', JSON.stringify(event, null, 2));
  
  try {
    // ประมวลผล event
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
```

### Express.js บน Lambda (ด้วย serverless-http)

```bash
npm install serverless-http express
npm install -D serverless
```

```javascript
// app.js (Express app เดิม)
const express = require('express');
const app = express();

app.use(express.json());

app.get('/users', async (req, res) => {
  const users = await getUsersFromDB();
  res.json(users);
});

app.post('/users', async (req, res) => {
  const user = await createUser(req.body);
  res.status(201).json(user);
});

module.exports = app;

// lambda.js (Lambda entry point)
const serverless = require('serverless-http');
const app = require('./app');

module.exports.handler = serverless(app, {
  request: (request, event, context) => {
    // เพิ่ม context ให้ request
    request.context = context;
    request.lambdaEvent = event;
  }
});
```

### Serverless Framework Configuration

```yaml
# serverless.yml
service: my-nodejs-api

provider:
  name: aws
  runtime: nodejs18.x
  region: ap-southeast-1
  stage: ${opt:stage, 'dev'}
  
  environment:
    NODE_ENV: ${self:provider.stage}
    DATABASE_URL: ${ssm:/myapp/${self:provider.stage}/database-url}
    JWT_SECRET: ${ssm:/myapp/${self:provider.stage}/jwt-secret}
  
  iam:
    role:
      statements:
        - Effect: Allow
          Action:
            - dynamodb:*
            - s3:GetObject
            - s3:PutObject
          Resource: "*"

functions:
  api:
    handler: lambda.handler
    events:
      - httpApi: '*'
    memorySize: 512
    timeout: 29
    reservedConcurrency: 100
  
  scheduled:
    handler: jobs/cleanup.handler
    events:
      - schedule: cron(0 2 * * ? *)  # ทุกวัน 2am UTC
    timeout: 300

  processQueue:
    handler: jobs/queue-processor.handler
    events:
      - sqs:
          arn: arn:aws:sqs:region:account:queue-name
          batchSize: 10

plugins:
  - serverless-offline
  - serverless-prune-plugin

custom:
  prune:
    automatic: true
    number: 5
```

### Lambda Optimization

```javascript
// optimized-handler.js

// ย้าย connections ออกมานอก handler (reuse ระหว่าง invocations)
const mongoose = require('mongoose');

let cachedDb = null;

async function connectDB() {
  if (cachedDb) {
    return cachedDb;
  }
  
  const connection = await mongoose.connect(process.env.DATABASE_URL, {
    serverSelectionTimeoutMS: 5000,
    bufferCommands: false,
    maxPoolSize: 1  // Lambda มี pool ขนาดเล็ก
  });
  
  cachedDb = connection;
  return cachedDb;
}

// Warm-up plugin สำหรับลด cold starts
exports.warmUp = async (event) => {
  if (event.source === 'serverless-plugin-warmup') {
    console.log('WarmUp - Lambda is warm!');
    await new Promise(resolve => setTimeout(resolve, 25));
    return 'Lambda is warm!';
  }
};

exports.handler = async (event, context) => {
  // ไม่รอให้ event loop เป็น empty ก่อน return
  context.callbackWaitsForEmptyEventLoop = false;
  
  await connectDB();
  
  // ... rest of handler
};
```

---

## 3. Vercel Functions

### API Routes

```javascript
// api/users.js (Vercel API Route)
import mongoose from 'mongoose';
import User from '../../models/User';

let cached = global.mongoose;
if (!cached) {
  cached = global.mongoose = { conn: null, promise: null };
}

async function dbConnect() {
  if (cached.conn) return cached.conn;
  
  cached.promise = mongoose.connect(process.env.MONGODB_URI);
  cached.conn = await cached.promise;
  return cached.conn;
}

export default async function handler(req, res) {
  await dbConnect();
  
  const { method } = req;
  
  switch (method) {
    case 'GET':
      const users = await User.find({}).lean();
      res.json(users);
      break;
    
    case 'POST':
      const user = await User.create(req.body);
      res.status(201).json(user);
      break;
    
    default:
      res.setHeader('Allow', ['GET', 'POST']);
      res.status(405).end(`Method ${method} Not Allowed`);
  }
}
```

### Vercel Configuration

```json
{
  "version": 2,
  "builds": [
    {
      "src": "api/**/*.js",
      "use": "@vercel/node"
    }
  ],
  "routes": [
    {
      "src": "/api/(.*)",
      "dest": "/api/$1"
    }
  ],
  "env": {
    "NODE_ENV": "production"
  },
  "functions": {
    "api/**/*.js": {
      "maxDuration": 30,
      "memory": 1024
    }
  }
}
```

---

## 4. Cold Start Optimization

### ลด Cold Start Time

```javascript
// 1. ลดขนาด deployment package
// .serverlessignore
test/
docs/
*.test.js
*.spec.js
node_modules/.cache
.git

// 2. ใช้ tree shaking
// webpack.config.js
module.exports = {
  mode: 'production',
  target: 'node',
  entry: './src/lambda.js',
  output: {
    libraryTarget: 'commonjs2',
    path: path.resolve(__dirname, '.webpack'),
    filename: 'handler.js'
  },
  optimization: {
    minimize: true
  }
};

// 3. Lazy loading
// แทน require ที่ top level
exports.handler = async (event) => {
  // Load หรือ เมื่อต้องการ
  if (event.path === '/heavy-operation') {
    const heavyLib = await import('./heavy-library');
    return heavyLib.process(event);
  }
  // ... other handlers
};
```

### Provisioned Concurrency (AWS)

```yaml
# serverless.yml
functions:
  api:
    handler: lambda.handler
    provisionedConcurrency: 5  # Keep 5 instances warm
```

---

## 5. Serverless Database

### DynamoDB

```javascript
// services/dynamodb.service.js
const { DynamoDBClient } = require('@aws-sdk/client-dynamodb');
const {
  DynamoDBDocumentClient,
  GetCommand,
  PutCommand,
  QueryCommand,
  DeleteCommand
} = require('@aws-sdk/lib-dynamodb');

const client = new DynamoDBClient({ region: process.env.AWS_REGION });
const docClient = DynamoDBDocumentClient.from(client);

const TABLE_NAME = process.env.DYNAMODB_TABLE;

async function getUser(userId) {
  const response = await docClient.send(new GetCommand({
    TableName: TABLE_NAME,
    Key: {
      PK: `USER#${userId}`,
      SK: `USER#${userId}`
    }
  }));
  
  return response.Item;
}

async function createUser(user) {
  await docClient.send(new PutCommand({
    TableName: TABLE_NAME,
    Item: {
      PK: `USER#${user.id}`,
      SK: `USER#${user.id}`,
      Type: 'USER',
      ...user,
      createdAt: new Date().toISOString()
    },
    ConditionExpression: 'attribute_not_exists(PK)'
  }));
  
  return user;
}

async function getUserOrders(userId) {
  const response = await docClient.send(new QueryCommand({
    TableName: TABLE_NAME,
    KeyConditionExpression: 'PK = :pk AND begins_with(SK, :sk)',
    ExpressionAttributeValues: {
      ':pk': `USER#${userId}`,
      ':sk': 'ORDER#'
    }
  }));
  
  return response.Items;
}

module.exports = { getUser, createUser, getUserOrders };
```

---

## แบบฝึกหัด

### ระดับ 1: พื้นฐาน
สร้าง API บน Lambda:
- Express app + serverless-http
- Deploy ด้วย Serverless Framework
- Test locally ด้วย serverless-offline

### ระดับ 2: กลาง
เพิ่ม:
- Database connection pooling
- Environment variables จาก SSM
- Scheduled functions

### ระดับ 3: ขั้นสูง
Optimize สำหรับ production:
- Webpack bundling
- Provisioned Concurrency
- DynamoDB

---

## สรุป

Serverless เหมาะกับ workloads ที่ไม่ต่อเนื่อง เช่น APIs ที่มี traffic ไม่สม่ำเสมอ หรือ background jobs การจัดการ cold starts เป็นสิ่งสำคัญ ควรใช้ connection pooling และ lazy loading เพื่อ performance ที่ดี

> ขั้นตอนต่อไป: Part 72 - Real-World E-commerce API
