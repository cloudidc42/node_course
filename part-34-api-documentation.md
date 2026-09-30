# Part 34: API Documentation

> ขั้นตอนที่ 34-34 จาก 1000

---

## สารบัญ

1. [ทำไมต้องมี API Documentation](#ทำไมตองมี-api-documentation)
2. [Swagger/OpenAPI Specification](#swaggeropenapi-specification)
3. [swagger-jsdoc](#swagger-jsdoc)
4. [swagger-ui-express](#swagger-ui-express)
5. [Postman Collections](#postman-collections)
6. [API Blueprint](#api-blueprint)
7. [Workshop: Complete API Docs](#workshop)
8. [แบบฝึกหัด](#แบบฝึกหัด)

---

## ทำไมต้องมี API Documentation

API Documentation ที่ดีทำให้:
- **นักพัฒนาอื่น** เข้าใจ API ได้เร็ว
- **Frontend team** รู้ว่า request/response format เป็นอะไร
- **ทดสอบ** API ได้โดยตรงจาก browser
- **Auto-generate** client SDK ได้
- **Contract** ระหว่าง teams ชัดเจน

---

## Swagger/OpenAPI Specification

OpenAPI (เดิมชื่อ Swagger) เป็น standard สำหรับ describe REST APIs

### โครงสร้าง OpenAPI Document

```yaml
# openapi.yaml
openapi: "3.0.3"
info:
  title: Blog API
  version: "1.0.0"
  description: |
    REST API สำหรับ Blog Platform
    
    ## Authentication
    ใช้ Bearer token ใน Authorization header:
    ```
    Authorization: Bearer <your-jwt-token>
    ```
  contact:
    name: API Support
    email: support@example.com
  license:
    name: MIT
    url: https://opensource.org/licenses/MIT

servers:
  - url: http://localhost:3000/api/v1
    description: Development server
  - url: https://api.example.com/v1
    description: Production server

tags:
  - name: Authentication
    description: Register, Login, Logout
  - name: Posts
    description: Blog post operations
  - name: Users
    description: User management

paths:
  /auth/register:
    post:
      tags: [Authentication]
      summary: Register a new user
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/RegisterRequest'
            example:
              name: "John Doe"
              email: "john@example.com"
              password: "SecurePass123!"
      responses:
        '201':
          description: User registered successfully
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/AuthResponse'
        '400':
          $ref: '#/components/responses/ValidationError'
        '409':
          $ref: '#/components/responses/ConflictError'

  /auth/login:
    post:
      tags: [Authentication]
      summary: Login with email and password
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/LoginRequest'
      responses:
        '200':
          description: Login successful
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/AuthResponse'
        '401':
          $ref: '#/components/responses/UnauthorizedError'

  /posts:
    get:
      tags: [Posts]
      summary: Get all published posts
      parameters:
        - $ref: '#/components/parameters/PageParam'
        - $ref: '#/components/parameters/LimitParam'
        - name: tag
          in: query
          description: Filter by tag slug
          schema:
            type: string
          example: "nodejs"
        - name: search
          in: query
          description: Search in title and content
          schema:
            type: string
        - name: sort
          in: query
          description: Sort field (prefix with - for descending)
          schema:
            type: string
            enum: [publishedAt, -publishedAt, title, -title]
          example: "-publishedAt"
      responses:
        '200':
          description: List of posts
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/PostListResponse'
    
    post:
      tags: [Posts]
      summary: Create a new post
      security:
        - BearerAuth: []
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/CreatePostRequest'
      responses:
        '201':
          description: Post created
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/PostResponse'
        '401':
          $ref: '#/components/responses/UnauthorizedError'
        '400':
          $ref: '#/components/responses/ValidationError'

  /posts/{id}:
    parameters:
      - name: id
        in: path
        required: true
        description: Post ID
        schema:
          type: string
          example: "507f1f77bcf86cd799439011"
    
    get:
      tags: [Posts]
      summary: Get a single post
      responses:
        '200':
          description: Post details
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/PostResponse'
        '404':
          $ref: '#/components/responses/NotFoundError'
    
    put:
      tags: [Posts]
      summary: Update a post
      security:
        - BearerAuth: []
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/UpdatePostRequest'
      responses:
        '200':
          description: Post updated
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/PostResponse'
        '401':
          $ref: '#/components/responses/UnauthorizedError'
        '403':
          $ref: '#/components/responses/ForbiddenError'
        '404':
          $ref: '#/components/responses/NotFoundError'
    
    delete:
      tags: [Posts]
      summary: Delete a post
      security:
        - BearerAuth: []
      responses:
        '204':
          description: Post deleted successfully
        '401':
          $ref: '#/components/responses/UnauthorizedError'
        '404':
          $ref: '#/components/responses/NotFoundError'

components:
  securitySchemes:
    BearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT
      description: JWT token obtained from /auth/login

  parameters:
    PageParam:
      name: page
      in: query
      description: Page number
      schema:
        type: integer
        minimum: 1
        default: 1
    LimitParam:
      name: limit
      in: query
      description: Items per page
      schema:
        type: integer
        minimum: 1
        maximum: 100
        default: 10

  schemas:
    RegisterRequest:
      type: object
      required: [name, email, password]
      properties:
        name:
          type: string
          minLength: 2
          maxLength: 100
          example: "John Doe"
        email:
          type: string
          format: email
          example: "john@example.com"
        password:
          type: string
          minLength: 8
          format: password
          example: "SecurePass123!"
    
    LoginRequest:
      type: object
      required: [email, password]
      properties:
        email:
          type: string
          format: email
        password:
          type: string
          format: password
    
    AuthResponse:
      type: object
      properties:
        status:
          type: string
          example: "success"
        message:
          type: string
          example: "Login successful"
        data:
          type: object
          properties:
            user:
              $ref: '#/components/schemas/User'
            accessToken:
              type: string
              example: "eyJhbGciOiJIUzI1NiIsInR5..."
            refreshToken:
              type: string
    
    User:
      type: object
      properties:
        id:
          type: string
          example: "507f1f77bcf86cd799439011"
        name:
          type: string
          example: "John Doe"
        email:
          type: string
          format: email
        role:
          type: string
          enum: [user, admin, moderator]
        avatar:
          type: string
          format: uri
          nullable: true
        createdAt:
          type: string
          format: date-time
    
    Post:
      type: object
      properties:
        id:
          type: string
        title:
          type: string
        slug:
          type: string
        excerpt:
          type: string
          nullable: true
        content:
          type: string
        author:
          $ref: '#/components/schemas/User'
        tags:
          type: array
          items:
            $ref: '#/components/schemas/Tag'
        status:
          type: string
          enum: [draft, published, archived]
        viewCount:
          type: integer
        publishedAt:
          type: string
          format: date-time
          nullable: true
        createdAt:
          type: string
          format: date-time
    
    Tag:
      type: object
      properties:
        id:
          type: string
        name:
          type: string
        slug:
          type: string
    
    CreatePostRequest:
      type: object
      required: [title, content]
      properties:
        title:
          type: string
          minLength: 5
          maxLength: 200
        content:
          type: string
          minLength: 50
        excerpt:
          type: string
          maxLength: 500
        tags:
          type: array
          items:
            type: string
          example: ["nodejs", "express"]
        status:
          type: string
          enum: [draft, published]
          default: draft
        featuredImage:
          type: string
          format: uri
    
    UpdatePostRequest:
      type: object
      properties:
        title:
          type: string
        content:
          type: string
        excerpt:
          type: string
        tags:
          type: array
          items:
            type: string
    
    PostResponse:
      type: object
      properties:
        status:
          type: string
          example: "success"
        data:
          $ref: '#/components/schemas/Post'
    
    PostListResponse:
      type: object
      properties:
        status:
          type: string
        data:
          type: array
          items:
            $ref: '#/components/schemas/Post'
        meta:
          type: object
          properties:
            pagination:
              $ref: '#/components/schemas/Pagination'
    
    Pagination:
      type: object
      properties:
        total:
          type: integer
        page:
          type: integer
        perPage:
          type: integer
        totalPages:
          type: integer
        hasNextPage:
          type: boolean
        hasPrevPage:
          type: boolean
    
    Error:
      type: object
      properties:
        status:
          type: string
          example: "error"
        code:
          type: string
          example: "NOT_FOUND"
        message:
          type: string
    
    ValidationError:
      allOf:
        - $ref: '#/components/schemas/Error'
        - type: object
          properties:
            errors:
              type: array
              items:
                type: object
                properties:
                  field:
                    type: string
                  message:
                    type: string

  responses:
    ValidationError:
      description: Validation failed
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/ValidationError'
          example:
            status: "error"
            code: "VALIDATION_ERROR"
            message: "Validation failed"
            errors:
              - field: "email"
                message: "Must be a valid email"
    
    UnauthorizedError:
      description: Authentication required
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/Error'
          example:
            status: "error"
            code: "UNAUTHORIZED"
            message: "Authentication required"
    
    ForbiddenError:
      description: Permission denied
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/Error'
    
    NotFoundError:
      description: Resource not found
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/Error'
    
    ConflictError:
      description: Resource conflict
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/Error'
```

---

## swagger-jsdoc

swagger-jsdoc ช่วยสร้าง OpenAPI spec จาก JSDoc comments ในโค้ด

### ติดตั้ง

```bash
npm install swagger-jsdoc swagger-ui-express
```

### Configuration

```javascript
// config/swagger.js
const swaggerJsdoc = require('swagger-jsdoc');

const options = {
  definition: {
    openapi: '3.0.3',
    info: {
      title: 'Blog API',
      version: '1.0.0',
      description: 'REST API สำหรับ Blog Platform',
      contact: {
        name: 'API Support',
        email: 'api@example.com',
      },
    },
    servers: [
      {
        url: process.env.API_URL || 'http://localhost:3000/api/v1',
        description: 'Current server',
      },
    ],
    components: {
      securitySchemes: {
        BearerAuth: {
          type: 'http',
          scheme: 'bearer',
          bearerFormat: 'JWT',
        },
      },
    },
    security: [], // Default: no auth required
  },
  apis: [
    './src/routes/*.js',     // Routes files
    './src/models/*.js',     // Models (สำหรับ schemas)
    './src/docs/*.yaml',     // Separate YAML files
  ],
};

const swaggerSpec = swaggerJsdoc(options);
module.exports = swaggerSpec;
```

### JSDoc Comments ใน Routes

```javascript
// routes/posts.js
const express = require('express');
const router = express.Router();
const postController = require('../controllers/postController');
const { authenticate } = require('../middleware/auth');

/**
 * @swagger
 * tags:
 *   name: Posts
 *   description: Blog post management
 */

/**
 * @swagger
 * components:
 *   schemas:
 *     Post:
 *       type: object
 *       properties:
 *         id:
 *           type: string
 *           description: Post ID
 *           example: "507f1f77bcf86cd799439011"
 *         title:
 *           type: string
 *           example: "Introduction to Node.js"
 *         slug:
 *           type: string
 *           example: "introduction-to-nodejs"
 *         content:
 *           type: string
 *         author:
 *           $ref: '#/components/schemas/User'
 *         status:
 *           type: string
 *           enum: [draft, published, archived]
 *         createdAt:
 *           type: string
 *           format: date-time
 *     
 *     CreatePost:
 *       type: object
 *       required:
 *         - title
 *         - content
 *       properties:
 *         title:
 *           type: string
 *           minLength: 5
 *           maxLength: 200
 *         content:
 *           type: string
 *           minLength: 50
 *         excerpt:
 *           type: string
 *         tags:
 *           type: array
 *           items:
 *             type: string
 *         status:
 *           type: string
 *           enum: [draft, published]
 *           default: draft
 */

/**
 * @swagger
 * /posts:
 *   get:
 *     summary: Get all published posts
 *     tags: [Posts]
 *     parameters:
 *       - in: query
 *         name: page
 *         schema:
 *           type: integer
 *           default: 1
 *         description: Page number
 *       - in: query
 *         name: limit
 *         schema:
 *           type: integer
 *           default: 10
 *           maximum: 100
 *         description: Items per page
 *       - in: query
 *         name: tag
 *         schema:
 *           type: string
 *         description: Filter by tag
 *       - in: query
 *         name: search
 *         schema:
 *           type: string
 *         description: Search term
 *     responses:
 *       200:
 *         description: List of posts
 *         content:
 *           application/json:
 *             schema:
 *               type: object
 *               properties:
 *                 status:
 *                   type: string
 *                   example: success
 *                 data:
 *                   type: array
 *                   items:
 *                     $ref: '#/components/schemas/Post'
 *                 meta:
 *                   type: object
 *                   properties:
 *                     pagination:
 *                       type: object
 */
router.get('/', postController.getPosts);

/**
 * @swagger
 * /posts/{id}:
 *   get:
 *     summary: Get a post by ID
 *     tags: [Posts]
 *     parameters:
 *       - in: path
 *         name: id
 *         required: true
 *         schema:
 *           type: string
 *         description: Post ID
 *     responses:
 *       200:
 *         description: Post details
 *         content:
 *           application/json:
 *             schema:
 *               type: object
 *               properties:
 *                 status:
 *                   type: string
 *                 data:
 *                   $ref: '#/components/schemas/Post'
 *       404:
 *         description: Post not found
 *         content:
 *           application/json:
 *             schema:
 *               $ref: '#/components/schemas/Error'
 */
router.get('/:id', postController.getPost);

/**
 * @swagger
 * /posts:
 *   post:
 *     summary: Create a new post
 *     tags: [Posts]
 *     security:
 *       - BearerAuth: []
 *     requestBody:
 *       required: true
 *       content:
 *         application/json:
 *           schema:
 *             $ref: '#/components/schemas/CreatePost'
 *           example:
 *             title: "My First Blog Post"
 *             content: "This is the content of my blog post..."
 *             tags: ["nodejs", "express"]
 *             status: "draft"
 *     responses:
 *       201:
 *         description: Post created successfully
 *         content:
 *           application/json:
 *             schema:
 *               type: object
 *               properties:
 *                 status:
 *                   type: string
 *                 message:
 *                   type: string
 *                 data:
 *                   $ref: '#/components/schemas/Post'
 *       400:
 *         description: Validation error
 *       401:
 *         description: Unauthorized
 */
router.post('/', authenticate, postController.createPost);

module.exports = router;
```

---

## swagger-ui-express

### Setup Swagger UI

```javascript
// app.js
const express = require('express');
const swaggerUi = require('swagger-ui-express');
const swaggerSpec = require('./config/swagger');

const app = express();

// Swagger UI
app.use('/api-docs', swaggerUi.serve, swaggerUi.setup(swaggerSpec, {
  customSiteTitle: 'Blog API Documentation',
  customCss: `
    .swagger-ui .topbar { display: none }
    .swagger-ui .info { margin: 30px 0 }
  `,
  customfavIcon: '/favicon.ico',
  swaggerOptions: {
    persistAuthorization: true,  // จำ auth token ไว้
    displayRequestDuration: true,
    filter: true,
    defaultModelsExpandDepth: 2,
    defaultModelExpandDepth: 2,
    tryItOutEnabled: true,
  },
}));

// Serve raw swagger JSON
app.get('/api-docs.json', (req, res) => {
  res.setHeader('Content-Type', 'application/json');
  res.send(swaggerSpec);
});

// Serve raw swagger YAML  
app.get('/api-docs.yaml', (req, res) => {
  const yaml = require('js-yaml');
  res.setHeader('Content-Type', 'text/yaml');
  res.send(yaml.dump(swaggerSpec));
});

console.log('📚 API Docs: http://localhost:3000/api-docs');
```

### Swagger UI Options ขั้นสูง

```javascript
// config/swaggerUiOptions.js
const swaggerUiOptions = {
  customSiteTitle: 'My API Docs',
  
  // Custom CSS
  customCss: `
    .swagger-ui .topbar { background-color: #1a1a2e }
    .swagger-ui .topbar .download-url-wrapper { display: none }
    .swagger-ui .info .title { color: #16213e }
    .swagger-ui .scheme-container { background: #f8f9fa; padding: 15px }
  `,
  
  swaggerOptions: {
    // URL ที่จะโหลด spec จาก
    url: '/api-docs.json',
    
    // หรือ URLs หลายอัน
    urls: [
      { url: '/api-docs/v1.json', name: 'API v1' },
      { url: '/api-docs/v2.json', name: 'API v2' },
    ],
    
    // การแสดงผล
    docExpansion: 'list',        // none, list, full
    defaultModelsExpandDepth: 3,
    displayRequestDuration: true,
    filter: true,                // เพิ่ม search box
    showExtensions: true,
    showCommonExtensions: true,
    
    // Authorization
    persistAuthorization: true,
    
    // การทดสอบ
    tryItOutEnabled: true,
    
    // Request interceptor (เพิ่ม custom headers)
    requestInterceptor: `(request) => {
      request.headers['X-Custom-Header'] = 'value';
      return request;
    }`,
  },
};

module.exports = swaggerUiOptions;
```

---

## Postman Collections

### สร้าง Postman Collection

```json
{
  "info": {
    "name": "Blog API",
    "description": "Blog API Collection",
    "schema": "https://schema.getpostman.com/json/collection/v2.1.0/collection.json",
    "version": "1.0.0"
  },
  "variable": [
    {
      "key": "baseUrl",
      "value": "http://localhost:3000/api/v1",
      "type": "string"
    },
    {
      "key": "accessToken",
      "value": "",
      "type": "string"
    }
  ],
  "auth": {
    "type": "bearer",
    "bearer": [
      {
        "key": "token",
        "value": "{{accessToken}}",
        "type": "string"
      }
    ]
  },
  "item": [
    {
      "name": "Auth",
      "item": [
        {
          "name": "Register",
          "request": {
            "method": "POST",
            "url": "{{baseUrl}}/auth/register",
            "header": [
              {
                "key": "Content-Type",
                "value": "application/json"
              }
            ],
            "body": {
              "mode": "raw",
              "raw": "{\n  \"name\": \"Test User\",\n  \"email\": \"test@example.com\",\n  \"password\": \"SecurePass123!\"\n}"
            }
          },
          "event": [
            {
              "listen": "test",
              "script": {
                "exec": [
                  "pm.test('Status code is 201', () => {",
                  "  pm.response.to.have.status(201);",
                  "});",
                  "",
                  "const response = pm.response.json();",
                  "pm.test('Has access token', () => {",
                  "  pm.expect(response.data.accessToken).to.exist;",
                  "});",
                  "",
                  "// Save token for other requests",
                  "if (response.data?.accessToken) {",
                  "  pm.collectionVariables.set('accessToken', response.data.accessToken);",
                  "}"
                ]
              }
            }
          ]
        },
        {
          "name": "Login",
          "request": {
            "method": "POST",
            "url": "{{baseUrl}}/auth/login",
            "header": [
              {"key": "Content-Type", "value": "application/json"}
            ],
            "body": {
              "mode": "raw",
              "raw": "{\n  \"email\": \"test@example.com\",\n  \"password\": \"SecurePass123!\"\n}"
            }
          },
          "event": [
            {
              "listen": "test",
              "script": {
                "exec": [
                  "pm.test('Status 200', () => pm.response.to.have.status(200));",
                  "const res = pm.response.json();",
                  "pm.collectionVariables.set('accessToken', res.data.accessToken);"
                ]
              }
            }
          ]
        }
      ]
    },
    {
      "name": "Posts",
      "item": [
        {
          "name": "Get All Posts",
          "request": {
            "method": "GET",
            "url": {
              "raw": "{{baseUrl}}/posts?page=1&limit=10",
              "query": [
                {"key": "page", "value": "1"},
                {"key": "limit", "value": "10"},
                {"key": "tag", "value": "nodejs", "disabled": true},
                {"key": "search", "value": "", "disabled": true}
              ]
            }
          },
          "event": [
            {
              "listen": "test",
              "script": {
                "exec": [
                  "pm.test('Status 200', () => pm.response.to.have.status(200));",
                  "pm.test('Has pagination', () => {",
                  "  const body = pm.response.json();",
                  "  pm.expect(body.meta.pagination).to.exist;",
                  "});"
                ]
              }
            }
          ]
        },
        {
          "name": "Create Post",
          "request": {
            "method": "POST",
            "url": "{{baseUrl}}/posts",
            "auth": {"type": "bearer", "bearer": [{"key": "token", "value": "{{accessToken}}"}]},
            "header": [{"key": "Content-Type", "value": "application/json"}],
            "body": {
              "mode": "raw",
              "raw": "{\n  \"title\": \"Test Post\",\n  \"content\": \"This is a test post content that is long enough\",\n  \"tags\": [\"test\", \"nodejs\"],\n  \"status\": \"draft\"\n}"
            }
          }
        }
      ]
    }
  ]
}
```

### Postman Environments

```json
{
  "name": "Development",
  "values": [
    {"key": "baseUrl", "value": "http://localhost:3000/api/v1", "enabled": true},
    {"key": "accessToken", "value": "", "enabled": true},
    {"key": "testEmail", "value": "dev@example.com", "enabled": true}
  ]
}
```

### Newman (Postman CLI)

```bash
# ติดตั้ง Newman
npm install -g newman

# รัน collection
newman run blog-api.postman_collection.json \
  --environment development.postman_environment.json \
  --reporters cli,html \
  --reporter-html-export report.html

# รันใน CI/CD
newman run blog-api.postman_collection.json \
  --environment production.postman_environment.json \
  --bail         # หยุดถ้า test fail
  --color off    # ปิด color สำหรับ CI logs
```

---

## API Blueprint

API Blueprint เป็น Markdown-based format สำหรับ describe APIs

```markdown
FORMAT: 1A
HOST: http://localhost:3000/api/v1

# Blog API

REST API สำหรับ Blog Platform

## Authentication [/auth]

### Register [POST /auth/register]

+ Request (application/json)
    + Attributes
        + name: `John Doe` (string, required)
        + email: `john@example.com` (string, required)
        + password: `SecurePass123!` (string, required)
    
    + Body
        {
            "name": "John Doe",
            "email": "john@example.com",
            "password": "SecurePass123!"
        }

+ Response 201 (application/json)
    + Body
        {
            "status": "success",
            "message": "User registered successfully",
            "data": {
                "user": {
                    "id": "507f1f77bcf86cd799439011",
                    "name": "John Doe",
                    "email": "john@example.com"
                },
                "accessToken": "eyJhbGciOiJIUzI1NiIsInR5..."
            }
        }

+ Response 400 (application/json)
    + Body
        {
            "status": "error",
            "code": "VALIDATION_ERROR",
            "message": "Validation failed",
            "errors": [
                {"field": "email", "message": "Must be a valid email"}
            ]
        }

## Posts Collection [/posts]

### List All Posts [GET /posts{?page,limit,tag,search}]

+ Parameters
    + page: 1 (number, optional) - Page number
    + limit: 10 (number, optional) - Items per page
    + tag: `nodejs` (string, optional) - Filter by tag
    + search: `tutorial` (string, optional) - Search term

+ Response 200 (application/json)
    + Body
        {
            "status": "success",
            "data": [],
            "meta": {
                "pagination": {
                    "total": 50,
                    "page": 1,
                    "perPage": 10,
                    "totalPages": 5
                }
            }
        }
```

---

## Workshop: Complete API Docs

```javascript
// ตัวอย่าง complete setup ทั้งหมด

// 1. package.json
{
  "scripts": {
    "docs:generate": "node scripts/generate-docs.js",
    "docs:serve": "swagger-ui-express src/docs/openapi.json",
    "docs:validate": "swagger-cli validate src/docs/openapi.json"
  }
}

// 2. scripts/generate-docs.js
const swaggerJsdoc = require('swagger-jsdoc');
const fs = require('fs');
const path = require('path');

const swaggerSpec = swaggerJsdoc({
  definition: {
    openapi: '3.0.3',
    info: { title: 'Blog API', version: '1.0.0' },
  },
  apis: ['./src/routes/*.js', './src/models/*.js'],
});

// บันทึก spec เป็นไฟล์
fs.writeFileSync(
  path.join(__dirname, '../src/docs/openapi.json'),
  JSON.stringify(swaggerSpec, null, 2)
);

console.log('✅ API docs generated');
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Swagger Setup

ติดตั้งและ configure swagger-jsdoc + swagger-ui-express สำหรับ Todo API:
- Document endpoints: GET/POST/PUT/DELETE /todos
- Schema: Todo (id, title, completed, userId, createdAt)
- Authentication: Bearer JWT

### แบบฝึกหัดที่ 2: Complete OpenAPI Spec

สร้าง openapi.yaml ครบสมบูรณ์สำหรับ E-commerce API:
- Products (CRUD)
- Orders (create, list, update status)
- Reviews (add, list)
- Components: reusable schemas, responses, parameters

### แบบฝึกหัดที่ 3: Postman Collection

สร้าง Postman Collection:
- Test scripts ทุก endpoint
- Variables: baseUrl, token
- Environment: development, production
- Newman สำหรับ CI/CD

---

## สรุป

API Documentation เป็นส่วนสำคัญของ API ที่ดี

| เครื่องมือ | ใช้สำหรับ |
|-----------|---------|
| OpenAPI/Swagger | Industry standard spec |
| swagger-jsdoc | Generate spec จาก comments |
| swagger-ui-express | Interactive browser UI |
| Postman | Manual + automated testing |
| Newman | CLI testing ใน CI/CD |
| API Blueprint | Markdown-based docs |

**ถัดไป**: [Part 35: Testing with Jest →](./part-35-testing-jest.md)
