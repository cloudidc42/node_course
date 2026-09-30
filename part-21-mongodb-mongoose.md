# Part 21: MongoDB และ Mongoose

> ขั้นตอนที่ 21-30 จาก 1000 — การจัดการฐานข้อมูล NoSQL ด้วย MongoDB และ Mongoose ORM

---

## สารบัญ

1. [MongoDB คืออะไร](#mongodb-คืออะไร)
2. [การติดตั้งและเชื่อมต่อ](#การติดตั้งและเชื่อมต่อ)
3. [Mongoose Schema และ Model](#mongoose-schema-และ-model)
4. [CRUD Operations](#crud-operations)
5. [Validation](#validation)
6. [Population (refs)](#population-refs)
7. [Aggregation Pipeline](#aggregation-pipeline)
8. [Indexing](#indexing)
9. [Practical: Blog API กับ MongoDB](#practical-blog-api-กับ-mongodb)
10. [แบบฝึกหัด](#แบบฝึกหัด)

---

## MongoDB คืออะไร

MongoDB เป็นฐานข้อมูลแบบ **NoSQL Document-based** ที่เก็บข้อมูลในรูปแบบ JSON-like documents (BSON) แทนที่จะเป็นตารางแบบ relational database

### ข้อดีของ MongoDB

- **Schema-less** — สามารถเพิ่ม field ได้โดยไม่ต้อง migrate
- **Horizontal Scaling** — scale out ได้ง่ายด้วย sharding
- **JSON Native** — เข้ากันได้ดีกับ JavaScript/Node.js
- **Rich Query Language** — มี query operators ครบถ้วน
- **High Performance** — เร็วสำหรับ read-heavy workload

### เปรียบเทียบกับ SQL

| SQL | MongoDB |
|-----|---------|
| Database | Database |
| Table | Collection |
| Row | Document |
| Column | Field |
| Primary Key | `_id` |
| JOIN | $lookup / populate |
| INDEX | Index |

### BSON Document ตัวอย่าง

```json
{
  "_id": "ObjectId('64abc123...')",
  "name": "สมชาย ใจดี",
  "email": "somchai@example.com",
  "age": 30,
  "address": {
    "street": "ถนนสุขุมวิท",
    "city": "กรุงเทพ",
    "zip": "10110"
  },
  "tags": ["developer", "nodejs", "mongodb"],
  "createdAt": "ISODate('2024-01-15T10:30:00Z')"
}
```

---

## การติดตั้งและเชื่อมต่อ

### ติดตั้ง MongoDB

**macOS (Homebrew):**
```bash
brew tap mongodb/brew
brew install mongodb-community
brew services start mongodb-community
```

**Ubuntu/Debian:**
```bash
# นำเข้า GPG key
curl -fsSL https://pgp.mongodb.com/server-7.0.asc | sudo gpg -o /usr/share/keyrings/mongodb-server-7.0.gpg --dearmor

# เพิ่ม repository
echo "deb [ arch=amd64,arm64 signed-by=/usr/share/keyrings/mongodb-server-7.0.gpg ] https://repo.mongodb.org/apt/ubuntu jammy/mongodb-org/7.0 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-org-7.0.list

# ติดตั้ง
sudo apt-get update
sudo apt-get install -y mongodb-org
sudo systemctl start mongod
```

**Docker (แนะนำสำหรับ development):**
```bash
# รัน MongoDB container
docker run -d \
  --name mongodb \
  -p 27017:27017 \
  -e MONGO_INITDB_ROOT_USERNAME=admin \
  -e MONGO_INITDB_ROOT_PASSWORD=secret \
  -v mongodb_data:/data/db \
  mongo:7.0
```

### ติดตั้ง Mongoose

```bash
npm install mongoose
npm install dotenv  # สำหรับ environment variables
```

### การเชื่อมต่อพื้นฐาน

```javascript
// config/database.js
const mongoose = require('mongoose');

// Connection URI
const MONGODB_URI = process.env.MONGODB_URI || 'mongodb://localhost:27017/myapp';

// ตัวเลือกการเชื่อมต่อ
const options = {
  maxPoolSize: 10,           // จำนวน connection สูงสุดใน pool
  serverSelectionTimeoutMS: 5000,  // timeout สำหรับ server selection
  socketTimeoutMS: 45000,          // timeout สำหรับ socket
};

// ฟังก์ชันเชื่อมต่อ
const connectDB = async () => {
  try {
    const conn = await mongoose.connect(MONGODB_URI, options);
    console.log(`MongoDB Connected: ${conn.connection.host}`);
    
    // Event listeners
    mongoose.connection.on('error', (err) => {
      console.error('MongoDB connection error:', err);
    });
    
    mongoose.connection.on('disconnected', () => {
      console.log('MongoDB disconnected');
    });
    
    // Graceful shutdown
    process.on('SIGINT', async () => {
      await mongoose.connection.close();
      console.log('MongoDB connection closed through app termination');
      process.exit(0);
    });
    
  } catch (error) {
    console.error('Error connecting to MongoDB:', error.message);
    process.exit(1);
  }
};

module.exports = connectDB;
```

```javascript
// app.js
require('dotenv').config();
const express = require('express');
const connectDB = require('./config/database');

const app = express();

// เชื่อมต่อ database ก่อนเริ่ม server
connectDB();

app.use(express.json());

// ... routes

const PORT = process.env.PORT || 3000;
app.listen(PORT, () => {
  console.log(`Server running on port ${PORT}`);
});
```

### ไฟล์ .env

```env
MONGODB_URI=mongodb://localhost:27017/blog_app
PORT=3000
NODE_ENV=development
```

---

## Mongoose Schema และ Model

### Schema Types

Mongoose รองรับ types ต่อไปนี้:

| Type | ตัวอย่าง |
|------|---------|
| `String` | `"hello"` |
| `Number` | `42` |
| `Date` | `new Date()` |
| `Boolean` | `true/false` |
| `Buffer` | binary data |
| `ObjectId` | `mongoose.Types.ObjectId` |
| `Array` | `[1, 2, 3]` |
| `Map` | key-value pairs |
| `Mixed` | any type |

### สร้าง Schema พื้นฐาน

```javascript
// models/User.js
const mongoose = require('mongoose');

const userSchema = new mongoose.Schema(
  {
    // String fields
    name: {
      type: String,
      required: [true, 'กรุณาระบุชื่อ'],
      trim: true,
      minlength: [2, 'ชื่อต้องมีอย่างน้อย 2 ตัวอักษร'],
      maxlength: [100, 'ชื่อต้องไม่เกิน 100 ตัวอักษร'],
    },
    
    email: {
      type: String,
      required: [true, 'กรุณาระบุ email'],
      unique: true,
      lowercase: true,  // แปลงเป็น lowercase อัตโนมัติ
      trim: true,
      match: [
        /^\w+([.-]?\w+)*@\w+([.-]?\w+)*(\.\w{2,3})+$/,
        'กรุณาระบุ email ที่ถูกต้อง',
      ],
    },
    
    // Number fields
    age: {
      type: Number,
      min: [0, 'อายุต้องไม่ติดลบ'],
      max: [150, 'อายุไม่ถูกต้อง'],
    },
    
    // Enum
    role: {
      type: String,
      enum: {
        values: ['user', 'admin', 'moderator'],
        message: '{VALUE} ไม่ใช่ role ที่รองรับ',
      },
      default: 'user',
    },
    
    // Boolean
    isActive: {
      type: Boolean,
      default: true,
    },
    
    // Nested object
    profile: {
      bio: {
        type: String,
        maxlength: 500,
      },
      avatar: String,
      website: String,
    },
    
    // Array of strings
    skills: [String],
    
    // Array of objects
    addresses: [
      {
        street: String,
        city: String,
        country: {
          type: String,
          default: 'Thailand',
        },
      },
    ],
    
    // Date
    lastLoginAt: Date,
    
    // Mixed type
    metadata: mongoose.Schema.Types.Mixed,
  },
  {
    // Schema options
    timestamps: true,          // เพิ่ม createdAt และ updatedAt อัตโนมัติ
    toJSON: { virtuals: true },  // รวม virtual fields ใน JSON output
    toObject: { virtuals: true },
  }
);

// Virtual field (ไม่ได้เก็บใน database)
userSchema.virtual('fullName').get(function () {
  return `${this.firstName} ${this.lastName}`;
});

// Instance method
userSchema.methods.toSafeObject = function () {
  const obj = this.toObject();
  delete obj.password;
  delete obj.__v;
  return obj;
};

// Static method
userSchema.statics.findByEmail = function (email) {
  return this.findOne({ email: email.toLowerCase() });
};

// Middleware (hooks)
userSchema.pre('save', function (next) {
  // ทำงานก่อน save ทุกครั้ง
  console.log(`กำลัง save user: ${this.email}`);
  next();
});

userSchema.post('save', function (doc) {
  // ทำงานหลัง save
  console.log(`User saved: ${doc._id}`);
});

// สร้าง Model
const User = mongoose.model('User', userSchema);

module.exports = User;
```

### Schema Composition

```javascript
// models/Post.js
const mongoose = require('mongoose');

// Schema ย่อยสำหรับ comments
const commentSchema = new mongoose.Schema(
  {
    content: {
      type: String,
      required: true,
      trim: true,
    },
    author: {
      type: mongoose.Schema.Types.ObjectId,
      ref: 'User',
      required: true,
    },
    likes: [
      {
        type: mongoose.Schema.Types.ObjectId,
        ref: 'User',
      },
    ],
  },
  { timestamps: true }
);

const postSchema = new mongoose.Schema(
  {
    title: {
      type: String,
      required: [true, 'กรุณาระบุหัวข้อ'],
      trim: true,
      maxlength: [200, 'หัวข้อต้องไม่เกิน 200 ตัวอักษร'],
    },
    
    slug: {
      type: String,
      unique: true,
      lowercase: true,
    },
    
    content: {
      type: String,
      required: [true, 'กรุณาระบุเนื้อหา'],
    },
    
    excerpt: {
      type: String,
      maxlength: 500,
    },
    
    author: {
      type: mongoose.Schema.Types.ObjectId,
      ref: 'User',
      required: true,
    },
    
    category: {
      type: String,
      enum: ['technology', 'lifestyle', 'travel', 'food', 'other'],
      default: 'other',
    },
    
    tags: [String],
    
    image: {
      url: String,
      alt: String,
    },
    
    // ฝัง commentSchema เข้าไป
    comments: [commentSchema],
    
    views: {
      type: Number,
      default: 0,
    },
    
    likes: [
      {
        type: mongoose.Schema.Types.ObjectId,
        ref: 'User',
      },
    ],
    
    status: {
      type: String,
      enum: ['draft', 'published', 'archived'],
      default: 'draft',
    },
    
    publishedAt: Date,
  },
  { timestamps: true }
);

// Auto-generate slug จาก title
postSchema.pre('save', function (next) {
  if (this.isModified('title')) {
    this.slug = this.title
      .toLowerCase()
      .replace(/[^a-z0-9\s-]/g, '')
      .replace(/\s+/g, '-')
      .replace(/-+/g, '-')
      .trim('-');
  }
  
  // auto-set publishedAt
  if (this.status === 'published' && !this.publishedAt) {
    this.publishedAt = new Date();
  }
  
  next();
});

// Virtual: จำนวน comments
postSchema.virtual('commentCount').get(function () {
  return this.comments.length;
});

// Virtual: จำนวน likes
postSchema.virtual('likeCount').get(function () {
  return this.likes.length;
});

// Index
postSchema.index({ slug: 1 });
postSchema.index({ author: 1, status: 1 });
postSchema.index({ tags: 1 });
postSchema.index({ createdAt: -1 });

const Post = mongoose.model('Post', postSchema);

module.exports = Post;
```

---

## CRUD Operations

### Create (สร้างข้อมูล)

```javascript
// controllers/userController.js
const User = require('../models/User');

// วิธีที่ 1: new + save
const createUserV1 = async (req, res) => {
  try {
    const user = new User({
      name: req.body.name,
      email: req.body.email,
      age: req.body.age,
    });
    
    const savedUser = await user.save();
    res.status(201).json({
      success: true,
      data: savedUser,
    });
  } catch (error) {
    res.status(400).json({
      success: false,
      message: error.message,
    });
  }
};

// วิธีที่ 2: Model.create() (แนะนำ)
const createUser = async (req, res) => {
  try {
    const user = await User.create(req.body);
    
    res.status(201).json({
      success: true,
      data: user,
    });
  } catch (error) {
    // จัดการ duplicate key error
    if (error.code === 11000) {
      return res.status(400).json({
        success: false,
        message: `${Object.keys(error.keyValue)} นี้มีอยู่แล้ว`,
      });
    }
    
    res.status(400).json({
      success: false,
      message: error.message,
    });
  }
};

// สร้างหลาย documents พร้อมกัน
const createManyUsers = async (req, res) => {
  try {
    const users = await User.insertMany(req.body.users, {
      ordered: false,  // false = ดำเนินการต่อแม้บางรายการผิดพลาด
    });
    
    res.status(201).json({
      success: true,
      count: users.length,
      data: users,
    });
  } catch (error) {
    res.status(400).json({
      success: false,
      message: error.message,
    });
  }
};
```

### Read (อ่านข้อมูล)

```javascript
// ดึงข้อมูลทั้งหมด พร้อม filtering, sorting, pagination
const getUsers = async (req, res) => {
  try {
    const {
      page = 1,
      limit = 10,
      sort = '-createdAt',
      fields,
      name,
      role,
      isActive,
      minAge,
      maxAge,
    } = req.query;
    
    // สร้าง filter object
    const filter = {};
    
    if (name) {
      filter.name = { $regex: name, $options: 'i' };  // case-insensitive search
    }
    if (role) filter.role = role;
    if (isActive !== undefined) filter.isActive = isActive === 'true';
    if (minAge || maxAge) {
      filter.age = {};
      if (minAge) filter.age.$gte = parseInt(minAge);
      if (maxAge) filter.age.$lte = parseInt(maxAge);
    }
    
    // Select specific fields
    const selectFields = fields ? fields.split(',').join(' ') : '-__v';
    
    // นับจำนวนทั้งหมด
    const total = await User.countDocuments(filter);
    
    // Query
    const users = await User.find(filter)
      .select(selectFields)
      .sort(sort)
      .limit(parseInt(limit))
      .skip((parseInt(page) - 1) * parseInt(limit));
    
    res.json({
      success: true,
      total,
      count: users.length,
      page: parseInt(page),
      totalPages: Math.ceil(total / parseInt(limit)),
      data: users,
    });
  } catch (error) {
    res.status(500).json({ success: false, message: error.message });
  }
};

// ดึงข้อมูลรายการเดียว
const getUser = async (req, res) => {
  try {
    const user = await User.findById(req.params.id).select('-__v');
    
    if (!user) {
      return res.status(404).json({
        success: false,
        message: 'ไม่พบผู้ใช้งาน',
      });
    }
    
    res.json({
      success: true,
      data: user,
    });
  } catch (error) {
    // จัดการ invalid ObjectId
    if (error.name === 'CastError') {
      return res.status(400).json({
        success: false,
        message: 'ID ไม่ถูกต้อง',
      });
    }
    res.status(500).json({ success: false, message: error.message });
  }
};

// ค้นหาด้วย conditions ต่างๆ
const searchUsers = async (req, res) => {
  try {
    // findOne — หาตัวแรกที่ match
    const userByEmail = await User.findOne({ email: req.query.email });
    
    // find ด้วย operators
    const users = await User.find({
      $or: [                    // OR condition
        { role: 'admin' },
        { age: { $gt: 30 } },  // greater than
      ],
      isActive: true,
      skills: { $in: ['nodejs', 'react'] },  // array contains
    });
    
    res.json({ success: true, data: users });
  } catch (error) {
    res.status(500).json({ success: false, message: error.message });
  }
};
```

### Update (แก้ไขข้อมูล)

```javascript
// อัปเดต document
const updateUser = async (req, res) => {
  try {
    // findByIdAndUpdate — หา + อัปเดตในคำสั่งเดียว
    const user = await User.findByIdAndUpdate(
      req.params.id,
      req.body,
      {
        new: true,          // return document ที่อัปเดตแล้ว
        runValidators: true, // รัน validation
        context: 'query',   // จำเป็นสำหรับ unique validator ใน update
      }
    );
    
    if (!user) {
      return res.status(404).json({
        success: false,
        message: 'ไม่พบผู้ใช้งาน',
      });
    }
    
    res.json({ success: true, data: user });
  } catch (error) {
    res.status(400).json({ success: false, message: error.message });
  }
};

// Update operators
const updateUserFields = async (req, res) => {
  try {
    const user = await User.findByIdAndUpdate(
      req.params.id,
      {
        $set: { 'profile.bio': req.body.bio },  // set specific field
        $push: { skills: req.body.skill },       // เพิ่มใน array
        $inc: { age: 1 },                        // increment
        $unset: { metadata: '' },               // ลบ field
      },
      { new: true, runValidators: true }
    );
    
    res.json({ success: true, data: user });
  } catch (error) {
    res.status(400).json({ success: false, message: error.message });
  }
};

// Update หลาย documents
const updateManyUsers = async (req, res) => {
  try {
    const result = await User.updateMany(
      { isActive: false, lastLoginAt: { $lt: new Date('2023-01-01') } },
      { $set: { role: 'archived' } }
    );
    
    res.json({
      success: true,
      matchedCount: result.matchedCount,
      modifiedCount: result.modifiedCount,
    });
  } catch (error) {
    res.status(500).json({ success: false, message: error.message });
  }
};
```

### Delete (ลบข้อมูล)

```javascript
// ลบ document
const deleteUser = async (req, res) => {
  try {
    const user = await User.findByIdAndDelete(req.params.id);
    
    if (!user) {
      return res.status(404).json({
        success: false,
        message: 'ไม่พบผู้ใช้งาน',
      });
    }
    
    res.json({
      success: true,
      message: 'ลบผู้ใช้งานเรียบร้อยแล้ว',
    });
  } catch (error) {
    res.status(500).json({ success: false, message: error.message });
  }
};

// Soft delete (แนะนำสำหรับ production)
const softDeleteUser = async (req, res) => {
  try {
    const user = await User.findByIdAndUpdate(
      req.params.id,
      {
        $set: {
          isDeleted: true,
          deletedAt: new Date(),
        },
      },
      { new: true }
    );
    
    res.json({
      success: true,
      message: 'ลบผู้ใช้งานเรียบร้อยแล้ว',
    });
  } catch (error) {
    res.status(500).json({ success: false, message: error.message });
  }
};

// ลบหลาย documents
const deleteManyUsers = async (req, res) => {
  try {
    const result = await User.deleteMany({
      isActive: false,
      createdAt: { $lt: new Date(Date.now() - 365 * 24 * 60 * 60 * 1000) },
    });
    
    res.json({
      success: true,
      deletedCount: result.deletedCount,
    });
  } catch (error) {
    res.status(500).json({ success: false, message: error.message });
  }
};
```

---

## Validation

### Built-in Validators

```javascript
const productSchema = new mongoose.Schema({
  name: {
    type: String,
    required: [true, 'ชื่อสินค้าจำเป็น'],
    minlength: [3, 'ชื่อต้องมีอย่างน้อย 3 ตัวอักษร'],
    maxlength: [100, 'ชื่อต้องไม่เกิน 100 ตัวอักษร'],
    trim: true,
  },
  
  price: {
    type: Number,
    required: true,
    min: [0, 'ราคาต้องไม่ติดลบ'],
    max: [1000000, 'ราคาเกินขีดจำกัด'],
  },
  
  category: {
    type: String,
    enum: {
      values: ['electronics', 'clothing', 'food', 'books'],
      message: 'ประเภท {VALUE} ไม่รองรับ',
    },
  },
  
  sku: {
    type: String,
    unique: true,
    match: [/^[A-Z]{3}-\d{6}$/, 'รูปแบบ SKU ไม่ถูกต้อง (ตัวอย่าง: ABC-123456)'],
  },
  
  stock: {
    type: Number,
    default: 0,
    validate: {
      validator: function (value) {
        return Number.isInteger(value) && value >= 0;
      },
      message: 'จำนวนสต็อกต้องเป็นจำนวนเต็มไม่ติดลบ',
    },
  },
});
```

### Custom Validators

```javascript
const mongoose = require('mongoose');
const validator = require('validator');  // npm install validator

const userSchema = new mongoose.Schema({
  email: {
    type: String,
    required: true,
    validate: {
      // Custom validator function
      validator: function (value) {
        return validator.isEmail(value);
      },
      message: 'กรุณาระบุ email ที่ถูกต้อง',
    },
  },
  
  phone: {
    type: String,
    validate: {
      validator: function (value) {
        // ตรวจสอบเบอร์โทรไทย
        return /^(0[6-9]\d{8}|0[2-8]\d{7})$/.test(value);
      },
      message: 'รูปแบบเบอร์โทรศัพท์ไม่ถูกต้อง',
    },
  },
  
  website: {
    type: String,
    validate: {
      validator: function (value) {
        if (!value) return true;  // optional field
        return validator.isURL(value);
      },
      message: 'URL ไม่ถูกต้อง',
    },
  },
  
  // Async validator
  username: {
    type: String,
    validate: {
      isAsync: true,
      validator: async function (value) {
        const count = await mongoose.model('User').countDocuments({
          username: value,
          _id: { $ne: this._id },  // exclude current document
        });
        return count === 0;
      },
      message: 'Username นี้มีผู้ใช้แล้ว',
    },
  },
});
```

---

## Population (refs)

Population คือการแทนที่ ObjectId ด้วยข้อมูลจริงจาก collection อื่น

### การตั้งค่า refs

```javascript
// models/Post.js
const postSchema = new mongoose.Schema({
  title: String,
  content: String,
  
  // ref ชี้ไปที่ User model
  author: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User',
    required: true,
  },
  
  // Array of refs
  tags: [
    {
      type: mongoose.Schema.Types.ObjectId,
      ref: 'Tag',
    },
  ],
  
  comments: [
    {
      content: String,
      author: {
        type: mongoose.Schema.Types.ObjectId,
        ref: 'User',
      },
      createdAt: {
        type: Date,
        default: Date.now,
      },
    },
  ],
});
```

### การใช้ populate

```javascript
// populate พื้นฐาน
const post = await Post.findById(id).populate('author');

// select เฉพาะ fields ที่ต้องการ
const post = await Post.findById(id).populate('author', 'name email avatar');

// populate หลาย fields
const post = await Post.findById(id)
  .populate('author', 'name avatar')
  .populate('tags', 'name slug');

// populate ซ้อนกัน (nested populate)
const post = await Post.findById(id).populate({
  path: 'comments.author',
  select: 'name avatar',
});

// Deep populate
const post = await Post.findById(id).populate({
  path: 'author',
  select: 'name',
  populate: {
    path: 'profile',
    select: 'bio website',
  },
});

// populate กับ filter
const posts = await Post.find({ status: 'published' }).populate({
  path: 'author',
  match: { isActive: true },  // เฉพาะ author ที่ active
  select: 'name email',
});

// populate กับ pagination
const posts = await Post.find().populate({
  path: 'comments',
  options: {
    limit: 5,
    sort: { createdAt: -1 },
  },
});
```

### Virtual Populate

```javascript
// models/User.js
const userSchema = new mongoose.Schema({ name: String, email: String });

// Virtual populate — ไม่ได้เก็บ posts ใน User document
userSchema.virtual('posts', {
  ref: 'Post',           // model ที่จะ reference
  localField: '_id',     // field ใน User
  foreignField: 'author', // field ใน Post ที่ชี้มา
});

const User = mongoose.model('User', userSchema);

// ใช้งาน
const user = await User.findById(id).populate('posts');
```

---

## Aggregation Pipeline

Aggregation pipeline คือการประมวลผลข้อมูลเป็นขั้นตอน (stages)

### Stages พื้นฐาน

```javascript
// ตัวอย่าง: สถิติ post ตาม category
const stats = await Post.aggregate([
  // Stage 1: กรองข้อมูล
  {
    $match: {
      status: 'published',
      createdAt: { $gte: new Date('2024-01-01') },
    },
  },
  
  // Stage 2: จัดกลุ่มและคำนวณ
  {
    $group: {
      _id: '$category',
      count: { $sum: 1 },
      totalViews: { $sum: '$views' },
      avgViews: { $avg: '$views' },
      maxViews: { $max: '$views' },
    },
  },
  
  // Stage 3: เพิ่ม/แก้ไข fields
  {
    $addFields: {
      avgViewsRounded: { $round: ['$avgViews', 0] },
    },
  },
  
  // Stage 4: เรียงลำดับ
  {
    $sort: { count: -1 },
  },
  
  // Stage 5: จำกัดจำนวน
  {
    $limit: 5,
  },
  
  // Stage 6: เลือก fields
  {
    $project: {
      category: '$_id',
      count: 1,
      totalViews: 1,
      avgViewsRounded: 1,
      _id: 0,
    },
  },
]);
```

### $lookup (JOIN)

```javascript
// Join posts กับ users
const postsWithAuthors = await Post.aggregate([
  {
    $match: { status: 'published' },
  },
  {
    $lookup: {
      from: 'users',          // collection name
      localField: 'author',   // field ใน Post
      foreignField: '_id',    // field ใน User
      as: 'authorInfo',       // ชื่อ field ที่จะเก็บผล
    },
  },
  {
    $unwind: '$authorInfo',   // แตก array ออกเป็น object
  },
  {
    $project: {
      title: 1,
      content: 1,
      'authorInfo.name': 1,
      'authorInfo.email': 1,
    },
  },
]);

// Pipeline $lookup (มี filter)
const postsWithActiveAuthors = await Post.aggregate([
  {
    $lookup: {
      from: 'users',
      let: { authorId: '$author' },
      pipeline: [
        {
          $match: {
            $expr: { $eq: ['$_id', '$$authorId'] },
            isActive: true,
          },
        },
        {
          $project: { name: 1, email: 1, avatar: 1 },
        },
      ],
      as: 'author',
    },
  },
]);
```

### Aggregation Expressions

```javascript
// คำนวณ engagement rate
const analytics = await Post.aggregate([
  {
    $addFields: {
      likeCount: { $size: '$likes' },
      commentCount: { $size: '$comments' },
      engagementRate: {
        $cond: {
          if: { $gt: ['$views', 0] },
          then: {
            $multiply: [
              {
                $divide: [
                  { $add: [{ $size: '$likes' }, { $size: '$comments' }] },
                  '$views',
                ],
              },
              100,
            ],
          },
          else: 0,
        },
      },
    },
  },
  
  {
    $group: {
      _id: '$author',
      totalPosts: { $sum: 1 },
      totalViews: { $sum: '$views' },
      avgEngagement: { $avg: '$engagementRate' },
      posts: { $push: '$$ROOT' },
    },
  },
]);
```

---

## Indexing

### ประเภทของ Index

```javascript
const postSchema = new mongoose.Schema({
  title: String,
  slug: String,
  author: mongoose.Schema.Types.ObjectId,
  category: String,
  tags: [String],
  views: Number,
  status: String,
  createdAt: Date,
  content: String,
});

// Single field index
postSchema.index({ slug: 1 });           // ascending
postSchema.index({ views: -1 });          // descending
postSchema.index({ slug: 1 }, { unique: true });

// Compound index
postSchema.index({ author: 1, status: 1 });
postSchema.index({ category: 1, createdAt: -1 });

// Text index (สำหรับ full-text search)
postSchema.index({ title: 'text', content: 'text', tags: 'text' });

// Sparse index (เฉพาะ documents ที่มี field นั้น)
postSchema.index({ deletedAt: 1 }, { sparse: true });

// TTL index (ลบ documents อัตโนมัติ)
postSchema.index(
  { createdAt: 1 },
  { expireAfterSeconds: 30 * 24 * 60 * 60 }  // ลบหลัง 30 วัน
);

// Partial index
postSchema.index(
  { views: -1 },
  { partialFilterExpression: { status: 'published' } }
);
```

### การใช้ Text Search

```javascript
// ค้นหาด้วย text search
const searchPosts = async (searchText) => {
  const posts = await Post.find(
    { $text: { $search: searchText } },
    { score: { $meta: 'textScore' } }
  )
    .sort({ score: { $meta: 'textScore' } })
    .limit(10);
  
  return posts;
};

// ตรวจสอบ index ที่ใช้
const explainResult = await Post.find({ slug: 'my-post' }).explain('executionStats');
console.log(explainResult.executionStats);
```

---

## Practical: Blog API กับ MongoDB

### โครงสร้างโปรเจค

```
blog-api/
├── config/
│   └── database.js
├── models/
│   ├── User.js
│   ├── Post.js
│   └── Category.js
├── controllers/
│   ├── authController.js
│   ├── postController.js
│   └── userController.js
├── routes/
│   ├── auth.js
│   ├── posts.js
│   └── users.js
├── middleware/
│   └── auth.js
├── app.js
└── .env
```

### โค้ดหลัก

```javascript
// models/Category.js
const mongoose = require('mongoose');

const categorySchema = new mongoose.Schema(
  {
    name: {
      type: String,
      required: true,
      unique: true,
      trim: true,
    },
    slug: {
      type: String,
      unique: true,
      lowercase: true,
    },
    description: String,
    color: {
      type: String,
      default: '#3498db',
    },
  },
  { timestamps: true }
);

categorySchema.pre('save', function (next) {
  if (this.isModified('name')) {
    this.slug = this.name.toLowerCase().replace(/\s+/g, '-');
  }
  next();
});

module.exports = mongoose.model('Category', categorySchema);
```

```javascript
// controllers/postController.js
const Post = require('../models/Post');
const User = require('../models/User');

// GET /api/posts — ดึง posts พร้อม pagination
exports.getPosts = async (req, res) => {
  try {
    const page = parseInt(req.query.page) || 1;
    const limit = parseInt(req.query.limit) || 10;
    const skip = (page - 1) * limit;
    
    // Build query
    const query = { status: 'published' };
    
    if (req.query.category) query.category = req.query.category;
    if (req.query.tag) query.tags = req.query.tag;
    if (req.query.author) query.author = req.query.author;
    if (req.query.search) {
      query.$text = { $search: req.query.search };
    }
    
    const [posts, total] = await Promise.all([
      Post.find(query)
        .populate('author', 'name avatar')
        .populate('tags', 'name slug color')
        .select('-content -comments')
        .sort(req.query.search ? { score: { $meta: 'textScore' } } : '-publishedAt')
        .limit(limit)
        .skip(skip),
      Post.countDocuments(query),
    ]);
    
    res.json({
      success: true,
      data: posts,
      pagination: {
        total,
        page,
        limit,
        totalPages: Math.ceil(total / limit),
        hasNextPage: page < Math.ceil(total / limit),
        hasPrevPage: page > 1,
      },
    });
  } catch (error) {
    res.status(500).json({ success: false, message: error.message });
  }
};

// POST /api/posts — สร้าง post
exports.createPost = async (req, res) => {
  try {
    const post = await Post.create({
      ...req.body,
      author: req.user.id,  // จาก auth middleware
    });
    
    await post.populate('author', 'name avatar');
    
    res.status(201).json({
      success: true,
      data: post,
    });
  } catch (error) {
    res.status(400).json({ success: false, message: error.message });
  }
};

// POST /api/posts/:id/like — กด like
exports.likePost = async (req, res) => {
  try {
    const post = await Post.findById(req.params.id);
    
    if (!post) {
      return res.status(404).json({
        success: false,
        message: 'ไม่พบบทความ',
      });
    }
    
    const userId = req.user.id;
    const likeIndex = post.likes.indexOf(userId);
    
    if (likeIndex === -1) {
      // เพิ่ม like
      post.likes.push(userId);
    } else {
      // ยกเลิก like
      post.likes.splice(likeIndex, 1);
    }
    
    await post.save();
    
    res.json({
      success: true,
      likeCount: post.likes.length,
      isLiked: likeIndex === -1,
    });
  } catch (error) {
    res.status(500).json({ success: false, message: error.message });
  }
};

// GET /api/posts/stats — สถิติ
exports.getStats = async (req, res) => {
  try {
    const stats = await Post.aggregate([
      { $match: { status: 'published' } },
      {
        $group: {
          _id: null,
          totalPosts: { $sum: 1 },
          totalViews: { $sum: '$views' },
          avgViews: { $avg: '$views' },
          totalLikes: {
            $sum: { $size: '$likes' },
          },
          totalComments: {
            $sum: { $size: '$comments' },
          },
        },
      },
    ]);
    
    const categoryStats = await Post.aggregate([
      { $match: { status: 'published' } },
      { $group: { _id: '$category', count: { $sum: 1 } } },
      { $sort: { count: -1 } },
    ]);
    
    res.json({
      success: true,
      data: {
        overview: stats[0] || {},
        byCategory: categoryStats,
      },
    });
  } catch (error) {
    res.status(500).json({ success: false, message: error.message });
  }
};
```

```javascript
// app.js
require('dotenv').config();
const express = require('express');
const connectDB = require('./config/database');
const postRoutes = require('./routes/posts');
const userRoutes = require('./routes/users');
const authRoutes = require('./routes/auth');

const app = express();
connectDB();

app.use(express.json());
app.use(express.urlencoded({ extended: true }));

// Routes
app.use('/api/auth', authRoutes);
app.use('/api/posts', postRoutes);
app.use('/api/users', userRoutes);

// Error handler
app.use((err, req, res, next) => {
  console.error(err.stack);
  res.status(500).json({
    success: false,
    message: process.env.NODE_ENV === 'production' ? 'เกิดข้อผิดพลาด' : err.message,
  });
});

app.listen(process.env.PORT || 3000, () => {
  console.log(`Server started on port ${process.env.PORT || 3000}`);
});
```

---

## แบบฝึกหัด

### Exercise 1: สร้าง E-commerce Product API

สร้าง API สำหรับจัดการสินค้า โดยมี:

1. **Schema** สำหรับ Product ที่มี fields: name, description, price, category, stock, images[], ratings[], reviews[]
2. **CRUD endpoints** สำหรับ Product
3. **Filtering** ตาม category, price range, rating
4. **Pagination** และ sorting
5. **Aggregation** สำหรับหา:
   - top 10 สินค้าขายดี
   - สินค้าที่มี rating สูงสุดแต่ละ category
   - ยอดขายรวมต่อเดือน

### Exercise 2: Social Media Feed

สร้างระบบ feed สำหรับ social media:

1. User สามารถ follow user อื่นได้
2. ดึง feed ของ user (posts จาก users ที่ follow)
3. Trending posts (views สูงสุดใน 24 ชั่วโมง)
4. Suggest users to follow (based on common followers)

### Exercise 3: Query Optimization

ใช้ `.explain('executionStats')` ตรวจสอบและปรับปรุง performance ของ queries:

1. หา queries ที่ใช้ COLLSCAN
2. เพิ่ม index ที่เหมาะสม
3. วัดเวลาก่อนและหลังเพิ่ม index
4. Document ผลลัพธ์

---

## สรุป

ในบทนี้เราได้เรียนรู้:

- **MongoDB** และความแตกต่างจาก SQL
- **Mongoose** สำหรับ Object Document Mapping
- **Schema** และ **Model** การออกแบบโครงสร้างข้อมูล
- **CRUD Operations** ครบทุกรูปแบบ
- **Validation** ทั้ง built-in และ custom
- **Population** สำหรับ join documents
- **Aggregation Pipeline** สำหรับวิเคราะห์ข้อมูล
- **Indexing** สำหรับเพิ่ม performance

> **บทถัดไป:** Part 22 — PostgreSQL และ Sequelize สำหรับ Relational Database
