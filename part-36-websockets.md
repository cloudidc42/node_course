# Part 36 | ขั้นตอนที่ 621-640 จาก 1000

## WebSockets และ Socket.io - การสื่อสาร Real-time

---

## สารบัญ

1. [WebSocket Protocol](#websocket-protocol)
2. [Socket.io เบื้องต้น](#socketio-เบื้องต้น)
3. [Events และ Namespaces](#events-และ-namespaces)
4. [Rooms และ Broadcasting](#rooms-และ-broadcasting)
5. [Real-time Chat Application](#real-time-chat-application)
6. [Authentication ใน WebSockets](#authentication-ใน-websockets)
7. [Scaling Socket.io](#scaling-socketio)
8. [แบบฝึกหัด](#แบบฝึกหัด)

---

## WebSocket Protocol

### ขั้นตอนที่ 621: WebSocket คืออะไร

WebSocket เป็น communication protocol ที่ให้ full-duplex communication ผ่าน TCP connection เดียว

```
HTTP (Request-Response):
  Client → Request → Server
  Client ← Response ← Server
  (connection ปิด)

WebSocket (Persistent Connection):
  Client ←→ Server (connection เปิดค้างไว้)
  Server สามารถ push data ได้ทุกเวลา
```

### ขั้นตอนที่ 622: WebSocket Handshake

```
1. Client ส่ง HTTP Upgrade Request:
   GET /ws HTTP/1.1
   Host: example.com
   Upgrade: websocket
   Connection: Upgrade
   Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
   Sec-WebSocket-Version: 13

2. Server ตอบกลับ:
   HTTP/1.1 101 Switching Protocols
   Upgrade: websocket
   Connection: Upgrade
   Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=

3. เปิด WebSocket connection สำเร็จ!
```

### ขั้นตอนที่ 623: Native WebSocket API

```javascript
// server.js - WebSocket Server โดยไม่ใช้ library
const http = require('http');
const WebSocket = require('ws');

const server = http.createServer();
const wss = new WebSocket.Server({ server });

// เก็บ clients ที่เชื่อมต่อ
const clients = new Map();

wss.on('connection', (ws, req) => {
  const clientId = generateId();
  clients.set(clientId, ws);

  console.log(`Client ${clientId} connected. Total: ${clients.size}`);

  // ส่ง welcome message
  ws.send(JSON.stringify({
    type: 'CONNECTED',
    clientId,
    message: 'ยินดีต้อนรับ!',
  }));

  // รับ messages
  ws.on('message', (data) => {
    try {
      const message = JSON.parse(data);
      console.log(`Message from ${clientId}:`, message);

      // จัดการตาม message type
      handleMessage(clientId, ws, message);
    } catch (error) {
      ws.send(JSON.stringify({ type: 'ERROR', message: 'Invalid JSON' }));
    }
  });

  // Handle disconnect
  ws.on('close', (code, reason) => {
    clients.delete(clientId);
    console.log(`Client ${clientId} disconnected. Code: ${code}`);
  });

  // Handle errors
  ws.on('error', (error) => {
    console.error(`Client ${clientId} error:`, error);
    clients.delete(clientId);
  });
});

function handleMessage(clientId, ws, message) {
  switch (message.type) {
    case 'BROADCAST':
      // ส่งให้ทุก clients
      broadcast(message.data, clientId);
      break;

    case 'PING':
      ws.send(JSON.stringify({ type: 'PONG', timestamp: Date.now() }));
      break;

    default:
      ws.send(JSON.stringify({ type: 'ERROR', message: 'Unknown message type' }));
  }
}

function broadcast(data, excludeClientId) {
  clients.forEach((client, clientId) => {
    if (clientId !== excludeClientId && client.readyState === WebSocket.OPEN) {
      client.send(JSON.stringify({ type: 'BROADCAST', data }));
    }
  });
}

function generateId() {
  return Math.random().toString(36).substr(2, 9);
}

server.listen(3000, () => {
  console.log('WebSocket Server running on port 3000');
});
```

```javascript
// client.js - WebSocket Client
const ws = new WebSocket('ws://localhost:3000');

ws.onopen = () => {
  console.log('Connected to server');

  // ส่ง message
  ws.send(JSON.stringify({
    type: 'BROADCAST',
    data: 'Hello everyone!',
  }));
};

ws.onmessage = (event) => {
  const message = JSON.parse(event.data);
  console.log('Received:', message);
};

ws.onclose = (event) => {
  console.log('Disconnected:', event.code, event.reason);
};

ws.onerror = (error) => {
  console.error('Error:', error);
};
```

---

## Socket.io เบื้องต้น

### ขั้นตอนที่ 624: ติดตั้ง Socket.io

```bash
mkdir realtime-app && cd realtime-app
npm init -y
npm install express socket.io
npm install --save-dev nodemon
```

### ขั้นตอนที่ 625: Socket.io Server

```javascript
// server.js
const express = require('express');
const http = require('http');
const { Server } = require('socket.io');
const path = require('path');

const app = express();
const httpServer = http.createServer(app);

const io = new Server(httpServer, {
  cors: {
    origin: process.env.CLIENT_URL || 'http://localhost:3000',
    methods: ['GET', 'POST'],
  },
  // ตั้งค่า connection options
  pingTimeout: 60000,
  pingInterval: 25000,
  transports: ['websocket', 'polling'],
});

// Serve static files
app.use(express.static(path.join(__dirname, 'public')));

// Socket.io connection handler
io.on('connection', (socket) => {
  console.log(`User connected: ${socket.id}`);

  // ส่ง welcome event
  socket.emit('welcome', {
    message: 'ยินดีต้อนรับสู่ระบบ!',
    socketId: socket.id,
  });

  // รับ custom events
  socket.on('chat message', (msg) => {
    console.log('Message:', msg);
    // ส่งให้ทุกคน
    io.emit('chat message', {
      id: Date.now(),
      text: msg,
      sender: socket.id,
      timestamp: new Date().toISOString(),
    });
  });

  // Disconnect
  socket.on('disconnect', (reason) => {
    console.log(`User disconnected: ${socket.id}, reason: ${reason}`);
  });
});

httpServer.listen(3000, () => {
  console.log('Server running on http://localhost:3000');
});
```

### ขั้นตอนที่ 626: Socket.io Client

```html
<!-- public/index.html -->
<!DOCTYPE html>
<html lang="th">
<head>
  <meta charset="UTF-8">
  <title>Real-time Chat</title>
  <style>
    body { font-family: sans-serif; max-width: 600px; margin: 50px auto; }
    #messages { height: 400px; overflow-y: auto; border: 1px solid #ccc; padding: 10px; }
    .message { margin: 5px 0; padding: 5px 10px; border-radius: 5px; }
    .own { background: #007bff; color: white; text-align: right; }
    .other { background: #f1f1f1; }
    #form { display: flex; gap: 10px; margin-top: 10px; }
    #input { flex: 1; padding: 8px; }
    button { padding: 8px 16px; background: #007bff; color: white; border: none; cursor: pointer; }
  </style>
</head>
<body>
  <h1>💬 Real-time Chat</h1>
  <div id="status">กำลังเชื่อมต่อ...</div>
  <ul id="messages"></ul>
  <form id="form">
    <input id="input" placeholder="พิมพ์ข้อความ..." autocomplete="off" />
    <button type="submit">ส่ง</button>
  </form>

  <script src="/socket.io/socket.io.js"></script>
  <script>
    const socket = io();
    const messages = document.getElementById('messages');
    const form = document.getElementById('form');
    const input = document.getElementById('input');
    const status = document.getElementById('status');

    // Connection events
    socket.on('connect', () => {
      status.textContent = `เชื่อมต่อแล้ว (${socket.id})`;
      status.style.color = 'green';
    });

    socket.on('disconnect', () => {
      status.textContent = 'ขาดการเชื่อมต่อ - กำลังเชื่อมต่อใหม่...';
      status.style.color = 'red';
    });

    socket.on('welcome', (data) => {
      addSystemMessage(data.message);
    });

    // Receive messages
    socket.on('chat message', (msg) => {
      const isOwn = msg.sender === socket.id;
      addMessage(msg.text, isOwn);
    });

    // Send message
    form.addEventListener('submit', (e) => {
      e.preventDefault();
      if (input.value.trim()) {
        socket.emit('chat message', input.value);
        input.value = '';
      }
    });

    function addMessage(text, isOwn) {
      const li = document.createElement('li');
      li.className = `message ${isOwn ? 'own' : 'other'}`;
      li.textContent = text;
      messages.appendChild(li);
      messages.scrollTop = messages.scrollHeight;
    }

    function addSystemMessage(text) {
      const li = document.createElement('li');
      li.style.color = '#888';
      li.style.textAlign = 'center';
      li.textContent = text;
      messages.appendChild(li);
    }
  </script>
</body>
</html>
```

---

## Events และ Namespaces

### ขั้นตอนที่ 627: Namespaces

Namespaces ช่วยให้แบ่ง Socket.io server ออกเป็น "sections" ที่แตกต่างกัน

```javascript
// server.js - Namespaces

// Default namespace: /
io.on('connection', (socket) => {
  socket.emit('welcome', 'Default namespace');
});

// Chat namespace: /chat
const chatNS = io.of('/chat');
chatNS.on('connection', (socket) => {
  console.log('Chat user connected:', socket.id);

  socket.on('message', (msg) => {
    chatNS.emit('message', { sender: socket.id, text: msg });
  });
});

// Notification namespace: /notifications
const notifNS = io.of('/notifications');
notifNS.on('connection', (socket) => {
  console.log('Notification user connected:', socket.id);

  // ส่ง notification ไปยัง client คนนี้
  socket.emit('notification', {
    type: 'INFO',
    message: 'ยินดีต้อนรับสู่ระบบ notifications',
  });
});

// Admin namespace: /admin
const adminNS = io.of('/admin');

// Middleware สำหรับ namespace
adminNS.use((socket, next) => {
  const token = socket.handshake.auth.token;
  if (!verifyAdminToken(token)) {
    next(new Error('ไม่มีสิทธิ์เข้าถึง admin namespace'));
    return;
  }
  next();
});

adminNS.on('connection', (socket) => {
  // Admin-only events
  socket.emit('adminData', { users: 100, messages: 5000 });
});
```

```javascript
// client.js - เชื่อมต่อ namespace
// Default
const defaultSocket = io('http://localhost:3000');

// Chat namespace
const chatSocket = io('http://localhost:3000/chat');

// Admin namespace พร้อม auth
const adminSocket = io('http://localhost:3000/admin', {
  auth: {
    token: 'admin-jwt-token',
  },
});

chatSocket.on('message', (msg) => {
  console.log('Chat message:', msg);
});
```

---

## Rooms และ Broadcasting

### ขั้นตอนที่ 628: Rooms

```javascript
// server.js - Room Management
io.on('connection', (socket) => {
  // Join a room
  socket.on('joinRoom', ({ roomId, username }) => {
    socket.join(roomId);
    socket.data.username = username;
    socket.data.roomId = roomId;

    // แจ้งคนในห้องว่ามีคนเข้ามา
    socket.to(roomId).emit('userJoined', {
      socketId: socket.id,
      username,
      message: `${username} เข้าร่วมห้อง`,
    });

    // ดึงรายชื่อคนในห้อง
    const roomMembers = getRoomMembers(roomId);
    socket.emit('roomInfo', {
      roomId,
      members: roomMembers,
    });

    console.log(`${username} joined room: ${roomId}`);
  });

  // Leave a room
  socket.on('leaveRoom', ({ roomId }) => {
    socket.leave(roomId);

    socket.to(roomId).emit('userLeft', {
      socketId: socket.id,
      username: socket.data.username,
    });
  });

  // Send message to room
  socket.on('roomMessage', ({ roomId, message }) => {
    const msgData = {
      id: Date.now().toString(),
      sender: socket.data.username,
      senderId: socket.id,
      text: message,
      roomId,
      timestamp: new Date().toISOString(),
    };

    // ส่งให้ทุกคนในห้อง รวมถึง sender
    io.to(roomId).emit('roomMessage', msgData);
  });

  // Private message (ระหว่าง 2 users)
  socket.on('privateMessage', ({ toSocketId, message }) => {
    const msgData = {
      from: socket.data.username,
      fromId: socket.id,
      text: message,
      timestamp: new Date().toISOString(),
    };

    // ส่งให้ recipient
    socket.to(toSocketId).emit('privateMessage', msgData);

    // ส่ง confirmation กลับมาที่ sender
    socket.emit('messageSent', { ...msgData, to: toSocketId });
  });

  // Disconnect
  socket.on('disconnect', () => {
    const { username, roomId } = socket.data;
    if (roomId) {
      socket.to(roomId).emit('userLeft', {
        socketId: socket.id,
        username,
      });
    }
  });
});

// Helper function
function getRoomMembers(roomId) {
  const room = io.sockets.adapter.rooms.get(roomId);
  if (!room) return [];

  return Array.from(room).map(socketId => {
    const socket = io.sockets.sockets.get(socketId);
    return {
      socketId,
      username: socket?.data.username,
    };
  });
}
```

### ขั้นตอนที่ 629: Broadcasting Patterns

```javascript
// Broadcasting Patterns
io.on('connection', (socket) => {
  // ส่งให้ทุกคน (รวม sender)
  io.emit('event', data);

  // ส่งให้ทุกคน ยกเว้น sender
  socket.broadcast.emit('event', data);

  // ส่งให้คนใน room เฉพาะ
  io.to('roomId').emit('event', data);

  // ส่งให้คนใน room ยกเว้น sender
  socket.to('roomId').emit('event', data);

  // ส่งให้หลาย rooms
  io.to('room1').to('room2').emit('event', data);

  // ส่งให้ socket เฉพาะ
  io.to(socketId).emit('event', data);

  // ส่งให้ namespace ทั้งหมด
  io.of('/chat').emit('event', data);

  // ส่งแบบ volatile (ถ้า client ไม่พร้อมรับ ก็ข้ามไป)
  socket.volatile.emit('event', data);

  // Acknowledge (callback เมื่อ client ได้รับแล้ว)
  socket.emit('event', data, (response) => {
    console.log('Client confirmed:', response);
  });
});
```

---

## Real-time Chat Application

### ขั้นตอนที่ 630: Chat Server สมบูรณ์

```javascript
// chat-server.js
const express = require('express');
const http = require('http');
const { Server } = require('socket.io');
const jwt = require('jsonwebtoken');

const app = express();
const httpServer = http.createServer(app);

const io = new Server(httpServer, {
  cors: { origin: '*' },
});

// เก็บข้อมูล in-memory (ใน production ใช้ Redis/DB)
const rooms = new Map();      // roomId → { name, members, messages }
const userSockets = new Map(); // userId → socketId

// Authentication Middleware
io.use((socket, next) => {
  const token = socket.handshake.auth.token;
  if (!token) {
    return next(new Error('ต้องมี token'));
  }

  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    socket.user = decoded;
    next();
  } catch (err) {
    next(new Error('Token ไม่ถูกต้อง'));
  }
});

io.on('connection', (socket) => {
  const { user } = socket;
  console.log(`✅ ${user.username} connected (${socket.id})`);

  // เก็บ socket ของ user
  userSockets.set(user.id, socket.id);

  // ส่ง rooms ที่มีอยู่
  socket.emit('rooms', Array.from(rooms.values()).map(r => ({
    id: r.id,
    name: r.name,
    memberCount: r.members.size,
  })));

  // === Room Events ===

  // สร้างห้องใหม่
  socket.on('createRoom', ({ name }, callback) => {
    const roomId = generateRoomId();
    const room = {
      id: roomId,
      name,
      creator: user.id,
      members: new Set(),
      messages: [],
      createdAt: new Date(),
    };

    rooms.set(roomId, room);

    // Broadcast ให้ทุกคนรู้ว่ามีห้องใหม่
    io.emit('roomCreated', {
      id: roomId,
      name,
      creator: user.username,
    });

    callback({ success: true, roomId });
  });

  // เข้าห้อง
  socket.on('joinRoom', ({ roomId }, callback) => {
    const room = rooms.get(roomId);
    if (!room) {
      return callback({ success: false, error: 'ไม่พบห้อง' });
    }

    socket.join(roomId);
    room.members.add(user.id);

    // ส่ง history messages (50 ล่าสุด)
    const recentMessages = room.messages.slice(-50);
    socket.emit('messageHistory', recentMessages);

    // แจ้งคนในห้อง
    socket.to(roomId).emit('userJoined', {
      userId: user.id,
      username: user.username,
      roomId,
    });

    callback({
      success: true,
      room: {
        id: room.id,
        name: room.name,
        memberCount: room.members.size,
      },
    });
  });

  // ออกห้อง
  socket.on('leaveRoom', ({ roomId }) => {
    const room = rooms.get(roomId);
    if (room) {
      room.members.delete(user.id);
      socket.leave(roomId);

      socket.to(roomId).emit('userLeft', {
        userId: user.id,
        username: user.username,
        roomId,
      });
    }
  });

  // === Message Events ===

  // ส่ง message
  socket.on('sendMessage', ({ roomId, text, type = 'text' }, callback) => {
    const room = rooms.get(roomId);
    if (!room) {
      return callback({ success: false, error: 'ไม่พบห้อง' });
    }

    if (!room.members.has(user.id)) {
      return callback({ success: false, error: 'ยังไม่ได้เข้าห้องนี้' });
    }

    const message = {
      id: generateMessageId(),
      roomId,
      sender: {
        id: user.id,
        username: user.username,
        avatar: user.avatar,
      },
      text,
      type,
      timestamp: new Date().toISOString(),
      reactions: {},
    };

    // เก็บใน history
    room.messages.push(message);
    if (room.messages.length > 1000) {
      room.messages.shift(); // ลบ message เก่าที่สุด
    }

    // ส่งให้ทุกคนในห้อง
    io.to(roomId).emit('newMessage', message);

    callback({ success: true, messageId: message.id });
  });

  // React to message
  socket.on('reactToMessage', ({ roomId, messageId, emoji }) => {
    const room = rooms.get(roomId);
    if (!room) return;

    const message = room.messages.find(m => m.id === messageId);
    if (!message) return;

    if (!message.reactions[emoji]) {
      message.reactions[emoji] = new Set();
    }

    // Toggle reaction
    if (message.reactions[emoji].has(user.id)) {
      message.reactions[emoji].delete(user.id);
    } else {
      message.reactions[emoji].add(user.id);
    }

    // Serialize สำหรับ emit
    const reactionsData = {};
    Object.entries(message.reactions).forEach(([emoji, users]) => {
      reactionsData[emoji] = Array.from(users);
    });

    io.to(roomId).emit('messageReaction', {
      messageId,
      reactions: reactionsData,
    });
  });

  // Typing indicator
  socket.on('typing', ({ roomId, isTyping }) => {
    socket.to(roomId).emit('userTyping', {
      userId: user.id,
      username: user.username,
      isTyping,
    });
  });

  // === Direct Messages ===

  socket.on('directMessage', ({ toUserId, text }, callback) => {
    const targetSocketId = userSockets.get(toUserId);

    const message = {
      id: generateMessageId(),
      from: { id: user.id, username: user.username },
      to: toUserId,
      text,
      timestamp: new Date().toISOString(),
    };

    if (targetSocketId) {
      // User ออนไลน์ - ส่งทันที
      io.to(targetSocketId).emit('directMessage', message);
      callback({ success: true, delivered: true });
    } else {
      // User ออฟไลน์ - เก็บไว้ (implement offline storage)
      callback({ success: true, delivered: false, queued: true });
    }
  });

  // === Disconnect ===

  socket.on('disconnect', () => {
    userSockets.delete(user.id);
    console.log(`❌ ${user.username} disconnected`);

    // แจ้งทุกห้องที่ user อยู่
    rooms.forEach((room, roomId) => {
      if (room.members.has(user.id)) {
        room.members.delete(user.id);
        socket.to(roomId).emit('userLeft', {
          userId: user.id,
          username: user.username,
          roomId,
        });
      }
    });
  });
});

function generateRoomId() {
  return Math.random().toString(36).substr(2, 8).toUpperCase();
}

function generateMessageId() {
  return `msg_${Date.now()}_${Math.random().toString(36).substr(2, 5)}`;
}

httpServer.listen(3000, () => {
  console.log('Chat server running on port 3000');
});
```

---

## Authentication ใน WebSockets

### ขั้นตอนที่ 631: JWT Authentication

```javascript
// Middleware สำหรับ authentication
io.use(async (socket, next) => {
  try {
    const token = socket.handshake.auth.token
      || socket.handshake.headers.authorization?.replace('Bearer ', '');

    if (!token) {
      return next(new Error('AUTH_REQUIRED'));
    }

    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    const user = await User.findById(decoded.id);

    if (!user) {
      return next(new Error('USER_NOT_FOUND'));
    }

    socket.user = user;
    next();
  } catch (error) {
    if (error.name === 'JsonWebTokenError') {
      next(new Error('INVALID_TOKEN'));
    } else if (error.name === 'TokenExpiredError') {
      next(new Error('TOKEN_EXPIRED'));
    } else {
      next(new Error('AUTH_ERROR'));
    }
  }
});
```

### ขั้นตอนที่ 632: Token Refresh สำหรับ WebSockets

```javascript
// client.js - Handle token refresh
let socket;
let currentToken = localStorage.getItem('token');

function connectSocket() {
  socket = io('http://localhost:3000', {
    auth: { token: currentToken },
    reconnection: true,
    reconnectionAttempts: 5,
    reconnectionDelay: 1000,
  });

  socket.on('connect_error', async (error) => {
    if (error.message === 'TOKEN_EXPIRED') {
      // Refresh token
      try {
        const response = await fetch('/api/auth/refresh', {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify({ refreshToken: localStorage.getItem('refreshToken') }),
        });
        const data = await response.json();

        currentToken = data.token;
        localStorage.setItem('token', currentToken);

        // อัปเดต auth และเชื่อมต่อใหม่
        socket.auth = { token: currentToken };
        socket.connect();
      } catch (err) {
        // Refresh ล้มเหลว - logout
        window.location.href = '/login';
      }
    }
  });
}
```

---

## Scaling Socket.io

### ขั้นตอนที่ 633: Socket.io Redis Adapter

เมื่อต้องการ scale เป็นหลาย servers จำเป็นต้องใช้ adapter

```bash
npm install @socket.io/redis-adapter ioredis
```

```javascript
// server.js - Redis Adapter
const { createServer } = require('http');
const { Server } = require('socket.io');
const { createAdapter } = require('@socket.io/redis-adapter');
const { createClient } = require('redis');

async function main() {
  const httpServer = createServer();

  const io = new Server(httpServer, {
    cors: { origin: '*' },
  });

  // สร้าง Redis clients
  const pubClient = createClient({ url: process.env.REDIS_URL });
  const subClient = pubClient.duplicate();

  await Promise.all([pubClient.connect(), subClient.connect()]);

  // ใช้ Redis adapter
  io.adapter(createAdapter(pubClient, subClient));

  io.on('connection', (socket) => {
    socket.on('message', (msg) => {
      // broadcast ทำงานข้าม servers ได้
      socket.broadcast.emit('message', msg);
    });
  });

  httpServer.listen(3000);
  console.log('Server with Redis adapter ready');
}

main();
```

### ขั้นตอนที่ 634: Load Balancing

```nginx
# nginx.conf - Sticky sessions สำหรับ Socket.io
upstream socketio_backend {
    ip_hash;  # Sticky sessions ตาม IP
    server server1:3000;
    server server2:3000;
    server server3:3000;
}

server {
    listen 80;

    location / {
        proxy_pass http://socketio_backend;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
        proxy_read_timeout 3600;
        proxy_send_timeout 3600;
    }
}
```

---

## Monitoring และ Debugging

### ขั้นตอนที่ 635: Socket.io Admin UI

```bash
npm install @socket.io/admin-ui
```

```javascript
const { instrument } = require('@socket.io/admin-ui');

const io = new Server(httpServer, {
  cors: {
    origin: ['http://localhost:3000', 'https://admin.socket.io'],
    credentials: true,
  },
});

instrument(io, {
  auth: {
    type: 'basic',
    username: 'admin',
    password: process.env.ADMIN_PASSWORD,
  },
  mode: process.env.NODE_ENV === 'production' ? 'production' : 'development',
});
```

### ขั้นตอนที่ 636: Custom Metrics

```javascript
// Metrics tracking
const metrics = {
  connections: 0,
  messages: 0,
  rooms: 0,
};

io.on('connection', (socket) => {
  metrics.connections++;

  socket.on('message', () => {
    metrics.messages++;
  });

  socket.on('joinRoom', () => {
    metrics.rooms = io.sockets.adapter.rooms.size;
  });

  socket.on('disconnect', () => {
    metrics.connections--;
  });
});

// Expose metrics endpoint
app.get('/metrics', (req, res) => {
  res.json({
    ...metrics,
    connectedSockets: io.sockets.sockets.size,
    rooms: io.sockets.adapter.rooms.size,
    timestamp: new Date().toISOString(),
  });
});
```

---

## Error Handling และ Reconnection

### ขั้นตอนที่ 637: Robust Client Connection

```javascript
// client.js - Robust connection handling
const socket = io('http://localhost:3000', {
  auth: { token: getToken() },

  // Reconnection config
  reconnection: true,
  reconnectionAttempts: Infinity,
  reconnectionDelay: 1000,
  reconnectionDelayMax: 5000,
  randomizationFactor: 0.5,

  // Timeout
  timeout: 20000,
});

let isConnected = false;

socket.on('connect', () => {
  isConnected = true;
  console.log('✅ Connected');

  // Re-join rooms หลัง reconnect
  const savedRooms = JSON.parse(localStorage.getItem('rooms') || '[]');
  savedRooms.forEach(roomId => {
    socket.emit('joinRoom', { roomId });
  });
});

socket.on('disconnect', (reason) => {
  isConnected = false;
  console.log('❌ Disconnected:', reason);

  if (reason === 'io server disconnect') {
    // Server ตัดการเชื่อมต่อ - reconnect ด้วยตัวเอง
    socket.connect();
  }
  // ถ้า reason อื่นๆ socket.io จะ reconnect อัตโนมัติ
});

socket.on('connect_error', (error) => {
  console.error('Connection error:', error.message);
  if (error.message === 'AUTH_REQUIRED') {
    window.location.href = '/login';
  }
});

socket.io.on('reconnect', (attempt) => {
  console.log(`Reconnected after ${attempt} attempts`);
});

socket.io.on('reconnect_attempt', (attempt) => {
  console.log(`Reconnection attempt ${attempt}...`);
});
```

---

## การ Test Socket.io

### ขั้นตอนที่ 638: Unit Testing

```javascript
// __tests__/socket.test.js
const { createServer } = require('http');
const { Server } = require('socket.io');
const { io: ioc } = require('socket.io-client');

describe('Chat Server', () => {
  let server;
  let io;
  let client1;
  let client2;
  const PORT = 3001;

  beforeAll((done) => {
    const httpServer = createServer();
    io = new Server(httpServer);
    setupSocketHandlers(io);
    httpServer.listen(PORT, done);
    server = httpServer;
  });

  afterAll((done) => {
    io.close();
    server.close(done);
  });

  beforeEach((done) => {
    client1 = ioc(`http://localhost:${PORT}`, {
      auth: { token: 'valid-token-1' }
    });
    client2 = ioc(`http://localhost:${PORT}`, {
      auth: { token: 'valid-token-2' }
    });

    let connected = 0;
    const onConnect = () => {
      connected++;
      if (connected === 2) done();
    };

    client1.on('connect', onConnect);
    client2.on('connect', onConnect);
  });

  afterEach(() => {
    client1.disconnect();
    client2.disconnect();
  });

  test('should broadcast message to room', (done) => {
    const roomId = 'test-room';

    client1.emit('joinRoom', { roomId }, () => {
      client2.emit('joinRoom', { roomId }, () => {
        client2.on('newMessage', (msg) => {
          expect(msg.text).toBe('Hello!');
          done();
        });

        client1.emit('sendMessage', { roomId, text: 'Hello!' });
      });
    });
  });

  test('should not receive messages from other rooms', (done) => {
    client1.emit('joinRoom', { roomId: 'room1' });
    client2.emit('joinRoom', { roomId: 'room2' });

    let received = false;
    client2.on('newMessage', () => {
      received = true;
    });

    client1.emit('sendMessage', { roomId: 'room1', text: 'Private message' });

    setTimeout(() => {
      expect(received).toBe(false);
      done();
    }, 500);
  });
});
```

---

## Production Tips

### ขั้นตอนที่ 639: Security Best Practices

```javascript
// server.js - Production security
const io = new Server(httpServer, {
  // ตั้งค่า CORS อย่างเข้มงวด
  cors: {
    origin: process.env.ALLOWED_ORIGINS?.split(',') || [],
    methods: ['GET', 'POST'],
    credentials: true,
  },

  // จำกัดขนาด payload
  maxHttpBufferSize: 1e6, // 1MB

  // Path ที่กำหนดเอง
  path: '/socket.io/',
});

// Rate limiting ต่อ socket
io.use((socket, next) => {
  const clientIp = socket.handshake.address;
  const rateKey = `ratelimit:${clientIp}`;

  // ตรวจสอบ rate limit ด้วย Redis
  checkRateLimit(rateKey, 100, 60)  // 100 connections ต่อนาที
    .then(allowed => {
      if (!allowed) {
        return next(new Error('RATE_LIMIT_EXCEEDED'));
      }
      next();
    });
});

// ป้องกัน XSS ใน messages
function sanitizeMessage(text) {
  return text
    .replace(/&/g, '&amp;')
    .replace(/</g, '&lt;')
    .replace(/>/g, '&gt;')
    .replace(/"/g, '&quot;')
    .slice(0, 1000); // จำกัดความยาว
}
```

### ขั้นตอนที่ 640: Connection Pooling

```javascript
// Efficient room management
class RoomManager {
  constructor(io) {
    this.io = io;
    this.rooms = new Map();
  }

  createRoom(name, options = {}) {
    const id = generateId();
    this.rooms.set(id, {
      id,
      name,
      maxMembers: options.maxMembers || 100,
      members: new Set(),
      messages: [],
      createdAt: new Date(),
    });
    return id;
  }

  joinRoom(socketId, roomId) {
    const room = this.rooms.get(roomId);
    if (!room) throw new Error('ไม่พบห้อง');
    if (room.members.size >= room.maxMembers) throw new Error('ห้องเต็มแล้ว');

    room.members.add(socketId);
    return room;
  }

  leaveRoom(socketId, roomId) {
    const room = this.rooms.get(roomId);
    if (room) {
      room.members.delete(socketId);
      if (room.members.size === 0 && !room.persistent) {
        this.rooms.delete(roomId);
      }
    }
  }

  getRoomInfo(roomId) {
    const room = this.rooms.get(roomId);
    if (!room) return null;
    return {
      id: room.id,
      name: room.name,
      memberCount: room.members.size,
    };
  }
}
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Implement Typing Indicators

```javascript
// TODO: สร้าง typing indicator ที่มี debounce
// เมื่อ user พิมพ์ ให้แสดง "กำลังพิมพ์..."
// หลังหยุดพิมพ์ 2 วินาที ให้ซ่อน indicator
```

### แบบฝึกหัดที่ 2: Message Read Receipts

```javascript
// TODO: สร้าง read receipt system
// - เมื่อ user อ่าน message ให้ emit 'messageRead'
// - แสดง checkmarks (✓ = ส่งแล้ว, ✓✓ = อ่านแล้ว)
```

### แบบฝึกหัดที่ 3: File Sharing

```javascript
// TODO: เพิ่ม file sharing ใน chat
// - ใช้ ArrayBuffer สำหรับ binary data
// - แสดง progress ขณะส่ง
// - จำกัดขนาดไฟล์
```

### แบบฝึกหัดที่ 4: Video Call Signaling

```javascript
// TODO: สร้าง WebRTC signaling server
// ใช้ Socket.io สำหรับ exchange SDP offers/answers และ ICE candidates
socket.on('offer', ({ to, sdp }) => {
  socket.to(to).emit('offer', { from: socket.id, sdp });
});

socket.on('answer', ({ to, sdp }) => {
  socket.to(to).emit('answer', { from: socket.id, sdp });
});

socket.on('ice-candidate', ({ to, candidate }) => {
  socket.to(to).emit('ice-candidate', { from: socket.id, candidate });
});
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **WebSocket Protocol** - full-duplex communication และ handshake
2. **Socket.io** - library ที่ช่วยให้ใช้งาน WebSockets ง่ายขึ้น
3. **Namespaces** - แบ่ง socket server ออกเป็น sections
4. **Rooms** - grouping sockets และ broadcasting
5. **Real-time Chat** - สร้าง chat application สมบูรณ์
6. **Authentication** - JWT authentication สำหรับ WebSockets
7. **Scaling** - Redis adapter สำหรับ multi-server deployment
8. **Testing** - Unit testing Socket.io

ในบทถัดไปเราจะเรียนรู้ **Redis Caching** - การใช้ Redis สำหรับ caching, sessions, และ pub/sub
