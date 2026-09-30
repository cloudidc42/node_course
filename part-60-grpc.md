# Part 60: gRPC (Google Remote Procedure Call)
## ขั้นตอนที่ 60-60 จาก 1000

---

## บทนำ

gRPC เป็น high-performance RPC framework จาก Google ที่ใช้ Protocol Buffers (protobuf) สำหรับ serialization และ HTTP/2 สำหรับ transport เหมาะสำหรับ microservices communication ที่ต้องการ performance สูง

---

## 60.1 gRPC Concepts

### gRPC vs REST

| Feature | REST | gRPC |
|---------|------|------|
| Protocol | HTTP/1.1 | HTTP/2 |
| Data Format | JSON (text) | Protocol Buffers (binary) |
| API Contract | OpenAPI (optional) | .proto (required) |
| Streaming | Limited | Full bidirectional |
| Code Generation | Manual/tools | Built-in |
| Performance | Good | Excellent |
| Browser Support | Full | Limited |

### เมื่อไหร่ควรใช้ gRPC?

- **Microservices communication** - service-to-service calls
- **Polyglot environments** - หลาย programming languages
- **High performance required** - real-time data processing
- **Streaming data** - live feeds, IoT
- **Strong typing** - contract-first API development

---

## 60.2 Protocol Buffers

### ติดตั้ง

```bash
npm install @grpc/grpc-js @grpc/proto-loader
npm install --save-dev grpc-tools  # Code generation
```

### สร้าง .proto file

```protobuf
// protos/user.proto
syntax = "proto3";

package user;

// Service definition
service UserService {
  // Unary RPC
  rpc GetUser (GetUserRequest) returns (UserResponse);
  rpc CreateUser (CreateUserRequest) returns (UserResponse);
  rpc UpdateUser (UpdateUserRequest) returns (UserResponse);
  rpc DeleteUser (DeleteUserRequest) returns (DeleteUserResponse);
  
  // Server streaming RPC
  rpc ListUsers (ListUsersRequest) returns (stream UserResponse);
  
  // Client streaming RPC
  rpc CreateBulkUsers (stream CreateUserRequest) returns (BulkCreateResponse);
  
  // Bidirectional streaming RPC
  rpc Chat (stream ChatMessage) returns (stream ChatMessage);
}

// Message types
message GetUserRequest {
  string id = 1;
}

message CreateUserRequest {
  string name = 1;
  string email = 2;
  string password = 3;
  optional string role = 4;
}

message UpdateUserRequest {
  string id = 1;
  optional string name = 2;
  optional string email = 3;
  optional string avatar = 4;
}

message DeleteUserRequest {
  string id = 1;
}

message ListUsersRequest {
  int32 limit = 1;
  int32 offset = 2;
  optional string role = 3;
}

message UserResponse {
  string id = 1;
  string name = 2;
  string email = 3;
  string role = 4;
  string created_at = 5;
  optional string avatar = 6;
}

message DeleteUserResponse {
  bool success = 1;
  string message = 2;
}

message BulkCreateResponse {
  int32 created = 1;
  int32 failed = 2;
  repeated string errors = 3;
}

message ChatMessage {
  string user_id = 1;
  string content = 2;
  string timestamp = 3;
}
```

```protobuf
// protos/product.proto
syntax = "proto3";

package product;

service ProductService {
  rpc GetProduct (GetProductRequest) returns (ProductResponse);
  rpc ListProducts (ListProductsRequest) returns (ListProductsResponse);
  rpc StreamProductUpdates (ProductUpdateFilter) returns (stream ProductResponse);
}

message GetProductRequest {
  string id = 1;
}

message ListProductsRequest {
  int32 limit = 1;
  int32 offset = 2;
  optional string category = 3;
  optional float min_price = 4;
  optional float max_price = 5;
}

message ListProductsResponse {
  repeated ProductResponse products = 1;
  int32 total = 2;
}

message ProductResponse {
  string id = 1;
  string name = 2;
  string description = 3;
  float price = 4;
  int32 stock = 5;
  string category = 6;
  repeated string images = 7;
}

message ProductUpdateFilter {
  optional string category = 1;
  bool include_stock_updates = 2;
  bool include_price_updates = 3;
}
```

---

## 60.3 Unary RPC

```javascript
// src/grpc/server/userServer.js
const grpc = require('@grpc/grpc-js');
const protoLoader = require('@grpc/proto-loader');
const path = require('path');
const User = require('../../models/User');
const bcrypt = require('bcrypt');

// Load proto file
const PROTO_PATH = path.join(__dirname, '../../../protos/user.proto');

const packageDefinition = protoLoader.loadSync(PROTO_PATH, {
  keepCase: true,
  longs: String,
  enums: String,
  defaults: true,
  oneofs: true
});

const protoDescriptor = grpc.loadPackageDefinition(packageDefinition);
const userProto = protoDescriptor.user;

/**
 * Unary RPC implementations
 */
const getUser = async (call, callback) => {
  try {
    const { id } = call.request;
    
    const user = await User.findByPk(id, {
      attributes: { exclude: ['password'] }
    });
    
    if (!user) {
      return callback({
        code: grpc.status.NOT_FOUND,
        message: `User ${id} not found`
      });
    }
    
    callback(null, {
      id: user.id.toString(),
      name: user.name,
      email: user.email,
      role: user.role,
      created_at: user.createdAt.toISOString(),
      avatar: user.avatar || ''
    });
  } catch (error) {
    callback({
      code: grpc.status.INTERNAL,
      message: error.message
    });
  }
};

const createUser = async (call, callback) => {
  try {
    const { name, email, password, role } = call.request;
    
    // Check existing email
    const existing = await User.findOne({ where: { email } });
    if (existing) {
      return callback({
        code: grpc.status.ALREADY_EXISTS,
        message: 'Email already in use'
      });
    }
    
    const hashedPassword = await bcrypt.hash(password, 12);
    
    const user = await User.create({
      name,
      email,
      password: hashedPassword,
      role: role || 'USER'
    });
    
    callback(null, {
      id: user.id.toString(),
      name: user.name,
      email: user.email,
      role: user.role,
      created_at: user.createdAt.toISOString()
    });
  } catch (error) {
    callback({
      code: grpc.status.INTERNAL,
      message: error.message
    });
  }
};

const updateUser = async (call, callback) => {
  try {
    const { id, name, email, avatar } = call.request;
    
    const user = await User.findByPk(id);
    if (!user) {
      return callback({
        code: grpc.status.NOT_FOUND,
        message: `User ${id} not found`
      });
    }
    
    const updates = {};
    if (name) updates.name = name;
    if (email) updates.email = email;
    if (avatar) updates.avatar = avatar;
    
    await user.update(updates);
    
    callback(null, {
      id: user.id.toString(),
      name: user.name,
      email: user.email,
      role: user.role,
      created_at: user.createdAt.toISOString(),
      avatar: user.avatar || ''
    });
  } catch (error) {
    callback({
      code: grpc.status.INTERNAL,
      message: error.message
    });
  }
};

const deleteUser = async (call, callback) => {
  try {
    const { id } = call.request;
    const deleted = await User.destroy({ where: { id } });
    
    callback(null, {
      success: deleted > 0,
      message: deleted > 0 ? 'User deleted' : 'User not found'
    });
  } catch (error) {
    callback({
      code: grpc.status.INTERNAL,
      message: error.message
    });
  }
};

module.exports = { getUser, createUser, updateUser, deleteUser };
```

---

## 60.4 Server Streaming

```javascript
// src/grpc/server/streaming/serverStreaming.js

/**
 * Server Streaming - ส่ง stream ของ users
 */
const listUsers = async (call) => {
  try {
    const { limit = 10, offset = 0, role } = call.request;
    
    const where = role ? { role } : {};
    
    const users = await User.findAll({
      where,
      limit,
      offset,
      attributes: { exclude: ['password'] },
      order: [['createdAt', 'DESC']]
    });
    
    // ส่งทีละ record
    for (const user of users) {
      // ตรวจสอบว่า client ยังเชื่อมต่ออยู่
      if (call.cancelled) {
        console.log('Client cancelled the stream');
        break;
      }
      
      call.write({
        id: user.id.toString(),
        name: user.name,
        email: user.email,
        role: user.role,
        created_at: user.createdAt.toISOString()
      });
      
      // Simulate delay (optional)
      await new Promise(resolve => setTimeout(resolve, 10));
    }
    
    call.end();
  } catch (error) {
    call.destroy(error);
  }
};

/**
 * Server streaming สำหรับ real-time product updates
 */
const streamProductUpdates = async (call) => {
  const { category, include_stock_updates, include_price_updates } = call.request;
  
  console.log(`Client subscribed to product updates (category: ${category || 'all'})`);
  
  // Subscribe to product events
  const eventEmitter = require('../../events/productEvents');
  
  const handleProductUpdate = (product) => {
    if (call.cancelled) return;
    
    // Filter by category
    if (category && product.category !== category) return;
    
    // Filter by update type
    if (!include_stock_updates && product.updateType === 'stock') return;
    if (!include_price_updates && product.updateType === 'price') return;
    
    call.write({
      id: product.id.toString(),
      name: product.name,
      price: product.price,
      stock: product.stock,
      category: product.category
    });
  };
  
  eventEmitter.on('product:updated', handleProductUpdate);
  
  // Cleanup เมื่อ client disconnect
  call.on('cancelled', () => {
    eventEmitter.off('product:updated', handleProductUpdate);
    console.log('Client disconnected from product stream');
  });
};

module.exports = { listUsers, streamProductUpdates };
```

---

## 60.5 Client Streaming

```javascript
// src/grpc/server/streaming/clientStreaming.js

/**
 * Client Streaming - รับ stream ของ users และ bulk create
 */
const createBulkUsers = async (call, callback) => {
  const results = {
    created: 0,
    failed: 0,
    errors: []
  };
  
  // รับข้อมูลจาก stream
  call.on('data', async (userData) => {
    try {
      const { name, email, password } = userData;
      
      const existing = await User.findOne({ where: { email } });
      if (existing) {
        results.failed++;
        results.errors.push(`Email ${email} already exists`);
        return;
      }
      
      const hashedPassword = await bcrypt.hash(password, 10);
      await User.create({ name, email, password: hashedPassword });
      results.created++;
    } catch (error) {
      results.failed++;
      results.errors.push(error.message);
    }
  });
  
  call.on('end', () => {
    // ส่ง response เมื่อ client ส่งข้อมูลครบ
    callback(null, results);
  });
  
  call.on('error', (error) => {
    console.error('Client stream error:', error);
    callback({
      code: grpc.status.INTERNAL,
      message: error.message
    });
  });
};

module.exports = { createBulkUsers };
```

---

## 60.6 Bidirectional Streaming

```javascript
// src/grpc/server/streaming/bidirectional.js
const { EventEmitter } = require('events');
const chatRooms = new Map();

/**
 * Bidirectional Streaming - Chat
 */
const chat = (call) => {
  let userId = null;
  let roomId = null;
  
  call.on('data', (message) => {
    // ข้อความแรกระบุ user และ room
    if (!userId) {
      userId = message.user_id;
      roomId = message.content.startsWith('JOIN:') 
        ? message.content.replace('JOIN:', '')
        : 'general';
      
      // เข้าร่วม room
      if (!chatRooms.has(roomId)) {
        chatRooms.set(roomId, new Set());
      }
      chatRooms.get(roomId).add(call);
      
      // แจ้ง users อื่นใน room
      broadcastToRoom(roomId, {
        user_id: 'system',
        content: `${userId} joined the room`,
        timestamp: new Date().toISOString()
      }, call);
      
      return;
    }
    
    // Broadcast ไปยัง users อื่นใน room
    broadcastToRoom(roomId, {
      user_id: userId,
      content: message.content,
      timestamp: new Date().toISOString()
    }, call);
  });
  
  call.on('end', () => {
    // ออกจาก room
    if (roomId && chatRooms.has(roomId)) {
      chatRooms.get(roomId).delete(call);
      
      broadcastToRoom(roomId, {
        user_id: 'system',
        content: `${userId} left the room`,
        timestamp: new Date().toISOString()
      });
    }
    
    call.end();
  });
  
  call.on('error', (error) => {
    console.error('Chat stream error:', error);
    if (roomId && chatRooms.has(roomId)) {
      chatRooms.get(roomId).delete(call);
    }
  });
};

const broadcastToRoom = (roomId, message, excludeCall = null) => {
  const room = chatRooms.get(roomId);
  if (!room) return;
  
  for (const clientCall of room) {
    if (clientCall !== excludeCall && !clientCall.cancelled) {
      try {
        clientCall.write(message);
      } catch (error) {
        console.error('Error broadcasting to client:', error.message);
      }
    }
  }
};

module.exports = { chat };
```

### กรวม Implementations

```javascript
// src/grpc/server/userService.js
const grpc = require('@grpc/grpc-js');
const protoLoader = require('@grpc/proto-loader');
const path = require('path');

const { getUser, createUser, updateUser, deleteUser } = require('./userServer');
const { listUsers } = require('./streaming/serverStreaming');
const { createBulkUsers } = require('./streaming/clientStreaming');
const { chat } = require('./streaming/bidirectional');

const PROTO_PATH = path.join(__dirname, '../../../protos/user.proto');

const packageDefinition = protoLoader.loadSync(PROTO_PATH, {
  keepCase: true,
  longs: String,
  enums: String,
  defaults: true,
  oneofs: true
});

const protoDescriptor = grpc.loadPackageDefinition(packageDefinition);
const userProto = protoDescriptor.user;

const startUserGrpcServer = (port = 50051) => {
  const server = new grpc.Server();
  
  server.addService(userProto.UserService.service, {
    getUser,
    createUser,
    updateUser,
    deleteUser,
    listUsers,
    createBulkUsers,
    chat
  });
  
  server.bindAsync(
    `0.0.0.0:${port}`,
    grpc.ServerCredentials.createInsecure(),
    (error, actualPort) => {
      if (error) {
        console.error('Failed to start gRPC server:', error);
        return;
      }
      console.log(`🚀 gRPC server running on port ${actualPort}`);
    }
  );
  
  return server;
};

module.exports = { startUserGrpcServer };
```

---

## 60.7 gRPC Client

```javascript
// src/grpc/client/userClient.js
const grpc = require('@grpc/grpc-js');
const protoLoader = require('@grpc/proto-loader');
const path = require('path');
const { promisify } = require('util');

const PROTO_PATH = path.join(__dirname, '../../../protos/user.proto');

const packageDefinition = protoLoader.loadSync(PROTO_PATH, {
  keepCase: true,
  longs: String,
  enums: String,
  defaults: true,
  oneofs: true
});

const protoDescriptor = grpc.loadPackageDefinition(packageDefinition);
const userProto = protoDescriptor.user;

// สร้าง client
const userClient = new userProto.UserService(
  `${process.env.USER_SERVICE_HOST || 'localhost'}:50051`,
  grpc.credentials.createInsecure()
);

// Promisify unary methods
const getUser = promisify(userClient.getUser.bind(userClient));
const createUser = promisify(userClient.createUser.bind(userClient));
const updateUser = promisify(userClient.updateUser.bind(userClient));
const deleteUser = promisify(userClient.deleteUser.bind(userClient));

/**
 * Server streaming - อ่าน stream ทั้งหมด
 */
const listAllUsers = (limit = 100) => {
  return new Promise((resolve, reject) => {
    const users = [];
    
    const stream = userClient.listUsers({ limit });
    
    stream.on('data', (user) => {
      users.push(user);
    });
    
    stream.on('end', () => resolve(users));
    stream.on('error', reject);
  });
};

/**
 * Client streaming - ส่ง bulk users
 */
const createBulkUsers = (usersData) => {
  return new Promise((resolve, reject) => {
    const call = userClient.createBulkUsers((error, response) => {
      if (error) reject(error);
      else resolve(response);
    });
    
    for (const userData of usersData) {
      call.write(userData);
    }
    
    call.end();
  });
};

/**
 * Bidirectional streaming - Chat
 */
const startChat = (userId, roomId, onMessage) => {
  const call = userClient.chat();
  
  // Join room
  call.write({
    user_id: userId,
    content: `JOIN:${roomId}`,
    timestamp: new Date().toISOString()
  });
  
  call.on('data', onMessage);
  call.on('error', console.error);
  call.on('end', () => console.log('Chat ended'));
  
  const sendMessage = (content) => {
    call.write({
      user_id: userId,
      content,
      timestamp: new Date().toISOString()
    });
  };
  
  const disconnect = () => call.end();
  
  return { sendMessage, disconnect };
};

module.exports = { getUser, createUser, updateUser, deleteUser, listAllUsers, createBulkUsers, startChat };
```

---

## แบบฝึกหัดที่ 60

### แบบฝึกหัดพื้นฐาน

**1. Unary gRPC Service**

สร้าง Product Service ด้วย gRPC:
- GetProduct, ListProducts, CreateProduct
- กรณีไม่พบ: return NOT_FOUND error
- Unit tests สำหรับ each method

**2. Server Streaming**

สร้าง streaming endpoint:
- Stream product inventory updates แบบ real-time
- Client สามารถ filter by category
- Handle client disconnection gracefully

### แบบฝึกหัดขั้นสูง

**3. Bidirectional Chat**

สร้าง real-time chat ด้วย bidirectional streaming:
- Multiple chat rooms
- Message history
- Online user list

**4. gRPC Gateway**

สร้าง HTTP/REST gateway ที่แปลง requests ไป gRPC:
- GET /users/:id → gRPC GetUser
- POST /users → gRPC CreateUser
- Server-sent events สำหรับ streaming

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **gRPC vs REST** - เมื่อไหรควรใช้อะไร
2. **Protocol Buffers** - schema definition
3. **Unary RPC** - basic request-response
4. **Server Streaming** - server sends multiple responses
5. **Client Streaming** - client sends multiple requests
6. **Bidirectional Streaming** - both sides stream

---

## สรุปภาพรวม Part 46-60

| Part | หัวข้อ | ความสำคัญ |
|------|--------|-----------|
| 46 | Pagination | ⭐⭐⭐⭐⭐ |
| 47 | Search & Filtering | ⭐⭐⭐⭐⭐ |
| 48 | Image Processing | ⭐⭐⭐⭐ |
| 49 | Background Jobs | ⭐⭐⭐⭐⭐ |
| 50 | Webhooks | ⭐⭐⭐⭐ |
| 51 | Microservices | ⭐⭐⭐⭐⭐ |
| 52 | Docker | ⭐⭐⭐⭐⭐ |
| 53 | CI/CD | ⭐⭐⭐⭐⭐ |
| 54 | AWS Deployment | ⭐⭐⭐⭐ |
| 55 | Performance | ⭐⭐⭐⭐⭐ |
| 56 | DB Optimization | ⭐⭐⭐⭐⭐ |
| 57 | API Gateway | ⭐⭐⭐⭐ |
| 58 | Message Queue | ⭐⭐⭐⭐⭐ |
| 59 | GraphQL | ⭐⭐⭐⭐ |
| 60 | gRPC | ⭐⭐⭐⭐ |

**ถัดไป:** Part 61-75 - Advanced Topics
