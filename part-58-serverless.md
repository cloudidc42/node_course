# Part 58 | ขั้นตอนที่ 1021-1040 จาก 1000

## Serverless Architecture กับ Node.js

Serverless คือ execution model ที่ cloud provider จัดการ infrastructure ให้ทั้งหมด เราเขียนแค่ function และ cloud จะ run เมื่อมี event

---

## ขั้นตอนที่ 1021: Serverless Concepts

```
Traditional Server:
  Server (Always Running) → Handle Requests → Pay 24/7

Serverless:
  Event → Function Invoked → Execute → Return → Shutdown
  Pay only per execution (milliseconds)

Benefits:
  - Auto scaling (0 to millions)
  - Pay per use
  - No server management
  - High availability built-in

Limitations:
  - Cold start latency
  - Execution time limits (15 min for Lambda)
  - Stateless (ต้องใช้ external state)
  - Vendor lock-in
```

---

## ขั้นตอนที่ 1022: AWS Lambda Basics

```javascript
// handlers/hello.js

// Lambda handler function signature
exports.handler = async (event, context) => {
  console.log('Event:', JSON.stringify(event));
  console.log('Context:', JSON.stringify(context));
  
  return {
    statusCode: 200,
    headers: {
      'Content-Type': 'application/json',
      'Access-Control-Allow-Origin': '*'
    },
    body: JSON.stringify({
      message: 'Hello from Lambda!',
      input: event
    })
  };
};
```

```javascript
// handlers/api.js
// API Gateway + Lambda handler

exports.handler = async (event) => {
  const { httpMethod, path, pathParameters, queryStringParameters, body } = event;
  
  try {
    let result;
    
    // Route based on HTTP method and path
    if (httpMethod === 'GET' && path === '/users') {
      result = await getUsers(queryStringParameters);
    } else if (httpMethod === 'GET' && pathParameters?.id) {
      result = await getUser(pathParameters.id);
    } else if (httpMethod === 'POST' && path === '/users') {
      result = await createUser(JSON.parse(body || '{}'));
    } else {
      return {
        statusCode: 404,
        body: JSON.stringify({ error: 'Route not found' })
      };
    }
    
    return {
      statusCode: 200,
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(result)
    };
  } catch (error) {
    console.error('Handler error:', error);
    
    return {
      statusCode: error.statusCode || 500,
      body: JSON.stringify({ error: error.message })
    };
  }
};

async function getUsers(params) {
  // Business logic here
  return { users: [], total: 0 };
}

async function getUser(id) {
  // Business logic here
  return { id, name: 'User' };
}

async function createUser(data) {
  // Business logic here
  return { id: 'new-id', ...data };
}
```

---

## ขั้นตอนที่ 1023: Serverless Framework

```yaml
# serverless.yml

service: my-node-api
frameworkVersion: '3'

provider:
  name: aws
  runtime: nodejs18.x
  region: ap-southeast-1
  stage: ${opt:stage, 'dev'}
  
  environment:
    MONGODB_URI: ${env:MONGODB_URI}
    JWT_SECRET: ${env:JWT_SECRET}
    NODE_ENV: ${self:provider.stage}
  
  iam:
    role:
      statements:
        - Effect: Allow
          Action:
            - dynamodb:*
            - s3:GetObject
            - s3:PutObject
          Resource:
            - !GetAtt UsersTable.Arn
            - !Sub '${FilesBucket.Arn}/*'

functions:
  # API Functions
  getUsers:
    handler: handlers/users.getAll
    events:
      - httpApi:
          path: /users
          method: GET
    memorySize: 256
    timeout: 10
  
  getUser:
    handler: handlers/users.getOne
    events:
      - httpApi:
          path: /users/{id}
          method: GET
  
  createUser:
    handler: handlers/users.create
    events:
      - httpApi:
          path: /users
          method: POST
  
  # Scheduled Function
  dailyReport:
    handler: handlers/reports.daily
    events:
      - schedule: cron(0 8 * * ? *)  # 8am UTC every day
    timeout: 300
  
  # S3 Trigger
  processUpload:
    handler: handlers/uploads.process
    events:
      - s3:
          bucket: ${self:custom.filesBucket}
          event: s3:ObjectCreated:*
  
  # SQS Trigger
  processQueue:
    handler: handlers/queue.process
    events:
      - sqs:
          arn: !GetAtt OrderQueue.Arn
          batchSize: 10

custom:
  filesBucket: my-app-files-${self:provider.stage}

resources:
  Resources:
    UsersTable:
      Type: AWS::DynamoDB::Table
      Properties:
        TableName: Users-${self:provider.stage}
        BillingMode: PAY_PER_REQUEST
        AttributeDefinitions:
          - AttributeName: id
            AttributeType: S
        KeySchema:
          - AttributeName: id
            KeyType: HASH
    
    FilesBucket:
      Type: AWS::S3::Bucket
      Properties:
        BucketName: ${self:custom.filesBucket}
    
    OrderQueue:
      Type: AWS::SQS::Queue
      Properties:
        QueueName: OrderQueue-${self:provider.stage}
        VisibilityTimeout: 60
```

---

## ขั้นตอนที่ 1024: Lambda with MongoDB

```javascript
// lib/database.js
// Connection pooling สำหรับ Lambda

const mongoose = require('mongoose');

let cachedConnection = null;

async function connectDB() {
  // ใช้ connection ที่ cache ไว้ถ้ายัง alive
  if (cachedConnection && mongoose.connection.readyState === 1) {
    return cachedConnection;
  }
  
  const connection = await mongoose.connect(process.env.MONGODB_URI, {
    maxPoolSize: 5,    // Lambda: ใช้ pool เล็ก
    serverSelectionTimeoutMS: 5000,
    socketTimeoutMS: 45000
  });
  
  cachedConnection = connection;
  return connection;
}

module.exports = { connectDB };
```

```javascript
// handlers/users.js

const { connectDB } = require('../lib/database');
const User = require('../models/User');

// Lambda reuse connection between invocations
exports.getAll = async (event) => {
  await connectDB();
  
  const { page = 1, limit = 20 } = event.queryStringParameters || {};
  
  const users = await User.find({})
    .limit(parseInt(limit))
    .skip((parseInt(page) - 1) * parseInt(limit))
    .lean();
  
  return {
    statusCode: 200,
    body: JSON.stringify({ users, page: parseInt(page) })
  };
};

exports.create = async (event) => {
  await connectDB();
  
  const data = JSON.parse(event.body);
  const user = await User.create(data);
  
  return {
    statusCode: 201,
    body: JSON.stringify(user)
  };
};
```

---

## ขั้นตอนที่ 1025: Lambda with DynamoDB

```javascript
// handlers/dynamoUsers.js

const { DynamoDBClient } = require('@aws-sdk/client-dynamodb');
const { DynamoDBDocumentClient, GetCommand, PutCommand, QueryCommand, DeleteCommand } = require('@aws-sdk/lib-dynamodb');

const client = new DynamoDBClient({ region: process.env.AWS_REGION });
const docClient = DynamoDBDocumentClient.from(client);
const TABLE_NAME = process.env.TABLE_NAME || 'Users';

exports.getUser = async (event) => {
  const { id } = event.pathParameters;
  
  const response = await docClient.send(new GetCommand({
    TableName: TABLE_NAME,
    Key: { id }
  }));
  
  if (!response.Item) {
    return {
      statusCode: 404,
      body: JSON.stringify({ error: 'User not found' })
    };
  }
  
  return {
    statusCode: 200,
    body: JSON.stringify(response.Item)
  };
};

exports.createUser = async (event) => {
  const data = JSON.parse(event.body);
  const id = require('crypto').randomUUID();
  
  const user = {
    id,
    ...data,
    createdAt: new Date().toISOString()
  };
  
  await docClient.send(new PutCommand({
    TableName: TABLE_NAME,
    Item: user,
    ConditionExpression: 'attribute_not_exists(id)' // Prevent overwrite
  }));
  
  return {
    statusCode: 201,
    body: JSON.stringify(user)
  };
};

exports.deleteUser = async (event) => {
  const { id } = event.pathParameters;
  
  await docClient.send(new DeleteCommand({
    TableName: TABLE_NAME,
    Key: { id }
  }));
  
  return { statusCode: 204, body: '' };
};
```

---

## ขั้นตอนที่ 1026: Cold Start Optimization

```javascript
// middleware/warmup.js
// ป้องกัน cold start จาก CloudWatch scheduled event

const isWarmupRequest = (event) => {
  return event.source === 'serverless-plugin-warmup' ||
         event['warm-up'] === true;
};

// handler ที่รับมือ warmup
exports.handler = async (event) => {
  // ตรวจสอบ warmup request
  if (isWarmupRequest(event)) {
    console.log('Warmup request - returning early');
    return { statusCode: 200, body: 'warm' };
  }
  
  // Business logic ตามปกติ
  return { statusCode: 200, body: 'ok' };
};

// serverless.yml: schedule warmup
/*
functions:
  myFunction:
    handler: handler.handler
    events:
      - schedule:
          rate: rate(5 minutes)
          enabled: true
          input:
            warm-up: true
*/
```

```javascript
// lib/connectionPool.js
// Initialize connections ไว้นอก handler (warm container reuse)

const mongoose = require('mongoose');
const Redis = require('ioredis');

// ตัวแปร global ใน Lambda container
let _mongoConnection;
let _redisClient;

// Initialize ที่ module load time
async function getMongoConnection() {
  if (_mongoConnection && mongoose.connection.readyState === 1) {
    return _mongoConnection;
  }
  
  console.log('Creating new MongoDB connection');
  _mongoConnection = await mongoose.connect(process.env.MONGODB_URI);
  return _mongoConnection;
}

function getRedisClient() {
  if (_redisClient) return _redisClient;
  
  console.log('Creating new Redis client');
  _redisClient = new Redis(process.env.REDIS_URL, {
    lazyConnect: false,
    enableOfflineQueue: false,
    connectTimeout: 2000
  });
  
  return _redisClient;
}

module.exports = { getMongoConnection, getRedisClient };
```

---

## ขั้นตอนที่ 1027: Lambda Middleware Pattern

```javascript
// middleware/lambda-middleware.js
// Middleware สำหรับ Lambda (Middy library)

const middy = require('@middy/core');
const httpJsonBodyParser = require('@middy/http-json-body-parser');
const httpErrorHandler = require('@middy/http-error-handler');
const httpCors = require('@middy/http-cors');
const validator = require('@middy/validator');
const createError = require('http-errors');

// Custom middleware: Database connection
const dbMiddleware = (connectFn) => ({
  before: async (request) => {
    await connectFn();
  }
});

// Custom middleware: Auth
const authMiddleware = (tokenService) => ({
  before: async (request) => {
    const { event } = request;
    const authHeader = event.headers?.Authorization || event.headers?.authorization;
    
    if (!authHeader?.startsWith('Bearer ')) {
      throw createError(401, 'Authentication required');
    }
    
    const token = authHeader.split(' ')[1];
    const payload = await tokenService.verify(token);
    
    // เพิ่ม user ใน context
    event.user = payload;
  }
});

// Handler ที่ใช้ Middy
const getUsers = middy(async (event) => {
  const { page = 1, limit = 20 } = event.queryStringParameters || {};
  const User = require('../models/User');
  
  const users = await User.find({}).limit(limit).skip((page - 1) * limit);
  
  return {
    statusCode: 200,
    body: JSON.stringify({ users })
  };
});

getUsers
  .use(httpJsonBodyParser())
  .use(httpCors())
  .use(dbMiddleware(require('../lib/database').connectDB))
  .use(httpErrorHandler());

module.exports = { getUsers };
```

---

## ขั้นตอนที่ 1028: SQS Event Processing

```javascript
// handlers/orderProcessor.js
// SQS trigger handler

exports.handler = async (event) => {
  const results = {
    batchItemFailures: []
  };
  
  for (const record of event.Records) {
    const messageId = record.messageId;
    
    try {
      const order = JSON.parse(record.body);
      await processOrder(order);
      console.log(`Processed order: ${order.id}`);
    } catch (error) {
      console.error(`Failed to process message ${messageId}:`, error);
      
      // ส่ง failed items กลับเพื่อให้ SQS retry
      results.batchItemFailures.push({
        itemIdentifier: messageId
      });
    }
  }
  
  return results;
};

async function processOrder(order) {
  await require('../lib/database').connectDB();
  
  const Order = require('../models/Order');
  const existing = await Order.findById(order.id);
  
  if (!existing) {
    throw new Error(`Order not found: ${order.id}`);
  }
  
  // Process...
  existing.status = 'processing';
  await existing.save();
  
  // Notify customer
  await sendConfirmation(existing);
}

async function sendConfirmation(order) {
  // Email notification
}
```

---

## ขั้นตอนที่ 1029: S3 Event Handler

```javascript
// handlers/imageProcessor.js
// S3 trigger: process uploaded images

const { S3Client, GetObjectCommand, PutObjectCommand } = require('@aws-sdk/client-s3');
const sharp = require('sharp');

const s3 = new S3Client({ region: process.env.AWS_REGION });

exports.handler = async (event) => {
  for (const record of event.Records) {
    const bucket = record.s3.bucket.name;
    const key = decodeURIComponent(record.s3.object.key.replace(/\+/g, ' '));
    
    console.log(`Processing: ${bucket}/${key}`);
    
    try {
      // ดาวน์โหลด original image
      const { Body } = await s3.send(new GetObjectCommand({
        Bucket: bucket,
        Key: key
      }));
      
      const buffer = await streamToBuffer(Body);
      
      // สร้าง thumbnails
      const sizes = [
        { width: 100, height: 100, suffix: 'thumb' },
        { width: 400, height: 400, suffix: 'medium' },
        { width: 800, height: 800, suffix: 'large' }
      ];
      
      await Promise.all(sizes.map(async ({ width, height, suffix }) => {
        const resized = await sharp(buffer)
          .resize(width, height, { fit: 'inside', withoutEnlargement: true })
          .webp({ quality: 80 })
          .toBuffer();
        
        const outputKey = key.replace(/\.[^.]+$/, `-${suffix}.webp`);
        
        await s3.send(new PutObjectCommand({
          Bucket: bucket,
          Key: `thumbnails/${outputKey}`,
          Body: resized,
          ContentType: 'image/webp',
          CacheControl: 'max-age=31536000'
        }));
      }));
      
      console.log(`Thumbnails created for ${key}`);
    } catch (error) {
      console.error(`Error processing ${key}:`, error);
      throw error;
    }
  }
};

async function streamToBuffer(stream) {
  const chunks = [];
  for await (const chunk of stream) {
    chunks.push(chunk);
  }
  return Buffer.concat(chunks);
}
```

---

## ขั้นตอนที่ 1030: Local Development with SAM

```yaml
# template.yaml (AWS SAM)

AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31

Globals:
  Function:
    Runtime: nodejs18.x
    Environment:
      Variables:
        MONGODB_URI: !Ref MongoDBUri
        NODE_ENV: development

Parameters:
  MongoDBUri:
    Type: String
    Default: mongodb://localhost:27017/myapp

Resources:
  UsersFunction:
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: src/
      Handler: handlers/users.handler
      Events:
        GetUsers:
          Type: Api
          Properties:
            Path: /users
            Method: get
        CreateUser:
          Type: Api
          Properties:
            Path: /users
            Method: post
```

```bash
# Local testing commands
sam local start-api --port 3000
sam local invoke UsersFunction --event events/get-users.json
sam local generate-event apigateway http-api-proxy > events/test-event.json
```

---

## ขั้นตอนที่ 1031: Serverless Offline Plugin

```javascript
// serverless.yml (Development)
/*
plugins:
  - serverless-offline
  - serverless-dotenv-plugin

custom:
  serverless-offline:
    httpPort: 3000
    lambdaPort: 3002
*/

// package.json scripts
// "dev": "serverless offline start"
// "deploy": "serverless deploy"
// "deploy:prod": "serverless deploy --stage prod"

// ทดสอบ local
// curl http://localhost:3000/users
// curl -X POST http://localhost:3000/users -d '{"name":"Test","email":"test@test.com"}'
```

---

## ขั้นตอนที่ 1032: Lambda Layers

```javascript
// layers/nodejs/node_modules โครงสร้าง
// หรือ layers/nodejs/lib/node_modules

// serverless.yml
/*
layers:
  commonDependencies:
    path: layers/common
    name: ${self:service}-common
    description: "Common dependencies"
    compatibleRuntimes:
      - nodejs18.x
    retain: false

functions:
  myFunction:
    handler: handler.main
    layers:
      - !Ref CommonDependenciesLambdaLayer
*/

// layer/common/nodejs/lib/database.js
// ไฟล์ใน layer ถูก mount ที่ /opt/nodejs/lib
const { connectDB } = require('/opt/nodejs/lib/database');
```

---

## ขั้นตอนที่ 1033: Environment Variables & Secrets

```javascript
// lib/secrets.js
// ดึง secrets จาก AWS Secrets Manager

const { SecretsManagerClient, GetSecretValueCommand } = require('@aws-sdk/client-secrets-manager');

const client = new SecretsManagerClient({ region: process.env.AWS_REGION });
let _cachedSecrets = {};

async function getSecret(secretName) {
  // Cache ใน container
  if (_cachedSecrets[secretName]) {
    return _cachedSecrets[secretName];
  }
  
  const response = await client.send(new GetSecretValueCommand({
    SecretId: secretName
  }));
  
  const secret = JSON.parse(response.SecretString);
  _cachedSecrets[secretName] = secret;
  
  return secret;
}

// ใช้งาน
async function getDBCredentials() {
  return getSecret(process.env.DB_SECRET_NAME);
}

module.exports = { getSecret, getDBCredentials };
```

---

## ขั้นตอนที่ 1034: API Gateway Authorizer

```javascript
// handlers/authorizer.js
// Lambda authorizer สำหรับ API Gateway

const jwt = require('jsonwebtoken');

exports.handler = async (event) => {
  const token = extractToken(event);
  
  try {
    const payload = jwt.verify(token, process.env.JWT_SECRET);
    
    return {
      principalId: payload.userId,
      policyDocument: {
        Version: '2012-10-17',
        Statement: [{
          Action: 'execute-api:Invoke',
          Effect: 'Allow',
          Resource: event.methodArn
        }]
      },
      context: {
        userId: payload.userId,
        role: payload.role
      }
    };
  } catch (error) {
    throw new Error('Unauthorized');
  }
};

function extractToken(event) {
  if (event.authorizationToken) {
    return event.authorizationToken.replace('Bearer ', '');
  }
  
  if (event.headers?.Authorization) {
    return event.headers.Authorization.replace('Bearer ', '');
  }
  
  throw new Error('No token provided');
}
```

---

## ขั้นตอนที่ 1035: Lambda Monitoring

```javascript
// lib/metrics.js
// Custom metrics ด้วย CloudWatch

const { CloudWatchClient, PutMetricDataCommand } = require('@aws-sdk/client-cloudwatch');
const client = new CloudWatchClient({ region: process.env.AWS_REGION });

class Metrics {
  constructor(namespace) {
    this.namespace = namespace;
  }

  async record(metricName, value, unit = 'Count') {
    // ส่ง metric แบบ async (fire and forget)
    client.send(new PutMetricDataCommand({
      Namespace: this.namespace,
      MetricData: [{
        MetricName: metricName,
        Value: value,
        Unit: unit,
        Timestamp: new Date()
      }]
    })).catch(err => console.error('Metrics error:', err));
  }
}

// Lambda middleware สำหรับ metrics
const metricsMiddleware = (metrics) => ({
  before: async (request) => {
    request.context._startTime = Date.now();
  },
  after: async (request) => {
    const duration = Date.now() - request.context._startTime;
    await metrics.record('ExecutionTime', duration, 'Milliseconds');
    await metrics.record('Invocations', 1);
  },
  onError: async (request) => {
    await metrics.record('Errors', 1);
  }
});

// Structured logging สำหรับ CloudWatch Insights
function createLogger(functionName) {
  return {
    info: (message, metadata = {}) => {
      console.log(JSON.stringify({
        level: 'INFO',
        function: functionName,
        message,
        ...metadata,
        timestamp: new Date().toISOString()
      }));
    },
    error: (message, error, metadata = {}) => {
      console.error(JSON.stringify({
        level: 'ERROR',
        function: functionName,
        message,
        errorName: error?.name,
        errorMessage: error?.message,
        ...metadata,
        timestamp: new Date().toISOString()
      }));
    }
  };
}

module.exports = { Metrics, metricsMiddleware, createLogger };
```

---

## ขั้นตอนที่ 1036: DynamoDB Streams

```javascript
// handlers/streamProcessor.js
// Process DynamoDB Streams events

exports.handler = async (event) => {
  for (const record of event.Records) {
    const { eventName, dynamodb } = record;
    
    const newImage = dynamodb.NewImage
      ? unmarshall(dynamodb.NewImage)
      : null;
    
    const oldImage = dynamodb.OldImage
      ? unmarshall(dynamodb.OldImage)
      : null;
    
    console.log(`Event: ${eventName}`);
    
    switch (eventName) {
      case 'INSERT':
        await handleNewUser(newImage);
        break;
      case 'MODIFY':
        await handleUpdatedUser(oldImage, newImage);
        break;
      case 'REMOVE':
        await handleDeletedUser(oldImage);
        break;
    }
  }
};

function unmarshall(item) {
  const { unmarshall: unmarshal } = require('@aws-sdk/util-dynamodb');
  return unmarshal(item);
}

async function handleNewUser(user) {
  console.log('New user created:', user.id);
  // ส่ง welcome email
  // Update search index
}

async function handleUpdatedUser(oldUser, newUser) {
  // ตรวจสอบว่า email เปลี่ยนหรือไม่
  if (oldUser.email !== newUser.email) {
    console.log('Email changed:', oldUser.email, '->', newUser.email);
    // Update related data
  }
}

async function handleDeletedUser(user) {
  console.log('User deleted:', user.id);
  // Cleanup related resources
}
```

---

## ขั้นตอนที่ 1037: Serverless with Express (Serverless-http)

```javascript
// app.js (ใช้ serverless-http ให้ Express ทำงานบน Lambda)

const express = require('express');
const serverless = require('serverless-http');
const mongoose = require('mongoose');

const app = express();
app.use(express.json());

// Routes
app.get('/users', async (req, res) => {
  // ...
  res.json({ users: [] });
});

app.post('/users', async (req, res) => {
  // ...
  res.status(201).json({ user: {} });
});

// Connect to DB before handling requests
const connectPromise = mongoose.connect(process.env.MONGODB_URI);

// Handler with connection warmup
module.exports.handler = async (event, context) => {
  // รอ DB connection
  await connectPromise;
  
  // Pass to serverless-http
  const handler = serverless(app);
  return handler(event, context);
};
```

```yaml
# serverless.yml
# functions:
#   api:
#     handler: app.handler
#     events:
#       - httpApi: '*'
```

---

## ขั้นตอนที่ 1038: Step Functions

```javascript
// handlers/orderWorkflow.js
// Lambda functions สำหรับ Step Functions workflow

exports.validateOrder = async (event) => {
  const { orderId, items } = event;
  
  // Validate order
  if (!items || items.length === 0) {
    throw new Error('Order has no items');
  }
  
  return { ...event, validated: true, validatedAt: new Date().toISOString() };
};

exports.checkInventory = async (event) => {
  const { items } = event;
  
  // Check inventory for each item
  const unavailableItems = [];
  
  for (const item of items) {
    const inStock = await checkStock(item.productId, item.quantity);
    if (!inStock) unavailableItems.push(item.productId);
  }
  
  if (unavailableItems.length > 0) {
    const error = new Error('Items out of stock');
    error.unavailableItems = unavailableItems;
    throw error;
  }
  
  return { ...event, inventoryChecked: true };
};

exports.processPayment = async (event) => {
  const { orderId, total, paymentMethod } = event;
  
  // Process payment
  const result = await chargePayment(paymentMethod, total);
  
  return {
    ...event,
    payment: {
      transactionId: result.transactionId,
      status: 'captured'
    }
  };
};

exports.fulfillOrder = async (event) => {
  // Create fulfillment
  return { ...event, status: 'fulfilled' };
};

async function checkStock(productId, quantity) {
  // DB check
  return true;
}

async function chargePayment(method, amount) {
  return { transactionId: 'txn-123' };
}
```

```json
// step-functions/orderWorkflow.json
{
  "Comment": "Order Processing Workflow",
  "StartAt": "ValidateOrder",
  "States": {
    "ValidateOrder": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:region:account:function:validateOrder",
      "Next": "CheckInventory",
      "Catch": [{
        "ErrorEquals": ["States.ALL"],
        "Next": "OrderFailed"
      }]
    },
    "CheckInventory": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:region:account:function:checkInventory",
      "Next": "ProcessPayment",
      "Retry": [{
        "ErrorEquals": ["Lambda.ServiceException"],
        "IntervalSeconds": 2,
        "MaxAttempts": 3,
        "BackoffRate": 2
      }]
    },
    "ProcessPayment": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:region:account:function:processPayment",
      "Next": "FulfillOrder"
    },
    "FulfillOrder": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:region:account:function:fulfillOrder",
      "End": true
    },
    "OrderFailed": {
      "Type": "Fail",
      "Error": "OrderFailed",
      "Cause": "Order processing failed"
    }
  }
}
```

---

## ขั้นตอนที่ 1039: Testing Lambda Functions

```javascript
// tests/handlers/users.test.js

// Mock AWS SDK
jest.mock('@aws-sdk/client-dynamodb', () => ({
  DynamoDBClient: jest.fn().mockImplementation(() => ({
    send: jest.fn()
  }))
}));

jest.mock('@aws-sdk/lib-dynamodb', () => ({
  DynamoDBDocumentClient: {
    from: jest.fn().mockReturnValue({
      send: jest.fn()
    })
  },
  GetCommand: jest.fn(),
  PutCommand: jest.fn(),
  QueryCommand: jest.fn()
}));

const { handler } = require('../../handlers/users');

describe('Users Lambda Handler', () => {
  test('GET /users returns list', async () => {
    const event = {
      httpMethod: 'GET',
      path: '/users',
      queryStringParameters: { page: '1', limit: '10' }
    };
    
    const response = await handler(event);
    
    expect(response.statusCode).toBe(200);
    
    const body = JSON.parse(response.body);
    expect(body).toHaveProperty('users');
  });

  test('POST /users creates user', async () => {
    const event = {
      httpMethod: 'POST',
      path: '/users',
      body: JSON.stringify({
        name: 'Test User',
        email: 'test@example.com'
      })
    };
    
    const response = await handler(event);
    expect(response.statusCode).toBe(201);
  });

  test('handles validation errors', async () => {
    const event = {
      httpMethod: 'POST',
      path: '/users',
      body: JSON.stringify({ name: '' })
    };
    
    const response = await handler(event);
    expect(response.statusCode).toBe(400);
  });
});
```

---

## ขั้นตอนที่ 1040: Production Best Practices

```javascript
// best-practices/serverless.js

/*
  Serverless Best Practices:
  
  1. Connection Pooling
     - Initialize connections นอก handler
     - Reuse connections ระหว่าง warm invocations
     - ใช้ small pool sizes (1-5 สำหรับ Lambda)
  
  2. Cold Start Reduction
     - Keep bundle size เล็ก (< 5MB)
     - ใช้ Lambda Layers สำหรับ dependencies
     - Provisioned Concurrency สำหรับ critical paths
     - ใช้ Scheduled warmup
  
  3. Error Handling
     - ใช้ Dead Letter Queue (DLQ) สำหรับ async functions
     - Retry logic ใน SQS/EventBridge
     - Structured error logging
  
  4. Security
     - Least privilege IAM roles
     - Environment variables ใน SSM Parameter Store
     - Secrets ใน Secrets Manager
     - VPC สำหรับ private resources
  
  5. Observability
     - X-Ray tracing
     - CloudWatch Metrics & Alarms
     - Structured logging สำหรับ CloudWatch Insights
*/

// ตัวอย่าง: Keep bundle เล็ก
// package.json
const bundleConfig = {
  "scripts": {
    "package": "serverless package",
    "deploy": "serverless deploy"
  },
  "devDependencies": {
    "serverless": "^3.0.0",
    "serverless-bundle": "^5.0.0" // minify + tree-shaking
  }
};

// serverless.yml สำหรับ smaller bundle
// plugins:
//   - serverless-bundle
// custom:
//   bundle:
//     linting: false
//     externals:
//       - aws-sdk   # Available in Lambda runtime, ไม่ต้อง bundle
```

---

## แบบฝึกหัดท้ายบท

### แบบฝึกหัดที่ 1: Serverless CRUD API
สร้าง CRUD API สำหรับ Products ด้วย Lambda + DynamoDB

### แบบฝึกหัดที่ 2: Image Processor
สร้าง S3 trigger ที่ resize images เป็น 3 sizes และบันทึกใน S3

### แบบฝึกหัดที่ 3: Scheduled Report
สร้าง scheduled function ที่ generate daily report และส่ง email

### แบบฝึกหัดที่ 4: Order Workflow
Implement order processing workflow ด้วย Step Functions

### แบบฝึกหัดที่ 5: Multi-Stage Deployment
ตั้งค่า dev, staging, production environments ด้วย Serverless Framework

---

## สรุป

Serverless ช่วยลด operational overhead แต่ต้องเข้าใจ trade-offs เช่น cold starts, stateless design, และ vendor lock-in ใช้ connection pooling, Lambda Layers, และ proper monitoring เพื่อให้ serverless applications มีประสิทธิภาพและ reliable ในการ production
