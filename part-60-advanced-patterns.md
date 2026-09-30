# Part 60 | ขั้นตอนที่ 1061-1080 จาก 1000

## Advanced Patterns สำหรับ Node.js Systems

บทสุดท้ายรวม advanced patterns ที่ใช้ในระบบขนาดใหญ่ระดับ production

---

## ขั้นตอนที่ 1061: Saga Pattern

Saga เป็น pattern สำหรับจัดการ distributed transactions โดยแบ่งเป็น local transactions และมี compensating transactions เมื่อเกิดความล้มเหลว

```javascript
// patterns/saga/OrderSaga.js

class SagaStep {
  constructor({ name, execute, compensate }) {
    this.name = name;
    this.execute = execute;
    this.compensate = compensate;
  }
}

class Saga {
  constructor(steps) {
    this.steps = steps;
    this.executedSteps = [];
  }

  async run(context) {
    for (const step of this.steps) {
      try {
        console.log(`Executing step: ${step.name}`);
        const result = await step.execute(context);
        context = { ...context, ...result };
        this.executedSteps.push({ step, result });
      } catch (error) {
        console.error(`Step ${step.name} failed:`, error.message);
        await this._compensate(context);
        throw new SagaError(step.name, error);
      }
    }
    
    return context;
  }

  async _compensate(context) {
    console.log('Starting compensation...');
    
    // ย้อนกลับใน reverse order
    for (const { step, result } of [...this.executedSteps].reverse()) {
      try {
        console.log(`Compensating: ${step.name}`);
        await step.compensate({ ...context, ...result });
      } catch (error) {
        console.error(`Compensation failed for ${step.name}:`, error.message);
        // Log แต่ continue compensating
      }
    }
  }
}

class SagaError extends Error {
  constructor(stepName, cause) {
    super(`Saga failed at step: ${stepName}`);
    this.stepName = stepName;
    this.cause = cause;
  }
}

// Order Processing Saga
class OrderProcessingSaga {
  constructor({ orderService, inventoryService, paymentService, shippingService }) {
    this.orderService = orderService;
    this.inventoryService = inventoryService;
    this.paymentService = paymentService;
    this.shippingService = shippingService;
  }

  async process(orderData) {
    const saga = new Saga([
      new SagaStep({
        name: 'CreateOrder',
        execute: async (ctx) => {
          const order = await this.orderService.create(orderData);
          return { orderId: order.id };
        },
        compensate: async (ctx) => {
          await this.orderService.cancel(ctx.orderId, 'saga_compensation');
        }
      }),
      
      new SagaStep({
        name: 'ReserveInventory',
        execute: async (ctx) => {
          const reservation = await this.inventoryService.reserve(
            orderData.items,
            ctx.orderId
          );
          return { reservationId: reservation.id };
        },
        compensate: async (ctx) => {
          await this.inventoryService.release(ctx.reservationId);
        }
      }),
      
      new SagaStep({
        name: 'ProcessPayment',
        execute: async (ctx) => {
          const payment = await this.paymentService.charge(
            orderData.paymentMethod,
            orderData.total
          );
          return { paymentId: payment.id, transactionId: payment.transactionId };
        },
        compensate: async (ctx) => {
          await this.paymentService.refund(ctx.paymentId);
        }
      }),
      
      new SagaStep({
        name: 'CreateShipment',
        execute: async (ctx) => {
          const shipment = await this.shippingService.create(ctx.orderId);
          return { shipmentId: shipment.id };
        },
        compensate: async (ctx) => {
          await this.shippingService.cancel(ctx.shipmentId);
        }
      })
    ]);
    
    return saga.run({});
  }
}

module.exports = { Saga, SagaStep, SagaError, OrderProcessingSaga };
```

---

## ขั้นตอนที่ 1062: Outbox Pattern

```javascript
// patterns/outbox/OutboxPublisher.js
// Transactional Outbox: เขียน event พร้อม DB transaction

const mongoose = require('mongoose');

const OutboxSchema = new mongoose.Schema({
  eventType: { type: String, required: true },
  payload: { type: mongoose.Schema.Types.Mixed, required: true },
  status: { type: String, enum: ['pending', 'published', 'failed'], default: 'pending' },
  retries: { type: Number, default: 0 },
  lastError: String,
  publishedAt: Date
}, { timestamps: true });

OutboxSchema.index({ status: 1, createdAt: 1 });

const OutboxModel = mongoose.model('Outbox', OutboxSchema);

class OutboxService {
  // เขียน event ใน transaction เดียวกับ business data
  static async writeEvent(session, eventType, payload) {
    const outbox = new OutboxModel({ eventType, payload });
    await outbox.save({ session });
    return outbox;
  }

  // Relay: ดึง pending events และส่งไปยัง message broker
  static async relayPendingEvents(publisher, batchSize = 10) {
    const events = await OutboxModel.find({ status: 'pending' })
      .sort({ createdAt: 1 })
      .limit(batchSize)
      .lean();
    
    for (const event of events) {
      try {
        await publisher.publish(event.eventType, event.payload);
        
        await OutboxModel.findByIdAndUpdate(event._id, {
          status: 'published',
          publishedAt: new Date()
        });
        
        console.log(`Published event: ${event.eventType}`);
      } catch (error) {
        await OutboxModel.findByIdAndUpdate(event._id, {
          $inc: { retries: 1 },
          lastError: error.message,
          status: event.retries >= 3 ? 'failed' : 'pending'
        });
      }
    }
    
    return events.length;
  }
}

// ใช้งานใน business code
class OrderService {
  async createOrder(orderData) {
    const session = await mongoose.startSession();
    session.startTransaction();
    
    try {
      const order = await OrderModel.create([orderData], { session });
      
      // เขียน event ใน transaction เดียวกัน
      await OutboxService.writeEvent(session, 'OrderCreated', {
        orderId: order[0]._id,
        customerId: order[0].customerId,
        total: order[0].total
      });
      
      await session.commitTransaction();
      return order[0];
    } catch (error) {
      await session.abortTransaction();
      throw error;
    } finally {
      session.endSession();
    }
  }
}

// Background worker: relay events ทุก 5 วินาที
class OutboxRelay {
  constructor(publisher) {
    this.publisher = publisher;
    this.isRunning = false;
  }

  start(intervalMs = 5000) {
    this.isRunning = true;
    this._scheduleNext(intervalMs);
  }

  _scheduleNext(intervalMs) {
    if (!this.isRunning) return;
    
    setTimeout(async () => {
      try {
        const count = await OutboxService.relayPendingEvents(this.publisher);
        if (count > 0) console.log(`Relayed ${count} events`);
      } catch (error) {
        console.error('Relay error:', error);
      }
      
      this._scheduleNext(intervalMs);
    }, intervalMs);
  }

  stop() {
    this.isRunning = false;
  }
}

module.exports = { OutboxService, OutboxRelay, OrderService };
```

---

## ขั้นตอนที่ 1063: Circuit Breaker Pattern (Advanced)

```javascript
// patterns/circuitBreaker/CircuitBreaker.js

const EventEmitter = require('events');

class CircuitBreaker extends EventEmitter {
  constructor(options = {}) {
    super();
    
    this.state = 'CLOSED';
    this.failureCount = 0;
    this.successCount = 0;
    this.lastFailureTime = null;
    this.nextAttemptTime = null;
    
    this.failureThreshold = options.failureThreshold || 5;
    this.successThreshold = options.successThreshold || 2;
    this.timeout = options.timeout || 60000;   // 1 minute
    this.halfOpenMaxCalls = options.halfOpenMaxCalls || 3;
    
    this._halfOpenCalls = 0;
  }

  async call(fn, ...args) {
    if (this.state === 'OPEN') {
      if (Date.now() < this.nextAttemptTime) {
        throw new CircuitBreakerOpenError(
          `Circuit is OPEN. Retry after ${new Date(this.nextAttemptTime).toISOString()}`
        );
      }
      
      // Transition to HALF_OPEN
      this._transitionTo('HALF_OPEN');
    }
    
    if (this.state === 'HALF_OPEN') {
      if (this._halfOpenCalls >= this.halfOpenMaxCalls) {
        throw new CircuitBreakerOpenError('Too many calls in HALF_OPEN state');
      }
      this._halfOpenCalls++;
    }
    
    try {
      const result = await fn(...args);
      this._onSuccess();
      return result;
    } catch (error) {
      this._onFailure(error);
      throw error;
    }
  }

  _onSuccess() {
    this.failureCount = 0;
    
    if (this.state === 'HALF_OPEN') {
      this.successCount++;
      
      if (this.successCount >= this.successThreshold) {
        this._transitionTo('CLOSED');
      }
    }
  }

  _onFailure(error) {
    this.lastFailureTime = Date.now();
    this.failureCount++;
    
    if (this.state === 'HALF_OPEN') {
      this._transitionTo('OPEN');
      return;
    }
    
    if (this.failureCount >= this.failureThreshold) {
      this._transitionTo('OPEN');
    }
  }

  _transitionTo(newState) {
    const oldState = this.state;
    this.state = newState;
    
    if (newState === 'OPEN') {
      this.nextAttemptTime = Date.now() + this.timeout;
      this._halfOpenCalls = 0;
    } else if (newState === 'HALF_OPEN') {
      this._halfOpenCalls = 0;
      this.successCount = 0;
    } else if (newState === 'CLOSED') {
      this.failureCount = 0;
      this.successCount = 0;
    }
    
    console.log(`Circuit Breaker: ${oldState} -> ${newState}`);
    this.emit('stateChange', { from: oldState, to: newState });
  }

  getStatus() {
    return {
      state: this.state,
      failureCount: this.failureCount,
      nextAttemptTime: this.nextAttemptTime
        ? new Date(this.nextAttemptTime)
        : null
    };
  }
}

class CircuitBreakerOpenError extends Error {
  constructor(message) {
    super(message);
    this.name = 'CircuitBreakerOpenError';
    this.statusCode = 503;
  }
}

// Circuit Breaker Registry
class CircuitBreakerRegistry {
  constructor() {
    this.breakers = new Map();
  }

  get(name, options) {
    if (!this.breakers.has(name)) {
      this.breakers.set(name, new CircuitBreaker(options));
    }
    return this.breakers.get(name);
  }

  getAll() {
    const status = {};
    for (const [name, breaker] of this.breakers) {
      status[name] = breaker.getStatus();
    }
    return status;
  }
}

const registry = new CircuitBreakerRegistry();

module.exports = { CircuitBreaker, CircuitBreakerOpenError, registry };
```

---

## ขั้นตอนที่ 1064: Bulkhead Pattern

```javascript
// patterns/bulkhead/Bulkhead.js
// แยก resource pools เพื่อป้องกัน cascade failures

class Bulkhead {
  constructor(options = {}) {
    this.maxConcurrent = options.maxConcurrent || 10;
    this.maxWaitQueue = options.maxWaitQueue || 100;
    this.timeout = options.timeout || 5000;
    
    this._activeCount = 0;
    this._waitQueue = [];
  }

  async execute(fn) {
    if (this._activeCount >= this.maxConcurrent) {
      if (this._waitQueue.length >= this.maxWaitQueue) {
        throw new BulkheadFullError('Bulkhead queue is full');
      }
      
      // รอใน queue
      await this._waitForSlot();
    }
    
    this._activeCount++;
    
    try {
      // Timeout สำหรับ execution
      return await Promise.race([
        fn(),
        new Promise((_, reject) =>
          setTimeout(() => reject(new Error('Execution timeout')), this.timeout)
        )
      ]);
    } finally {
      this._activeCount--;
      this._releaseNext();
    }
  }

  _waitForSlot() {
    return new Promise((resolve, reject) => {
      const timeoutId = setTimeout(() => {
        // ลบออกจาก queue เมื่อ timeout
        const index = this._waitQueue.findIndex(item => item.resolve === resolve);
        if (index >= 0) this._waitQueue.splice(index, 1);
        reject(new Error('Bulkhead wait timeout'));
      }, this.timeout);
      
      this._waitQueue.push({
        resolve: () => {
          clearTimeout(timeoutId);
          resolve();
        }
      });
    });
  }

  _releaseNext() {
    if (this._waitQueue.length > 0 && this._activeCount < this.maxConcurrent) {
      const next = this._waitQueue.shift();
      next.resolve();
    }
  }

  getStatus() {
    return {
      active: this._activeCount,
      queued: this._waitQueue.length,
      maxConcurrent: this.maxConcurrent,
      available: this.maxConcurrent - this._activeCount
    };
  }
}

class BulkheadFullError extends Error {
  constructor(message) {
    super(message);
    this.name = 'BulkheadFullError';
    this.statusCode = 503;
  }
}

// Service ที่ใช้ Bulkhead
class ExternalServiceClient {
  constructor(baseUrl) {
    this.baseUrl = baseUrl;
    
    // แยก bulkhead สำหรับแต่ละ operation
    this.criticalBulkhead = new Bulkhead({ maxConcurrent: 20, maxWaitQueue: 50 });
    this.normalBulkhead = new Bulkhead({ maxConcurrent: 10, maxWaitQueue: 100 });
    this.batchBulkhead = new Bulkhead({ maxConcurrent: 5, maxWaitQueue: 20 });
  }

  async getUser(id) {
    return this.criticalBulkhead.execute(() => this._fetchUser(id));
  }

  async createReport(data) {
    return this.batchBulkhead.execute(() => this._createReport(data));
  }

  async _fetchUser(id) {
    // HTTP call
  }

  async _createReport(data) {
    // Heavy operation
  }
}

module.exports = { Bulkhead, BulkheadFullError, ExternalServiceClient };
```

---

## ขั้นตอนที่ 1065: Retry Pattern

```javascript
// patterns/retry/Retry.js

class RetryConfig {
  constructor({
    maxAttempts = 3,
    baseDelay = 1000,
    maxDelay = 30000,
    backoffMultiplier = 2,
    retryOn = ['ECONNRESET', 'ETIMEDOUT', 'ECONNREFUSED'],
    jitter = true
  } = {}) {
    this.maxAttempts = maxAttempts;
    this.baseDelay = baseDelay;
    this.maxDelay = maxDelay;
    this.backoffMultiplier = backoffMultiplier;
    this.retryOn = retryOn;
    this.jitter = jitter;
  }

  calculateDelay(attempt) {
    const delay = Math.min(
      this.baseDelay * Math.pow(this.backoffMultiplier, attempt - 1),
      this.maxDelay
    );
    
    if (this.jitter) {
      // Full jitter: random ระหว่าง 0 ถึง delay
      return Math.random() * delay;
    }
    
    return delay;
  }

  shouldRetry(error) {
    if (this.retryOn.includes(error.code)) return true;
    if (error.statusCode && error.statusCode >= 500) return true;
    if (error.name === 'NetworkError') return true;
    return false;
  }
}

async function withRetry(fn, config = new RetryConfig()) {
  let lastError;
  
  for (let attempt = 1; attempt <= config.maxAttempts; attempt++) {
    try {
      return await fn();
    } catch (error) {
      lastError = error;
      
      if (!config.shouldRetry(error) || attempt === config.maxAttempts) {
        throw error;
      }
      
      const delay = config.calculateDelay(attempt);
      console.log(`Attempt ${attempt} failed. Retrying in ${Math.round(delay)}ms...`);
      
      await sleep(delay);
    }
  }
  
  throw lastError;
}

function sleep(ms) {
  return new Promise(resolve => setTimeout(resolve, ms));
}

// ใช้งาน
async function fetchUserWithRetry(userId) {
  return withRetry(
    () => httpClient.get(`/users/${userId}`),
    new RetryConfig({
      maxAttempts: 3,
      baseDelay: 500,
      retryOn: ['ECONNRESET', 'ETIMEDOUT']
    })
  );
}

module.exports = { RetryConfig, withRetry };
```

---

## ขั้นตอนที่ 1066: Cache-Aside with Stampede Prevention

```javascript
// patterns/cache/StampedePreventionCache.js
// ป้องกัน Cache Stampede โดยใช้ probabilistic early expiration

class StampedePreventionCache {
  constructor(cacheClient) {
    this.cache = cacheClient;
    this.recomputeThreshold = 0.1; // 10% chance of early recompute
  }

  async get(key, fetchFn, ttl, beta = 1) {
    const cached = await this.cache.get(key);
    
    if (cached) {
      const { value, expiresAt, computeDuration } = cached;
      
      // XFetch algorithm: probabilistic early expiration
      const earlyExpireTime = expiresAt - beta * computeDuration * Math.log(Math.random());
      
      if (Date.now() < earlyExpireTime) {
        return value;
      }
      
      // Early recompute (background)
      this._recompute(key, fetchFn, ttl).catch(console.error);
      return value;
    }
    
    return this._recompute(key, fetchFn, ttl);
  }

  async _recompute(key, fetchFn, ttl) {
    const start = Date.now();
    const value = await fetchFn();
    const computeDuration = Date.now() - start;
    
    await this.cache.set(key, {
      value,
      expiresAt: Date.now() + ttl * 1000,
      computeDuration
    }, ttl + 60); // Store extra 60 seconds for early expiration window
    
    return value;
  }
}

// Distributed Lock สำหรับ Cache Population
class DistributedLockCache {
  constructor(cacheClient, lockClient) {
    this.cache = cacheClient;
    this.lock = lockClient;
  }

  async getOrSet(key, fetchFn, ttl) {
    // ตรวจสอบ cache ก่อน
    const cached = await this.cache.get(key);
    if (cached !== null) return cached;
    
    const lockKey = `lock:${key}`;
    
    // พยายาม acquire lock
    const acquired = await this.lock.acquire(lockKey, 5000);
    
    if (!acquired) {
      // รอและ retry
      await sleep(100);
      return this.getOrSet(key, fetchFn, ttl);
    }
    
    try {
      // Double check หลัง acquire lock
      const doubleCheck = await this.cache.get(key);
      if (doubleCheck !== null) return doubleCheck;
      
      // Fetch and cache
      const value = await fetchFn();
      await this.cache.set(key, value, ttl);
      
      return value;
    } finally {
      await this.lock.release(lockKey);
    }
  }
}

function sleep(ms) {
  return new Promise(resolve => setTimeout(resolve, ms));
}

module.exports = { StampedePreventionCache, DistributedLockCache };
```

---

## ขั้นตอนที่ 1067: Event-Driven Architecture

```javascript
// patterns/eventDriven/EventBus.js

const EventEmitter = require('events');

class DomainEventBus {
  constructor() {
    this._emitter = new EventEmitter();
    this._emitter.setMaxListeners(100);
    this._handlers = new Map();
  }

  subscribe(eventType, handler, options = {}) {
    const { once = false, filter } = options;
    
    const wrappedHandler = async (event) => {
      // ตรวจสอบ filter ก่อน
      if (filter && !filter(event)) return;
      
      try {
        await handler(event);
      } catch (error) {
        console.error(`Handler error for ${eventType}:`, error);
        this._emitter.emit('handlerError', { eventType, error, event });
      }
    };
    
    if (once) {
      this._emitter.once(eventType, wrappedHandler);
    } else {
      this._emitter.on(eventType, wrappedHandler);
    }
    
    return () => this._emitter.off(eventType, wrappedHandler);
  }

  async publish(event) {
    if (!event.type) throw new Error('Event must have a type');
    
    const enriched = {
      ...event,
      id: require('crypto').randomUUID(),
      publishedAt: new Date().toISOString()
    };
    
    this._emitter.emit(event.type, enriched);
    this._emitter.emit('*', enriched); // Wildcard subscribers
  }

  onError(handler) {
    this._emitter.on('handlerError', handler);
  }
}

// Event Handlers
class OrderEventHandlers {
  constructor(emailService, inventoryService) {
    this.emailService = emailService;
    this.inventoryService = inventoryService;
  }

  register(eventBus) {
    eventBus.subscribe('OrderSubmitted', this.onOrderSubmitted.bind(this));
    eventBus.subscribe('OrderCancelled', this.onOrderCancelled.bind(this));
    eventBus.subscribe('PaymentFailed', this.onPaymentFailed.bind(this));
  }

  async onOrderSubmitted(event) {
    const { orderId, customerId, items } = event.data;
    
    // ส่ง confirmation email
    await this.emailService.sendOrderConfirmation(customerId, orderId);
    
    // Reserve inventory
    await this.inventoryService.reserveForOrder(orderId, items);
  }

  async onOrderCancelled(event) {
    const { orderId, customerId } = event.data;
    
    await this.emailService.sendCancellationNotice(customerId, orderId);
    await this.inventoryService.releaseForOrder(orderId);
  }

  async onPaymentFailed(event) {
    const { orderId, customerId, reason } = event.data;
    
    await this.emailService.sendPaymentFailureNotice(customerId, reason);
  }
}

const eventBus = new DomainEventBus();
module.exports = { DomainEventBus, OrderEventHandlers, eventBus };
```

---

## ขั้นตอนที่ 1068: Repository Pattern (Advanced)

```javascript
// patterns/repository/BaseRepository.js

class BaseRepository {
  constructor(Model) {
    this.Model = Model;
  }

  async findById(id) {
    const doc = await this.Model.findById(id).lean();
    return doc ? this._toDomain(doc) : null;
  }

  async findOne(criteria) {
    const doc = await this.Model.findOne(criteria).lean();
    return doc ? this._toDomain(doc) : null;
  }

  async findMany(criteria, options = {}) {
    const { sort, limit, skip, projection } = options;
    
    let query = this.Model.find(criteria);
    
    if (sort) query = query.sort(sort);
    if (limit) query = query.limit(limit);
    if (skip) query = query.skip(skip);
    if (projection) query = query.select(projection);
    
    const docs = await query.lean();
    return docs.map(doc => this._toDomain(doc));
  }

  async save(entity) {
    const data = this._toPersistence(entity);
    
    const doc = await this.Model.findOneAndUpdate(
      { _id: entity.id },
      { $set: data },
      { upsert: true, new: true }
    );
    
    return this._toDomain(doc.toObject());
  }

  async delete(id) {
    await this.Model.findByIdAndDelete(id);
  }

  async count(criteria = {}) {
    return this.Model.countDocuments(criteria);
  }

  // Override ใน subclasses
  _toDomain(doc) {
    throw new Error('_toDomain must be implemented');
  }

  _toPersistence(entity) {
    throw new Error('_toPersistence must be implemented');
  }
}

// Unit of Work Pattern
class UnitOfWork {
  constructor() {
    this.session = null;
    this._repositories = {};
  }

  async begin() {
    const mongoose = require('mongoose');
    this.session = await mongoose.startSession();
    this.session.startTransaction();
  }

  async commit() {
    if (this.session) {
      await this.session.commitTransaction();
      this.session.endSession();
      this.session = null;
    }
  }

  async rollback() {
    if (this.session) {
      await this.session.abortTransaction();
      this.session.endSession();
      this.session = null;
    }
  }

  getRepository(RepositoryClass) {
    const key = RepositoryClass.name;
    if (!this._repositories[key]) {
      this._repositories[key] = new RepositoryClass(this.session);
    }
    return this._repositories[key];
  }
}

module.exports = { BaseRepository, UnitOfWork };
```

---

## ขั้นตอนที่ 1069: Observer Pattern (Reactive)

```javascript
// patterns/reactive/Observable.js
// Simple reactive streams

class Observable {
  constructor(subscribeFn) {
    this._subscribeFn = subscribeFn;
  }

  subscribe(observer) {
    const wrappedObserver = {
      next: observer.next || (() => {}),
      error: observer.error || console.error,
      complete: observer.complete || (() => {})
    };
    
    const cleanup = this._subscribeFn(wrappedObserver);
    
    return {
      unsubscribe: cleanup || (() => {})
    };
  }

  pipe(...operators) {
    return operators.reduce((observable, operator) => operator(observable), this);
  }

  static create(subscribeFn) {
    return new Observable(subscribeFn);
  }

  static fromEventEmitter(emitter, eventName) {
    return new Observable(observer => {
      const handler = (data) => observer.next(data);
      emitter.on(eventName, handler);
      return () => emitter.off(eventName, handler);
    });
  }
}

// Operators
const map = (transformFn) => (source) =>
  new Observable(observer => {
    const sub = source.subscribe({
      next: (value) => observer.next(transformFn(value)),
      error: observer.error,
      complete: observer.complete
    });
    return () => sub.unsubscribe();
  });

const filter = (predicateFn) => (source) =>
  new Observable(observer => {
    const sub = source.subscribe({
      next: (value) => {
        if (predicateFn(value)) observer.next(value);
      },
      error: observer.error,
      complete: observer.complete
    });
    return () => sub.unsubscribe();
  });

const debounce = (delayMs) => (source) =>
  new Observable(observer => {
    let timer;
    const sub = source.subscribe({
      next: (value) => {
        clearTimeout(timer);
        timer = setTimeout(() => observer.next(value), delayMs);
      },
      error: observer.error,
      complete: () => {
        clearTimeout(timer);
        observer.complete();
      }
    });
    return () => { sub.unsubscribe(); clearTimeout(timer); };
  });

module.exports = { Observable, map, filter, debounce };
```

---

## ขั้นตอนที่ 1070: Decorator Pattern

```javascript
// patterns/decorator/ServiceDecorators.js

// Timing Decorator
function withTiming(service, logger) {
  return new Proxy(service, {
    get(target, prop) {
      const original = target[prop];
      
      if (typeof original !== 'function') return original;
      
      return async function(...args) {
        const start = Date.now();
        
        try {
          const result = await original.apply(target, args);
          logger.info(`${target.constructor.name}.${prop}`, {
            duration: Date.now() - start
          });
          return result;
        } catch (error) {
          logger.error(`${target.constructor.name}.${prop} failed`, error, {
            duration: Date.now() - start
          });
          throw error;
        }
      };
    }
  });
}

// Caching Decorator
function withCaching(service, cacheClient, ttl = 300) {
  return new Proxy(service, {
    get(target, prop) {
      const original = target[prop];
      if (typeof original !== 'function') return original;
      
      // Only cache GET operations
      if (!prop.startsWith('get') && !prop.startsWith('find')) {
        return original.bind(target);
      }
      
      return async function(...args) {
        const cacheKey = `${target.constructor.name}:${prop}:${JSON.stringify(args)}`;
        
        const cached = await cacheClient.get(cacheKey);
        if (cached !== null) return cached;
        
        const result = await original.apply(target, args);
        
        if (result !== null) {
          await cacheClient.set(cacheKey, result, ttl);
        }
        
        return result;
      };
    }
  });
}

// Validation Decorator
function withValidation(service, schemas) {
  return new Proxy(service, {
    get(target, prop) {
      const original = target[prop];
      if (typeof original !== 'function') return original;
      
      const schema = schemas[prop];
      
      return async function(...args) {
        if (schema) {
          const { error } = schema.validate(args[0]);
          if (error) throw new Error(`Validation failed: ${error.message}`);
        }
        
        return original.apply(target, args);
      };
    }
  });
}

// ใช้งาน
function createProductService(deps) {
  let service = new ProductService(deps);
  
  service = withTiming(service, logger);
  service = withCaching(service, redisClient, 300);
  service = withValidation(service, productSchemas);
  
  return service;
}

module.exports = { withTiming, withCaching, withValidation };
```

---

## ขั้นตอนที่ 1071: CQRS Advanced Implementation

```javascript
// patterns/cqrs/CommandBus.js

class CommandBus {
  constructor() {
    this.handlers = new Map();
    this.middlewares = [];
  }

  register(CommandClass, handler) {
    this.handlers.set(CommandClass.name, handler);
    return this;
  }

  use(...middlewares) {
    this.middlewares.push(...middlewares);
    return this;
  }

  async dispatch(command) {
    const handler = this.handlers.get(command.constructor.name);
    if (!handler) throw new Error(`No handler for: ${command.constructor.name}`);
    
    const pipeline = [...this.middlewares, async (cmd, _) => handler.handle(cmd)];
    
    let index = 0;
    const execute = async (cmd) => {
      if (index >= pipeline.length) return;
      const current = pipeline[index++];
      return current(cmd, execute);
    };
    
    return execute(command);
  }
}

// Query Bus
class QueryBus {
  constructor() {
    this.handlers = new Map();
  }

  register(QueryClass, handler) {
    this.handlers.set(QueryClass.name, handler);
    return this;
  }

  async dispatch(query) {
    const handler = this.handlers.get(query.constructor.name);
    if (!handler) throw new Error(`No handler for: ${query.constructor.name}`);
    return handler.handle(query);
  }
}

// Commands
class CreateProductCommand {
  constructor({ name, price, stock, categoryId }) {
    this.name = name;
    this.price = price;
    this.stock = stock;
    this.categoryId = categoryId;
    
    if (!name || !price) throw new Error('Name and price are required');
  }
}

// Queries
class GetProductQuery {
  constructor(productId) {
    this.productId = productId;
  }
}

class ListProductsQuery {
  constructor({ page = 1, limit = 20, categoryId, search } = {}) {
    this.page = page;
    this.limit = limit;
    this.categoryId = categoryId;
    this.search = search;
  }
}

// Setup
const commandBus = new CommandBus();
const queryBus = new QueryBus();

// Middlewares
const validationMiddleware = async (command, next) => {
  if (typeof command.validate === 'function') command.validate();
  return next(command);
};

const loggingMiddleware = async (command, next) => {
  console.log(`Command: ${command.constructor.name}`);
  const result = await next(command);
  console.log(`Command completed: ${command.constructor.name}`);
  return result;
};

commandBus.use(validationMiddleware, loggingMiddleware);

module.exports = { CommandBus, QueryBus, commandBus, queryBus };
```

---

## ขั้นตอนที่ 1072: Rate Limiting ขั้นสูง

```javascript
// patterns/rateLimit/AdaptiveRateLimiter.js
// Rate limiter ที่ปรับตามสถานการณ์

class AdaptiveRateLimiter {
  constructor(baseLimit, window, redis) {
    this.baseLimit = baseLimit;
    this.window = window;
    this.redis = redis;
    
    this.systemLoad = 0;
    this._updateLoad();
  }

  async allow(key) {
    // คำนวณ effective limit ตาม load
    const effectiveLimit = this._getEffectiveLimit();
    
    const now = Date.now();
    const windowStart = now - this.window * 1000;
    const redisKey = `rate:${key}`;
    
    const pipeline = this.redis.pipeline();
    pipeline.zremrangebyscore(redisKey, 0, windowStart);
    pipeline.zadd(redisKey, now, `${now}-${Math.random()}`);
    pipeline.zcard(redisKey);
    pipeline.expire(redisKey, this.window);
    
    const results = await pipeline.exec();
    const count = results[2][1];
    
    return {
      allowed: count <= effectiveLimit,
      count,
      limit: effectiveLimit,
      remaining: Math.max(0, effectiveLimit - count),
      resetIn: this.window
    };
  }

  _getEffectiveLimit() {
    // ลด limit เมื่อ system load สูง
    if (this.systemLoad > 0.9) return Math.floor(this.baseLimit * 0.5);
    if (this.systemLoad > 0.7) return Math.floor(this.baseLimit * 0.75);
    return this.baseLimit;
  }

  _updateLoad() {
    setInterval(() => {
      const os = require('os');
      const cpus = os.cpus();
      const load = os.loadavg()[0] / cpus.length;
      this.systemLoad = Math.min(load, 1);
    }, 5000);
  }
}

module.exports = { AdaptiveRateLimiter };
```

---

## ขั้นตอนที่ 1073: Health Check System

```javascript
// patterns/health/HealthChecker.js

class HealthChecker {
  constructor() {
    this.checks = new Map();
    this.timeout = 5000;
  }

  register(name, checkFn, options = {}) {
    this.checks.set(name, {
      fn: checkFn,
      critical: options.critical !== false,
      timeout: options.timeout || this.timeout
    });
    return this;
  }

  async check() {
    const results = {};
    let overallStatus = 'healthy';
    
    await Promise.all(
      Array.from(this.checks.entries()).map(async ([name, check]) => {
        try {
          const result = await Promise.race([
            check.fn(),
            new Promise((_, reject) =>
              setTimeout(() => reject(new Error('Timeout')), check.timeout)
            )
          ]);
          
          results[name] = {
            status: 'healthy',
            ...result
          };
        } catch (error) {
          results[name] = {
            status: 'unhealthy',
            error: error.message
          };
          
          if (check.critical) overallStatus = 'unhealthy';
          else if (overallStatus !== 'unhealthy') overallStatus = 'degraded';
        }
      })
    );
    
    return {
      status: overallStatus,
      timestamp: new Date().toISOString(),
      checks: results
    };
  }
}

// Setup health checks
const healthChecker = new HealthChecker();

healthChecker.register('mongodb', async () => {
  const mongoose = require('mongoose');
  if (mongoose.connection.readyState !== 1) throw new Error('Not connected');
  return { message: 'Connected' };
}, { critical: true });

healthChecker.register('redis', async () => {
  const redis = require('./redisClient');
  await redis.ping();
  return { message: 'Pong' };
}, { critical: false });

healthChecker.register('memory', () => {
  const used = process.memoryUsage();
  const heapUsed = Math.round(used.heapUsed / 1024 / 1024);
  const heapTotal = Math.round(used.heapTotal / 1024 / 1024);
  
  return {
    heapUsed: `${heapUsed}MB`,
    heapTotal: `${heapTotal}MB`,
    rss: `${Math.round(used.rss / 1024 / 1024)}MB`
  };
});

// Express endpoint
async function healthHandler(req, res) {
  const result = await healthChecker.check();
  const statusCode = result.status === 'healthy' ? 200 : 
                    result.status === 'degraded' ? 200 : 503;
  
  res.status(statusCode).json(result);
}

module.exports = { HealthChecker, healthChecker, healthHandler };
```

---

## ขั้นตอนที่ 1074: Feature Flags

```javascript
// patterns/featureFlags/FeatureFlags.js

class FeatureFlags {
  constructor(config = {}) {
    this.flags = new Map(Object.entries(config));
    this.userFlags = new Map(); // Per-user overrides
  }

  isEnabled(flagName, context = {}) {
    const flag = this.flags.get(flagName);
    
    if (!flag) return false;
    if (typeof flag === 'boolean') return flag;
    
    // Check user-specific override
    if (context.userId) {
      const userFlag = this.userFlags.get(`${context.userId}:${flagName}`);
      if (userFlag !== undefined) return userFlag;
    }
    
    // Percentage rollout
    if (flag.percentage !== undefined) {
      const key = context.userId || context.ip || Math.random().toString();
      const hash = this._hash(key + flagName);
      return hash % 100 < flag.percentage;
    }
    
    // User list
    if (flag.users && context.userId) {
      return flag.users.includes(context.userId);
    }
    
    // Environment
    if (flag.environments) {
      return flag.environments.includes(process.env.NODE_ENV);
    }
    
    return flag.enabled || false;
  }

  enable(flagName) {
    this.flags.set(flagName, true);
  }

  disable(flagName) {
    this.flags.set(flagName, false);
  }

  setUserFlag(userId, flagName, enabled) {
    this.userFlags.set(`${userId}:${flagName}`, enabled);
  }

  _hash(str) {
    let hash = 0;
    for (let i = 0; i < str.length; i++) {
      hash = ((hash << 5) - hash) + str.charCodeAt(i);
      hash = hash & hash;
    }
    return Math.abs(hash);
  }

  getAll(context = {}) {
    const result = {};
    for (const [name] of this.flags) {
      result[name] = this.isEnabled(name, context);
    }
    return result;
  }
}

// Config
const featureFlags = new FeatureFlags({
  newCheckoutFlow: {
    percentage: 10,    // 10% users
    environments: ['production', 'staging']
  },
  betaFeature: {
    users: ['user-123', 'user-456']
  },
  darkMode: true,
  experimentalApi: false
});

// Express middleware
function featureFlagMiddleware(req, res, next) {
  req.features = featureFlags;
  req.isFeatureEnabled = (flagName) => featureFlags.isEnabled(flagName, {
    userId: req.user?.id,
    ip: req.ip
  });
  next();
}

module.exports = { FeatureFlags, featureFlags, featureFlagMiddleware };
```

---

## ขั้นตอนที่ 1075: Graceful Shutdown

```javascript
// patterns/gracefulShutdown/GracefulShutdown.js

class GracefulShutdown {
  constructor(options = {}) {
    this.timeout = options.timeout || 30000;
    this.handlers = [];
    this._isShuttingDown = false;
    this._activeRequests = 0;
  }

  register(name, handler, priority = 0) {
    this.handlers.push({ name, handler, priority });
    this.handlers.sort((a, b) => b.priority - a.priority);
    return this;
  }

  trackRequest(req, res, next) {
    this._activeRequests++;
    
    res.on('finish', () => this._activeRequests--);
    res.on('close', () => this._activeRequests--);
    
    if (this._isShuttingDown) {
      res.set('Connection', 'close');
    }
    
    next();
  }

  async shutdown(signal) {
    if (this._isShuttingDown) return;
    this._isShuttingDown = true;
    
    console.log(`Received ${signal}. Starting graceful shutdown...`);
    
    // รอให้ active requests เสร็จ
    const waitForRequests = new Promise(resolve => {
      if (this._activeRequests === 0) return resolve();
      
      const interval = setInterval(() => {
        console.log(`Waiting for ${this._activeRequests} active requests...`);
        if (this._activeRequests === 0) {
          clearInterval(interval);
          resolve();
        }
      }, 1000);
    });
    
    const timeout = new Promise((_, reject) =>
      setTimeout(() => reject(new Error('Shutdown timeout')), this.timeout)
    );
    
    try {
      await Promise.race([waitForRequests, timeout]);
    } catch (error) {
      console.warn('Shutdown timeout - forcing close');
    }
    
    // รัน cleanup handlers
    for (const { name, handler } of this.handlers) {
      console.log(`Running shutdown handler: ${name}`);
      try {
        await Promise.race([
          handler(),
          new Promise((_, reject) => setTimeout(() => reject(new Error('Handler timeout')), 10000))
        ]);
        console.log(`Handler ${name} completed`);
      } catch (error) {
        console.error(`Handler ${name} failed:`, error.message);
      }
    }
    
    console.log('Graceful shutdown complete');
    process.exit(0);
  }

  setupHandlers() {
    process.on('SIGTERM', () => this.shutdown('SIGTERM'));
    process.on('SIGINT', () => this.shutdown('SIGINT'));
    process.on('uncaughtException', (error) => {
      console.error('Uncaught exception:', error);
      this.shutdown('uncaughtException');
    });
  }
}

// ใช้งาน
const shutdown = new GracefulShutdown({ timeout: 30000 });

shutdown
  .register('HTTP Server', async () => {
    await new Promise(resolve => httpServer.close(resolve));
  }, 10)
  .register('MongoDB', async () => {
    await mongoose.connection.close();
  }, 5)
  .register('Redis', async () => {
    await redis.quit();
  }, 5);

shutdown.setupHandlers();
app.use(shutdown.trackRequest.bind(shutdown));

module.exports = GracefulShutdown;
```

---

## ขั้นตอนที่ 1076-1080: Summary of Advanced Patterns

```javascript
// summary.js

/*
  Advanced Patterns Overview:
  
  1. Saga Pattern
     - Distributed transactions across multiple services
     - Compensating transactions เมื่อ step ล้มเหลว
     - Choreography vs Orchestration
  
  2. Outbox Pattern
     - Atomic writes: business data + events ใน transaction เดียวกัน
     - Relay service ส่ง events ไปยัง message broker
     - Guarantees at-least-once delivery
  
  3. Circuit Breaker
     - CLOSED → OPEN เมื่อ failures ถึง threshold
     - OPEN → HALF_OPEN เมื่อ timeout ผ่านไป
     - HALF_OPEN → CLOSED เมื่อ successes ถึง threshold
  
  4. Bulkhead
     - แยก thread pools สำหรับ operations ต่างๆ
     - ป้องกัน cascade failures
     - Configurable concurrency limits
  
  5. Retry with Exponential Backoff
     - Retry เฉพาะ transient errors
     - Exponential backoff + jitter
     - ไม่ retry business errors
  
  6. Cache Patterns
     - Cache-Aside (Lazy Loading)
     - Write-Through
     - Stale-While-Revalidate
     - Stampede Prevention
  
  7. CQRS
     - Command Bus สำหรับ write operations
     - Query Bus สำหรับ read operations
     - Separate read/write models
  
  8. Event-Driven Architecture
     - Loose coupling ระหว่าง services
     - Domain Events สะท้อน business
     - Event sourcing as audit log
  
  9. Feature Flags
     - Percentage rollout
     - A/B testing
     - Emergency kill switch
  
  10. Graceful Shutdown
      - Handle SIGTERM/SIGINT
      - รอ active requests
      - Close connections อย่างถูกต้อง
*/

// Production Checklist
const productionChecklist = {
  resilience: [
    'Circuit breaker สำหรับ external calls',
    'Retry logic สำหรับ transient failures',
    'Bulkhead สำหรับ resource isolation',
    'Timeout ทุก external call',
    'Graceful shutdown'
  ],
  
  observability: [
    'Structured logging (JSON)',
    'Distributed tracing (correlation IDs)',
    'Custom metrics (Prometheus)',
    'Health check endpoints',
    'Error tracking (Sentry)'
  ],
  
  scalability: [
    'Stateless application design',
    'Cache ที่เหมาะสม',
    'Connection pooling',
    'Database indexing',
    'Async operations'
  ],
  
  security: [
    'Input validation',
    'Authentication + Authorization',
    'Rate limiting',
    'Secrets management',
    'HTTPS everywhere'
  ]
};

module.exports = productionChecklist;
```

---

## แบบฝึกหัดท้ายบท

### แบบฝึกหัดที่ 1: Saga Implementation
Implement complete order processing saga ที่มี 4+ steps และ compensation logic

### แบบฝึกหัดที่ 2: Resilient API Client
สร้าง HTTP client ที่มี Circuit Breaker + Retry + Bulkhead + Timeout

### แบบฝึกหัดที่ 3: Event-Driven System
สร้าง mini event-driven system ด้วย Order → Inventory → Notification services

### แบบฝึกหัดที่ 4: Feature Flag System
Implement feature flags ด้วย A/B testing และ percentage rollout

### แบบฝึกหัดที่ 5: Production-Ready Service
สร้าง Node.js microservice ที่มี patterns ทั้งหมด: health check, graceful shutdown, metrics, logging, circuit breaker

---

## สรุปหลักสูตรทั้งหมด

คุณได้เรียนรู้ Node.js/Express.js ครบ 1,000+ ขั้นตอน ตั้งแต่พื้นฐาน HTTP, Middleware, Database จนถึง Advanced Architecture Patterns:

- **Security**: Authentication, Authorization, Rate Limiting, Input Validation
- **Performance**: Caching, Database Optimization, Connection Pooling
- **Architecture**: Clean Architecture, DDD, Hexagonal, Microservices
- **Reliability**: Circuit Breaker, Retry, Saga, Outbox Pattern
- **Operations**: Docker, Kubernetes, Serverless, CI/CD
- **Protocols**: REST, GraphQL, WebSocket, gRPC
- **Patterns**: CQRS, Event Sourcing, Feature Flags

นำความรู้เหล่านี้ไปสร้าง production-grade Node.js applications ได้เลย!
