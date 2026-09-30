# Part 59: GraphQL
## ขั้นตอนที่ 59-59 จาก 1000

---

## บทนำ

GraphQL เป็น query language สำหรับ API ที่ Facebook พัฒนา ช่วยให้ client สามารถขอข้อมูลเฉพาะที่ต้องการได้ แก้ปัญหา over-fetching และ under-fetching

---

## 59.1 GraphQL Concepts

### ปัญหาของ REST API

```
Over-fetching: ดึงข้อมูลมากเกินต้องการ
GET /users/1 → { id, name, email, age, address, phone, ... }
(ต้องการแค่ name และ email)

Under-fetching: ต้องทำ request หลายครั้ง
GET /users/1      → user data
GET /users/1/orders → orders
GET /orders/1/items → items
(3 requests สำหรับข้อมูลที่เกี่ยวข้องกัน)
```

### GraphQL แก้ปัญหาอย่างไร

```graphql
# ขอเฉพาะที่ต้องการ, ได้ทุกอย่างใน 1 request
query {
  user(id: "1") {
    name
    email
    orders {
      id
      total
      items {
        product {
          name
          price
        }
        quantity
      }
    }
  }
}
```

---

## 59.2 Apollo Server

### ติดตั้ง

```bash
npm install @apollo/server graphql
npm install @apollo/server @as-integrations/express express
```

### Setup พื้นฐาน

```javascript
// src/graphql/server.js
const { ApolloServer } = require('@apollo/server');
const { expressMiddleware } = require('@apollo/server/express4');
const express = require('express');

const typeDefs = require('./typeDefs');
const resolvers = require('./resolvers');
const { createContext } = require('./context');

const startServer = async () => {
  const app = express();
  
  const server = new ApolloServer({
    typeDefs,
    resolvers,
    
    // Formatting errors
    formatError: (formattedError, error) => {
      // Log server errors
      if (formattedError.extensions?.code === 'INTERNAL_SERVER_ERROR') {
        console.error('GraphQL Error:', error);
      }
      
      // ไม่ expose internal errors ใน production
      if (process.env.NODE_ENV === 'production') {
        if (formattedError.extensions?.code === 'INTERNAL_SERVER_ERROR') {
          return { message: 'Internal server error', extensions: { code: 'INTERNAL_SERVER_ERROR' } };
        }
      }
      
      return formattedError;
    },
    
    // Introspection ใน development เท่านั้น
    introspection: process.env.NODE_ENV !== 'production'
  });
  
  await server.start();
  
  app.use(express.json());
  
  app.use('/graphql', expressMiddleware(server, {
    context: createContext
  }));
  
  app.listen(4000, () => {
    console.log('🚀 GraphQL server ready at http://localhost:4000/graphql');
  });
};

startServer();
```

---

## 59.3 Schema Definition

```javascript
// src/graphql/typeDefs/index.js
const { gql } = require('graphql-tag');

const typeDefs = gql`
  # Scalar types
  scalar DateTime
  scalar JSON
  
  # Enums
  enum OrderStatus {
    PENDING
    PROCESSING
    SHIPPED
    DELIVERED
    CANCELLED
  }
  
  enum UserRole {
    USER
    ADMIN
    MODERATOR
  }
  
  # Types
  type User {
    id: ID!
    name: String!
    email: String!
    role: UserRole!
    avatar: String
    createdAt: DateTime!
    
    # Relationships
    orders(status: OrderStatus, limit: Int, offset: Int): [Order!]!
    orderCount: Int!
    totalSpent: Float!
  }
  
  type Product {
    id: ID!
    name: String!
    description: String
    price: Float!
    stock: Int!
    category: Category!
    images: [String!]!
    
    # Computed fields
    isInStock: Boolean!
    discountedPrice: Float
  }
  
  type Category {
    id: ID!
    name: String!
    slug: String!
    products(limit: Int, offset: Int): [Product!]!
    productCount: Int!
  }
  
  type Order {
    id: ID!
    user: User!
    items: [OrderItem!]!
    status: OrderStatus!
    total: Float!
    createdAt: DateTime!
    updatedAt: DateTime!
  }
  
  type OrderItem {
    id: ID!
    product: Product!
    quantity: Int!
    price: Float!
    subtotal: Float!
  }
  
  # Input types
  input CreateUserInput {
    name: String!
    email: String!
    password: String!
  }
  
  input UpdateUserInput {
    name: String
    avatar: String
  }
  
  input CreateOrderInput {
    items: [OrderItemInput!]!
    shippingAddress: AddressInput!
  }
  
  input OrderItemInput {
    productId: ID!
    quantity: Int!
  }
  
  input AddressInput {
    street: String!
    city: String!
    country: String!
    zipCode: String!
  }
  
  input ProductFilter {
    categoryId: ID
    minPrice: Float
    maxPrice: Float
    inStock: Boolean
    search: String
  }
  
  # Pagination types
  type PageInfo {
    hasNextPage: Boolean!
    hasPreviousPage: Boolean!
    startCursor: String
    endCursor: String
    totalCount: Int!
  }
  
  type ProductConnection {
    edges: [ProductEdge!]!
    pageInfo: PageInfo!
  }
  
  type ProductEdge {
    node: Product!
    cursor: String!
  }
  
  # Auth types
  type AuthPayload {
    token: String!
    user: User!
    expiresAt: DateTime!
  }
  
  # Queries
  type Query {
    # Users
    me: User
    user(id: ID!): User
    users(limit: Int, offset: Int, role: UserRole): [User!]!
    
    # Products
    product(id: ID!): Product
    products(
      filter: ProductFilter
      limit: Int
      offset: Int
      sortBy: String
      sortOrder: String
    ): ProductConnection!
    
    # Orders
    order(id: ID!): Order
    myOrders(status: OrderStatus, limit: Int, offset: Int): [Order!]!
    
    # Categories
    categories: [Category!]!
    category(slug: String!): Category
  }
  
  # Mutations
  type Mutation {
    # Auth
    register(input: CreateUserInput!): AuthPayload!
    login(email: String!, password: String!): AuthPayload!
    logout: Boolean!
    
    # Users
    updateProfile(input: UpdateUserInput!): User!
    
    # Orders
    createOrder(input: CreateOrderInput!): Order!
    cancelOrder(id: ID!): Order!
    
    # Admin mutations
    updateOrderStatus(id: ID!, status: OrderStatus!): Order!
  }
  
  # Subscriptions
  type Subscription {
    orderStatusUpdated(orderId: ID!): Order!
    newOrder: Order!
    productStockUpdated(productId: ID!): Product!
  }
`;

module.exports = typeDefs;
```

---

## 59.4 Resolvers

```javascript
// src/graphql/resolvers/index.js
const { GraphQLDateTime } = require('graphql-scalars');
const { GraphQLJSON } = require('graphql-scalars');

const userResolvers = require('./userResolvers');
const productResolvers = require('./productResolvers');
const orderResolvers = require('./orderResolvers');
const mutationResolvers = require('./mutations');
const subscriptionResolvers = require('./subscriptions');

const resolvers = {
  DateTime: GraphQLDateTime,
  JSON: GraphQLJSON,
  
  Query: {
    ...userResolvers.Query,
    ...productResolvers.Query,
    ...orderResolvers.Query
  },
  
  Mutation: mutationResolvers,
  Subscription: subscriptionResolvers,
  
  User: userResolvers.User,
  Product: productResolvers.Product,
  Order: orderResolvers.Order,
  OrderItem: orderResolvers.OrderItem
};

module.exports = resolvers;
```

```javascript
// src/graphql/resolvers/userResolvers.js
const User = require('../../models/User');
const Order = require('../../models/Order');
const { AuthenticationError, ForbiddenError } = require('@apollo/server');

const userResolvers = {
  Query: {
    me: (_, __, { user }) => {
      if (!user) throw new AuthenticationError('Not authenticated');
      return User.findByPk(user.id);
    },
    
    user: async (_, { id }, { user }) => {
      if (!user || user.role !== 'ADMIN') {
        throw new ForbiddenError('Admin access required');
      }
      return User.findByPk(id);
    },
    
    users: async (_, { limit = 10, offset = 0, role }, { user }) => {
      if (!user || user.role !== 'ADMIN') {
        throw new ForbiddenError('Admin access required');
      }
      
      const where = role ? { role } : {};
      return User.findAll({ where, limit, offset });
    }
  },
  
  User: {
    // Field resolver - ดึง orders ของ user
    orders: async (parent, { status, limit = 10, offset = 0 }, { loaders }) => {
      const where = { userId: parent.id };
      if (status) where.status = status;
      
      return Order.findAll({ where, limit, offset });
    },
    
    orderCount: async (parent) => {
      return Order.count({ where: { userId: parent.id } });
    },
    
    totalSpent: async (parent) => {
      const result = await Order.sum('total', {
        where: { userId: parent.id, status: 'DELIVERED' }
      });
      return result || 0;
    }
  }
};

module.exports = userResolvers;
```

```javascript
// src/graphql/resolvers/productResolvers.js
const { Op } = require('sequelize');
const Product = require('../../models/Product');
const Category = require('../../models/Category');

const productResolvers = {
  Query: {
    product: (_, { id }) => Product.findByPk(id),
    
    products: async (_, { filter = {}, limit = 20, offset = 0, sortBy = 'createdAt', sortOrder = 'DESC' }) => {
      const where = {};
      
      if (filter.categoryId) where.categoryId = filter.categoryId;
      if (filter.minPrice) where.price = { ...where.price, [Op.gte]: filter.minPrice };
      if (filter.maxPrice) where.price = { ...where.price, [Op.lte]: filter.maxPrice };
      if (filter.inStock) where.stock = { [Op.gt]: 0 };
      if (filter.search) {
        where[Op.or] = [
          { name: { [Op.iLike]: `%${filter.search}%` } },
          { description: { [Op.iLike]: `%${filter.search}%` } }
        ];
      }
      
      const { count, rows } = await Product.findAndCountAll({
        where,
        limit,
        offset,
        order: [[sortBy, sortOrder]],
        include: [{ model: Category }]
      });
      
      return {
        edges: rows.map((product, index) => ({
          node: product,
          cursor: Buffer.from(`${offset + index}`).toString('base64')
        })),
        pageInfo: {
          hasNextPage: offset + limit < count,
          hasPreviousPage: offset > 0,
          totalCount: count
        }
      };
    }
  },
  
  Product: {
    category: async (parent, _, { loaders }) => {
      return loaders.categoryLoader.load(parent.categoryId);
    },
    
    isInStock: (parent) => parent.stock > 0,
    
    discountedPrice: (parent) => {
      if (parent.discount > 0) {
        return parent.price * (1 - parent.discount / 100);
      }
      return null;
    }
  }
};

module.exports = productResolvers;
```

---

## 59.5 Mutations

```javascript
// src/graphql/resolvers/mutations.js
const bcrypt = require('bcrypt');
const jwt = require('jsonwebtoken');
const User = require('../../models/User');
const Order = require('../../models/Order');
const OrderItem = require('../../models/OrderItem');
const Product = require('../../models/Product');
const { AuthenticationError, UserInputError } = require('@apollo/server');
const { pubsub } = require('../pubsub');

const mutations = {
  register: async (_, { input }) => {
    const { name, email, password } = input;
    
    // Validate
    const existing = await User.findOne({ where: { email } });
    if (existing) {
      throw new UserInputError('Email already in use');
    }
    
    const hashedPassword = await bcrypt.hash(password, 12);
    
    const user = await User.create({
      name,
      email,
      password: hashedPassword,
      role: 'USER'
    });
    
    const token = generateToken(user);
    const expiresAt = new Date(Date.now() + 7 * 24 * 60 * 60 * 1000);
    
    return { token, user, expiresAt };
  },
  
  login: async (_, { email, password }) => {
    const user = await User.findOne({ where: { email } });
    
    if (!user || !await bcrypt.compare(password, user.password)) {
      throw new AuthenticationError('Invalid email or password');
    }
    
    const token = generateToken(user);
    const expiresAt = new Date(Date.now() + 7 * 24 * 60 * 60 * 1000);
    
    return { token, user, expiresAt };
  },
  
  createOrder: async (_, { input }, { user }) => {
    if (!user) throw new AuthenticationError('Not authenticated');
    
    const { items, shippingAddress } = input;
    
    // Validate products และคำนวณ total
    let total = 0;
    const orderItems = [];
    
    for (const item of items) {
      const product = await Product.findByPk(item.productId);
      
      if (!product) {
        throw new UserInputError(`Product ${item.productId} not found`);
      }
      
      if (product.stock < item.quantity) {
        throw new UserInputError(`Insufficient stock for ${product.name}`);
      }
      
      const subtotal = product.price * item.quantity;
      total += subtotal;
      
      orderItems.push({
        productId: product.id,
        quantity: item.quantity,
        price: product.price,
        subtotal
      });
    }
    
    // สร้าง order
    const order = await Order.create({
      userId: user.id,
      status: 'PENDING',
      total,
      shippingAddress
    });
    
    // สร้าง order items
    await OrderItem.bulkCreate(
      orderItems.map(item => ({ ...item, orderId: order.id }))
    );
    
    // อัพเดท stock
    for (const item of orderItems) {
      await Product.decrement('stock', {
        by: item.quantity,
        where: { id: item.productId }
      });
    }
    
    // Publish subscription event
    pubsub.publish('NEW_ORDER', { newOrder: order });
    
    return order;
  },
  
  updateOrderStatus: async (_, { id, status }, { user }) => {
    if (!user || user.role !== 'ADMIN') {
      throw new ForbiddenError('Admin access required');
    }
    
    const order = await Order.findByPk(id);
    if (!order) throw new UserInputError('Order not found');
    
    await order.update({ status });
    
    // Publish subscription event
    pubsub.publish('ORDER_STATUS_UPDATED', {
      orderStatusUpdated: order,
      orderId: id
    });
    
    return order;
  }
};

const generateToken = (user) => {
  return jwt.sign(
    { id: user.id, email: user.email, role: user.role },
    process.env.JWT_SECRET,
    { expiresIn: '7d' }
  );
};

module.exports = mutations;
```

---

## 59.6 Subscriptions

```javascript
// src/graphql/pubsub.js
const { PubSub } = require('graphql-subscriptions');

const pubsub = new PubSub();
module.exports = { pubsub };
```

```javascript
// src/graphql/resolvers/subscriptions.js
const { pubsub } = require('../pubsub');
const { withFilter } = require('graphql-subscriptions');

const subscriptions = {
  orderStatusUpdated: {
    subscribe: withFilter(
      () => pubsub.asyncIterator('ORDER_STATUS_UPDATED'),
      (payload, variables) => {
        return payload.orderId === variables.orderId;
      }
    )
  },
  
  newOrder: {
    subscribe: (_, __, { user }) => {
      if (!user || user.role !== 'ADMIN') {
        throw new Error('Admin access required');
      }
      return pubsub.asyncIterator('NEW_ORDER');
    }
  },
  
  productStockUpdated: {
    subscribe: withFilter(
      () => pubsub.asyncIterator('PRODUCT_STOCK_UPDATED'),
      (payload, variables) => {
        return payload.productId === variables.productId;
      }
    )
  }
};

module.exports = subscriptions;
```

### Setup WebSocket สำหรับ Subscriptions

```javascript
// src/graphql/subscriptionServer.js
const { createServer } = require('http');
const { WebSocketServer } = require('ws');
const { useServer } = require('graphql-ws/lib/use/ws');
const { makeExecutableSchema } = require('@graphql-tools/schema');
const typeDefs = require('./typeDefs');
const resolvers = require('./resolvers');

const schema = makeExecutableSchema({ typeDefs, resolvers });

const httpServer = createServer(app);

const wsServer = new WebSocketServer({
  server: httpServer,
  path: '/graphql'
});

useServer({ schema }, wsServer);

httpServer.listen(4000, () => {
  console.log('🚀 HTTP server ready at http://localhost:4000/graphql');
  console.log('🔌 WebSocket server ready at ws://localhost:4000/graphql');
});
```

---

## 59.7 DataLoader สำหรับ GraphQL

```javascript
// src/graphql/dataloaders.js
const DataLoader = require('dataloader');
const User = require('../models/User');
const Product = require('../models/Product');
const Category = require('../models/Category');
const { Op } = require('sequelize');

const createDataLoaders = () => ({
  userLoader: new DataLoader(async (ids) => {
    const users = await User.findAll({ where: { id: { [Op.in]: ids } } });
    const map = new Map(users.map(u => [u.id.toString(), u]));
    return ids.map(id => map.get(id.toString()) || null);
  }),
  
  productLoader: new DataLoader(async (ids) => {
    const products = await Product.findAll({ where: { id: { [Op.in]: ids } } });
    const map = new Map(products.map(p => [p.id.toString(), p]));
    return ids.map(id => map.get(id.toString()) || null);
  }),
  
  categoryLoader: new DataLoader(async (ids) => {
    const categories = await Category.findAll({ where: { id: { [Op.in]: ids } } });
    const map = new Map(categories.map(c => [c.id.toString(), c]));
    return ids.map(id => map.get(id.toString()) || null);
  })
});

module.exports = { createDataLoaders };
```

### Context ที่รวม Auth และ DataLoader

```javascript
// src/graphql/context.js
const jwt = require('jsonwebtoken');
const { createDataLoaders } = require('./dataloaders');

const createContext = async ({ req }) => {
  const token = req.headers.authorization?.replace('Bearer ', '');
  
  let user = null;
  
  if (token) {
    try {
      user = jwt.verify(token, process.env.JWT_SECRET);
    } catch {
      // Invalid token - user จะเป็น null
    }
  }
  
  return {
    user,
    loaders: createDataLoaders(),  // สร้างใหม่ทุก request
    req
  };
};

module.exports = { createContext };
```

---

## แบบฝึกหัดที่ 59

### แบบฝึกหัดพื้นฐาน

**1. Blog GraphQL API**

สร้าง GraphQL API สำหรับ blog:
- Types: User, Post, Comment
- Queries: posts, post, me
- Mutations: createPost, updatePost, addComment
- Authentication

**2. Real-time Chat**

สร้าง chat ด้วย Subscriptions:
- Mutation: sendMessage
- Subscription: messageReceived
- Room-based chat

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **GraphQL** - concepts, advantages
2. **Apollo Server** - setup, middleware
3. **Schema** - types, queries, mutations
4. **Resolvers** - field resolvers, context
5. **Mutations** - create, update operations
6. **Subscriptions** - real-time updates
7. **DataLoader** - solving N+1 in GraphQL

**ถัดไป:** Part 60 - gRPC
