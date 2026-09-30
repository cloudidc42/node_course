# Part 79: Apache Kafka
## ขั้นตอนที่ 781-790 จาก 1000

---

## Apache Kafka คืออะไร?

Apache Kafka เป็น distributed event streaming platform ที่ออกแบบมาสำหรับ high-throughput, fault-tolerant messaging ใช้สำหรับ real-time data pipelines และ event-driven architectures

---

## 1. Kafka Concepts

### Architecture

```
Producer → Topic (Partitions) → Consumer Groups

Topic: หมวดหมู่ของ messages
Partition: การแบ่ง topic ย่อย (เพิ่ม parallelism)
Offset: ตำแหน่งของ message ใน partition
Consumer Group: กลุ่ม consumers ที่ consume topic เดียวกัน
Broker: Kafka server
```

### เปรียบเทียบกับ RabbitMQ

```
Kafka:
- Retention: เก็บ messages ตาม retention period (default 7 days)
- Replay: replay messages ย้อนหลังได้
- High throughput: ล้าน messages/วินาที
- เหมาะกับ: event sourcing, log aggregation, stream processing

RabbitMQ:
- Message ถูกลบหลัง consume
- Routing ยืดหยุ่นกว่า (exchanges, bindings)
- เหมาะกับ: task queues, RPC, complex routing
```

---

## 2. Setup KafkaJS

```bash
npm install kafkajs
```

### Docker Compose สำหรับ Kafka

```yaml
# docker-compose.yml
version: '3.8'

services:
  zookeeper:
    image: confluentinc/cp-zookeeper:latest
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_TICK_TIME: 2000

  kafka:
    image: confluentinc/cp-kafka:latest
    depends_on:
      - zookeeper
    ports:
      - "9092:9092"
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_AUTO_CREATE_TOPICS_ENABLE: true

  kafka-ui:
    image: provectuslabs/kafka-ui:latest
    depends_on:
      - kafka
    ports:
      - "8080:8080"
    environment:
      KAFKA_CLUSTERS_0_NAME: local
      KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS: kafka:9092
```

### KafkaJS Client Setup

```javascript
// kafka/client.js
const { Kafka, logLevel } = require('kafkajs');

const kafka = new Kafka({
  clientId: process.env.KAFKA_CLIENT_ID || 'my-app',
  brokers: (process.env.KAFKA_BROKERS || 'localhost:9092').split(','),
  logLevel: process.env.NODE_ENV === 'production' ? logLevel.WARN : logLevel.INFO,
  
  // Retry configuration
  retry: {
    initialRetryTime: 300,
    retries: 10
  },
  
  // SSL (สำหรับ production)
  ssl: process.env.KAFKA_SSL === 'true',
  
  // SASL Authentication (Confluent Cloud)
  ...(process.env.KAFKA_SASL_USERNAME && {
    sasl: {
      mechanism: 'plain',
      username: process.env.KAFKA_SASL_USERNAME,
      password: process.env.KAFKA_SASL_PASSWORD
    }
  })
});

module.exports = kafka;
```

---

## 3. Producers

```javascript
// kafka/producer.js
const kafka = require('./client');

class KafkaProducer {
  constructor() {
    this.producer = kafka.producer({
      allowAutoTopicCreation: true,
      transactionTimeout: 30000,
      
      // Idempotent producer (ป้องกัน duplicate messages)
      idempotent: true,
      
      // Batching
      maxInFlightRequests: 5
    });
    
    this.connected = false;
  }

  async connect() {
    if (!this.connected) {
      await this.producer.connect();
      this.connected = true;
      console.log('Kafka producer connected');
    }
  }

  async disconnect() {
    await this.producer.disconnect();
    this.connected = false;
  }

  async send(topic, messages) {
    await this.connect();
    
    const formattedMessages = Array.isArray(messages)
      ? messages.map(msg => this.formatMessage(msg))
      : [this.formatMessage(messages)];

    return this.producer.send({
      topic,
      messages: formattedMessages
    });
  }

  async sendBatch(topicMessages) {
    await this.connect();
    
    return this.producer.sendBatch({
      topicMessages: topicMessages.map(({ topic, messages }) => ({
        topic,
        messages: messages.map(msg => this.formatMessage(msg))
      }))
    });
  }

  // Transactional producer
  async sendWithTransaction(topicMessages) {
    await this.connect();
    
    const transaction = await this.producer.transaction();
    
    try {
      await transaction.sendBatch({ topicMessages });
      await transaction.commit();
    } catch (error) {
      await transaction.abort();
      throw error;
    }
  }

  formatMessage(msg) {
    const value = typeof msg.value === 'object'
      ? JSON.stringify(msg.value)
      : String(msg.value);
    
    return {
      key: msg.key ? String(msg.key) : undefined,
      value,
      headers: {
        ...msg.headers,
        'content-type': 'application/json',
        'timestamp': Date.now().toString()
      }
    };
  }
}

module.exports = new KafkaProducer();
```

### ส่ง Events

```javascript
// services/event.service.js
const producer = require('../kafka/producer');

class EventService {
  async publishOrderCreated(order) {
    await producer.send('order.created', {
      key: order.id,  // partition by order ID
      value: {
        eventType: 'ORDER_CREATED',
        orderId: order.id,
        userId: order.userId,
        total: order.total,
        items: order.items,
        timestamp: new Date().toISOString()
      }
    });
  }

  async publishUserRegistered(user) {
    await producer.send('user.events', {
      key: user.id,
      value: {
        eventType: 'USER_REGISTERED',
        userId: user.id,
        email: user.email,
        name: user.name,
        timestamp: new Date().toISOString()
      }
    });
  }

  async publishPaymentProcessed(payment) {
    await producer.sendBatch([
      {
        topic: 'payment.events',
        messages: [{
          key: payment.orderId,
          value: { eventType: 'PAYMENT_PROCESSED', ...payment }
        }]
      },
      {
        topic: 'notification.events',
        messages: [{
          key: payment.userId,
          value: { 
            type: 'payment_success',
            userId: payment.userId,
            amount: payment.amount
          }
        }]
      }
    ]);
  }
}

module.exports = new EventService();
```

---

## 4. Consumers

```javascript
// kafka/consumer.js
const kafka = require('./client');

class KafkaConsumer {
  constructor(groupId) {
    this.consumer = kafka.consumer({
      groupId,
      sessionTimeout: 30000,
      heartbeatInterval: 3000,
      maxWaitTimeInMs: 5000,
      
      // Auto commit สำหรับ at-most-once
      // ปิดเพื่อ manual commit (at-least-once)
      autoCommit: false
    });
    
    this.handlers = new Map();
    this.running = false;
  }

  async connect() {
    await this.consumer.connect();
    console.log(`Consumer ${this.consumer.groupId} connected`);
  }

  async subscribe(topics) {
    await this.consumer.subscribe({
      topics: Array.isArray(topics) ? topics : [topics],
      fromBeginning: false
    });
  }

  on(eventType, handler) {
    this.handlers.set(eventType, handler);
  }

  async start() {
    this.running = true;
    
    await this.consumer.run({
      eachBatchAutoResolve: false,
      
      eachBatch: async ({ batch, resolveOffset, heartbeat, isRunning, isStale }) => {
        for (const message of batch.messages) {
          if (!isRunning() || isStale()) break;

          try {
            const value = JSON.parse(message.value.toString());
            const eventType = value.eventType;
            
            const handler = this.handlers.get(eventType);
            
            if (handler) {
              await handler(value, message);
            }

            // Manual commit
            resolveOffset(message.offset);
            await heartbeat();
            
          } catch (error) {
            console.error(`Error processing message:`, error);
            // Handle error (DLQ, retry, etc.)
            await this.handleError(error, message, batch);
          }
        }
      }
    });
  }

  async handleError(error, message, batch) {
    // Dead Letter Queue
    const dlqTopic = `${batch.topic}.dlq`;
    
    await producer.send(dlqTopic, {
      key: message.key?.toString(),
      value: {
        originalTopic: batch.topic,
        originalMessage: message.value?.toString(),
        error: error.message,
        failedAt: new Date().toISOString()
      }
    });
    
    // Still resolve offset เพื่อ skip message ที่ fail
    // ขึ้นอยู่กับ business requirement
  }

  async stop() {
    this.running = false;
    await this.consumer.stop();
    await this.consumer.disconnect();
  }
}

module.exports = KafkaConsumer;
```

### Consumer Services

```javascript
// services/order-processor.service.js
const KafkaConsumer = require('../kafka/consumer');
const OrderService = require('./order.service');
const EmailService = require('./email.service');
const InventoryService = require('./inventory.service');

const consumer = new KafkaConsumer('order-processor-group');

async function startOrderProcessor() {
  await consumer.connect();
  await consumer.subscribe(['order.created', 'order.cancelled']);

  // Register handlers
  consumer.on('ORDER_CREATED', async (event) => {
    const { orderId, userId, items } = event;
    
    console.log(`Processing order: ${orderId}`);
    
    // ลด inventory
    await InventoryService.deduct(items);
    
    // ส่ง confirmation email
    await EmailService.sendOrderConfirmation(userId, orderId);
    
    // อัพเดต order status
    await OrderService.updateStatus(orderId, 'confirmed');
  });

  consumer.on('ORDER_CANCELLED', async (event) => {
    const { orderId, items } = event;
    
    // คืน inventory
    await InventoryService.release(items);
    
    // คืน payment
    await PaymentService.refund(orderId);
  });

  await consumer.start();
  console.log('Order processor started');
}

module.exports = { startOrderProcessor };
```

---

## 5. Topics และ Partitions

```javascript
// kafka/admin.js
const kafka = require('./client');

const admin = kafka.admin();

async function createTopics() {
  await admin.connect();
  
  await admin.createTopics({
    topics: [
      {
        topic: 'order.created',
        numPartitions: 6,          // สำหรับ 6 consumers parallel
        replicationFactor: 3,       // HA - ทน broker failure ได้ 2 ตัว
        configEntries: [
          { name: 'retention.ms', value: '604800000' },  // 7 days
          { name: 'cleanup.policy', value: 'delete' }
        ]
      },
      {
        topic: 'user.events',
        numPartitions: 3,
        replicationFactor: 3,
        configEntries: [
          { name: 'retention.ms', value: '2592000000' }  // 30 days
        ]
      },
      {
        topic: 'notification.events',
        numPartitions: 12,
        replicationFactor: 1
      }
    ]
  });

  await admin.disconnect();
}

async function getTopicInfo(topic) {
  await admin.connect();
  
  const [metadata, offsets] = await Promise.all([
    admin.fetchTopicMetadata({ topics: [topic] }),
    admin.fetchTopicOffsets(topic)
  ]);

  await admin.disconnect();
  
  return { metadata: metadata.topics[0], offsets };
}

module.exports = { createTopics, getTopicInfo };
```

---

## 6. Consumer Groups

```javascript
// การทำงานของ Consumer Groups

// Topic: order.events (6 partitions)
// 
// Consumer Group A (order-processor):
//   Consumer 1 → Partitions 0, 1
//   Consumer 2 → Partitions 2, 3
//   Consumer 3 → Partitions 4, 5
//
// Consumer Group B (analytics):
//   Consumer 1 → Partitions 0, 1, 2
//   Consumer 2 → Partitions 3, 4, 5

// ถ้าต้องการ process order events ใน 2 services แยกกัน
// ใช้คนละ consumer group จะได้รับ message เดียวกัน

const orderProcessor = new KafkaConsumer('order-processor-group');
const analyticsProcessor = new KafkaConsumer('analytics-group');

// ทั้งสองจะได้รับ message เดียวกันจาก order.created topic
await orderProcessor.subscribe('order.created');
await analyticsProcessor.subscribe('order.created');
```

---

## แบบฝึกหัด

### ระดับ 1: พื้นฐาน
- Setup Kafka ด้วย Docker
- ส่งและรับ messages
- ทดสอบด้วย Kafka UI

### ระดับ 2: กลาง
- Event-driven order processing
- Consumer groups
- Error handling + DLQ

### ระดับ 3: ขั้นสูง
- Transactional messaging
- Kafka Streams
- Schema Registry + Avro

---

## สรุป

Kafka เหมาะสำหรับ high-throughput event streaming ที่ต้องการ reliability และ scalability สูง การ design partitioning strategy ที่ดีเป็นกุญแจสำคัญ Consumer groups ช่วยให้ services ต่างๆ process events เดียวกันอย่างอิสระ

> ขั้นตอนต่อไป: Part 80 - Real-World Banking API
