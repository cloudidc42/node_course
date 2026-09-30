# Part 53 | ขั้นตอนที่ 921-940 จาก 1000

## gRPC - High-Performance RPC Framework

gRPC เป็น framework สำหรับ Remote Procedure Call ที่พัฒนาโดย Google ใช้ Protocol Buffers เป็น interface definition language และ HTTP/2 เป็น transport layer

---

## ขั้นตอนที่ 921: ทำความเข้าใจ gRPC

### gRPC vs REST:
| Feature | gRPC | REST |
|---------|------|------|
| Protocol | HTTP/2 | HTTP/1.1 |
| Data Format | Protocol Buffers (binary) | JSON (text) |
| Streaming | Built-in | Limited |
| Type Safety | Strict | Loose |
| Performance | High | Medium |
| Browser Support | Limited | Full |

### ประเภทของ gRPC calls:
1. **Unary** - Client ส่ง request, server ตอบ response
2. **Server Streaming** - Client ส่ง request, server ตอบ stream of responses
3. **Client Streaming** - Client ส่ง stream, server ตอบ response
4. **Bidirectional Streaming** - ทั้งสองฝ่าย stream ข้อมูล

---

## ขั้นตอนที่ 922: ติดตั้ง gRPC และ Protocol Buffers

```bash
npm install @grpc/grpc-js @grpc/proto-loader
npm install google-protobuf
```

---

## ขั้นตอนที่ 923: สร้าง Protocol Buffer Definition

```protobuf
// proto/user.proto

syntax = "proto3";

package user;

// Import สำหรับ Timestamp
import "google/protobuf/timestamp.proto";
import "google/protobuf/empty.proto";

// UserService
service UserService {
  // Unary RPCs
  rpc GetUser (GetUserRequest) returns (UserResponse);
  rpc CreateUser (CreateUserRequest) returns (UserResponse);
  rpc UpdateUser (UpdateUserRequest) returns (UserResponse);
  rpc DeleteUser (DeleteUserRequest) returns (google.protobuf.Empty);
  
  // Server streaming
  rpc ListUsers (ListUsersRequest) returns (stream UserResponse);
  
  // Client streaming
  rpc BulkCreateUsers (stream CreateUserRequest) returns (BulkCreateResponse);
  
  // Bidirectional streaming
  rpc Chat (stream ChatMessage) returns (stream ChatMessage);
}

// Messages
message User {
  string id = 1;
  string name = 2;
  string email = 3;
  string role = 4;
  bool is_active = 5;
  google.protobuf.Timestamp created_at = 6;
  google.protobuf.Timestamp updated_at = 7;
}

message GetUserRequest {
  string id = 1;
}

message CreateUserRequest {
  string name = 1;
  string email = 2;
  string password = 3;
  string role = 4;
}

message UpdateUserRequest {
  string id = 1;
  optional string name = 2;
  optional string email = 3;
  optional string role = 4;
}

message DeleteUserRequest {
  string id = 1;
}

message ListUsersRequest {
  int32 page = 1;
  int32 limit = 2;
  string search = 3;
  string role = 4;
}

message UserResponse {
  bool success = 1;
  User user = 2;
  string message = 3;
}

message BulkCreateResponse {
  int32 created = 1;
  int32 failed = 2;
  repeated string errors = 3;
}

message ChatMessage {
  string user_id = 1;
  string message = 2;
  google.protobuf.Timestamp timestamp = 3;
}
```

```protobuf
// proto/product.proto
syntax = "proto3";

package product;

import "google/protobuf/timestamp.proto";

service ProductService {
  rpc GetProduct (GetProductRequest) returns (ProductResponse);
  rpc SearchProducts (SearchProductsRequest) returns (stream ProductResponse);
  rpc UpdateInventory (stream InventoryUpdate) returns (InventoryUpdateResponse);
}

message Product {
  string id = 1;
  string name = 2;
  string description = 3;
  float price = 4;
  int32 stock = 5;
  string category_id = 6;
  bool is_active = 7;
  repeated string tags = 8;
  google.protobuf.Timestamp created_at = 9;
}

message GetProductRequest {
  string id = 1;
}

message SearchProductsRequest {
  string query = 1;
  float min_price = 2;
  float max_price = 3;
  string category_id = 4;
  int32 limit = 5;
}

message ProductResponse {
  bool success = 1;
  Product product = 2;
  string error = 3;
}

message InventoryUpdate {
  string product_id = 1;
  int32 quantity_change = 2;
  string reason = 3;
}

message InventoryUpdateResponse {
  int32 updated = 1;
  int32 failed = 2;
}
```

---

## ขั้นตอนที่ 924: สร้าง gRPC Server

```javascript
// server/grpcServer.js
const grpc = require('@grpc/grpc-js');
const protoLoader = require('@grpc/proto-loader');
const path = require('path');

// โหลด proto files
const loadProto = (filename) => {
  const packageDefinition = protoLoader.loadSync(
    path.join(__dirname, '../proto', filename),
    {
      keepCase: true,
      longs: String,
      enums: String,
      defaults: true,
      oneofs: true,
      includeDirs: [path.join(__dirname, '../proto')]
    }
  );
  
  return grpc.loadPackageDefinition(packageDefinition);
};

const userProto = loadProto('user.proto');
const productProto = loadProto('product.proto');

// User Service Implementations
const userService = {
  // Unary: Get single user
  getUser: async (call, callback) => {
    try {
      const { id } = call.request;
      const User = require('../models/User');
      
      const user = await User.findById(id).lean();
      
      if (!user) {
        return callback({
          code: grpc.status.NOT_FOUND,
          message: `User ${id} not found`
        });
      }
      
      callback(null, {
        success: true,
        user: {
          id: user._id.toString(),
          name: user.name,
          email: user.email,
          role: user.role,
          is_active: user.isActive,
          created_at: {
            seconds: Math.floor(user.createdAt.getTime() / 1000),
            nanos: 0
          }
        }
      });
    } catch (error) {
      callback({
        code: grpc.status.INTERNAL,
        message: error.message
      });
    }
  },

  // Unary: Create user
  createUser: async (call, callback) => {
    try {
      const { name, email, password, role } = call.request;
      const User = require('../models/User');
      
      const existingUser = await User.findOne({ email });
      if (existingUser) {
        return callback({
          code: grpc.status.ALREADY_EXISTS,
          message: 'Email already exists'
        });
      }
      
      const user = await User.create({ name, email, password, role: role || 'user' });
      
      callback(null, {
        success: true,
        user: {
          id: user._id.toString(),
          name: user.name,
          email: user.email,
          role: user.role,
          is_active: user.isActive
        },
        message: 'User created successfully'
      });
    } catch (error) {
      callback({
        code: grpc.status.INTERNAL,
        message: error.message
      });
    }
  },

  // Server streaming: List users
  listUsers: async (call) => {
    try {
      const { page = 1, limit = 20, search, role } = call.request;
      const User = require('../models/User');
      
      const query = {};
      if (search) {
        query.$or = [
          { name: { $regex: search, $options: 'i' } },
          { email: { $regex: search, $options: 'i' } }
        ];
      }
      if (role) query.role = role;
      
      const cursor = User.find(query)
        .skip((page - 1) * limit)
        .limit(limit)
        .cursor();
      
      for await (const user of cursor) {
        // ส่ง user ทีละคน (streaming)
        call.write({
          success: true,
          user: {
            id: user._id.toString(),
            name: user.name,
            email: user.email,
            role: user.role,
            is_active: user.isActive
          }
        });
      }
      
      call.end();
    } catch (error) {
      call.destroy(error);
    }
  },

  // Client streaming: Bulk create
  bulkCreateUsers: async (call, callback) => {
    const users = [];
    let created = 0;
    let failed = 0;
    const errors = [];
    
    call.on('data', (createRequest) => {
      users.push(createRequest);
    });
    
    call.on('end', async () => {
      const User = require('../models/User');
      
      for (const userData of users) {
        try {
          await User.create(userData);
          created++;
        } catch (error) {
          failed++;
          errors.push(`${userData.email}: ${error.message}`);
        }
      }
      
      callback(null, { created, failed, errors });
    });
    
    call.on('error', (error) => {
      console.error('Stream error:', error);
    });
  },

  // Bidirectional streaming: Chat
  chat: (call) => {
    call.on('data', (message) => {
      console.log(`Received from ${message.user_id}: ${message.message}`);
      
      // Echo back with timestamp
      call.write({
        user_id: 'server',
        message: `Echo: ${message.message}`,
        timestamp: {
          seconds: Math.floor(Date.now() / 1000),
          nanos: 0
        }
      });
    });
    
    call.on('end', () => {
      call.end();
    });
  }
};

// สร้าง gRPC Server
const createServer = () => {
  const server = new grpc.Server();
  
  server.addService(userProto.user.UserService.service, userService);
  server.addService(productProto.product.ProductService.service, productService);
  
  return server;
};

// เริ่ม server
const startServer = (port = 50051) => {
  const server = createServer();
  
  server.bindAsync(
    `0.0.0.0:${port}`,
    grpc.ServerCredentials.createInsecure(),
    (error, port) => {
      if (error) {
        console.error('Failed to start gRPC server:', error);
        return;
      }
      console.log(`gRPC server running on port ${port}`);
    }
  );
  
  return server;
};

module.exports = { createServer, startServer };
```

---

## ขั้นตอนที่ 925: สร้าง gRPC Client

```javascript
// client/grpcClient.js
const grpc = require('@grpc/grpc-js');
const protoLoader = require('@grpc/proto-loader');
const path = require('path');

const packageDefinition = protoLoader.loadSync(
  path.join(__dirname, '../proto/user.proto'),
  {
    keepCase: true,
    longs: String,
    enums: String,
    defaults: true,
    oneofs: true
  }
);

const userProto = grpc.loadPackageDefinition(packageDefinition);

class UserServiceClient {
  constructor(host = 'localhost', port = 50051) {
    this.client = new userProto.user.UserService(
      `${host}:${port}`,
      grpc.credentials.createInsecure()
    );
  }

  // Unary: Get user
  getUser(id) {
    return new Promise((resolve, reject) => {
      this.client.getUser({ id }, (error, response) => {
        if (error) reject(error);
        else resolve(response);
      });
    });
  }

  // Unary: Create user
  createUser(userData) {
    return new Promise((resolve, reject) => {
      this.client.createUser(userData, (error, response) => {
        if (error) reject(error);
        else resolve(response);
      });
    });
  }

  // Server streaming: List users
  listUsers(options = {}) {
    return new Promise((resolve, reject) => {
      const users = [];
      const call = this.client.listUsers(options);
      
      call.on('data', (response) => {
        if (response.user) {
          users.push(response.user);
        }
      });
      
      call.on('end', () => resolve(users));
      call.on('error', reject);
    });
  }

  // Async iterator สำหรับ server streaming
  async *listUsersStream(options = {}) {
    const call = this.client.listUsers(options);
    
    for await (const response of call) {
      yield response.user;
    }
  }

  // Client streaming: Bulk create
  bulkCreateUsers(usersData) {
    return new Promise((resolve, reject) => {
      const call = this.client.bulkCreateUsers((error, response) => {
        if (error) reject(error);
        else resolve(response);
      });
      
      usersData.forEach(userData => {
        call.write(userData);
      });
      
      call.end();
    });
  }

  // Bidirectional streaming: Chat
  createChatSession() {
    const call = this.client.chat();
    
    return {
      send: (userId, message) => {
        call.write({
          user_id: userId,
          message,
          timestamp: {
            seconds: Math.floor(Date.now() / 1000),
            nanos: 0
          }
        });
      },
      onMessage: (handler) => {
        call.on('data', handler);
      },
      close: () => call.end()
    };
  }

  // ปิด connection
  close() {
    this.client.close();
  }
}

module.exports = UserServiceClient;
```

---

## ขั้นตอนที่ 926: ใช้งาน gRPC Client

```javascript
// examples/grpcClientExample.js
const UserServiceClient = require('../client/grpcClient');

async function main() {
  const client = new UserServiceClient('localhost', 50051);

  try {
    // 1. สร้างผู้ใช้
    console.log('Creating user...');
    const createResult = await client.createUser({
      name: 'สมชาย ใจดี',
      email: 'somchai@example.com',
      password: 'Password123',
      role: 'user'
    });
    console.log('Created:', createResult.user);

    // 2. ดึงข้อมูลผู้ใช้
    console.log('\nGetting user...');
    const getResult = await client.getUser(createResult.user.id);
    console.log('Got:', getResult.user);

    // 3. ดึงรายชื่อผู้ใช้ (server streaming)
    console.log('\nListing users...');
    const users = await client.listUsers({ page: 1, limit: 10 });
    console.log(`Found ${users.length} users`);

    // 4. Async iteration สำหรับ large datasets
    console.log('\nStreaming users...');
    for await (const user of client.listUsersStream({ limit: 100 })) {
      console.log(`User: ${user.name} (${user.email})`);
    }

    // 5. Bulk create (client streaming)
    console.log('\nBulk creating users...');
    const bulkResult = await client.bulkCreateUsers([
      { name: 'User 1', email: 'user1@example.com', password: 'Pass123' },
      { name: 'User 2', email: 'user2@example.com', password: 'Pass123' },
      { name: 'User 3', email: 'user3@example.com', password: 'Pass123' }
    ]);
    console.log(`Bulk result: ${bulkResult.created} created, ${bulkResult.failed} failed`);

    // 6. Chat (bidirectional streaming)
    console.log('\nStarting chat session...');
    const chatSession = client.createChatSession();
    
    chatSession.onMessage((message) => {
      console.log(`Server: ${message.message}`);
    });
    
    chatSession.send('user123', 'Hello server!');
    chatSession.send('user123', 'How are you?');
    
    await new Promise(resolve => setTimeout(resolve, 1000));
    chatSession.close();

  } catch (error) {
    console.error('Error:', error);
  } finally {
    client.close();
  }
}

main();
```

---

## ขั้นตอนที่ 927: gRPC Interceptors (Middleware)

```javascript
// interceptors/authInterceptor.js
const grpc = require('@grpc/grpc-js');
const jwt = require('jsonwebtoken');

// Server-side interceptor สำหรับ authentication
const authInterceptor = (methodDescriptor, handler) => {
  return (call, callback) => {
    // ดึง metadata
    const metadata = call.metadata;
    const token = metadata.get('authorization')[0];
    
    if (!token) {
      return callback({
        code: grpc.status.UNAUTHENTICATED,
        message: 'Authentication required'
      });
    }
    
    try {
      // ตรวจสอบ JWT token
      const bearer = token.replace('Bearer ', '');
      const decoded = jwt.verify(bearer, process.env.JWT_SECRET);
      
      // เพิ่ม user info ใน call
      call.user = decoded;
      
      // เรียก handler ต่อไป
      handler(call, callback);
    } catch (error) {
      callback({
        code: grpc.status.UNAUTHENTICATED,
        message: 'Invalid token'
      });
    }
  };
};

// Logging interceptor
const loggingInterceptor = (methodDescriptor, handler) => {
  return (call, callback) => {
    const startTime = Date.now();
    const method = methodDescriptor.path;
    
    console.log(`gRPC call: ${method}`);
    
    const wrappedCallback = (error, response) => {
      const duration = Date.now() - startTime;
      
      if (error) {
        console.error(`gRPC ${method} failed (${duration}ms):`, error);
      } else {
        console.log(`gRPC ${method} success (${duration}ms)`);
      }
      
      callback(error, response);
    };
    
    handler(call, wrappedCallback);
  };
};

module.exports = { authInterceptor, loggingInterceptor };
```

---

## ขั้นตอนที่ 928: gRPC với TLS/SSL

```javascript
// server/secureGrpcServer.js
const grpc = require('@grpc/grpc-js');
const fs = require('fs');
const path = require('path');

// โหลด certificates
const loadCredentials = () => {
  const certPath = process.env.GRPC_CERT_PATH || './certs';
  
  const serverCert = fs.readFileSync(path.join(certPath, 'server.crt'));
  const serverKey = fs.readFileSync(path.join(certPath, 'server.key'));
  const caCert = fs.readFileSync(path.join(certPath, 'ca.crt'));
  
  return grpc.ServerCredentials.createSsl(
    caCert,
    [{
      cert_chain: serverCert,
      private_key: serverKey
    }],
    true // require client certificate
  );
};

// Client credentials
const loadClientCredentials = () => {
  const certPath = process.env.GRPC_CERT_PATH || './certs';
  
  const rootCert = fs.readFileSync(path.join(certPath, 'ca.crt'));
  const clientCert = fs.readFileSync(path.join(certPath, 'client.crt'));
  const clientKey = fs.readFileSync(path.join(certPath, 'client.key'));
  
  return grpc.credentials.createSsl(
    rootCert,
    clientKey,
    clientCert
  );
};

// Secure server
const startSecureServer = (port = 50051) => {
  const server = new grpc.Server();
  const credentials = loadCredentials();
  
  server.bindAsync(
    `0.0.0.0:${port}`,
    credentials,
    (error, port) => {
      if (error) throw error;
      console.log(`Secure gRPC server on port ${port}`);
    }
  );
  
  return server;
};

module.exports = { loadCredentials, loadClientCredentials, startSecureServer };
```

---

## ขั้นตอนที่ 929: gRPC Error Handling

```javascript
// utils/grpcErrors.js
const grpc = require('@grpc/grpc-js');

class GRPCError extends Error {
  constructor(code, message, details = null) {
    super(message);
    this.code = code;
    this.details = details;
  }
}

// สร้าง error factories
const errors = {
  notFound: (resource, id) => new GRPCError(
    grpc.status.NOT_FOUND,
    `${resource} with id '${id}' not found`
  ),
  
  alreadyExists: (resource, field, value) => new GRPCError(
    grpc.status.ALREADY_EXISTS,
    `${resource} with ${field} '${value}' already exists`
  ),
  
  invalidArgument: (message, details = null) => new GRPCError(
    grpc.status.INVALID_ARGUMENT,
    message,
    details
  ),
  
  unauthenticated: (message = 'Authentication required') => new GRPCError(
    grpc.status.UNAUTHENTICATED,
    message
  ),
  
  permissionDenied: (message = 'Permission denied') => new GRPCError(
    grpc.status.PERMISSION_DENIED,
    message
  ),
  
  internal: (message = 'Internal server error') => new GRPCError(
    grpc.status.INTERNAL,
    message
  ),
  
  unavailable: (message = 'Service unavailable') => new GRPCError(
    grpc.status.UNAVAILABLE,
    message
  )
};

// Wrapper สำหรับ handle errors ใน service methods
const handleError = (error, callback) => {
  if (error instanceof GRPCError) {
    return callback({
      code: error.code,
      message: error.message,
      details: error.details ? JSON.stringify(error.details) : undefined
    });
  }
  
  // Log unexpected errors
  console.error('Unexpected gRPC error:', error);
  
  callback({
    code: grpc.status.INTERNAL,
    message: 'An unexpected error occurred'
  });
};

// Decorator สำหรับ wrap service methods
const withErrorHandling = (fn) => {
  return async (call, callback) => {
    try {
      await fn(call, callback);
    } catch (error) {
      handleError(error, callback);
    }
  };
};

module.exports = { GRPCError, errors, handleError, withErrorHandling };
```

---

## ขั้นตอนที่ 930: Product Service Implementation

```javascript
// services/grpc/productService.js
const { withErrorHandling, errors } = require('../../utils/grpcErrors');
const Product = require('../../models/Product');

const productService = {
  getProduct: withErrorHandling(async (call, callback) => {
    const { id } = call.request;
    
    const product = await Product.findById(id).lean();
    if (!product) {
      throw errors.notFound('Product', id);
    }
    
    callback(null, {
      success: true,
      product: {
        id: product._id.toString(),
        name: product.name,
        description: product.description,
        price: product.price,
        stock: product.stock,
        category_id: product.category?.toString(),
        is_active: product.isActive,
        tags: product.tags || []
      }
    });
  }),

  searchProducts: async (call) => {
    const { query, min_price, max_price, category_id, limit = 20 } = call.request;
    
    const filter = { isActive: true };
    
    if (query) {
      filter.$text = { $search: query };
    }
    
    if (min_price > 0 || max_price > 0) {
      filter.price = {};
      if (min_price > 0) filter.price.$gte = min_price;
      if (max_price > 0) filter.price.$lte = max_price;
    }
    
    if (category_id) {
      filter.category = category_id;
    }
    
    try {
      const cursor = Product.find(filter)
        .limit(limit)
        .cursor();
      
      for await (const product of cursor) {
        call.write({
          success: true,
          product: {
            id: product._id.toString(),
            name: product.name,
            price: product.price,
            stock: product.stock
          }
        });
      }
      
      call.end();
    } catch (error) {
      call.destroy(error);
    }
  },

  updateInventory: async (call, callback) => {
    let updated = 0;
    let failed = 0;
    const updates = [];
    
    call.on('data', (update) => {
      updates.push(update);
    });
    
    call.on('end', async () => {
      for (const update of updates) {
        try {
          const result = await Product.findByIdAndUpdate(
            update.product_id,
            { $inc: { stock: update.quantity_change } },
            { new: true }
          );
          
          if (result) {
            updated++;
          } else {
            failed++;
          }
        } catch (error) {
          failed++;
        }
      }
      
      callback(null, { updated, failed });
    });
  }
};

module.exports = productService;
```

---

## ขั้นตอนที่ 931: gRPC-Gateway (REST to gRPC)

```javascript
// gateway/grpcGateway.js
// แปลง REST requests ไปยัง gRPC calls

const express = require('express');
const router = express.Router();
const UserServiceClient = require('../client/grpcClient');
const grpc = require('@grpc/grpc-js');

const userClient = new UserServiceClient();

// Map gRPC error codes to HTTP status codes
const grpcToHttpStatus = (code) => {
  const mapping = {
    [grpc.status.OK]: 200,
    [grpc.status.NOT_FOUND]: 404,
    [grpc.status.ALREADY_EXISTS]: 409,
    [grpc.status.INVALID_ARGUMENT]: 400,
    [grpc.status.UNAUTHENTICATED]: 401,
    [grpc.status.PERMISSION_DENIED]: 403,
    [grpc.status.INTERNAL]: 500,
    [grpc.status.UNAVAILABLE]: 503
  };
  
  return mapping[code] || 500;
};

// Handle gRPC errors
const handleGRPCError = (error, res) => {
  const httpStatus = grpcToHttpStatus(error.code);
  res.status(httpStatus).json({
    success: false,
    error: {
      code: grpc.status[error.code],
      message: error.message
    }
  });
};

// REST endpoints ที่แปลงไปยัง gRPC
router.get('/users', async (req, res) => {
  try {
    const { page = 1, limit = 20, search, role } = req.query;
    const users = await userClient.listUsers({
      page: parseInt(page),
      limit: parseInt(limit),
      search: search || '',
      role: role || ''
    });
    
    res.json({ success: true, data: users });
  } catch (error) {
    handleGRPCError(error, res);
  }
});

router.get('/users/:id', async (req, res) => {
  try {
    const result = await userClient.getUser(req.params.id);
    res.json({ success: true, data: result.user });
  } catch (error) {
    handleGRPCError(error, res);
  }
});

router.post('/users', async (req, res) => {
  try {
    const result = await userClient.createUser(req.body);
    res.status(201).json({ success: true, data: result.user });
  } catch (error) {
    handleGRPCError(error, res);
  }
});

module.exports = router;
```

---

## ขั้นตอนที่ 932: gRPC Health Check

```javascript
// services/grpc/healthService.js
const grpc = require('@grpc/grpc-js');
const protoLoader = require('@grpc/proto-loader');
const path = require('path');

// ใช้ gRPC Health Checking Protocol
const healthProtoDefinition = protoLoader.loadSync(
  path.join(__dirname, '../../node_modules/grpc-health-probe/proto/grpc/health/v1/health.proto'),
  { keepCase: true, longs: String, enums: String, defaults: true, oneofs: true }
);

const healthProto = grpc.loadPackageDefinition(healthProtoDefinition);

// สถานะของแต่ละ service
const serviceStatuses = new Map();

const healthService = {
  check: (call, callback) => {
    const { service } = call.request;
    
    // ถ้าไม่ระบุ service ให้ตรวจสอบ overall health
    if (!service) {
      const allHealthy = Array.from(serviceStatuses.values())
        .every(status => status === healthProto.grpc.health.v1.HealthCheckResponse.ServingStatus.SERVING);
      
      return callback(null, {
        status: allHealthy 
          ? healthProto.grpc.health.v1.HealthCheckResponse.ServingStatus.SERVING
          : healthProto.grpc.health.v1.HealthCheckResponse.ServingStatus.NOT_SERVING
      });
    }
    
    const status = serviceStatuses.get(service);
    
    if (!status) {
      return callback({
        code: grpc.status.NOT_FOUND,
        message: `Service '${service}' not found`
      });
    }
    
    callback(null, { status });
  },

  watch: (call) => {
    const { service } = call.request;
    
    // ส่ง status ทันที
    const currentStatus = serviceStatuses.get(service) || 
      healthProto.grpc.health.v1.HealthCheckResponse.ServingStatus.UNKNOWN;
    
    call.write({ status: currentStatus });
    
    // Watch สำหรับ status changes
    const interval = setInterval(() => {
      const status = serviceStatuses.get(service) || 
        healthProto.grpc.health.v1.HealthCheckResponse.ServingStatus.UNKNOWN;
      call.write({ status });
    }, 5000);
    
    call.on('cancelled', () => {
      clearInterval(interval);
    });
  }
};

// Helper สำหรับอัพเดท service status
const setServiceStatus = (service, status) => {
  serviceStatuses.set(service, status);
};

const SERVING = healthProto.grpc.health.v1.HealthCheckResponse.ServingStatus.SERVING;
const NOT_SERVING = healthProto.grpc.health.v1.HealthCheckResponse.ServingStatus.NOT_SERVING;

// ตั้งค่า initial status
setServiceStatus('user.UserService', SERVING);
setServiceStatus('product.ProductService', SERVING);

module.exports = { healthService, setServiceStatus, SERVING, NOT_SERVING };
```

---

## ขั้นตอนที่ 933: gRPC Load Balancing

```javascript
// client/loadBalancedClient.js
const grpc = require('@grpc/grpc-js');
const protoLoader = require('@grpc/proto-loader');

class LoadBalancedGRPCClient {
  constructor(servers, protoPath, service) {
    this.servers = servers;
    this.protoPath = protoPath;
    this.service = service;
    this.clients = [];
    this.currentIndex = 0;
    
    this.initialize();
  }

  initialize() {
    const packageDefinition = protoLoader.loadSync(this.protoPath, {
      keepCase: true,
      longs: String,
      enums: String,
      defaults: true,
      oneofs: true
    });
    
    const proto = grpc.loadPackageDefinition(packageDefinition);
    
    // สร้าง client สำหรับแต่ละ server
    this.clients = this.servers.map(server => {
      const ServiceClass = this.getServiceClass(proto);
      return new ServiceClass(server, grpc.credentials.createInsecure());
    });
  }

  getServiceClass(proto) {
    const parts = this.service.split('.');
    let current = proto;
    for (const part of parts) {
      current = current[part];
    }
    return current;
  }

  // Round-robin load balancing
  getClient() {
    const client = this.clients[this.currentIndex];
    this.currentIndex = (this.currentIndex + 1) % this.clients.length;
    return client;
  }

  // Retry สำหรับ failed requests
  async callWithRetry(method, request, retries = 3) {
    for (let attempt = 1; attempt <= retries; attempt++) {
      try {
        const client = this.getClient();
        return await new Promise((resolve, reject) => {
          client[method](request, (error, response) => {
            if (error) reject(error);
            else resolve(response);
          });
        });
      } catch (error) {
        if (attempt === retries) throw error;
        
        if (error.code === grpc.status.UNAVAILABLE) {
          console.log(`Retry ${attempt}/${retries} after server unavailable`);
          await new Promise(resolve => setTimeout(resolve, 1000 * attempt));
        } else {
          throw error; // ไม่ retry สำหรับ non-unavailable errors
        }
      }
    }
  }
}

module.exports = LoadBalancedGRPCClient;
```

---

## ขั้นตอนที่ 934: Protocol Buffer Optimization

```javascript
// utils/protoUtils.js

// ใช้ proto-js แทน JSON serialization
const protobuf = require('protobufjs');
const path = require('path');

class ProtoSerializer {
  constructor() {
    this.root = null;
    this.cache = new Map();
  }

  async initialize() {
    this.root = await protobuf.load([
      path.join(__dirname, '../proto/user.proto'),
      path.join(__dirname, '../proto/product.proto')
    ]);
  }

  // Serialize message
  encode(typeName, data) {
    if (!this.cache.has(typeName)) {
      this.cache.set(typeName, this.root.lookupType(typeName));
    }
    
    const type = this.cache.get(typeName);
    const message = type.create(data);
    return type.encode(message).finish();
  }

  // Deserialize message
  decode(typeName, buffer) {
    if (!this.cache.has(typeName)) {
      this.cache.set(typeName, this.root.lookupType(typeName));
    }
    
    const type = this.cache.get(typeName);
    return type.decode(buffer);
  }

  // เปรียบเทียบขนาด
  compareSize(typeName, data) {
    const protoBuffer = this.encode(typeName, data);
    const jsonString = JSON.stringify(data);
    
    return {
      proto: protoBuffer.length,
      json: Buffer.byteLength(jsonString),
      reduction: ((1 - protoBuffer.length / Buffer.byteLength(jsonString)) * 100).toFixed(1) + '%'
    };
  }
}

module.exports = new ProtoSerializer();
```

---

## ขั้นตอนที่ 935: gRPC Monitoring และ Metrics

```javascript
// monitoring/grpcMetrics.js
const grpc = require('@grpc/grpc-js');
const prometheus = require('prom-client');

// Prometheus metrics สำหรับ gRPC
const grpcRequestsTotal = new prometheus.Counter({
  name: 'grpc_server_requests_total',
  help: 'Total number of gRPC requests',
  labelNames: ['method', 'status']
});

const grpcRequestDuration = new prometheus.Histogram({
  name: 'grpc_server_request_duration_seconds',
  help: 'gRPC request duration in seconds',
  labelNames: ['method'],
  buckets: [0.001, 0.005, 0.01, 0.05, 0.1, 0.5, 1]
});

const grpcActiveStreams = new prometheus.Gauge({
  name: 'grpc_server_active_streams',
  help: 'Number of active gRPC streams',
  labelNames: ['method']
});

// Monitoring interceptor
const monitoringInterceptor = (methodDescriptor, handler) => {
  const method = methodDescriptor.path;
  
  return async (call, callback) => {
    const start = Date.now();
    grpcActiveStreams.inc({ method });
    
    const wrappedCallback = (error, response) => {
      const duration = (Date.now() - start) / 1000;
      const status = error ? grpc.status[error.code] || 'UNKNOWN' : 'OK';
      
      grpcRequestsTotal.inc({ method, status });
      grpcRequestDuration.observe({ method }, duration);
      grpcActiveStreams.dec({ method });
      
      callback(error, response);
    };
    
    try {
      await handler(call, wrappedCallback);
    } catch (error) {
      grpcActiveStreams.dec({ method });
      throw error;
    }
  };
};

// Route สำหรับ metrics
const express = require('express');
const metricsRouter = express.Router();

metricsRouter.get('/grpc', async (req, res) => {
  res.set('Content-Type', prometheus.register.contentType);
  res.send(await prometheus.register.metrics());
});

module.exports = { monitoringInterceptor, metricsRouter };
```

---

## ขั้นตอนที่ 936: Testing gRPC Services

```javascript
// tests/grpc.test.js
const grpc = require('@grpc/grpc-js');
const protoLoader = require('@grpc/proto-loader');
const path = require('path');
const { startServer } = require('../server/grpcServer');

describe('gRPC Service Tests', () => {
  let server;
  let client;
  const TEST_PORT = 50052;

  beforeAll(async () => {
    // เริ่ม test server
    server = await startServer(TEST_PORT);
    
    // สร้าง client
    const packageDefinition = protoLoader.loadSync(
      path.join(__dirname, '../proto/user.proto')
    );
    const userProto = grpc.loadPackageDefinition(packageDefinition);
    
    client = new userProto.user.UserService(
      `localhost:${TEST_PORT}`,
      grpc.credentials.createInsecure()
    );
  });

  afterAll(async () => {
    await new Promise((resolve) => server.tryShutdown(resolve));
    client.close();
  });

  describe('UserService', () => {
    let createdUserId;
    
    it('should create a user', (done) => {
      client.createUser({
        name: 'Test User',
        email: `test-${Date.now()}@example.com`,
        password: 'Password123',
        role: 'user'
      }, (error, response) => {
        expect(error).toBeNull();
        expect(response.success).toBe(true);
        expect(response.user).toBeDefined();
        expect(response.user.id).toBeTruthy();
        
        createdUserId = response.user.id;
        done();
      });
    });

    it('should get a user by id', (done) => {
      client.getUser({ id: createdUserId }, (error, response) => {
        expect(error).toBeNull();
        expect(response.success).toBe(true);
        expect(response.user.id).toBe(createdUserId);
        done();
      });
    });

    it('should return NOT_FOUND for non-existent user', (done) => {
      client.getUser({ id: '000000000000000000000000' }, (error, response) => {
        expect(error).toBeDefined();
        expect(error.code).toBe(grpc.status.NOT_FOUND);
        done();
      });
    });

    it('should stream users', (done) => {
      const users = [];
      const call = client.listUsers({ page: 1, limit: 10 });
      
      call.on('data', (response) => {
        if (response.user) users.push(response.user);
      });
      
      call.on('end', () => {
        expect(users.length).toBeGreaterThanOrEqual(0);
        done();
      });
      
      call.on('error', done);
    });
  });
});
```

---

## ขั้นตอนที่ 937: gRPC reflection สำหรับ debugging

```javascript
// server/grpcServerWithReflection.js
const grpc = require('@grpc/grpc-js');
const protoLoader = require('@grpc/proto-loader');
const reflection = require('@grpc/reflection');

const setupReflection = (server) => {
  // เพิ่ม reflection service (สำหรับ tools เช่น grpcurl, grpcui)
  const serviceNames = [
    'user.UserService',
    'product.ProductService'
  ];
  
  const reflectionServer = new reflection.ReflectionService(
    server.grpc_proto, // proto file descriptors
    { services: serviceNames }
  );
  
  reflectionServer.addToServer(server);
  
  console.log('gRPC reflection enabled');
  console.log('Use: grpcurl -plaintext localhost:50051 list');
};

module.exports = setupReflection;
```

---

## ขั้นตอนที่ 938: gRPC Streaming Patterns

```javascript
// patterns/grpcPatterns.js

// Pattern: Transform streaming data
const transformStream = async (inputStream, transformFn, outputStream) => {
  for await (const item of inputStream) {
    const transformed = await transformFn(item);
    if (transformed) {
      outputStream.write(transformed);
    }
  }
  outputStream.end();
};

// Pattern: Aggregated streaming response
const aggregateStream = (call, aggregateFn) => {
  const items = [];
  
  return new Promise((resolve, reject) => {
    call.on('data', (item) => items.push(item));
    call.on('end', async () => {
      try {
        const result = await aggregateFn(items);
        resolve(result);
      } catch (error) {
        reject(error);
      }
    });
    call.on('error', reject);
  });
};

// Pattern: Pagination streaming
const paginatedStream = async (call, fetchPageFn) => {
  let page = 1;
  const pageSize = call.request.page_size || 20;
  
  while (true) {
    const items = await fetchPageFn(page, pageSize);
    
    if (items.length === 0) break;
    
    for (const item of items) {
      call.write(item);
    }
    
    if (items.length < pageSize) break;
    page++;
  }
  
  call.end();
};

// Pattern: Backpressure-aware streaming
const backpressureStream = async (call, items, chunkSize = 10) => {
  for (let i = 0; i < items.length; i += chunkSize) {
    const chunk = items.slice(i, i + chunkSize);
    
    for (const item of chunk) {
      const canContinue = call.write(item);
      
      if (!canContinue) {
        // รอจนกว่า buffer จะว่าง
        await new Promise(resolve => call.once('drain', resolve));
      }
    }
  }
  
  call.end();
};

module.exports = {
  transformStream,
  aggregateStream,
  paginatedStream,
  backpressureStream
};
```

---

## ขั้นตอนที่ 939: Express + gRPC Integration

```javascript
// app.js - รวม Express REST และ gRPC
const express = require('express');
const app = express();
const { startServer: startGRPCServer } = require('./server/grpcServer');

// Express REST API
app.use(express.json());
app.use('/api/v2', require('./routes/v2'));

// gRPC Gateway (แปลง REST ไป gRPC)
const grpcGateway = require('./gateway/grpcGateway');
app.use('/api/grpc', grpcGateway);

// เริ่มทั้ง HTTP และ gRPC servers
const HTTP_PORT = process.env.PORT || 3000;
const GRPC_PORT = process.env.GRPC_PORT || 50051;

const startAll = async () => {
  // เริ่ม gRPC server
  const grpcServer = startGRPCServer(GRPC_PORT);
  console.log(`gRPC server started on port ${GRPC_PORT}`);
  
  // เริ่ม HTTP server
  app.listen(HTTP_PORT, () => {
    console.log(`HTTP server started on port ${HTTP_PORT}`);
  });
  
  // Graceful shutdown
  const shutdown = async () => {
    console.log('Shutting down...');
    
    await new Promise((resolve) => {
      grpcServer.tryShutdown(resolve);
    });
    
    process.exit(0);
  };
  
  process.on('SIGINT', shutdown);
  process.on('SIGTERM', shutdown);
};

startAll().catch(console.error);
```

---

## ขั้นตอนที่ 940: Best Practices สำหรับ gRPC

```javascript
// best-practices/grpcBestPractices.js

/*
  gRPC Best Practices:
  
  1. ใช้ Unary calls สำหรับ simple request-response
  2. ใช้ Server streaming สำหรับ large datasets
  3. ใช้ Client streaming สำหรับ bulk uploads
  4. ใช้ Bidirectional streaming สำหรับ real-time communication
  
  5. ตั้ง timeout สำหรับทุก call
  6. Implement retry logic สำหรับ transient errors
  7. ใช้ TLS ใน production
  8. Enable health checking
  9. Enable reflection ใน development
  10. Monitor metrics ด้วย Prometheus
*/

// Example: Unary call with timeout and deadline
const callWithDeadline = (client, method, request, deadlineMs = 5000) => {
  return new Promise((resolve, reject) => {
    const deadline = new Date();
    deadline.setMilliseconds(deadline.getMilliseconds() + deadlineMs);
    
    client[method](request, { deadline }, (error, response) => {
      if (error) reject(error);
      else resolve(response);
    });
  });
};

// Example: Retry with exponential backoff
const callWithRetry = async (fn, maxRetries = 3, baseDelay = 100) => {
  for (let attempt = 0; attempt < maxRetries; attempt++) {
    try {
      return await fn();
    } catch (error) {
      const isRetryable = [
        grpc.status.UNAVAILABLE,
        grpc.status.RESOURCE_EXHAUSTED,
        grpc.status.DEADLINE_EXCEEDED
      ].includes(error.code);
      
      if (!isRetryable || attempt === maxRetries - 1) {
        throw error;
      }
      
      const delay = baseDelay * Math.pow(2, attempt);
      const jitter = Math.random() * delay * 0.1;
      
      await new Promise(resolve => setTimeout(resolve, delay + jitter));
    }
  }
};

module.exports = { callWithDeadline, callWithRetry };
```

---

## แบบฝึกหัดท้ายบท

### แบบฝึกหัดที่ 1: Product gRPC Service
สร้าง complete gRPC service สำหรับ Product management ด้วย CRUD operations และ streaming

### แบบฝึกหัดที่ 2: Auth Interceptor
สร้าง authentication interceptor ที่ตรวจสอบ JWT token จาก gRPC metadata

### แบบฝึกหัดที่ 3: Load Balancing Test
เปรียบเทียบ throughput ระหว่าง gRPC และ REST สำหรับ bulk operations

### แบบฝึกหัดที่ 4: Bidirectional Chat
สร้าง real-time chat application ด้วย gRPC bidirectional streaming

### แบบฝึกหัดที่ 5: gRPC Gateway
สร้าง HTTP/REST gateway ที่แปลง REST requests ไปยัง gRPC backend services

---

## สรุป

gRPC เหมาะสำหรับ microservices communication ที่ต้องการ high performance, type safety, และ streaming capabilities การใช้ Protocol Buffers ลด network overhead และ serialization time อย่างมีนัยสำคัญ
