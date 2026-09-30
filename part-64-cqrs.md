# Part 64: CQRS Pattern
## ขั้นตอนที่ 631-640 จาก 1000

---

## CQRS คืออะไร?

CQRS (Command Query Responsibility Segregation) เป็น pattern ที่แยก operations ออกเป็น 2 ประเภท:
- **Commands**: เปลี่ยนแปลงข้อมูล (Write operations)
- **Queries**: อ่านข้อมูล (Read operations)

---

## 1. ทำไมต้องใช้ CQRS?

### ปัญหาของ Traditional CRUD

```javascript
// Traditional CRUD - ใช้ model เดียวทำทุกอย่าง
class OrderService {
  // Read
  async getOrder(id) {
    return Order.findById(id);
  }

  // Write
  async createOrder(data) {
    return Order.create(data);
  }

  // ปัญหา: Read และ Write ใช้ model เดียวกัน
  // ทำให้ยาก optimize แยกกัน
}
```

### CQRS แก้ปัญหาอย่างไร

```javascript
// CQRS แยก Read และ Write
class OrderCommandHandler {
  async handle(command) {
    // เฉพาะ Write operations
  }
}

class OrderQueryHandler {
  async handle(query) {
    // เฉพาะ Read operations - อาจใช้ denormalized/cached data
  }
}
```

---

## 2. Command Handlers

### Command Interface

```javascript
// commands/base-command.js
class Command {
  constructor(data) {
    this.id = require('uuid').v4();
    this.timestamp = new Date();
    this.data = data;
  }
}

// commands/create-order.command.js
class CreateOrderCommand extends Command {
  constructor({ userId, items, shippingAddress, paymentMethod }) {
    super({ userId, items, shippingAddress, paymentMethod });
    this.type = 'CREATE_ORDER';
  }
}

// commands/update-order-status.command.js
class UpdateOrderStatusCommand extends Command {
  constructor({ orderId, status, reason }) {
    super({ orderId, status, reason });
    this.type = 'UPDATE_ORDER_STATUS';
  }
}

module.exports = { Command, CreateOrderCommand, UpdateOrderStatusCommand };
```

### Command Handler

```javascript
// handlers/commands/create-order.handler.js
const Order = require('../../models/Order');
const ProductRepository = require('../../repositories/product.repository');
const EventBus = require('../../events/event-bus');
const { OrderCreatedEvent } = require('../../events/order.events');

class CreateOrderHandler {
  constructor() {
    this.productRepository = new ProductRepository();
  }

  async handle(command) {
    const { userId, items, shippingAddress, paymentMethod } = command.data;

    // Validate business rules
    await this.validateItems(items);

    // Calculate total
    const enrichedItems = await this.enrichItemsWithPrices(items);
    const total = this.calculateTotal(enrichedItems);

    // Create order
    const order = await Order.create({
      userId,
      items: enrichedItems,
      total,
      shippingAddress,
      paymentMethod,
      status: 'pending'
    });

    // Reserve inventory
    await this.reserveInventory(items);

    // Publish domain event
    await EventBus.publish(new OrderCreatedEvent({
      orderId: order.id,
      userId,
      total,
      items: enrichedItems
    }));

    return order;
  }

  async validateItems(items) {
    if (!items || items.length === 0) {
      throw new Error('Order must have at least one item');
    }

    for (const item of items) {
      const product = await this.productRepository.findById(item.productId);
      if (!product) {
        throw new Error(`Product ${item.productId} not found`);
      }
      if (product.stock < item.quantity) {
        throw new Error(`Insufficient stock for product ${product.name}`);
      }
    }
  }

  async enrichItemsWithPrices(items) {
    return Promise.all(items.map(async (item) => {
      const product = await this.productRepository.findById(item.productId);
      return {
        ...item,
        name: product.name,
        price: product.price,
        subtotal: product.price * item.quantity
      };
    }));
  }

  calculateTotal(items) {
    return items.reduce((sum, item) => sum + item.subtotal, 0);
  }

  async reserveInventory(items) {
    await Promise.all(items.map(item =>
      this.productRepository.decrementStock(item.productId, item.quantity)
    ));
  }
}

module.exports = CreateOrderHandler;
```

### Command Bus

```javascript
// events/command-bus.js
class CommandBus {
  constructor() {
    this.handlers = new Map();
    this.middlewares = [];
  }

  register(commandType, handler) {
    this.handlers.set(commandType, handler);
    return this;
  }

  use(middleware) {
    this.middlewares.push(middleware);
    return this;
  }

  async execute(command) {
    const handler = this.handlers.get(command.type);
    
    if (!handler) {
      throw new Error(`No handler registered for command: ${command.type}`);
    }

    // Run middlewares
    let index = 0;
    const runNext = async () => {
      if (index < this.middlewares.length) {
        const middleware = this.middlewares[index++];
        await middleware(command, runNext);
      } else {
        return handler.handle(command);
      }
    };

    return runNext();
  }
}

// Setup
const commandBus = new CommandBus();

// Register handlers
const CreateOrderHandler = require('../handlers/commands/create-order.handler');
const UpdateOrderStatusHandler = require('../handlers/commands/update-order-status.handler');

commandBus
  .register('CREATE_ORDER', new CreateOrderHandler())
  .register('UPDATE_ORDER_STATUS', new UpdateOrderStatusHandler());

// Middlewares
commandBus.use(async (command, next) => {
  console.log(`Executing command: ${command.type}`);
  const start = Date.now();
  const result = await next();
  console.log(`Command ${command.type} took ${Date.now() - start}ms`);
  return result;
});

module.exports = commandBus;
```

---

## 3. Query Handlers

### Query Interface

```javascript
// queries/get-orders.query.js
class GetOrdersQuery {
  constructor({ userId, status, page, limit, dateFrom, dateTo }) {
    this.type = 'GET_ORDERS';
    this.userId = userId;
    this.status = status;
    this.page = page || 1;
    this.limit = limit || 10;
    this.dateFrom = dateFrom;
    this.dateTo = dateTo;
  }
}

// queries/get-order-detail.query.js
class GetOrderDetailQuery {
  constructor({ orderId, userId }) {
    this.type = 'GET_ORDER_DETAIL';
    this.orderId = orderId;
    this.userId = userId;
  }
}

module.exports = { GetOrdersQuery, GetOrderDetailQuery };
```

### Query Handler (Read Model)

```javascript
// handlers/queries/get-orders.handler.js
// Query handlers อาจใช้ read-optimized database แยกต่างหาก

class GetOrdersHandler {
  constructor(readDb) {
    this.readDb = readDb; // อาจเป็น Elasticsearch, Redis, etc.
  }

  async handle(query) {
    const { userId, status, page, limit, dateFrom, dateTo } = query;

    const skip = (page - 1) * limit;
    const filter = { userId };

    if (status) filter.status = status;

    if (dateFrom || dateTo) {
      filter.createdAt = {};
      if (dateFrom) filter.createdAt.$gte = new Date(dateFrom);
      if (dateTo) filter.createdAt.$lte = new Date(dateTo);
    }

    const [orders, total] = await Promise.all([
      this.readDb.collection('orders_view').find(filter)
        .sort({ createdAt: -1 })
        .skip(skip)
        .limit(limit)
        .toArray(),
      this.readDb.collection('orders_view').countDocuments(filter)
    ]);

    return {
      data: orders,
      pagination: {
        page,
        limit,
        total,
        pages: Math.ceil(total / limit)
      }
    };
  }
}

// handlers/queries/get-order-detail.handler.js
class GetOrderDetailHandler {
  constructor(readDb) {
    this.readDb = readDb;
  }

  async handle(query) {
    const { orderId, userId } = query;
    
    // ดึงจาก read model (denormalized data)
    const order = await this.readDb.collection('orders_detail_view').findOne({
      _id: orderId,
      userId  // ตรวจสอบ ownership
    });

    if (!order) {
      throw new Error('Order not found or access denied');
    }

    return order;
  }
}

module.exports = { GetOrdersHandler, GetOrderDetailHandler };
```

### Query Bus

```javascript
// events/query-bus.js
class QueryBus {
  constructor() {
    this.handlers = new Map();
    this.cache = null;
  }

  useCache(cacheService) {
    this.cache = cacheService;
    return this;
  }

  register(queryType, handler) {
    this.handlers.set(queryType, handler);
    return this;
  }

  async execute(query) {
    const handler = this.handlers.get(query.type);
    
    if (!handler) {
      throw new Error(`No handler for query: ${query.type}`);
    }

    // Check cache
    if (this.cache && query.cacheable) {
      const cacheKey = this.getCacheKey(query);
      const cached = await this.cache.get(cacheKey);
      
      if (cached) {
        return JSON.parse(cached);
      }

      const result = await handler.handle(query);
      await this.cache.set(cacheKey, JSON.stringify(result), query.cacheTTL || 60);
      
      return result;
    }

    return handler.handle(query);
  }

  getCacheKey(query) {
    return `query:${query.type}:${JSON.stringify(query)}`;
  }
}

module.exports = new QueryBus();
```

---

## 4. Event Sourcing Basics

### Domain Events

```javascript
// events/order.events.js
class DomainEvent {
  constructor(data) {
    this.id = require('uuid').v4();
    this.timestamp = new Date();
    this.data = data;
    this.version = 1;
  }
}

class OrderCreatedEvent extends DomainEvent {
  constructor(data) {
    super(data);
    this.type = 'ORDER_CREATED';
    this.aggregateId = data.orderId;
    this.aggregateType = 'Order';
  }
}

class OrderStatusUpdatedEvent extends DomainEvent {
  constructor(data) {
    super(data);
    this.type = 'ORDER_STATUS_UPDATED';
    this.aggregateId = data.orderId;
    this.aggregateType = 'Order';
  }
}

class OrderCancelledEvent extends DomainEvent {
  constructor(data) {
    super(data);
    this.type = 'ORDER_CANCELLED';
    this.aggregateId = data.orderId;
    this.aggregateType = 'Order';
  }
}

module.exports = {
  OrderCreatedEvent,
  OrderStatusUpdatedEvent,
  OrderCancelledEvent
};
```

### Event Store

```javascript
// repositories/event-store.js
const mongoose = require('mongoose');

const eventSchema = new mongoose.Schema({
  id: { type: String, required: true, unique: true },
  type: { type: String, required: true },
  aggregateId: { type: String, required: true },
  aggregateType: { type: String, required: true },
  data: { type: mongoose.Schema.Types.Mixed },
  version: { type: Number, required: true },
  timestamp: { type: Date, default: Date.now }
});

eventSchema.index({ aggregateId: 1, version: 1 }, { unique: true });
eventSchema.index({ aggregateType: 1, timestamp: 1 });

const EventModel = mongoose.model('Event', eventSchema);

class EventStore {
  async append(event) {
    const lastEvent = await EventModel.findOne(
      { aggregateId: event.aggregateId },
      null,
      { sort: { version: -1 } }
    );

    const expectedVersion = lastEvent ? lastEvent.version + 1 : 1;
    
    if (event.version !== expectedVersion) {
      throw new Error(
        `Concurrency conflict. Expected version ${expectedVersion}, got ${event.version}`
      );
    }

    return EventModel.create({
      id: event.id,
      type: event.type,
      aggregateId: event.aggregateId,
      aggregateType: event.aggregateType,
      data: event.data,
      version: event.version,
      timestamp: event.timestamp
    });
  }

  async getEvents(aggregateId, fromVersion = 1) {
    return EventModel.find({
      aggregateId,
      version: { $gte: fromVersion }
    }).sort({ version: 1 });
  }

  async getAllEvents(aggregateType, fromTimestamp) {
    const filter = { aggregateType };
    if (fromTimestamp) {
      filter.timestamp = { $gte: fromTimestamp };
    }
    return EventModel.find(filter).sort({ timestamp: 1 });
  }
}

module.exports = new EventStore();
```

### Aggregate with Event Sourcing

```javascript
// aggregates/order.aggregate.js
class OrderAggregate {
  constructor() {
    this.id = null;
    this.userId = null;
    this.status = null;
    this.items = [];
    this.total = 0;
    this.version = 0;
    this._uncommittedEvents = [];
  }

  // สร้าง Order จาก events
  static async load(orderId, eventStore) {
    const order = new OrderAggregate();
    const events = await eventStore.getEvents(orderId);
    
    events.forEach(event => order.apply(event, false));
    
    return order;
  }

  // Apply event (เปลี่ยน state)
  apply(event, isNew = true) {
    const methodName = `on${event.type.split('_').map(w => 
      w.charAt(0) + w.slice(1).toLowerCase()
    ).join('')}`;
    
    if (this[methodName]) {
      this[methodName](event.data);
    }
    
    this.version = event.version;
    
    if (isNew) {
      this._uncommittedEvents.push(event);
    }
  }

  // Event handlers
  onOrderCreated(data) {
    this.id = data.orderId;
    this.userId = data.userId;
    this.items = data.items;
    this.total = data.total;
    this.status = 'pending';
  }

  onOrderStatusUpdated(data) {
    this.status = data.status;
  }

  onOrderCancelled(data) {
    this.status = 'cancelled';
  }

  // Commands
  create(data) {
    const { OrderCreatedEvent } = require('../events/order.events');
    const event = new OrderCreatedEvent({
      orderId: require('uuid').v4(),
      ...data
    });
    event.version = 1;
    this.apply(event);
  }

  updateStatus(status, reason) {
    if (!['processing', 'shipped', 'delivered'].includes(status)) {
      throw new Error(`Invalid status: ${status}`);
    }

    const { OrderStatusUpdatedEvent } = require('../events/order.events');
    const event = new OrderStatusUpdatedEvent({
      orderId: this.id,
      status,
      reason
    });
    event.version = this.version + 1;
    this.apply(event);
  }

  cancel(reason) {
    if (['shipped', 'delivered'].includes(this.status)) {
      throw new Error(`Cannot cancel order in ${this.status} status`);
    }

    const { OrderCancelledEvent } = require('../events/order.events');
    const event = new OrderCancelledEvent({ orderId: this.id, reason });
    event.version = this.version + 1;
    this.apply(event);
  }

  getUncommittedEvents() {
    return this._uncommittedEvents;
  }

  clearUncommittedEvents() {
    this._uncommittedEvents = [];
  }
}

module.exports = OrderAggregate;
```

---

## 5. Read Model Projection

```javascript
// projections/order.projection.js
const { OrderCreatedEvent, OrderStatusUpdatedEvent } = require('../events/order.events');
const ReadDB = require('../databases/read-db');

class OrderProjection {
  constructor(readDb) {
    this.readDb = readDb;
  }

  async handle(event) {
    switch (event.type) {
      case 'ORDER_CREATED':
        await this.onOrderCreated(event);
        break;
      case 'ORDER_STATUS_UPDATED':
        await this.onOrderStatusUpdated(event);
        break;
      case 'ORDER_CANCELLED':
        await this.onOrderCancelled(event);
        break;
    }
  }

  async onOrderCreated(event) {
    const { orderId, userId, items, total } = event.data;
    
    // สร้าง read model (denormalized สำหรับ query ง่าย)
    await this.readDb.collection('orders_view').insertOne({
      _id: orderId,
      userId,
      items,
      total,
      status: 'pending',
      itemCount: items.length,
      createdAt: event.timestamp
    });
  }

  async onOrderStatusUpdated(event) {
    await this.readDb.collection('orders_view').updateOne(
      { _id: event.data.orderId },
      {
        $set: {
          status: event.data.status,
          updatedAt: event.timestamp
        }
      }
    );
  }

  async rebuild() {
    // Rebuild read model จาก event store
    await this.readDb.collection('orders_view').drop().catch(() => {});
    
    const eventStore = require('../repositories/event-store');
    const events = await eventStore.getAllEvents('Order');
    
    for (const event of events) {
      await this.handle(event);
    }
    
    console.log('Read model rebuilt successfully');
  }
}

module.exports = OrderProjection;
```

---

## 6. CQRS Controller

```javascript
// controllers/order.controller.js
const commandBus = require('../events/command-bus');
const queryBus = require('../events/query-bus');
const { CreateOrderCommand, UpdateOrderStatusCommand } = require('../commands');
const { GetOrdersQuery, GetOrderDetailQuery } = require('../queries');

// Commands (Write)
exports.createOrder = async (req, res) => {
  const command = new CreateOrderCommand({
    userId: req.user.id,
    ...req.body
  });

  const order = await commandBus.execute(command);
  res.status(201).json({ success: true, data: order });
};

exports.updateOrderStatus = async (req, res) => {
  const command = new UpdateOrderStatusCommand({
    orderId: req.params.id,
    status: req.body.status,
    reason: req.body.reason
  });

  await commandBus.execute(command);
  res.json({ success: true });
};

// Queries (Read)
exports.getOrders = async (req, res) => {
  const query = new GetOrdersQuery({
    userId: req.user.id,
    ...req.query
  });

  const result = await queryBus.execute(query);
  res.json(result);
};

exports.getOrderDetail = async (req, res) => {
  const query = new GetOrderDetailQuery({
    orderId: req.params.id,
    userId: req.user.id
  });

  const order = await queryBus.execute(query);
  res.json(order);
};
```

---

## แบบฝึกหัด

### ระดับ 1: พื้นฐาน
สร้าง simple CQRS system:
- แยก Command และ Query handlers
- มี Command Bus และ Query Bus
- ทดสอบ with simple CRUD

### ระดับ 2: กลาง
สร้าง Order system ด้วย CQRS:
- Command: CreateOrder, UpdateStatus, Cancel
- Query: GetOrders, GetOrderDetail
- Domain events

### ระดับ 3: ขั้นสูง
เพิ่ม Event Sourcing:
- Event Store
- Aggregate reconstruction
- Read model projection
- Event replay

---

## สรุป

CQRS ช่วยแยก concerns ระหว่าง Read และ Write ทำให้ optimize แยกกันได้ สามารถ scale read side และ write side อิสระต่อกัน Event Sourcing เป็น complement ที่ดีสำหรับ CQRS เพราะเก็บ history ทุก state change ไว้

> ขั้นตอนต่อไป: Part 65 - Domain-Driven Design
