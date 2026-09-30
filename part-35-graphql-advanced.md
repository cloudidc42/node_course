# Part 35 | ขั้นตอนที่ 601-620 จาก 1000

## Advanced GraphQL - GraphQL ขั้นสูง

---

## สารบัญ

1. [DataLoader - แก้ปัญหา N+1](#dataloader---แก้ปัญหา-n1)
2. [GraphQL Subscriptions](#graphql-subscriptions)
3. [Schema Stitching และ Federation](#schema-stitching-และ-federation)
4. [Directives](#directives)
5. [Persisted Queries](#persisted-queries)
6. [Caching Strategies](#caching-strategies)
7. [Authentication และ Authorization](#authentication-และ-authorization)
8. [Performance Optimization](#performance-optimization)
9. [แบบฝึกหัด](#แบบฝึกหัด)

---

## DataLoader - แก้ปัญหา N+1

### ขั้นตอนที่ 601: ทำความเข้าใจปัญหา N+1

```
ปัญหา N+1 Query:

Query: posts { author { name } }

1. Query ดึง 10 posts → 1 query
2. สำหรับแต่ละ post ดึง author → 10 queries

รวม: 11 queries แทนที่จะเป็น 2 queries!
```

```javascript
// ❌ ปัญหา N+1 - ทุก post จะ query author แยกกัน
const postResolvers = {
  Post: {
    author: async (post) => {
      // ถ้ามี 100 posts จะเกิด 100 queries!
      return User.findById(post.author);
    },
  },
};
```

### ขั้นตอนที่ 602: ติดตั้งและตั้งค่า DataLoader

```bash
npm install dataloader
```

```javascript
// src/dataloaders/index.js
const DataLoader = require('dataloader');
const User = require('../models/User');
const Post = require('../models/Post');

// UserLoader - batch load users by IDs
function createUserLoader() {
  return new DataLoader(async (userIds) => {
    console.log(`Batching ${userIds.length} user queries`);

    // Query users ทั้งหมดในครั้งเดียว
    const users = await User.find({ _id: { $in: userIds } });

    // Map ให้ตรงกับ order ของ userIds
    const userMap = {};
    users.forEach(user => {
      userMap[user._id.toString()] = user;
    });

    // ต้องคืนค่าในลำดับเดียวกับ input
    return userIds.map(id => userMap[id.toString()] || null);
  });
}

// PostLoader - batch load posts by authorIds
function createPostsByAuthorLoader() {
  return new DataLoader(async (authorIds) => {
    const posts = await Post.find({
      author: { $in: authorIds }
    });

    // Group posts by author
    const postsByAuthor = {};
    posts.forEach(post => {
      const authorId = post.author.toString();
      if (!postsByAuthor[authorId]) {
        postsByAuthor[authorId] = [];
      }
      postsByAuthor[authorId].push(post);
    });

    return authorIds.map(id => postsByAuthor[id.toString()] || []);
  });
}

// สร้าง loaders ต่อ request
function createLoaders() {
  return {
    userLoader: createUserLoader(),
    postsByAuthorLoader: createPostsByAuthorLoader(),
  };
}

module.exports = { createLoaders };
```

### ขั้นตอนที่ 603: ใช้ DataLoader ใน Resolvers

```javascript
// ใน context function
const { createLoaders } = require('./dataloaders');

async function createContext({ req }) {
  const user = await getUser(req.headers.authorization);

  return {
    user,
    // สร้าง loaders ใหม่ต่อ request (ไม่ share ข้าม requests)
    loaders: createLoaders(),
  };
}

// ✅ ใช้ DataLoader
const postResolvers = {
  Post: {
    // ใช้ loader แทน direct query
    author: async (post, args, { loaders }) => {
      return loaders.userLoader.load(post.author.toString());
    },
  },

  User: {
    posts: async (user, args, { loaders }) => {
      return loaders.postsByAuthorLoader.load(user._id.toString());
    },
  },
};

// ผลลัพธ์: 100 posts = เพียง 2 queries (1 สำหรับ posts, 1 สำหรับ users)
```

### ขั้นตอนที่ 604: Advanced DataLoader Patterns

```javascript
// Cache DataLoader
function createCachedUserLoader() {
  const cache = new Map();

  return new DataLoader(
    async (userIds) => {
      const users = await User.find({ _id: { $in: userIds } });
      const userMap = {};
      users.forEach(u => (userMap[u._id.toString()] = u));
      return userIds.map(id => userMap[id.toString()] || null);
    },
    {
      // Cache ภายใน request
      cacheKeyFn: (key) => key.toString(),
      cache: true,

      // Batch options
      maxBatchSize: 100,
      batchScheduleFn: (callback) => setTimeout(callback, 0),
    }
  );
}

// DataLoader พร้อม Prime Cache
async function getUserWithPrime(loader, id) {
  // Prime the cache ด้วยข้อมูลที่รู้แล้ว
  const user = await User.findById(id);
  if (user) {
    loader.prime(id.toString(), user);
  }
  return user;
}
```

---

## GraphQL Subscriptions

### ขั้นตอนที่ 605: ตั้งค่า Subscriptions

```bash
npm install graphql-subscriptions subscriptions-transport-ws ws
```

```javascript
// src/index.js - เพิ่ม Subscription support
const { ApolloServer } = require('apollo-server-express');
const { SubscriptionServer } = require('subscriptions-transport-ws');
const { execute, subscribe } = require('graphql');
const { makeExecutableSchema } = require('@graphql-tools/schema');
const http = require('http');
const express = require('express');

async function startServer() {
  const app = express();
  const httpServer = http.createServer(app);

  const schema = makeExecutableSchema({ typeDefs, resolvers });

  const server = new ApolloServer({
    schema,
    plugins: [{
      async serverWillStart() {
        return {
          async drainServer() {
            subscriptionServer.close();
          }
        };
      }
    }],
  });

  const subscriptionServer = SubscriptionServer.create({
    schema,
    execute,
    subscribe,
    onConnect: (connectionParams) => {
      // ตรวจสอบ auth สำหรับ subscriptions
      const token = connectionParams.Authorization;
      const user = verifyToken(token);
      return { user };
    },
  }, {
    server: httpServer,
    path: server.graphqlPath,
  });

  await server.start();
  server.applyMiddleware({ app });

  httpServer.listen(4000, () => {
    console.log('🚀 Server ready at http://localhost:4000/graphql');
    console.log('🔌 Subscriptions at ws://localhost:4000/graphql');
  });
}
```

### ขั้นตอนที่ 606: Subscription Schema

```javascript
// เพิ่ม Subscription type ใน schema
const typeDefs = gql`
  type Subscription {
    # ติดตาม posts ใหม่
    postCreated: Post!

    # ติดตาม post ที่ถูก update
    postUpdated(id: ID!): Post!

    # ติดตาม comments ใหม่ใน post
    commentAdded(postId: ID!): Comment!

    # Online users
    userOnline: UserStatus!

    # Notification
    notificationReceived(userId: ID!): Notification!
  }

  type UserStatus {
    user: User!
    isOnline: Boolean!
    lastSeen: DateTime
  }

  type Notification {
    id: ID!
    type: NotificationType!
    message: String!
    data: JSON
    read: Boolean!
    createdAt: DateTime!
  }

  enum NotificationType {
    NEW_COMMENT
    NEW_LIKE
    NEW_FOLLOWER
    MENTION
  }
`;
```

### ขั้นตอนที่ 607: PubSub และ Subscription Resolvers

```javascript
// src/pubsub.js
const { PubSub } = require('graphql-subscriptions');
const pubsub = new PubSub();

// Events
const EVENTS = {
  POST_CREATED: 'POST_CREATED',
  POST_UPDATED: 'POST_UPDATED',
  COMMENT_ADDED: 'COMMENT_ADDED',
  USER_STATUS_CHANGED: 'USER_STATUS_CHANGED',
  NOTIFICATION_RECEIVED: 'NOTIFICATION_RECEIVED',
};

module.exports = { pubsub, EVENTS };
```

```javascript
// src/schema/resolvers/subscriptions.js
const { pubsub, EVENTS } = require('../../pubsub');
const { withFilter } = require('graphql-subscriptions');

const subscriptionResolvers = {
  Subscription: {
    // Subscription ง่ายๆ
    postCreated: {
      subscribe: () => pubsub.asyncIterator([EVENTS.POST_CREATED]),
    },

    // Filtered Subscription
    postUpdated: {
      subscribe: withFilter(
        () => pubsub.asyncIterator([EVENTS.POST_UPDATED]),
        (payload, variables) => {
          // ส่งเฉพาะ post ที่ตรงกับ id ที่ subscribe
          return payload.postUpdated.id.toString() === variables.id;
        }
      ),
    },

    // Comment subscription with filter
    commentAdded: {
      subscribe: withFilter(
        () => pubsub.asyncIterator([EVENTS.COMMENT_ADDED]),
        (payload, variables) => {
          return payload.commentAdded.post.toString() === variables.postId;
        }
      ),
    },

    // Notification สำหรับ user เฉพาะ
    notificationReceived: {
      subscribe: withFilter(
        () => pubsub.asyncIterator([EVENTS.NOTIFICATION_RECEIVED]),
        (payload, variables, context) => {
          // ตรวจสอบ auth
          if (!context.user) return false;
          return payload.notificationReceived.userId === variables.userId;
        }
      ),
    },
  },
};

module.exports = subscriptionResolvers;
```

### ขั้นตอนที่ 608: Publish Events จาก Mutations

```javascript
// ใน mutation resolvers
const { pubsub, EVENTS } = require('../../pubsub');

const mutationResolvers = {
  Mutation: {
    createPost: async (parent, { input }, context) => {
      if (!context.user) throw new AuthenticationError('ต้องเข้าสู่ระบบ');

      const post = new Post({ ...input, author: context.user.id });
      await post.save();
      await post.populate('author');

      // Publish event
      pubsub.publish(EVENTS.POST_CREATED, {
        postCreated: post,
      });

      return post;
    },

    addComment: async (parent, { postId, content }, context) => {
      if (!context.user) throw new AuthenticationError('ต้องเข้าสู่ระบบ');

      const comment = new Comment({
        content,
        author: context.user.id,
        post: postId,
      });
      await comment.save();
      await comment.populate(['author', 'post']);

      // Publish comment event
      pubsub.publish(EVENTS.COMMENT_ADDED, {
        commentAdded: comment,
      });

      return comment;
    },
  },
};
```

### ขั้นตอนที่ 609: Redis PubSub สำหรับ Production

```bash
npm install graphql-redis-subscriptions ioredis
```

```javascript
// src/pubsub.js - Redis PubSub สำหรับ scale
const { RedisPubSub } = require('graphql-redis-subscriptions');
const Redis = require('ioredis');

const options = {
  host: process.env.REDIS_HOST || 'localhost',
  port: parseInt(process.env.REDIS_PORT) || 6379,
  password: process.env.REDIS_PASSWORD,
  retryStrategy: (times) => Math.min(times * 50, 2000),
};

const pubsub = new RedisPubSub({
  publisher: new Redis(options),
  subscriber: new Redis(options),
});

module.exports = { pubsub };
```

---

## Schema Stitching และ Federation

### ขั้นตอนที่ 610: Apollo Federation

Federation ช่วยให้แบ่ง GraphQL schema ออกเป็นหลาย services

```bash
npm install @apollo/federation @apollo/gateway
```

```javascript
// user-service/src/schema.js - User Service
const { gql, buildFederatedSchema } = require('@apollo/federation');

const typeDefs = gql`
  type User @key(fields: "id") {
    id: ID!
    username: String!
    email: String!
    role: String!
  }

  extend type Query {
    me: User
    user(id: ID!): User
    users: [User!]!
  }

  extend type Mutation {
    register(input: CreateUserInput!): AuthPayload!
    login(email: String!, password: String!): AuthPayload!
  }

  input CreateUserInput {
    username: String!
    email: String!
    password: String!
  }

  type AuthPayload {
    token: String!
    user: User!
  }
`;

const resolvers = {
  User: {
    // Reference resolver - ให้ services อื่น load User
    __resolveReference: async (reference) => {
      return User.findById(reference.id);
    },
  },
  Query: {
    me: async (_, __, { user }) => {
      if (!user) throw new AuthenticationError('ต้องเข้าสู่ระบบ');
      return User.findById(user.id);
    },
    user: (_, { id }) => User.findById(id),
    users: () => User.find(),
  },
};

module.exports = buildFederatedSchema([{ typeDefs, resolvers }]);
```

```javascript
// post-service/src/schema.js - Post Service
const typeDefs = gql`
  extend type User @key(fields: "id") {
    id: ID! @external
    posts: [Post!]!  # เพิ่ม field ให้ User
  }

  type Post @key(fields: "id") {
    id: ID!
    title: String!
    content: String!
    author: User!
  }

  extend type Query {
    post(id: ID!): Post
    posts: [Post!]!
  }
`;

const resolvers = {
  User: {
    posts: async (user) => Post.find({ author: user.id }),
  },
  Post: {
    __resolveReference: (reference) => Post.findById(reference.id),
    author: (post) => ({ __typename: 'User', id: post.author }),
  },
};
```

### ขั้นตอนที่ 611: API Gateway

```javascript
// gateway/src/index.js
const { ApolloServer } = require('apollo-server');
const { ApolloGateway, RemoteGraphQLDataSource } = require('@apollo/gateway');

// Custom DataSource สำหรับ forward auth headers
class AuthenticatedDataSource extends RemoteGraphQLDataSource {
  willSendRequest({ request, context }) {
    // Forward Authorization header ไปยัง services
    request.http.headers.set(
      'Authorization',
      context.token ? `Bearer ${context.token}` : ''
    );
  }
}

const gateway = new ApolloGateway({
  serviceList: [
    { name: 'users', url: 'http://user-service:4001/graphql' },
    { name: 'posts', url: 'http://post-service:4002/graphql' },
    { name: 'comments', url: 'http://comment-service:4003/graphql' },
  ],
  buildService({ url }) {
    return new AuthenticatedDataSource({ url });
  },
});

const server = new ApolloServer({
  gateway,
  subscriptions: false,
  context: ({ req }) => ({
    token: req.headers.authorization?.replace('Bearer ', ''),
  }),
});

server.listen(4000).then(({ url }) => {
  console.log(`🚀 Gateway ready at ${url}`);
});
```

---

## Directives

### ขั้นตอนที่ 612: Custom Directives

```javascript
// src/directives/auth.js
const { SchemaDirectiveVisitor } = require('@graphql-tools/utils');
const { defaultFieldResolver } = require('graphql');
const { AuthenticationError, ForbiddenError } = require('apollo-server-express');

class AuthDirective extends SchemaDirectiveVisitor {
  visitObject(object) {
    this.ensureFieldsWrapped(object);
    object._requiredAuthRole = this.args.requires;
  }

  visitFieldDefinition(field, details) {
    const { resolve = defaultFieldResolver } = field;
    const requiredRole = this.args.requires;

    field.resolve = async function(source, args, context, info) {
      if (!context.user) {
        throw new AuthenticationError('ต้องเข้าสู่ระบบก่อน');
      }

      if (requiredRole && context.user.role !== requiredRole) {
        throw new ForbiddenError(`ต้องมีบทบาท ${requiredRole}`);
      }

      return resolve.call(this, source, args, context, info);
    };
  }

  ensureFieldsWrapped(objectType) {
    if (objectType._authFieldsWrapped) return;
    objectType._authFieldsWrapped = true;

    const fields = objectType.getFields();
    Object.values(fields).forEach(field => {
      const { resolve = defaultFieldResolver } = field;
      field.resolve = async function(source, args, context, info) {
        const requiredRole = field._requiredAuthRole || objectType._requiredAuthRole;

        if (!context.user) {
          throw new AuthenticationError('ต้องเข้าสู่ระบบก่อน');
        }

        if (requiredRole && context.user.role !== requiredRole) {
          throw new ForbiddenError(`ต้องมีบทบาท ${requiredRole}`);
        }

        return resolve.call(this, source, args, context, info);
      };
    });
  }
}

module.exports = { AuthDirective };
```

```javascript
// ใช้ directive ใน schema
const typeDefs = gql`
  directive @auth(requires: Role = USER) on OBJECT | FIELD_DEFINITION

  type AdminPanel @auth(requires: ADMIN) {
    users: [User!]!
    reports: [Report!]!
    settings: Settings!
  }

  type Query {
    publicData: String
    privateData: String @auth
    adminData: String @auth(requires: ADMIN)
  }
`;
```

### ขั้นตอนที่ 613: Rate Limit Directive

```javascript
// src/directives/rateLimit.js
const { defaultFieldResolver } = require('graphql');
const { SchemaDirectiveVisitor } = require('@graphql-tools/utils');

class RateLimitDirective extends SchemaDirectiveVisitor {
  visitFieldDefinition(field) {
    const { resolve = defaultFieldResolver } = field;
    const { max, window } = this.args;

    // Simple in-memory rate limit (ใช้ Redis ใน production)
    const requestCounts = new Map();

    field.resolve = async function(source, args, context, info) {
      const key = `${context.user?.id || context.ip}:${info.fieldName}`;
      const now = Date.now();
      const windowMs = parseWindow(window);

      if (!requestCounts.has(key)) {
        requestCounts.set(key, []);
      }

      const requests = requestCounts.get(key).filter(
        time => now - time < windowMs
      );
      requests.push(now);
      requestCounts.set(key, requests);

      if (requests.length > max) {
        throw new Error(`Rate limit exceeded: สูงสุด ${max} requests ต่อ ${window}`);
      }

      return resolve.call(this, source, args, context, info);
    };
  }
}

function parseWindow(window) {
  const match = window.match(/^(\d+)(s|m|h)$/);
  if (!match) throw new Error('Invalid window format');

  const [, amount, unit] = match;
  const multipliers = { s: 1000, m: 60000, h: 3600000 };
  return parseInt(amount) * multipliers[unit];
}

module.exports = { RateLimitDirective };
```

---

## Persisted Queries

### ขั้นตอนที่ 614: Automatic Persisted Queries

```javascript
// ติดตั้ง
// npm install apollo-server-cache-redis keyv @keyv/redis

const { ApolloServer } = require('apollo-server-express');
const { BaseRedisCache } = require('apollo-server-cache-redis');
const Redis = require('ioredis');

const server = new ApolloServer({
  typeDefs,
  resolvers,

  // Cache สำหรับ persisted queries
  cache: new BaseRedisCache({
    client: new Redis({
      host: process.env.REDIS_HOST,
    }),
  }),

  // เปิด Automatic Persisted Queries
  persistedQueries: {
    ttl: 300, // 5 minutes
  },
});
```

---

## Caching Strategies

### ขั้นตอนที่ 615: Response Caching

```javascript
// npm install apollo-server-plugin-response-cache

const responseCachePlugin = require('apollo-server-plugin-response-cache');

const server = new ApolloServer({
  typeDefs,
  resolvers,
  plugins: [
    responseCachePlugin({
      // Cache สำหรับ authenticated users
      sessionId: (requestContext) => {
        return requestContext.request.http.headers.get('Authorization') || null;
      },
    }),
  ],
  cache: new BaseRedisCache({
    client: new Redis({ host: process.env.REDIS_HOST }),
  }),
});

// ใช้ @cacheControl directive ใน schema
const typeDefs = gql`
  type Query {
    # Cache เป็น public 10 วินาที
    publicPosts: [Post!]! @cacheControl(maxAge: 10, scope: PUBLIC)

    # Cache เป็น private 30 วินาที
    myPosts: [Post!]! @cacheControl(maxAge: 30, scope: PRIVATE)

    # ไม่ cache
    liveData: LiveData! @cacheControl(maxAge: 0)
  }
`;
```

---

## Authentication และ Authorization

### ขั้นตอนที่ 616: Role-Based Access Control

```javascript
// src/auth/permissions.js
const { shield, rule, and, or, not } = require('graphql-shield');

// กำหนด rules
const isAuthenticated = rule({ cache: 'contextual' })(
  async (parent, args, ctx) => {
    return ctx.user !== null && ctx.user !== undefined;
  }
);

const isAdmin = rule({ cache: 'contextual' })(
  async (parent, args, ctx) => {
    return ctx.user?.role === 'ADMIN';
  }
);

const isModerator = rule({ cache: 'contextual' })(
  async (parent, args, ctx) => {
    return ['ADMIN', 'MODERATOR'].includes(ctx.user?.role);
  }
);

const isPostOwner = rule({ cache: 'strict' })(
  async (parent, { id }, ctx) => {
    const post = await Post.findById(id);
    return post?.author.toString() === ctx.user?.id;
  }
);

// กำหนด permissions
const permissions = shield({
  Query: {
    me: isAuthenticated,
    users: isModerator,
    adminPanel: isAdmin,
  },
  Mutation: {
    createPost: isAuthenticated,
    updatePost: or(isAdmin, isPostOwner),
    deletePost: or(isAdmin, isPostOwner),
    deleteUser: isAdmin,
  },
}, {
  allowExternalErrors: true,
  fallbackError: 'ไม่มีสิทธิ์ดำเนินการนี้',
});

module.exports = { permissions };
```

### ขั้นตอนที่ 617: เพิ่ม Shield ใน Server

```javascript
// src/index.js
const { applyMiddleware } = require('graphql-middleware');
const { makeExecutableSchema } = require('@graphql-tools/schema');
const { permissions } = require('./auth/permissions');

const schema = makeExecutableSchema({ typeDefs, resolvers });
const schemaWithPermissions = applyMiddleware(schema, permissions);

const server = new ApolloServer({
  schema: schemaWithPermissions,
  context: createContext,
});
```

---

## Performance Optimization

### ขั้นตอนที่ 618: Query Complexity Analysis

```javascript
// ป้องกัน queries ที่ซับซ้อนเกินไป
const { createComplexityLimitRule } = require('graphql-validation-complexity');

const server = new ApolloServer({
  typeDefs,
  resolvers,
  validationRules: [
    createComplexityLimitRule(1000, {
      // กำหนด complexity ของแต่ละ type
      scalarCost: 1,
      objectCost: 2,
      listFactor: 10,
    }),
  ],
});
```

```javascript
// Query Depth Limiting
const depthLimit = require('graphql-depth-limit');

const server = new ApolloServer({
  validationRules: [
    depthLimit(5), // จำกัด depth ไม่เกิน 5 levels
  ],
});
```

### ขั้นตอนที่ 619: Monitoring GraphQL Performance

```javascript
// src/plugins/performance.js
const performancePlugin = {
  requestDidStart(requestContext) {
    const start = Date.now();

    return {
      executionDidStart() {
        return {
          willResolveField({ source, args, context, info }) {
            const fieldStart = Date.now();
            return (error, result) => {
              const duration = Date.now() - fieldStart;
              if (duration > 100) {
                console.warn(`Slow field: ${info.parentType}.${info.fieldName} took ${duration}ms`);
              }
            };
          },
        };
      },

      willSendResponse({ response }) {
        const duration = Date.now() - start;
        console.log(`Request completed in ${duration}ms`);

        // เพิ่ม timing info ใน response
        if (response.data) {
          response.extensions = {
            ...response.extensions,
            timing: { total: duration },
          };
        }
      },
    };
  },
};

module.exports = { performancePlugin };
```

### ขั้นตอนที่ 620: Full Setup Example

```javascript
// src/index.js - Complete Server Setup
const express = require('express');
const { ApolloServer } = require('apollo-server-express');
const { makeExecutableSchema } = require('@graphql-tools/schema');
const { applyMiddleware } = require('graphql-middleware');
const depthLimit = require('graphql-depth-limit');
const mongoose = require('mongoose');

const { typeDefs } = require('./schema/typeDefs');
const { resolvers } = require('./schema/resolvers');
const { permissions } = require('./auth/permissions');
const { createContext } = require('./middleware/auth');
const { createLoaders } = require('./dataloaders');
const { performancePlugin } = require('./plugins/performance');

async function startServer() {
  // Connect to MongoDB
  await mongoose.connect(process.env.MONGODB_URI, {
    useNewUrlParser: true,
    useUnifiedTopology: true,
  });
  console.log('✅ Connected to MongoDB');

  const app = express();

  // Build schema with permissions
  const schema = applyMiddleware(
    makeExecutableSchema({ typeDefs, resolvers }),
    permissions
  );

  const server = new ApolloServer({
    schema,
    context: async ({ req }) => {
      const baseContext = await createContext({ req });
      return {
        ...baseContext,
        loaders: createLoaders(),
      };
    },
    validationRules: [depthLimit(7)],
    plugins: [performancePlugin],
    formatError: (error) => {
      console.error('GraphQL Error:', error);
      return error;
    },
  });

  await server.start();
  server.applyMiddleware({ app, path: '/graphql' });

  app.listen(process.env.PORT || 4000, () => {
    console.log(`🚀 Server ready at http://localhost:${process.env.PORT || 4000}/graphql`);
  });
}

startServer().catch(console.error);
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Implement DataLoader สำหรับ Comments

```javascript
// TODO: สร้าง CommentsByPostLoader
function createCommentsByPostLoader() {
  return new DataLoader(async (postIds) => {
    // Load comments for all post IDs at once
    // Return comments grouped by postId
  });
}
```

### แบบฝึกหัดที่ 2: Real-time Notifications

```graphql
# TODO: Implement notification subscription
type Subscription {
  newNotification: Notification!
}

# และ publish notification เมื่อมี new comment, like หรือ follow
```

### แบบฝึกหัดที่ 3: File Upload Subscription

```javascript
// TODO: เมื่อ upload file เสร็จ ให้ publish progress
type Subscription {
  uploadProgress(uploadId: ID!): UploadProgress!
}

type UploadProgress {
  uploadId: ID!
  progress: Float!
  status: UploadStatus!
  url: String
}
```

### แบบฝึกหัดที่ 4: Federation - Comments Service

```javascript
// TODO: สร้าง comments-service ที่ extends Post type
// โดยใช้ Apollo Federation
const typeDefs = gql`
  extend type Post @key(fields: "id") {
    id: ID! @external
    comments: [Comment!]!  # เพิ่ม field
  }

  type Comment @key(fields: "id") {
    id: ID!
    content: String!
    author: User!
    post: Post!
  }
`;
```

---

## สรุป

Advanced GraphQL ประกอบด้วย:

1. **DataLoader** - แก้ปัญหา N+1 queries ด้วย batching และ caching
2. **Subscriptions** - real-time data ด้วย WebSockets และ PubSub
3. **Federation** - แบ่ง schema ออกเป็นหลาย microservices
4. **Directives** - authentication, rate limiting แบบ declarative
5. **Caching** - response caching และ persisted queries
6. **Authorization** - role-based access control ด้วย graphql-shield
7. **Performance** - query complexity, depth limiting, monitoring

ในบทถัดไปเราจะเรียนรู้ **WebSockets** - Socket.io และการสร้าง real-time applications
