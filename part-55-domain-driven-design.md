# Part 55 | ขั้นตอนที่ 961-980 จาก 1000

## Domain-Driven Design (DDD) - การออกแบบโดยใช้ Domain เป็นศูนย์กลาง

DDD เป็น approach ในการพัฒนา software ที่เน้นการสร้าง model ที่สะท้อน business domain อย่างใกล้ชิด

---

## ขั้นตอนที่ 961: ทำความเข้าใจ DDD Concepts

### Building Blocks ของ DDD:

**Tactical Patterns**:
- **Entities** - Objects ที่มี identity
- **Value Objects** - Objects ที่กำหนดโดย attributes
- **Aggregates** - Cluster ของ entities/value objects
- **Repositories** - Data access abstraction
- **Domain Services** - Business logic ที่ไม่เหมาะกับ entity
- **Domain Events** - การบันทึกเหตุการณ์ใน domain

**Strategic Patterns**:
- **Bounded Contexts** - ขอบเขตของ domain model
- **Context Maps** - ความสัมพันธ์ระหว่าง bounded contexts
- **Ubiquitous Language** - ภาษากลางร่วมกัน

---

## ขั้นตอนที่ 962: Value Objects

```javascript
// domain/valueObjects/Money.js

class Money {
  constructor(amount, currency = 'THB') {
    if (typeof amount !== 'number' || isNaN(amount)) {
      throw new Error('Amount must be a valid number');
    }
    
    if (amount < 0) {
      throw new Error('Amount cannot be negative');
    }
    
    this._amount = Math.round(amount * 100) / 100; // Round to 2 decimal places
    this._currency = currency.toUpperCase();
    
    Object.freeze(this); // Value objects are immutable
  }

  get amount() { return this._amount; }
  get currency() { return this._currency; }

  add(other) {
    this._assertSameCurrency(other);
    return new Money(this._amount + other.amount, this._currency);
  }

  subtract(other) {
    this._assertSameCurrency(other);
    const result = this._amount - other.amount;
    if (result < 0) throw new Error('Insufficient funds');
    return new Money(result, this._currency);
  }

  multiply(factor) {
    return new Money(this._amount * factor, this._currency);
  }

  equals(other) {
    if (!(other instanceof Money)) return false;
    return this._amount === other.amount && this._currency === other.currency;
  }

  isGreaterThan(other) {
    this._assertSameCurrency(other);
    return this._amount > other.amount;
  }

  isLessThan(other) {
    this._assertSameCurrency(other);
    return this._amount < other.amount;
  }

  _assertSameCurrency(other) {
    if (this._currency !== other.currency) {
      throw new Error(`Currency mismatch: ${this._currency} vs ${other.currency}`);
    }
  }

  toString() {
    return `${this._currency} ${this._amount.toFixed(2)}`;
  }

  toJSON() {
    return { amount: this._amount, currency: this._currency };
  }

  static fromJSON({ amount, currency }) {
    return new Money(amount, currency);
  }

  static zero(currency = 'THB') {
    return new Money(0, currency);
  }
}

module.exports = Money;
```

```javascript
// domain/valueObjects/Address.js

class Address {
  constructor({ street, city, province, postalCode, country = 'Thailand' }) {
    if (!street || !city || !postalCode) {
      throw new Error('Street, city, and postal code are required');
    }
    
    this._street = street;
    this._city = city;
    this._province = province;
    this._postalCode = postalCode;
    this._country = country;
    
    Object.freeze(this);
  }

  get street() { return this._street; }
  get city() { return this._city; }
  get province() { return this._province; }
  get postalCode() { return this._postalCode; }
  get country() { return this._country; }

  equals(other) {
    if (!(other instanceof Address)) return false;
    return (
      this._street === other.street &&
      this._city === other.city &&
      this._postalCode === other.postalCode &&
      this._country === other.country
    );
  }

  toString() {
    return `${this._street}, ${this._city}, ${this._province} ${this._postalCode}, ${this._country}`;
  }

  toJSON() {
    return {
      street: this._street,
      city: this._city,
      province: this._province,
      postalCode: this._postalCode,
      country: this._country
    };
  }
}

module.exports = Address;
```

```javascript
// domain/valueObjects/Email.js

class Email {
  constructor(email) {
    const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
    
    if (!email || !emailRegex.test(email)) {
      throw new Error(`Invalid email address: ${email}`);
    }
    
    this._value = email.toLowerCase().trim();
    
    Object.freeze(this);
  }

  get value() { return this._value; }
  
  get domain() {
    return this._value.split('@')[1];
  }

  equals(other) {
    if (!(other instanceof Email)) return false;
    return this._value === other.value;
  }

  toString() { return this._value; }
}

module.exports = Email;
```

---

## ขั้นตอนที่ 963: Entities

```javascript
// domain/entities/Product.js

const { v4: uuidv4 } = require('uuid');
const Money = require('../valueObjects/Money');

class Product {
  constructor({ id, name, description, price, category, stock = 0 }) {
    this._id = id || uuidv4();
    this._name = name;
    this._description = description;
    this._price = price instanceof Money ? price : new Money(price);
    this._category = category;
    this._stock = stock;
    this._isActive = true;
    this._version = 0;
    this._createdAt = new Date();
    this._updatedAt = new Date();
  }

  // Getters
  get id() { return this._id; }
  get name() { return this._name; }
  get price() { return this._price; }
  get stock() { return this._stock; }
  get isActive() { return this._isActive; }
  get version() { return this._version; }

  // Business methods
  updatePrice(newPrice) {
    const price = newPrice instanceof Money ? newPrice : new Money(newPrice);
    
    if (price.isLessThan(Money.zero())) {
      throw new Error('Price cannot be negative');
    }
    
    this._price = price;
    this._updatedAt = new Date();
    this._version++;
    
    return this;
  }

  adjustStock(quantity) {
    const newStock = this._stock + quantity;
    
    if (newStock < 0) {
      throw new Error(`Insufficient stock. Current: ${this._stock}, Requested: ${Math.abs(quantity)}`);
    }
    
    this._stock = newStock;
    this._updatedAt = new Date();
    this._version++;
    
    return this;
  }

  reserveStock(quantity) {
    if (quantity <= 0) throw new Error('Quantity must be positive');
    if (this._stock < quantity) throw new Error('Insufficient stock');
    
    return this.adjustStock(-quantity);
  }

  restoreStock(quantity) {
    if (quantity <= 0) throw new Error('Quantity must be positive');
    return this.adjustStock(quantity);
  }

  deactivate() {
    if (!this._isActive) throw new Error('Product is already inactive');
    this._isActive = false;
    this._updatedAt = new Date();
    this._version++;
    return this;
  }

  activate() {
    if (this._isActive) throw new Error('Product is already active');
    this._isActive = true;
    this._updatedAt = new Date();
    this._version++;
    return this;
  }

  // Identity comparison
  equals(other) {
    if (!(other instanceof Product)) return false;
    return this._id === other.id;
  }

  toJSON() {
    return {
      id: this._id,
      name: this._name,
      description: this._description,
      price: this._price.toJSON(),
      category: this._category,
      stock: this._stock,
      isActive: this._isActive,
      version: this._version,
      createdAt: this._createdAt,
      updatedAt: this._updatedAt
    };
  }
}

module.exports = Product;
```

---

## ขั้นตอนที่ 964: Aggregates

```javascript
// domain/aggregates/Order.js

const { v4: uuidv4 } = require('uuid');
const Money = require('../valueObjects/Money');
const Address = require('../valueObjects/Address');

class OrderLine {
  constructor({ productId, productName, price, quantity }) {
    this.productId = productId;
    this.productName = productName;
    this.price = price instanceof Money ? price : new Money(price);
    this.quantity = quantity;
    
    if (quantity <= 0) throw new Error('Quantity must be positive');
    
    Object.freeze(this);
  }

  get subtotal() {
    return this.price.multiply(this.quantity);
  }
}

class Order {
  constructor({ id, customerId, shippingAddress }) {
    this._id = id || uuidv4();
    this._customerId = customerId;
    this._shippingAddress = shippingAddress instanceof Address 
      ? shippingAddress 
      : new Address(shippingAddress);
    this._lines = [];
    this._status = 'draft';
    this._domainEvents = [];
    this._createdAt = new Date();
    this._version = 0;
  }

  // Getters
  get id() { return this._id; }
  get customerId() { return this._customerId; }
  get status() { return this._status; }
  get lines() { return [...this._lines]; }
  
  get total() {
    return this._lines.reduce(
      (sum, line) => sum.add(line.subtotal),
      Money.zero()
    );
  }

  // Business methods
  addLine(orderLine) {
    if (this._status !== 'draft') {
      throw new Error('Can only add lines to draft orders');
    }
    
    if (!(orderLine instanceof OrderLine)) {
      throw new Error('Expected OrderLine instance');
    }
    
    // ตรวจสอบว่ามี product อยู่แล้วหรือไม่
    const existingIndex = this._lines.findIndex(
      l => l.productId === orderLine.productId
    );
    
    if (existingIndex >= 0) {
      // อัพเดท quantity แทน
      const existing = this._lines[existingIndex];
      this._lines[existingIndex] = new OrderLine({
        productId: existing.productId,
        productName: existing.productName,
        price: existing.price,
        quantity: existing.quantity + orderLine.quantity
      });
    } else {
      this._lines.push(orderLine);
    }
    
    this._version++;
    
    return this;
  }

  removeLine(productId) {
    if (this._status !== 'draft') {
      throw new Error('Can only remove lines from draft orders');
    }
    
    this._lines = this._lines.filter(l => l.productId !== productId);
    this._version++;
    
    return this;
  }

  submit() {
    if (this._status !== 'draft') {
      throw new Error('Only draft orders can be submitted');
    }
    
    if (this._lines.length === 0) {
      throw new Error('Cannot submit empty order');
    }
    
    this._status = 'submitted';
    this._version++;
    
    // Domain event
    this._addDomainEvent({
      type: 'OrderSubmitted',
      data: {
        orderId: this._id,
        customerId: this._customerId,
        total: this.total.toJSON(),
        lines: this._lines.map(l => ({
          productId: l.productId,
          quantity: l.quantity
        }))
      }
    });
    
    return this;
  }

  confirm() {
    if (this._status !== 'submitted') {
      throw new Error('Only submitted orders can be confirmed');
    }
    
    this._status = 'confirmed';
    this._version++;
    
    this._addDomainEvent({
      type: 'OrderConfirmed',
      data: { orderId: this._id }
    });
    
    return this;
  }

  cancel(reason) {
    if (['delivered', 'cancelled'].includes(this._status)) {
      throw new Error(`Cannot cancel ${this._status} order`);
    }
    
    this._status = 'cancelled';
    this._cancellationReason = reason;
    this._version++;
    
    this._addDomainEvent({
      type: 'OrderCancelled',
      data: { orderId: this._id, reason }
    });
    
    return this;
  }

  // Domain events
  _addDomainEvent(event) {
    this._domainEvents.push({ ...event, occurredAt: new Date() });
  }

  getDomainEvents() {
    return [...this._domainEvents];
  }

  clearDomainEvents() {
    this._domainEvents = [];
  }

  // Identity
  equals(other) {
    if (!(other instanceof Order)) return false;
    return this._id === other.id;
  }

  toJSON() {
    return {
      id: this._id,
      customerId: this._customerId,
      shippingAddress: this._shippingAddress.toJSON(),
      lines: this._lines.map(l => ({
        productId: l.productId,
        productName: l.productName,
        price: l.price.toJSON(),
        quantity: l.quantity,
        subtotal: l.subtotal.toJSON()
      })),
      total: this.total.toJSON(),
      status: this._status,
      version: this._version,
      createdAt: this._createdAt
    };
  }
}

module.exports = { Order, OrderLine };
```

---

## ขั้นตอนที่ 965: Repositories

```javascript
// domain/repositories/IProductRepository.js

// Interface (Abstract class ใน JavaScript)
class IProductRepository {
  async findById(id) {
    throw new Error('findById must be implemented');
  }

  async findByIds(ids) {
    throw new Error('findByIds must be implemented');
  }

  async save(product) {
    throw new Error('save must be implemented');
  }

  async delete(id) {
    throw new Error('delete must be implemented');
  }

  async findByCategory(categoryId, options) {
    throw new Error('findByCategory must be implemented');
  }
}

module.exports = IProductRepository;
```

```javascript
// infrastructure/repositories/MongoProductRepository.js

const IProductRepository = require('../../domain/repositories/IProductRepository');
const Product = require('../../domain/entities/Product');
const Money = require('../../domain/valueObjects/Money');
const ProductModel = require('../models/ProductMongooseModel');

class MongoProductRepository extends IProductRepository {
  async findById(id) {
    const doc = await ProductModel.findById(id).lean();
    
    if (!doc) return null;
    
    return this._toDomain(doc);
  }

  async findByIds(ids) {
    const docs = await ProductModel.find({ _id: { $in: ids } }).lean();
    return docs.map(doc => this._toDomain(doc));
  }

  async save(product) {
    const data = this._toPersistence(product);
    
    await ProductModel.findOneAndUpdate(
      { _id: product.id },
      {
        $set: data,
        $setOnInsert: { _id: product.id }
      },
      { upsert: true, new: true }
    );
    
    return product;
  }

  async delete(id) {
    await ProductModel.findByIdAndDelete(id);
  }

  async findByCategory(categoryId, options = {}) {
    const { page = 1, limit = 20 } = options;
    
    const docs = await ProductModel.find({
      category: categoryId,
      isActive: true
    })
    .skip((page - 1) * limit)
    .limit(limit)
    .lean();
    
    return docs.map(doc => this._toDomain(doc));
  }

  // Mapping functions
  _toDomain(doc) {
    return new Product({
      id: doc._id.toString(),
      name: doc.name,
      description: doc.description,
      price: new Money(doc.price, doc.currency || 'THB'),
      category: doc.category?.toString(),
      stock: doc.stock
    });
  }

  _toPersistence(product) {
    return {
      name: product.name,
      description: product._description,
      price: product.price.amount,
      currency: product.price.currency,
      category: product._category,
      stock: product.stock,
      isActive: product.isActive,
      version: product.version,
      updatedAt: new Date()
    };
  }
}

module.exports = MongoProductRepository;
```

---

## ขั้นตอนที่ 966: Domain Services

```javascript
// domain/services/PricingService.js

const Money = require('../valueObjects/Money');

class PricingService {
  constructor(discountRepository, taxRateService) {
    this.discountRepository = discountRepository;
    this.taxRateService = taxRateService;
  }

  // คำนวณราคาสุดท้ายพร้อม discount และ tax
  async calculateOrderTotal(order, customerId) {
    const subtotal = order.total;
    
    // คำนวณ discount
    const discount = await this.calculateDiscount(
      order,
      customerId,
      subtotal
    );
    
    const afterDiscount = subtotal.subtract(discount);
    
    // คำนวณ tax
    const taxRate = await this.taxRateService.getRate(
      order.shippingAddress.country
    );
    
    const tax = afterDiscount.multiply(taxRate / 100);
    const total = afterDiscount.add(tax);
    
    return {
      subtotal,
      discount,
      afterDiscount,
      tax,
      total,
      taxRate
    };
  }

  async calculateDiscount(order, customerId, subtotal) {
    // ตรวจสอบ discounts ที่ apply ได้
    const discounts = await this.discountRepository.findActiveForCustomer(customerId);
    
    let totalDiscount = Money.zero();
    
    for (const discount of discounts) {
      const amount = this._applyDiscount(discount, subtotal, order);
      totalDiscount = totalDiscount.add(amount);
    }
    
    return totalDiscount;
  }

  _applyDiscount(discount, subtotal, order) {
    switch (discount.type) {
      case 'percentage':
        return subtotal.multiply(discount.value / 100);
      
      case 'fixed':
        return new Money(Math.min(discount.value, subtotal.amount));
      
      case 'buy_x_get_y':
        // ตรวจสอบว่าซื้อครบเงื่อนไข
        const eligibleLines = order.lines.filter(l => 
          discount.eligibleProducts.includes(l.productId) &&
          l.quantity >= discount.buyQuantity
        );
        
        if (eligibleLines.length > 0) {
          const freeItems = Math.floor(
            eligibleLines[0].quantity / discount.buyQuantity
          );
          return eligibleLines[0].price.multiply(
            Math.min(freeItems * discount.getQuantity, discount.maxFreeItems || Infinity)
          );
        }
        
        return Money.zero();
      
      default:
        return Money.zero();
    }
  }
}

module.exports = PricingService;
```

---

## ขั้นตอนที่ 967: Bounded Contexts

```javascript
// contexts/orderContext/index.js
// Order Bounded Context

module.exports = {
  // Domain
  aggregates: {
    Order: require('./domain/aggregates/Order')
  },
  entities: {
    Customer: require('./domain/entities/Customer')
  },
  valueObjects: {
    Money: require('./domain/valueObjects/Money'),
    Address: require('./domain/valueObjects/Address')
  },
  services: {
    OrderService: require('./domain/services/OrderService'),
    PricingService: require('./domain/services/PricingService')
  },
  
  // Application
  commands: {
    CreateOrderCommand: require('./application/commands/CreateOrderCommand'),
    SubmitOrderCommand: require('./application/commands/SubmitOrderCommand')
  },
  queries: {
    GetOrderQuery: require('./application/queries/GetOrderQuery'),
    ListUserOrdersQuery: require('./application/queries/ListUserOrdersQuery')
  },
  
  // Infrastructure
  repositories: {
    OrderRepository: require('./infrastructure/repositories/MongoOrderRepository')
  }
};
```

```javascript
// contexts/catalogContext/index.js
// Catalog Bounded Context - แยกจาก Order Context

module.exports = {
  aggregates: {
    Product: require('./domain/aggregates/Product'),
    Category: require('./domain/aggregates/Category')
  },
  services: {
    ProductSearchService: require('./domain/services/ProductSearchService'),
    InventoryService: require('./domain/services/InventoryService')
  }
};
```

---

## ขั้นตอนที่ 968: Context Map - Anti-Corruption Layer

```javascript
// contexts/orderContext/infrastructure/adapters/CatalogAdapter.js
// Anti-Corruption Layer: แปลง Catalog context objects ไปยัง Order context

const ProductCatalogService = require('../../../catalogContext').services.ProductSearchService;

class CatalogAdapter {
  constructor() {
    this.catalogService = new ProductCatalogService();
  }

  // แปลง catalog product เป็น order context model
  async getOrderableProduct(productId) {
    const catalogProduct = await this.catalogService.findById(productId);
    
    if (!catalogProduct) return null;
    
    // Anti-corruption: แปลง catalog model ไปยัง order model
    return {
      id: catalogProduct.id,
      name: catalogProduct.name,
      price: this._toMoney(catalogProduct.pricing.current),
      isAvailable: catalogProduct.status === 'active' && 
                   catalogProduct.inventory.quantity > 0,
      availableQuantity: catalogProduct.inventory.quantity
    };
  }

  _toMoney(pricingData) {
    const Money = require('../../domain/valueObjects/Money');
    return new Money(pricingData.amount, pricingData.currency || 'THB');
  }
}

module.exports = CatalogAdapter;
```

---

## ขั้นตอนที่ 969: Application Services

```javascript
// contexts/orderContext/application/OrderApplicationService.js

const { Order, OrderLine } = require('../domain/aggregates/Order');
const Address = require('../domain/valueObjects/Address');
const MongoOrderRepository = require('../infrastructure/repositories/MongoOrderRepository');
const CatalogAdapter = require('../infrastructure/adapters/CatalogAdapter');
const PricingService = require('../domain/services/PricingService');
const EventPublisher = require('../../../shared/EventPublisher');

class OrderApplicationService {
  constructor() {
    this.orderRepository = new MongoOrderRepository();
    this.catalogAdapter = new CatalogAdapter();
    this.pricingService = new PricingService();
  }

  async createOrder({ customerId, shippingAddress, items }) {
    // สร้าง Order aggregate
    const order = new Order({
      customerId,
      shippingAddress: new Address(shippingAddress)
    });
    
    // เพิ่ม order lines
    for (const item of items) {
      // ดึงข้อมูลจาก catalog context (ผ่าน ACL)
      const product = await this.catalogAdapter.getOrderableProduct(item.productId);
      
      if (!product) {
        throw new Error(`Product not found: ${item.productId}`);
      }
      
      if (!product.isAvailable) {
        throw new Error(`Product not available: ${product.name}`);
      }
      
      if (product.availableQuantity < item.quantity) {
        throw new Error(`Insufficient stock for: ${product.name}`);
      }
      
      const orderLine = new OrderLine({
        productId: product.id,
        productName: product.name,
        price: product.price,
        quantity: item.quantity
      });
      
      order.addLine(orderLine);
    }
    
    // บันทึก
    await this.orderRepository.save(order);
    
    // Publish domain events
    for (const event of order.getDomainEvents()) {
      await EventPublisher.publish(event);
    }
    
    order.clearDomainEvents();
    
    return order;
  }

  async submitOrder(orderId) {
    const order = await this.orderRepository.findById(orderId);
    
    if (!order) {
      throw new Error(`Order not found: ${orderId}`);
    }
    
    // คำนวณราคาสุดท้าย
    const pricing = await this.pricingService.calculateOrderTotal(
      order,
      order.customerId
    );
    
    order.submit();
    
    await this.orderRepository.save(order);
    
    // Publish events
    for (const event of order.getDomainEvents()) {
      await EventPublisher.publish(event);
    }
    
    order.clearDomainEvents();
    
    return { order, pricing };
  }

  async cancelOrder(orderId, reason) {
    const order = await this.orderRepository.findById(orderId);
    
    if (!order) {
      throw new Error(`Order not found: ${orderId}`);
    }
    
    order.cancel(reason);
    
    await this.orderRepository.save(order);
    
    for (const event of order.getDomainEvents()) {
      await EventPublisher.publish(event);
    }
    
    order.clearDomainEvents();
    
    return order;
  }
}

module.exports = OrderApplicationService;
```

---

## ขั้นตอนที่ 970: Ubiquitous Language

```javascript
// domain/ubiquitousLanguage.js

/*
  Ubiquitous Language สำหรับ E-Commerce Domain:
  
  === Order Context ===
  
  Order (คำสั่งซื้อ):
    - Draft: กำลังสร้าง ยังไม่ submit
    - Submitted: ส่งคำสั่งซื้อแล้ว รอ confirm
    - Confirmed: ยืนยันแล้ว กำลังดำเนินการ
    - Shipped: จัดส่งแล้ว
    - Delivered: ส่งถึงแล้ว
    - Cancelled: ยกเลิก
  
  OrderLine (รายการสินค้าในคำสั่งซื้อ):
    - Subtotal: ราคา x จำนวน
  
  Customer (ลูกค้า):
    - Loyalty Tier: ระดับสมาชิก (Regular, Silver, Gold, Platinum)
  
  === Catalog Context ===
  
  Product (สินค้า):
    - Available: พร้อมขาย (active + มี stock)
    - Out of Stock: หมด
    - Discontinued: เลิกผลิต
  
  Inventory (สินค้าคงคลัง):
    - Reserve: จอง (ยังไม่ตัด)
    - Commit: ตัดจริง
    - Release: คืน reservation
  
  === Payment Context ===
  
  Payment (การชำระเงิน):
    - Pending: รอดำเนินการ
    - Authorized: อนุมัติแล้ว ยังไม่ capture
    - Captured: เก็บเงินแล้ว
    - Refunded: คืนเงินแล้ว
    - Failed: ล้มเหลว
*/

// สร้าง Type definitions ที่สะท้อน ubiquitous language
const OrderStatus = Object.freeze({
  DRAFT: 'draft',
  SUBMITTED: 'submitted',
  CONFIRMED: 'confirmed',
  SHIPPED: 'shipped',
  DELIVERED: 'delivered',
  CANCELLED: 'cancelled'
});

const PaymentStatus = Object.freeze({
  PENDING: 'pending',
  AUTHORIZED: 'authorized',
  CAPTURED: 'captured',
  REFUNDED: 'refunded',
  FAILED: 'failed'
});

const CustomerTier = Object.freeze({
  REGULAR: 'regular',
  SILVER: 'silver',
  GOLD: 'gold',
  PLATINUM: 'platinum'
});

module.exports = { OrderStatus, PaymentStatus, CustomerTier };
```

---

## ขั้นตอนที่ 971: Domain Events

```javascript
// domain/events/DomainEvent.js

class DomainEvent {
  constructor(eventType, aggregateId, data) {
    this.eventId = require('crypto').randomUUID();
    this.eventType = eventType;
    this.aggregateId = aggregateId;
    this.data = data;
    this.occurredAt = new Date();
    this.version = 1;
  }
}

// Specific domain events
class OrderSubmitted extends DomainEvent {
  constructor(order) {
    super('OrderSubmitted', order.id, {
      orderId: order.id,
      customerId: order.customerId,
      total: order.total.toJSON(),
      itemCount: order.lines.length,
      lineItems: order.lines.map(l => ({
        productId: l.productId,
        quantity: l.quantity
      }))
    });
  }
}

class OrderCancelled extends DomainEvent {
  constructor(order, reason) {
    super('OrderCancelled', order.id, {
      orderId: order.id,
      customerId: order.customerId,
      reason,
      total: order.total.toJSON()
    });
  }
}

class ProductPriceChanged extends DomainEvent {
  constructor(product, oldPrice) {
    super('ProductPriceChanged', product.id, {
      productId: product.id,
      oldPrice: oldPrice.toJSON(),
      newPrice: product.price.toJSON()
    });
  }
}

class InventoryDepleted extends DomainEvent {
  constructor(product) {
    super('InventoryDepleted', product.id, {
      productId: product.id,
      productName: product.name
    });
  }
}

module.exports = { DomainEvent, OrderSubmitted, OrderCancelled, ProductPriceChanged, InventoryDepleted };
```

---

## ขั้นตอนที่ 972: Specification Pattern

```javascript
// domain/specifications/ProductSpecification.js

class Specification {
  isSatisfiedBy(candidate) {
    throw new Error('isSatisfiedBy must be implemented');
  }

  and(other) {
    return new AndSpecification(this, other);
  }

  or(other) {
    return new OrSpecification(this, other);
  }

  not() {
    return new NotSpecification(this);
  }
}

class AndSpecification extends Specification {
  constructor(left, right) {
    super();
    this.left = left;
    this.right = right;
  }

  isSatisfiedBy(candidate) {
    return this.left.isSatisfiedBy(candidate) && this.right.isSatisfiedBy(candidate);
  }
}

class OrSpecification extends Specification {
  constructor(left, right) {
    super();
    this.left = left;
    this.right = right;
  }

  isSatisfiedBy(candidate) {
    return this.left.isSatisfiedBy(candidate) || this.right.isSatisfiedBy(candidate);
  }
}

class NotSpecification extends Specification {
  constructor(spec) {
    super();
    this.spec = spec;
  }

  isSatisfiedBy(candidate) {
    return !this.spec.isSatisfiedBy(candidate);
  }
}

// Concrete specifications
class ActiveProductSpecification extends Specification {
  isSatisfiedBy(product) {
    return product.isActive;
  }
}

class InStockSpecification extends Specification {
  isSatisfiedBy(product) {
    return product.stock > 0;
  }
}

class AffordableSpecification extends Specification {
  constructor(maxPrice) {
    super();
    this.maxPrice = maxPrice;
  }

  isSatisfiedBy(product) {
    return product.price.amount <= this.maxPrice;
  }
}

class InCategorySpecification extends Specification {
  constructor(categoryId) {
    super();
    this.categoryId = categoryId;
  }

  isSatisfiedBy(product) {
    return product._category === this.categoryId;
  }
}

// ใช้งาน
const availableAndAffordable = new ActiveProductSpecification()
  .and(new InStockSpecification())
  .and(new AffordableSpecification(1000));

const products = allProducts.filter(p => availableAndAffordable.isSatisfiedBy(p));

module.exports = {
  Specification,
  ActiveProductSpecification,
  InStockSpecification,
  AffordableSpecification,
  InCategorySpecification
};
```

---

## ขั้นตอนที่ 973: Factory Pattern

```javascript
// domain/factories/OrderFactory.js

const { Order, OrderLine } = require('../aggregates/Order');
const Address = require('../valueObjects/Address');
const Money = require('../valueObjects/Money');

class OrderFactory {
  static createFromCart(cart, customer, shippingAddress) {
    const order = new Order({
      customerId: customer.id,
      shippingAddress: new Address(shippingAddress)
    });
    
    for (const cartItem of cart.items) {
      const orderLine = new OrderLine({
        productId: cartItem.product.id,
        productName: cartItem.product.name,
        price: cartItem.product.price,
        quantity: cartItem.quantity
      });
      
      order.addLine(orderLine);
    }
    
    return order;
  }

  static createReorder(previousOrder) {
    // Reorder: สร้าง order ใหม่จาก order เก่า
    const order = new Order({
      customerId: previousOrder.customerId,
      shippingAddress: previousOrder._shippingAddress
    });
    
    for (const line of previousOrder.lines) {
      const orderLine = new OrderLine({
        productId: line.productId,
        productName: line.productName,
        price: line.price,
        quantity: line.quantity
      });
      
      order.addLine(orderLine);
    }
    
    return order;
  }
}

module.exports = OrderFactory;
```

---

## ขั้นตอนที่ 974: Policy Objects

```javascript
// domain/policies/OrderPolicy.js

class OrderPolicy {
  static canCancel(order, customer) {
    // Draft orders สามารถ cancel ได้เสมอ
    if (order.status === 'draft') return true;
    
    // Submitted orders: customer ยกเลิกได้ภายใน 30 นาที
    if (order.status === 'submitted') {
      const minutesSinceSubmit = (Date.now() - order._submittedAt) / 60000;
      return minutesSinceSubmit <= 30;
    }
    
    // Confirmed orders: admin เท่านั้นที่ยกเลิกได้
    if (order.status === 'confirmed') {
      return customer.role === 'admin';
    }
    
    return false;
  }

  static canModify(order, customer) {
    if (order.status !== 'draft') return false;
    
    // เฉพาะ owner หรือ admin
    return order.customerId === customer.id || customer.role === 'admin';
  }

  static requiresApproval(order) {
    // Orders ที่มีมูลค่า > 50,000 ต้องผ่านการ approve
    return order.total.amount > 50000;
  }
}

// Discount Policy
class DiscountPolicy {
  static isEligibleForFreeShipping(order, customerTier) {
    const freeShippingThresholds = {
      regular: 1000,
      silver: 500,
      gold: 0,      // Gold ได้ free shipping เสมอ
      platinum: 0   // Platinum ได้ free shipping เสมอ
    };
    
    const threshold = freeShippingThresholds[customerTier] || 1000;
    return order.total.amount >= threshold;
  }

  static calculateLoyaltyDiscount(order, customerTier) {
    const discountRates = {
      regular: 0,
      silver: 0.05,  // 5%
      gold: 0.10,    // 10%
      platinum: 0.15 // 15%
    };
    
    const rate = discountRates[customerTier] || 0;
    return order.total.multiply(rate);
  }
}

module.exports = { OrderPolicy, DiscountPolicy };
```

---

## ขั้นตอนที่ 975: Testing Domain Objects

```javascript
// tests/domain/Order.test.js
const { Order, OrderLine } = require('../domain/aggregates/Order');
const Money = require('../domain/valueObjects/Money');
const Address = require('../domain/valueObjects/Address');

describe('Order Aggregate', () => {
  let order;

  beforeEach(() => {
    order = new Order({
      customerId: 'customer-123',
      shippingAddress: new Address({
        street: '123 Test St',
        city: 'Bangkok',
        postalCode: '10100'
      })
    });
  });

  describe('addLine', () => {
    it('should add line to draft order', () => {
      const line = new OrderLine({
        productId: 'prod-1',
        productName: 'Test Product',
        price: new Money(100),
        quantity: 2
      });
      
      order.addLine(line);
      
      expect(order.lines).toHaveLength(1);
      expect(order.total.amount).toBe(200);
    });

    it('should merge lines with same product', () => {
      const line = new OrderLine({
        productId: 'prod-1',
        productName: 'Test Product',
        price: new Money(100),
        quantity: 2
      });
      
      order.addLine(line);
      order.addLine(line); // เพิ่มซ้ำ
      
      expect(order.lines).toHaveLength(1);
      expect(order.lines[0].quantity).toBe(4);
    });

    it('should not allow adding to non-draft order', () => {
      const line = new OrderLine({
        productId: 'prod-1',
        productName: 'Test Product',
        price: new Money(100),
        quantity: 1
      });
      
      order.addLine(line);
      order.submit();
      
      expect(() => order.addLine(line)).toThrow('Can only add lines to draft orders');
    });
  });

  describe('submit', () => {
    it('should submit order with lines', () => {
      const line = new OrderLine({
        productId: 'prod-1',
        productName: 'Product',
        price: new Money(500),
        quantity: 1
      });
      
      order.addLine(line);
      order.submit();
      
      expect(order.status).toBe('submitted');
      
      const events = order.getDomainEvents();
      expect(events).toHaveLength(1);
      expect(events[0].type).toBe('OrderSubmitted');
    });

    it('should not submit empty order', () => {
      expect(() => order.submit()).toThrow('Cannot submit empty order');
    });
  });
});

describe('Money Value Object', () => {
  it('should add two Money objects', () => {
    const m1 = new Money(100);
    const m2 = new Money(200);
    
    expect(m1.add(m2).amount).toBe(300);
  });

  it('should be immutable', () => {
    const money = new Money(100);
    
    expect(() => {
      money._amount = 200;
    }).toThrow();
  });

  it('should compare equality', () => {
    const m1 = new Money(100, 'THB');
    const m2 = new Money(100, 'THB');
    const m3 = new Money(200, 'THB');
    
    expect(m1.equals(m2)).toBe(true);
    expect(m1.equals(m3)).toBe(false);
  });
});
```

---

## ขั้นตอนที่ 976: DDD Application Structure

```
src/
  contexts/
    order-context/
      domain/
        aggregates/
          Order.js
        entities/
          Customer.js
        valueObjects/
          Money.js
          Address.js
        services/
          PricingService.js
          OrderValidationService.js
        policies/
          OrderPolicy.js
        events/
          OrderEvents.js
        repositories/   (interfaces only)
          IOrderRepository.js
        factories/
          OrderFactory.js
        specifications/
          OrderSpecification.js
      application/
        commands/
          CreateOrderCommand.js
          SubmitOrderCommand.js
          CancelOrderCommand.js
        commandHandlers/
          CreateOrderHandler.js
          SubmitOrderHandler.js
        queries/
          GetOrderQuery.js
          ListUserOrdersQuery.js
        queryHandlers/
          GetOrderHandler.js
          ListUserOrdersHandler.js
        services/
          OrderApplicationService.js
      infrastructure/
        repositories/
          MongoOrderRepository.js
        adapters/
          CatalogAdapter.js
          PaymentAdapter.js
        models/
          OrderMongooseModel.js
      interfaces/
        api/
          routes/
            orderRoutes.js
          controllers/
            OrderController.js
    
    catalog-context/
      domain/...
      application/...
      infrastructure/...
    
    payment-context/
      domain/...
      
  shared/
    valueObjects/
      UniqueEntityId.js
    events/
      EventPublisher.js
    infrastructure/
      database.js
```

---

## ขั้นตอนที่ 977: Express Controller ด้วย DDD

```javascript
// contexts/order-context/interfaces/api/controllers/OrderController.js

const OrderApplicationService = require('../../../application/services/OrderApplicationService');
const OrderFactory = require('../../../domain/factories/OrderFactory');

class OrderController {
  constructor() {
    this.orderService = new OrderApplicationService();
  }

  createOrder = async (req, res) => {
    try {
      const { items, shippingAddress } = req.body;
      const customerId = req.user.id;
      
      const order = await this.orderService.createOrder({
        customerId,
        items,
        shippingAddress
      });
      
      res.status(201).json({
        success: true,
        data: order.toJSON()
      });
    } catch (error) {
      this._handleError(error, res);
    }
  }

  submitOrder = async (req, res) => {
    try {
      const { order, pricing } = await this.orderService.submitOrder(req.params.id);
      
      res.json({
        success: true,
        data: {
          order: order.toJSON(),
          pricing
        }
      });
    } catch (error) {
      this._handleError(error, res);
    }
  }

  cancelOrder = async (req, res) => {
    try {
      const order = await this.orderService.cancelOrder(
        req.params.id,
        req.body.reason
      );
      
      res.json({
        success: true,
        data: order.toJSON()
      });
    } catch (error) {
      this._handleError(error, res);
    }
  }

  _handleError(error, res) {
    // Map domain errors to HTTP status codes
    if (error.message.includes('not found')) {
      return res.status(404).json({ success: false, error: error.message });
    }
    if (error.message.includes('Cannot') || 
        error.message.includes('Insufficient') ||
        error.message.includes('Invalid')) {
      return res.status(400).json({ success: false, error: error.message });
    }
    
    console.error('Unexpected error:', error);
    res.status(500).json({ success: false, error: 'Internal server error' });
  }
}

module.exports = OrderController;
```

---

## ขั้นตอนที่ 978: CQRS ด้วย Command Bus

```javascript
// shared/CommandBus.js

class CommandBus {
  constructor() {
    this.handlers = new Map();
    this.middlewares = [];
  }

  register(commandType, handler) {
    this.handlers.set(commandType.name, handler);
    return this;
  }

  use(middleware) {
    this.middlewares.push(middleware);
    return this;
  }

  async dispatch(command) {
    const handlerName = command.constructor.name;
    const handler = this.handlers.get(handlerName);
    
    if (!handler) {
      throw new Error(`No handler registered for: ${handlerName}`);
    }
    
    // รัน middlewares
    let index = 0;
    const next = async () => {
      if (index < this.middlewares.length) {
        const middleware = this.middlewares[index++];
        return middleware(command, next);
      }
      return handler.handle(command);
    };
    
    return next();
  }
}

// Middleware สำหรับ Command Bus
const loggingMiddleware = async (command, next) => {
  const start = Date.now();
  console.log(`Executing command: ${command.constructor.name}`);
  
  try {
    const result = await next();
    console.log(`Command ${command.constructor.name} completed in ${Date.now() - start}ms`);
    return result;
  } catch (error) {
    console.error(`Command ${command.constructor.name} failed:`, error.message);
    throw error;
  }
};

const validationMiddleware = async (command, next) => {
  if (typeof command.validate === 'function') {
    command.validate();
  }
  return next();
};

// Setup
const commandBus = new CommandBus();
commandBus.use(loggingMiddleware);
commandBus.use(validationMiddleware);

const { CreateOrderHandler } = require('./contexts/order-context/application/commandHandlers/CreateOrderHandler');
const CreateOrderCommand = require('./contexts/order-context/application/commands/CreateOrderCommand');

commandBus.register(CreateOrderCommand, new CreateOrderHandler());

module.exports = commandBus;
```

---

## ขั้นตอนที่ 979: DDD Integration Test

```javascript
// tests/integration/order-context.test.js
const mongoose = require('mongoose');
const OrderApplicationService = require('../../contexts/order-context/application/services/OrderApplicationService');
const { Order } = require('../../contexts/order-context/domain/aggregates/Order');

describe('Order Context Integration Tests', () => {
  let orderService;

  beforeAll(async () => {
    await mongoose.connect(process.env.TEST_MONGODB_URI);
    orderService = new OrderApplicationService();
  });

  afterAll(async () => {
    await mongoose.disconnect();
  });

  describe('Create Order', () => {
    it('should create a valid order', async () => {
      const order = await orderService.createOrder({
        customerId: 'customer-123',
        items: [
          { productId: 'prod-1', quantity: 2 },
          { productId: 'prod-2', quantity: 1 }
        ],
        shippingAddress: {
          street: '123 Main St',
          city: 'Bangkok',
          postalCode: '10100'
        }
      });
      
      expect(order).toBeInstanceOf(Order);
      expect(order.status).toBe('draft');
      expect(order.lines.length).toBeGreaterThan(0);
    });

    it('should emit OrderCreated domain event', async () => {
      const order = await orderService.createOrder({
        customerId: 'customer-456',
        items: [{ productId: 'prod-1', quantity: 1 }],
        shippingAddress: {
          street: '456 Test Ave',
          city: 'Chiang Mai',
          postalCode: '50000'
        }
      });
      
      // ตรวจสอบว่า events ถูก publish
      // (ในกรณีจริงจะ verify จาก event store)
      expect(order.id).toBeTruthy();
    });
  });

  describe('Submit Order', () => {
    it('should submit draft order', async () => {
      // สร้าง order ก่อน
      const order = await orderService.createOrder({
        customerId: 'customer-789',
        items: [{ productId: 'prod-1', quantity: 1 }],
        shippingAddress: {
          street: '789 Test Rd',
          city: 'Pattaya',
          postalCode: '20150'
        }
      });
      
      // Submit
      const { order: submittedOrder } = await orderService.submitOrder(order.id);
      
      expect(submittedOrder.status).toBe('submitted');
    });
  });
});
```

---

## ขั้นตอนที่ 980: DDD Best Practices

```javascript
// patterns/dddBestPractices.js

/*
  DDD Best Practices:
  
  1. Entities vs Value Objects
     - Entity: ใช้เมื่อ identity สำคัญ (User, Order)
     - Value Object: ใช้เมื่อ attributes สำคัญ (Money, Address)
     - Value Objects ควรเป็น immutable
  
  2. Aggregate Design
     - กำหนด Aggregate Root ที่ชัดเจน
     - อ้างอิง Aggregates อื่นด้วย ID เท่านั้น
     - Transaction ไม่ข้าม Aggregate boundaries
     - เก็บ Aggregates ให้เล็ก (Small Aggregates)
  
  3. Repositories
     - Interface ใน domain layer
     - Implementation ใน infrastructure layer
     - คืน domain objects เสมอ (ไม่ใช่ database models)
  
  4. Domain Services
     - ใช้เมื่อ logic ไม่ belong กับ entity ใด entity หนึ่ง
     - Stateless
  
  5. Anti-Corruption Layer
     - ใช้เมื่อ integrate กับ external systems
     - แปลง foreign model ไปยัง domain model
*/

// ตัวอย่าง: ไม่อ้างอิง Aggregate โดยตรง ใช้ ID แทน
class OrderLine {
  constructor({ productId }) { // ใช้ ID ไม่ใช่ Product object
    this.productId = productId; // ✓ Reference by ID
    // this.product = product;  // ✗ ไม่ควรทำ
  }
}

// ตัวอย่าง: Aggregate Root ควบคุม children
class Order {
  addLine(line) {
    // Order Root ควบคุมการเพิ่ม OrderLine
    this._lines.push(line);
  }
  
  // ✗ ไม่ควรทำ: ให้ external code แก้ไข children โดยตรง
  // get lines() { return this._lines; } // จะ allow mutation
  
  // ✓ ควรทำ: Return copy เพื่อป้องกัน mutation
  get lines() { return [...this._lines]; }
}
```

---

## แบบฝึกหัดท้ายบท

### แบบฝึกหัดที่ 1: Customer Aggregate
สร้าง Customer aggregate ด้วย value objects: Email, PhoneNumber, CustomerTier

### แบบฝึกหัดที่ 2: Payment Context
สร้าง Payment bounded context แยกจาก Order context ด้วย Anti-Corruption Layer

### แบบฝึกหัดที่ 3: Specification Pattern
Implement complex product search โดยใช้ Specification pattern

### แบบฝึกหัดที่ 4: Domain Event Testing
เขียน tests ที่ตรวจสอบว่า domain events ถูก raised อย่างถูกต้อง

### แบบฝึกหัดที่ 5: Bounded Context Integration
เชื่อม Order context กับ Inventory context ผ่าน Domain Events

---

## สรุป

DDD ช่วยให้ code สะท้อน business domain อย่างชัดเจน ทำให้นักพัฒนาและ business stakeholders สื่อสารด้วยภาษาเดียวกัน Value Objects, Entities, Aggregates และ Bounded Contexts เป็น building blocks หลักที่ทำให้ระบบมีโครงสร้างที่ดีและบำรุงรักษาได้ง่าย
