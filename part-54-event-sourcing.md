# Part 54 | ขั้นตอนที่ 941-960 จาก 1000

## Event Sourcing - การบันทึกข้อมูลแบบ Event

Event Sourcing เป็น pattern การออกแบบที่เก็บ state ของ application เป็น sequence of events แทนที่จะเก็บ current state

---

## ขั้นตอนที่ 941: ทำความเข้าใจ Event Sourcing

### Traditional State vs Event Sourcing:

**Traditional (Current State)**:
```
Order { status: 'delivered', total: 1500 }
```

**Event Sourcing (Events)**:
```
1. OrderCreated { total: 1500 }
2. PaymentReceived { amount: 1500 }
3. OrderShipped { trackingNo: 'TH001' }
4. OrderDelivered { at: '2024-01-01' }
```

### ข้อดีของ Event Sourcing:
- **Full Audit Trail**: ประวัติการเปลี่ยนแปลงทั้งหมด
- **Time Travel**: ย้อนดูสถานะในอดีตได้
- **Replay Events**: สร้าง state ใหม่จาก events
- **Event-driven Architecture**: ง่ายต่อการ integrate
- **Debug Friendly**: เข้าใจ "ทำไมถึงเป็นแบบนี้"

---

## ขั้นตอนที่ 942: Event Store Schema

```javascript
// models/EventStore.js
const mongoose = require('mongoose');

const eventSchema = new mongoose.Schema({
  // Event identity
  eventId: {
    type: String,
    required: true,
    unique: true,
    default: () => require('crypto').randomUUID()
  },
  
  // Aggregate identity
  aggregateId: { type: String, required: true, index: true },
  aggregateType: { type: String, required: true },
  
  // Event info
  eventType: { type: String, required: true },
  version: { type: Number, required: true },
  
  // Event data
  data: { type: mongoose.Schema.Types.Mixed, required: true },
  metadata: {
    userId: String,
    correlationId: String,
    causationId: String,
    timestamp: { type: Date, default: Date.now },
    source: String
  },
  
  // Position สำหรับ ordering
  position: { type: Number }
}, {
  timestamps: true,
  collection: 'event_store'
});

// Compound index สำหรับ optimistic locking
eventSchema.index({ aggregateId: 1, version: 1 }, { unique: true });
eventSchema.index({ aggregateType: 1, eventType: 1 });
eventSchema.index({ 'metadata.timestamp': -1 });
eventSchema.index({ position: 1 });

// Auto-increment position
eventSchema.pre('save', async function() {
  if (this.isNew) {
    const lastEvent = await this.constructor.findOne()
      .sort({ position: -1 })
      .select('position');
    
    this.position = lastEvent ? lastEvent.position + 1 : 1;
  }
});

module.exports = mongoose.model('Event', eventSchema);
```

---

## ขั้นตอนที่ 943: Event Store Service

```javascript
// services/eventStore.js
const Event = require('../models/EventStore');

class EventStore {
  // บันทึก events
  async append(aggregateId, aggregateType, events, expectedVersion = null) {
    // ตรวจสอบ version (Optimistic Concurrency Control)
    if (expectedVersion !== null) {
      const latestEvent = await Event.findOne({ aggregateId })
        .sort({ version: -1 })
        .select('version');
      
      const currentVersion = latestEvent ? latestEvent.version : 0;
      
      if (currentVersion !== expectedVersion) {
        throw new Error(
          `Concurrency conflict: expected version ${expectedVersion}, got ${currentVersion}`
        );
      }
    }
    
    // หา version ปัจจุบัน
    const latestEvent = await Event.findOne({ aggregateId })
      .sort({ version: -1 })
      .select('version');
    
    let version = latestEvent ? latestEvent.version : 0;
    
    // บันทึก events
    const eventDocuments = events.map(event => ({
      aggregateId,
      aggregateType,
      eventType: event.type,
      version: ++version,
      data: event.data,
      metadata: {
        ...event.metadata,
        timestamp: new Date()
      }
    }));
    
    await Event.insertMany(eventDocuments);
    
    return eventDocuments;
  }

  // ดึง events ของ aggregate
  async getEvents(aggregateId, fromVersion = 0) {
    return Event.find({
      aggregateId,
      version: { $gt: fromVersion }
    })
    .sort({ version: 1 })
    .lean();
  }

  // ดึง events ทั้งหมดตั้งแต่ position
  async getAllEvents(fromPosition = 0, limit = 1000) {
    return Event.find({
      position: { $gt: fromPosition }
    })
    .sort({ position: 1 })
    .limit(limit)
    .lean();
  }

  // ดึง events ตาม type
  async getEventsByType(eventType, options = {}) {
    const { skip = 0, limit = 100, fromDate, toDate } = options;
    
    const query = { eventType };
    
    if (fromDate || toDate) {
      query['metadata.timestamp'] = {};
      if (fromDate) query['metadata.timestamp'].$gte = fromDate;
      if (toDate) query['metadata.timestamp'].$lte = toDate;
    }
    
    return Event.find(query)
      .sort({ 'metadata.timestamp': 1 })
      .skip(skip)
      .limit(limit)
      .lean();
  }

  // ลบ events (soft delete สำหรับ GDPR)
  async redactEvent(eventId) {
    return Event.findOneAndUpdate(
      { eventId },
      { 
        data: { redacted: true },
        'metadata.redactedAt': new Date()
      }
    );
  }
}

module.exports = new EventStore();
```

---

## ขั้นตอนที่ 944: Aggregate Base Class

```javascript
// domain/aggregate.js

class AggregateRoot {
  constructor() {
    this._id = null;
    this._version = 0;
    this._uncommittedEvents = [];
  }

  // Apply event (เปลี่ยน state)
  apply(event) {
    // เรียก handler method ตาม event type
    const handlerName = `on${event.type}`;
    
    if (typeof this[handlerName] === 'function') {
      this[handlerName](event.data);
    }
    
    this._version++;
    
    // เพิ่มใน uncommitted events ถ้าเป็น new event
    if (event.isNew) {
      this._uncommittedEvents.push(event);
    }
  }

  // สร้าง event ใหม่
  raiseEvent(type, data) {
    const event = {
      type,
      data,
      isNew: true,
      metadata: {
        timestamp: new Date(),
        version: this._version + 1
      }
    };
    
    this.apply(event);
  }

  // ดึง uncommitted events
  getUncommittedEvents() {
    return [...this._uncommittedEvents];
  }

  // ล้าง uncommitted events หลัง save
  clearUncommittedEvents() {
    this._uncommittedEvents = [];
  }

  // Rebuild state จาก events
  static rebuild(events) {
    const aggregate = new this();
    
    for (const event of events) {
      aggregate.apply({ ...event, isNew: false });
    }
    
    return aggregate;
  }
}

module.exports = AggregateRoot;
```

---

## ขั้นตอนที่ 945: Order Aggregate

```javascript
// domain/order/Order.js
const AggregateRoot = require('../aggregate');

class Order extends AggregateRoot {
  constructor() {
    super();
    this.userId = null;
    this.items = [];
    this.total = 0;
    this.status = null;
    this.shippingAddress = null;
    this.trackingNumber = null;
    this.paymentId = null;
  }

  // === Commands (Actions) ===
  
  // สร้าง order ใหม่
  create({ userId, items, total, shippingAddress }) {
    if (this.status) {
      throw new Error('Order already exists');
    }
    
    if (!userId || !items || items.length === 0) {
      throw new Error('Invalid order data');
    }
    
    this.raiseEvent('OrderCreated', {
      userId,
      items,
      total,
      shippingAddress
    });
    
    return this;
  }

  // ยืนยันการชำระเงิน
  confirmPayment({ paymentId, amount }) {
    if (this.status !== 'pending') {
      throw new Error(`Cannot confirm payment for order in status: ${this.status}`);
    }
    
    if (amount < this.total) {
      throw new Error('Payment amount insufficient');
    }
    
    this.raiseEvent('PaymentConfirmed', { paymentId, amount });
    
    return this;
  }

  // จัดส่งสินค้า
  ship({ trackingNumber, carrier }) {
    if (this.status !== 'paid') {
      throw new Error(`Cannot ship order in status: ${this.status}`);
    }
    
    this.raiseEvent('OrderShipped', { trackingNumber, carrier });
    
    return this;
  }

  // ยืนยันการรับสินค้า
  deliver({ deliveredAt }) {
    if (this.status !== 'shipped') {
      throw new Error(`Cannot deliver order in status: ${this.status}`);
    }
    
    this.raiseEvent('OrderDelivered', { deliveredAt: deliveredAt || new Date() });
    
    return this;
  }

  // ยกเลิก order
  cancel({ reason, cancelledBy }) {
    if (['delivered', 'cancelled'].includes(this.status)) {
      throw new Error(`Cannot cancel order in status: ${this.status}`);
    }
    
    this.raiseEvent('OrderCancelled', { reason, cancelledBy });
    
    return this;
  }

  // === Event Handlers (State Mutations) ===
  
  onOrderCreated({ userId, items, total, shippingAddress }) {
    this.userId = userId;
    this.items = items;
    this.total = total;
    this.shippingAddress = shippingAddress;
    this.status = 'pending';
    this.createdAt = new Date();
  }

  onPaymentConfirmed({ paymentId, amount }) {
    this.paymentId = paymentId;
    this.paidAmount = amount;
    this.status = 'paid';
    this.paidAt = new Date();
  }

  onOrderShipped({ trackingNumber, carrier }) {
    this.trackingNumber = trackingNumber;
    this.carrier = carrier;
    this.status = 'shipped';
    this.shippedAt = new Date();
  }

  onOrderDelivered({ deliveredAt }) {
    this.status = 'delivered';
    this.deliveredAt = deliveredAt;
  }

  onOrderCancelled({ reason, cancelledBy }) {
    this.status = 'cancelled';
    this.cancellationReason = reason;
    this.cancelledBy = cancelledBy;
    this.cancelledAt = new Date();
  }
}

module.exports = Order;
```

---

## ขั้นตอนที่ 946: Order Repository

```javascript
// repositories/OrderRepository.js
const EventStore = require('../services/eventStore');
const Order = require('../domain/order/Order');

class OrderRepository {
  constructor() {
    this.aggregateType = 'Order';
  }

  // บันทึก order
  async save(order) {
    const uncommittedEvents = order.getUncommittedEvents();
    
    if (uncommittedEvents.length === 0) {
      return order;
    }
    
    // บันทึก events
    await EventStore.append(
      order._id || require('crypto').randomUUID(),
      this.aggregateType,
      uncommittedEvents,
      order._version - uncommittedEvents.length // expected version
    );
    
    order.clearUncommittedEvents();
    
    return order;
  }

  // โหลด order จาก events
  async findById(orderId) {
    const events = await EventStore.getEvents(orderId);
    
    if (events.length === 0) {
      return null;
    }
    
    return Order.rebuild(events.map(e => ({
      type: e.eventType,
      data: e.data,
      metadata: e.metadata
    })));
  }

  // โหลด order ณ version ที่กำหนด (Time Travel)
  async findByIdAtVersion(orderId, version) {
    const events = await EventStore.getEvents(orderId);
    const eventsUpToVersion = events.filter(e => e.version <= version);
    
    if (eventsUpToVersion.length === 0) {
      return null;
    }
    
    return Order.rebuild(eventsUpToVersion.map(e => ({
      type: e.eventType,
      data: e.data
    })));
  }
}

module.exports = new OrderRepository();
```

---

## ขั้นตอนที่ 947: CQRS - Command Query Responsibility Segregation

```javascript
// cqrs/commands/CreateOrderCommand.js
class CreateOrderCommand {
  constructor({ userId, items, shippingAddress }) {
    this.userId = userId;
    this.items = items;
    this.shippingAddress = shippingAddress;
    this.timestamp = new Date();
  }
}

// cqrs/commands/handlers/CreateOrderHandler.js
const Order = require('../../domain/order/Order');
const OrderRepository = require('../../repositories/OrderRepository');
const ProductRepository = require('../../repositories/ProductRepository');

class CreateOrderHandler {
  async handle(command) {
    const { userId, items, shippingAddress } = command;
    
    // คำนวณ total จาก products
    const productIds = items.map(i => i.productId);
    const products = await ProductRepository.findByIds(productIds);
    
    const orderItems = items.map(item => {
      const product = products.find(p => p.id === item.productId);
      if (!product) {
        throw new Error(`Product not found: ${item.productId}`);
      }
      if (product.stock < item.quantity) {
        throw new Error(`Insufficient stock: ${product.name}`);
      }
      
      return {
        productId: item.productId,
        name: product.name,
        price: product.price,
        quantity: item.quantity,
        subtotal: product.price * item.quantity
      };
    });
    
    const total = orderItems.reduce((sum, item) => sum + item.subtotal, 0);
    
    // สร้าง Order aggregate
    const order = new Order();
    order._id = require('crypto').randomUUID();
    order.create({ userId, items: orderItems, total, shippingAddress });
    
    // บันทึก events
    await OrderRepository.save(order);
    
    return { orderId: order._id, total };
  }
}

module.exports = { CreateOrderCommand, CreateOrderHandler };
```

---

## ขั้นตอนที่ 948: Read Model (Query Side)

```javascript
// readModels/OrderReadModel.js
const mongoose = require('mongoose');

// Read model ที่ optimize สำหรับ queries
const orderReadModelSchema = new mongoose.Schema({
  orderId: { type: String, required: true, unique: true, index: true },
  userId: { type: String, required: true, index: true },
  status: { type: String, index: true },
  items: [{
    productId: String,
    name: String,
    price: Number,
    quantity: Number,
    subtotal: Number
  }],
  total: Number,
  shippingAddress: {
    street: String,
    city: String,
    country: String
  },
  paymentId: String,
  trackingNumber: String,
  carrier: String,
  createdAt: Date,
  updatedAt: Date,
  deliveredAt: Date,
  cancelledAt: Date,
  version: Number
}, {
  timestamps: true,
  collection: 'order_read_models'
});

// Indexes สำหรับ common queries
orderReadModelSchema.index({ userId: 1, createdAt: -1 });
orderReadModelSchema.index({ status: 1, createdAt: -1 });

const OrderReadModel = mongoose.model('OrderReadModel', orderReadModelSchema);

module.exports = OrderReadModel;
```

```javascript
// projections/OrderProjection.js
const OrderReadModel = require('../readModels/OrderReadModel');

// Projection: สร้างและอัพเดท read model จาก events
class OrderProjection {
  async handle(event) {
    const handler = `on${event.eventType}`;
    
    if (typeof this[handler] === 'function') {
      await this[handler](event);
    }
  }

  async onOrderCreated(event) {
    await OrderReadModel.create({
      orderId: event.aggregateId,
      userId: event.data.userId,
      items: event.data.items,
      total: event.data.total,
      shippingAddress: event.data.shippingAddress,
      status: 'pending',
      createdAt: event.metadata.timestamp,
      version: event.version
    });
  }

  async onPaymentConfirmed(event) {
    await OrderReadModel.findOneAndUpdate(
      { orderId: event.aggregateId },
      {
        status: 'paid',
        paymentId: event.data.paymentId,
        paidAt: event.metadata.timestamp,
        version: event.version
      }
    );
  }

  async onOrderShipped(event) {
    await OrderReadModel.findOneAndUpdate(
      { orderId: event.aggregateId },
      {
        status: 'shipped',
        trackingNumber: event.data.trackingNumber,
        carrier: event.data.carrier,
        shippedAt: event.metadata.timestamp,
        version: event.version
      }
    );
  }

  async onOrderDelivered(event) {
    await OrderReadModel.findOneAndUpdate(
      { orderId: event.aggregateId },
      {
        status: 'delivered',
        deliveredAt: event.data.deliveredAt,
        version: event.version
      }
    );
  }

  async onOrderCancelled(event) {
    await OrderReadModel.findOneAndUpdate(
      { orderId: event.aggregateId },
      {
        status: 'cancelled',
        cancellationReason: event.data.reason,
        cancelledAt: event.metadata.timestamp,
        version: event.version
      }
    );
  }
}

module.exports = new OrderProjection();
```

---

## ขั้นตอนที่ 949: Event Bus และ Projection Runner

```javascript
// infrastructure/eventBus.js
const EventEmitter = require('events');
const Event = require('../models/EventStore');

class EventBus extends EventEmitter {
  constructor() {
    super();
    this.projections = [];
    this.eventHandlers = new Map();
    this.lastPosition = 0;
  }

  // ลงทะเบียน projection
  registerProjection(projection) {
    this.projections.push(projection);
  }

  // ลงทะเบียน event handler
  on(eventType, handler) {
    if (!this.eventHandlers.has(eventType)) {
      this.eventHandlers.set(eventType, []);
    }
    this.eventHandlers.get(eventType).push(handler);
    return this;
  }

  // Publish event ไปยัง handlers
  async publish(event) {
    const handlers = this.eventHandlers.get(event.eventType) || [];
    
    // รัน projections และ handlers แบบ parallel
    await Promise.all([
      ...this.projections.map(proj => proj.handle(event)),
      ...handlers.map(h => h(event))
    ]);
    
    this.emit(event.eventType, event);
  }

  // Process events จาก event store
  async processEvents(fromPosition = 0) {
    const events = await Event.find({
      position: { $gt: fromPosition }
    })
    .sort({ position: 1 })
    .limit(100)
    .lean();
    
    for (const event of events) {
      await this.publish(event);
      this.lastPosition = event.position;
    }
    
    return events.length;
  }

  // Start continuous processing
  async startProcessing(intervalMs = 1000) {
    const processLoop = async () => {
      try {
        await this.processEvents(this.lastPosition);
      } catch (error) {
        console.error('Event processing error:', error);
      }
    };
    
    setInterval(processLoop, intervalMs);
  }
}

module.exports = new EventBus();
```

---

## ขั้นตอนที่ 950: Snapshot Strategy

```javascript
// services/snapshotService.js
const mongoose = require('mongoose');

const snapshotSchema = new mongoose.Schema({
  aggregateId: { type: String, required: true, index: true },
  aggregateType: { type: String, required: true },
  version: { type: Number, required: true },
  state: { type: mongoose.Schema.Types.Mixed, required: true },
  createdAt: { type: Date, default: Date.now }
}, {
  collection: 'snapshots'
});

snapshotSchema.index({ aggregateId: 1, version: -1 });

const Snapshot = mongoose.model('Snapshot', snapshotSchema);

class SnapshotService {
  constructor(snapshotFrequency = 10) {
    this.snapshotFrequency = snapshotFrequency;
  }

  // ตรวจสอบว่าควร create snapshot หรือไม่
  shouldCreateSnapshot(version) {
    return version % this.snapshotFrequency === 0;
  }

  // สร้าง snapshot
  async createSnapshot(aggregateId, aggregateType, version, state) {
    await Snapshot.findOneAndUpdate(
      { aggregateId, version },
      {
        aggregateId,
        aggregateType,
        version,
        state,
        createdAt: new Date()
      },
      { upsert: true }
    );
  }

  // ดึง latest snapshot
  async getLatestSnapshot(aggregateId) {
    return Snapshot.findOne({ aggregateId })
      .sort({ version: -1 })
      .lean();
  }
}

// Repository ที่ใช้ snapshot
class SnapshotOrderRepository {
  constructor(eventStore, snapshotService) {
    this.eventStore = eventStore;
    this.snapshotService = snapshotService;
  }

  async findById(orderId) {
    // ลอง load จาก snapshot ก่อน
    const snapshot = await this.snapshotService.getLatestSnapshot(orderId);
    
    let order;
    let fromVersion = 0;
    
    if (snapshot) {
      // Rebuild จาก snapshot
      order = Object.assign(new Order(), snapshot.state);
      fromVersion = snapshot.version;
    } else {
      order = new Order();
    }
    
    // โหลด events หลัง snapshot
    const events = await this.eventStore.getEvents(orderId, fromVersion);
    
    for (const event of events) {
      order.apply({
        type: event.eventType,
        data: event.data,
        isNew: false
      });
    }
    
    // สร้าง snapshot ถ้าถึงเวลา
    if (this.snapshotService.shouldCreateSnapshot(order._version)) {
      await this.snapshotService.createSnapshot(
        orderId,
        'Order',
        order._version,
        { ...order }
      );
    }
    
    return order;
  }
}

module.exports = { SnapshotService, SnapshotOrderRepository };
```

---

## ขั้นตอนที่ 951: Event Sourcing Query Service

```javascript
// services/queryService.js
const OrderReadModel = require('../readModels/OrderReadModel');
const Event = require('../models/EventStore');

class OrderQueryService {
  // ดู orders ของ user
  async getUserOrders(userId, options = {}) {
    const { page = 1, limit = 20, status } = options;
    
    const query = { userId };
    if (status) query.status = status;
    
    const [orders, total] = await Promise.all([
      OrderReadModel.find(query)
        .sort({ createdAt: -1 })
        .skip((page - 1) * limit)
        .limit(limit)
        .lean(),
      OrderReadModel.countDocuments(query)
    ]);
    
    return { orders, total, page, limit };
  }

  // ดู order details
  async getOrder(orderId) {
    return OrderReadModel.findOne({ orderId }).lean();
  }

  // ดู order history (timeline of events)
  async getOrderHistory(orderId) {
    return Event.find({ aggregateId: orderId })
      .sort({ version: 1 })
      .select('eventType data metadata version')
      .lean();
  }

  // ย้อนดูสถานะ order ณ เวลาที่กำหนด
  async getOrderAtTime(orderId, timestamp) {
    const events = await Event.find({
      aggregateId: orderId,
      'metadata.timestamp': { $lte: timestamp }
    })
    .sort({ version: 1 })
    .lean();
    
    if (events.length === 0) return null;
    
    const Order = require('../domain/order/Order');
    return Order.rebuild(events.map(e => ({
      type: e.eventType,
      data: e.data
    })));
  }

  // สถิติ orders
  async getOrderStats(options = {}) {
    const { fromDate, toDate } = options;
    
    const query = {};
    if (fromDate || toDate) {
      query.createdAt = {};
      if (fromDate) query.createdAt.$gte = fromDate;
      if (toDate) query.createdAt.$lte = toDate;
    }
    
    return OrderReadModel.aggregate([
      { $match: query },
      {
        $group: {
          _id: '$status',
          count: { $sum: 1 },
          totalRevenue: { $sum: '$total' }
        }
      }
    ]);
  }
}

module.exports = new OrderQueryService();
```

---

## ขั้นตอนที่ 952: Event Replay

```javascript
// services/replayService.js
const Event = require('../models/EventStore');
const EventBus = require('../infrastructure/eventBus');

class ReplayService {
  // Replay events ทั้งหมดตั้งแต่ต้น
  async replayAll(projection) {
    console.log('Starting full replay...');
    
    // ล้าง read model ก่อน
    if (projection.reset) {
      await projection.reset();
    }
    
    let processed = 0;
    let position = 0;
    const batchSize = 100;
    
    while (true) {
      const events = await Event.find({
        position: { $gt: position }
      })
      .sort({ position: 1 })
      .limit(batchSize)
      .lean();
      
      if (events.length === 0) break;
      
      for (const event of events) {
        await projection.handle(event);
        position = event.position;
        processed++;
      }
      
      console.log(`Replayed ${processed} events...`);
    }
    
    console.log(`Replay complete: ${processed} events processed`);
    return processed;
  }

  // Replay สำหรับ aggregate เฉพาะ
  async replayAggregate(aggregateId, projection) {
    const events = await Event.find({ aggregateId })
      .sort({ version: 1 })
      .lean();
    
    for (const event of events) {
      await projection.handle(event);
    }
    
    return events.length;
  }

  // Replay ช่วงเวลาที่กำหนด
  async replayTimeRange(fromDate, toDate, projection) {
    const events = await Event.find({
      'metadata.timestamp': {
        $gte: fromDate,
        $lte: toDate
      }
    })
    .sort({ position: 1 })
    .lean();
    
    for (const event of events) {
      await projection.handle(event);
    }
    
    return events.length;
  }
}

module.exports = new ReplayService();
```

---

## ขั้นตอนที่ 953: API Routes สำหรับ Event Sourcing

```javascript
// routes/orders.js
const express = require('express');
const router = express.Router();
const { CreateOrderCommand, CreateOrderHandler } = require('../cqrs/commands/CreateOrderCommand');
const OrderQueryService = require('../services/queryService');
const EventStore = require('../services/eventStore');

const createOrderHandler = new CreateOrderHandler();

// Commands (Write side)
router.post('/', async (req, res) => {
  try {
    const command = new CreateOrderCommand({
      userId: req.user.id,
      items: req.body.items,
      shippingAddress: req.body.shippingAddress
    });
    
    const result = await createOrderHandler.handle(command);
    
    res.status(201).json({
      success: true,
      data: result
    });
  } catch (error) {
    res.status(400).json({ success: false, error: error.message });
  }
});

router.post('/:id/confirm-payment', async (req, res) => {
  try {
    const OrderRepository = require('../repositories/OrderRepository');
    const order = await OrderRepository.findById(req.params.id);
    
    if (!order) {
      return res.status(404).json({ success: false, error: 'Order not found' });
    }
    
    order.confirmPayment(req.body);
    await OrderRepository.save(order);
    
    res.json({ success: true, message: 'Payment confirmed' });
  } catch (error) {
    res.status(400).json({ success: false, error: error.message });
  }
});

// Queries (Read side)
router.get('/', async (req, res) => {
  try {
    const result = await OrderQueryService.getUserOrders(req.user.id, req.query);
    res.json({ success: true, ...result });
  } catch (error) {
    res.status(500).json({ success: false, error: error.message });
  }
});

router.get('/:id', async (req, res) => {
  try {
    const order = await OrderQueryService.getOrder(req.params.id);
    
    if (!order) {
      return res.status(404).json({ success: false, error: 'Order not found' });
    }
    
    res.json({ success: true, data: order });
  } catch (error) {
    res.status(500).json({ success: false, error: error.message });
  }
});

// Order history (timeline)
router.get('/:id/history', async (req, res) => {
  try {
    const history = await OrderQueryService.getOrderHistory(req.params.id);
    res.json({ success: true, data: history });
  } catch (error) {
    res.status(500).json({ success: false, error: error.message });
  }
});

// Time travel - ดูสถานะ order ณ เวลาที่กำหนด
router.get('/:id/at', async (req, res) => {
  try {
    const { timestamp } = req.query;
    
    if (!timestamp) {
      return res.status(400).json({ error: 'timestamp parameter required' });
    }
    
    const order = await OrderQueryService.getOrderAtTime(
      req.params.id,
      new Date(timestamp)
    );
    
    if (!order) {
      return res.status(404).json({ error: 'Order not found at that time' });
    }
    
    res.json({ success: true, data: order, timestamp });
  } catch (error) {
    res.status(500).json({ success: false, error: error.message });
  }
});

module.exports = router;
```

---

## ขั้นตอนที่ 954: Event Store with PostgreSQL

```javascript
// config/postgres-eventStore.js
const { Pool } = require('pg');

const pool = new Pool({
  connectionString: process.env.POSTGRES_URI
});

// สร้าง event store table
const createEventStoreTable = async () => {
  await pool.query(`
    CREATE TABLE IF NOT EXISTS events (
      event_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
      aggregate_id VARCHAR(255) NOT NULL,
      aggregate_type VARCHAR(100) NOT NULL,
      event_type VARCHAR(100) NOT NULL,
      version INTEGER NOT NULL,
      data JSONB NOT NULL,
      metadata JSONB DEFAULT '{}',
      position BIGSERIAL,
      created_at TIMESTAMP DEFAULT NOW(),
      UNIQUE(aggregate_id, version)
    );
    
    CREATE INDEX IF NOT EXISTS idx_events_aggregate ON events(aggregate_id, version);
    CREATE INDEX IF NOT EXISTS idx_events_type ON events(event_type);
    CREATE INDEX IF NOT EXISTS idx_events_position ON events(position);
  `);
};

class PostgresEventStore {
  async append(aggregateId, aggregateType, events, expectedVersion = null) {
    const client = await pool.connect();
    
    try {
      await client.query('BEGIN');
      
      // ตรวจสอบ version
      if (expectedVersion !== null) {
        const { rows } = await client.query(
          'SELECT MAX(version) as max_version FROM events WHERE aggregate_id = $1',
          [aggregateId]
        );
        
        const currentVersion = rows[0].max_version || 0;
        
        if (currentVersion !== expectedVersion) {
          throw new Error('Concurrency conflict');
        }
      }
      
      // ดึง current version
      const { rows: versionRows } = await client.query(
        'SELECT MAX(version) as max_version FROM events WHERE aggregate_id = $1',
        [aggregateId]
      );
      
      let version = versionRows[0].max_version || 0;
      
      // Insert events
      for (const event of events) {
        await client.query(
          `INSERT INTO events (aggregate_id, aggregate_type, event_type, version, data, metadata)
           VALUES ($1, $2, $3, $4, $5, $6)`,
          [
            aggregateId,
            aggregateType,
            event.type,
            ++version,
            JSON.stringify(event.data),
            JSON.stringify(event.metadata || {})
          ]
        );
      }
      
      await client.query('COMMIT');
    } catch (error) {
      await client.query('ROLLBACK');
      throw error;
    } finally {
      client.release();
    }
  }

  async getEvents(aggregateId, fromVersion = 0) {
    const { rows } = await pool.query(
      `SELECT * FROM events 
       WHERE aggregate_id = $1 AND version > $2 
       ORDER BY version ASC`,
      [aggregateId, fromVersion]
    );
    
    return rows.map(row => ({
      ...row,
      data: row.data,
      metadata: row.metadata
    }));
  }
}

module.exports = { PostgresEventStore, createEventStoreTable };
```

---

## ขั้นตอนที่ 955: Integration Testing สำหรับ Event Sourcing

```javascript
// tests/eventSourcing.test.js
const mongoose = require('mongoose');
const Event = require('../models/EventStore');
const OrderRepository = require('../repositories/OrderRepository');
const Order = require('../domain/order/Order');

describe('Event Sourcing Tests', () => {
  beforeEach(async () => {
    await Event.deleteMany({});
  });

  describe('Order Aggregate', () => {
    it('should create order and store events', async () => {
      const orderId = require('crypto').randomUUID();
      
      const order = new Order();
      order._id = orderId;
      order.create({
        userId: 'user123',
        items: [{ productId: 'prod1', name: 'Product 1', price: 100, quantity: 2 }],
        total: 200,
        shippingAddress: { city: 'Bangkok' }
      });
      
      await OrderRepository.save(order);
      
      // ตรวจสอบว่า events ถูกบันทึก
      const events = await Event.find({ aggregateId: orderId });
      expect(events).toHaveLength(1);
      expect(events[0].eventType).toBe('OrderCreated');
      expect(events[0].data.total).toBe(200);
    });

    it('should rebuild order from events', async () => {
      const orderId = require('crypto').randomUUID();
      
      // สร้าง order
      const order = new Order();
      order._id = orderId;
      order.create({
        userId: 'user123',
        items: [{ productId: 'prod1', price: 100, quantity: 2 }],
        total: 200,
        shippingAddress: {}
      });
      
      await OrderRepository.save(order);
      
      // โหลด order ใหม่จาก events
      const loadedOrder = await OrderRepository.findById(orderId);
      
      expect(loadedOrder).toBeDefined();
      expect(loadedOrder.status).toBe('pending');
      expect(loadedOrder.total).toBe(200);
    });

    it('should maintain event version integrity', async () => {
      const orderId = require('crypto').randomUUID();
      
      const order = new Order();
      order._id = orderId;
      order.create({
        userId: 'user123',
        items: [],
        total: 500,
        shippingAddress: {}
      });
      
      await OrderRepository.save(order);
      order.confirmPayment({ paymentId: 'pay123', amount: 500 });
      await OrderRepository.save(order);
      
      const events = await Event.find({ aggregateId: orderId }).sort({ version: 1 });
      expect(events).toHaveLength(2);
      expect(events[0].version).toBe(1);
      expect(events[1].version).toBe(2);
    });

    it('should time travel to previous state', async () => {
      const OrderQueryService = require('../services/queryService');
      const orderId = require('crypto').randomUUID();
      
      const order = new Order();
      order._id = orderId;
      order.create({
        userId: 'user123',
        items: [],
        total: 500,
        shippingAddress: {}
      });
      
      await OrderRepository.save(order);
      
      const createdAt = new Date();
      
      // รอ 100ms
      await new Promise(resolve => setTimeout(resolve, 100));
      
      const loadedOrder = await OrderRepository.findById(orderId);
      loadedOrder.confirmPayment({ paymentId: 'pay123', amount: 500 });
      await OrderRepository.save(loadedOrder);
      
      // ดูสถานะก่อน payment
      const pastOrder = await OrderQueryService.getOrderAtTime(orderId, createdAt);
      expect(pastOrder.status).toBe('pending');
      
      // ดูสถานะปัจจุบัน
      const currentOrder = await OrderRepository.findById(orderId);
      expect(currentOrder.status).toBe('paid');
    });
  });
});
```

---

## ขั้นตอนที่ 956: Event Sourcing Patterns สำหรับ Real World

```javascript
// patterns/sagaOrchestrator.js

class OrderSaga {
  constructor(eventBus, services) {
    this.eventBus = eventBus;
    this.inventoryService = services.inventory;
    this.paymentService = services.payment;
    this.notificationService = services.notification;
    
    this.setupHandlers();
  }

  setupHandlers() {
    this.eventBus.on('OrderCreated', this.onOrderCreated.bind(this));
    this.eventBus.on('PaymentConfirmed', this.onPaymentConfirmed.bind(this));
    this.eventBus.on('OrderShipped', this.onOrderShipped.bind(this));
    this.eventBus.on('OrderCancelled', this.onOrderCancelled.bind(this));
  }

  async onOrderCreated(event) {
    const { orderId, userId, items } = event.data;
    
    try {
      // Reserve inventory
      await this.inventoryService.reserveItems(orderId, items);
      
      // ส่ง notification
      await this.notificationService.sendOrderConfirmation(userId, orderId);
      
      console.log(`Order ${orderId}: inventory reserved`);
    } catch (error) {
      // Compensating transaction
      await this.cancelOrder(orderId, error.message);
    }
  }

  async onPaymentConfirmed(event) {
    const { orderId } = event.data;
    
    // Confirm inventory deduction
    await this.inventoryService.confirmReservation(orderId);
    
    // Initiate fulfillment
    await this.fulfillmentService.createPickList(orderId);
  }

  async onOrderShipped(event) {
    const { orderId, userId, trackingNumber } = event.data;
    
    // ส่ง shipping notification
    await this.notificationService.sendShippingNotification(
      userId,
      orderId,
      trackingNumber
    );
  }

  async onOrderCancelled(event) {
    const { orderId, userId } = event.data;
    
    // Release inventory
    await this.inventoryService.releaseReservation(orderId);
    
    // Process refund ถ้า payment ได้รับแล้ว
    const payment = await this.paymentService.getPaymentByOrderId(orderId);
    if (payment && payment.status === 'completed') {
      await this.paymentService.refund(payment.id);
    }
    
    // ส่ง notification
    await this.notificationService.sendCancellationNotification(userId, orderId);
  }

  async cancelOrder(orderId, reason) {
    const OrderRepository = require('../repositories/OrderRepository');
    const order = await OrderRepository.findById(orderId);
    
    if (order && order.status === 'pending') {
      order.cancel({ reason, cancelledBy: 'system' });
      await OrderRepository.save(order);
    }
  }
}

module.exports = OrderSaga;
```

---

## ขั้นตอนที่ 957: Event Versioning

```javascript
// utils/eventVersioning.js

// Event upcasters สำหรับ migrate เวอร์ชันเก่า
const eventUpcasters = {
  'OrderCreated': {
    1: (eventData) => {
      // V1 -> V2: เพิ่ม shippingAddress field
      return {
        ...eventData,
        shippingAddress: eventData.address || {}
      };
    },
    2: (eventData) => {
      // V2 -> V3: แยก items เป็น orderItems
      return {
        ...eventData,
        orderItems: eventData.items || []
      };
    }
  }
};

const currentEventVersions = {
  'OrderCreated': 3,
  'PaymentConfirmed': 1,
  'OrderShipped': 1
};

// Upcast event ให้เป็นเวอร์ชันล่าสุด
const upcastEvent = (eventType, data, fromVersion = 1) => {
  const upcasters = eventUpcasters[eventType];
  
  if (!upcasters) return data;
  
  let currentData = data;
  const targetVersion = currentEventVersions[eventType] || 1;
  
  for (let v = fromVersion; v < targetVersion; v++) {
    if (upcasters[v]) {
      currentData = upcasters[v](currentData);
    }
  }
  
  return currentData;
};

module.exports = { upcastEvent, currentEventVersions };
```

---

## ขั้นตอนที่ 958: Event Sourcing Admin API

```javascript
// routes/admin/eventStore.js
const express = require('express');
const router = express.Router();
const Event = require('../../models/EventStore');
const ReplayService = require('../../services/replayService');
const OrderProjection = require('../../projections/OrderProjection');

router.use(require('../../middleware/auth').requireAdmin);

// ดู events ของ aggregate
router.get('/aggregates/:id/events', async (req, res) => {
  try {
    const events = await Event.find({ aggregateId: req.params.id })
      .sort({ version: 1 })
      .lean();
    
    res.json({ success: true, data: events });
  } catch (error) {
    res.status(500).json({ success: false, error: error.message });
  }
});

// Replay events สำหรับ aggregate
router.post('/aggregates/:id/replay', async (req, res) => {
  try {
    const count = await ReplayService.replayAggregate(
      req.params.id,
      OrderProjection
    );
    
    res.json({ success: true, eventsReplayed: count });
  } catch (error) {
    res.status(500).json({ success: false, error: error.message });
  }
});

// Full replay
router.post('/replay/full', async (req, res) => {
  try {
    const count = await ReplayService.replayAll(OrderProjection);
    res.json({ success: true, eventsReplayed: count });
  } catch (error) {
    res.status(500).json({ success: false, error: error.message });
  }
});

// Event stats
router.get('/stats', async (req, res) => {
  try {
    const stats = await Event.aggregate([
      {
        $group: {
          _id: '$eventType',
          count: { $sum: 1 },
          latestAt: { $max: '$metadata.timestamp' }
        }
      },
      { $sort: { count: -1 } }
    ]);
    
    const totalEvents = stats.reduce((sum, s) => sum + s.count, 0);
    
    res.json({
      success: true,
      data: {
        total: totalEvents,
        byType: stats
      }
    });
  } catch (error) {
    res.status(500).json({ success: false, error: error.message });
  }
});

module.exports = router;
```

---

## ขั้นตอนที่ 959: Event Store Monitoring

```javascript
// monitoring/eventStoreMonitor.js
const Event = require('../models/EventStore');
const redis = require('../config/redis');

class EventStoreMonitor {
  async getMetrics() {
    const [
      totalEvents,
      recentEvents,
      eventsByType,
      laggedAggregates
    ] = await Promise.all([
      Event.countDocuments(),
      Event.countDocuments({
        'metadata.timestamp': { $gte: new Date(Date.now() - 3600000) }
      }),
      Event.aggregate([
        { $group: { _id: '$eventType', count: { $sum: 1 } } },
        { $sort: { count: -1 } },
        { $limit: 10 }
      ]),
      this.findLaggedAggregates()
    ]);
    
    return {
      totalEvents,
      eventsLastHour: recentEvents,
      topEventTypes: eventsByType,
      laggedAggregates
    };
  }

  async findLaggedAggregates() {
    // หา aggregates ที่ projection ล้าหลัง
    const lastProjectionPosition = parseInt(
      await redis.get('projection:last_position') || '0'
    );
    
    const lastEventPosition = await Event.findOne()
      .sort({ position: -1 })
      .select('position');
    
    const lag = lastEventPosition 
      ? lastEventPosition.position - lastProjectionPosition 
      : 0;
    
    return { lag, lastProjectionPosition };
  }
}

module.exports = new EventStoreMonitor();
```

---

## ขั้นตอนที่ 960: สรุป Event Sourcing Patterns

```javascript
// patterns/summary.js

/*
  Event Sourcing Key Concepts:
  
  1. Events เป็น source of truth - ไม่ใช่ current state
  2. Events เป็น immutable - ไม่แก้ไขหรือลบ
  3. Aggregates ใช้ events สร้าง state
  4. Projections สร้าง read models จาก events
  5. CQRS แยก write (commands) จาก read (queries)
  
  เมื่อไรควรใช้ Event Sourcing:
  ✓ ต้องการ audit trail สมบูรณ์
  ✓ ต้องการ time travel debugging
  ✓ Domain มีความซับซ้อนสูง
  ✓ ต้องการ event-driven integrations
  
  เมื่อไรไม่ควรใช้:
  ✗ Simple CRUD applications
  ✗ ไม่ต้องการ history
  ✗ ทีมไม่คุ้นเคย patterns
*/

// Simple Event Sourcing setup
const setupEventSourcing = async (app) => {
  const EventBus = require('./infrastructure/eventBus');
  const OrderProjection = require('./projections/OrderProjection');
  const OrderSaga = require('./patterns/sagaOrchestrator');
  const mongoose = require('mongoose');
  
  // สร้าง indexes
  await mongoose.model('Event').createIndexes();
  
  // ลงทะเบียน projections
  EventBus.registerProjection(OrderProjection);
  
  // เริ่ม event processing
  await EventBus.startProcessing(1000);
  
  // ลงทะเบียน saga
  const saga = new OrderSaga(EventBus, {
    inventory: require('./services/inventoryService'),
    payment: require('./services/paymentService'),
    notification: require('./services/notificationService')
  });
  
  console.log('Event sourcing initialized');
};

module.exports = setupEventSourcing;
```

---

## แบบฝึกหัดท้ายบท

### แบบฝึกหัดที่ 1: Product Aggregate
สร้าง Product aggregate ด้วย events: ProductCreated, PriceUpdated, StockUpdated, ProductDeactivated

### แบบฝึกหัดที่ 2: User Registration Saga
สร้าง saga ที่จัดการ user registration flow รวมถึง email verification และ welcome email

### แบบฝึกหัดที่ 3: Time Travel Feature
สร้าง UI ที่แสดง timeline ของ order events พร้อม ability ให้ดู state ณ เวลาต่างๆ

### แบบฝึกหัดที่ 4: Event Migration
สร้าง upcaster สำหรับ migrate event format เก่าไปยังรูปแบบใหม่

### แบบฝึกหัดที่ 5: Projection Rebuild
สร้าง mechanism ที่ rebuild read models ทั้งหมดจาก events โดยไม่ downtime

---

## สรุป

Event Sourcing เป็น powerful pattern สำหรับ domains ที่ต้องการ audit trail สมบูรณ์และความสามารถในการ debug ที่ดี เมื่อรวมกับ CQRS จะทำให้ระบบมีความยืดหยุ่นและ scalable สูงมาก
