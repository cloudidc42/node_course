# Part 52 | ขั้นตอนที่ 901-920 จาก 1000

## API Documentation - การจัดทำเอกสาร API

การทำ documentation ที่ดีเป็นส่วนสำคัญของการพัฒนา API ช่วยให้นักพัฒนาเข้าใจและใช้งาน API ได้อย่างถูกต้อง

---

## ขั้นตอนที่ 901: ทำความเข้าใจ API Documentation

### เครื่องมือหลักในการทำ Documentation:
1. **Swagger/OpenAPI** - มาตรฐานอุตสาหกรรม
2. **JSDoc** - Documentation ใน code comments
3. **Postman Collections** - Interactive testing + documentation
4. **Redoc** - Beautiful API documentation
5. **API Blueprint** - Markdown-based documentation

---

## ขั้นตอนที่ 902: ติดตั้ง Swagger/OpenAPI

```bash
npm install swagger-jsdoc swagger-ui-express
```

```javascript
// config/swagger.js
const swaggerJsdoc = require('swagger-jsdoc');

const options = {
  definition: {
    openapi: '3.0.0',
    info: {
      title: 'My API',
      version: '2.0.0',
      description: 'API documentation สำหรับระบบ E-Commerce',
      termsOfService: 'http://example.com/terms/',
      contact: {
        name: 'API Support',
        url: 'http://example.com/support',
        email: 'api@example.com'
      },
      license: {
        name: 'MIT',
        url: 'https://opensource.org/licenses/MIT'
      }
    },
    servers: [
      {
        url: 'http://localhost:3000/api/v2',
        description: 'Development server'
      },
      {
        url: 'https://api.example.com/v2',
        description: 'Production server'
      }
    ],
    components: {
      securitySchemes: {
        bearerAuth: {
          type: 'http',
          scheme: 'bearer',
          bearerFormat: 'JWT',
          description: 'Enter JWT token'
        },
        apiKeyAuth: {
          type: 'apiKey',
          in: 'header',
          name: 'X-API-Key'
        }
      },
      schemas: {
        Error: {
          type: 'object',
          properties: {
            success: { type: 'boolean', example: false },
            error: {
              type: 'object',
              properties: {
                code: { type: 'string', example: 'NOT_FOUND' },
                message: { type: 'string', example: 'Resource not found' }
              }
            }
          }
        },
        Pagination: {
          type: 'object',
          properties: {
            page: { type: 'integer', example: 1 },
            limit: { type: 'integer', example: 20 },
            total: { type: 'integer', example: 100 },
            pages: { type: 'integer', example: 5 }
          }
        }
      },
      responses: {
        Unauthorized: {
          description: 'Unauthorized - Invalid or missing token',
          content: {
            'application/json': {
              schema: { $ref: '#/components/schemas/Error' }
            }
          }
        },
        NotFound: {
          description: 'Resource not found',
          content: {
            'application/json': {
              schema: { $ref: '#/components/schemas/Error' }
            }
          }
        }
      }
    },
    security: [{ bearerAuth: [] }]
  },
  apis: [
    './routes/**/*.js',
    './models/*.js',
    './docs/schemas/*.yaml'
  ]
};

const specs = swaggerJsdoc(options);

module.exports = specs;
```

---

## ขั้นตอนที่ 903: Setup Swagger UI

```javascript
// app.js
const swaggerUi = require('swagger-ui-express');
const specs = require('./config/swagger');

// Custom Swagger UI options
const swaggerUiOptions = {
  customCss: '.swagger-ui .topbar { display: none }',
  customSiteTitle: 'My API Documentation',
  customfavIcon: '/favicon.ico',
  swaggerOptions: {
    persistAuthorization: true, // จำ token ข้าม browser refresh
    displayRequestDuration: true,
    docExpansion: 'none',
    filter: true,
    showCommonExtensions: true
  }
};

// Serve Swagger UI
app.use('/api/docs', swaggerUi.serve, swaggerUi.setup(specs, swaggerUiOptions));

// Serve raw OpenAPI spec
app.get('/api/openapi.json', (req, res) => {
  res.setHeader('Content-Type', 'application/json');
  res.json(specs);
});

app.get('/api/openapi.yaml', (req, res) => {
  const yaml = require('yaml');
  res.setHeader('Content-Type', 'text/yaml');
  res.send(yaml.stringify(specs));
});
```

---

## ขั้นตอนที่ 904: JSDoc สำหรับ Routes

```javascript
// routes/users.js

/**
 * @swagger
 * tags:
 *   name: Users
 *   description: User management endpoints
 */

/**
 * @swagger
 * components:
 *   schemas:
 *     User:
 *       type: object
 *       required:
 *         - name
 *         - email
 *       properties:
 *         _id:
 *           type: string
 *           description: MongoDB ObjectId
 *           example: "507f1f77bcf86cd799439011"
 *         name:
 *           type: string
 *           description: ชื่อผู้ใช้
 *           example: "สมชาย ใจดี"
 *           minLength: 2
 *           maxLength: 100
 *         email:
 *           type: string
 *           format: email
 *           description: อีเมลผู้ใช้
 *           example: "somchai@example.com"
 *         role:
 *           type: string
 *           enum: [user, admin, moderator]
 *           default: user
 *           description: บทบาทของผู้ใช้
 *         createdAt:
 *           type: string
 *           format: date-time
 *           description: วันที่สร้าง
 *         updatedAt:
 *           type: string
 *           format: date-time
 *           description: วันที่แก้ไขล่าสุด
 *     UserCreate:
 *       type: object
 *       required:
 *         - name
 *         - email
 *         - password
 *       properties:
 *         name:
 *           type: string
 *           example: "สมชาย ใจดี"
 *         email:
 *           type: string
 *           format: email
 *           example: "somchai@example.com"
 *         password:
 *           type: string
 *           format: password
 *           minLength: 8
 *           example: "SecurePass123"
 */

const express = require('express');
const router = express.Router();

/**
 * @swagger
 * /users:
 *   get:
 *     summary: ดึงรายชื่อผู้ใช้ทั้งหมด
 *     description: ดึงรายชื่อผู้ใช้ทั้งหมดพร้อม pagination และ filtering
 *     tags: [Users]
 *     security:
 *       - bearerAuth: []
 *     parameters:
 *       - in: query
 *         name: page
 *         schema:
 *           type: integer
 *           minimum: 1
 *           default: 1
 *         description: หมายเลขหน้า
 *       - in: query
 *         name: limit
 *         schema:
 *           type: integer
 *           minimum: 1
 *           maximum: 100
 *           default: 20
 *         description: จำนวนผู้ใช้ต่อหน้า
 *       - in: query
 *         name: search
 *         schema:
 *           type: string
 *         description: ค้นหาจากชื่อหรืออีเมล
 *       - in: query
 *         name: role
 *         schema:
 *           type: string
 *           enum: [user, admin, moderator]
 *         description: กรองตาม role
 *     responses:
 *       200:
 *         description: รายชื่อผู้ใช้
 *         content:
 *           application/json:
 *             schema:
 *               type: object
 *               properties:
 *                 success:
 *                   type: boolean
 *                   example: true
 *                 data:
 *                   type: array
 *                   items:
 *                     $ref: '#/components/schemas/User'
 *                 pagination:
 *                   $ref: '#/components/schemas/Pagination'
 *             example:
 *               success: true
 *               data:
 *                 - _id: "507f1f77bcf86cd799439011"
 *                   name: "สมชาย ใจดี"
 *                   email: "somchai@example.com"
 *                   role: "user"
 *               pagination:
 *                 page: 1
 *                 limit: 20
 *                 total: 100
 *                 pages: 5
 *       401:
 *         $ref: '#/components/responses/Unauthorized'
 */
router.get('/', authenticate, UserController.getAll);

/**
 * @swagger
 * /users/{id}:
 *   get:
 *     summary: ดึงข้อมูลผู้ใช้ตาม ID
 *     tags: [Users]
 *     security:
 *       - bearerAuth: []
 *     parameters:
 *       - in: path
 *         name: id
 *         required: true
 *         schema:
 *           type: string
 *         description: MongoDB ObjectId ของผู้ใช้
 *         example: "507f1f77bcf86cd799439011"
 *     responses:
 *       200:
 *         description: ข้อมูลผู้ใช้
 *         content:
 *           application/json:
 *             schema:
 *               type: object
 *               properties:
 *                 success:
 *                   type: boolean
 *                 data:
 *                   $ref: '#/components/schemas/User'
 *       404:
 *         $ref: '#/components/responses/NotFound'
 */
router.get('/:id', authenticate, UserController.getById);

/**
 * @swagger
 * /users:
 *   post:
 *     summary: สร้างผู้ใช้ใหม่
 *     tags: [Users]
 *     requestBody:
 *       required: true
 *       content:
 *         application/json:
 *           schema:
 *             $ref: '#/components/schemas/UserCreate'
 *           example:
 *             name: "สมชาย ใจดี"
 *             email: "somchai@example.com"
 *             password: "SecurePass123"
 *     responses:
 *       201:
 *         description: ผู้ใช้ถูกสร้างสำเร็จ
 *         content:
 *           application/json:
 *             schema:
 *               type: object
 *               properties:
 *                 success:
 *                   type: boolean
 *                   example: true
 *                 data:
 *                   $ref: '#/components/schemas/User'
 *       422:
 *         description: ข้อมูลไม่ถูกต้อง
 *         content:
 *           application/json:
 *             schema:
 *               type: object
 *               properties:
 *                 success:
 *                   type: boolean
 *                   example: false
 *                 errors:
 *                   type: array
 *                   items:
 *                     type: object
 *                     properties:
 *                       field:
 *                         type: string
 *                       message:
 *                         type: string
 */
router.post('/', validate('create'), UserController.create);

module.exports = router;
```

---

## ขั้นตอนที่ 905: OpenAPI YAML Schemas

```yaml
# docs/schemas/product.yaml

components:
  schemas:
    Product:
      type: object
      properties:
        _id:
          type: string
          description: MongoDB ObjectId
          example: "507f1f77bcf86cd799439011"
        name:
          type: string
          description: ชื่อสินค้า
          example: "iPhone 15 Pro"
        slug:
          type: string
          description: URL-friendly name
          example: "iphone-15-pro"
        description:
          type: string
          description: รายละเอียดสินค้า
        price:
          type: number
          format: float
          description: ราคาสินค้า (บาท)
          example: 49900
          minimum: 0
        category:
          type: object
          properties:
            _id:
              type: string
            name:
              type: string
        stock:
          type: integer
          description: จำนวนสินค้าคงเหลือ
          example: 50
          minimum: 0
        images:
          type: array
          items:
            type: string
            format: uri
          example:
            - "https://cdn.example.com/products/iphone-15-pro-1.jpg"
        isActive:
          type: boolean
          default: true
        tags:
          type: array
          items:
            type: string
          example: ["smartphone", "apple", "premium"]
        rating:
          type: number
          minimum: 0
          maximum: 5
          example: 4.8
        reviewCount:
          type: integer
          example: 128
        createdAt:
          type: string
          format: date-time

    ProductCreate:
      type: object
      required:
        - name
        - price
        - category
      properties:
        name:
          type: string
          minLength: 2
          maxLength: 255
        price:
          type: number
          minimum: 0
        category:
          type: string
          description: Category ID
        description:
          type: string
        stock:
          type: integer
          minimum: 0
          default: 0
        tags:
          type: array
          items:
            type: string

    ProductUpdate:
      type: object
      minProperties: 1
      properties:
        name:
          type: string
        price:
          type: number
          minimum: 0
        description:
          type: string
        stock:
          type: integer
          minimum: 0
        isActive:
          type: boolean
        tags:
          type: array
          items:
            type: string
```

---

## ขั้นตอนที่ 906: JSDoc สำหรับ Service Classes

```javascript
// services/userService.js

/**
 * @module UserService
 * @description บริการจัดการผู้ใช้งาน
 */

/**
 * @typedef {Object} UserData
 * @property {string} name - ชื่อผู้ใช้
 * @property {string} email - อีเมล
 * @property {string} password - รหัสผ่าน (จะถูก hash)
 * @property {string} [role='user'] - บทบาท: user, admin, moderator
 */

/**
 * @typedef {Object} PaginationOptions
 * @property {number} [page=1] - หมายเลขหน้า
 * @property {number} [limit=20] - จำนวนต่อหน้า
 * @property {string} [sort='-createdAt'] - การเรียงลำดับ
 * @property {string} [search] - คำค้นหา
 */

/**
 * @typedef {Object} PaginatedResult
 * @property {Array} data - ข้อมูล
 * @property {Object} pagination - ข้อมูล pagination
 * @property {number} pagination.total - จำนวนทั้งหมด
 * @property {number} pagination.page - หน้าปัจจุบัน
 * @property {number} pagination.pages - จำนวนหน้าทั้งหมด
 */

class UserService {
  /**
   * ดึงรายชื่อผู้ใช้ทั้งหมดพร้อม pagination
   * @param {PaginationOptions} options - ตัวเลือก pagination
   * @returns {Promise<PaginatedResult>} รายชื่อผู้ใช้พร้อม pagination info
   * @throws {Error} ถ้าเกิดข้อผิดพลาด database
   * 
   * @example
   * const result = await userService.getUsers({ page: 1, limit: 10, search: 'john' });
   * console.log(result.data); // [{name: 'John', ...}]
   * console.log(result.pagination); // {total: 100, page: 1, pages: 10}
   */
  async getUsers(options = {}) {
    const { page = 1, limit = 20, sort = '-createdAt', search } = options;
    
    const query = {};
    if (search) {
      query.$or = [
        { name: { $regex: search, $options: 'i' } },
        { email: { $regex: search, $options: 'i' } }
      ];
    }
    
    const [users, total] = await Promise.all([
      User.find(query)
        .sort(sort)
        .skip((page - 1) * limit)
        .limit(parseInt(limit))
        .select('-password')
        .lean(),
      User.countDocuments(query)
    ]);
    
    return {
      data: users,
      pagination: {
        total,
        page: parseInt(page),
        limit: parseInt(limit),
        pages: Math.ceil(total / limit)
      }
    };
  }

  /**
   * สร้างผู้ใช้ใหม่
   * @param {UserData} userData - ข้อมูลผู้ใช้
   * @returns {Promise<Object>} ผู้ใช้ที่สร้างใหม่ (ไม่มี password)
   * @throws {ConflictError} ถ้า email ซ้ำ
   * @throws {ValidationError} ถ้าข้อมูลไม่ถูกต้อง
   * 
   * @example
   * const user = await userService.createUser({
   *   name: 'John Doe',
   *   email: 'john@example.com',
   *   password: 'SecurePass123'
   * });
   */
  async createUser(userData) {
    const existingUser = await User.findOne({ email: userData.email });
    
    if (existingUser) {
      throw new ConflictError('Email already exists');
    }
    
    const user = await User.create(userData);
    const { password, ...userWithoutPassword } = user.toObject();
    
    return userWithoutPassword;
  }
}
```

---

## ขั้นตอนที่ 907: Postman Collection

```javascript
// scripts/generatePostmanCollection.js
const swaggerToPostman = require('swagger2-to-postman');
const fs = require('fs');
const specs = require('../config/swagger');

// สร้าง Postman collection จาก Swagger spec
const generatePostmanCollection = () => {
  const postmanOptions = {
    requestName: 'URL',
    indentCharacter: '\t'
  };
  
  const result = swaggerToPostman.convert(specs, postmanOptions);
  
  if (result.result) {
    const collection = result.output[0].data;
    
    // เพิ่ม environment variables
    const environment = {
      id: 'my-api-env',
      name: 'My API Environment',
      values: [
        {
          key: 'base_url',
          value: 'http://localhost:3000/api/v2',
          enabled: true
        },
        {
          key: 'token',
          value: '',
          enabled: true
        }
      ]
    };
    
    // บันทึกไฟล์
    fs.writeFileSync(
      './docs/postman/my-api.postman_collection.json',
      JSON.stringify(collection, null, 2)
    );
    
    fs.writeFileSync(
      './docs/postman/my-api.postman_environment.json',
      JSON.stringify(environment, null, 2)
    );
    
    console.log('Postman collection generated successfully');
  }
};

generatePostmanCollection();
```

```json
// docs/postman/my-api.postman_collection.json (ตัวอย่าง structure)
{
  "info": {
    "name": "My API Collection",
    "schema": "https://schema.getpostman.com/json/collection/v2.1.0/collection.json"
  },
  "auth": {
    "type": "bearer",
    "bearer": [
      {
        "key": "token",
        "value": "{{token}}",
        "type": "string"
      }
    ]
  },
  "item": [
    {
      "name": "Authentication",
      "item": [
        {
          "name": "Login",
          "event": [
            {
              "listen": "test",
              "script": {
                "exec": [
                  "const response = pm.response.json();",
                  "if (response.data && response.data.token) {",
                  "  pm.environment.set('token', response.data.token);",
                  "  pm.test('Token saved', () => pm.expect(response.data.token).to.be.a('string'));",
                  "}"
                ]
              }
            }
          ],
          "request": {
            "method": "POST",
            "url": "{{base_url}}/auth/login",
            "body": {
              "mode": "raw",
              "raw": "{\n  \"email\": \"test@example.com\",\n  \"password\": \"password123\"\n}",
              "options": {
                "raw": { "language": "json" }
              }
            }
          }
        }
      ]
    },
    {
      "name": "Users",
      "item": [
        {
          "name": "Get All Users",
          "request": {
            "method": "GET",
            "url": {
              "raw": "{{base_url}}/users?page=1&limit=20",
              "host": ["{{base_url}}"],
              "path": ["users"],
              "query": [
                { "key": "page", "value": "1" },
                { "key": "limit", "value": "20" }
              ]
            }
          }
        }
      ]
    }
  ]
}
```

---

## ขั้นตอนที่ 908: ReDoc Documentation

```javascript
// routes/docs.js
const express = require('express');
const router = express.Router();
const redoc = require('redoc-express');
const specs = require('../config/swagger');

// Serve ReDoc
router.get('/redoc', redoc({
  title: 'My API Documentation',
  specUrl: '/api/openapi.json'
}));

// Custom ReDoc page
router.get('/docs', (req, res) => {
  res.send(`
    <!DOCTYPE html>
    <html>
      <head>
        <title>My API Documentation</title>
        <meta charset="utf-8"/>
        <meta name="viewport" content="width=device-width, initial-scale=1">
        <link href="https://fonts.googleapis.com/css?family=Montserrat:300,400,700|Roboto:300,400,700" rel="stylesheet">
        <style>
          body { margin: 0; padding: 0; }
          .page-header {
            background: #1a1a2e;
            color: white;
            padding: 20px;
            display: flex;
            align-items: center;
            justify-content: space-between;
          }
          .api-version { font-size: 0.8em; opacity: 0.7; }
        </style>
      </head>
      <body>
        <div class="page-header">
          <div>
            <h1 style="margin: 0">My API</h1>
            <div class="api-version">Version 2.0.0</div>
          </div>
          <div>
            <a href="/api/docs" style="color: white; margin: 0 10px">Swagger UI</a>
            <a href="/api/openapi.json" style="color: white">OpenAPI JSON</a>
          </div>
        </div>
        <redoc spec-url='/api/openapi.json'></redoc>
        <script src="https://cdn.jsdelivr.net/npm/redoc/bundles/redoc.standalone.js"></script>
      </body>
    </html>
  `);
});

module.exports = router;
```

---

## ขั้นตอนที่ 909: Automated Documentation Testing

```javascript
// tests/documentation.test.js
const request = require('supertest');
const app = require('../app');
const Ajv = require('ajv');
const addFormats = require('ajv-formats');

const ajv = new Ajv({ allErrors: true });
addFormats(ajv);

describe('API Documentation Tests', () => {
  let specs;
  
  beforeAll(async () => {
    const res = await request(app).get('/api/openapi.json');
    specs = res.body;
  });

  it('should serve OpenAPI spec', async () => {
    const res = await request(app)
      .get('/api/openapi.json')
      .expect(200)
      .expect('Content-Type', /json/);
    
    expect(res.body).toHaveProperty('openapi');
    expect(res.body).toHaveProperty('info');
    expect(res.body).toHaveProperty('paths');
  });

  it('should have all required info fields', () => {
    expect(specs.info).toHaveProperty('title');
    expect(specs.info).toHaveProperty('version');
    expect(specs.info).toHaveProperty('description');
  });

  it('should have security schemes defined', () => {
    expect(specs.components).toHaveProperty('securitySchemes');
    expect(specs.components.securitySchemes).toHaveProperty('bearerAuth');
  });

  it('should validate against OpenAPI 3.0 schema', async () => {
    const { validate } = require('openapi-schema-validator');
    const result = validate(specs);
    expect(result.errors).toHaveLength(0);
  });

  // ทดสอบว่า endpoints ที่ document ไว้มีจริง
  it('should have working endpoints for all documented paths', async () => {
    const paths = Object.keys(specs.paths);
    
    for (const path of paths.slice(0, 5)) { // ทดสอบแค่ 5 paths แรก
      const methods = Object.keys(specs.paths[path]);
      
      for (const method of methods) {
        if (method === 'get') {
          const res = await request(app)[method](
            `/api/v2${path.replace(/{[^}]+}/g, 'test-id')}`
          );
          
          // ต้องได้ response (ไม่ใช่ 404 จาก missing route)
          expect(res.status).not.toBe(404);
        }
      }
    }
  });
});
```

---

## ขั้นตอนที่ 910: API Changelog Documentation

```javascript
// docs/CHANGELOG.md generator
const fs = require('fs');
const path = require('path');

class ChangelogGenerator {
  constructor() {
    this.changes = [];
  }

  addChange(version, date, changes) {
    this.changes.push({ version, date, changes });
    return this;
  }

  generateMarkdown() {
    let markdown = '# API Changelog\n\n';
    markdown += 'All notable changes to this API will be documented here.\n\n';
    markdown += '---\n\n';
    
    for (const release of this.changes) {
      markdown += `## Version ${release.version} (${release.date})\n\n`;
      
      const breaking = release.changes.filter(c => c.type === 'breaking');
      const features = release.changes.filter(c => c.type === 'feature');
      const fixes = release.changes.filter(c => c.type === 'fix');
      const improvements = release.changes.filter(c => c.type === 'improvement');
      
      if (breaking.length > 0) {
        markdown += '### ⚠️ Breaking Changes\n';
        breaking.forEach(c => {
          markdown += `- **${c.endpoint || ''}** ${c.description}\n`;
          if (c.migration) {
            markdown += `  > Migration: ${c.migration}\n`;
          }
        });
        markdown += '\n';
      }
      
      if (features.length > 0) {
        markdown += '### ✨ New Features\n';
        features.forEach(c => markdown += `- ${c.description}\n`);
        markdown += '\n';
      }
      
      if (improvements.length > 0) {
        markdown += '### 🔧 Improvements\n';
        improvements.forEach(c => markdown += `- ${c.description}\n`);
        markdown += '\n';
      }
      
      if (fixes.length > 0) {
        markdown += '### 🐛 Bug Fixes\n';
        fixes.forEach(c => markdown += `- ${c.description}\n`);
        markdown += '\n';
      }
      
      markdown += '---\n\n';
    }
    
    return markdown;
  }

  writeToFile(filePath = './docs/CHANGELOG.md') {
    const markdown = this.generateMarkdown();
    fs.writeFileSync(filePath, markdown);
    console.log(`Changelog written to ${filePath}`);
  }
}

const generator = new ChangelogGenerator();

generator
  .addChange('2.5.0', '2024-03-01', [
    {
      type: 'feature',
      description: 'Added full-text search for products'
    },
    {
      type: 'improvement',
      description: 'Improved error messages with machine-readable codes'
    }
  ])
  .addChange('2.0.0', '2024-01-01', [
    {
      type: 'breaking',
      endpoint: 'ALL',
      description: 'Response format changed to { success, data, meta }',
      migration: 'Update all API calls to handle new response format'
    },
    {
      type: 'breaking',
      endpoint: 'PUT /users/:id',
      description: 'PUT replaced by PATCH for partial updates',
      migration: 'Change HTTP method from PUT to PATCH'
    },
    {
      type: 'feature',
      description: 'Added pagination, filtering, and sorting to list endpoints'
    }
  ]);

generator.writeToFile();
```

---

## ขั้นตอนที่ 911: Interactive API Playground

```javascript
// routes/playground.js
const express = require('express');
const router = express.Router();

router.get('/', (req, res) => {
  res.send(`
    <!DOCTYPE html>
    <html>
    <head>
      <title>API Playground</title>
      <style>
        body { font-family: Arial, sans-serif; max-width: 900px; margin: 0 auto; padding: 20px; }
        .endpoint { border: 1px solid #ddd; padding: 15px; margin: 10px 0; border-radius: 5px; }
        .method { display: inline-block; padding: 3px 8px; border-radius: 3px; font-weight: bold; }
        .get { background: #61affe; color: white; }
        .post { background: #49cc90; color: white; }
        .put { background: #fca130; color: white; }
        .delete { background: #f93e3e; color: white; }
        textarea { width: 100%; height: 100px; margin: 5px 0; }
        pre { background: #f4f4f4; padding: 10px; overflow-x: auto; }
        button { background: #4a90e2; color: white; border: none; padding: 8px 16px; cursor: pointer; border-radius: 3px; }
      </style>
    </head>
    <body>
      <h1>API Playground</h1>
      <div id="auth">
        <h3>Authentication</h3>
        <input type="text" id="token" placeholder="JWT Token" style="width: 400px; padding: 5px;">
        <button onclick="saveToken()">Save Token</button>
      </div>
      
      <div class="endpoint">
        <span class="method get">GET</span>
        <strong>/api/v2/users</strong>
        <p>ดึงรายชื่อผู้ใช้ทั้งหมด</p>
        <div>
          <input type="number" id="users-page" value="1" placeholder="page" style="width: 60px;">
          <input type="number" id="users-limit" value="20" placeholder="limit" style="width: 60px;">
          <input type="text" id="users-search" placeholder="search">
          <button onclick="callAPI('GET', '/api/v2/users', null, {page: document.getElementById('users-page').value, limit: document.getElementById('users-limit').value, search: document.getElementById('users-search').value}, 'users-result')">Execute</button>
        </div>
        <pre id="users-result">Response will appear here...</pre>
      </div>
      
      <div class="endpoint">
        <span class="method post">POST</span>
        <strong>/api/v2/users</strong>
        <p>สร้างผู้ใช้ใหม่</p>
        <textarea id="create-user-body">{"name": "Test User", "email": "test@example.com", "password": "Password123"}</textarea>
        <button onclick="callAPI('POST', '/api/v2/users', 'create-user-body', null, 'create-user-result')">Execute</button>
        <pre id="create-user-result">Response will appear here...</pre>
      </div>
      
      <script>
        let authToken = '';
        
        function saveToken() {
          authToken = document.getElementById('token').value;
          alert('Token saved!');
        }
        
        async function callAPI(method, path, bodyId, params, resultId) {
          try {
            let url = path;
            if (params) {
              const queryString = Object.entries(params)
                .filter(([k, v]) => v)
                .map(([k, v]) => \`\${k}=\${encodeURIComponent(v)}\`)
                .join('&');
              if (queryString) url += '?' + queryString;
            }
            
            const options = {
              method,
              headers: {
                'Content-Type': 'application/json',
                ...(authToken ? { 'Authorization': 'Bearer ' + authToken } : {})
              }
            };
            
            if (bodyId && document.getElementById(bodyId)) {
              options.body = document.getElementById(bodyId).value;
            }
            
            const res = await fetch(url, options);
            const data = await res.json();
            
            document.getElementById(resultId).textContent = 
              JSON.stringify(data, null, 2);
          } catch (error) {
            document.getElementById(resultId).textContent = 'Error: ' + error.message;
          }
        }
      </script>
    </body>
    </html>
  `);
});

module.exports = router;
```

---

## ขั้นตอนที่ 912: Error Code Documentation

```javascript
// docs/errorCodes.js

/**
 * @swagger
 * components:
 *   schemas:
 *     ErrorCode:
 *       type: string
 *       description: Machine-readable error codes
 *       enum:
 *         - VALIDATION_ERROR
 *         - UNAUTHORIZED
 *         - FORBIDDEN
 *         - NOT_FOUND
 *         - CONFLICT
 *         - RATE_LIMIT_EXCEEDED
 *         - INTERNAL_ERROR
 */

const ERROR_CODES = {
  // 4xx Client errors
  VALIDATION_ERROR: {
    httpStatus: 422,
    description: 'ข้อมูลที่ส่งมาไม่ถูกต้องหรือไม่ครบถ้วน',
    example: {
      success: false,
      errors: [
        { field: 'email', code: 'INVALID_EMAIL', message: 'Invalid email format' }
      ]
    }
  },
  UNAUTHORIZED: {
    httpStatus: 401,
    description: 'ยังไม่ได้ login หรือ token หมดอายุ',
    example: {
      success: false,
      error: { code: 'UNAUTHORIZED', message: 'Authentication required' }
    }
  },
  FORBIDDEN: {
    httpStatus: 403,
    description: 'ไม่มีสิทธิ์เข้าถึง resource นี้',
    example: {
      success: false,
      error: { code: 'FORBIDDEN', message: 'Insufficient permissions' }
    }
  },
  NOT_FOUND: {
    httpStatus: 404,
    description: 'ไม่พบ resource ที่ต้องการ',
    example: {
      success: false,
      error: { code: 'NOT_FOUND', message: 'User not found' }
    }
  },
  CONFLICT: {
    httpStatus: 409,
    description: 'ข้อมูลซ้ำ เช่น email ที่มีอยู่แล้ว',
    example: {
      success: false,
      error: { code: 'CONFLICT', message: 'Email already exists' }
    }
  },
  RATE_LIMIT_EXCEEDED: {
    httpStatus: 429,
    description: 'ส่ง request มากเกินไป',
    example: {
      success: false,
      error: { code: 'RATE_LIMIT_EXCEEDED', message: 'Too many requests' }
    }
  },
  
  // 5xx Server errors
  INTERNAL_ERROR: {
    httpStatus: 500,
    description: 'เกิดข้อผิดพลาดภายใน server',
    example: {
      success: false,
      error: { code: 'INTERNAL_ERROR', message: 'An unexpected error occurred' }
    }
  }
};

// Route สำหรับ error codes documentation
const express = require('express');
const router = express.Router();

router.get('/error-codes', (req, res) => {
  res.json({
    success: true,
    data: Object.entries(ERROR_CODES).map(([code, info]) => ({
      code,
      ...info
    }))
  });
});

module.exports = { ERROR_CODES, router };
```

---

## ขั้นตอนที่ 913: SDK Generation

```javascript
// scripts/generateSDK.js
// สร้าง SDK จาก OpenAPI spec

const { execSync } = require('child_process');
const fs = require('fs');

const generateSDK = (language) => {
  const specs = require('../config/swagger');
  
  // บันทึก spec ลงไฟล์
  fs.writeFileSync('/tmp/openapi.json', JSON.stringify(specs, null, 2));
  
  // ใช้ OpenAPI Generator
  const outputDir = `./sdk/${language}`;
  
  execSync(`
    openapi-generator-cli generate \
      -i /tmp/openapi.json \
      -g ${language} \
      -o ${outputDir} \
      --additional-properties=packageName=my-api-sdk,packageVersion=1.0.0
  `);
  
  console.log(`${language} SDK generated at ${outputDir}`);
};

// สร้าง JavaScript SDK แบบง่าย
const generateJSSDK = (specs) => {
  const endpoints = [];
  
  for (const [path, methods] of Object.entries(specs.paths)) {
    for (const [method, operation] of Object.entries(methods)) {
      endpoints.push({
        path,
        method,
        operationId: operation.operationId,
        summary: operation.summary,
        parameters: operation.parameters || [],
        requestBody: operation.requestBody
      });
    }
  }
  
  let sdk = `// Auto-generated API SDK
// Generated at: ${new Date().toISOString()}

class APIClient {
  constructor(options = {}) {
    this.baseURL = options.baseURL || '${specs.servers[0].url}';
    this.token = options.token || null;
  }
  
  setToken(token) {
    this.token = token;
    return this;
  }
  
  async request(method, path, options = {}) {
    const url = this.baseURL + path;
    const headers = {
      'Content-Type': 'application/json',
      ...options.headers
    };
    
    if (this.token) {
      headers['Authorization'] = 'Bearer ' + this.token;
    }
    
    const response = await fetch(url, {
      method: method.toUpperCase(),
      headers,
      body: options.body ? JSON.stringify(options.body) : undefined
    });
    
    const data = await response.json();
    
    if (!response.ok) {
      throw new APIError(data.error?.message || 'API Error', response.status, data);
    }
    
    return data;
  }
`;

  // สร้าง method สำหรับแต่ละ endpoint
  for (const endpoint of endpoints) {
    if (!endpoint.operationId) continue;
    
    const pathParams = endpoint.parameters.filter(p => p.in === 'path');
    const queryParams = endpoint.parameters.filter(p => p.in === 'query');
    
    const paramNames = pathParams.map(p => p.name);
    if (queryParams.length > 0) paramNames.push('params = {}');
    if (endpoint.requestBody) paramNames.push('data');
    
    let methodPath = `'${endpoint.path.replace(/{(\w+)}/g, '${$1}')}'`;
    
    sdk += `
  /**
   * ${endpoint.summary || endpoint.operationId}
   */
  async ${endpoint.operationId}(${paramNames.join(', ')}) {
    const path = \`${endpoint.path.replace(/{(\w+)}/g, '${$1}')}\`;
    const queryString = ${queryParams.length > 0 ? 'new URLSearchParams(params).toString()' : '""'};
    return this.request('${endpoint.method}', path + (queryString ? '?' + queryString : ''), {
      body: ${endpoint.requestBody ? 'data' : 'undefined'}
    });
  }
`;
  }

  sdk += `}

class APIError extends Error {
  constructor(message, status, data) {
    super(message);
    this.name = 'APIError';
    this.status = status;
    this.data = data;
  }
}

module.exports = APIClient;
`;

  return sdk;
};

module.exports = { generateSDK, generateJSSDK };
```

---

## ขั้นตอนที่ 914: Documentation Versioning

```javascript
// middleware/docsVersioning.js

const docsVersioning = (app) => {
  const swaggerUi = require('swagger-ui-express');
  const path = require('path');
  const fs = require('fs');
  
  // โหลด spec สำหรับแต่ละ version
  const loadVersionSpec = (version) => {
    const specPath = path.join(__dirname, `../docs/specs/v${version}.json`);
    
    if (fs.existsSync(specPath)) {
      return require(specPath);
    }
    
    // Generate on-the-fly
    return require(`../config/swagger.v${version}`);
  };
  
  // Serve Swagger UI สำหรับแต่ละ version
  ['1', '2', '3'].forEach(version => {
    try {
      const spec = loadVersionSpec(version);
      
      app.use(`/docs/v${version}`, swaggerUi.serve, swaggerUi.setup(spec, {
        customSiteTitle: `API v${version} Documentation`
      }));
      
      app.get(`/docs/v${version}/openapi.json`, (req, res) => res.json(spec));
      
      console.log(`Docs for v${version} available at /docs/v${version}`);
    } catch (error) {
      console.log(`No docs for v${version}`);
    }
  });
  
  // Redirect /docs ไปยัง latest version
  app.get('/docs', (req, res) => res.redirect('/docs/v2'));
};

module.exports = docsVersioning;
```

---

## ขั้นตอนที่ 915: Request/Response Examples

```javascript
// docs/examples/users.js

/**
 * @swagger
 * /users:
 *   get:
 *     summary: Get all users
 *     responses:
 *       200:
 *         content:
 *           application/json:
 *             examples:
 *               success:
 *                 summary: Successful response with users
 *                 value:
 *                   success: true
 *                   data:
 *                     - _id: "507f1f77bcf86cd799439011"
 *                       name: "สมชาย ใจดี"
 *                       email: "somchai@example.com"
 *                       role: "user"
 *                       createdAt: "2024-01-01T00:00:00.000Z"
 *                     - _id: "507f1f77bcf86cd799439012"
 *                       name: "สมหญิง ใจงาม"
 *                       email: "somying@example.com"
 *                       role: "admin"
 *                   pagination:
 *                     page: 1
 *                     limit: 20
 *                     total: 2
 *                     pages: 1
 *               empty:
 *                 summary: Empty result
 *                 value:
 *                   success: true
 *                   data: []
 *                   pagination:
 *                     page: 1
 *                     limit: 20
 *                     total: 0
 *                     pages: 0
 */
```

---

## ขั้นตอนที่ 916: Webhook Documentation

```javascript
/**
 * @swagger
 * components:
 *   schemas:
 *     WebhookEvent:
 *       type: object
 *       properties:
 *         id:
 *           type: string
 *           description: Unique event ID
 *         type:
 *           type: string
 *           enum:
 *             - order.created
 *             - order.completed
 *             - payment.succeeded
 *             - payment.failed
 *             - user.registered
 *         timestamp:
 *           type: string
 *           format: date-time
 *         data:
 *           type: object
 *           description: Event-specific data
 * 
 * webhooks:
 *   orderCreated:
 *     post:
 *       summary: Order created webhook
 *       description: |
 *         ส่งไปยัง webhook URL ที่ลงทะเบียนไว้เมื่อมีคำสั่งซื้อใหม่
 *         
 *         ### Signature Verification
 *         ตรวจสอบ HMAC-SHA256 signature จาก header `X-Webhook-Signature`
 *         ```
 *         const signature = crypto.createHmac('sha256', webhookSecret)
 *           .update(JSON.stringify(requestBody))
 *           .digest('hex');
 *         ```
 *       requestBody:
 *         content:
 *           application/json:
 *             schema:
 *               $ref: '#/components/schemas/WebhookEvent'
 *             example:
 *               id: "evt_123456"
 *               type: "order.created"
 *               timestamp: "2024-01-01T10:00:00Z"
 *               data:
 *                 orderId: "ord_789"
 *                 userId: "usr_456"
 *                 total: 1500
 *       responses:
 *         '200':
 *           description: Webhook received successfully
 */
```

---

## ขั้นตอนที่ 917: API Rate Limit Documentation

```javascript
/**
 * @swagger
 * info:
 *   x-rateLimit:
 *     description: |
 *       ## Rate Limiting
 *       
 *       API มีการจำกัดจำนวน requests เพื่อความเสถียรของระบบ
 *       
 *       ### Default Limits
 *       | Endpoint | Limit |
 *       |----------|-------|
 *       | General API | 100 requests / 15 minutes |
 *       | Authentication | 5 attempts / hour |
 *       | File Upload | 20 files / hour |
 *       
 *       ### Response Headers
 *       เมื่อถูก rate limited จะได้รับ HTTP 429 พร้อม headers:
 *       - `X-RateLimit-Limit`: จำนวน request สูงสุด
 *       - `X-RateLimit-Remaining`: จำนวน request ที่เหลือ
 *       - `X-RateLimit-Reset`: เวลา (Unix timestamp) ที่ limit จะ reset
 *       - `Retry-After`: วินาทีที่ต้องรอก่อน retry
 *       
 *       ### Tiers
 *       | Tier | Limit |
 *       |------|-------|
 *       | Free | 100/hour |
 *       | Basic | 1,000/hour |
 *       | Premium | 10,000/hour |
 *       | Enterprise | Unlimited |
 */
```

---

## ขั้นตอนที่ 918: Documentation CI/CD

```yaml
# .github/workflows/docs.yml
name: Generate and Deploy API Documentation

on:
  push:
    branches: [main]
    paths:
      - 'routes/**'
      - 'models/**'
      - 'config/swagger.js'

jobs:
  generate-docs:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'
      
      - name: Install dependencies
        run: npm install
      
      - name: Generate OpenAPI spec
        run: node scripts/generateSpec.js
      
      - name: Validate OpenAPI spec
        run: npx @apidevtools/swagger-cli validate ./docs/openapi.json
      
      - name: Generate SDK
        run: node scripts/generateSDK.js
      
      - name: Deploy to GitHub Pages
        uses: peaceiris/actions-gh-pages@v3
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          publish_dir: ./docs/html
```

```javascript
// scripts/generateSpec.js
const fs = require('fs');
const path = require('path');
const specs = require('../config/swagger');
const yaml = require('yaml');

// บันทึก OpenAPI spec ในรูปแบบ JSON และ YAML
fs.writeFileSync(
  './docs/openapi.json',
  JSON.stringify(specs, null, 2)
);

fs.writeFileSync(
  './docs/openapi.yaml',
  yaml.stringify(specs)
);

// สร้าง HTML documentation ด้วย ReDoc
const htmlContent = `
<!DOCTYPE html>
<html>
  <head>
    <title>My API Documentation</title>
    <meta charset="utf-8"/>
    <meta name="viewport" content="width=device-width, initial-scale=1">
  </head>
  <body>
    <redoc spec-url='./openapi.json'></redoc>
    <script src="https://cdn.jsdelivr.net/npm/redoc/bundles/redoc.standalone.js"></script>
  </body>
</html>
`;

fs.mkdirSync('./docs/html', { recursive: true });
fs.writeFileSync('./docs/html/index.html', htmlContent);
fs.copyFileSync('./docs/openapi.json', './docs/html/openapi.json');

console.log('Documentation generated successfully');
```

---

## ขั้นตอนที่ 919: GraphQL Documentation

```javascript
// docs/graphqlSchema.js
// สำหรับ API ที่ใช้ GraphQL

const { buildSchema } = require('graphql');

/**
 * GraphQL Schema with documentation
 */
const schema = buildSchema(`
  """
  ผู้ใช้งานระบบ
  """
  type User {
    """
    MongoDB ObjectId
    """
    id: ID!
    
    """
    ชื่อผู้ใช้
    """
    name: String!
    
    """
    อีเมลผู้ใช้ (unique)
    """
    email: String!
    
    """
    บทบาท: user, admin, moderator
    """
    role: String!
    
    """
    วันที่สร้างบัญชี
    """
    createdAt: String!
    
    """
    สินค้าที่ชื่นชอบ
    """
    favorites: [Product!]
  }
  
  """
  สินค้า
  """
  type Product {
    id: ID!
    name: String!
    price: Float!
    category: Category!
    stock: Int!
    isActive: Boolean!
  }
  
  """
  หมวดหมู่สินค้า
  """
  type Category {
    id: ID!
    name: String!
    products: [Product!]
  }
  
  type Query {
    """
    ดึงข้อมูลผู้ใช้ตาม ID
    """
    user(id: ID!): User
    
    """
    ดึงรายชื่อผู้ใช้ทั้งหมด
    \`\`\`
    query GetUsers {
      users(limit: 10, offset: 0) {
        id
        name
        email
      }
    }
    \`\`\`
    """
    users(limit: Int, offset: Int, search: String): [User!]!
    
    """
    ดึงสินค้าตาม ID
    """
    product(id: ID!): Product
    
    """
    ดึงรายการสินค้า
    """
    products(limit: Int, offset: Int, categoryId: ID): [Product!]!
  }
  
  type Mutation {
    """
    สร้างผู้ใช้ใหม่
    """
    createUser(input: CreateUserInput!): User!
    
    """
    อัพเดทข้อมูลผู้ใช้
    """
    updateUser(id: ID!, input: UpdateUserInput!): User!
  }
  
  input CreateUserInput {
    name: String!
    email: String!
    password: String!
  }
  
  input UpdateUserInput {
    name: String
    email: String
  }
`);

module.exports = schema;
```

---

## ขั้นตอนที่ 920: Complete Documentation Setup

```javascript
// setup/documentation.js

const setupDocumentation = (app) => {
  const express = require('express');
  const swaggerUi = require('swagger-ui-express');
  const specs = require('../config/swagger');
  const { changelogRouter } = require('../config/changelog');
  const { router: errorCodesRouter } = require('../docs/errorCodes');
  const migrationRouter = require('../utils/migrationGuide');
  const playgroundRouter = require('../routes/playground');
  
  // Swagger UI
  app.use('/docs', swaggerUi.serve, swaggerUi.setup(specs, {
    customSiteTitle: 'My API Documentation',
    swaggerOptions: {
      persistAuthorization: true,
      displayRequestDuration: true,
      filter: true
    }
  }));
  
  // Raw OpenAPI spec
  app.get('/openapi.json', (req, res) => res.json(specs));
  app.get('/openapi.yaml', (req, res) => {
    const yaml = require('yaml');
    res.type('yaml').send(yaml.stringify(specs));
  });
  
  // Additional documentation
  app.use('/docs/changelog', changelogRouter);
  app.use('/docs/errors', errorCodesRouter);
  app.use('/docs/migration', migrationRouter);
  
  // Interactive playground
  app.use('/playground', playgroundRouter);
  
  // Documentation index
  app.get('/docs', (req, res) => {
    res.json({
      documentation: {
        swagger: '/docs',
        openapi_json: '/openapi.json',
        openapi_yaml: '/openapi.yaml',
        changelog: '/docs/changelog',
        error_codes: '/docs/errors/error-codes',
        migration: '/docs/migration',
        playground: '/playground'
      }
    });
  });
  
  console.log('Documentation available at /docs');
};

module.exports = setupDocumentation;
```

---

## แบบฝึกหัดท้ายบท

### แบบฝึกหัดที่ 1: Document Existing API
เพิ่ม Swagger/JSDoc annotations ให้ครบทุก endpoints ใน project ของคุณ

### แบบฝึกหัดที่ 2: Postman Collection
สร้าง Postman collection ที่มี tests สำหรับทุก endpoints รวมถึง auth flow

### แบบฝึกหัดที่ 3: SDK Generator
สร้าง script ที่ generate JavaScript SDK จาก OpenAPI spec

### แบบฝึกหัดที่ 4: Documentation Testing
เขียน tests ที่ตรวจสอบว่า API behavior ตรงกับ documentation

### แบบฝึกหัดที่ 5: Changelog Automation
สร้าง script ที่ generate changelog อัตโนมัติจาก git commits

---

## สรุป

Documentation ที่ดีเป็นส่วนสำคัญของการพัฒนา API ที่ยั่งยืน Swagger/OpenAPI เป็นมาตรฐานที่นิยมใช้ในอุตสาหกรรม การ automate generation และ testing ของ documentation ช่วยให้มั่นใจได้ว่า documentation ถูกต้องและทันสมัยอยู่เสมอ
