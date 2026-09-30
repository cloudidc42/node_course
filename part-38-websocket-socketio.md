# Part 38: WebSocket และ Socket.io

> ขั้นตอนที่ 38-38 จาก 1000

---

## สารบัญ

1. [WebSocket Protocol คืออะไร](#websocket-protocol-คืออะไร)
2. [Socket.io Setup](#socketio-setup)
3. [Events (emit/on)](#events-emiton)
4. [Rooms และ Namespaces](#rooms-และ-namespaces)
5. [Authentication](#authentication)
6. [Scaling ด้วย Redis Adapter](#scaling-ดวย-redis-adapter)
7. [Practical: Real-time Chat](#practical-real-time-chat)
8. [แบบฝึกหัด](#แบบฝึกหัด)

---

## WebSocket Protocol คืออะไร

WebSocket เป็น protocol ที่ให้ full-duplex communication บน TCP connection เดียว

```
HTTP (Polling):
Client → Request  → Server
Client ← Response ← Server
Client → Request  → Server  (ต้องส่งทุกครั้ง)
Client ← Response ← Server

WebSocket:
Client ←→ Connection ←→ Server
  ↕ (ส่งได้ทั้งสองทิศทาง ตลอดเวลา)
  ↕
  ↕
```

### WebSocket vs HTTP

| | HTTP | WebSocket |
|-|------|-----------|
| Connection | ใหม่ทุก request | Persistent |
| Direction | Request/Response | Bidirectional |
| Latency | สูงกว่า | ต่ำกว่า |
| Use case | CRUD operations | Real-time |
| Overhead | Headers ทุก request | ต่ำมาก |

### WebSocket ดิบ ใน Node.js

```javascript
// ไม่ต้อง library เพิ่ม - built into browsers
const WebSocket = require('ws');

const wss = new WebSocket.Server({ port: 8080 });

wss.on('connection', (ws, req) => {
  console.log('Client connected:', req.socket.remoteAddress);
  
  ws.on('message', (data) => {
    console.log('Received:', data.toString());
    
    // Echo กลับ
    ws.send(`Server received: ${data}`);
    
    // ส่งไปทุก client
    wss.clients.forEach(client => {
      if (client.readyState === WebSocket.OPEN) {
        client.send(data.toString());
      }
    });
  });
  
  ws.on('close', () => {
    console.log('Client disconnected');
  });
  
  ws.on('error', (error) => {
    console.error('WebSocket error:', error);
  });
  
  // ส่ง welcome message
  ws.send(JSON.stringify({ type: 'connected', message: 'Welcome!' }));
});

// Client (browser)
const socket = new WebSocket('ws://localhost:8080');
socket.onopen = () => socket.send('Hello Server!');
socket.onmessage = (event) => console.log(event.data);
```

---

## Socket.io Setup

Socket.io เป็น library ที่ทำงานบน WebSocket (พร้อม fallback)

### ติดตั้ง

```bash
npm install socket.io
npm install socket.io-client  # สำหรับ testing หรือ Node.js client
```

### Server Setup

```javascript
// server.js
const express = require('express');
const http = require('http');
const { Server } = require('socket.io');
const cors = require('cors');

const app = express();
const httpServer = http.createServer(app);

// สร้าง Socket.io server
const io = new Server(httpServer, {
  cors: {
    origin: process.env.FRONTEND_URL || 'http://localhost:5173',
    methods: ['GET', 'POST'],
    credentials: true,
  },
  pingTimeout: 60000,        // 60 seconds ก่อน disconnect
  pingInterval: 25000,       // ส่ง ping ทุก 25 seconds
  maxHttpBufferSize: 1e6,    // Max message size: 1MB
});

// Middleware สำหรับ socket connections
io.use((socket, next) => {
  console.log(`Socket ${socket.id} connecting...`);
  next();
});

// Handle connections
io.on('connection', (socket) => {
  console.log(`User connected: ${socket.id}`);
  
  socket.on('disconnect', (reason) => {
    console.log(`User disconnected: ${socket.id}, reason: ${reason}`);
  });
  
  socket.on('error', (error) => {
    console.error(`Socket error: ${socket.id}`, error);
  });
});

// Start server
const PORT = process.env.PORT || 3000;
httpServer.listen(PORT, () => {
  console.log(`Server running on port ${PORT}`);
});

module.exports = { app, io, httpServer };
```

### Client Setup (Browser)

```html
<!-- index.html -->
<!DOCTYPE html>
<html>
<head>
  <script src="https://cdn.socket.io/4.6.0/socket.io.min.js"></script>
</head>
<body>
  <script>
    const socket = io('http://localhost:3000', {
      auth: {
        token: localStorage.getItem('token')
      },
      reconnection: true,
      reconnectionDelay: 1000,
      reconnectionAttempts: 5,
    });
    
    socket.on('connect', () => {
      console.log('Connected:', socket.id);
    });
    
    socket.on('disconnect', (reason) => {
      console.log('Disconnected:', reason);
    });
    
    socket.on('connect_error', (error) => {
      console.error('Connection error:', error.message);
    });
  </script>
</body>
</html>
```

---

## Events (emit/on)

### Emit/On Pattern

```javascript
// === SERVER SIDE ===

io.on('connection', (socket) => {
  
  // รับ event จาก client
  socket.on('message', (data) => {
    console.log('Got message:', data);
  });
  
  // รับ event พร้อม callback (acknowledgement)
  socket.on('join_room', (roomId, callback) => {
    socket.join(roomId);
    callback({ success: true, roomId });
  });
  
  // ส่ง event ไปยัง client คนเดียว
  socket.emit('welcome', { message: 'Hello!', id: socket.id });
  
  // ส่ง event ไปทุก client ยกเว้นตัวเอง
  socket.broadcast.emit('user_joined', { userId: socket.id });
  
  // ส่ง event ไปทุก client รวมตัวเอง
  io.emit('announcement', { message: 'Server message to all' });
  
  // ส่ง event ไปยัง room
  io.to('room1').emit('room_message', { text: 'Hello room 1!' });
  
  // ส่ง event ไปทุก room ยกเว้นตัวเอง
  socket.to('room1').emit('message', { text: 'From another user' });
});
```

### Event Types และ Data

```javascript
// Socket.io รองรับ event data หลาย types
socket.emit('event_name', {
  string: 'hello',
  number: 42,
  boolean: true,
  array: [1, 2, 3],
  object: { key: 'value' },
  buffer: Buffer.from('binary data'),
  // ไม่รองรับ: functions, undefined
});

// ส่งหลาย arguments
socket.emit('event', arg1, arg2, arg3);

socket.on('event', (arg1, arg2, arg3) => {
  console.log(arg1, arg2, arg3);
});
```

### Acknowledgements

```javascript
// Client ส่ง event พร้อม callback
socket.emit('create_room', { name: 'Room 1' }, (response) => {
  if (response.error) {
    console.error('Error:', response.error);
  } else {
    console.log('Room created:', response.roomId);
  }
});

// Server ตอบกลับผ่าน callback
socket.on('create_room', (data, callback) => {
  try {
    const room = createRoom(data.name);
    callback({ success: true, roomId: room.id });
  } catch (error) {
    callback({ error: error.message });
  }
});

// Timeout สำหรับ acknowledgement
socket.timeout(5000).emit('get_data', (err, response) => {
  if (err) {
    console.error('Timeout or error:', err);
  } else {
    console.log('Data:', response);
  }
});
```

---

## Rooms และ Namespaces

### Rooms

```javascript
io.on('connection', (socket) => {
  // เข้า room
  socket.on('join_room', async ({ roomId, userId }) => {
    socket.join(roomId);
    
    // บอก clients อื่นใน room
    socket.to(roomId).emit('user_joined', {
      userId,
      socketId: socket.id,
      timestamp: new Date(),
    });
    
    // ส่ง room info กลับไปยัง user ที่เพิ่งเข้า
    const sockets = await io.in(roomId).allSockets();
    socket.emit('room_joined', {
      roomId,
      members: sockets.size,
    });
  });
  
  // ออกจาก room
  socket.on('leave_room', ({ roomId, userId }) => {
    socket.leave(roomId);
    
    socket.to(roomId).emit('user_left', { userId });
  });
  
  // ส่งข้อความใน room
  socket.on('room_message', ({ roomId, message, userId }) => {
    // ส่งไปทุก client ใน room รวมตัวเอง
    io.to(roomId).emit('new_message', {
      message,
      userId,
      timestamp: new Date(),
    });
  });
  
  // ดู rooms ที่ socket อยู่
  console.log(socket.rooms); // Set { socketId, 'room1', 'room2' }
  
  // Auto leave rooms เมื่อ disconnect
  socket.on('disconnect', () => {
    // socket.rooms ว่างแล้ว เพราะ socket.io จัดการให้
  });
});

// ส่งไปหลาย rooms
io.to('room1').to('room2').emit('event', data);

// ดู sockets ใน room
const sockets = await io.in('room1').allSockets();
console.log(`Users in room1: ${sockets.size}`);
```

### Namespaces

```javascript
// Default namespace: /
io.on('connection', (socket) => { /* ... */ });

// Custom namespaces
const chatNS = io.of('/chat');
const gameNS = io.of('/game');
const adminNS = io.of('/admin');

// Chat namespace
chatNS.on('connection', (socket) => {
  console.log('User in /chat namespace:', socket.id);
  
  socket.on('message', (data) => {
    chatNS.emit('message', data); // ส่งไปทุก users ใน /chat
  });
});

// Admin namespace - ต้อง auth
adminNS.use((socket, next) => {
  const token = socket.handshake.auth.token;
  
  if (!isAdmin(token)) {
    return next(new Error('Admin access required'));
  }
  
  next();
});

adminNS.on('connection', (socket) => {
  socket.on('get_stats', () => {
    const stats = {
      totalConnections: io.engine.clientsCount,
      chatConnections: chatNS.sockets.size,
    };
    socket.emit('stats', stats);
  });
});

// Client เชื่อมต่อ namespace
const chatSocket = io('http://localhost:3000/chat');
const adminSocket = io('http://localhost:3000/admin', {
  auth: { token: 'admin-token' }
});
```

---

## Authentication

### JWT Authentication

```javascript
// middleware/socketAuth.js
const jwt = require('jsonwebtoken');
const config = require('../config');
const User = require('../models/User');

const socketAuth = async (socket, next) => {
  try {
    // ดึง token จาก auth หรือ query
    const token = socket.handshake.auth.token
      || socket.handshake.query.token;
    
    if (!token) {
      return next(new Error('Authentication required'));
    }
    
    // Verify JWT
    const decoded = jwt.verify(token, config.auth.jwtSecret);
    
    // ดึงข้อมูล user
    const user = await User.findById(decoded.id).select('-password');
    
    if (!user || !user.isActive) {
      return next(new Error('User not found or inactive'));
    }
    
    // เก็บ user data ไว้ใน socket
    socket.user = {
      id: user._id.toString(),
      name: user.name,
      email: user.email,
      role: user.role,
    };
    
    next();
  } catch (error) {
    if (error.name === 'JsonWebTokenError') {
      return next(new Error('Invalid token'));
    }
    if (error.name === 'TokenExpiredError') {
      return next(new Error('Token expired'));
    }
    next(error);
  }
};

module.exports = socketAuth;
```

```javascript
// ใช้งาน middleware
const socketAuth = require('./middleware/socketAuth');

// Apply ให้ทุก namespace
io.use(socketAuth);

// หรือเฉพาะบาง namespace
chatNS.use(socketAuth);

// หลังจาก auth แล้ว socket.user มีข้อมูล user
io.on('connection', (socket) => {
  console.log(`User ${socket.user.name} connected`);
  
  // Track online users
  onlineUsers.set(socket.user.id, {
    socketId: socket.id,
    user: socket.user,
    connectedAt: new Date(),
  });
  
  socket.on('disconnect', () => {
    onlineUsers.delete(socket.user.id);
    io.emit('user_offline', { userId: socket.user.id });
  });
});
```

---

## Scaling ด้วย Redis Adapter

Socket.io ใช้ในหลาย server instances ไม่ได้โดยตรง ต้องใช้ Redis adapter

```
Server 1 ←→ Redis ←→ Server 2
   ↕                      ↕
Client A              Client B

Client A emit → Server 1 → Redis → Server 2 → Client B
```

### ติดตั้ง

```bash
npm install @socket.io/redis-adapter ioredis
```

### Setup

```javascript
// server.js
const { createServer } = require('http');
const { Server } = require('socket.io');
const { createAdapter } = require('@socket.io/redis-adapter');
const { createClient } = require('ioredis');

const httpServer = createServer();
const io = new Server(httpServer);

// สร้าง Redis clients (pub/sub)
const pubClient = createClient({
  host: process.env.REDIS_HOST || 'localhost',
  port: parseInt(process.env.REDIS_PORT) || 6379,
  password: process.env.REDIS_PASSWORD,
});

const subClient = pubClient.duplicate();

// Handle Redis errors
pubClient.on('error', (err) => console.error('Redis pub error:', err));
subClient.on('error', (err) => console.error('Redis sub error:', err));

async function startServer() {
  // เชื่อมต่อ Redis
  await Promise.all([
    pubClient.connect(),
    subClient.connect(),
  ]);
  
  // Set adapter
  io.adapter(createAdapter(pubClient, subClient));
  
  console.log('Socket.io connected to Redis adapter');
  
  httpServer.listen(process.env.PORT || 3000);
}

startServer().catch(console.error);
```

---

## Practical: Real-time Chat

### โครงสร้าง Chat System

```javascript
// ======= SERVER =======
// chatServer.js

const express = require('express');
const http = require('http');
const { Server } = require('socket.io');
const jwt = require('jsonwebtoken');
const config = require('./config');
const Message = require('./models/Message');
const Room = require('./models/Room');

const app = express();
const httpServer = http.createServer(app);
const io = new Server(httpServer, {
  cors: { origin: '*', credentials: true }
});

// Online users tracker
const onlineUsers = new Map();

// Socket authentication middleware
io.use((socket, next) => {
  const token = socket.handshake.auth.token;
  if (!token) return next(new Error('Auth required'));
  
  try {
    const user = jwt.verify(token, config.auth.jwtSecret);
    socket.user = user;
    next();
  } catch (err) {
    next(new Error('Invalid token'));
  }
});

io.on('connection', (socket) => {
  const { id: userId, name: userName } = socket.user;
  
  console.log(`${userName} connected`);
  
  // Mark user as online
  onlineUsers.set(userId, { socketId: socket.id, name: userName });
  io.emit('user_status', { userId, status: 'online', name: userName });
  
  // Send current online users
  socket.emit('online_users', Array.from(onlineUsers.entries()).map(([id, data]) => ({
    userId: id,
    ...data,
  })));
  
  // ========================
  // ROOM MANAGEMENT
  // ========================
  
  socket.on('join_room', async ({ roomId }, callback) => {
    try {
      const room = await Room.findById(roomId);
      if (!room) {
        return callback({ error: 'Room not found' });
      }
      
      socket.join(roomId);
      
      // โหลด message history (50 ล่าสุด)
      const messages = await Message.find({ roomId })
        .sort('-createdAt')
        .limit(50)
        .populate('sender', 'name avatar')
        .lean();
      
      // บอก room ว่า user เข้ามา
      socket.to(roomId).emit('user_joined_room', {
        userId,
        userName,
        roomId,
        timestamp: new Date(),
      });
      
      callback({
        success: true,
        room: { id: room._id, name: room.name },
        messages: messages.reverse(),
      });
      
    } catch (error) {
      callback({ error: 'Failed to join room' });
    }
  });
  
  socket.on('leave_room', ({ roomId }) => {
    socket.leave(roomId);
    socket.to(roomId).emit('user_left_room', { userId, userName, roomId });
  });
  
  // ========================
  // MESSAGING
  // ========================
  
  socket.on('send_message', async ({ roomId, content, type = 'text' }, callback) => {
    try {
      if (!content || content.trim().length === 0) {
        return callback?.({ error: 'Message cannot be empty' });
      }
      
      // บันทึกใน database
      const message = await Message.create({
        roomId,
        sender: userId,
        content: content.trim(),
        type,
      });
      
      const populated = await message.populate('sender', 'name avatar');
      
      const messageData = {
        id: message._id,
        content: populated.content,
        type: populated.type,
        sender: {
          id: populated.sender._id,
          name: populated.sender.name,
          avatar: populated.sender.avatar,
        },
        roomId,
        timestamp: message.createdAt,
      };
      
      // ส่งไปทุก client ใน room
      io.to(roomId).emit('new_message', messageData);
      
      callback?.({ success: true, message: messageData });
      
    } catch (error) {
      callback?.({ error: 'Failed to send message' });
    }
  });
  
  // ========================
  // TYPING INDICATORS
  // ========================
  
  socket.on('typing_start', ({ roomId }) => {
    socket.to(roomId).emit('user_typing', { userId, userName, roomId });
  });
  
  socket.on('typing_stop', ({ roomId }) => {
    socket.to(roomId).emit('user_stopped_typing', { userId, roomId });
  });
  
  // ========================
  // DIRECT MESSAGES
  // ========================
  
  socket.on('direct_message', async ({ toUserId, content }, callback) => {
    const targetSocket = onlineUsers.get(toUserId);
    
    if (!targetSocket) {
      // ผู้รับ offline - บันทึกแต่ไม่ส่ง real-time
      await Message.create({
        sender: userId,
        recipient: toUserId,
        content,
        type: 'direct',
        delivered: false,
      });
      
      return callback?.({ success: true, delivered: false });
    }
    
    const message = await Message.create({
      sender: userId,
      recipient: toUserId,
      content,
      type: 'direct',
      delivered: true,
    });
    
    // ส่งไปยัง recipient
    io.to(targetSocket.socketId).emit('new_direct_message', {
      id: message._id,
      from: { id: userId, name: userName },
      content,
      timestamp: message.createdAt,
    });
    
    callback?.({ success: true, delivered: true });
  });
  
  // ========================
  // READ RECEIPTS
  // ========================
  
  socket.on('mark_read', async ({ messageId, roomId }) => {
    await Message.findByIdAndUpdate(messageId, {
      $addToSet: { readBy: userId }
    });
    
    socket.to(roomId).emit('message_read', { messageId, userId, userName });
  });
  
  // ========================
  // DISCONNECT
  // ========================
  
  socket.on('disconnect', () => {
    onlineUsers.delete(userId);
    io.emit('user_status', { userId, status: 'offline' });
    console.log(`${userName} disconnected`);
  });
});

httpServer.listen(3000, () => console.log('Chat server started'));
```

### Client-side Chat

```javascript
// chat.js (Browser)
class ChatClient {
  constructor(token) {
    this.socket = io('http://localhost:3000', {
      auth: { token },
      reconnection: true,
    });
    
    this.setupListeners();
  }
  
  setupListeners() {
    this.socket.on('connect', () => {
      console.log('Connected to chat server');
      this.onConnect?.();
    });
    
    this.socket.on('new_message', (message) => {
      this.onMessage?.(message);
    });
    
    this.socket.on('user_typing', ({ userName, roomId }) => {
      this.onTyping?.(userName, roomId);
    });
    
    this.socket.on('user_stopped_typing', ({ userId, roomId }) => {
      this.onStopTyping?.(userId, roomId);
    });
    
    this.socket.on('online_users', (users) => {
      this.onOnlineUsers?.(users);
    });
    
    this.socket.on('user_status', ({ userId, status }) => {
      this.onStatusChange?.(userId, status);
    });
  }
  
  async joinRoom(roomId) {
    return new Promise((resolve, reject) => {
      this.socket.emit('join_room', { roomId }, (response) => {
        if (response.error) reject(new Error(response.error));
        else resolve(response);
      });
    });
  }
  
  sendMessage(roomId, content) {
    this.socket.emit('send_message', { roomId, content });
  }
  
  startTyping(roomId) {
    this.socket.emit('typing_start', { roomId });
  }
  
  stopTyping(roomId) {
    this.socket.emit('typing_stop', { roomId });
  }
  
  disconnect() {
    this.socket.disconnect();
  }
}

// ใช้งาน
const chat = new ChatClient(localStorage.getItem('token'));

chat.onMessage = (message) => {
  addMessageToUI(message);
};

chat.onTyping = (userName) => {
  showTypingIndicator(userName);
};

await chat.joinRoom('room-id-here');
chat.sendMessage('room-id-here', 'Hello everyone!');
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Basic Socket.io

สร้าง real-time notification system:
- User A ทำ action → notify User B ทันที
- Events: order_placed, payment_received, message_received
- แสดง notification badge count

### แบบฝึกหัดที่ 2: Live Collaboration

สร้าง shared document editor:
- User หลายคน edit document พร้อมกัน
- แสดง cursor position ของแต่ละ user
- Broadcast changes ไปทุก editor

### แบบฝึกหัดที่ 3: Complete Chat App

สร้าง chat application:
- ห้อง chat หลายห้อง
- Online/offline status
- Typing indicators
- Read receipts
- File sharing

---

## สรุป

Socket.io ทำให้ real-time application ง่ายขึ้นมาก

| Feature | ใช้สำหรับ |
|---------|---------|
| emit/on | ส่งและรับ events |
| Rooms | จัดกลุ่ม connections |
| Namespaces | แยก application logic |
| Acknowledgements | Confirm message delivery |
| Redis Adapter | Scale ไป multiple servers |

**ถัดไป**: [Part 39: Caching ด้วย Redis →](./part-39-caching-redis.md)
