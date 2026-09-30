# Part 85 | ขั้นตอนที่ 1501-1520 จาก 1000+

## Event-Driven Architecture สำหรับ Node.js

ในส่วนนี้เราจะเรียนรู้การสร้างระบบ event-driven ที่ scale ได้ รวมถึง event storming, choreography vs orchestration

---

## ขั้นตอนที่ 1501: Event-Driven Architecture Fundamentals

```javascript
// event-bus.js
// Core event bus implementation

const EventEmitter = require('events');

class EventBus extends EventEmitter {
  constructor() {
    super();
    this.setMaxListeners(100);
    this.eventLog = [];
    this.middlewares = [];
  }

  use(middleware) {
    this.middlewares.push(middleware);
    return this;
  }

  async publish(eventType, payload, metadata = {}) {
    const event = {
      id: require('crypto').randomUUID(),
      type: eventType,
      payload,
      metadata: {
        timestamp: new Date().toISOString(),
        source: process.env.SERVICE_NAME || 'unknown',
        correlationId: metadata.correlationId || require('crypto').randomUUID(),
        causationId: metadata.causationId,
        version: metadata.version || 1,
        ...metadata
      }
    };

    // Run through middlewares
    let processedEvent = event;
    for (const middleware of this.middlewares) {
      processedEvent = await middleware(processedEvent) || processedEvent;
    }

    this.eventLog.push({ event: processedEvent, publishedAt: Date.now() });
    this.emit(eventType, processedEvent);
    this.emit('*', processedEvent); // Wildcard listener
    
    return processedEvent;
  }

  subscribe(eventType, handler, options = {}) {
    const wrappedHandler = async (event) => {
      try {
        await handler(event);
      } catch (err) {
        console.error(`Error handling event ${eventType}:`, err);
        if (options.onError) {
          options.onError(err, event);
        }
      }
    };

    this.on(eventType, wrappedHandler);

    // Return unsubscribe function
    return () => this.off(eventType, wrappedHandler);
  }

  getEventLog(limit = 100) {
    return this.eventLog.slice(-limit);
  }
}

// Middleware examples
const loggingMiddleware = async (event) => {
  console.log(`[EVENT] ${event.type}`, {
    id: event.id,
    correlationId: event.metadata.correlationId
  });
  return event;
};

const validationMiddleware = (schemas) => async (event) => {
  const schema = schemas[event.type];
  if (schema) {
    const valid = schema.validate(event.payload);
    if (!valid) throw new Error(`Invalid event payload for ${event.type}`);
  }
  return event;
};

const enrichmentMiddleware = async (event) => {
  event.metadata.processedAt = Date.now();
  return event;
};

// Create and configure event bus
const bus = new EventBus();
bus.use(loggingMiddleware);
bus.use(enrichmentMiddleware);

module.exports = { EventBus, bus };
```

---

## ขั้นตอนที่ 1502: Domain Events Pattern

```javascript
// domain-events.js
// Domain-driven event patterns

class DomainEvent {
  constructor(aggregateId, type, payload) {
    this.id = require('crypto').randomUUID();
    this.aggregateId = aggregateId;
    this.type = type;
    this.payload = payload;
    this.occurredAt = new Date().toISOString();
    this.version = 1;
  }
}

// Order Domain Events
class OrderCreated extends DomainEvent {
  constructor(orderId, orderData) {
    super(orderId, 'order.created', {
      orderId,
      userId: orderData.userId,
      items: orderData.items,
      total: orderData.total
    });
  }
}

class OrderPaid extends DomainEvent {
  constructor(orderId, paymentData) {
    super(orderId, 'order.paid', {
      orderId,
      paymentId: paymentData.paymentId,
      amount: paymentData.amount,
      method: paymentData.method
    });
  }
}

class OrderShipped extends DomainEvent {
  constructor(orderId, shippingData) {
    super(orderId, 'order.shipped', {
      orderId,
      trackingNumber: shippingData.trackingNumber,
      carrier: shippingData.carrier,
      estimatedDelivery: shippingData.estimatedDelivery
    });
  }
}

class OrderCancelled extends DomainEvent {
  constructor(orderId, reason) {
    super(orderId, 'order.cancelled', { orderId, reason });
  }
}

// Order Aggregate with domain events
class Order {
  constructor() {
    this._domainEvents = [];
    this.status = null;
    this.id = null;
  }

  create(id, userId, items) {
    if (this.status !== null) throw new Error('Order already exists');
    
    this.id = id;
    this.userId = userId;
    this.items = items;
    this.total = items.reduce((sum, item) => sum + item.price * item.quantity, 0);
    this.status = 'pending';
    
    this._domainEvents.push(new OrderCreated(id, {
      userId, items, total: this.total
    }));
    
    return this;
  }

  pay(paymentId, amount, method) {
    if (this.status !== 'pending') throw new Error('Order must be pending to pay');
    if (amount !== this.total) throw new Error('Payment amount mismatch');
    
    this.paymentId = paymentId;
    this.status = 'paid';
    
    this._domainEvents.push(new OrderPaid(this.id, { paymentId, amount, method }));
    
    return this;
  }

  ship(trackingNumber, carrier) {
    if (this.status !== 'paid') throw new Error('Order must be paid to ship');
    
    this.trackingNumber = trackingNumber;
    this.carrier = carrier;
    this.status = 'shipped';
    
    this._domainEvents.push(new OrderShipped(this.id, {
      trackingNumber,
      carrier,
      estimatedDelivery: new Date(Date.now() + 7 * 24 * 60 * 60 * 1000).toISOString()
    }));
    
    return this;
  }

  cancel(reason) {
    if (['shipped', 'delivered'].includes(this.status)) {
      throw new Error('Cannot cancel shipped or delivered order');
    }
    
    this.status = 'cancelled';
    this._domainEvents.push(new OrderCancelled(this.id, reason));
    
    return this;
  }

  pullDomainEvents() {
    const events = [...this._domainEvents];
    this._domainEvents = [];
    return events;
  }
}

// Repository publishes events after save
class OrderRepository {
  constructor(db, eventBus) {
    this.db = db;
    this.eventBus = eventBus;
  }

  async save(order) {
    await this.db.transaction(async (client) => {
      // Save order state
      await client.query(
        'INSERT INTO orders (id, user_id, status, total, items) VALUES ($1, $2, $3, $4, $5) ON CONFLICT (id) DO UPDATE SET status = $3',
        [order.id, order.userId, order.status, order.total, JSON.stringify(order.items)]
      );
      
      // Store events for outbox pattern
      const events = order.pullDomainEvents();
      for (const event of events) {
        await client.query(
          'INSERT INTO domain_events (id, aggregate_id, type, payload, occurred_at) VALUES ($1, $2, $3, $4, $5)',
          [event.id, event.aggregateId, event.type, JSON.stringify(event.payload), event.occurredAt]
        );
      }
    });
    
    // Publish events (after successful transaction)
    const events = order._publishedEvents || [];
    for (const event of events) {
      await this.eventBus.publish(event.type, event.payload, {
        correlationId: event.id
      });
    }
  }
}

module.exports = { Order, OrderRepository, OrderCreated, OrderPaid, OrderShipped };
```

---

## ขั้นตอนที่ 1503: Choreography vs Orchestration

```javascript
// choreography.js
// Services react to events independently (no central coordinator)

const { bus } = require('./event-bus');

// === CHOREOGRAPHY PATTERN ===
// Each service listens and reacts independently

// Inventory Service
bus.subscribe('order.created', async (event) => {
  const { orderId, items } = event.payload;
  
  try {
    // Check and reserve inventory
    for (const item of items) {
      const available = await checkInventory(item.productId);
      if (available < item.quantity) {
        await bus.publish('inventory.reservation.failed', {
          orderId,
          productId: item.productId,
          reason: 'Insufficient stock'
        }, { causationId: event.id });
        return;
      }
    }
    
    await reserveInventory(orderId, items);
    
    await bus.publish('inventory.reserved', {
      orderId,
      items
    }, { causationId: event.id });
    
  } catch (err) {
    await bus.publish('inventory.reservation.failed', {
      orderId,
      error: err.message
    });
  }
});

// Payment Service
bus.subscribe('inventory.reserved', async (event) => {
  const { orderId } = event.payload;
  
  try {
    const order = await getOrder(orderId);
    const payment = await processPayment(order);
    
    await bus.publish('payment.processed', {
      orderId,
      paymentId: payment.id,
      amount: payment.amount
    }, { causationId: event.id });
    
  } catch (err) {
    await bus.publish('payment.failed', {
      orderId,
      error: err.message
    });
    
    // Trigger compensation
    await bus.publish('inventory.release', {
      orderId
    });
  }
});

// Notification Service
bus.subscribe('payment.processed', async (event) => {
  const { orderId } = event.payload;
  const order = await getOrder(orderId);
  
  await sendEmail({
    to: order.userEmail,
    subject: 'Payment Confirmed',
    template: 'payment-confirmed',
    data: { orderId, amount: event.payload.amount }
  });
});

bus.subscribe('payment.failed', async (event) => {
  const { orderId } = event.payload;
  const order = await getOrder(orderId);
  
  await sendEmail({
    to: order.userEmail,
    subject: 'Payment Failed',
    template: 'payment-failed',
    data: { orderId, reason: event.payload.error }
  });
});

// Inventory compensation
bus.subscribe('inventory.release', async (event) => {
  await releaseInventory(event.payload.orderId);
});
```

```javascript
// orchestration.js
// Central coordinator manages the workflow

class OrderFulfillmentOrchestrator {
  constructor(services) {
    this.inventoryService = services.inventory;
    this.paymentService = services.payment;
    this.shippingService = services.shipping;
    this.notificationService = services.notification;
    
    this.workflows = new Map(); // orderId -> workflow state
  }

  async startWorkflow(orderId, orderData) {
    const workflow = {
      orderId,
      status: 'started',
      steps: [],
      startedAt: Date.now()
    };
    
    this.workflows.set(orderId, workflow);
    
    try {
      // Step 1: Reserve Inventory
      console.log(`[${orderId}] Step 1: Reserving inventory...`);
      const reservation = await this.inventoryService.reserve(orderId, orderData.items);
      workflow.steps.push({ step: 'inventory', status: 'done', result: reservation });
      
      // Step 2: Process Payment
      console.log(`[${orderId}] Step 2: Processing payment...`);
      const payment = await this.paymentService.charge({
        orderId,
        amount: orderData.total,
        paymentMethod: orderData.paymentMethod
      });
      workflow.steps.push({ step: 'payment', status: 'done', result: payment });
      
      // Step 3: Create Shipment
      console.log(`[${orderId}] Step 3: Creating shipment...`);
      const shipment = await this.shippingService.createShipment({
        orderId,
        address: orderData.shippingAddress,
        items: orderData.items
      });
      workflow.steps.push({ step: 'shipping', status: 'done', result: shipment });
      
      // Step 4: Send Notification
      console.log(`[${orderId}] Step 4: Sending notification...`);
      await this.notificationService.sendOrderConfirmation({
        orderId,
        userId: orderData.userId,
        trackingNumber: shipment.trackingNumber
      });
      workflow.steps.push({ step: 'notification', status: 'done' });
      
      workflow.status = 'completed';
      workflow.completedAt = Date.now();
      
      console.log(`[${orderId}] Workflow completed successfully`);
      
      return workflow;
      
    } catch (err) {
      workflow.status = 'failed';
      workflow.error = err.message;
      workflow.failedAt = Date.now();
      
      console.error(`[${orderId}] Workflow failed at step:`, err);
      
      // Compensate
      await this.compensate(orderId, workflow.steps);
      
      throw err;
    }
  }

  async compensate(orderId, completedSteps) {
    console.log(`[${orderId}] Starting compensation...`);
    
    const compensations = completedSteps
      .filter(step => step.status === 'done')
      .reverse()
      .map(step => this.getCompensation(orderId, step));
    
    await Promise.allSettled(compensations);
    console.log(`[${orderId}] Compensation complete`);
  }

  async getCompensation(orderId, step) {
    switch (step.step) {
      case 'inventory':
        return this.inventoryService.release(orderId);
      case 'payment':
        return this.paymentService.refund(step.result.paymentId);
      case 'shipping':
        return this.shippingService.cancelShipment(step.result.shipmentId);
      default:
        return Promise.resolve();
    }
  }

  getWorkflowStatus(orderId) {
    return this.workflows.get(orderId) || null;
  }
}

// API
const express = require('express');
const app = express();
app.use(express.json());

const orchestrator = new OrderFulfillmentOrchestrator({
  inventory: inventoryService,
  payment: paymentService,
  shipping: shippingService,
  notification: notificationService
});

app.post('/orders', async (req, res) => {
  const orderId = require('crypto').randomUUID();
  
  try {
    const workflow = await orchestrator.startWorkflow(orderId, req.body);
    res.json({ orderId, status: workflow.status });
  } catch (err) {
    res.status(400).json({ error: err.message, orderId });
  }
});

app.get('/orders/:orderId/workflow', (req, res) => {
  const status = orchestrator.getWorkflowStatus(req.params.orderId);
  if (!status) return res.status(404).json({ error: 'Workflow not found' });
  res.json(status);
});

module.exports = { OrderFulfillmentOrchestrator };
```

---

## ขั้นตอนที่ 1504: Event Storming

```javascript
// event-storming-model.js
// นำ event storming results มาเป็น code

/*
Event Storming Artifacts:
- Domain Events (สิ่งที่เกิดขึ้น - orange sticky notes)
- Commands (สิ่งที่ trigger events - blue sticky notes)
- Aggregates (entities - yellow sticky notes)
- Policies (automated responses - purple sticky notes)
- External Systems (pink sticky notes)
- Read Models (views - green sticky notes)
*/

// ผลจาก Event Storming ของ E-Commerce System:

const ecommerceEvents = {
  // Customer Journey
  customerRegistered: { type: 'customer.registered', aggregate: 'Customer' },
  emailVerified: { type: 'customer.email_verified', aggregate: 'Customer' },
  
  // Product Catalog
  productAdded: { type: 'product.added', aggregate: 'Product' },
  productPriceChanged: { type: 'product.price_changed', aggregate: 'Product' },
  productOutOfStock: { type: 'product.out_of_stock', aggregate: 'Product' },
  
  // Shopping Cart
  itemAddedToCart: { type: 'cart.item_added', aggregate: 'Cart' },
  itemRemovedFromCart: { type: 'cart.item_removed', aggregate: 'Cart' },
  cartAbandoned: { type: 'cart.abandoned', aggregate: 'Cart' },
  checkoutStarted: { type: 'cart.checkout_started', aggregate: 'Cart' },
  
  // Order
  orderCreated: { type: 'order.created', aggregate: 'Order' },
  orderConfirmed: { type: 'order.confirmed', aggregate: 'Order' },
  orderCancelled: { type: 'order.cancelled', aggregate: 'Order' },
  
  // Payment
  paymentInitiated: { type: 'payment.initiated', aggregate: 'Payment' },
  paymentSucceeded: { type: 'payment.succeeded', aggregate: 'Payment' },
  paymentFailed: { type: 'payment.failed', aggregate: 'Payment' },
  
  // Fulfillment
  warehousePickupStarted: { type: 'fulfillment.pickup_started', aggregate: 'Fulfillment' },
  orderShipped: { type: 'fulfillment.shipped', aggregate: 'Fulfillment' },
  orderDelivered: { type: 'fulfillment.delivered', aggregate: 'Fulfillment' }
};

// Bounded Contexts จาก Event Storming
const boundedContexts = {
  catalog: {
    events: ['product.added', 'product.price_changed', 'product.out_of_stock'],
    commands: ['AddProduct', 'UpdatePrice', 'UpdateInventory'],
    aggregates: ['Product', 'Category']
  },
  
  ordering: {
    events: ['order.created', 'order.confirmed', 'order.cancelled'],
    commands: ['CreateOrder', 'ConfirmOrder', 'CancelOrder'],
    aggregates: ['Order', 'Cart']
  },
  
  payment: {
    events: ['payment.initiated', 'payment.succeeded', 'payment.failed'],
    commands: ['InitiatePayment', 'RefundPayment'],
    aggregates: ['Payment', 'Invoice']
  },
  
  fulfillment: {
    events: ['fulfillment.pickup_started', 'fulfillment.shipped', 'fulfillment.delivered'],
    commands: ['StartPickup', 'ShipOrder', 'MarkDelivered'],
    aggregates: ['Shipment', 'WarehouseOrder']
  }
};

// Policy: Automatic reactions to events
const policies = [
  {
    name: 'NotifyCustomerOnOrderConfirmed',
    trigger: 'order.confirmed',
    action: async (event) => {
      await notificationService.send({
        userId: event.payload.userId,
        type: 'order_confirmed',
        data: event.payload
      });
    }
  },
  {
    name: 'UpdateInventoryOnOrderCreated',
    trigger: 'order.created',
    action: async (event) => {
      for (const item of event.payload.items) {
        await inventoryService.reserve(item.productId, item.quantity);
      }
    }
  },
  {
    name: 'SendAbandonedCartEmail',
    trigger: 'cart.abandoned',
    action: async (event) => {
      // Wait 24 hours, then send email if order not placed
      await scheduleEmail({
        userId: event.payload.userId,
        template: 'abandoned-cart',
        delay: 24 * 60 * 60 * 1000,
        cartId: event.payload.cartId
      });
    }
  }
];

// Register policies
const registerPolicies = (bus) => {
  policies.forEach(policy => {
    bus.subscribe(policy.trigger, async (event) => {
      console.log(`[Policy] ${policy.name} triggered by ${event.type}`);
      await policy.action(event);
    });
  });
};

module.exports = { ecommerceEvents, boundedContexts, registerPolicies };
```

---

## ขั้นตอนที่ 1505: Message Broker Integration

```javascript
// kafka-integration.js
// Apache Kafka สำหรับ production event streaming

const { Kafka } = require('kafkajs');

const kafka = new Kafka({
  clientId: 'node-api',
  brokers: process.env.KAFKA_BROKERS?.split(',') || ['kafka:9092'],
  ssl: process.env.NODE_ENV === 'production',
  sasl: process.env.KAFKA_USERNAME ? {
    mechanism: 'scram-sha-256',
    username: process.env.KAFKA_USERNAME,
    password: process.env.KAFKA_PASSWORD
  } : undefined,
  retry: {
    initialRetryTime: 100,
    retries: 8
  }
});

// Producer
class EventProducer {
  constructor() {
    this.producer = kafka.producer({
      idempotent: true,        // Exactly-once semantics
      maxInFlightRequests: 5
    });
    this.connected = false;
  }

  async connect() {
    await this.producer.connect();
    this.connected = true;
    console.log('Kafka producer connected');
  }

  async publish(topic, events) {
    if (!this.connected) await this.connect();
    
    const messages = (Array.isArray(events) ? events : [events]).map(event => ({
      key: event.aggregateId || event.id,
      value: JSON.stringify(event),
      headers: {
        'event-type': event.type,
        'correlation-id': event.metadata?.correlationId || '',
        'source-service': process.env.SERVICE_NAME || 'unknown'
      }
    }));
    
    await this.producer.send({ topic, messages });
  }

  async publishBatch(topicMessages) {
    // topicMessages: [{ topic, messages: [...] }]
    await this.producer.sendBatch({ topicMessages });
  }

  async disconnect() {
    await this.producer.disconnect();
    this.connected = false;
  }
}

// Consumer
class EventConsumer {
  constructor(groupId, handlers) {
    this.consumer = kafka.consumer({
      groupId,
      sessionTimeout: 30000,
      heartbeatInterval: 10000,
      maxBytesPerPartition: 1048576 // 1MB
    });
    this.handlers = handlers;
    this.running = false;
  }

  async start(topics) {
    await this.consumer.connect();
    
    await this.consumer.subscribe({
      topics,
      fromBeginning: false
    });
    
    this.running = true;
    
    await this.consumer.run({
      autoCommit: false,
      eachMessage: async ({ topic, partition, message, heartbeat }) => {
        const event = JSON.parse(message.value.toString());
        const eventType = message.headers['event-type']?.toString();
        
        try {
          const handler = this.handlers[eventType] || this.handlers['*'];
          
          if (handler) {
            await handler(event, { topic, partition, offset: message.offset });
          }
          
          // Manual commit after successful processing
          await this.consumer.commitOffsets([{
            topic,
            partition,
            offset: String(Number(message.offset) + 1)
          }]);
          
        } catch (err) {
          console.error(`Error processing event ${eventType}:`, err);
          
          // Dead letter queue
          await this.sendToDeadLetterQueue(topic, message, err);
          
          // Commit even on error (don't block the consumer)
          await this.consumer.commitOffsets([{
            topic,
            partition,
            offset: String(Number(message.offset) + 1)
          }]);
        }
        
        // Keep consumer alive
        await heartbeat();
      }
    });
  }

  async sendToDeadLetterQueue(originalTopic, message, error) {
    const dlqProducer = new EventProducer();
    await dlqProducer.connect();
    
    await dlqProducer.publish(`${originalTopic}.dlq`, {
      originalMessage: message.value.toString(),
      error: error.message,
      failedAt: new Date().toISOString()
    });
    
    await dlqProducer.disconnect();
  }

  async stop() {
    this.running = false;
    await this.consumer.disconnect();
  }
}

// Transactional Outbox Pattern
class OutboxProcessor {
  constructor(db, producer) {
    this.db = db;
    this.producer = producer;
    this.processing = false;
  }

  async processOutbox() {
    if (this.processing) return;
    this.processing = true;
    
    try {
      // Fetch unprocessed events
      const result = await this.db.query(`
        SELECT * FROM outbox_events 
        WHERE processed = false 
        ORDER BY created_at ASC 
        LIMIT 100
        FOR UPDATE SKIP LOCKED
      `);
      
      if (result.rows.length === 0) return;
      
      // Publish to Kafka
      const topicMessages = {};
      
      result.rows.forEach(row => {
        const topic = row.topic || `events.${row.aggregate_type.toLowerCase()}`;
        if (!topicMessages[topic]) topicMessages[topic] = [];
        
        topicMessages[topic].push({
          key: row.aggregate_id,
          value: row.payload,
          headers: {
            'event-id': row.id,
            'event-type': row.event_type
          }
        });
      });
      
      await this.producer.publishBatch(
        Object.entries(topicMessages).map(([topic, messages]) => ({ topic, messages }))
      );
      
      // Mark as processed
      const ids = result.rows.map(r => r.id);
      await this.db.query(
        'UPDATE outbox_events SET processed = true, processed_at = NOW() WHERE id = ANY($1)',
        [ids]
      );
      
      console.log(`Processed ${result.rows.length} outbox events`);
      
    } finally {
      this.processing = false;
    }
  }

  start(intervalMs = 1000) {
    this.interval = setInterval(() => this.processOutbox(), intervalMs);
    return this;
  }

  stop() {
    clearInterval(this.interval);
  }
}

module.exports = { EventProducer, EventConsumer, OutboxProcessor };
```

---

## ขั้นตอนที่ 1506-1520: Advanced Event Patterns

### CQRS Pattern

```javascript
// cqrs.js
// Command Query Responsibility Segregation

// Write Model (Commands)
class OrderCommandHandler {
  constructor(orderRepository, eventBus) {
    this.orderRepo = orderRepository;
    this.eventBus = eventBus;
  }

  async handle(command) {
    switch (command.type) {
      case 'CreateOrder':
        return this.createOrder(command);
      case 'ConfirmOrder':
        return this.confirmOrder(command);
      case 'CancelOrder':
        return this.cancelOrder(command);
      default:
        throw new Error(`Unknown command: ${command.type}`);
    }
  }

  async createOrder({ orderId, userId, items }) {
    const order = new Order();
    order.create(orderId, userId, items);
    
    await this.orderRepo.save(order);
    
    const events = order.pullDomainEvents();
    for (const event of events) {
      await this.eventBus.publish(event.type, event.payload);
    }
    
    return { orderId, status: 'created' };
  }

  async confirmOrder({ orderId }) {
    const order = await this.orderRepo.findById(orderId);
    if (!order) throw new Error('Order not found');
    
    order.confirm();
    await this.orderRepo.save(order);
    
    return { orderId, status: 'confirmed' };
  }

  async cancelOrder({ orderId, reason }) {
    const order = await this.orderRepo.findById(orderId);
    if (!order) throw new Error('Order not found');
    
    order.cancel(reason);
    await this.orderRepo.save(order);
    
    return { orderId, status: 'cancelled' };
  }
}

// Read Model (Queries) - denormalized for fast reads
class OrderQueryService {
  constructor(readDb) {
    this.db = readDb; // Can be separate optimized read database
  }

  async getOrder(orderId) {
    const result = await this.db.query(`
      SELECT 
        o.id,
        o.status,
        o.total,
        o.created_at,
        u.name as user_name,
        u.email as user_email,
        json_agg(
          json_build_object(
            'productId', oi.product_id,
            'name', p.name,
            'quantity', oi.quantity,
            'price', oi.price
          )
        ) as items
      FROM orders o
      JOIN users u ON o.user_id = u.id
      JOIN order_items oi ON o.id = oi.order_id
      JOIN products p ON oi.product_id = p.id
      WHERE o.id = $1
      GROUP BY o.id, u.name, u.email
    `, [orderId]);
    
    return result.rows[0] || null;
  }

  async getUserOrders(userId, page = 1, limit = 20) {
    const offset = (page - 1) * limit;
    
    const result = await this.db.query(`
      SELECT id, status, total, created_at
      FROM orders
      WHERE user_id = $1
      ORDER BY created_at DESC
      LIMIT $2 OFFSET $3
    `, [userId, limit, offset]);
    
    return result.rows;
  }

  async getOrderStats(dateFrom, dateTo) {
    const result = await this.db.query(`
      SELECT 
        COUNT(*) as total_orders,
        SUM(total) as total_revenue,
        AVG(total) as avg_order_value,
        COUNT(CASE WHEN status = 'cancelled' THEN 1 END) as cancelled_orders
      FROM orders
      WHERE created_at BETWEEN $1 AND $2
    `, [dateFrom, dateTo]);
    
    return result.rows[0];
  }
}

// Update Read Model based on events
class OrderReadModelUpdater {
  constructor(readDb, eventBus) {
    this.db = readDb;
    
    // Listen to domain events and update read model
    eventBus.subscribe('order.created', (event) => this.handleOrderCreated(event));
    eventBus.subscribe('order.confirmed', (event) => this.handleOrderConfirmed(event));
    eventBus.subscribe('order.shipped', (event) => this.handleOrderShipped(event));
    eventBus.subscribe('order.cancelled', (event) => this.handleOrderCancelled(event));
  }

  async handleOrderCreated(event) {
    const { orderId, userId, items, total } = event.payload;
    
    await this.db.query(
      'INSERT INTO order_summaries (id, user_id, status, total, item_count, created_at) VALUES ($1, $2, $3, $4, $5, $6)',
      [orderId, userId, 'pending', total, items.length, new Date()]
    );
  }

  async handleOrderConfirmed(event) {
    await this.db.query(
      'UPDATE order_summaries SET status = $1, confirmed_at = $2 WHERE id = $3',
      ['confirmed', new Date(), event.payload.orderId]
    );
  }

  async handleOrderShipped(event) {
    await this.db.query(
      'UPDATE order_summaries SET status = $1, tracking_number = $2, shipped_at = $3 WHERE id = $4',
      ['shipped', event.payload.trackingNumber, new Date(), event.payload.orderId]
    );
  }

  async handleOrderCancelled(event) {
    await this.db.query(
      'UPDATE order_summaries SET status = $1, cancel_reason = $2, cancelled_at = $3 WHERE id = $4',
      ['cancelled', event.payload.reason, new Date(), event.payload.orderId]
    );
  }
}

// Express API ด้วย CQRS
const express = require('express');
const app = express();
app.use(express.json());

// Commands -> CommandHandler
app.post('/orders', async (req, res) => {
  const command = {
    type: 'CreateOrder',
    orderId: require('crypto').randomUUID(),
    userId: req.user.id,
    items: req.body.items
  };
  
  const result = await commandHandler.handle(command);
  res.status(201).json(result);
});

app.post('/orders/:id/confirm', async (req, res) => {
  const result = await commandHandler.handle({
    type: 'ConfirmOrder',
    orderId: req.params.id
  });
  res.json(result);
});

// Queries -> QueryService (fast reads)
app.get('/orders/:id', async (req, res) => {
  const order = await queryService.getOrder(req.params.id);
  if (!order) return res.status(404).json({ error: 'Order not found' });
  res.json(order);
});

app.get('/users/:userId/orders', async (req, res) => {
  const orders = await queryService.getUserOrders(
    req.params.userId,
    parseInt(req.query.page) || 1
  );
  res.json(orders);
});

module.exports = { OrderCommandHandler, OrderQueryService, OrderReadModelUpdater };
```

---

## แบบฝึกหัด

### Exercise 1: Implement Event Versioning
สร้างระบบที่รองรับ event schema evolution โดยไม่ทำให้ old consumers พัง

### Exercise 2: Dead Letter Queue Processing
สร้าง worker ที่ process events จาก DLQ พร้อม retry logic และ alerting

### Exercise 3: Event Replay System
สร้างระบบที่ replay events ตั้งแต่ timestamp ใดก็ได้ เพื่อ rebuild state

### คำถามทบทวน
1. Choreography กับ Orchestration ต่างกันอย่างไร? ข้อดีข้อเสียของแต่ละ pattern คืออะไร?
2. CQRS ช่วยแก้ปัญหาอะไร? เมื่อใดควรใช้?
3. Outbox pattern แก้ปัญหา distributed transaction ได้อย่างไร?
4. Event Storming ใช้ทำอะไร และ stakeholders ใดควรมีส่วนร่วม?

---

*ต่อไป: Part 86 - Advanced API Security*
