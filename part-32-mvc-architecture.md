# Part 32: MVC Architecture ใน Node.js/Express

> ขั้นตอนที่ 32-32 จาก 1000

---

## สารบัญ

1. [MVC Pattern คืออะไร](#mvc-pattern-คืออะไร)
2. [Models, Views, Controllers](#models-views-controllers)
3. [Express MVC Project Structure](#express-mvc-project-structure)
4. [Service Layer](#service-layer)
5. [Repository Pattern](#repository-pattern)
6. [Practical: Blog API ด้วย MVC](#practical-blog-api)
7. [แบบฝึกหัด](#แบบฝึกหัด)

---

## MVC Pattern คืออะไร

MVC (Model-View-Controller) เป็น architectural pattern ที่แยก application ออกเป็น 3 ส่วน เพื่อให้ง่ายต่อการพัฒนาและดูแลรักษา

```
┌─────────────────────────────────────────┐
│                 MVC Pattern              │
│                                         │
│  ┌─────────┐    ┌────────────┐          │
│  │  View   │◄───│ Controller │          │
│  └─────────┘    └──────┬─────┘          │
│                        │                │
│                   ┌────▼─────┐          │
│                   │  Model   │          │
│                   └──────────┘          │
└─────────────────────────────────────────┘

Flow:
Request → Controller → Model (data) → View → Response
```

### ประโยชน์ของ MVC

| ข้อดี | คำอธิบาย |
|-------|----------|
| Separation of Concerns | แต่ละส่วนมีหน้าที่ชัดเจน |
| Reusability | Model สามารถใช้กับ Controller หลายตัว |
| Testability | ทดสอบแต่ละส่วนแยกกันได้ |
| Maintainability | แก้ไขส่วนหนึ่งโดยไม่กระทบส่วนอื่น |
| Scalability | ขยายระบบได้ง่ายขึ้น |

---

## Models, Views, Controllers

### Model

Model คือชั้นที่จัดการ data logic และ business rules มีหน้าที่:
- ติดต่อกับ database
- Validate data
- Business logic
- Data transformation

```javascript
// models/User.js - ตัวอย่างด้วย Mongoose
const mongoose = require('mongoose');
const bcrypt = require('bcryptjs');

const userSchema = new mongoose.Schema({
  name: {
    type: String,
    required: [true, 'Name is required'],
    trim: true,
    maxlength: [100, 'Name cannot exceed 100 characters'],
  },
  email: {
    type: String,
    required: [true, 'Email is required'],
    unique: true,
    lowercase: true,
    match: [/^\S+@\S+\.\S+$/, 'Please provide a valid email'],
  },
  password: {
    type: String,
    required: [true, 'Password is required'],
    minlength: [8, 'Password must be at least 8 characters'],
    select: false, // ไม่ส่ง password ใน queries โดยปริยาย
  },
  role: {
    type: String,
    enum: ['user', 'admin', 'moderator'],
    default: 'user',
  },
  avatar: String,
  bio: String,
  isActive: {
    type: Boolean,
    default: true,
  },
  lastLogin: Date,
}, {
  timestamps: true, // สร้าง createdAt และ updatedAt อัตโนมัติ
  toJSON: { virtuals: true },
  toObject: { virtuals: true },
});

// Virtual field
userSchema.virtual('fullProfile').get(function() {
  return {
    id: this._id,
    name: this.name,
    email: this.email,
    role: this.role,
  };
});

// Pre-save hook: hash password
userSchema.pre('save', async function(next) {
  if (!this.isModified('password')) return next();
  
  const salt = await bcrypt.genSalt(10);
  this.password = await bcrypt.hash(this.password, salt);
  next();
});

// Instance method
userSchema.methods.comparePassword = async function(candidatePassword) {
  return bcrypt.compare(candidatePassword, this.password);
};

// Static method
userSchema.statics.findByEmail = function(email) {
  return this.findOne({ email: email.toLowerCase() });
};

const User = mongoose.model('User', userSchema);
module.exports = User;
```

### View (สำหรับ REST API)

สำหรับ REST API View คือ JSON response ที่ส่งกลับ เราใช้ "view helpers" หรือ serializers:

```javascript
// views/userView.js - Serializer/Transformer
class UserView {
  static single(user) {
    return {
      id: user._id || user.id,
      name: user.name,
      email: user.email,
      role: user.role,
      avatar: user.avatar,
      bio: user.bio,
      createdAt: user.createdAt,
    };
    // ไม่ส่ง password, __v, internal fields
  }
  
  static list(users, pagination = {}) {
    return {
      data: users.map(user => this.single(user)),
      pagination: {
        page: pagination.page || 1,
        limit: pagination.limit || 10,
        total: pagination.total || users.length,
        pages: Math.ceil((pagination.total || users.length) / (pagination.limit || 10)),
      },
    };
  }
  
  static created(user) {
    return {
      message: 'User created successfully',
      user: this.single(user),
    };
  }
  
  static updated(user) {
    return {
      message: 'User updated successfully',
      user: this.single(user),
    };
  }
}

module.exports = UserView;
```

### Controller

Controller จัดการ HTTP requests/responses และประสานงานระหว่าง Model และ View:

```javascript
// controllers/userController.js
const User = require('../models/User');
const UserView = require('../views/userView');
const { AppError } = require('../utils/errors');

class UserController {
  // GET /users
  async getAll(req, res, next) {
    try {
      const page = parseInt(req.query.page) || 1;
      const limit = parseInt(req.query.limit) || 10;
      const skip = (page - 1) * limit;
      
      const [users, total] = await Promise.all([
        User.find({ isActive: true })
          .select('-password')
          .skip(skip)
          .limit(limit)
          .sort('-createdAt'),
        User.countDocuments({ isActive: true }),
      ]);
      
      res.json(UserView.list(users, { page, limit, total }));
    } catch (error) {
      next(error);
    }
  }
  
  // GET /users/:id
  async getOne(req, res, next) {
    try {
      const user = await User.findById(req.params.id).select('-password');
      
      if (!user) {
        return next(new AppError('User not found', 404));
      }
      
      res.json(UserView.single(user));
    } catch (error) {
      next(error);
    }
  }
  
  // POST /users
  async create(req, res, next) {
    try {
      const { name, email, password, role } = req.body;
      
      const user = await User.create({ name, email, password, role });
      
      res.status(201).json(UserView.created(user));
    } catch (error) {
      if (error.code === 11000) {
        return next(new AppError('Email already exists', 409));
      }
      next(error);
    }
  }
  
  // PUT /users/:id
  async update(req, res, next) {
    try {
      const { name, bio, avatar } = req.body;
      
      const user = await User.findByIdAndUpdate(
        req.params.id,
        { name, bio, avatar },
        { new: true, runValidators: true }
      ).select('-password');
      
      if (!user) {
        return next(new AppError('User not found', 404));
      }
      
      res.json(UserView.updated(user));
    } catch (error) {
      next(error);
    }
  }
  
  // DELETE /users/:id
  async delete(req, res, next) {
    try {
      const user = await User.findByIdAndUpdate(
        req.params.id,
        { isActive: false },
        { new: true }
      );
      
      if (!user) {
        return next(new AppError('User not found', 404));
      }
      
      res.json({ message: 'User deleted successfully' });
    } catch (error) {
      next(error);
    }
  }
}

module.exports = new UserController();
```

---

## Express MVC Project Structure

### โครงสร้าง Project

```
src/
├── config/
│   ├── index.js          # Main config
│   ├── database.js       # DB connection
│   └── redis.js          # Redis connection
│
├── models/
│   ├── User.js
│   ├── Post.js
│   ├── Comment.js
│   └── index.js          # Export all models
│
├── views/                # Response serializers
│   ├── userView.js
│   ├── postView.js
│   └── commentView.js
│
├── controllers/
│   ├── authController.js
│   ├── userController.js
│   ├── postController.js
│   └── commentController.js
│
├── routes/
│   ├── auth.js
│   ├── users.js
│   ├── posts.js
│   └── index.js          # Route aggregator
│
├── middleware/
│   ├── auth.js           # JWT verification
│   ├── validate.js       # Request validation
│   ├── rateLimiter.js
│   └── errorHandler.js
│
├── services/             # Business logic
│   ├── authService.js
│   ├── emailService.js
│   └── uploadService.js
│
├── utils/
│   ├── errors.js         # Custom error classes
│   ├── logger.js
│   └── helpers.js
│
└── app.js                # Express app setup
```

### Route Setup

```javascript
// routes/posts.js
const express = require('express');
const router = express.Router();
const postController = require('../controllers/postController');
const { authenticate, authorize } = require('../middleware/auth');
const { validate } = require('../middleware/validate');
const { createPostSchema, updatePostSchema } = require('../validators/postValidator');

router.get('/', postController.getAll);
router.get('/:id', postController.getOne);

router.use(authenticate); // Routes ด้านล่างต้อง login ก่อน

router.post('/', validate(createPostSchema), postController.create);
router.put('/:id', validate(updatePostSchema), postController.update);
router.delete('/:id', authorize('admin', 'moderator'), postController.delete);

module.exports = router;
```

```javascript
// routes/index.js
const express = require('express');
const router = express.Router();

router.use('/auth', require('./auth'));
router.use('/users', require('./users'));
router.use('/posts', require('./posts'));
router.use('/comments', require('./comments'));

module.exports = router;
```

```javascript
// app.js
const express = require('express');
const cors = require('cors');
const helmet = require('helmet');
const morgan = require('morgan');
const routes = require('./routes');
const errorHandler = require('./middleware/errorHandler');
const config = require('./config');

const app = express();

// Middleware
app.use(helmet());
app.use(cors({ origin: config.server.allowedOrigins }));
app.use(express.json({ limit: '10mb' }));
app.use(express.urlencoded({ extended: true }));
app.use(morgan('combined'));

// Routes
app.use('/api/v1', routes);

// Health check
app.get('/health', (req, res) => res.json({ status: 'ok' }));

// 404 handler
app.use((req, res) => {
  res.status(404).json({ error: 'Route not found' });
});

// Error handler (ต้องอยู่สุดท้าย)
app.use(errorHandler);

module.exports = app;
```

---

## Service Layer

Service Layer เป็น layer เพิ่มเติมระหว่าง Controller กับ Model สำหรับ complex business logic:

```
Request → Controller → Service → Model → Database
                     ↑
               Business Logic
               - Transactions
               - Multiple models
               - External APIs
               - Complex queries
```

### ตัวอย่าง Auth Service

```javascript
// services/authService.js
const jwt = require('jsonwebtoken');
const crypto = require('crypto');
const User = require('../models/User');
const Token = require('../models/Token');
const emailService = require('./emailService');
const config = require('../config');
const { AppError } = require('../utils/errors');

class AuthService {
  async register(userData) {
    const { name, email, password } = userData;
    
    // ตรวจสอบว่า email มีอยู่แล้วหรือไม่
    const existingUser = await User.findByEmail(email);
    if (existingUser) {
      throw new AppError('Email already registered', 409);
    }
    
    // สร้าง user
    const user = await User.create({ name, email, password });
    
    // ส่ง verification email
    const verificationToken = crypto.randomBytes(32).toString('hex');
    await Token.create({
      userId: user._id,
      token: verificationToken,
      type: 'email_verification',
      expiresAt: new Date(Date.now() + 24 * 60 * 60 * 1000), // 24 hours
    });
    
    await emailService.sendVerificationEmail(user.email, verificationToken);
    
    return user;
  }
  
  async login(email, password) {
    // ดึง user พร้อม password field
    const user = await User.findOne({ email }).select('+password');
    
    if (!user || !await user.comparePassword(password)) {
      throw new AppError('Invalid email or password', 401);
    }
    
    if (!user.isActive) {
      throw new AppError('Account is disabled', 403);
    }
    
    // อัพเดต lastLogin
    user.lastLogin = new Date();
    await user.save({ validateBeforeSave: false });
    
    // สร้าง tokens
    const accessToken = this.generateAccessToken(user);
    const refreshToken = await this.generateRefreshToken(user._id);
    
    return { user, accessToken, refreshToken };
  }
  
  async refreshAccessToken(refreshTokenString) {
    const tokenDoc = await Token.findOne({
      token: refreshTokenString,
      type: 'refresh',
      expiresAt: { $gt: new Date() },
    });
    
    if (!tokenDoc) {
      throw new AppError('Invalid or expired refresh token', 401);
    }
    
    const user = await User.findById(tokenDoc.userId);
    if (!user || !user.isActive) {
      throw new AppError('User not found or inactive', 401);
    }
    
    const accessToken = this.generateAccessToken(user);
    return { accessToken, user };
  }
  
  async logout(userId, refreshToken) {
    await Token.deleteOne({ userId, token: refreshToken, type: 'refresh' });
  }
  
  generateAccessToken(user) {
    return jwt.sign(
      { id: user._id, role: user.role },
      config.auth.jwtSecret,
      { expiresIn: config.auth.jwtExpires }
    );
  }
  
  async generateRefreshToken(userId) {
    const token = crypto.randomBytes(64).toString('hex');
    
    await Token.create({
      userId,
      token,
      type: 'refresh',
      expiresAt: new Date(Date.now() + 30 * 24 * 60 * 60 * 1000), // 30 days
    });
    
    return token;
  }
}

module.exports = new AuthService();
```

### ตัวอย่าง Post Service

```javascript
// services/postService.js
const Post = require('../models/Post');
const User = require('../models/User');
const Tag = require('../models/Tag');
const { AppError } = require('../utils/errors');
const slugify = require('slugify');

class PostService {
  async createPost(authorId, postData) {
    const { title, content, tags: tagNames, status } = postData;
    
    // สร้าง slug จาก title
    let slug = slugify(title, { lower: true, strict: true });
    
    // ตรวจสอบว่า slug ซ้ำหรือไม่
    const existingPost = await Post.findOne({ slug });
    if (existingPost) {
      slug = `${slug}-${Date.now()}`;
    }
    
    // จัดการ tags
    const tagIds = await this.processTagNames(tagNames || []);
    
    // สร้าง post
    const post = await Post.create({
      title,
      slug,
      content,
      author: authorId,
      tags: tagIds,
      status: status || 'draft',
    });
    
    return post.populate(['author', 'tags']);
  }
  
  async processTagNames(tagNames) {
    const tagIds = [];
    
    for (const name of tagNames) {
      // upsert tag
      const tag = await Tag.findOneAndUpdate(
        { name: name.toLowerCase() },
        { $setOnInsert: { name: name.toLowerCase(), slug: slugify(name, { lower: true }) } },
        { upsert: true, new: true }
      );
      tagIds.push(tag._id);
    }
    
    return tagIds;
  }
  
  async getPublishedPosts(options = {}) {
    const {
      page = 1,
      limit = 10,
      tag,
      author,
      search,
      sortBy = '-publishedAt',
    } = options;
    
    const query = { status: 'published' };
    
    if (tag) {
      const tagDoc = await Tag.findOne({ slug: tag });
      if (tagDoc) query.tags = tagDoc._id;
    }
    
    if (author) {
      const authorDoc = await User.findOne({ username: author });
      if (authorDoc) query.author = authorDoc._id;
    }
    
    if (search) {
      query.$text = { $search: search };
    }
    
    const skip = (page - 1) * limit;
    
    const [posts, total] = await Promise.all([
      Post.find(query)
        .populate('author', 'name avatar')
        .populate('tags', 'name slug')
        .skip(skip)
        .limit(limit)
        .sort(sortBy),
      Post.countDocuments(query),
    ]);
    
    return { posts, total, page, limit };
  }
  
  async publishPost(postId, authorId) {
    const post = await Post.findOne({ _id: postId, author: authorId });
    
    if (!post) {
      throw new AppError('Post not found', 404);
    }
    
    if (post.status === 'published') {
      throw new AppError('Post is already published', 400);
    }
    
    post.status = 'published';
    post.publishedAt = new Date();
    await post.save();
    
    return post;
  }
}

module.exports = new PostService();
```

---

## Repository Pattern

Repository Pattern เพิ่ม abstraction layer ระหว่าง business logic และ data access:

```
Controller → Service → Repository → Database
                      ↑
              Data Access Layer
              - Database queries
              - Caching
              - Data mapping
```

### Base Repository

```javascript
// repositories/baseRepository.js
class BaseRepository {
  constructor(model) {
    this.model = model;
  }
  
  async findAll(filter = {}, options = {}) {
    const {
      page = 1,
      limit = 10,
      sort = '-createdAt',
      select,
      populate,
    } = options;
    
    const skip = (page - 1) * limit;
    
    let query = this.model.find(filter).skip(skip).limit(limit).sort(sort);
    
    if (select) query = query.select(select);
    if (populate) query = query.populate(populate);
    
    const [data, total] = await Promise.all([
      query.exec(),
      this.model.countDocuments(filter),
    ]);
    
    return { data, total, page, limit, pages: Math.ceil(total / limit) };
  }
  
  async findById(id, options = {}) {
    let query = this.model.findById(id);
    if (options.select) query = query.select(options.select);
    if (options.populate) query = query.populate(options.populate);
    return query.exec();
  }
  
  async findOne(filter, options = {}) {
    let query = this.model.findOne(filter);
    if (options.select) query = query.select(options.select);
    if (options.populate) query = query.populate(options.populate);
    return query.exec();
  }
  
  async create(data) {
    return this.model.create(data);
  }
  
  async updateById(id, data, options = { new: true, runValidators: true }) {
    return this.model.findByIdAndUpdate(id, data, options);
  }
  
  async deleteById(id) {
    return this.model.findByIdAndDelete(id);
  }
  
  async count(filter = {}) {
    return this.model.countDocuments(filter);
  }
  
  async exists(filter) {
    return this.model.exists(filter);
  }
}

module.exports = BaseRepository;
```

### Post Repository

```javascript
// repositories/postRepository.js
const BaseRepository = require('./baseRepository');
const Post = require('../models/Post');

class PostRepository extends BaseRepository {
  constructor() {
    super(Post);
  }
  
  async findPublished(options = {}) {
    return this.findAll(
      { status: 'published' },
      { ...options, populate: ['author', 'tags'] }
    );
  }
  
  async findBySlug(slug) {
    return this.findOne({ slug }, {
      populate: [
        { path: 'author', select: 'name avatar bio' },
        { path: 'tags', select: 'name slug' },
      ],
    });
  }
  
  async findByAuthor(authorId, options = {}) {
    return this.findAll({ author: authorId }, options);
  }
  
  async incrementViewCount(id) {
    return this.model.findByIdAndUpdate(
      id,
      { $inc: { viewCount: 1 } },
      { new: true }
    );
  }
  
  async findRelated(postId, tags, limit = 5) {
    return this.model.find({
      _id: { $ne: postId },
      tags: { $in: tags },
      status: 'published',
    })
    .limit(limit)
    .select('title slug excerpt author publishedAt')
    .populate('author', 'name');
  }
  
  async searchPosts(searchTerm, options = {}) {
    return this.findAll(
      {
        $text: { $search: searchTerm },
        status: 'published',
      },
      {
        ...options,
        sort: { score: { $meta: 'textScore' } },
        select: { score: { $meta: 'textScore' } },
      }
    );
  }
}

module.exports = new PostRepository();
```

---

## Practical: Blog API ด้วย MVC

### สร้าง Blog API แบบ Complete

```javascript
// models/Post.js
const mongoose = require('mongoose');

const postSchema = new mongoose.Schema({
  title: {
    type: String,
    required: [true, 'Title is required'],
    trim: true,
    maxlength: 200,
  },
  slug: {
    type: String,
    unique: true,
    lowercase: true,
  },
  excerpt: {
    type: String,
    maxlength: 500,
  },
  content: {
    type: String,
    required: [true, 'Content is required'],
  },
  author: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User',
    required: true,
  },
  tags: [{
    type: mongoose.Schema.Types.ObjectId,
    ref: 'Tag',
  }],
  status: {
    type: String,
    enum: ['draft', 'published', 'archived'],
    default: 'draft',
  },
  featuredImage: String,
  viewCount: { type: Number, default: 0 },
  likeCount: { type: Number, default: 0 },
  commentCount: { type: Number, default: 0 },
  publishedAt: Date,
}, {
  timestamps: true,
  toJSON: { virtuals: true },
});

// Text index สำหรับ full-text search
postSchema.index({ title: 'text', content: 'text', excerpt: 'text' });

// Index สำหรับ query performance
postSchema.index({ author: 1, status: 1, createdAt: -1 });
postSchema.index({ tags: 1, status: 1 });
postSchema.index({ slug: 1 });

module.exports = mongoose.model('Post', postSchema);
```

```javascript
// controllers/postController.js
const postService = require('../services/postService');
const PostView = require('../views/postView');
const { AppError } = require('../utils/errors');

class PostController {
  async getPosts(req, res, next) {
    try {
      const { page, limit, tag, author, search, sort } = req.query;
      
      const result = await postService.getPublishedPosts({
        page: parseInt(page) || 1,
        limit: parseInt(limit) || 10,
        tag,
        author,
        search,
        sortBy: sort || '-publishedAt',
      });
      
      res.json(PostView.list(result));
    } catch (error) {
      next(error);
    }
  }
  
  async getPost(req, res, next) {
    try {
      const post = await postService.getPostBySlug(req.params.slug);
      
      if (!post) {
        return next(new AppError('Post not found', 404));
      }
      
      // increment view count (non-blocking)
      postService.incrementViewCount(post._id).catch(console.error);
      
      res.json(PostView.single(post));
    } catch (error) {
      next(error);
    }
  }
  
  async createPost(req, res, next) {
    try {
      const post = await postService.createPost(req.user.id, req.body);
      res.status(201).json(PostView.created(post));
    } catch (error) {
      next(error);
    }
  }
  
  async updatePost(req, res, next) {
    try {
      const post = await postService.updatePost(
        req.params.id,
        req.user.id,
        req.body
      );
      res.json(PostView.updated(post));
    } catch (error) {
      next(error);
    }
  }
  
  async publishPost(req, res, next) {
    try {
      const post = await postService.publishPost(req.params.id, req.user.id);
      res.json({ message: 'Post published successfully', post: PostView.single(post) });
    } catch (error) {
      next(error);
    }
  }
  
  async deletePost(req, res, next) {
    try {
      await postService.deletePost(req.params.id, req.user.id);
      res.json({ message: 'Post deleted successfully' });
    } catch (error) {
      next(error);
    }
  }
  
  async getMyPosts(req, res, next) {
    try {
      const { page, limit, status } = req.query;
      
      const result = await postService.getUserPosts(req.user.id, {
        page: parseInt(page) || 1,
        limit: parseInt(limit) || 10,
        status,
      });
      
      res.json(PostView.list(result));
    } catch (error) {
      next(error);
    }
  }
}

module.exports = new PostController();
```

```javascript
// views/postView.js
class PostView {
  static single(post) {
    return {
      id: post._id,
      title: post.title,
      slug: post.slug,
      excerpt: post.excerpt,
      content: post.content,
      author: post.author ? {
        id: post.author._id,
        name: post.author.name,
        avatar: post.author.avatar,
      } : post.author,
      tags: (post.tags || []).map(tag => ({
        id: tag._id || tag,
        name: tag.name,
        slug: tag.slug,
      })),
      status: post.status,
      featuredImage: post.featuredImage,
      viewCount: post.viewCount,
      likeCount: post.likeCount,
      commentCount: post.commentCount,
      publishedAt: post.publishedAt,
      createdAt: post.createdAt,
      updatedAt: post.updatedAt,
    };
  }
  
  static list({ data: posts, ...pagination }) {
    return {
      posts: posts.map(p => this.single(p)),
      pagination,
    };
  }
  
  static created(post) {
    return {
      message: 'Post created successfully',
      post: this.single(post),
    };
  }
  
  static updated(post) {
    return {
      message: 'Post updated successfully',
      post: this.single(post),
    };
  }
}

module.exports = PostView;
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: สร้าง MVC Structure

สร้าง Express app พร้อม MVC structure สำหรับ Task Manager:
- Model: Task (title, description, status, priority, dueDate, userId)
- Controller: taskController (CRUD operations)
- Service: taskService (business logic)
- View: taskView (response formatting)
- Routes: /api/tasks

### แบบฝึกหัดที่ 2: Repository Pattern

เพิ่ม Repository Pattern:
- BaseRepository (findAll, findById, create, update, delete)
- TaskRepository (findByUser, findByStatus, findOverdue)
- Unit tests สำหรับ repository methods

### แบบฝึกหัดที่ 3: Complete Blog API

สร้าง Blog API ที่ครบสมบูรณ์:
- Authentication (register/login/logout)
- Posts CRUD (with slug, tags, status)
- Comments
- Pagination
- Search

```bash
# Expected endpoints
POST   /api/auth/register
POST   /api/auth/login
GET    /api/posts?page=1&limit=10&tag=nodejs
GET    /api/posts/:slug
POST   /api/posts              # authenticated
PUT    /api/posts/:id          # author only
DELETE /api/posts/:id          # author or admin
GET    /api/posts/me           # my posts
```

---

## สรุป

MVC Architecture ช่วยให้ code มีโครงสร้างที่ดีและง่ายต่อการดูแล

```
Model      → Data + Business rules
View       → Response formatting
Controller → HTTP handling + orchestration
Service    → Complex business logic
Repository → Data access abstraction
```

**ถัดไป**: [Part 33: RESTful API Design →](./part-33-restful-api-design.md)
