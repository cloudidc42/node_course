# Part 61: Server-Sent Events (SSE)
## ขั้นตอนที่ 601-610 จาก 1000

---

## SSE คืออะไร?

Server-Sent Events (SSE) เป็นเทคโนโลยีที่ช่วยให้ Server สามารถส่งข้อมูลไปยัง Client แบบ real-time ผ่าน HTTP connection ที่เปิดค้างไว้ ต่างจาก WebSocket ที่เป็น bidirectional communication, SSE เป็น unidirectional (server → client เท่านั้น)

---

## 1. SSE vs WebSocket

### เปรียบเทียบ

| Feature | SSE | WebSocket |
|---------|-----|-----------|
| Direction | Server → Client | Bidirectional |
| Protocol | HTTP/HTTPS | WS/WSS |
| Browser Support | ดีมาก (ทุก modern browser) | ดีมาก |
| Reconnection | Built-in auto reconnect | ต้องเขียนเอง |
| Firewall | ผ่านได้ง่าย (ใช้ HTTP) | อาจถูก block |
| Complexity | ง่าย | ซับซ้อนกว่า |
| Use Case | Notifications, feeds, logs | Chat, gaming, collaborative |

### เมื่อไหร่ควรใช้ SSE

```
ใช้ SSE เมื่อ:
- ต้องการ real-time updates จาก server
- Client ไม่ต้องส่งข้อมูลบ่อย
- ต้องการ reconnection อัตโนมัติ
- ใช้งานผ่าน HTTP/2

ใช้ WebSocket เมื่อ:
- ต้องการสื่อสารสองทิศทาง
- ต้องการ latency ต่ำมาก
- สร้าง game หรือ collaborative editor
```

### ตัวอย่าง SSE ง่ายๆ

```javascript
// server.js
const express = require('express');
const app = express();

app.get('/events', (req, res) => {
  // ตั้งค่า headers สำหรับ SSE
  res.setHeader('Content-Type', 'text/event-stream');
  res.setHeader('Cache-Control', 'no-cache');
  res.setHeader('Connection', 'keep-alive');
  res.setHeader('Access-Control-Allow-Origin', '*');

  // ส่ง event ทุก 1 วินาที
  const intervalId = setInterval(() => {
    const data = JSON.stringify({
      time: new Date().toISOString(),
      message: 'Hello from server!'
    });
    res.write(`data: ${data}\n\n`);
  }, 1000);

  // เมื่อ client disconnect
  req.on('close', () => {
    clearInterval(intervalId);
    console.log('Client disconnected');
  });
});

app.listen(3000);
```

```html
<!-- client.html -->
<!DOCTYPE html>
<html>
<body>
  <div id="events"></div>
  
  <script>
    const eventsDiv = document.getElementById('events');
    const eventSource = new EventSource('/events');
    
    eventSource.onmessage = (event) => {
      const data = JSON.parse(event.data);
      const p = document.createElement('p');
      p.textContent = `${data.time}: ${data.message}`;
      eventsDiv.appendChild(p);
    };
    
    eventSource.onerror = (error) => {
      console.error('SSE Error:', error);
    };
  </script>
</body>
</html>
```

---

## 2. EventSource API

### EventSource Methods และ Properties

```javascript
// สร้าง EventSource
const eventSource = new EventSource('/events');

// สร้างด้วย credentials
const eventSourceWithAuth = new EventSource('/events', {
  withCredentials: true
});

// Properties
console.log(eventSource.url);        // URL ที่เชื่อมต่อ
console.log(eventSource.readyState); // 0=CONNECTING, 1=OPEN, 2=CLOSED
console.log(eventSource.withCredentials); // boolean

// Event Handlers
eventSource.onopen = (event) => {
  console.log('Connection opened');
};

eventSource.onmessage = (event) => {
  console.log('Data:', event.data);
  console.log('Last Event ID:', event.lastEventId);
  console.log('Origin:', event.origin);
};

eventSource.onerror = (event) => {
  if (event.readyState === EventSource.CLOSED) {
    console.log('Connection closed');
  } else {
    console.error('Error occurred');
  }
};

// Custom event types
eventSource.addEventListener('notification', (event) => {
  console.log('Notification:', event.data);
});

eventSource.addEventListener('update', (event) => {
  console.log('Update:', event.data);
});

// ปิด connection
eventSource.close();
```

### SSE Message Format

```
# รูปแบบ SSE message (text format)

# ส่ง data ธรรมดา
data: Hello World\n\n

# ส่ง JSON
data: {"key": "value"}\n\n

# ส่งพร้อม event type
event: notification\n
data: {"message": "New notification"}\n\n

# ส่งพร้อม ID
id: 12345\n
data: {"message": "With ID"}\n\n

# ส่งพร้อม retry interval (ms)
retry: 3000\n
data: {"message": "Retry after 3 seconds"}\n\n

# Comment (ใช้เป็น heartbeat)
: ping\n\n

# Multi-line data
data: line 1\n
data: line 2\n
data: line 3\n\n
```

---

## 3. Express SSE Implementation

### SSE Helper Class

```javascript
// sse-helper.js
class SSEHelper {
  constructor(req, res) {
    this.req = req;
    this.res = res;
    this.lastEventId = req.headers['last-event-id'] || 0;
    
    this.setupHeaders();
    this.setupHeartbeat();
  }

  setupHeaders() {
    this.res.setHeader('Content-Type', 'text/event-stream');
    this.res.setHeader('Cache-Control', 'no-cache');
    this.res.setHeader('Connection', 'keep-alive');
    this.res.setHeader('X-Accel-Buffering', 'no'); // สำหรับ Nginx
    
    // ส่ง HTTP 200 OK ทันที
    this.res.writeHead(200);
  }

  setupHeartbeat() {
    // ส่ง heartbeat ทุก 30 วินาทีเพื่อ keep connection alive
    this.heartbeatInterval = setInterval(() => {
      this.sendComment('heartbeat');
    }, 30000);

    this.req.on('close', () => {
      clearInterval(this.heartbeatInterval);
    });
  }

  send(data, options = {}) {
    const { event, id, retry } = options;
    let message = '';

    if (retry) {
      message += `retry: ${retry}\n`;
    }

    if (id !== undefined) {
      message += `id: ${id}\n`;
    }

    if (event) {
      message += `event: ${event}\n`;
    }

    const dataStr = typeof data === 'object' ? JSON.stringify(data) : data;
    message += `data: ${dataStr}\n\n`;

    this.res.write(message);
  }

  sendComment(comment) {
    this.res.write(`: ${comment}\n\n`);
  }

  close() {
    clearInterval(this.heartbeatInterval);
    this.res.end();
  }
}

module.exports = SSEHelper;
```

### SSE Manager (จัดการ multiple clients)

```javascript
// sse-manager.js
class SSEManager {
  constructor() {
    this.clients = new Map();
    this.channels = new Map();
  }

  addClient(clientId, sse) {
    this.clients.set(clientId, sse);
    console.log(`Client ${clientId} connected. Total: ${this.clients.size}`);
  }

  removeClient(clientId) {
    this.clients.delete(clientId);
    // ลบออกจาก channels ด้วย
    this.channels.forEach((clients, channel) => {
      clients.delete(clientId);
      if (clients.size === 0) {
        this.channels.delete(channel);
      }
    });
    console.log(`Client ${clientId} disconnected. Total: ${this.clients.size}`);
  }

  subscribe(clientId, channel) {
    if (!this.channels.has(channel)) {
      this.channels.set(channel, new Set());
    }
    this.channels.get(channel).add(clientId);
  }

  unsubscribe(clientId, channel) {
    if (this.channels.has(channel)) {
      this.channels.get(channel).delete(clientId);
    }
  }

  // ส่งให้ client เดียว
  sendToClient(clientId, data, options = {}) {
    const sse = this.clients.get(clientId);
    if (sse) {
      sse.send(data, options);
      return true;
    }
    return false;
  }

  // ส่งให้ทุก client
  broadcast(data, options = {}) {
    let sent = 0;
    this.clients.forEach((sse, clientId) => {
      try {
        sse.send(data, options);
        sent++;
      } catch (err) {
        console.error(`Error sending to client ${clientId}:`, err);
        this.removeClient(clientId);
      }
    });
    return sent;
  }

  // ส่งให้ channel
  sendToChannel(channel, data, options = {}) {
    const channelClients = this.channels.get(channel);
    if (!channelClients) return 0;

    let sent = 0;
    channelClients.forEach(clientId => {
      if (this.sendToClient(clientId, data, options)) {
        sent++;
      }
    });
    return sent;
  }

  getClientCount() {
    return this.clients.size;
  }

  getChannelInfo() {
    const info = {};
    this.channels.forEach((clients, channel) => {
      info[channel] = clients.size;
    });
    return info;
  }
}

module.exports = new SSEManager(); // Singleton
```

### Express Routes สำหรับ SSE

```javascript
// routes/notifications.js
const express = require('express');
const router = express.Router();
const SSEHelper = require('../sse-helper');
const sseManager = require('../sse-manager');
const { v4: uuidv4 } = require('uuid');
const authenticate = require('../middleware/authenticate');

// SSE endpoint หลัก
router.get('/subscribe', authenticate, (req, res) => {
  const clientId = uuidv4();
  const userId = req.user.id;
  
  const sse = new SSEHelper(req, res);
  sseManager.addClient(clientId, sse);
  
  // Subscribe ไปยัง user's personal channel
  sseManager.subscribe(clientId, `user:${userId}`);
  
  // Subscribe ไปยัง global channel
  sseManager.subscribe(clientId, 'global');
  
  // ส่ง initial message
  sse.send({
    type: 'connected',
    clientId,
    message: 'Connected to notifications'
  });

  // Cleanup เมื่อ disconnect
  req.on('close', () => {
    sseManager.removeClient(clientId);
  });
});

// Endpoint สำหรับส่ง notification (เรียกจาก API)
router.post('/send', authenticate, async (req, res) => {
  const { userId, message, type } = req.body;
  
  const notification = {
    id: uuidv4(),
    type,
    message,
    timestamp: new Date().toISOString()
  };

  // ส่งให้ user specific
  const sent = sseManager.sendToChannel(`user:${userId}`, notification, {
    event: 'notification'
  });

  res.json({
    success: true,
    sent,
    notification
  });
});

// Broadcast ไปทุกคน
router.post('/broadcast', authenticate, async (req, res) => {
  const { message, type } = req.body;
  
  const announcement = {
    id: uuidv4(),
    type,
    message,
    timestamp: new Date().toISOString()
  };

  const sent = sseManager.broadcast(announcement, {
    event: 'announcement'
  });

  res.json({ success: true, sent });
});

// Status endpoint
router.get('/status', (req, res) => {
  res.json({
    connectedClients: sseManager.getClientCount(),
    channels: sseManager.getChannelInfo()
  });
});

module.exports = router;
```

---

## 4. Reconnection Handling

### Auto Reconnection ฝั่ง Server

```javascript
// ส่ง retry interval ให้ client
app.get('/events', (req, res) => {
  const sse = new SSEHelper(req, res);
  
  // บอก client ให้ reconnect หลัง 5 วินาที ถ้า connection หาย
  sse.send({ message: 'Connected' }, { retry: 5000 });
  
  // ตรวจสอบ Last-Event-ID สำหรับ replay missed events
  const lastEventId = parseInt(req.headers['last-event-id'] || '0');
  console.log(`Client reconnected from event ID: ${lastEventId}`);
});
```

### Event Store สำหรับ Missed Events

```javascript
// event-store.js
class EventStore {
  constructor(maxSize = 1000) {
    this.events = [];
    this.maxSize = maxSize;
    this.counter = 0;
  }

  store(data, options = {}) {
    this.counter++;
    const event = {
      id: this.counter,
      data,
      options,
      timestamp: Date.now()
    };
    
    this.events.push(event);
    
    // จำกัดขนาด
    if (this.events.length > this.maxSize) {
      this.events.shift();
    }
    
    return event;
  }

  getEventsSince(lastEventId) {
    const id = parseInt(lastEventId) || 0;
    return this.events.filter(e => e.id > id);
  }

  getLatestId() {
    return this.counter;
  }
}

module.exports = new EventStore();
```

```javascript
// ใช้งาน Event Store กับ SSE
const eventStore = require('./event-store');

app.get('/events', (req, res) => {
  const sse = new SSEHelper(req, res);
  const lastEventId = req.headers['last-event-id'];

  // Replay missed events
  if (lastEventId) {
    const missedEvents = eventStore.getEventsSince(lastEventId);
    missedEvents.forEach(event => {
      sse.send(event.data, { ...event.options, id: event.id });
    });
  }

  // Store เพื่อ replay ในอนาคต
  const sendEvent = (data, options = {}) => {
    const stored = eventStore.store(data, options);
    sse.send(data, { ...options, id: stored.id });
  };

  // ส่ง events
  const interval = setInterval(() => {
    sendEvent({ time: new Date().toISOString(), type: 'heartbeat' });
  }, 5000);

  req.on('close', () => {
    clearInterval(interval);
  });
});
```

### Client-side Reconnection Logic

```javascript
// sse-client.js
class SSEClient {
  constructor(url, options = {}) {
    this.url = url;
    this.options = options;
    this.eventSource = null;
    this.reconnectDelay = options.reconnectDelay || 3000;
    this.maxReconnectAttempts = options.maxReconnectAttempts || 10;
    this.reconnectAttempts = 0;
    this.handlers = {};
    this.lastEventId = null;
    
    this.connect();
  }

  connect() {
    const url = this.lastEventId 
      ? `${this.url}?lastEventId=${this.lastEventId}`
      : this.url;
    
    this.eventSource = new EventSource(url, this.options);
    
    this.eventSource.onopen = () => {
      console.log('SSE Connected');
      this.reconnectAttempts = 0;
      this.emit('open');
    };

    this.eventSource.onmessage = (event) => {
      this.lastEventId = event.lastEventId;
      this.emit('message', event.data);
    };

    this.eventSource.onerror = (event) => {
      console.error('SSE Error, attempting reconnect...');
      this.eventSource.close();
      
      if (this.reconnectAttempts < this.maxReconnectAttempts) {
        this.reconnectAttempts++;
        const delay = this.reconnectDelay * Math.pow(2, this.reconnectAttempts - 1);
        console.log(`Reconnecting in ${delay}ms (attempt ${this.reconnectAttempts})`);
        
        setTimeout(() => this.connect(), delay);
      } else {
        console.error('Max reconnection attempts reached');
        this.emit('maxReconnect');
      }
    };

    // Register custom event listeners
    Object.keys(this.handlers).forEach(eventType => {
      if (eventType !== 'open' && eventType !== 'message' && eventType !== 'error') {
        this.handlers[eventType].forEach(handler => {
          this.eventSource.addEventListener(eventType, handler);
        });
      }
    });
  }

  on(eventType, handler) {
    if (!this.handlers[eventType]) {
      this.handlers[eventType] = [];
    }
    this.handlers[eventType].push(handler);
    
    if (this.eventSource && eventType !== 'open' && eventType !== 'message') {
      this.eventSource.addEventListener(eventType, handler);
    }
  }

  emit(eventType, ...args) {
    const handlers = this.handlers[eventType] || [];
    handlers.forEach(handler => handler(...args));
  }

  close() {
    if (this.eventSource) {
      this.eventSource.close();
    }
  }
}

// ใช้งาน
const client = new SSEClient('/events', {
  withCredentials: true,
  reconnectDelay: 2000,
  maxReconnectAttempts: 5
});

client.on('message', (data) => {
  console.log('Received:', data);
});

client.on('notification', (event) => {
  const data = JSON.parse(event.data);
  console.log('Notification:', data);
});
```

---

## 5. Load Balancing SSE

### ปัญหาของ SSE กับ Load Balancer

```
ปัญหาหลัก: SSE connection เป็น long-lived connection
เมื่อมี load balancer, client อาจเชื่อมต่อกับ server instance ต่างๆ
ถ้า event ถูก publish ที่ instance A แต่ client อยู่ที่ instance B จะไม่ได้รับ event
```

### Solution 1: Sticky Sessions

```nginx
# nginx.conf
upstream backend {
    ip_hash;  # Sticky sessions by IP
    server 127.0.0.1:3001;
    server 127.0.0.1:3002;
    server 127.0.0.1:3003;
}

server {
    listen 80;
    
    location /events {
        proxy_pass http://backend;
        proxy_http_version 1.1;
        proxy_set_header Connection '';
        proxy_set_header Cache-Control 'no-cache';
        proxy_set_header X-Accel-Buffering 'no';
        proxy_buffering off;
        proxy_read_timeout 86400s;
        proxy_send_timeout 86400s;
        
        # ปิด gzip สำหรับ SSE
        gzip off;
    }
}
```

### Solution 2: Redis Pub/Sub

```javascript
// redis-sse.js
const Redis = require('ioredis');
const sseManager = require('./sse-manager');

const publisher = new Redis(process.env.REDIS_URL);
const subscriber = new Redis(process.env.REDIS_URL);

// Subscribe ไปยัง Redis channel
subscriber.subscribe('notifications', (err, count) => {
  if (err) {
    console.error('Failed to subscribe:', err.message);
    return;
  }
  console.log(`Subscribed to ${count} channel(s)`);
});

// เมื่อได้รับ message จาก Redis ส่งต่อให้ SSE clients
subscriber.on('message', (channel, message) => {
  const event = JSON.parse(message);
  
  if (event.targetUserId) {
    sseManager.sendToChannel(`user:${event.targetUserId}`, event.data, {
      event: event.type
    });
  } else {
    sseManager.broadcast(event.data, { event: event.type });
  }
});

// Function สำหรับ publish event
async function publishEvent(type, data, targetUserId = null) {
  const message = JSON.stringify({
    type,
    data,
    targetUserId,
    timestamp: Date.now()
  });
  
  await publisher.publish('notifications', message);
}

module.exports = { publishEvent };
```

```javascript
// app.js
const { publishEvent } = require('./redis-sse');

// ส่ง notification จาก API endpoint
app.post('/api/notify', async (req, res) => {
  const { userId, message } = req.body;
  
  await publishEvent('notification', { message }, userId);
  
  res.json({ success: true });
});
```

### Solution 3: Message Queue

```javascript
// bull-sse.js
const Bull = require('bull');
const sseManager = require('./sse-manager');

const notificationQueue = new Bull('notifications', {
  redis: process.env.REDIS_URL
});

// Worker: รับ job และส่งต่อให้ SSE clients
notificationQueue.process(async (job) => {
  const { type, data, targetUserId } = job.data;
  
  let sent = 0;
  if (targetUserId) {
    sent = sseManager.sendToChannel(`user:${targetUserId}`, data, { event: type });
  } else {
    sent = sseManager.broadcast(data, { event: type });
  }
  
  return { sent };
});

// Producer: เพิ่ม job เข้า queue
async function queueNotification(type, data, targetUserId = null) {
  return notificationQueue.add({
    type,
    data,
    targetUserId
  }, {
    attempts: 3,
    backoff: { type: 'exponential', delay: 2000 },
    removeOnComplete: 100,
    removeOnFail: 50
  });
}

module.exports = { queueNotification };
```

---

## 6. Real-World SSE Examples

### Live Dashboard

```javascript
// dashboard-sse.js
const express = require('express');
const router = express.Router();
const SSEHelper = require('../sse-helper');
const os = require('os');

router.get('/dashboard/live', (req, res) => {
  const sse = new SSEHelper(req, res);
  
  // ส่ง system metrics ทุก 2 วินาที
  const metricsInterval = setInterval(() => {
    const metrics = {
      cpu: process.cpuUsage(),
      memory: {
        total: os.totalmem(),
        free: os.freemem(),
        used: os.totalmem() - os.freemem()
      },
      uptime: process.uptime(),
      timestamp: new Date().toISOString()
    };
    
    sse.send(metrics, { event: 'metrics' });
  }, 2000);

  // ส่ง active users ทุก 5 วินาที
  const usersInterval = setInterval(async () => {
    const activeUsers = await getActiveUserCount(); // query จาก DB
    sse.send({ count: activeUsers }, { event: 'activeUsers' });
  }, 5000);

  req.on('close', () => {
    clearInterval(metricsInterval);
    clearInterval(usersInterval);
  });
});
```

### Progress Tracking

```javascript
// progress-sse.js
router.post('/jobs/start', async (req, res) => {
  const jobId = uuidv4();
  
  // เริ่ม background job
  processJobInBackground(jobId);
  
  res.json({ jobId });
});

router.get('/jobs/:jobId/progress', (req, res) => {
  const { jobId } = req.params;
  const sse = new SSEHelper(req, res);
  
  // Listen ไปยัง job events
  jobEmitter.on(`job:${jobId}`, (event) => {
    sse.send(event, { event: event.type });
    
    if (event.type === 'completed' || event.type === 'failed') {
      sse.close();
      jobEmitter.removeAllListeners(`job:${jobId}`);
    }
  });

  req.on('close', () => {
    jobEmitter.removeAllListeners(`job:${jobId}`);
  });
});

async function processJobInBackground(jobId) {
  try {
    jobEmitter.emit(`job:${jobId}`, {
      type: 'started',
      progress: 0,
      message: 'Job started'
    });

    // ทำงาน...
    for (let i = 1; i <= 10; i++) {
      await sleep(1000);
      jobEmitter.emit(`job:${jobId}`, {
        type: 'progress',
        progress: i * 10,
        message: `Processing step ${i}/10`
      });
    }

    jobEmitter.emit(`job:${jobId}`, {
      type: 'completed',
      progress: 100,
      message: 'Job completed successfully'
    });
  } catch (error) {
    jobEmitter.emit(`job:${jobId}`, {
      type: 'failed',
      error: error.message
    });
  }
}
```

---

## แบบฝึกหัด

### ระดับ 1: พื้นฐาน
สร้าง SSE server ที่:
- ส่ง timestamp ทุก 2 วินาที
- รองรับ multiple clients
- แสดงจำนวน connected clients

### ระดับ 2: กลาง
สร้างระบบ notification ที่:
- ใช้ SSE สำหรับ real-time delivery
- จัดการ channels (global, user-specific)
- มี heartbeat ป้องกัน timeout
- Handle reconnection ด้วย lastEventId

### ระดับ 3: ขั้นสูง
สร้าง scalable SSE system ที่:
- ใช้ Redis Pub/Sub เพื่อ sync ระหว่าง server instances
- มี event store สำหรับ replay missed events
- รองรับ load balancing
- มี monitoring dashboard

```javascript
// โค้ด starter สำหรับแบบฝึกหัด
const express = require('express');
const Redis = require('ioredis');
const { v4: uuidv4 } = require('uuid');
const app = express();

// TODO: สร้าง SSEHelper class
// TODO: สร้าง SSEManager class
// TODO: เพิ่ม Redis Pub/Sub
// TODO: สร้าง event store
// TODO: สร้าง routes

app.listen(3000, () => {
  console.log('SSE Server running on port 3000');
});
```

---

## สรุป

SSE เป็นเครื่องมือที่เหมาะสำหรับ real-time communication จาก server ไป client เมื่อไม่ต้องการ bidirectional communication เต็มรูปแบบ ใช้ง่ายกว่า WebSocket และมี auto-reconnection built-in แต่ต้องจัดการเรื่อง load balancing อย่างระมัดระวัง โดยใช้ Redis Pub/Sub หรือ Message Queue เพื่อ scale ได้

> ขั้นตอนต่อไป: Part 62 - OAuth2 Authentication
