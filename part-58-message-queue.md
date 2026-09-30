# Part 58: Message Queue (คิวข้อความ)
## ขั้นตอนที่ 58-58 จาก 1000

---

## บทนำ

Message Queue ช่วยให้ services สื่อสารกันแบบ asynchronous ทำให้ระบบมี decoupling สูง, ทนต่อ failure ได้ดี, และ scale ได้ง่าย

---

## 58.1 Message Queue Concepts

### ทำไมต้องใช้ Message Queue?

```
ปัญหาของ Direct Service Calls:
Service A → [HTTP] → Service B → [HTTP] → Service C

ถ้า Service B หรือ C ล่ม → Service A ได้รับ error
ถ้า load สูง → timeout
ถ้า Service C ช้า → block Service A
```

```
ด้วย Message Queue:
Service A → [Publish] → Queue → [Subscribe] → Service B
                               Queue → [Subscribe] → Service C

Service B หรือ C ล่ม → messages ค้างใน queue รอ
Load spike → messages ค้างรอ consumer ว่าง
Service A ไม่ต้องรอ → response ทันที
```

### คำศัพท์

| คำศัพท์ | ความหมาย |
|---------|---------|
| Producer | ผู้ส่ง message |
| Consumer | ผู้รับ message |
| Queue | คิวเก็บ messages |
| Exchange | Router สำหรับ distribute messages |
| Binding | การเชื่อม Queue กับ Exchange |
| Routing Key | Key สำหรับ route messages |

---

## 58.2 RabbitMQ

### ติดตั้ง RabbitMQ

```bash
# Docker
docker run -d \
  --name rabbitmq \
  -p 5672:5672 \
  -p 15672:15672 \
  -e RABBITMQ_DEFAULT_USER=admin \
  -e RABBITMQ_DEFAULT_PASS=password \
  rabbitmq:3.12-management

# เปิด Management UI: http://localhost:15672
```

### ติดตั้ง amqplib

```bash
npm install amqplib
```

---

## 58.3 amqplib - ใช้งาน RabbitMQ

### Connection Manager

```javascript
// src/config/rabbitmq.js
const amqp = require('amqplib');

let connection = null;
let channel = null;

const RABBITMQ_URL = process.env.RABBITMQ_URL || 'amqp://admin:password@localhost:5672';

/**
 * สร้าง connection และ channel
 */
const connect = async () => {
  try {
    connection = await amqp.connect(RABBITMQ_URL);
    channel = await connection.createChannel();
    
    console.log('✅ RabbitMQ connected');
    
    // Handle connection errors
    connection.on('error', (err) => {
      console.error('RabbitMQ connection error:', err.message);
      reconnect();
    });
    
    connection.on('close', () => {
      console.warn('RabbitMQ connection closed, reconnecting...');
      reconnect();
    });
    
    return channel;
  } catch (error) {
    console.error('Failed to connect to RabbitMQ:', error.message);
    await reconnect();
  }
};

/**
 * Reconnect ด้วย exponential backoff
 */
let reconnectAttempts = 0;
const reconnect = async () => {
  const delay = Math.min(1000 * Math.pow(2, reconnectAttempts), 30000);
  reconnectAttempts++;
  
  console.log(`Reconnecting in ${delay}ms... (attempt ${reconnectAttempts})`);
  
  await new Promise(resolve => setTimeout(resolve, delay));
  
  try {
    await connect();
    reconnectAttempts = 0;
  } catch {
    await reconnect();
  }
};

const getChannel = () => {
  if (!channel) throw new Error('RabbitMQ not connected');
  return channel;
};

const disconnect = async () => {
  if (channel) await channel.close();
  if (connection) await connection.close();
  console.log('RabbitMQ disconnected');
};

module.exports = { connect, getChannel, disconnect };
```

---

## 58.4 Work Queues

Work Queues ใช้สำหรับ distribute tasks ระหว่าง multiple workers

```javascript
// src/queues/workQueue.js
const { connect, getChannel } = require('../config/rabbitmq');

const QUEUE_NAME = 'task_queue';

/**
 * Producer - ส่ง task เข้า queue
 */
const sendTask = async (task) => {
  const channel = getChannel();
  
  // Durable queue - ค้างอยู่แม้ RabbitMQ restart
  await channel.assertQueue(QUEUE_NAME, { durable: true });
  
  const message = JSON.stringify({
    id: require('crypto').randomUUID(),
    ...task,
    timestamp: new Date().toISOString()
  });
  
  // Persistent message - ค้างอยู่แม้ server restart
  channel.sendToQueue(
    QUEUE_NAME,
    Buffer.from(message),
    { persistent: true }
  );
  
  console.log(`📤 Task sent: ${message}`);
  return JSON.parse(message).id;
};

/**
 * Consumer - รับ task และประมวลผล
 */
const consumeTasks = async (processor) => {
  const channel = getChannel();
  
  await channel.assertQueue(QUEUE_NAME, { durable: true });
  
  // Fair dispatch - รับ message ทีละ 1 จนกว่าจะ ack
  channel.prefetch(1);
  
  console.log(`🎧 Listening on queue: ${QUEUE_NAME}`);
  
  channel.consume(QUEUE_NAME, async (msg) => {
    if (!msg) return;
    
    const task = JSON.parse(msg.content.toString());
    console.log(`📨 Received task: ${task.id}`);
    
    try {
      await processor(task);
      
      // Acknowledge สำเร็จ
      channel.ack(msg);
      console.log(`✅ Task ${task.id} completed`);
    } catch (error) {
      console.error(`❌ Task ${task.id} failed:`, error.message);
      
      // Reject และ requeue (สูงสุด 3 ครั้ง)
      const retryCount = (msg.properties.headers?.['x-retry-count'] || 0);
      
      if (retryCount < 3) {
        channel.nack(msg, false, false);  // Dead letter
      } else {
        channel.nack(msg, false, true);   // Requeue
      }
    }
  });
};

module.exports = { sendTask, consumeTasks };
```

---

## 58.5 Topic Exchange (Topics)

Topic Exchange route messages ตาม routing key pattern

```javascript
// src/exchanges/topicExchange.js
const { getChannel } = require('../config/rabbitmq');

const EXCHANGE_NAME = 'app_events';

/**
 * สร้าง Topic Exchange
 */
const setupTopicExchange = async () => {
  const channel = getChannel();
  
  await channel.assertExchange(EXCHANGE_NAME, 'topic', { durable: true });
  console.log(`✅ Topic exchange '${EXCHANGE_NAME}' created`);
};

/**
 * Publish event ด้วย routing key
 * Pattern: <domain>.<action>.<entity>
 * เช่น: user.created, order.payment.success, product.stock.low
 */
const publishEvent = async (routingKey, data) => {
  const channel = getChannel();
  
  const message = {
    id: require('crypto').randomUUID(),
    event: routingKey,
    data,
    timestamp: new Date().toISOString(),
    source: process.env.SERVICE_NAME || 'unknown'
  };
  
  channel.publish(
    EXCHANGE_NAME,
    routingKey,
    Buffer.from(JSON.stringify(message)),
    { persistent: true }
  );
  
  console.log(`📤 Event published: ${routingKey}`);
};

/**
 * Subscribe ด้วย pattern
 * * = ตรงหนึ่ง word
 * # = ตรง 0 หรือมากกว่า words
 * 
 * ตัวอย่าง patterns:
 * user.*    = user.created, user.updated, user.deleted
 * *.created = user.created, order.created, product.created
 * order.#   = order.created, order.payment.success, order.item.added
 * #         = ทุก events
 */
const subscribeToPattern = async (pattern, handler, queueName) => {
  const channel = getChannel();
  
  // สร้าง queue ถ้ายังไม่มี
  const { queue } = await channel.assertQueue(
    queueName || `${process.env.SERVICE_NAME}.${pattern}`,
    { durable: true }
  );
  
  // Bind queue กับ exchange ด้วย pattern
  await channel.bindQueue(queue, EXCHANGE_NAME, pattern);
  
  channel.prefetch(5);
  
  console.log(`🎧 Subscribed to: ${pattern}`);
  
  channel.consume(queue, async (msg) => {
    if (!msg) return;
    
    const event = JSON.parse(msg.content.toString());
    
    try {
      await handler(event);
      channel.ack(msg);
    } catch (error) {
      console.error(`Error handling event ${event.event}:`, error.message);
      channel.nack(msg, false, true);
    }
  });
};

module.exports = { setupTopicExchange, publishEvent, subscribeToPattern };
```

### ตัวอย่างใช้งาน Topic Exchange

```javascript
// notification-service/src/index.js
const { connect } = require('./config/rabbitmq');
const { subscribeToPattern } = require('./exchanges/topicExchange');

const startNotificationService = async () => {
  await connect();
  
  // Subscribe ทุก user events
  await subscribeToPattern('user.*', handleUserEvent, 'notification.user');
  
  // Subscribe ทุก order events
  await subscribeToPattern('order.#', handleOrderEvent, 'notification.order');
  
  // Subscribe payment success เท่านั้น
  await subscribeToPattern('order.payment.success', handlePaymentSuccess, 'notification.payment');
  
  console.log('🔔 Notification service started');
};

const handleUserEvent = async (event) => {
  console.log(`User event: ${event.event}`);
  
  switch (event.event) {
    case 'user.created':
      await sendWelcomeEmail(event.data);
      break;
    case 'user.deleted':
      await cleanupUserData(event.data);
      break;
  }
};

const handleOrderEvent = async (event) => {
  switch (event.event) {
    case 'order.created':
      await notifyWarehouse(event.data);
      break;
    case 'order.cancelled':
      await sendCancellationEmail(event.data);
      break;
    case 'order.payment.success':
      await sendReceiptEmail(event.data);
      break;
    case 'order.shipped':
      await sendShippingNotification(event.data);
      break;
  }
};

startNotificationService();
```

---

## 58.6 Fanout Exchange

```javascript
// src/exchanges/fanoutExchange.js
const { getChannel } = require('../config/rabbitmq');

const EXCHANGE_NAME = 'broadcast';

const setupFanoutExchange = async () => {
  const channel = getChannel();
  await channel.assertExchange(EXCHANGE_NAME, 'fanout', { durable: true });
};

/**
 * Broadcast ไปทุก subscribers
 */
const broadcast = async (data) => {
  const channel = getChannel();
  
  channel.publish(
    EXCHANGE_NAME,
    '',  // Routing key ไม่สำคัญสำหรับ fanout
    Buffer.from(JSON.stringify(data)),
    { persistent: true }
  );
};

/**
 * Subscribe to broadcasts
 */
const subscribeBroadcast = async (handler, queueName) => {
  const channel = getChannel();
  
  // สร้าง exclusive queue (ลบเมื่อ disconnect)
  const { queue } = await channel.assertQueue(
    queueName || '',
    { exclusive: !queueName, durable: !!queueName }
  );
  
  await channel.bindQueue(queue, EXCHANGE_NAME, '');
  
  channel.consume(queue, async (msg) => {
    if (!msg) return;
    const data = JSON.parse(msg.content.toString());
    await handler(data);
    channel.ack(msg);
  });
};

module.exports = { setupFanoutExchange, broadcast, subscribeBroadcast };
```

---

## 58.7 Dead Letter Queues

Dead Letter Queue (DLQ) รับ messages ที่ถูก reject หรือ expire

```javascript
// src/queues/dlqSetup.js
const { getChannel } = require('../config/rabbitmq');

/**
 * ตั้งค่า Queue พร้อม Dead Letter Queue
 */
const setupQueueWithDLQ = async (queueName, options = {}) => {
  const channel = getChannel();
  const dlqName = `${queueName}.dlq`;
  const dlxName = `${queueName}.dlx`;
  
  // สร้าง Dead Letter Exchange
  await channel.assertExchange(dlxName, 'direct', { durable: true });
  
  // สร้าง Dead Letter Queue
  await channel.assertQueue(dlqName, { durable: true });
  
  // Bind DLQ กับ DLX
  await channel.bindQueue(dlqName, dlxName, queueName);
  
  // สร้าง main queue ที่ route failures ไปยัง DLX
  await channel.assertQueue(queueName, {
    durable: true,
    arguments: {
      'x-dead-letter-exchange': dlxName,
      'x-dead-letter-routing-key': queueName,
      'x-message-ttl': options.messageTTL || 24 * 60 * 60 * 1000,  // 24 ชั่วโมง
      'x-max-retries': options.maxRetries || 3
    }
  });
  
  console.log(`✅ Queue setup: ${queueName} (DLQ: ${dlqName})`);
  
  return { queueName, dlqName };
};

/**
 * Process DLQ - retry หรือ log failures
 */
const processDLQ = async (dlqName, handler) => {
  const channel = getChannel();
  
  channel.consume(dlqName, async (msg) => {
    if (!msg) return;
    
    const message = JSON.parse(msg.content.toString());
    const deathCount = msg.properties.headers?.['x-death']?.[0]?.count || 0;
    
    console.log(`💀 DLQ message: ${message.id}, deaths: ${deathCount}`);
    
    // Log ลง database
    await logFailedMessage({
      queue: dlqName,
      message,
      deathCount,
      reason: msg.properties.headers?.['x-death']?.[0]?.reason,
      failedAt: new Date()
    });
    
    // อาจจะ notify admin
    if (deathCount >= 5) {
      await notifyAdmin({ dlqName, message, deathCount });
    }
    
    // Acknowledge เพื่อลบออกจาก DLQ (หรือ reject เพื่อเก็บไว้)
    channel.ack(msg);
  });
};

module.exports = { setupQueueWithDLQ, processDLQ };
```

---

## แบบฝึกหัดที่ 58

### แบบฝึกหัดพื้นฐาน

**1. Work Queue**

สร้าง email processing system:
- Producer: API endpoint ส่ง email tasks
- 3 Workers: process emails พร้อมกัน
- Fair dispatch ด้วย prefetch

**2. Topic Exchange**

สร้าง e-commerce event system:
- Order service publish events
- Inventory service subscribe order events
- Notification service subscribe ทุก events

### แบบฝึกหัดขั้นสูง

**3. Dead Letter Queue**

เพิ่ม DLQ:
- ลอง retry 3 ครั้ง
- ถ้าล้มเหลวทุกครั้ง → DLQ
- Monitor DLQ ด้วย dashboard

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **RabbitMQ** - installation, management UI
2. **amqplib** - Node.js client
3. **Work Queues** - distributed task processing
4. **Topic Exchange** - pattern-based routing
5. **Fanout Exchange** - broadcast to all
6. **Dead Letter Queue** - handle failed messages

**ถัดไป:** Part 59 - GraphQL
