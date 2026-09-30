# Part 56 | ขั้นตอนที่ 981-1000 จาก 1000

## Clean Architecture - สถาปัตยกรรมที่สะอาดและยืดหยุ่น

Clean Architecture เป็น architectural pattern ที่แบ่งระบบออกเป็น layers โดยมี dependency rule ที่ชัดเจน: dependencies ชี้เข้าสู่ศูนย์กลาง (inner layers) เท่านั้น

---

## ขั้นตอนที่ 981: ทำความเข้าใจ Clean Architecture Layers

```
                  ┌──────────────────────────────────┐
                  │       Frameworks & Drivers        │
                  │   (Express, MongoDB, Redis, etc.) │
                  │  ┌────────────────────────────┐   │
                  │  │    Interface Adapters      │   │
                  │  │ (Controllers, Gateways,    │   │
                  │  │   Presenters, Repositories)│   │
                  │  │  ┌──────────────────────┐  │   │
                  │  │  │  Application Layer   │  │   │
                  │  │  │  (Use Cases)         │  │   │
                  │  │  │  ┌────────────────┐  │  │   │
                  │  │  │  │  Enterprise    │  │  │   │
                  │  │  │  │  Business      │  │  │   │
                  │  │  │  │  Rules         │  │  │   │
                  │  │  │  │  (Entities)    │  │  │   │
                  │  │  │  └────────────────┘  │  │   │
                  │  │  └──────────────────────┘  │   │
                  │  └────────────────────────────┘   │
                  └──────────────────────────────────┘

  Dependency Rule: ทุก layer depend เฉพาะ inner layers
```

**4 Layers หลัก:**
1. **Entities** - Enterprise business rules (innermost)
2. **Use Cases** - Application business rules
3. **Interface Adapters** - Controllers, Presenters, Gateways
4. **Frameworks & Drivers** - External frameworks (outermost)

---

## ขั้นตอนที่ 982: Entities Layer

```javascript
// src/entities/User.js

class User {
  constructor({ id, email, name, role = 'user', hashedPassword }) {
    this._id = id;
    this._email = email;
    this._name = name;
    this._role = role;
    this._hashedPassword = hashedPassword;
    this._createdAt = new Date();
    this._isActive = true;
  }

  // Getters
  get id() { return this._id; }
  get email() { return this._email; }
  get name() { return this._name; }
  get role() { return this._role; }
  get isActive() { return this._isActive; }

  // Enterprise Business Rules
  isAdmin() { return this._role === 'admin'; }
  canManageUsers() { return ['admin', 'moderator'].includes(this._role); }
  canPublishContent() { return this._isActive && this._role !== 'banned'; }

  deactivate() {
    if (!this._isActive) throw new Error('User already deactivated');
    this._isActive = false;
    return this;
  }

  verifyPassword(password, compareFunction) {
    return compareFunction(password, this._hashedPassword);
  }

  toJSON() {
    return {
      id: this._id,
      email: this._email,
      name: this._name,
      role: this._role,
      isActive: this._isActive,
      createdAt: this._createdAt
    };
  }
}

// Error types ที่เป็น part ของ domain
class EntityValidationError extends Error {
  constructor(message, field) {
    super(message);
    this.name = 'EntityValidationError';
    this.field = field;
  }
}

module.exports = { User, EntityValidationError };
```

```javascript
// src/entities/Post.js

class Post {
  constructor({ id, title, content, authorId, status = 'draft', tags = [] }) {
    this._validate({ title, content, authorId });
    
    this._id = id;
    this._title = title;
    this._content = content;
    this._authorId = authorId;
    this._status = status;
    this._tags = tags;
    this._publishedAt = null;
    this._createdAt = new Date();
  }

  // Getters
  get id() { return this._id; }
  get title() { return this._title; }
  get status() { return this._status; }
  get authorId() { return this._authorId; }

  // Enterprise Business Rules
  publish() {
    if (this._status === 'published') {
      throw new Error('Post is already published');
    }
    if (this._content.length < 100) {
      throw new Error('Content too short to publish');
    }
    
    this._status = 'published';
    this._publishedAt = new Date();
    return this;
  }

  archive() {
    this._status = 'archived';
    return this;
  }

  isOwnedBy(userId) {
    return this._authorId === userId;
  }

  _validate({ title, content, authorId }) {
    if (!title || title.length < 3) {
      throw new Error('Title must be at least 3 characters');
    }
    if (!content) throw new Error('Content is required');
    if (!authorId) throw new Error('Author is required');
  }

  toJSON() {
    return {
      id: this._id,
      title: this._title,
      content: this._content,
      authorId: this._authorId,
      status: this._status,
      tags: this._tags,
      publishedAt: this._publishedAt,
      createdAt: this._createdAt
    };
  }
}

module.exports = Post;
```

---

## ขั้นตอนที่ 983: Repository Interfaces

```javascript
// src/useCases/ports/IUserRepository.js
// Port (Interface) - อยู่ใน Application layer

class IUserRepository {
  async findById(id) { throw new Error('Not implemented'); }
  async findByEmail(email) { throw new Error('Not implemented'); }
  async save(user) { throw new Error('Not implemented'); }
  async update(id, data) { throw new Error('Not implemented'); }
  async delete(id) { throw new Error('Not implemented'); }
  async findAll(filters, pagination) { throw new Error('Not implemented'); }
  async countByFilters(filters) { throw new Error('Not implemented'); }
}

// src/useCases/ports/IPostRepository.js
class IPostRepository {
  async findById(id) { throw new Error('Not implemented'); }
  async findByAuthor(authorId, options) { throw new Error('Not implemented'); }
  async findPublished(options) { throw new Error('Not implemented'); }
  async save(post) { throw new Error('Not implemented'); }
  async update(id, data) { throw new Error('Not implemented'); }
  async delete(id) { throw new Error('Not implemented'); }
}

// src/useCases/ports/IEmailService.js
class IEmailService {
  async sendWelcomeEmail(user) { throw new Error('Not implemented'); }
  async sendPasswordResetEmail(user, token) { throw new Error('Not implemented'); }
  async sendNotification(userId, message) { throw new Error('Not implemented'); }
}

// src/useCases/ports/IPasswordHasher.js
class IPasswordHasher {
  async hash(password) { throw new Error('Not implemented'); }
  async compare(password, hash) { throw new Error('Not implemented'); }
}

module.exports = { IUserRepository, IPostRepository, IEmailService, IPasswordHasher };
```

---

## ขั้นตอนที่ 984: Use Cases

```javascript
// src/useCases/user/RegisterUser.js

class RegisterUser {
  constructor({ userRepository, emailService, passwordHasher }) {
    this.userRepository = userRepository;
    this.emailService = emailService;
    this.passwordHasher = passwordHasher;
  }

  async execute({ email, name, password }) {
    // ตรวจสอบ input
    this._validateInput({ email, name, password });
    
    // ตรวจสอบว่า email ซ้ำ
    const existingUser = await this.userRepository.findByEmail(email);
    if (existingUser) {
      throw new Error('Email already registered');
    }
    
    // Hash password
    const hashedPassword = await this.passwordHasher.hash(password);
    
    // สร้าง User entity
    const { User } = require('../../entities/User');
    const user = new User({
      id: require('crypto').randomUUID(),
      email,
      name,
      hashedPassword
    });
    
    // บันทึก
    await this.userRepository.save(user);
    
    // ส่ง welcome email (fire and forget)
    this.emailService.sendWelcomeEmail(user).catch(err => {
      console.error('Failed to send welcome email:', err);
    });
    
    return user;
  }

  _validateInput({ email, name, password }) {
    const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
    
    if (!email || !emailRegex.test(email)) {
      throw new Error('Invalid email address');
    }
    
    if (!name || name.length < 2) {
      throw new Error('Name must be at least 2 characters');
    }
    
    if (!password || password.length < 8) {
      throw new Error('Password must be at least 8 characters');
    }
    
    if (!/(?=.*[0-9])(?=.*[a-z])(?=.*[A-Z])/.test(password)) {
      throw new Error('Password must contain numbers, lowercase and uppercase letters');
    }
  }
}

module.exports = RegisterUser;
```

```javascript
// src/useCases/user/LoginUser.js

class LoginUser {
  constructor({ userRepository, passwordHasher, tokenService }) {
    this.userRepository = userRepository;
    this.passwordHasher = passwordHasher;
    this.tokenService = tokenService;
  }

  async execute({ email, password }) {
    // หา user จาก email
    const user = await this.userRepository.findByEmail(email);
    
    if (!user) {
      throw new Error('Invalid credentials');
    }
    
    if (!user.isActive) {
      throw new Error('Account is deactivated');
    }
    
    // ตรวจสอบ password
    const isValid = await user.verifyPassword(password, 
      (p, h) => this.passwordHasher.compare(p, h)
    );
    
    if (!isValid) {
      throw new Error('Invalid credentials');
    }
    
    // สร้าง token
    const token = await this.tokenService.generateToken({
      userId: user.id,
      role: user.role
    });
    
    return { user, token };
  }
}

module.exports = LoginUser;
```

```javascript
// src/useCases/post/CreatePost.js

class CreatePost {
  constructor({ postRepository, userRepository }) {
    this.postRepository = postRepository;
    this.userRepository = userRepository;
  }

  async execute({ title, content, authorId, tags }) {
    // ตรวจสอบว่า author มีอยู่จริง
    const author = await this.userRepository.findById(authorId);
    
    if (!author) {
      throw new Error('Author not found');
    }
    
    if (!author.canPublishContent()) {
      throw new Error('User cannot publish content');
    }
    
    // สร้าง Post entity
    const Post = require('../../entities/Post');
    const post = new Post({
      id: require('crypto').randomUUID(),
      title,
      content,
      authorId,
      tags
    });
    
    await this.postRepository.save(post);
    
    return post;
  }
}

// src/useCases/post/PublishPost.js
class PublishPost {
  constructor({ postRepository, userRepository }) {
    this.postRepository = postRepository;
    this.userRepository = userRepository;
  }

  async execute({ postId, userId }) {
    const post = await this.postRepository.findById(postId);
    
    if (!post) {
      throw new Error('Post not found');
    }
    
    // ตรวจสอบ ownership
    if (!post.isOwnedBy(userId)) {
      const user = await this.userRepository.findById(userId);
      if (!user || !user.isAdmin()) {
        throw new Error('Unauthorized to publish this post');
      }
    }
    
    post.publish();
    await this.postRepository.update(post.id, { 
      status: post.status,
      publishedAt: post._publishedAt
    });
    
    return post;
  }
}

module.exports = { CreatePost, PublishPost };
```

---

## ขั้นตอนที่ 985: Interface Adapters - Repositories

```javascript
// src/adapters/repositories/MongoUserRepository.js
// Infrastructure layer - implements IUserRepository

const { IUserRepository } = require('../../useCases/ports/IUserRepository');
const { User } = require('../../entities/User');
const UserModel = require('../database/models/UserModel');

class MongoUserRepository extends IUserRepository {
  async findById(id) {
    const doc = await UserModel.findById(id).lean();
    return doc ? this._toDomain(doc) : null;
  }

  async findByEmail(email) {
    const doc = await UserModel.findOne({ email: email.toLowerCase() }).lean();
    return doc ? this._toDomain(doc) : null;
  }

  async save(user) {
    const data = this._toPersistence(user);
    await UserModel.create({ _id: user.id, ...data });
    return user;
  }

  async update(id, data) {
    await UserModel.findByIdAndUpdate(id, { $set: data, updatedAt: new Date() });
  }

  async delete(id) {
    await UserModel.findByIdAndDelete(id);
  }

  async findAll(filters = {}, pagination = {}) {
    const { page = 1, limit = 20 } = pagination;
    const query = this._buildQuery(filters);
    
    const docs = await UserModel.find(query)
      .skip((page - 1) * limit)
      .limit(limit)
      .sort({ createdAt: -1 })
      .lean();
    
    return docs.map(doc => this._toDomain(doc));
  }

  async countByFilters(filters = {}) {
    return UserModel.countDocuments(this._buildQuery(filters));
  }

  // Mapping: MongoDB doc -> Domain entity
  _toDomain(doc) {
    return new User({
      id: doc._id.toString(),
      email: doc.email,
      name: doc.name,
      role: doc.role,
      hashedPassword: doc.password,
      isActive: doc.isActive
    });
  }

  // Mapping: Domain entity -> MongoDB doc
  _toPersistence(user) {
    return {
      email: user.email,
      name: user.name,
      role: user.role,
      password: user._hashedPassword,
      isActive: user.isActive
    };
  }

  _buildQuery(filters) {
    const query = {};
    if (filters.role) query.role = filters.role;
    if (filters.isActive !== undefined) query.isActive = filters.isActive;
    if (filters.search) {
      query.$or = [
        { name: new RegExp(filters.search, 'i') },
        { email: new RegExp(filters.search, 'i') }
      ];
    }
    return query;
  }
}

module.exports = MongoUserRepository;
```

---

## ขั้นตอนที่ 986: Interface Adapters - Controllers

```javascript
// src/adapters/controllers/UserController.js

class UserController {
  constructor({ registerUser, loginUser, getUserById }) {
    this.registerUser = registerUser;
    this.loginUser = loginUser;
    this.getUserById = getUserById;
  }

  register = async (req, res) => {
    try {
      const user = await this.registerUser.execute(req.body);
      
      res.status(201).json({
        success: true,
        data: this._presentUser(user)
      });
    } catch (error) {
      this._handleError(error, res);
    }
  }

  login = async (req, res) => {
    try {
      const { user, token } = await this.loginUser.execute(req.body);
      
      res.json({
        success: true,
        data: {
          user: this._presentUser(user),
          token
        }
      });
    } catch (error) {
      this._handleError(error, res);
    }
  }

  getProfile = async (req, res) => {
    try {
      const user = await this.getUserById.execute(req.user.id);
      
      res.json({
        success: true,
        data: this._presentUser(user)
      });
    } catch (error) {
      this._handleError(error, res);
    }
  }

  // Presenter: แปลง entity ไปยัง response format
  _presentUser(user) {
    return {
      id: user.id,
      email: user.email,
      name: user.name,
      role: user.role
      // ไม่ส่ง hashedPassword กลับ
    };
  }

  _handleError(error, res) {
    const errorMap = {
      'Email already registered': 409,
      'Invalid credentials': 401,
      'Account is deactivated': 403,
      'Not found': 404
    };
    
    const status = errorMap[error.message] || 
      (error.message.includes('Invalid') ? 400 : 500);
    
    res.status(status).json({
      success: false,
      error: error.message
    });
  }
}

module.exports = UserController;
```

---

## ขั้นตอนที่ 987: Interface Adapters - External Services

```javascript
// src/adapters/services/BcryptPasswordHasher.js
// Infrastructure: implements IPasswordHasher

const bcrypt = require('bcrypt');
const { IPasswordHasher } = require('../../useCases/ports/IPasswordHasher');

class BcryptPasswordHasher extends IPasswordHasher {
  constructor(saltRounds = 12) {
    super();
    this.saltRounds = saltRounds;
  }

  async hash(password) {
    return bcrypt.hash(password, this.saltRounds);
  }

  async compare(password, hash) {
    return bcrypt.compare(password, hash);
  }
}

module.exports = BcryptPasswordHasher;
```

```javascript
// src/adapters/services/JWTTokenService.js

const jwt = require('jsonwebtoken');

class JWTTokenService {
  constructor(secret, expiresIn = '24h') {
    this.secret = secret;
    this.expiresIn = expiresIn;
  }

  async generateToken(payload) {
    return jwt.sign(payload, this.secret, { expiresIn: this.expiresIn });
  }

  async verifyToken(token) {
    try {
      return jwt.verify(token, this.secret);
    } catch (error) {
      throw new Error('Invalid or expired token');
    }
  }
}

module.exports = JWTTokenService;
```

```javascript
// src/adapters/services/SendGridEmailService.js
// Infrastructure: implements IEmailService

const sgMail = require('@sendgrid/mail');
const { IEmailService } = require('../../useCases/ports/IEmailService');

class SendGridEmailService extends IEmailService {
  constructor(apiKey, fromEmail) {
    super();
    sgMail.setApiKey(apiKey);
    this.fromEmail = fromEmail;
  }

  async sendWelcomeEmail(user) {
    await sgMail.send({
      to: user.email,
      from: this.fromEmail,
      subject: 'ยินดีต้อนรับสู่ระบบ',
      html: `<h1>สวัสดี ${user.name}!</h1><p>ขอบคุณที่สมัครสมาชิก</p>`
    });
  }

  async sendPasswordResetEmail(user, token) {
    await sgMail.send({
      to: user.email,
      from: this.fromEmail,
      subject: 'รีเซ็ตรหัสผ่าน',
      html: `<p>คลิกลิงก์เพื่อรีเซ็ตรหัสผ่าน: <a href="/reset?token=${token}">Reset</a></p>`
    });
  }

  async sendNotification(userId, message) {
    // ส่ง in-app notification
  }
}

module.exports = SendGridEmailService;
```

---

## ขั้นตอนที่ 988: Dependency Injection Container

```javascript
// src/config/container.js
// Dependency Injection: wire dependencies ทั้งหมด

const MongoUserRepository = require('../adapters/repositories/MongoUserRepository');
const MongoPostRepository = require('../adapters/repositories/MongoPostRepository');
const BcryptPasswordHasher = require('../adapters/services/BcryptPasswordHasher');
const JWTTokenService = require('../adapters/services/JWTTokenService');
const SendGridEmailService = require('../adapters/services/SendGridEmailService');

// Use Cases
const RegisterUser = require('../useCases/user/RegisterUser');
const LoginUser = require('../useCases/user/LoginUser');
const { CreatePost, PublishPost } = require('../useCases/post/CreatePost');

// Controllers
const UserController = require('../adapters/controllers/UserController');
const PostController = require('../adapters/controllers/PostController');

function createContainer(config) {
  // Repositories
  const userRepository = new MongoUserRepository();
  const postRepository = new MongoPostRepository();
  
  // Services
  const passwordHasher = new BcryptPasswordHasher(config.bcryptSaltRounds);
  const tokenService = new JWTTokenService(config.jwtSecret, config.jwtExpiry);
  const emailService = new SendGridEmailService(
    config.sendgridKey,
    config.fromEmail
  );
  
  // Use Cases
  const registerUser = new RegisterUser({
    userRepository,
    emailService,
    passwordHasher
  });
  
  const loginUser = new LoginUser({
    userRepository,
    passwordHasher,
    tokenService
  });
  
  const createPost = new CreatePost({ postRepository, userRepository });
  const publishPost = new PublishPost({ postRepository, userRepository });
  
  // Controllers
  const userController = new UserController({
    registerUser,
    loginUser
  });
  
  const postController = new PostController({
    createPost,
    publishPost
  });
  
  return {
    userController,
    postController,
    tokenService // ใช้ใน middleware
  };
}

module.exports = createContainer;
```

---

## ขั้นตอนที่ 989: Frameworks Layer - Express Routes

```javascript
// src/frameworks/express/routes/userRoutes.js

const express = require('express');
const { authMiddleware } = require('../middleware/authMiddleware');

function createUserRouter(userController) {
  const router = express.Router();
  
  router.post('/register', userController.register);
  router.post('/login', userController.login);
  router.get('/profile', authMiddleware, userController.getProfile);
  
  return router;
}

module.exports = createUserRouter;
```

```javascript
// src/frameworks/express/middleware/authMiddleware.js

function createAuthMiddleware(tokenService) {
  return async (req, res, next) => {
    try {
      const authHeader = req.headers.authorization;
      
      if (!authHeader?.startsWith('Bearer ')) {
        return res.status(401).json({ error: 'Authentication required' });
      }
      
      const token = authHeader.split(' ')[1];
      const payload = await tokenService.verifyToken(token);
      
      req.user = payload;
      next();
    } catch (error) {
      res.status(401).json({ error: 'Invalid token' });
    }
  };
}

module.exports = createAuthMiddleware;
```

```javascript
// src/frameworks/express/app.js

const express = require('express');
const helmet = require('helmet');
const cors = require('cors');
const createContainer = require('../../config/container');
const createUserRouter = require('./routes/userRoutes');
const createPostRouter = require('./routes/postRoutes');
const createAuthMiddleware = require('./middleware/authMiddleware');

function createApp(config) {
  const app = express();
  
  // Setup
  app.use(express.json());
  app.use(helmet());
  app.use(cors());
  
  // Create DI Container
  const container = createContainer(config);
  
  // Create middleware
  const authMiddleware = createAuthMiddleware(container.tokenService);
  
  // Register routes
  app.use('/api/users', createUserRouter(container.userController, authMiddleware));
  app.use('/api/posts', createPostRouter(container.postController, authMiddleware));
  
  // Error handling
  app.use((err, req, res, next) => {
    console.error(err.stack);
    res.status(500).json({ success: false, error: 'Internal server error' });
  });
  
  return app;
}

module.exports = createApp;
```

---

## ขั้นตอนที่ 990: Database Models

```javascript
// src/adapters/database/models/UserModel.js

const mongoose = require('mongoose');

const userSchema = new mongoose.Schema({
  _id: String,
  email: { type: String, required: true, unique: true, lowercase: true },
  name: { type: String, required: true },
  password: { type: String, required: true },
  role: { type: String, enum: ['user', 'admin', 'moderator'], default: 'user' },
  isActive: { type: Boolean, default: true }
}, {
  timestamps: true,
  versionKey: false
});

userSchema.index({ email: 1 });

module.exports = mongoose.model('User', userSchema);
```

---

## ขั้นตอนที่ 991: Testing Use Cases

```javascript
// tests/useCases/RegisterUser.test.js

const RegisterUser = require('../../src/useCases/user/RegisterUser');

describe('RegisterUser Use Case', () => {
  let registerUser;
  let mockUserRepository;
  let mockEmailService;
  let mockPasswordHasher;

  beforeEach(() => {
    // Mocks (ไม่ต้องใช้ database จริง)
    mockUserRepository = {
      findByEmail: jest.fn(),
      save: jest.fn()
    };
    
    mockEmailService = {
      sendWelcomeEmail: jest.fn().mockResolvedValue(true)
    };
    
    mockPasswordHasher = {
      hash: jest.fn().mockResolvedValue('hashed-password')
    };
    
    registerUser = new RegisterUser({
      userRepository: mockUserRepository,
      emailService: mockEmailService,
      passwordHasher: mockPasswordHasher
    });
  });

  test('should register user successfully', async () => {
    mockUserRepository.findByEmail.mockResolvedValue(null);
    mockUserRepository.save.mockResolvedValue(true);
    
    const user = await registerUser.execute({
      email: 'test@example.com',
      name: 'Test User',
      password: 'Password123'
    });
    
    expect(user.email).toBe('test@example.com');
    expect(user.name).toBe('Test User');
    expect(mockUserRepository.save).toHaveBeenCalledTimes(1);
    expect(mockEmailService.sendWelcomeEmail).toHaveBeenCalledTimes(1);
  });

  test('should throw if email already exists', async () => {
    mockUserRepository.findByEmail.mockResolvedValue({ id: 'existing-user' });
    
    await expect(registerUser.execute({
      email: 'existing@example.com',
      name: 'Test',
      password: 'Password123'
    })).rejects.toThrow('Email already registered');
  });

  test('should throw if password is weak', async () => {
    await expect(registerUser.execute({
      email: 'test@example.com',
      name: 'Test',
      password: 'weak'
    })).rejects.toThrow('Password must be at least 8 characters');
  });
});
```

---

## ขั้นตอนที่ 992: Use Case Output Ports

```javascript
// src/useCases/user/GetUserProfile.js
// Use Case ที่มี Output Port ชัดเจน

class GetUserProfile {
  constructor({ userRepository }) {
    this.userRepository = userRepository;
  }

  async execute(userId) {
    const user = await this.userRepository.findById(userId);
    
    if (!user) {
      throw new NotFoundError(`User not found: ${userId}`);
    }
    
    return user;
  }
}

// Use Case Result Object
class UseCaseResult {
  constructor({ success, data, error }) {
    this.success = success;
    this.data = data;
    this.error = error;
  }

  static ok(data) {
    return new UseCaseResult({ success: true, data });
  }

  static fail(error) {
    return new UseCaseResult({ success: false, error });
  }
}

// Domain Errors
class NotFoundError extends Error {
  constructor(message) {
    super(message);
    this.name = 'NotFoundError';
    this.statusCode = 404;
  }
}

class UnauthorizedError extends Error {
  constructor(message) {
    super(message);
    this.name = 'UnauthorizedError';
    this.statusCode = 401;
  }
}

class ValidationError extends Error {
  constructor(message, fields) {
    super(message);
    this.name = 'ValidationError';
    this.statusCode = 400;
    this.fields = fields;
  }
}

module.exports = { GetUserProfile, UseCaseResult, NotFoundError, UnauthorizedError, ValidationError };
```

---

## ขั้นตอนที่ 993: Clean Architecture กับ CQRS

```javascript
// src/useCases/queries/GetPostsQuery.js

class GetPostsQuery {
  constructor({ postRepository, userRepository, cacheService }) {
    this.postRepository = postRepository;
    this.userRepository = userRepository;
    this.cacheService = cacheService;
  }

  async execute({ page = 1, limit = 20, authorId, tag }) {
    const cacheKey = `posts:${page}:${limit}:${authorId || ''}:${tag || ''}`;
    
    // ตรวจสอบ cache ก่อน
    const cached = await this.cacheService.get(cacheKey);
    if (cached) return cached;
    
    // ดึงข้อมูล
    const filters = {};
    if (authorId) filters.authorId = authorId;
    if (tag) filters.tag = tag;
    
    const [posts, total] = await Promise.all([
      this.postRepository.findPublished({ page, limit, filters }),
      this.postRepository.countPublished(filters)
    ]);
    
    // Enrich ด้วย author info
    const authorIds = [...new Set(posts.map(p => p.authorId))];
    const authors = await this.userRepository.findByIds(authorIds);
    const authorMap = new Map(authors.map(a => [a.id, a]));
    
    const result = {
      posts: posts.map(post => ({
        ...post.toJSON(),
        author: authorMap.get(post.authorId)
          ? { id: post.authorId, name: authorMap.get(post.authorId).name }
          : null
      })),
      pagination: {
        page,
        limit,
        total,
        totalPages: Math.ceil(total / limit)
      }
    };
    
    // Cache ผลลัพธ์
    await this.cacheService.set(cacheKey, result, 300); // 5 minutes
    
    return result;
  }
}

module.exports = GetPostsQuery;
```

---

## ขั้นตอนที่ 994: Cross-Cutting Concerns

```javascript
// src/shared/Logger.js

class Logger {
  constructor(context) {
    this.context = context;
  }

  info(message, metadata = {}) {
    console.log(JSON.stringify({
      level: 'info',
      context: this.context,
      message,
      ...metadata,
      timestamp: new Date().toISOString()
    }));
  }

  error(message, error, metadata = {}) {
    console.error(JSON.stringify({
      level: 'error',
      context: this.context,
      message,
      error: {
        name: error?.name,
        message: error?.message,
        stack: error?.stack
      },
      ...metadata,
      timestamp: new Date().toISOString()
    }));
  }
}

// Use Case Decorator: เพิ่ม logging ให้ use case โดยไม่แก้โค้ด
class LoggedUseCase {
  constructor(useCase, logger) {
    this.useCase = useCase;
    this.logger = logger;
  }

  async execute(input) {
    const useCaseName = this.useCase.constructor.name;
    
    this.logger.info(`Executing ${useCaseName}`, { input: this._sanitize(input) });
    
    const start = Date.now();
    
    try {
      const result = await this.useCase.execute(input);
      this.logger.info(`${useCaseName} completed`, {
        duration: Date.now() - start
      });
      return result;
    } catch (error) {
      this.logger.error(`${useCaseName} failed`, error, {
        duration: Date.now() - start
      });
      throw error;
    }
  }

  _sanitize(input) {
    // ลบ sensitive fields
    const { password, token, ...safe } = input || {};
    return safe;
  }
}

module.exports = { Logger, LoggedUseCase };
```

---

## ขั้นตอนที่ 995: Complete Project Structure

```
src/
├── entities/               (1. Enterprise Business Rules)
│   ├── User.js
│   ├── Post.js
│   └── errors/
│       ├── NotFoundError.js
│       └── ValidationError.js
│
├── useCases/               (2. Application Business Rules)
│   ├── user/
│   │   ├── RegisterUser.js
│   │   ├── LoginUser.js
│   │   └── GetUserProfile.js
│   ├── post/
│   │   ├── CreatePost.js
│   │   ├── PublishPost.js
│   │   └── DeletePost.js
│   ├── queries/
│   │   ├── GetPostsQuery.js
│   │   └── GetUserPostsQuery.js
│   └── ports/              (Interfaces/Abstract)
│       ├── IUserRepository.js
│       ├── IPostRepository.js
│       ├── IEmailService.js
│       └── IPasswordHasher.js
│
├── adapters/               (3. Interface Adapters)
│   ├── controllers/
│   │   ├── UserController.js
│   │   └── PostController.js
│   ├── repositories/
│   │   ├── MongoUserRepository.js
│   │   └── MongoPostRepository.js
│   ├── services/
│   │   ├── BcryptPasswordHasher.js
│   │   ├── JWTTokenService.js
│   │   └── SendGridEmailService.js
│   └── database/
│       └── models/
│           ├── UserModel.js
│           └── PostModel.js
│
├── frameworks/             (4. Frameworks & Drivers)
│   └── express/
│       ├── app.js
│       ├── routes/
│       │   ├── userRoutes.js
│       │   └── postRoutes.js
│       └── middleware/
│           ├── authMiddleware.js
│           └── errorHandler.js
│
├── config/
│   └── container.js        (Dependency Injection)
│
└── shared/
    ├── Logger.js
    └── EventPublisher.js
```

---

## ขั้นตอนที่ 996: Entry Point

```javascript
// src/main.js

const mongoose = require('mongoose');
const createApp = require('./frameworks/express/app');

const config = {
  port: process.env.PORT || 3000,
  mongoUri: process.env.MONGODB_URI || 'mongodb://localhost:27017/cleanapp',
  jwtSecret: process.env.JWT_SECRET || 'secret',
  jwtExpiry: process.env.JWT_EXPIRY || '24h',
  bcryptSaltRounds: parseInt(process.env.BCRYPT_ROUNDS || '12'),
  sendgridKey: process.env.SENDGRID_API_KEY,
  fromEmail: process.env.FROM_EMAIL || 'noreply@example.com'
};

async function bootstrap() {
  try {
    // เชื่อมต่อ database
    await mongoose.connect(config.mongoUri);
    console.log('Connected to MongoDB');
    
    // สร้าง Express app
    const app = createApp(config);
    
    // Start server
    app.listen(config.port, () => {
      console.log(`Server running on port ${config.port}`);
    });
  } catch (error) {
    console.error('Failed to start server:', error);
    process.exit(1);
  }
}

bootstrap();
```

---

## ขั้นตอนที่ 997: Testing Strategy

```javascript
// tests/adapters/repositories/MongoUserRepository.test.js
// Integration tests สำหรับ Repository

const mongoose = require('mongoose');
const MongoUserRepository = require('../../../src/adapters/repositories/MongoUserRepository');
const { User } = require('../../../src/entities/User');

describe('MongoUserRepository', () => {
  let repository;

  beforeAll(async () => {
    await mongoose.connect(process.env.TEST_DB_URI);
    repository = new MongoUserRepository();
  });

  afterAll(async () => {
    await mongoose.connection.dropDatabase();
    await mongoose.disconnect();
  });

  afterEach(async () => {
    await mongoose.connection.collections.users?.deleteMany({});
  });

  test('should save and retrieve user', async () => {
    const user = new User({
      id: 'test-id',
      email: 'test@example.com',
      name: 'Test User',
      hashedPassword: 'hashed'
    });
    
    await repository.save(user);
    
    const found = await repository.findById('test-id');
    
    expect(found).toBeTruthy();
    expect(found.email).toBe('test@example.com');
  });

  test('should find by email case insensitively', async () => {
    const user = new User({
      id: 'test-2',
      email: 'TEST@example.com',
      name: 'Test',
      hashedPassword: 'hashed'
    });
    
    await repository.save(user);
    
    const found = await repository.findByEmail('test@example.com');
    expect(found).toBeTruthy();
  });
});
```

---

## ขั้นตอนที่ 998: Validation Decorators

```javascript
// src/shared/Validator.js

class Validator {
  constructor() {
    this._errors = [];
  }

  required(value, fieldName) {
    if (!value || (typeof value === 'string' && !value.trim())) {
      this._errors.push(`${fieldName} is required`);
    }
    return this;
  }

  email(value, fieldName = 'email') {
    const regex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
    if (value && !regex.test(value)) {
      this._errors.push(`${fieldName} must be a valid email`);
    }
    return this;
  }

  minLength(value, min, fieldName) {
    if (value && value.length < min) {
      this._errors.push(`${fieldName} must be at least ${min} characters`);
    }
    return this;
  }

  maxLength(value, max, fieldName) {
    if (value && value.length > max) {
      this._errors.push(`${fieldName} must be at most ${max} characters`);
    }
    return this;
  }

  throw() {
    if (this._errors.length > 0) {
      const { ValidationError } = require('../useCases/user/GetUserProfile');
      throw new ValidationError('Validation failed', this._errors);
    }
    return this;
  }
}

// ใช้งาน
function validateRegisterInput({ email, name, password }) {
  new Validator()
    .required(email, 'email')
    .email(email)
    .required(name, 'name')
    .minLength(name, 2, 'name')
    .required(password, 'password')
    .minLength(password, 8, 'password')
    .throw();
}

module.exports = { Validator, validateRegisterInput };
```

---

## ขั้นตอนที่ 999: Error Handling

```javascript
// src/frameworks/express/middleware/errorHandler.js

function errorHandler(err, req, res, next) {
  // Map application errors to HTTP status codes
  const statusMap = {
    'NotFoundError': 404,
    'UnauthorizedError': 401,
    'ValidationError': 400,
    'ConflictError': 409,
    'ForbiddenError': 403
  };
  
  const status = err.statusCode || statusMap[err.name] || 500;
  
  const response = {
    success: false,
    error: {
      message: err.message,
      type: err.name
    }
  };
  
  // เพิ่ม field errors สำหรับ ValidationError
  if (err.name === 'ValidationError' && err.fields) {
    response.error.fields = err.fields;
  }
  
  // ใน production ไม่ส่ง stack trace
  if (process.env.NODE_ENV === 'development') {
    response.error.stack = err.stack;
  }
  
  // Log server errors
  if (status >= 500) {
    console.error('Server error:', err);
  }
  
  res.status(status).json(response);
}

module.exports = errorHandler;
```

---

## ขั้นตอนที่ 1000: Final Integration

```javascript
// src/frameworks/express/app.js (Final version)

const express = require('express');
const helmet = require('helmet');
const cors = require('cors');
const morgan = require('morgan');
const createContainer = require('../../config/container');
const createUserRouter = require('./routes/userRoutes');
const createPostRouter = require('./routes/postRoutes');
const createAuthMiddleware = require('./middleware/authMiddleware');
const errorHandler = require('./middleware/errorHandler');

function createApp(config) {
  const app = express();
  
  // Security & Parsing middleware
  app.use(helmet());
  app.use(cors({
    origin: config.allowedOrigins || '*',
    credentials: true
  }));
  app.use(express.json({ limit: '10mb' }));
  app.use(express.urlencoded({ extended: true }));
  
  // Logging
  if (config.nodeEnv !== 'test') {
    app.use(morgan('combined'));
  }
  
  // DI Container
  const container = createContainer(config);
  const authMiddleware = createAuthMiddleware(container.tokenService);
  
  // Health check
  app.get('/health', (req, res) => {
    res.json({ status: 'ok', timestamp: new Date() });
  });
  
  // API routes
  app.use('/api/v1/users', createUserRouter(container.userController, authMiddleware));
  app.use('/api/v1/posts', createPostRouter(container.postController, authMiddleware));
  
  // 404 handler
  app.use((req, res) => {
    res.status(404).json({ error: 'Route not found' });
  });
  
  // Error handler (ต้องเป็น last middleware)
  app.use(errorHandler);
  
  return app;
}

module.exports = createApp;
```

---

## แบบฝึกหัดท้ายบท

### แบบฝึกหัดที่ 1: เพิ่ม Comment Feature
เพิ่ม Comment entity พร้อม use cases: AddComment, DeleteComment โดยไม่แก้ไข Post entity

### แบบฝึกหัดที่ 2: Implement Redis Cache
สร้าง RedisCacheService ที่ implement interface และ inject ผ่าน DI container

### แบบฝึกหัดที่ 3: Change Database
เปลี่ยนจาก MongoDB เป็น PostgreSQL โดยสร้าง PostgresUserRepository ใหม่ โดยไม่แก้ไข use cases

### แบบฝึกหัดที่ 4: Add Rate Limiting Use Case
สร้าง CheckRateLimit use case ที่ depend บน IRateLimitRepository interface

### แบบฝึกหัดที่ 5: End-to-End Test
เขียน E2E test ที่ test ทั้ง register -> login -> create post -> publish post flow

---

## สรุป

Clean Architecture ช่วยให้ระบบมีโครงสร้างที่:
- **Independent of frameworks**: สามารถเปลี่ยน Express เป็น Fastify ได้โดยไม่กระทบ business logic
- **Testable**: Use cases ทดสอบได้โดยไม่ต้องใช้ database จริง
- **Independent of UI**: สามารถเพิ่ม GraphQL หรือ CLI ได้โดยไม่แก้ use cases
- **Independent of databases**: เปลี่ยน MongoDB เป็น PostgreSQL ได้
- **Independent of external systems**: เปลี่ยน email service ได้ง่าย

Dependency Rule เป็น core ของ Clean Architecture: outer layers depend on inner layers เท่านั้น
