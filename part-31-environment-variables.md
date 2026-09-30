# Part 31: Environment Variables และการจัดการ Configuration

> ขั้นตอนที่ 31-31 จาก 1000

---

## สารบัญ

1. [Environment Variables คืออะไร](#environment-variables-คืออะไร)
2. [dotenv และ Best Practices](#dotenv-และ-best-practices)
3. [Environment-Specific Configs](#environment-specific-configs)
4. [Secrets Management](#secrets-management)
5. [Config Packages: node-config และ convict](#config-packages)
6. [Validation ด้วย Joi และ Zod](#validation-ดวย-joi-และ-zod)
7. [Workshop: ระบบ Config จริง](#workshop)
8. [แบบฝึกหัด](#แบบฝึกหัด)

---

## Environment Variables คืออะไร

Environment Variables (ตัวแปรสภาพแวดล้อม) คือค่าที่ถูกกำหนดไว้นอกโค้ดโปรแกรม ซึ่งโปรแกรมสามารถอ่านได้ขณะทำงาน แนวคิดนี้มาจากหลักการ **12-Factor App** ที่ระบุว่า configuration ควรแยกออกจาก code

### ทำไมต้องใช้ Environment Variables?

- **ความปลอดภัย**: ไม่ต้อง hardcode รหัสผ่านหรือ API keys ในโค้ด
- **ความยืดหยุ่น**: ใช้ค่าต่างกันในแต่ละสภาพแวดล้อม (dev/staging/production)
- **ง่ายต่อการเปลี่ยน**: เปลี่ยน config โดยไม่ต้อง deploy ใหม่
- **ทำงานร่วมกับ CI/CD**: ระบบ deploy สามารถฉีดค่าได้โดยตรง

### การอ่าน Environment Variables ใน Node.js

```javascript
// Node.js มี process.env object ให้ใช้งานได้ทันที
console.log(process.env.NODE_ENV);      // 'development' หรือ 'production'
console.log(process.env.PORT);          // '3000'
console.log(process.env.DATABASE_URL);  // connection string

// กำหนดค่า default ถ้าไม่มีการตั้งค่า
const port = process.env.PORT || 3000;
const env = process.env.NODE_ENV || 'development';
```

### ตั้งค่า Environment Variables บน Linux/Mac

```bash
# ตั้งค่าชั่วคราวใน shell session ปัจจุบัน
export PORT=3000
export NODE_ENV=development

# ตั้งค่าและรันโปรแกรมในคำสั่งเดียว
PORT=3000 NODE_ENV=production node app.js

# ดู environment variables ทั้งหมด
env
printenv

# ดูค่าตัวแปรเดียว
echo $PORT
```

### ตั้งค่าบน Windows

```cmd
:: Command Prompt
set PORT=3000
set NODE_ENV=development

:: PowerShell
$env:PORT = "3000"
$env:NODE_ENV = "development"
```

---

## dotenv และ Best Practices

### การติดตั้ง dotenv

```bash
npm install dotenv
```

### การสร้างไฟล์ .env

```bash
# .env - ไฟล์นี้ต้องไม่ถูก commit ขึ้น git
NODE_ENV=development
PORT=3000

# Database
DATABASE_URL=postgresql://user:password@localhost:5432/mydb
DATABASE_HOST=localhost
DATABASE_PORT=5432
DATABASE_NAME=mydb
DATABASE_USER=user
DATABASE_PASSWORD=secret123

# JWT
JWT_SECRET=your-super-secret-jwt-key-here
JWT_EXPIRES_IN=7d

# Email
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your@email.com
SMTP_PASS=your-email-password

# External APIs
STRIPE_SECRET_KEY=sk_test_xxxxxxxxxxxxx
STRIPE_WEBHOOK_SECRET=whsec_xxxxxxxxxxxxx
AWS_ACCESS_KEY_ID=AKIAIOSFODNN7EXAMPLE
AWS_SECRET_ACCESS_KEY=wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
AWS_REGION=ap-southeast-1
AWS_S3_BUCKET=my-app-bucket

# Redis
REDIS_URL=redis://localhost:6379
REDIS_PASSWORD=

# App Settings
APP_URL=http://localhost:3000
FRONTEND_URL=http://localhost:5173
ALLOWED_ORIGINS=http://localhost:3000,http://localhost:5173

# Logging
LOG_LEVEL=debug
LOG_FILE=./logs/app.log
```

### การใช้งาน dotenv

```javascript
// app.js หรือ index.js - ต้องเรียกก่อนโค้ดอื่นทั้งหมด
require('dotenv').config();

// หรือใช้ ES Modules
import 'dotenv/config';

// ระบุ path ของไฟล์ .env
require('dotenv').config({ path: '.env.local' });

// ใช้กับ TypeScript
import * as dotenv from 'dotenv';
dotenv.config();
```

### Best Practices สำหรับ dotenv

#### 1. ใช้ .env.example เป็น template

```bash
# .env.example - commit ไฟล์นี้ขึ้น git
# คัดลอกไฟล์นี้เป็น .env และเติมค่าที่ถูกต้อง

NODE_ENV=development
PORT=3000

DATABASE_URL=postgresql://user:password@localhost:5432/mydb

JWT_SECRET=replace-with-strong-random-secret
JWT_EXPIRES_IN=7d

SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your@email.com
SMTP_PASS=your-email-password

STRIPE_SECRET_KEY=sk_test_your_key_here
AWS_ACCESS_KEY_ID=your_access_key
AWS_SECRET_ACCESS_KEY=your_secret_key
AWS_REGION=ap-southeast-1
```

#### 2. เพิ่ม .env ใน .gitignore

```gitignore
# .gitignore
.env
.env.local
.env.*.local
.env.production

# แต่ยังคง track ไฟล์ example
!.env.example
```

#### 3. สร้าง config module แยกต่างหาก

```javascript
// config/index.js
require('dotenv').config();

const config = {
  env: process.env.NODE_ENV || 'development',
  port: parseInt(process.env.PORT, 10) || 3000,
  
  database: {
    url: process.env.DATABASE_URL,
    host: process.env.DATABASE_HOST || 'localhost',
    port: parseInt(process.env.DATABASE_PORT, 10) || 5432,
    name: process.env.DATABASE_NAME,
    user: process.env.DATABASE_USER,
    password: process.env.DATABASE_PASSWORD,
  },
  
  jwt: {
    secret: process.env.JWT_SECRET,
    expiresIn: process.env.JWT_EXPIRES_IN || '7d',
  },
  
  email: {
    host: process.env.SMTP_HOST,
    port: parseInt(process.env.SMTP_PORT, 10) || 587,
    user: process.env.SMTP_USER,
    pass: process.env.SMTP_PASS,
  },
  
  redis: {
    url: process.env.REDIS_URL || 'redis://localhost:6379',
    password: process.env.REDIS_PASSWORD,
  },
  
  app: {
    url: process.env.APP_URL || 'http://localhost:3000',
    frontendUrl: process.env.FRONTEND_URL || 'http://localhost:5173',
    allowedOrigins: process.env.ALLOWED_ORIGINS
      ? process.env.ALLOWED_ORIGINS.split(',')
      : ['http://localhost:3000'],
  },
};

module.exports = config;
```

---

## Environment-Specific Configs

### โครงสร้างไฟล์ Config หลายสภาพแวดล้อม

```
config/
├── default.js       # ค่า default ทั้งหมด
├── development.js   # override สำหรับ dev
├── staging.js       # override สำหรับ staging
├── production.js    # override สำหรับ production
├── test.js          # override สำหรับ testing
└── index.js         # รวมทุกอย่าง
```

### ไฟล์ Config แต่ละ Environment

```javascript
// config/default.js
module.exports = {
  app: {
    name: 'My Node App',
    version: '1.0.0',
  },
  server: {
    port: 3000,
    host: 'localhost',
  },
  database: {
    dialect: 'postgres',
    port: 5432,
    pool: {
      min: 2,
      max: 10,
      acquire: 30000,
      idle: 10000,
    },
  },
  logging: {
    level: 'info',
    format: 'combined',
  },
  cache: {
    ttl: 3600, // 1 hour
    prefix: 'app:',
  },
};
```

```javascript
// config/development.js
module.exports = {
  server: {
    port: process.env.PORT || 3000,
  },
  database: {
    host: 'localhost',
    name: 'myapp_dev',
    user: 'postgres',
    password: 'postgres',
    logging: true, // log SQL queries ใน dev
  },
  logging: {
    level: 'debug',
  },
};
```

```javascript
// config/production.js
module.exports = {
  server: {
    port: process.env.PORT || 80,
    host: '0.0.0.0',
  },
  database: {
    url: process.env.DATABASE_URL,
    ssl: {
      rejectUnauthorized: false,
    },
    logging: false,
  },
  logging: {
    level: 'warn',
    format: 'json',
  },
  cache: {
    ttl: 86400, // 24 hours ใน production
  },
};
```

```javascript
// config/test.js
module.exports = {
  server: {
    port: 4000, // ใช้ port ต่างกันสำหรับ test
  },
  database: {
    name: 'myapp_test',
    logging: false,
  },
  logging: {
    level: 'error', // ลด noise ระหว่าง test
  },
};
```

### Config Merger

```javascript
// config/index.js
const defaultConfig = require('./default');
const merge = require('lodash/merge');

const env = process.env.NODE_ENV || 'development';

let envConfig = {};
try {
  envConfig = require(`./${env}`);
} catch (err) {
  console.warn(`No config found for environment: ${env}`);
}

// Merge deep - envConfig values override defaultConfig
const config = merge({}, defaultConfig, envConfig);

module.exports = config;

// การใช้งาน
const config = require('./config');
console.log(config.database.host);
console.log(config.server.port);
```

---

## Secrets Management

### หลักการ Secrets Management

```
❌ วิธีที่ไม่ดี:
- Hardcode secrets ใน code
- Commit .env ขึ้น git
- เก็บ secrets ใน comment
- ส่ง secrets ทาง email/Slack

✅ วิธีที่ดี:
- ใช้ Environment Variables
- ใช้ Secret Management Service
- Rotate secrets เป็นประจำ
- จำกัด access ตาม principle of least privilege
```

### ใช้ AWS Secrets Manager

```javascript
// secrets/aws.js
const { SecretsManagerClient, GetSecretValueCommand } = require('@aws-sdk/client-secrets-manager');

const client = new SecretsManagerClient({ region: process.env.AWS_REGION });

async function getSecret(secretName) {
  try {
    const response = await client.send(
      new GetSecretValueCommand({ SecretId: secretName })
    );
    
    if (response.SecretString) {
      return JSON.parse(response.SecretString);
    }
    
    // Binary secret
    const buff = Buffer.from(response.SecretBinary, 'base64');
    return buff.toString('ascii');
  } catch (error) {
    console.error('Error retrieving secret:', error);
    throw error;
  }
}

// ใช้งาน
async function initializeApp() {
  const dbSecrets = await getSecret('myapp/production/database');
  const apiSecrets = await getSecret('myapp/production/api-keys');
  
  // ใช้ secrets
  const dbConnection = createConnection({
    host: dbSecrets.host,
    password: dbSecrets.password,
    // ...
  });
}
```

### ใช้ HashiCorp Vault

```javascript
// secrets/vault.js
const vault = require('node-vault');

const client = vault({
  apiVersion: 'v1',
  endpoint: process.env.VAULT_ADDR || 'http://localhost:8200',
  token: process.env.VAULT_TOKEN,
});

async function getSecret(path) {
  try {
    const result = await client.read(path);
    return result.data.data;
  } catch (error) {
    console.error('Vault error:', error);
    throw error;
  }
}

async function loadSecrets() {
  const dbCreds = await getSecret('secret/myapp/database');
  const apiKeys = await getSecret('secret/myapp/api-keys');
  
  process.env.DATABASE_PASSWORD = dbCreds.password;
  process.env.STRIPE_SECRET_KEY = apiKeys.stripe;
}
```

### Encryption ของ Secrets ด้วย dotenv-vault

```bash
# ติดตั้ง
npm install dotenv --save
npm install @dotenvx/dotenvx --save-dev

# สร้าง encrypted vault
npx dotenvx encrypt

# decrypt และโหลด
npx dotenvx run -- node app.js
```

```javascript
// การใช้ dotenvx สำหรับ encryption
// .env.vault - สามารถ commit ไฟล์นี้ได้ (เพราะ encrypted)
// DOTENV_VAULT_DEVELOPMENT="encrypted-value-here"

require('@dotenvx/dotenvx').config();
// จะ decrypt อัตโนมัติโดยใช้ DOTENV_KEY
```

---

## Config Packages

### node-config

```bash
npm install config
```

```
config/
├── default.json
├── development.json
├── production.json
└── test.json
```

```json
// config/default.json
{
  "app": {
    "name": "My Node App",
    "port": 3000
  },
  "database": {
    "host": "localhost",
    "port": 5432,
    "name": "myapp",
    "pool": {
      "min": 2,
      "max": 10
    }
  },
  "cache": {
    "ttl": 3600
  }
}
```

```json
// config/production.json
{
  "app": {
    "port": 80
  },
  "database": {
    "host": "prod-db.example.com",
    "ssl": true
  },
  "cache": {
    "ttl": 86400
  }
}
```

```javascript
// การใช้งาน node-config
const config = require('config');

const dbConfig = config.get('database');
const port = config.get('app.port');
const cacheConfig = config.has('cache') ? config.get('cache') : {};

console.log('Connecting to:', dbConfig.host);
console.log('Port:', port);

// รองรับ environment variables ด้วย custom-environment-variables.json
// config/custom-environment-variables.json
```

```json
// config/custom-environment-variables.json
{
  "app": {
    "port": "PORT"
  },
  "database": {
    "host": "DATABASE_HOST",
    "password": "DATABASE_PASSWORD",
    "name": "DATABASE_NAME"
  },
  "jwt": {
    "secret": "JWT_SECRET"
  }
}
```

### convict

```bash
npm install convict
npm install @hapi/joi  # สำหรับ validation
```

```javascript
// config/schema.js
const convict = require('convict');

// เพิ่ม custom formats
convict.addFormats(require('convict-format-with-validator'));

const config = convict({
  env: {
    doc: 'The application environment.',
    format: ['production', 'development', 'test'],
    default: 'development',
    env: 'NODE_ENV',
  },
  
  port: {
    doc: 'The port to bind.',
    format: 'port',
    default: 3000,
    env: 'PORT',
    arg: 'port',
  },
  
  database: {
    url: {
      doc: 'Database connection URL',
      format: String,
      default: null,
      env: 'DATABASE_URL',
      sensitive: true, // จะ mask ใน logs
    },
    host: {
      doc: 'Database host',
      format: String,
      default: 'localhost',
      env: 'DATABASE_HOST',
    },
    port: {
      doc: 'Database port',
      format: 'port',
      default: 5432,
      env: 'DATABASE_PORT',
    },
    name: {
      doc: 'Database name',
      format: String,
      default: 'myapp',
      env: 'DATABASE_NAME',
    },
  },
  
  jwt: {
    secret: {
      doc: 'JWT signing secret',
      format: String,
      default: null,
      env: 'JWT_SECRET',
      sensitive: true,
    },
    expiresIn: {
      doc: 'JWT expiration time',
      format: String,
      default: '7d',
      env: 'JWT_EXPIRES_IN',
    },
  },
  
  redis: {
    url: {
      doc: 'Redis URL',
      format: String,
      default: 'redis://localhost:6379',
      env: 'REDIS_URL',
    },
  },
  
  logging: {
    level: {
      doc: 'Log level',
      format: ['trace', 'debug', 'info', 'warn', 'error', 'fatal'],
      default: 'info',
      env: 'LOG_LEVEL',
    },
  },
});

// Load environment-specific config files
const env = config.get('env');
config.loadFile(`./config/${env}.json`);

// Validate the config against the schema
config.validate({ allowed: 'strict' });

module.exports = config;
```

```javascript
// การใช้งาน convict
const config = require('./config/schema');

const port = config.get('port');
const dbUrl = config.get('database.url');
const jwtSecret = config.get('jwt.secret');

console.log(`Starting server on port ${port}`);
```

---

## Validation ด้วย Joi และ Zod

### Validation ด้วย Joi

```bash
npm install joi
```

```javascript
// config/validate.js
const Joi = require('joi');

// Schema สำหรับ validate environment variables
const envSchema = Joi.object({
  // App
  NODE_ENV: Joi.string()
    .valid('development', 'production', 'test', 'staging')
    .default('development'),
  PORT: Joi.number().port().default(3000),
  
  // Database
  DATABASE_URL: Joi.string().uri().when('NODE_ENV', {
    is: 'production',
    then: Joi.required(),
    otherwise: Joi.optional(),
  }),
  DATABASE_HOST: Joi.string().hostname().default('localhost'),
  DATABASE_PORT: Joi.number().port().default(5432),
  DATABASE_NAME: Joi.string().required(),
  DATABASE_USER: Joi.string().required(),
  DATABASE_PASSWORD: Joi.string().required(),
  
  // JWT
  JWT_SECRET: Joi.string().min(32).required(),
  JWT_EXPIRES_IN: Joi.string()
    .pattern(/^\d+[smhd]$/)
    .default('7d'),
  
  // Email
  SMTP_HOST: Joi.string().hostname().optional(),
  SMTP_PORT: Joi.number().port().default(587),
  SMTP_USER: Joi.string().email().optional(),
  SMTP_PASS: Joi.string().optional(),
  
  // Redis
  REDIS_URL: Joi.string().uri().default('redis://localhost:6379'),
  
  // Stripe
  STRIPE_SECRET_KEY: Joi.string()
    .pattern(/^sk_(test|live)_/)
    .optional(),
  
  // AWS
  AWS_REGION: Joi.string()
    .valid('us-east-1', 'ap-southeast-1', 'eu-west-1')
    .optional(),
  AWS_ACCESS_KEY_ID: Joi.string().optional(),
  AWS_SECRET_ACCESS_KEY: Joi.string().optional(),
  
  // Logging
  LOG_LEVEL: Joi.string()
    .valid('debug', 'info', 'warn', 'error')
    .default('info'),
    
}).unknown(true); // อนุญาต env vars อื่นๆ ที่ไม่ได้ define

function validateEnv() {
  const { error, value } = envSchema.validate(process.env);
  
  if (error) {
    throw new Error(`Config validation error: ${error.message}`);
  }
  
  return value;
}

module.exports = { validateEnv };
```

```javascript
// การใช้งาน Joi validation
require('dotenv').config();
const { validateEnv } = require('./config/validate');

// Validate ก่อน start app
const env = validateEnv();

// App จะ crash ถ้า config ไม่ถูกต้อง
// แสดง error message ที่ชัดเจน
```

### Validation ด้วย Zod

```bash
npm install zod
```

```javascript
// config/validate-zod.js
const { z } = require('zod');

const envSchema = z.object({
  // App
  NODE_ENV: z.enum(['development', 'production', 'test', 'staging'])
    .default('development'),
  PORT: z.coerce.number().int().min(1).max(65535).default(3000),
  
  // Database
  DATABASE_URL: z.string().url().optional(),
  DATABASE_HOST: z.string().default('localhost'),
  DATABASE_PORT: z.coerce.number().int().default(5432),
  DATABASE_NAME: z.string().min(1),
  DATABASE_USER: z.string().min(1),
  DATABASE_PASSWORD: z.string().min(1),
  
  // JWT
  JWT_SECRET: z.string().min(32, 'JWT_SECRET must be at least 32 characters'),
  JWT_EXPIRES_IN: z.string().default('7d'),
  
  // Email (optional)
  SMTP_HOST: z.string().optional(),
  SMTP_PORT: z.coerce.number().int().default(587),
  SMTP_USER: z.string().email().optional(),
  SMTP_PASS: z.string().optional(),
  
  // Redis
  REDIS_URL: z.string().default('redis://localhost:6379'),
  
  // Logging
  LOG_LEVEL: z.enum(['debug', 'info', 'warn', 'error']).default('info'),
});

function validateEnv() {
  const result = envSchema.safeParse(process.env);
  
  if (!result.success) {
    console.error('❌ Invalid environment variables:');
    
    const errors = result.error.flatten().fieldErrors;
    Object.entries(errors).forEach(([field, messages]) => {
      console.error(`  ${field}: ${messages.join(', ')}`);
    });
    
    process.exit(1);
  }
  
  return result.data;
}

// Type-safe config (สำหรับ TypeScript)
// type Env = z.infer<typeof envSchema>;

module.exports = { validateEnv, envSchema };
```

```typescript
// TypeScript version
import { z } from 'zod';

const envSchema = z.object({
  NODE_ENV: z.enum(['development', 'production', 'test']),
  PORT: z.coerce.number().default(3000),
  DATABASE_URL: z.string().url(),
  JWT_SECRET: z.string().min(32),
  REDIS_URL: z.string().default('redis://localhost:6379'),
});

type Env = z.infer<typeof envSchema>;

function validateEnv(): Env {
  const result = envSchema.safeParse(process.env);
  
  if (!result.success) {
    throw new Error(`Invalid env: ${result.error.message}`);
  }
  
  return result.data;
}

export const env = validateEnv();

// ใช้งาน - TypeScript รู้ types ทั้งหมด
console.log(env.PORT);         // number
console.log(env.DATABASE_URL); // string
```

---

## Workshop: ระบบ Config จริง

### โครงสร้าง Project

```
my-app/
├── config/
│   ├── index.js          # main config
│   ├── schema.js         # Joi/Zod schema
│   ├── default.js        # defaults
│   ├── development.js    # dev overrides
│   └── production.js     # prod overrides
├── src/
│   └── app.js
├── .env
├── .env.example
└── package.json
```

### Complete Config System

```javascript
// config/schema.js
const { z } = require('zod');

const schema = z.object({
  NODE_ENV: z.enum(['development', 'production', 'test']).default('development'),
  PORT: z.coerce.number().default(3000),
  
  // Database
  DB_HOST: z.string().default('localhost'),
  DB_PORT: z.coerce.number().default(5432),
  DB_NAME: z.string(),
  DB_USER: z.string(),
  DB_PASS: z.string(),
  
  // Auth
  JWT_SECRET: z.string().min(32),
  JWT_EXPIRES: z.string().default('1d'),
  
  // Cache
  REDIS_URL: z.string().default('redis://localhost:6379'),
  CACHE_TTL: z.coerce.number().default(3600),
  
  // App
  ALLOWED_ORIGINS: z.string()
    .transform(val => val.split(',').map(s => s.trim()))
    .default('http://localhost:3000'),
  
  LOG_LEVEL: z.enum(['debug', 'info', 'warn', 'error']).default('info'),
});

module.exports = schema;
```

```javascript
// config/index.js
require('dotenv').config();
const schema = require('./schema');

// Validate
const result = schema.safeParse(process.env);
if (!result.success) {
  console.error('Configuration errors:');
  result.error.issues.forEach(issue => {
    console.error(`  [${issue.path.join('.')}] ${issue.message}`);
  });
  process.exit(1);
}

const env = result.data;

// Build structured config
const config = {
  env: env.NODE_ENV,
  isDev: env.NODE_ENV === 'development',
  isProd: env.NODE_ENV === 'production',
  isTest: env.NODE_ENV === 'test',
  
  server: {
    port: env.PORT,
    allowedOrigins: env.ALLOWED_ORIGINS,
  },
  
  db: {
    host: env.DB_HOST,
    port: env.DB_PORT,
    name: env.DB_NAME,
    user: env.DB_USER,
    password: env.DB_PASS,
    url: `postgresql://${env.DB_USER}:${env.DB_PASS}@${env.DB_HOST}:${env.DB_PORT}/${env.DB_NAME}`,
    ssl: env.NODE_ENV === 'production' ? { rejectUnauthorized: false } : false,
    pool: {
      min: env.NODE_ENV === 'production' ? 5 : 2,
      max: env.NODE_ENV === 'production' ? 20 : 5,
    },
  },
  
  auth: {
    jwtSecret: env.JWT_SECRET,
    jwtExpires: env.JWT_EXPIRES,
  },
  
  cache: {
    url: env.REDIS_URL,
    ttl: env.CACHE_TTL,
  },
  
  logging: {
    level: env.LOG_LEVEL,
    prettyPrint: env.NODE_ENV === 'development',
  },
};

module.exports = config;
```

```javascript
// src/app.js
const express = require('express');
const config = require('../config');

const app = express();

app.get('/health', (req, res) => {
  res.json({
    status: 'ok',
    env: config.env,
    version: process.env.npm_package_version,
  });
});

app.listen(config.server.port, () => {
  console.log(`
🚀 Server started
   Environment: ${config.env}
   Port: ${config.server.port}
   Database: ${config.db.host}:${config.db.port}/${config.db.name}
   Cache: ${config.cache.url}
  `);
});
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Basic dotenv Setup

สร้างโปรเจกต์ Express ที่:
1. ใช้ dotenv โหลด environment variables
2. สร้าง config module ที่ structured และมี defaults
3. สร้าง .env.example สำหรับ documentation
4. หน้า `/config` แสดง non-sensitive config (ห้ามแสดง secrets)

```javascript
// TODO: สร้างไฟล์เหล่านี้
// - .env (ห้าม commit)
// - .env.example (commit ได้)
// - config/index.js
// - src/app.js

// Expected output ของ GET /config:
// {
//   "env": "development",
//   "port": 3000,
//   "database": {
//     "host": "localhost",
//     "port": 5432,
//     "name": "myapp_dev"
//   }
// }
```

### แบบฝึกหัดที่ 2: Zod Validation

เพิ่ม Zod validation ให้กับ config:
1. Required fields: DATABASE_URL, JWT_SECRET (min 32 chars)
2. Optional fields with defaults: PORT (3000), LOG_LEVEL ('info')
3. Custom transform: ALLOWED_ORIGINS string → array
4. App ต้อง fail fast ด้วย clear error ถ้า config ไม่ถูกต้อง

### แบบฝึกหัดที่ 3: Multi-Environment Config

สร้าง config system ที่รองรับ 3 environments:
- **development**: verbose logging, local DB, no SSL
- **staging**: warning logging, staging DB, SSL
- **production**: error logging, prod DB, SSL, connection pooling

```javascript
// Test cases
// NODE_ENV=development node app.js
// → database.ssl = false, logging.level = 'debug'

// NODE_ENV=production node app.js  
// → database.ssl = {rejectUnauthorized: false}, logging.level = 'warn'
```

### แบบฝึกหัดที่ 4: Secrets Rotation

สร้าง script สำหรับ rotate JWT secret:
1. Generate ค่า secret ใหม่ (32+ chars random)
2. อัพเดต .env ไฟล์
3. Log timestamp ของการ rotation
4. ทำให้ตรวจสอบ secret ที่ expire ได้

```javascript
// scripts/rotate-secret.js
const crypto = require('crypto');
const fs = require('fs');
const path = require('path');

function generateSecret(length = 64) {
  return crypto.randomBytes(length).toString('hex');
}

function rotateJwtSecret() {
  const envPath = path.resolve('.env');
  // TODO: implement
}

rotateJwtSecret();
```

---

## สรุป

Environment Variables เป็นส่วนสำคัญของ Node.js application ที่ดี หลักการสำคัญ:

| หัวข้อ | Best Practice |
|--------|--------------|
| Secrets | ไม่ commit ขึ้น git เด็ดขาด |
| Validation | Validate ทุก env var ตอน startup |
| Defaults | มีค่า default สำหรับ optional settings |
| Structure | จัด config เป็น module ที่ structured |
| Documentation | มี .env.example ที่อัพเดต |
| Environments | แยก config สำหรับแต่ละ environment |

### เครื่องมือที่แนะนำ

- **dotenv**: อ่านไฟล์ .env
- **Zod**: Type-safe validation
- **convict**: Schema-based config management
- **node-config**: File-based multi-environment config
- **dotenv-vault**: Encrypted .env files

---

**ถัดไป**: [Part 32: MVC Architecture →](./part-32-mvc-architecture.md)
