# Part 69 | ขั้นตอนที่ 1181-1200 จาก 1000+

# Apache Kafka กับ Node.js

---

## 🎯 เป้าหมายของ Part นี้

หลังจากเรียนจบ Part นี้ คุณจะสามารถ:
1. เข้าใจ Kafka architecture และ concepts
2. สร้าง Producers และ Consumers
3. จัดการ Topics และ Partitions
4. implement Consumer Groups
5. ใช้ Kafka กับ Node.js ด้วย KafkaJS
6. implement Event-Driven Architecture

---

## ขั้นตอนที่ 1181: Apache Kafka คืออะไร?

Apache Kafka เป็น distributed event streaming platform ที่ใช้สำหรับ high-throughput, fault-tolerant messaging

```
Kafka Architecture:
┌─────────────────────────────────────────────────────┐
│                    Kafka Cluster                     │
│                                                      │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐          │
│  │ Broker 1 │  │ Broker 2 │  │ Broker 3 │          │
│  │          │  │          │  │          │          │
│  │ Partition│  │ Partition│  │ Partition│          │
│  │    0     │  │    1     │  │    2     │          │
│  └──────────┘  └──────────┘  └──────────┘          │
│                                                      │
│  Topics: [orders] [payments] [notifications]         │
└─────────────────────────────────────────────────────┘

Producer → Topic → Consumer Group
```

### Key Concepts

1. **Topic** - Named channel สำหรับ events
2. **Partition** - Ordered, immutable sequence of records
3. **Producer** - Application ที่ส่ง events
4. **Consumer** - Application ที่รับ events
5. **Consumer Group** - Group of consumers ที่แบ่งงานกัน
6. **Offset** - Position ใน partition

---

## ขั้นตอนที่ 1182: การติดตั้ง Kafka

```bash
# Docker Compose สำหรับ Kafka
cat > docker-compose.yml << 'EOF'
version: "3"
services:
  zookeeper:
    image: confluentinc/cp-zookeeper:7.5.0
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_TICK_TIME: 2000

  kafka:
    image: confluentinc/cp-kafka:7.5.0
    depends_on:
      - zookeeper
    ports:
      - "9092:9092"
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1

  kafka-ui:
    image: provectuslabs/kafka-ui:latest
    depends_on:
      - kafka
    ports:
      - "8080:8080"
    environment:
      KAFKA_CLUSTERS_0_NAME: local
      KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS: kafka:9092
EOF

docker-compose up -d

# ติดตั้ง KafkaJS
npm install kafkajs

# ติดตั้ง @nestjs/microservices
npm install @nestjs/microservices
```

---

## ขั้นตอนที่ 1183: KafkaJS Client Setup

```typescript
// src/kafka/kafka.client.ts
import { Kafka, Producer, Consumer, Admin, logLevel } from "kafkajs";

export function createKafkaClient(config: {
  clientId: string;
  brokers: string[];
  ssl?: boolean;
  sasl?: {
    mechanism: "plain" | "scram-sha-256" | "scram-sha-512";
    username: string;
    password: string;
  };
}): Kafka {
  return new Kafka({
    clientId: config.clientId,
    brokers: config.brokers,
    ssl: config.ssl,
    sasl: config.sasl,
    logLevel: process.env.NODE_ENV === "development"
      ? logLevel.DEBUG
      : logLevel.ERROR,
    retry: {
      initialRetryTime: 300,
      retries: 10
    }
  });
}
```

---

## ขั้นตอนที่ 1184: Topic Management

```typescript
// src/kafka/admin.service.ts
import { Injectable, OnModuleInit, Logger } from "@nestjs/common";
import { Admin, Kafka } from "kafkajs";

interface TopicConfig {
  name: string;
  numPartitions?: number;
  replicationFactor?: number;
  retention?: string; // e.g., "7d", "604800000"
}

@Injectable()
export class KafkaAdminService implements OnModuleInit {
  private readonly logger = new Logger(KafkaAdminService.name);
  private admin: Admin;

  constructor(private readonly kafka: Kafka) {
    this.admin = kafka.admin();
  }

  async onModuleInit() {
    await this.admin.connect();
    await this.setupTopics();
  }

  private async setupTopics() {
    const topics: TopicConfig[] = [
      { name: "user.created", numPartitions: 3, replicationFactor: 1 },
      { name: "user.updated", numPartitions: 3, replicationFactor: 1 },
      { name: "order.created", numPartitions: 6, replicationFactor: 1 },
      { name: "order.status.updated", numPartitions: 6, replicationFactor: 1 },
      { name: "payment.processed", numPartitions: 3, replicationFactor: 1 },
      { name: "notification.send", numPartitions: 3, replicationFactor: 1 }
    ];

    const existingTopics = await this.admin.listTopics();
    
    const topicsToCreate = topics
      .filter(t => !existingTopics.includes(t.name))
      .map(t => ({
        topic: t.name,
        numPartitions: t.numPartitions ?? 1,
        replicationFactor: t.replicationFactor ?? 1,
        configEntries: t.retention ? [
          { name: "retention.ms", value: t.retention }
        ] : []
      }));

    if (topicsToCreate.length > 0) {
      await this.admin.createTopics({ topics: topicsToCreate });
      this.logger.log(`Created topics: ${topicsToCreate.map(t => t.topic).join(", ")}`);
    }
  }

  async getTopicMetadata(topicName: string) {
    const metadata = await this.admin.fetchTopicMetadata({ topics: [topicName] });
    return metadata.topics[0];
  }

  async getConsumerGroupOffsets(groupId: string, topicName: string) {
    const offsets = await this.admin.fetchOffsets({
      groupId,
      topics: [topicName]
    });
    return offsets;
  }

  async resetOffsets(groupId: string, topicName: string, toEarliest: boolean = true) {
    await this.admin.resetOffsets({
      groupId,
      topic: topicName,
      earliest: toEarliest
    });
  }
}
```

---

## ขั้นตอนที่ 1185: Producer

```typescript
// src/kafka/producer.service.ts
import { Injectable, OnModuleInit, OnModuleDestroy, Logger } from "@nestjs/common";
import { Producer, Kafka, Message, ProducerRecord } from "kafkajs";
import { v4 as uuidv4 } from "uuid";

interface EventMessage<T> {
  key?: string;
  value: T;
  headers?: Record<string, string>;
}

@Injectable()
export class KafkaProducerService implements OnModuleInit, OnModuleDestroy {
  private readonly logger = new Logger(KafkaProducerService.name);
  private producer: Producer;
  private isConnected = false;

  constructor(private readonly kafka: Kafka) {
    this.producer = kafka.producer({
      allowAutoTopicCreation: false,
      transactionTimeout: 30000,
      retry: {
        initialRetryTime: 100,
        retries: 8
      }
    });
  }

  async onModuleInit() {
    await this.connect();
  }

  async onModuleDestroy() {
    await this.disconnect();
  }

  async connect() {
    if (!this.isConnected) {
      await this.producer.connect();
      this.isConnected = true;
      this.logger.log("Producer connected");
    }
  }

  async disconnect() {
    if (this.isConnected) {
      await this.producer.disconnect();
      this.isConnected = false;
    }
  }

  // Send single message
  async send<T>(topic: string, message: EventMessage<T>): Promise<void> {
    const record: ProducerRecord = {
      topic,
      messages: [this.formatMessage(message)]
    };
    
    await this.producer.send(record);
    this.logger.debug(`Sent message to ${topic}: ${message.key}`);
  }

  // Send multiple messages
  async sendBatch<T>(topic: string, messages: EventMessage<T>[]): Promise<void> {
    await this.producer.send({
      topic,
      messages: messages.map(m => this.formatMessage(m))
    });
    this.logger.debug(`Sent ${messages.length} messages to ${topic}`);
  }

  // Send to multiple topics
  async sendToMultiple(records: Array<{
    topic: string;
    messages: EventMessage<any>[];
  }>): Promise<void> {
    await this.producer.sendBatch({
      topicMessages: records.map(r => ({
        topic: r.topic,
        messages: r.messages.map(m => this.formatMessage(m))
      }))
    });
  }

  // Transactional producer
  async sendTransactional<T>(
    operations: Array<{ topic: string; message: EventMessage<T> }>
  ): Promise<void> {
    const transaction = await this.producer.transaction();
    
    try {
      for (const op of operations) {
        await transaction.send({
          topic: op.topic,
          messages: [this.formatMessage(op.message)]
        });
      }
      
      await transaction.commit();
    } catch (error) {
      await transaction.abort();
      throw error;
    }
  }

  private formatMessage<T>(message: EventMessage<T>): Message {
    return {
      key: message.key,
      value: JSON.stringify({
        ...message.value,
        eventId: uuidv4(),
        timestamp: new Date().toISOString()
      }),
      headers: {
        "content-type": "application/json",
        ...message.headers
      }
    };
  }
}
```

---

## ขั้นตอนที่ 1186: Consumer

```typescript
// src/kafka/consumer.service.ts
import {
  Injectable, OnModuleInit, OnModuleDestroy, Logger
} from "@nestjs/common";
import { Consumer, Kafka, EachMessagePayload } from "kafkajs";

interface MessageHandler<T = any> {
  topic: string;
  handler: (message: T, metadata: MessageMetadata) => Promise<void>;
}

interface MessageMetadata {
  partition: number;
  offset: string;
  timestamp: string;
  headers: Record<string, string>;
}

@Injectable()
export class KafkaConsumerService implements OnModuleInit, OnModuleDestroy {
  private readonly logger = new Logger(KafkaConsumerService.name);
  private consumer: Consumer;
  private handlers = new Map<string, (payload: EachMessagePayload) => Promise<void>>();

  constructor(
    private readonly kafka: Kafka,
    private readonly groupId: string
  ) {
    this.consumer = kafka.consumer({
      groupId,
      maxWaitTimeInMs: 5000,
      sessionTimeout: 30000,
      heartbeatInterval: 3000
    });
  }

  async onModuleInit() {
    await this.consumer.connect();
    this.logger.log(`Consumer ${this.groupId} connected`);
  }

  async onModuleDestroy() {
    await this.consumer.disconnect();
  }

  register<T>(handler: MessageHandler<T>): void {
    this.handlers.set(handler.topic, async (payload) => {
      const { message, partition, topic } = payload;
      
      try {
        const value = JSON.parse(message.value?.toString() ?? "{}") as T;
        const headers: Record<string, string> = {};
        
        for (const [key, val] of Object.entries(message.headers ?? {})) {
          headers[key] = val?.toString() ?? "";
        }
        
        await handler.handler(value, {
          partition,
          offset: message.offset,
          timestamp: message.timestamp,
          headers
        });
        
        this.logger.debug(`Processed message from ${topic}[${partition}] offset ${message.offset}`);
      } catch (error) {
        this.logger.error(
          `Error processing message from ${topic}: ${error.message}`,
          error.stack
        );
        throw error;
      }
    });
  }

  async subscribe(): Promise<void> {
    const topics = Array.from(this.handlers.keys());
    
    await this.consumer.subscribe({
      topics,
      fromBeginning: false
    });
    
    await this.consumer.run({
      autoCommit: true,
      autoCommitInterval: 5000,
      eachMessage: async (payload) => {
        const handler = this.handlers.get(payload.topic);
        if (handler) {
          await handler(payload);
        }
      }
    });
    
    this.logger.log(`Subscribed to topics: ${topics.join(", ")}`);
  }
}
```

---

## ขั้นตอนที่ 1187: Consumer Groups

```typescript
// Consumer groups ช่วย scale consumption ของ messages
// แต่ละ consumer ใน group จะรับ partition แยกกัน

// Example: 3 partitions, 3 consumers in group
// Partition 0 → Consumer A
// Partition 1 → Consumer B  
// Partition 2 → Consumer C

// หาก Consumer B ล้มเหลว → Rebalance
// Partition 0 → Consumer A
// Partition 1 → Consumer A หรือ C (rebalanced)
// Partition 2 → Consumer C

// src/order-service/consumers/order.consumer.ts
import { Injectable, OnModuleInit } from "@nestjs/common";
import { KafkaConsumerService } from "../../kafka/consumer.service";
import { OrdersService } from "../orders.service";
import { NotificationsService } from "../../notifications/notifications.service";

interface OrderCreatedEvent {
  eventId: string;
  timestamp: string;
  orderId: string;
  userId: string;
  items: Array<{ productId: string; quantity: number; price: number }>;
  total: number;
}

@Injectable()
export class OrderConsumer implements OnModuleInit {
  constructor(
    private readonly consumer: KafkaConsumerService,
    private readonly ordersService: OrdersService,
    private readonly notificationsService: NotificationsService
  ) {}

  async onModuleInit() {
    this.consumer.register<OrderCreatedEvent>({
      topic: "order.created",
      handler: this.handleOrderCreated.bind(this)
    });
    
    this.consumer.register<any>({
      topic: "order.status.updated",
      handler: this.handleOrderStatusUpdated.bind(this)
    });
    
    await this.consumer.subscribe();
  }

  private async handleOrderCreated(
    event: OrderCreatedEvent,
    metadata: any
  ): Promise<void> {
    // Idempotency check
    const processed = await this.ordersService.isEventProcessed(event.eventId);
    if (processed) {
      return; // Skip duplicate
    }
    
    // Process order
    await this.ordersService.processOrder(event);
    
    // Mark as processed
    await this.ordersService.markEventProcessed(event.eventId);
    
    // Send confirmation notification
    await this.notificationsService.sendOrderConfirmation(
      event.userId,
      event.orderId
    );
  }

  private async handleOrderStatusUpdated(event: any): Promise<void> {
    await this.notificationsService.sendStatusUpdate(
      event.userId,
      event.orderId,
      event.newStatus
    );
  }
}
```

---

## ขั้นตอนที่ 1188: Dead Letter Queue

```typescript
// src/kafka/dlq.service.ts - Dead Letter Queue
import { Injectable } from "@nestjs/common";
import { KafkaProducerService } from "./producer.service";

@Injectable()
export class DeadLetterQueueService {
  constructor(private readonly producer: KafkaProducerService) {}

  async sendToDeadLetter(
    originalTopic: string,
    message: any,
    error: Error,
    metadata: { partition: number; offset: string }
  ): Promise<void> {
    const dlqTopic = `${originalTopic}.dlq`;
    
    await this.producer.send(dlqTopic, {
      key: metadata.offset,
      value: {
        originalTopic,
        originalMessage: message,
        error: {
          message: error.message,
          stack: error.stack,
          name: error.name
        },
        failedAt: new Date().toISOString(),
        partition: metadata.partition,
        offset: metadata.offset
      }
    });
  }
}

// Consumer with DLQ
@Injectable()
export class ResilientConsumer implements OnModuleInit {
  constructor(
    private readonly consumer: KafkaConsumerService,
    private readonly dlq: DeadLetterQueueService
  ) {}

  async onModuleInit() {
    this.consumer.register({
      topic: "order.created",
      handler: async (message, metadata) => {
        try {
          await this.processWithRetry(message, 3);
        } catch (error) {
          await this.dlq.sendToDeadLetter(
            "order.created",
            message,
            error as Error,
            metadata
          );
        }
      }
    });
    
    await this.consumer.subscribe();
  }

  private async processWithRetry(message: any, maxRetries: number): Promise<void> {
    for (let attempt = 1; attempt <= maxRetries; attempt++) {
      try {
        await this.processMessage(message);
        return;
      } catch (error) {
        if (attempt === maxRetries) throw error;
        await new Promise(r => setTimeout(r, 1000 * attempt));
      }
    }
  }

  private async processMessage(message: any): Promise<void> {
    // Process message
  }
}
```

---

## ขั้นตอนที่ 1189: Partitioning Strategies

```typescript
// Custom partitioner
import { Partitioners } from "kafkajs";

// Round-robin (default) - distributes evenly
const roundRobinPartitioner = Partitioners.DefaultPartitioner;

// Custom partitioner - route by userId to same partition
const userPartitioner = ({ topic, partitionMetadata, message }) => {
  const numPartitions = partitionMetadata.length;
  const userId = message.headers?.userId?.toString();
  
  if (userId) {
    // Hash userId to ensure same user goes to same partition
    let hash = 0;
    for (let i = 0; i < userId.length; i++) {
      hash = ((hash << 5) + hash) + userId.charCodeAt(i);
      hash = hash & hash;
    }
    return Math.abs(hash) % numPartitions;
  }
  
  // Random partition if no userId
  return Math.floor(Math.random() * numPartitions);
};

// Using custom partitioner
const producer = kafka.producer({
  createPartitioner: () => userPartitioner
});

// Sending with partition key
await producer.send({
  topic: "user.events",
  messages: [
    {
      key: userId, // Key determines partition
      value: JSON.stringify(event)
    }
  ]
});
```

---

## ขั้นตอนที่ 1190: Kafka Streams Pattern

```typescript
// src/kafka/stream.processor.ts - Stream processing

interface StreamProcessor<TInput, TOutput> {
  process(input: TInput): Promise<TOutput | null>;
}

// Event enrichment processor
@Injectable()
export class OrderEnrichmentProcessor
  implements StreamProcessor<OrderCreated, EnrichedOrder> {
  constructor(
    private readonly userService: UserService,
    private readonly productService: ProductService
  ) {}

  async process(order: OrderCreated): Promise<EnrichedOrder> {
    const [user, products] = await Promise.all([
      this.userService.findById(order.userId),
      Promise.all(order.items.map(i => this.productService.findById(i.productId)))
    ]);
    
    return {
      ...order,
      user: { id: user.id, name: user.name, email: user.email },
      items: order.items.map((item, i) => ({
        ...item,
        product: { id: products[i].id, name: products[i].name }
      }))
    };
  }
}

// Pipeline processor
@Injectable()
export class KafkaPipelineService implements OnModuleInit {
  constructor(
    private readonly consumer: KafkaConsumerService,
    private readonly producer: KafkaProducerService,
    private readonly enricher: OrderEnrichmentProcessor
  ) {}

  async onModuleInit() {
    this.consumer.register({
      topic: "order.created",
      handler: async (rawOrder) => {
        // Enrich
        const enriched = await this.enricher.process(rawOrder);
        
        // Forward to enriched topic
        await this.producer.send("order.enriched", {
          key: rawOrder.orderId,
          value: enriched
        });
      }
    });
    
    await this.consumer.subscribe();
  }
}

interface OrderCreated {
  orderId: string;
  userId: string;
  items: any[];
}

interface EnrichedOrder extends OrderCreated {
  user: any;
}

class UserService {
  async findById(id: string): Promise<any> { return null; }
}

class ProductService {
  async findById(id: string): Promise<any> { return null; }
}
```

---

## ขั้นตอนที่ 1191: Schema Registry

```typescript
// npm install @kafkajs/confluent-schema-registry

// src/kafka/schema-registry.service.ts
import { SchemaRegistry, SchemaType } from "@kafkajs/confluent-schema-registry";
import { Injectable } from "@nestjs/common";

@Injectable()
export class SchemaRegistryService {
  private registry: SchemaRegistry;

  constructor() {
    this.registry = new SchemaRegistry({
      host: process.env.SCHEMA_REGISTRY_URL!
    });
  }

  async register(subject: string, schema: object): Promise<number> {
    const { id } = await this.registry.register({
      type: SchemaType.AVRO,
      schema: JSON.stringify(schema)
    }, { subject });
    
    return id;
  }

  async encode(schemaId: number, value: object): Promise<Buffer> {
    return this.registry.encode(schemaId, value);
  }

  async decode(buffer: Buffer): Promise<any> {
    return this.registry.decode(buffer);
  }
}

// Avro schema example
const OrderCreatedSchema = {
  type: "record",
  name: "OrderCreated",
  namespace: "com.example.orders",
  fields: [
    { name: "eventId", type: "string" },
    { name: "orderId", type: "string" },
    { name: "userId", type: "string" },
    { name: "total", type: "double" },
    { name: "timestamp", type: "string" }
  ]
};
```

---

## ขั้นตอนที่ 1192: Kafka กับ NestJS Microservices

```typescript
// apps/notification-service/src/main.ts
import { NestFactory } from "@nestjs/core";
import { Transport, MicroserviceOptions } from "@nestjs/microservices";

async function bootstrap() {
  const app = await NestFactory.createMicroservice<MicroserviceOptions>(
    NotificationModule,
    {
      transport: Transport.KAFKA,
      options: {
        client: {
          clientId: "notification-service",
          brokers: [process.env.KAFKA_BROKER!],
          retry: { retries: 5 }
        },
        consumer: {
          groupId: "notification-consumer-group",
          sessionTimeout: 30000
        },
        producer: {
          allowAutoTopicCreation: false
        }
      }
    }
  );
  
  await app.listen();
}

// Controller
import { Controller } from "@nestjs/common";
import { MessagePattern, Payload, Ctx, KafkaContext } from "@nestjs/microservices";

@Controller()
export class NotificationController {
  @MessagePattern("notification.send")
  async handleSendNotification(
    @Payload() message: { userId: string; content: string },
    @Ctx() context: KafkaContext
  ) {
    const originalMessage = context.getMessage();
    const partition = context.getPartition();
    const { offset } = originalMessage;
    
    console.log(`Processing notification for user ${message.userId}`);
    console.log(`Partition: ${partition}, Offset: ${offset}`);
    
    // Send notification
  }
}
```

---

## ขั้นตอนที่ 1193: Exactly-Once Semantics

```typescript
// Exactly-once delivery using idempotent producer and transactions
@Injectable()
export class ExactlyOnceProducer implements OnModuleInit {
  private producer: Producer;

  constructor(private readonly kafka: Kafka) {
    this.producer = kafka.producer({
      idempotent: true,                     // Enable idempotent producer
      maxInFlightRequests: 5,              // Required for idempotent
      transactionalId: "transaction-id-1"  // For transactions
    });
  }

  async onModuleInit() {
    await this.producer.connect();
  }

  async processAndPublish(input: any, outputTopic: string): Promise<void> {
    const transaction = await this.producer.transaction();
    
    try {
      // 1. Process message
      const result = await this.processMessage(input);
      
      // 2. Publish result
      await transaction.send({
        topic: outputTopic,
        messages: [
          {
            key: input.id,
            value: JSON.stringify(result)
          }
        ]
      });
      
      // 3. Commit offset of input message
      await transaction.sendOffsets({
        consumerGroupId: "my-consumer-group",
        topics: [
          {
            topic: "input-topic",
            partitions: [
              { partition: input.partition, offset: (parseInt(input.offset) + 1).toString() }
            ]
          }
        ]
      });
      
      await transaction.commit();
    } catch (error) {
      await transaction.abort();
      throw error;
    }
  }

  private async processMessage(input: any): Promise<any> {
    return { ...input, processed: true };
  }
}
```

---

## ขั้นตอนที่ 1194: Monitoring Kafka

```typescript
// src/kafka/monitoring.service.ts
import { Injectable, Logger } from "@nestjs/common";
import { KafkaAdminService } from "./admin.service";
import { InjectMetric } from "@willsoto/nestjs-prometheus";
import { Gauge, Counter } from "prom-client";

@Injectable()
export class KafkaMonitoringService {
  private readonly logger = new Logger(KafkaMonitoringService.name);

  constructor(
    private readonly adminService: KafkaAdminService,
    @InjectMetric("kafka_consumer_lag")
    private readonly consumerLag: Gauge<string>,
    @InjectMetric("kafka_messages_processed")
    private readonly messagesProcessed: Counter<string>
  ) {}

  async measureConsumerLag(groupId: string, topics: string[]) {
    for (const topic of topics) {
      const offsets = await this.adminService.getConsumerGroupOffsets(groupId, topic);
      
      for (const partition of offsets.partitions) {
        const lag = parseInt(partition.logEndOffset) - parseInt(partition.offset);
        
        this.consumerLag.set(
          { group_id: groupId, topic, partition: String(partition.partition) },
          lag
        );
        
        if (lag > 1000) {
          this.logger.warn(
            `High consumer lag detected: ${topic}[${partition.partition}] = ${lag}`
          );
        }
      }
    }
  }

  incrementProcessed(topic: string): void {
    this.messagesProcessed.labels({ topic }).inc();
  }
}
```

---

## ขั้นตอนที่ 1195: Event Sourcing กับ Kafka

```typescript
// src/event-sourcing/event-store.ts
// Kafka as event store
@Injectable()
export class KafkaEventStore {
  private readonly EVENT_TOPIC_PREFIX = "events.";

  constructor(
    private readonly producer: KafkaProducerService,
    private readonly consumer: KafkaConsumerService
  ) {}

  async append(
    aggregateType: string,
    aggregateId: string,
    events: DomainEvent[]
  ): Promise<void> {
    const topic = `${this.EVENT_TOPIC_PREFIX}${aggregateType}`;
    
    await this.producer.sendBatch(
      topic,
      events.map(event => ({
        key: aggregateId,
        value: event,
        headers: {
          "event-type": event.type,
          "aggregate-id": aggregateId,
          "aggregate-type": aggregateType
        }
      }))
    );
  }

  async replay(
    aggregateType: string,
    aggregateId: string,
    handler: (event: DomainEvent) => Promise<void>
  ): Promise<void> {
    const topic = `${this.EVENT_TOPIC_PREFIX}${aggregateType}`;
    
    // Create temporary consumer for replay
    const consumer = new KafkaConsumerService(
      this.kafka,
      `replay-${aggregateId}-${Date.now()}`
    );
    
    await consumer.connect();
    
    // Subscribe from beginning
    await consumer.subscribe({ topic, fromBeginning: true });
    
    // Process all events for this aggregate
    await consumer.run({
      eachMessage: async ({ message }) => {
        if (message.key?.toString() === aggregateId) {
          const event = JSON.parse(message.value?.toString() ?? "{}");
          await handler(event);
        }
      }
    });
  }
}

interface DomainEvent {
  type: string;
  payload: any;
  timestamp: string;
}
```

---

## ขั้นตอนที่ 1196: Kafka Connect สำหรับ CDC

```json
// kafka-connect/postgres-source.json
{
  "name": "postgres-source",
  "config": {
    "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
    "database.hostname": "postgres",
    "database.port": "5432",
    "database.user": "postgres",
    "database.password": "password",
    "database.dbname": "myapp",
    "database.server.name": "myapp",
    "table.include.list": "public.users,public.products,public.orders",
    "plugin.name": "pgoutput",
    "publication.name": "dbz_publication",
    "topic.prefix": "cdc",
    "transforms": "route",
    "transforms.route.type": "org.apache.kafka.connect.transforms.ReplaceField$Value",
    "key.converter": "org.apache.kafka.connect.json.JsonConverter",
    "value.converter": "org.apache.kafka.connect.json.JsonConverter"
  }
}
```

```typescript
// Consuming CDC events
@Injectable()
export class CDCConsumer implements OnModuleInit {
  constructor(private readonly consumer: KafkaConsumerService) {}

  async onModuleInit() {
    this.consumer.register({
      topic: "cdc.myapp.public.users",
      handler: async (event: CDCEvent) => {
        const { op, before, after } = event.payload;
        
        switch (op) {
          case "c": // Create
            await this.handleCreate(after);
            break;
          case "u": // Update
            await this.handleUpdate(before, after);
            break;
          case "d": // Delete
            await this.handleDelete(before);
            break;
        }
      }
    });
    
    await this.consumer.subscribe();
  }

  private async handleCreate(data: any) { /* ... */ }
  private async handleUpdate(before: any, after: any) { /* ... */ }
  private async handleDelete(data: any) { /* ... */ }
}

interface CDCEvent {
  payload: {
    op: "c" | "u" | "d" | "r";
    before: any;
    after: any;
    ts_ms: number;
    source: { table: string; db: string };
  };
}
```

---

## ขั้นตอนที่ 1197: Kafka Streams Alternative (Node.js)

```typescript
// Simple stream processing with Node.js
@Injectable()
export class OrderStreamProcessor implements OnModuleInit {
  private orderCounts = new Map<string, number>();
  private windowDuration = 60 * 1000; // 1 minute
  private windowStart = Date.now();

  constructor(
    private readonly consumer: KafkaConsumerService,
    private readonly producer: KafkaProducerService
  ) {}

  async onModuleInit() {
    this.consumer.register({
      topic: "order.created",
      handler: async (order: any) => {
        await this.processOrderWindow(order);
      }
    });
    
    await this.consumer.subscribe();
    this.startWindowFlush();
  }

  private async processOrderWindow(order: any) {
    const now = Date.now();
    
    // Check if window has passed
    if (now - this.windowStart > this.windowDuration) {
      await this.flushWindow();
      this.windowStart = now;
      this.orderCounts.clear();
    }
    
    // Increment count for user
    const count = (this.orderCounts.get(order.userId) ?? 0) + 1;
    this.orderCounts.set(order.userId, count);
    
    // Alert if too many orders
    if (count > 10) {
      await this.producer.send("fraud.alert", {
        key: order.userId,
        value: {
          userId: order.userId,
          orderCount: count,
          windowMs: this.windowDuration,
          alertType: "high_order_frequency"
        }
      });
    }
  }

  private async flushWindow() {
    const stats = Array.from(this.orderCounts.entries())
      .map(([userId, count]) => ({ userId, count, windowStart: this.windowStart }));
    
    if (stats.length > 0) {
      await this.producer.send("order.window.stats", {
        key: String(this.windowStart),
        value: { stats, windowDuration: this.windowDuration }
      });
    }
  }

  private startWindowFlush() {
    setInterval(() => this.flushWindow(), this.windowDuration);
  }
}
```

---

## ขั้นตอนที่ 1198: Testing Kafka

```typescript
// src/kafka/__tests__/kafka.producer.spec.ts
import { Test } from "@nestjs/testing";
import { KafkaProducerService } from "../producer.service";
import { Kafka } from "kafkajs";

describe("KafkaProducerService", () => {
  let service: KafkaProducerService;
  let mockProducer: any;

  beforeEach(async () => {
    mockProducer = {
      connect: jest.fn(),
      disconnect: jest.fn(),
      send: jest.fn().mockResolvedValue([]),
      sendBatch: jest.fn().mockResolvedValue([]),
      transaction: jest.fn()
    };

    const mockKafka = {
      producer: jest.fn().mockReturnValue(mockProducer)
    };

    const module = await Test.createTestingModule({
      providers: [
        KafkaProducerService,
        { provide: Kafka, useValue: mockKafka }
      ]
    }).compile();

    service = module.get(KafkaProducerService);
    await service.onModuleInit();
  });

  it("should send message to topic", async () => {
    await service.send("test-topic", {
      key: "key1",
      value: { data: "test" }
    });
    
    expect(mockProducer.send).toHaveBeenCalledWith({
      topic: "test-topic",
      messages: expect.arrayContaining([
        expect.objectContaining({ key: "key1" })
      ])
    });
  });
});
```

---

## ขั้นตอนที่ 1199: Complete Kafka Setup

```typescript
// src/kafka/kafka.module.ts
import { DynamicModule, Module } from "@nestjs/common";
import { Kafka } from "kafkajs";
import { KafkaProducerService } from "./producer.service";
import { KafkaAdminService } from "./admin.service";

interface KafkaModuleOptions {
  clientId: string;
  brokers: string[];
  groupId?: string;
}

@Module({})
export class KafkaModule {
  static forRoot(options: KafkaModuleOptions): DynamicModule {
    const kafkaProvider = {
      provide: "KAFKA_CLIENT",
      useFactory: () => new Kafka({
        clientId: options.clientId,
        brokers: options.brokers
      })
    };

    return {
      module: KafkaModule,
      global: true,
      providers: [
        kafkaProvider,
        {
          provide: KafkaProducerService,
          useFactory: (kafka: Kafka) => new KafkaProducerService(kafka),
          inject: ["KAFKA_CLIENT"]
        },
        {
          provide: KafkaAdminService,
          useFactory: (kafka: Kafka) => new KafkaAdminService(kafka),
          inject: ["KAFKA_CLIENT"]
        }
      ],
      exports: [KafkaProducerService, KafkaAdminService]
    };
  }
}
```

---

## ขั้นตอนที่ 1200: Production Tips

```typescript
// Best practices for Kafka in production

// 1. Message compression
const producer = kafka.producer({
  compression: CompressionTypes.GZIP // or SNAPPY, LZ4
});

// 2. Consumer pause/resume
await consumer.pause([{ topic: "orders", partitions: [0] }]);
// ... do some work
await consumer.resume([{ topic: "orders", partitions: [0] }]);

// 3. Seek to specific offset
await consumer.seek({ topic: "orders", partition: 0, offset: "100" });

// 4. Graceful shutdown
process.on("SIGTERM", async () => {
  await consumer.disconnect();
  await producer.disconnect();
  process.exit(0);
});

// 5. Health check
async function healthCheck() {
  const admin = kafka.admin();
  await admin.connect();
  const topics = await admin.listTopics();
  await admin.disconnect();
  return { status: "healthy", topics: topics.length };
}
```

---

## 🏋️ แบบฝึกหัด

### แบบฝึกหัดที่ 1: Order Processing Pipeline
1. สร้าง order.created topic
2. implement producer ใน order service
3. สร้าง consumers ใน payment service, inventory service, notification service

### แบบฝึกหัดที่ 2: Event Sourcing
1. ใช้ Kafka เป็น event store
2. implement CQRS กับ Kafka
3. สร้าง projection สำหรับ read model

### แบบฝึกหัดที่ 3: Stream Processing
1. สร้าง windowed aggregations
2. implement fraud detection pattern
3. สร้าง real-time dashboard

---

## 📚 สรุป

ใน Part นี้เราได้เรียนรู้:
- Kafka concepts: Topics, Partitions, Consumer Groups
- KafkaJS producer และ consumer
- Dead Letter Queue pattern
- Exactly-once semantics
- Schema Registry
- CDC (Change Data Capture)
- Stream processing patterns
- Production deployment

**Part ถัดไป**: Distributed Tracing
