# Part 66: Clean Architecture
## ขั้นตอนที่ 651-660 จาก 1000

---

## Clean Architecture คืออะไร?

Clean Architecture (โดย Robert C. Martin หรือ Uncle Bob) เป็น architectural pattern ที่จัดระเบียบโค้ดเป็น layers โดยมีหลักการ Dependency Rule ที่ dependencies ต้องชี้เข้าหา center (business logic) เสมอ

---

## 1. Clean Architecture Layers

```
         ┌──────────────────────────────────┐
         │      Frameworks & Drivers         │ (Web, DB, UI)
         │  ┌────────────────────────────┐   │
         │  │   Interface Adapters        │   │ (Controllers, Presenters, Gateways)
         │  │  ┌──────────────────────┐  │   │
         │  │  │   Application BL     │  │   │ (Use Cases)
         │  │  │  ┌────────────────┐  │  │   │
         │  │  │  │  Enterprise BL  │  │  │   │ (Entities)
         │  │  │  └────────────────┘  │  │   │
         │  │  └──────────────────────┘  │   │
         │  └────────────────────────────┘   │
         └──────────────────────────────────┘
```

### Dependency Rule

```
Inner layers ไม่รู้เรื่องของ outer layers
Entities ไม่รู้เรื่อง Use Cases
Use Cases ไม่รู้เรื่อง Controllers
Controllers ไม่รู้เรื่อง Frameworks
```

---

## 2. Entities (Enterprise Business Rules)

```javascript
// src/domain/entities/user.entity.js
class User {
  constructor({ id, email, name, role, createdAt }) {
    this.id = id;
    this.email = email;
    this.name = name;
    this.role = role || 'user';
    this.createdAt = createdAt || new Date();
  }

  isAdmin() {
    return this.role === 'admin';
  }

  canAccessResource(resource) {
    if (this.isAdmin()) return true;
    return resource.ownerId === this.id;
  }

  promote() {
    return new User({ ...this, role: 'admin' });
  }
}

// src/domain/entities/post.entity.js
class Post {
  constructor({ id, title, content, authorId, publishedAt, status }) {
    this.id = id;
    this.title = title;
    this.content = content;
    this.authorId = authorId;
    this.publishedAt = publishedAt;
    this.status = status || 'draft';
  }

  isPublished() {
    return this.status === 'published';
  }

  publish() {
    if (this.isPublished()) throw new Error('Post already published');
    return new Post({ ...this, status: 'published', publishedAt: new Date() });
  }

  unpublish() {
    if (!this.isPublished()) throw new Error('Post is not published');
    return new Post({ ...this, status: 'draft', publishedAt: null });
  }

  canBeEditedBy(user) {
    return user.id === this.authorId || user.isAdmin();
  }
}

module.exports = { User, Post };
```

---

## 3. Use Cases (Application Business Rules)

```javascript
// src/application/use-cases/create-post.use-case.js
class CreatePostUseCase {
  constructor({ postRepository, userRepository, notificationService }) {
    this.postRepository = postRepository;
    this.userRepository = userRepository;
    this.notificationService = notificationService;
  }

  async execute({ title, content, authorId }) {
    // Validate input
    if (!title?.trim()) throw new Error('Title is required');
    if (!content?.trim()) throw new Error('Content is required');

    // Check author exists
    const author = await this.userRepository.findById(authorId);
    if (!author) throw new Error('Author not found');

    // Create post entity
    const { Post } = require('../../domain/entities/post.entity');
    const post = new Post({
      id: require('uuid').v4(),
      title: title.trim(),
      content: content.trim(),
      authorId
    });

    // Persist
    const savedPost = await this.postRepository.save(post);

    // Side effects (ไม่กระทบ main flow)
    await this.notificationService
      .notifyFollowers(authorId, savedPost)
      .catch(err => console.error('Notification failed:', err));

    return savedPost;
  }
}

// src/application/use-cases/publish-post.use-case.js
class PublishPostUseCase {
  constructor({ postRepository, userRepository }) {
    this.postRepository = postRepository;
    this.userRepository = userRepository;
  }

  async execute({ postId, requesterId }) {
    const post = await this.postRepository.findById(postId);
    if (!post) throw new Error('Post not found');

    const requester = await this.userRepository.findById(requesterId);
    if (!requester) throw new Error('User not found');

    if (!post.canBeEditedBy(requester)) {
      throw new Error('Not authorized to publish this post');
    }

    const publishedPost = post.publish();
    return this.postRepository.save(publishedPost);
  }
}

// src/application/use-cases/get-posts.use-case.js
class GetPostsUseCase {
  constructor({ postRepository }) {
    this.postRepository = postRepository;
  }

  async execute({ page = 1, limit = 10, status, authorId } = {}) {
    const result = await this.postRepository.findAll({
      page,
      limit,
      status,
      authorId
    });

    return result;
  }
}

module.exports = { CreatePostUseCase, PublishPostUseCase, GetPostsUseCase };
```

---

## 4. Interface Adapters

### Repositories (Interface)

```javascript
// src/application/ports/post-repository.port.js
// Interface (Port) - กำหนดว่า repository ต้องมี methods อะไร
class PostRepositoryPort {
  async findById(id) { throw new Error('Not implemented'); }
  async findAll(options) { throw new Error('Not implemented'); }
  async save(post) { throw new Error('Not implemented'); }
  async delete(id) { throw new Error('Not implemented'); }
}

// src/application/ports/user-repository.port.js
class UserRepositoryPort {
  async findById(id) { throw new Error('Not implemented'); }
  async findByEmail(email) { throw new Error('Not implemented'); }
  async save(user) { throw new Error('Not implemented'); }
  async delete(id) { throw new Error('Not implemented'); }
}

module.exports = { PostRepositoryPort, UserRepositoryPort };
```

### Repositories (Implementation - Adapter)

```javascript
// src/infrastructure/repositories/mongo-post.repository.js
const { PostRepositoryPort } = require('../../application/ports/post-repository.port');
const PostModel = require('../models/post.model');
const { Post } = require('../../domain/entities/post.entity');

class MongoPostRepository extends PostRepositoryPort {
  async findById(id) {
    const doc = await PostModel.findById(id);
    return doc ? this.toDomain(doc) : null;
  }

  async findAll({ page = 1, limit = 10, status, authorId } = {}) {
    const filter = {};
    if (status) filter.status = status;
    if (authorId) filter.authorId = authorId;

    const [docs, total] = await Promise.all([
      PostModel.find(filter)
        .sort({ createdAt: -1 })
        .skip((page - 1) * limit)
        .limit(limit),
      PostModel.countDocuments(filter)
    ]);

    return {
      data: docs.map(d => this.toDomain(d)),
      total,
      page,
      limit
    };
  }

  async save(post) {
    const data = this.toData(post);
    const doc = await PostModel.findByIdAndUpdate(
      post.id,
      data,
      { new: true, upsert: true }
    );
    return this.toDomain(doc);
  }

  async delete(id) {
    await PostModel.findByIdAndDelete(id);
  }

  toDomain(doc) {
    return new Post({
      id: doc._id.toString(),
      title: doc.title,
      content: doc.content,
      authorId: doc.authorId,
      status: doc.status,
      publishedAt: doc.publishedAt
    });
  }

  toData(post) {
    return {
      _id: post.id,
      title: post.title,
      content: post.content,
      authorId: post.authorId,
      status: post.status,
      publishedAt: post.publishedAt
    };
  }
}

module.exports = MongoPostRepository;
```

### Controllers

```javascript
// src/infrastructure/http/controllers/post.controller.js
class PostController {
  constructor({
    createPostUseCase,
    publishPostUseCase,
    getPostsUseCase
  }) {
    this.createPostUseCase = createPostUseCase;
    this.publishPostUseCase = publishPostUseCase;
    this.getPostsUseCase = getPostsUseCase;
  }

  async create(req, res) {
    const post = await this.createPostUseCase.execute({
      title: req.body.title,
      content: req.body.content,
      authorId: req.user.id
    });

    res.status(201).json({
      success: true,
      data: this.toResponse(post)
    });
  }

  async publish(req, res) {
    const post = await this.publishPostUseCase.execute({
      postId: req.params.id,
      requesterId: req.user.id
    });

    res.json({
      success: true,
      data: this.toResponse(post)
    });
  }

  async list(req, res) {
    const { page, limit, status } = req.query;
    const result = await this.getPostsUseCase.execute({ page, limit, status });

    res.json({
      success: true,
      ...result
    });
  }

  // Controller เป็นผู้กำหนด response format
  toResponse(post) {
    return {
      id: post.id,
      title: post.title,
      content: post.content,
      status: post.status,
      publishedAt: post.publishedAt
    };
  }
}

module.exports = PostController;
```

---

## 5. Frameworks Layer (Dependency Injection)

```javascript
// src/infrastructure/container.js
// Composition Root - ประกอบ dependencies ทั้งหมด

const mongoose = require('mongoose');

// Repositories
const MongoPostRepository = require('./repositories/mongo-post.repository');
const MongoUserRepository = require('./repositories/mongo-user.repository');

// Services
const EmailNotificationService = require('./services/email-notification.service');

// Use Cases
const { CreatePostUseCase, PublishPostUseCase, GetPostsUseCase } = require('../application/use-cases');

// Controllers
const PostController = require('./http/controllers/post.controller');

class Container {
  constructor() {
    this._instances = new Map();
  }

  get(name) {
    if (!this._instances.has(name)) {
      this._instances.set(name, this._create(name));
    }
    return this._instances.get(name);
  }

  _create(name) {
    switch (name) {
      case 'postRepository':
        return new MongoPostRepository();
      
      case 'userRepository':
        return new MongoUserRepository();
      
      case 'notificationService':
        return new EmailNotificationService();
      
      case 'createPostUseCase':
        return new CreatePostUseCase({
          postRepository: this.get('postRepository'),
          userRepository: this.get('userRepository'),
          notificationService: this.get('notificationService')
        });
      
      case 'publishPostUseCase':
        return new PublishPostUseCase({
          postRepository: this.get('postRepository'),
          userRepository: this.get('userRepository')
        });
      
      case 'getPostsUseCase':
        return new GetPostsUseCase({
          postRepository: this.get('postRepository')
        });
      
      case 'postController':
        return new PostController({
          createPostUseCase: this.get('createPostUseCase'),
          publishPostUseCase: this.get('publishPostUseCase'),
          getPostsUseCase: this.get('getPostsUseCase')
        });
      
      default:
        throw new Error(`Unknown dependency: ${name}`);
    }
  }
}

module.exports = new Container();
```

### Express Router Setup

```javascript
// src/infrastructure/http/routes/post.routes.js
const express = require('express');
const router = express.Router();
const container = require('../../container');
const authenticate = require('../middleware/authenticate');

const postController = container.get('postController');

// Bind methods เพื่อรักษา context
router.post('/',
  authenticate,
  (req, res, next) => postController.create(req, res).catch(next)
);

router.put('/:id/publish',
  authenticate,
  (req, res, next) => postController.publish(req, res).catch(next)
);

router.get('/',
  (req, res, next) => postController.list(req, res).catch(next)
);

module.exports = router;
```

### Main App

```javascript
// src/main.js
const express = require('express');
const mongoose = require('mongoose');
const postRoutes = require('./infrastructure/http/routes/post.routes');
const errorHandler = require('./infrastructure/http/middleware/error-handler');

const app = express();
app.use(express.json());

// Routes
app.use('/api/posts', postRoutes);

// Error handler
app.use(errorHandler);

// Database connection
mongoose.connect(process.env.MONGODB_URI)
  .then(() => {
    app.listen(process.env.PORT || 3000, () => {
      console.log(`Server running on port ${process.env.PORT || 3000}`);
    });
  });

module.exports = app;
```

---

## 6. Testing Clean Architecture

```javascript
// tests/use-cases/create-post.test.js
const { CreatePostUseCase } = require('../../src/application/use-cases');

describe('CreatePostUseCase', () => {
  let useCase;
  let mockPostRepository;
  let mockUserRepository;
  let mockNotificationService;

  beforeEach(() => {
    // Mock repositories (ง่ายมาก เพราะ use case ไม่รู้จัก implementation)
    mockPostRepository = {
      save: jest.fn()
    };

    mockUserRepository = {
      findById: jest.fn()
    };

    mockNotificationService = {
      notifyFollowers: jest.fn().mockResolvedValue(undefined)
    };

    useCase = new CreatePostUseCase({
      postRepository: mockPostRepository,
      userRepository: mockUserRepository,
      notificationService: mockNotificationService
    });
  });

  it('should create a post successfully', async () => {
    const mockUser = { id: 'user-1', name: 'Test User' };
    const mockSavedPost = { id: 'post-1', title: 'Test', content: 'Content' };
    
    mockUserRepository.findById.mockResolvedValue(mockUser);
    mockPostRepository.save.mockResolvedValue(mockSavedPost);

    const result = await useCase.execute({
      title: 'Test Post',
      content: 'Test content',
      authorId: 'user-1'
    });

    expect(result).toEqual(mockSavedPost);
    expect(mockPostRepository.save).toHaveBeenCalled();
  });

  it('should throw error if title is empty', async () => {
    await expect(useCase.execute({
      title: '',
      content: 'Content',
      authorId: 'user-1'
    })).rejects.toThrow('Title is required');
  });

  it('should throw error if author not found', async () => {
    mockUserRepository.findById.mockResolvedValue(null);

    await expect(useCase.execute({
      title: 'Test',
      content: 'Content',
      authorId: 'invalid-id'
    })).rejects.toThrow('Author not found');
  });
});
```

---

## โครงสร้างโฟลเดอร์

```
src/
├── domain/
│   ├── entities/
│   │   ├── user.entity.js
│   │   └── post.entity.js
│   └── value-objects/
│       ├── email.vo.js
│       └── money.vo.js
├── application/
│   ├── use-cases/
│   │   ├── create-post.use-case.js
│   │   └── get-posts.use-case.js
│   └── ports/
│       ├── post-repository.port.js
│       └── notification.port.js
└── infrastructure/
    ├── repositories/
    │   └── mongo-post.repository.js
    ├── services/
    │   └── email-notification.service.js
    ├── http/
    │   ├── controllers/
    │   ├── middleware/
    │   └── routes/
    └── container.js
```

---

## แบบฝึกหัด

### ระดับ 1: พื้นฐาน
สร้าง Blog API ด้วย Clean Architecture:
- Post entity
- Create/Get Use Cases
- MongoDB Repository

### ระดับ 2: กลาง
เพิ่ม:
- User authentication
- Authorization (owner check)
- Comment system

### ระดับ 3: ขั้นสูง
สร้าง complete system:
- Dependency Injection container
- Unit tests สำหรับ use cases
- Integration tests

---

## สรุป

Clean Architecture ทำให้โค้ดแยก concerns ชัดเจน ทดสอบง่าย และยืดหยุ่นสูง business logic อยู่ใน entities และ use cases ที่ไม่ขึ้นกับ framework หรือ database framework สามารถเปลี่ยนได้โดยไม่กระทบ business logic

> ขั้นตอนต่อไป: Part 67 - Monitoring & Alerting
