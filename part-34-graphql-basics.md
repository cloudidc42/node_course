# Part 34 | ขั้นตอนที่ 581-600 จาก 1000

## GraphQL Fundamentals - พื้นฐาน GraphQL

---

## สารบัญ

1. [GraphQL คืออะไร](#graphql-คืออะไร)
2. [การติดตั้งและตั้งค่า](#การติดตั้งและตั้งค่า)
3. [Schema Definition Language (SDL)](#schema-definition-language)
4. [Types และ Fields](#types-และ-fields)
5. [Queries - การดึงข้อมูล](#queries---การดึงข้อมูล)
6. [Mutations - การเปลี่ยนแปลงข้อมูล](#mutations---การเปลี่ยนแปลงข้อมูล)
7. [Resolvers](#resolvers)
8. [Arguments และ Variables](#arguments-และ-variables)
9. [Fragments](#fragments)
10. [Error Handling](#error-handling)
11. [แบบฝึกหัด](#แบบฝึกหัด)

---

## GraphQL คืออะไร

GraphQL เป็น query language สำหรับ API และ runtime สำหรับ execute queries พัฒนาโดย Facebook ในปี 2012 และเปิดเป็น open source ในปี 2015

### ข้อดีของ GraphQL เทียบกับ REST

```
REST API:
  GET /users/:id          → ได้ข้อมูล user ทั้งหมด (overfetching)
  GET /users/:id/posts    → request แยก (underfetching)
  GET /users/:id/friends  → request แยกอีก

GraphQL:
  query {
    user(id: "1") {
      name           ← เลือกเฉพาะ fields ที่ต้องการ
      posts { title }
      friends { name }
    }
  }
```

### แผนภาพสถาปัตยกรรม GraphQL

```
Client
  │
  │ Single Endpoint: POST /graphql
  │
  ▼
GraphQL Server
  ├── Schema (Type Definitions)
  ├── Resolvers (Business Logic)
  └── Data Sources
        ├── Database (MongoDB, PostgreSQL)
        ├── REST APIs
        └── Other Services
```

---

## การติดตั้งและตั้งค่า

### ขั้นตอนที่ 581: ติดตั้ง Dependencies

```bash
mkdir graphql-course && cd graphql-course
npm init -y

# ติดตั้ง Apollo Server กับ GraphQL
npm install apollo-server-express graphql express

# ติดตั้ง tools เพิ่มเติม
npm install mongoose dotenv
npm install --save-dev nodemon
```

### ขั้นตอนที่ 582: โครงสร้างโปรเจกต์

```
graphql-course/
├── src/
│   ├── schema/
│   │   ├── typeDefs.js
│   │   └── resolvers.js
│   ├── models/
│   │   ├── User.js
│   │   └── Post.js
│   ├── datasources/
│   │   └── UserAPI.js
│   └── index.js
├── .env
└── package.json
```

### ขั้นตอนที่ 583: สร้าง Apollo Server พื้นฐาน

```javascript
// src/index.js
const express = require('express');
const { ApolloServer } = require('apollo-server-express');
const { typeDefs } = require('./schema/typeDefs');
const { resolvers } = require('./schema/resolvers');
require('dotenv').config();

async function startServer() {
  const app = express();

  const server = new ApolloServer({
    typeDefs,
    resolvers,
    context: ({ req }) => {
      // ส่ง context ไปยัง resolvers ทุกตัว
      return {
        user: req.user,
        dataSources: {
          // data sources
        }
      };
    },
    // เปิด introspection ใน development
    introspection: process.env.NODE_ENV !== 'production',
    playground: process.env.NODE_ENV !== 'production',
  });

  await server.start();
  server.applyMiddleware({ app, path: '/graphql' });

  const PORT = process.env.PORT || 4000;
  app.listen(PORT, () => {
    console.log(`🚀 Server ready at http://localhost:${PORT}${server.graphqlPath}`);
  });
}

startServer().catch(console.error);
```

---

## Schema Definition Language

### ขั้นตอนที่ 584: Types พื้นฐาน

```javascript
// src/schema/typeDefs.js
const { gql } = require('apollo-server-express');

const typeDefs = gql`
  # Scalar Types พื้นฐาน
  # String, Int, Float, Boolean, ID

  # Custom Scalar
  scalar DateTime

  # Enum Type
  enum Role {
    ADMIN
    USER
    MODERATOR
  }

  enum PostStatus {
    DRAFT
    PUBLISHED
    ARCHIVED
  }

  # Object Types
  type User {
    id: ID!
    username: String!
    email: String!
    role: Role!
    profile: Profile
    posts: [Post!]!
    createdAt: DateTime!
    updatedAt: DateTime!
  }

  type Profile {
    bio: String
    avatar: String
    website: String
    location: String
  }

  type Post {
    id: ID!
    title: String!
    content: String!
    status: PostStatus!
    author: User!
    tags: [String!]!
    viewCount: Int!
    createdAt: DateTime!
    updatedAt: DateTime!
  }

  # Input Types (สำหรับ Mutations)
  input CreateUserInput {
    username: String!
    email: String!
    password: String!
    role: Role = USER
  }

  input UpdateUserInput {
    username: String
    email: String
    profile: ProfileInput
  }

  input ProfileInput {
    bio: String
    avatar: String
    website: String
    location: String
  }

  input CreatePostInput {
    title: String!
    content: String!
    status: PostStatus = DRAFT
    tags: [String!]
  }

  # Pagination Types
  type PageInfo {
    hasNextPage: Boolean!
    hasPreviousPage: Boolean!
    startCursor: String
    endCursor: String
    totalCount: Int!
  }

  type UserConnection {
    edges: [UserEdge!]!
    pageInfo: PageInfo!
  }

  type UserEdge {
    node: User!
    cursor: String!
  }

  # Query Type - entry point สำหรับการอ่านข้อมูล
  type Query {
    # User queries
    me: User
    user(id: ID!): User
    users(
      limit: Int = 10
      offset: Int = 0
      search: String
    ): [User!]!
    usersConnection(
      first: Int
      after: String
    ): UserConnection!

    # Post queries
    post(id: ID!): Post
    posts(
      authorId: ID
      status: PostStatus
      limit: Int = 10
      offset: Int = 0
    ): [Post!]!
  }

  # Mutation Type - entry point สำหรับการเปลี่ยนแปลงข้อมูล
  type Mutation {
    # Auth mutations
    register(input: CreateUserInput!): AuthPayload!
    login(email: String!, password: String!): AuthPayload!
    logout: Boolean!

    # User mutations
    updateUser(id: ID!, input: UpdateUserInput!): User!
    deleteUser(id: ID!): Boolean!

    # Post mutations
    createPost(input: CreatePostInput!): Post!
    updatePost(id: ID!, input: UpdatePostInput!): Post!
    deletePost(id: ID!): Boolean!
    publishPost(id: ID!): Post!
  }

  input UpdatePostInput {
    title: String
    content: String
    status: PostStatus
    tags: [String!]
  }

  # Auth Response
  type AuthPayload {
    token: String!
    user: User!
  }
`;

module.exports = { typeDefs };
```

---

## Types และ Fields

### ขั้นตอนที่ 585: Custom Scalar Types

```javascript
// src/schema/scalars.js
const { GraphQLScalarType, Kind } = require('graphql');

// DateTime Scalar
const DateTimeScalar = new GraphQLScalarType({
  name: 'DateTime',
  description: 'ISO 8601 DateTime string',

  // แปลงจาก JavaScript value → JSON (สำหรับ serialize)
  serialize(value) {
    if (value instanceof Date) {
      return value.toISOString();
    }
    if (typeof value === 'string') {
      return new Date(value).toISOString();
    }
    throw new Error('DateTime must be a Date object or ISO string');
  },

  // แปลงจาก JSON string → JavaScript value (สำหรับ parse input)
  parseValue(value) {
    if (typeof value === 'string') {
      const date = new Date(value);
      if (isNaN(date.getTime())) {
        throw new Error('Invalid DateTime string');
      }
      return date;
    }
    throw new Error('DateTime must be a string');
  },

  // แปลงจาก AST → JavaScript value (สำหรับ inline values ใน query)
  parseLiteral(ast) {
    if (ast.kind === Kind.STRING) {
      const date = new Date(ast.value);
      if (isNaN(date.getTime())) {
        throw new Error('Invalid DateTime string');
      }
      return date;
    }
    throw new Error('DateTime must be a string literal');
  },
});

// JSON Scalar
const JSONScalar = new GraphQLScalarType({
  name: 'JSON',
  description: 'Arbitrary JSON value',
  serialize: (value) => value,
  parseValue: (value) => value,
  parseLiteral: (ast) => {
    switch (ast.kind) {
      case Kind.STRING:
        return JSON.parse(ast.value);
      case Kind.OBJECT:
        return ast.fields.reduce((obj, field) => {
          obj[field.name.value] = field.value.value;
          return obj;
        }, {});
      default:
        return null;
    }
  },
});

module.exports = { DateTimeScalar, JSONScalar };
```

### ขั้นตอนที่ 586: Mongoose Models

```javascript
// src/models/User.js
const mongoose = require('mongoose');
const bcrypt = require('bcrypt');

const userSchema = new mongoose.Schema({
  username: {
    type: String,
    required: true,
    unique: true,
    trim: true,
    minlength: 3,
    maxlength: 50,
  },
  email: {
    type: String,
    required: true,
    unique: true,
    lowercase: true,
    trim: true,
  },
  password: {
    type: String,
    required: true,
    minlength: 6,
  },
  role: {
    type: String,
    enum: ['ADMIN', 'USER', 'MODERATOR'],
    default: 'USER',
  },
  profile: {
    bio: String,
    avatar: String,
    website: String,
    location: String,
  },
}, {
  timestamps: true,
});

// Hash password ก่อน save
userSchema.pre('save', async function(next) {
  if (!this.isModified('password')) return next();
  this.password = await bcrypt.hash(this.password, 12);
  next();
});

// Method สำหรับเปรียบเทียบ password
userSchema.methods.comparePassword = async function(password) {
  return bcrypt.compare(password, this.password);
};

// ไม่ส่ง password ออกไปใน JSON
userSchema.set('toJSON', {
  transform: (doc, ret) => {
    delete ret.password;
    return ret;
  }
});

module.exports = mongoose.model('User', userSchema);
```

```javascript
// src/models/Post.js
const mongoose = require('mongoose');

const postSchema = new mongoose.Schema({
  title: {
    type: String,
    required: true,
    trim: true,
    maxlength: 200,
  },
  content: {
    type: String,
    required: true,
  },
  status: {
    type: String,
    enum: ['DRAFT', 'PUBLISHED', 'ARCHIVED'],
    default: 'DRAFT',
  },
  author: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User',
    required: true,
  },
  tags: [String],
  viewCount: {
    type: Number,
    default: 0,
  },
}, {
  timestamps: true,
});

// Index สำหรับ search
postSchema.index({ title: 'text', content: 'text' });
postSchema.index({ author: 1, status: 1 });

module.exports = mongoose.model('Post', postSchema);
```

---

## Queries - การดึงข้อมูล

### ขั้นตอนที่ 587: Query Resolvers

```javascript
// src/schema/resolvers/queries.js
const User = require('../../models/User');
const Post = require('../../models/Post');
const { AuthenticationError, UserInputError } = require('apollo-server-express');

const queryResolvers = {
  Query: {
    // ดึงข้อมูล user ปัจจุบัน
    me: async (parent, args, context) => {
      if (!context.user) {
        throw new AuthenticationError('ต้องเข้าสู่ระบบก่อน');
      }
      return User.findById(context.user.id);
    },

    // ดึงข้อมูล user ตาม ID
    user: async (parent, { id }, context) => {
      const user = await User.findById(id);
      if (!user) {
        throw new UserInputError('ไม่พบผู้ใช้', {
          argumentName: 'id',
        });
      }
      return user;
    },

    // ดึงรายการ users พร้อม pagination
    users: async (parent, { limit, offset, search }) => {
      const query = {};

      if (search) {
        query.$or = [
          { username: { $regex: search, $options: 'i' } },
          { email: { $regex: search, $options: 'i' } },
        ];
      }

      return User.find(query)
        .limit(limit)
        .skip(offset)
        .sort({ createdAt: -1 });
    },

    // Cursor-based pagination
    usersConnection: async (parent, { first = 10, after }) => {
      const query = {};

      if (after) {
        // Decode cursor (base64 encoded ID)
        const cursorId = Buffer.from(after, 'base64').toString('utf8');
        query._id = { $gt: cursorId };
      }

      const users = await User.find(query)
        .limit(first + 1)
        .sort({ _id: 1 });

      const hasNextPage = users.length > first;
      const edges = hasNextPage ? users.slice(0, -1) : users;

      const totalCount = await User.countDocuments({});

      return {
        edges: edges.map(user => ({
          node: user,
          cursor: Buffer.from(user._id.toString()).toString('base64'),
        })),
        pageInfo: {
          hasNextPage,
          hasPreviousPage: !!after,
          startCursor: edges.length > 0
            ? Buffer.from(edges[0]._id.toString()).toString('base64')
            : null,
          endCursor: edges.length > 0
            ? Buffer.from(edges[edges.length - 1]._id.toString()).toString('base64')
            : null,
          totalCount,
        },
      };
    },

    // ดึง post ตาม ID
    post: async (parent, { id }) => {
      const post = await Post.findByIdAndUpdate(
        id,
        { $inc: { viewCount: 1 } },
        { new: true }
      );

      if (!post) {
        throw new UserInputError('ไม่พบบทความ', {
          argumentName: 'id',
        });
      }

      return post;
    },

    // ดึงรายการ posts
    posts: async (parent, { authorId, status, limit, offset }) => {
      const query = {};
      if (authorId) query.author = authorId;
      if (status) query.status = status;

      return Post.find(query)
        .limit(limit)
        .skip(offset)
        .sort({ createdAt: -1 });
    },
  },
};

module.exports = queryResolvers;
```

### ขั้นตอนที่ 588: ตัวอย่าง GraphQL Queries

```graphql
# ดึงข้อมูล user ปัจจุบัน
query GetMe {
  me {
    id
    username
    email
    role
    profile {
      bio
      avatar
    }
    posts {
      id
      title
      status
    }
  }
}

# ดึง user เฉพาะ
query GetUser($id: ID!) {
  user(id: $id) {
    id
    username
    email
    createdAt
  }
}

# ค้นหา users
query SearchUsers($search: String, $limit: Int) {
  users(search: $search, limit: $limit) {
    id
    username
    email
  }
}

# Cursor-based pagination
query GetUsersPage($first: Int, $after: String) {
  usersConnection(first: $first, after: $after) {
    edges {
      cursor
      node {
        id
        username
        email
      }
    }
    pageInfo {
      hasNextPage
      endCursor
      totalCount
    }
  }
}

# ดึง posts ของ author
query GetAuthorPosts($authorId: ID!, $status: PostStatus) {
  posts(authorId: $authorId, status: $status) {
    id
    title
    status
    viewCount
    author {
      username
    }
    tags
    createdAt
  }
}
```

---

## Mutations - การเปลี่ยนแปลงข้อมูล

### ขั้นตอนที่ 589: Mutation Resolvers

```javascript
// src/schema/resolvers/mutations.js
const jwt = require('jsonwebtoken');
const User = require('../../models/User');
const Post = require('../../models/Post');
const {
  AuthenticationError,
  UserInputError,
  ForbiddenError,
} = require('apollo-server-express');

const mutationResolvers = {
  Mutation: {
    // สมัครสมาชิก
    register: async (parent, { input }) => {
      const { username, email, password, role } = input;

      // ตรวจสอบว่ามี user อยู่แล้วหรือไม่
      const existingUser = await User.findOne({
        $or: [{ email }, { username }]
      });

      if (existingUser) {
        throw new UserInputError('มีผู้ใช้นี้อยู่แล้ว', {
          invalidArgs: existingUser.email === email ? 'email' : 'username',
        });
      }

      const user = new User({ username, email, password, role });
      await user.save();

      const token = jwt.sign(
        { id: user._id, role: user.role },
        process.env.JWT_SECRET,
        { expiresIn: '7d' }
      );

      return { token, user };
    },

    // เข้าสู่ระบบ
    login: async (parent, { email, password }) => {
      const user = await User.findOne({ email });

      if (!user || !(await user.comparePassword(password))) {
        throw new AuthenticationError('อีเมลหรือรหัสผ่านไม่ถูกต้อง');
      }

      const token = jwt.sign(
        { id: user._id, role: user.role },
        process.env.JWT_SECRET,
        { expiresIn: '7d' }
      );

      return { token, user };
    },

    // อัปเดต user
    updateUser: async (parent, { id, input }, context) => {
      if (!context.user) {
        throw new AuthenticationError('ต้องเข้าสู่ระบบก่อน');
      }

      // ตรวจสอบสิทธิ์
      if (context.user.id !== id && context.user.role !== 'ADMIN') {
        throw new ForbiddenError('ไม่มีสิทธิ์แก้ไขข้อมูลของผู้ใช้อื่น');
      }

      const user = await User.findByIdAndUpdate(
        id,
        { $set: input },
        { new: true, runValidators: true }
      );

      if (!user) {
        throw new UserInputError('ไม่พบผู้ใช้');
      }

      return user;
    },

    // ลบ user
    deleteUser: async (parent, { id }, context) => {
      if (!context.user) {
        throw new AuthenticationError('ต้องเข้าสู่ระบบก่อน');
      }

      if (context.user.role !== 'ADMIN') {
        throw new ForbiddenError('เฉพาะ Admin เท่านั้น');
      }

      const user = await User.findByIdAndDelete(id);
      if (!user) {
        throw new UserInputError('ไม่พบผู้ใช้');
      }

      // ลบ posts ของ user ด้วย
      await Post.deleteMany({ author: id });

      return true;
    },

    // สร้าง post
    createPost: async (parent, { input }, context) => {
      if (!context.user) {
        throw new AuthenticationError('ต้องเข้าสู่ระบบก่อน');
      }

      const post = new Post({
        ...input,
        author: context.user.id,
      });

      await post.save();
      return post;
    },

    // อัปเดต post
    updatePost: async (parent, { id, input }, context) => {
      if (!context.user) {
        throw new AuthenticationError('ต้องเข้าสู่ระบบก่อน');
      }

      const post = await Post.findById(id);
      if (!post) {
        throw new UserInputError('ไม่พบบทความ');
      }

      // ตรวจสอบ ownership
      if (
        post.author.toString() !== context.user.id &&
        context.user.role !== 'ADMIN'
      ) {
        throw new ForbiddenError('ไม่มีสิทธิ์แก้ไขบทความนี้');
      }

      return Post.findByIdAndUpdate(id, { $set: input }, { new: true });
    },

    // เผยแพร่ post
    publishPost: async (parent, { id }, context) => {
      if (!context.user) {
        throw new AuthenticationError('ต้องเข้าสู่ระบบก่อน');
      }

      const post = await Post.findById(id);
      if (!post) {
        throw new UserInputError('ไม่พบบทความ');
      }

      if (post.author.toString() !== context.user.id) {
        throw new ForbiddenError('ไม่มีสิทธิ์');
      }

      if (post.status === 'PUBLISHED') {
        throw new UserInputError('บทความนี้เผยแพร่แล้ว');
      }

      return Post.findByIdAndUpdate(
        id,
        { status: 'PUBLISHED' },
        { new: true }
      );
    },

    // ลบ post
    deletePost: async (parent, { id }, context) => {
      if (!context.user) {
        throw new AuthenticationError('ต้องเข้าสู่ระบบก่อน');
      }

      const post = await Post.findById(id);
      if (!post) {
        throw new UserInputError('ไม่พบบทความ');
      }

      if (
        post.author.toString() !== context.user.id &&
        context.user.role !== 'ADMIN'
      ) {
        throw new ForbiddenError('ไม่มีสิทธิ์ลบบทความนี้');
      }

      await Post.findByIdAndDelete(id);
      return true;
    },
  },
};

module.exports = mutationResolvers;
```

### ขั้นตอนที่ 590: ตัวอย่าง Mutations

```graphql
# สมัครสมาชิก
mutation Register {
  register(input: {
    username: "johndoe"
    email: "john@example.com"
    password: "secret123"
  }) {
    token
    user {
      id
      username
      email
      role
    }
  }
}

# เข้าสู่ระบบ
mutation Login($email: String!, $password: String!) {
  login(email: $email, password: $password) {
    token
    user {
      id
      username
      role
    }
  }
}

# สร้างบทความ
mutation CreatePost($input: CreatePostInput!) {
  createPost(input: $input) {
    id
    title
    status
    author {
      username
    }
    createdAt
  }
}

# Variables สำหรับ CreatePost
# {
#   "input": {
#     "title": "บทความแรกของฉัน",
#     "content": "เนื้อหาบทความ...",
#     "tags": ["nodejs", "graphql"]
#   }
# }
```

---

## Resolvers

### ขั้นตอนที่ 591: Field Resolvers

Field resolvers ใช้สำหรับ resolve fields ที่ต้องการ logic พิเศษ

```javascript
// src/schema/resolvers/types.js
const User = require('../../models/User');
const Post = require('../../models/Post');

const typeResolvers = {
  // User type resolvers
  User: {
    // resolve posts ของ user
    posts: async (parent, args, context) => {
      return Post.find({ author: parent._id })
        .sort({ createdAt: -1 });
    },

    // แปลง _id → id
    id: (parent) => parent._id.toString(),

    // สร้าง virtual field
    postsCount: async (parent) => {
      return Post.countDocuments({ author: parent._id });
    },
  },

  // Post type resolvers
  Post: {
    // Populate author
    author: async (parent, args, context) => {
      // ถ้า author ถูก populate แล้ว ให้ return ตรงๆ
      if (parent.author && parent.author.username) {
        return parent.author;
      }
      // ถ้าไม่ ให้ query จาก database
      return User.findById(parent.author);
    },

    id: (parent) => parent._id.toString(),

    // Format content preview
    contentPreview: (parent) => {
      if (parent.content.length <= 200) return parent.content;
      return parent.content.substring(0, 200) + '...';
    },
  },
};

module.exports = typeResolvers;
```

### ขั้นตอนที่ 592: รวม Resolvers

```javascript
// src/schema/resolvers.js
const queryResolvers = require('./resolvers/queries');
const mutationResolvers = require('./resolvers/mutations');
const typeResolvers = require('./resolvers/types');
const { DateTimeScalar } = require('./scalars');

const resolvers = {
  DateTime: DateTimeScalar,
  ...queryResolvers,
  ...mutationResolvers,
  ...typeResolvers,
};

module.exports = { resolvers };
```

---

## Arguments และ Variables

### ขั้นตอนที่ 593: การใช้งาน Variables

```graphql
# กำหนด operation name และ variable types
query GetPosts(
  $authorId: ID
  $status: PostStatus
  $limit: Int = 10
  $offset: Int = 0
) {
  posts(
    authorId: $authorId
    status: $status
    limit: $limit
    offset: $offset
  ) {
    id
    title
    status
    author {
      username
    }
    viewCount
  }
}

# ใช้ Default Values
query GetPublishedPosts {
  posts(status: PUBLISHED) {
    id
    title
  }
}
```

```javascript
// ส่ง variables จาก client
const { ApolloClient, gql, InMemoryCache } = require('@apollo/client');

const client = new ApolloClient({
  uri: 'http://localhost:4000/graphql',
  cache: new InMemoryCache(),
  headers: {
    Authorization: `Bearer ${token}`,
  },
});

// Query พร้อม variables
const result = await client.query({
  query: gql`
    query GetPosts($status: PostStatus, $limit: Int) {
      posts(status: $status, limit: $limit) {
        id
        title
        viewCount
      }
    }
  `,
  variables: {
    status: 'PUBLISHED',
    limit: 5,
  },
});
```

---

## Fragments

### ขั้นตอนที่ 594: การใช้งาน Fragments

Fragment คือการแยก fields ที่ใช้ซ้ำๆ ออกมาเป็น reusable piece

```graphql
# กำหนด Fragment
fragment UserBasic on User {
  id
  username
  email
  role
}

fragment UserFull on User {
  ...UserBasic
  profile {
    bio
    avatar
    website
  }
  createdAt
}

fragment PostBasic on Post {
  id
  title
  status
  viewCount
  createdAt
}

# ใช้ Fragment ใน Query
query GetUserWithPosts($id: ID!) {
  user(id: $id) {
    ...UserFull
    posts {
      ...PostBasic
      author {
        ...UserBasic
      }
    }
  }
}

query GetMe {
  me {
    ...UserFull
  }
}
```

### ขั้นตอนที่ 595: Inline Fragments

```graphql
# Union Types
union SearchResult = User | Post

type Query {
  search(query: String!): [SearchResult!]!
}

# การใช้ Inline Fragments กับ Union
query Search($query: String!) {
  search(query: $query) {
    __typename
    ... on User {
      id
      username
      email
    }
    ... on Post {
      id
      title
      status
    }
  }
}
```

```javascript
// Resolver สำหรับ Union
const resolvers = {
  SearchResult: {
    __resolveType(obj) {
      if (obj.username) return 'User';
      if (obj.title) return 'Post';
      return null;
    },
  },

  Query: {
    search: async (parent, { query }) => {
      const users = await User.find({
        username: { $regex: query, $options: 'i' }
      });
      const posts = await Post.find({
        title: { $regex: query, $options: 'i' }
      });
      return [...users, ...posts];
    },
  },
};
```

---

## Error Handling

### ขั้นตอนที่ 596: Custom Errors

```javascript
// src/errors/index.js
const { ApolloError } = require('apollo-server-express');

// Custom Error Classes
class DatabaseError extends ApolloError {
  constructor(message) {
    super(message, 'DATABASE_ERROR');
    Object.defineProperty(this, 'name', { value: 'DatabaseError' });
  }
}

class ValidationError extends ApolloError {
  constructor(message, fields) {
    super(message, 'VALIDATION_ERROR', { fields });
    Object.defineProperty(this, 'name', { value: 'ValidationError' });
  }
}

class NotFoundError extends ApolloError {
  constructor(resource) {
    super(`ไม่พบ ${resource}`, 'NOT_FOUND', { resource });
    Object.defineProperty(this, 'name', { value: 'NotFoundError' });
  }
}

module.exports = { DatabaseError, ValidationError, NotFoundError };
```

### ขั้นตอนที่ 597: Error Formatting

```javascript
// src/index.js - Error formatting configuration
const server = new ApolloServer({
  typeDefs,
  resolvers,

  // Format errors ก่อน send ไปยัง client
  formatError: (error) => {
    // Log error ทั้งหมด
    console.error('GraphQL Error:', {
      message: error.message,
      code: error.extensions?.code,
      path: error.path,
      locations: error.locations,
    });

    // ซ่อน internal errors ใน production
    if (process.env.NODE_ENV === 'production') {
      if (error.extensions?.code === 'INTERNAL_SERVER_ERROR') {
        return new Error('เกิดข้อผิดพลาดภายในระบบ');
      }
    }

    // Return original error สำหรับ known errors
    return error;
  },
});
```

### ขั้นตอนที่ 598: Authentication Middleware

```javascript
// src/middleware/auth.js
const jwt = require('jsonwebtoken');
const User = require('../models/User');

async function getUser(token) {
  if (!token) return null;

  try {
    // ลบ "Bearer " prefix
    const cleanToken = token.replace('Bearer ', '');
    const decoded = jwt.verify(cleanToken, process.env.JWT_SECRET);

    const user = await User.findById(decoded.id);
    return user;
  } catch (error) {
    return null;
  }
}

// Context function
async function createContext({ req }) {
  const token = req.headers.authorization || '';
  const user = await getUser(token);

  return {
    user,
    // เพิ่ม helpers
    isAuthenticated: !!user,
    isAdmin: user?.role === 'ADMIN',
  };
}

module.exports = { createContext };
```

---

## Introspection และ Schema Documentation

### ขั้นตอนที่ 599: Schema Documentation

```javascript
// เพิ่ม descriptions ใน schema
const typeDefs = gql`
  """
  User ในระบบ
  สามารถเป็น admin, user หรือ moderator
  """
  type User {
    "ID ไม่ซ้ำของ user"
    id: ID!

    "ชื่อผู้ใช้ (unique)"
    username: String!

    "อีเมล (unique)"
    email: String!

    "บทบาทของ user"
    role: Role!

    "โปรไฟล์ user"
    profile: Profile

    "บทความทั้งหมดของ user"
    posts: [Post!]!

    "วันที่สร้าง"
    createdAt: DateTime!
  }
`;
```

### ขั้นตอนที่ 600: Testing GraphQL API

```javascript
// src/__tests__/graphql.test.js
const { createTestClient } = require('apollo-server-testing');
const { ApolloServer } = require('apollo-server-express');
const { typeDefs } = require('../schema/typeDefs');
const { resolvers } = require('../schema/resolvers');

describe('GraphQL API', () => {
  let server;
  let query;
  let mutate;

  beforeAll(async () => {
    server = new ApolloServer({
      typeDefs,
      resolvers,
      context: () => ({
        user: { id: 'test-user-id', role: 'USER' }
      }),
    });

    const testClient = createTestClient(server);
    query = testClient.query;
    mutate = testClient.mutate;
  });

  it('should get posts', async () => {
    const GET_POSTS = `
      query {
        posts {
          id
          title
          status
        }
      }
    `;

    const result = await query({ query: GET_POSTS });
    expect(result.errors).toBeUndefined();
    expect(result.data.posts).toBeDefined();
    expect(Array.isArray(result.data.posts)).toBe(true);
  });

  it('should create a post', async () => {
    const CREATE_POST = `
      mutation CreatePost($input: CreatePostInput!) {
        createPost(input: $input) {
          id
          title
          status
        }
      }
    `;

    const result = await mutate({
      mutation: CREATE_POST,
      variables: {
        input: {
          title: 'Test Post',
          content: 'Test content',
        }
      }
    });

    expect(result.errors).toBeUndefined();
    expect(result.data.createPost.title).toBe('Test Post');
    expect(result.data.createPost.status).toBe('DRAFT');
  });
});
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: สร้าง Comment System

สร้าง GraphQL schema และ resolvers สำหรับระบบ comments:

```graphql
# TODO: เพิ่ม types เหล่านี้
type Comment {
  id: ID!
  content: String!
  author: User!
  post: Post!
  likes: Int!
  createdAt: DateTime!
}

# TODO: เพิ่ม queries
extend type Query {
  comments(postId: ID!): [Comment!]!
}

# TODO: เพิ่ม mutations
extend type Mutation {
  addComment(postId: ID!, content: String!): Comment!
  deleteComment(id: ID!): Boolean!
  likeComment(id: ID!): Comment!
}
```

### แบบฝึกหัดที่ 2: Implement Search

```javascript
// TODO: implement full-text search
const searchResolver = async (parent, { query }, context) => {
  // ค้นหา users และ posts พร้อมกัน
  const [users, posts] = await Promise.all([
    User.find({ $text: { $search: query } }),
    Post.find({ $text: { $search: query }, status: 'PUBLISHED' }),
  ]);

  return [...users, ...posts];
};
```

### แบบฝึกหัดที่ 3: Rate Limiting

```javascript
// TODO: เพิ่ม rate limiting สำหรับ mutations
const { createRateLimitRule } = require('graphql-rate-limit');

const rateLimitRule = createRateLimitRule({
  identifyContext: (ctx) => ctx.user?.id || ctx.ip,
});

// ใช้กับ directive
// @rateLimit(max: 10, window: "1m")
```

### แบบฝึกหัดที่ 4: File Upload

```javascript
// TODO: เพิ่ม file upload ใน GraphQL
// ใช้ apollo-upload-server
const { GraphQLUpload } = require('graphql-upload');

// Scalar
scalar Upload

// Mutation
type Mutation {
  uploadAvatar(file: Upload!): User!
}
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **GraphQL คืออะไร** - query language สำหรับ API ที่แก้ปัญหา over/underfetching
2. **Schema Definition** - การกำหนด types, queries, mutations ด้วย SDL
3. **Resolvers** - การ implement logic สำหรับ resolve ข้อมูล
4. **Authentication** - การใช้ context และ JWT
5. **Error Handling** - custom errors และ error formatting
6. **Testing** - การทดสอบ GraphQL API

ในบทถัดไปเราจะเรียนรู้ **Advanced GraphQL** - DataLoader, Subscriptions, Federation และ Authentication แบบ advanced
