# Part 65: Domain-Driven Design (DDD)
## ขั้นตอนที่ 641-650 จาก 1000

---

## DDD คืออะไร?

Domain-Driven Design (DDD) เป็น approach ในการ design software โดยเน้นที่ domain model (business logic) เป็นศูนย์กลาง ช่วยให้โค้ดสะท้อน business requirements ได้ชัดเจน

---

## 1. DDD Concepts

### Ubiquitous Language

```
Ubiquitous Language = ภาษาร่วมระหว่าง developer และ domain expert
ใช้ terms เดียวกันทั้งในโค้ดและการสนทนา

ตัวอย่าง e-commerce:
- Order (ไม่ใช่ Transaction หรือ Purchase)
- Customer (ไม่ใช่ User หรือ Person)
- Product (ไม่ใช่ Item หรือ Goods)
- Invoice (ไม่ใช่ Bill หรือ Receipt)
```

### Bounded Context

```
Bounded Context = ขอบเขตที่ Ubiquitous Language มีความหมายเดียวกัน

ตัวอย่าง:
Billing Context: Customer = person who pays
Sales Context: Customer = person who buys
Support Context: Customer = person who needs help
```

---

## 2. Entities

```javascript
// entities/order.entity.js
const { v4: uuidv4 } = require('uuid');
const OrderItem = require('./order-item.entity');
const Money = require('../value-objects/money.vo');

class Order {
  constructor({ id, customerId, items = [], status = 'pending' }) {
    this._id = id || uuidv4();
    this._customerId = customerId;
    this._items = items.map(i => new OrderItem(i));
    this._status = status;
    this._createdAt = new Date();
    this._updatedAt = new Date();
  }

  // Identity
  get id() { return this._id; }
  get customerId() { return this._customerId; }
  get status() { return this._status; }
  get items() { return [...this._items]; }
  get createdAt() { return this._createdAt; }

  // Business logic
  get total() {
    return this._items.reduce(
      (sum, item) => sum.add(item.subtotal),
      Money.zero()
    );
  }

  addItem(productId, name, price, quantity) {
    if (this._status !== 'pending') {
      throw new Error('Cannot add items to non-pending order');
    }

    const existingItem = this._items.find(i => i.productId === productId);
    
    if (existingItem) {
      existingItem.increaseQuantity(quantity);
    } else {
      this._items.push(new OrderItem({ productId, name, price, quantity }));
    }

    this._updatedAt = new Date();
  }

  removeItem(productId) {
    if (this._status !== 'pending') {
      throw new Error('Cannot remove items from non-pending order');
    }

    const index = this._items.findIndex(i => i.productId === productId);
    if (index === -1) throw new Error('Item not found in order');

    this._items.splice(index, 1);
    this._updatedAt = new Date();
  }

  submit() {
    if (this._status !== 'pending') {
      throw new Error(`Cannot submit order in ${this._status} status`);
    }
    if (this._items.length === 0) {
      throw new Error('Cannot submit empty order');
    }

    this._status = 'submitted';
    this._updatedAt = new Date();
  }

  cancel(reason) {
    const cancellableStatuses = ['pending', 'submitted'];
    if (!cancellableStatuses.includes(this._status)) {
      throw new Error(`Cannot cancel order in ${this._status} status`);
    }

    this._status = 'cancelled';
    this._cancelReason = reason;
    this._updatedAt = new Date();
  }

  equals(other) {
    return other instanceof Order && this._id === other._id;
  }

  toJSON() {
    return {
      id: this._id,
      customerId: this._customerId,
      items: this._items.map(i => i.toJSON()),
      status: this._status,
      total: this.total.toJSON(),
      createdAt: this._createdAt,
      updatedAt: this._updatedAt
    };
  }
}

module.exports = Order;
```

---

## 3. Value Objects

```javascript
// value-objects/money.vo.js
class Money {
  constructor(amount, currency = 'THB') {
    if (typeof amount !== 'number' || isNaN(amount)) {
      throw new Error('Invalid money amount');
    }
    if (amount < 0) {
      throw new Error('Money cannot be negative');
    }

    this._amount = Math.round(amount * 100) / 100; // Round to 2 decimal places
    this._currency = currency;

    Object.freeze(this); // Immutable
  }

  get amount() { return this._amount; }
  get currency() { return this._currency; }

  add(other) {
    this.assertSameCurrency(other);
    return new Money(this._amount + other._amount, this._currency);
  }

  subtract(other) {
    this.assertSameCurrency(other);
    const result = this._amount - other._amount;
    if (result < 0) throw new Error('Insufficient amount');
    return new Money(result, this._currency);
  }

  multiply(factor) {
    return new Money(this._amount * factor, this._currency);
  }

  isGreaterThan(other) {
    this.assertSameCurrency(other);
    return this._amount > other._amount;
  }

  isLessThan(other) {
    this.assertSameCurrency(other);
    return this._amount < other._amount;
  }

  equals(other) {
    return other instanceof Money &&
      this._amount === other._amount &&
      this._currency === other._currency;
  }

  assertSameCurrency(other) {
    if (this._currency !== other._currency) {
      throw new Error(`Cannot operate on different currencies: ${this._currency} vs ${other._currency}`);
    }
  }

  static zero(currency = 'THB') {
    return new Money(0, currency);
  }

  toString() {
    return `${this._currency} ${this._amount.toFixed(2)}`;
  }

  toJSON() {
    return { amount: this._amount, currency: this._currency };
  }
}

// value-objects/email.vo.js
class Email {
  constructor(value) {
    const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
    if (!emailRegex.test(value)) {
      throw new Error(`Invalid email: ${value}`);
    }
    this._value = value.toLowerCase().trim();
    Object.freeze(this);
  }

  get value() { return this._value; }

  equals(other) {
    return other instanceof Email && this._value === other._value;
  }

  toString() { return this._value; }
}

// value-objects/address.vo.js
class Address {
  constructor({ street, city, state, zipCode, country }) {
    if (!street || !city || !country) {
      throw new Error('Address requires street, city, and country');
    }

    this._street = street;
    this._city = city;
    this._state = state;
    this._zipCode = zipCode;
    this._country = country;

    Object.freeze(this);
  }

  equals(other) {
    return other instanceof Address &&
      this._street === other._street &&
      this._city === other._city &&
      this._country === other._country;
  }

  toString() {
    return `${this._street}, ${this._city}, ${this._state} ${this._zipCode}, ${this._country}`;
  }

  toJSON() {
    return {
      street: this._street,
      city: this._city,
      state: this._state,
      zipCode: this._zipCode,
      country: this._country
    };
  }
}

module.exports = { Money, Email, Address };
```

---

## 4. Aggregates

```javascript
// aggregates/shopping-cart.aggregate.js
class ShoppingCart {
  constructor(customerId) {
    this._id = require('uuid').v4();
    this._customerId = customerId;
    this._items = new Map(); // productId -> CartItem
    this._appliedCoupons = [];
    this._createdAt = new Date();
  }

  addItem(product, quantity) {
    if (quantity <= 0) throw new Error('Quantity must be positive');
    if (!product.isAvailable()) throw new Error(`Product ${product.name} is not available`);

    const productId = product.id;
    
    if (this._items.has(productId)) {
      const item = this._items.get(productId);
      item.quantity += quantity;
    } else {
      this._items.set(productId, {
        productId,
        name: product.name,
        price: product.price,
        quantity
      });
    }
  }

  removeItem(productId) {
    if (!this._items.has(productId)) {
      throw new Error('Item not in cart');
    }
    this._items.delete(productId);
  }

  updateQuantity(productId, quantity) {
    if (!this._items.has(productId)) throw new Error('Item not in cart');
    
    if (quantity <= 0) {
      this._items.delete(productId);
    } else {
      this._items.get(productId).quantity = quantity;
    }
  }

  applyCoupon(coupon) {
    if (coupon.isExpired()) throw new Error('Coupon is expired');
    if (!coupon.isApplicableTo(this)) throw new Error('Coupon not applicable');
    if (this._appliedCoupons.find(c => c.code === coupon.code)) {
      throw new Error('Coupon already applied');
    }

    this._appliedCoupons.push(coupon);
  }

  get subtotal() {
    let total = 0;
    this._items.forEach(item => {
      total += item.price * item.quantity;
    });
    return total;
  }

  get discount() {
    return this._appliedCoupons.reduce((sum, coupon) => {
      return sum + coupon.calculateDiscount(this.subtotal);
    }, 0);
  }

  get total() {
    return Math.max(0, this.subtotal - this.discount);
  }

  get items() {
    return Array.from(this._items.values());
  }

  get isEmpty() {
    return this._items.size === 0;
  }

  checkout() {
    if (this.isEmpty) throw new Error('Cannot checkout empty cart');
    
    return {
      customerId: this._customerId,
      items: this.items,
      subtotal: this.subtotal,
      discount: this.discount,
      total: this.total,
      coupons: this._appliedCoupons.map(c => c.code)
    };
  }

  clear() {
    this._items.clear();
    this._appliedCoupons = [];
  }
}

module.exports = ShoppingCart;
```

---

## 5. Domain Services

```javascript
// services/domain/pricing.service.js
class PricingService {
  constructor(productRepository, promotionRepository) {
    this.productRepository = productRepository;
    this.promotionRepository = promotionRepository;
  }

  async calculateOrderTotal(items) {
    let total = 0;
    const enrichedItems = [];

    for (const item of items) {
      const product = await this.productRepository.findById(item.productId);
      if (!product) throw new Error(`Product ${item.productId} not found`);

      const price = await this.getEffectivePrice(product, item.quantity);
      
      enrichedItems.push({
        ...item,
        price,
        subtotal: price * item.quantity
      });

      total += price * item.quantity;
    }

    return { items: enrichedItems, total };
  }

  async getEffectivePrice(product, quantity) {
    // ตรวจสอบ volume pricing
    const promotions = await this.promotionRepository.findActive();
    
    for (const promo of promotions) {
      if (promo.appliesTo(product) && quantity >= promo.minQuantity) {
        return promo.applyDiscount(product.price);
      }
    }

    return product.price;
  }
}

// services/domain/inventory.service.js
class InventoryService {
  constructor(inventoryRepository) {
    this.inventoryRepository = inventoryRepository;
  }

  async checkAvailability(items) {
    const results = await Promise.all(
      items.map(async (item) => {
        const inventory = await this.inventoryRepository.findByProduct(item.productId);
        return {
          productId: item.productId,
          requested: item.quantity,
          available: inventory?.quantity || 0,
          isAvailable: (inventory?.quantity || 0) >= item.quantity
        };
      })
    );

    const unavailableItems = results.filter(r => !r.isAvailable);
    
    return {
      allAvailable: unavailableItems.length === 0,
      items: results,
      unavailableItems
    };
  }

  async reserve(items) {
    const availability = await this.checkAvailability(items);
    
    if (!availability.allAvailable) {
      const names = availability.unavailableItems.map(i => i.productId).join(', ');
      throw new Error(`Insufficient inventory for: ${names}`);
    }

    // Reserve items (เพิ่ม reserved count)
    await Promise.all(
      items.map(item =>
        this.inventoryRepository.reserve(item.productId, item.quantity)
      )
    );
  }

  async release(items) {
    await Promise.all(
      items.map(item =>
        this.inventoryRepository.release(item.productId, item.quantity)
      )
    );
  }
}

module.exports = { PricingService, InventoryService };
```

---

## 6. Repositories

```javascript
// repositories/order.repository.js
class OrderRepository {
  constructor(db) {
    this.collection = db.collection('orders');
  }

  async findById(id) {
    const data = await this.collection.findOne({ _id: id });
    if (!data) return null;
    return this.toDomain(data);
  }

  async findByCustomer(customerId, options = {}) {
    const { limit = 20, offset = 0, status } = options;
    const filter = { customerId };
    if (status) filter.status = status;

    const [docs, total] = await Promise.all([
      this.collection.find(filter)
        .sort({ createdAt: -1 })
        .skip(offset)
        .limit(limit)
        .toArray(),
      this.collection.countDocuments(filter)
    ]);

    return {
      data: docs.map(d => this.toDomain(d)),
      total,
      limit,
      offset
    };
  }

  async save(order) {
    const data = this.toData(order);
    
    await this.collection.replaceOne(
      { _id: order.id },
      data,
      { upsert: true }
    );

    return order;
  }

  async delete(id) {
    await this.collection.deleteOne({ _id: id });
  }

  // Mapping ระหว่าง Domain Object และ Database Document
  toDomain(data) {
    const Order = require('../entities/order.entity');
    return new Order({
      id: data._id,
      customerId: data.customerId,
      items: data.items,
      status: data.status
    });
  }

  toData(order) {
    return {
      _id: order.id,
      customerId: order.customerId,
      items: order.items,
      status: order.status,
      total: order.total.toJSON(),
      createdAt: order.createdAt,
      updatedAt: new Date()
    };
  }
}

module.exports = OrderRepository;
```

---

## แบบฝึกหัด

### ระดับ 1: พื้นฐาน
สร้าง Value Objects:
- Money, Email, PhoneNumber
- Address, DateRange
- ทดสอบ immutability และ equality

### ระดับ 2: กลาง
สร้าง Order Domain:
- Order Entity
- OrderItem Value Object
- OrderRepository
- OrderService

### ระดับ 3: ขั้นสูง
สร้าง complete e-commerce domain:
- Multiple Bounded Contexts
- Domain Services
- Application Services
- Aggregates

---

## สรุป

DDD ช่วยให้โค้ดสะท้อน business domain ได้ชัดเจน ทำให้ communicate กับ stakeholders ง่ายขึ้น และโค้ดยืดหยุ่นต่อการเปลี่ยนแปลง business rules กุญแจสำคัญคือ Ubiquitous Language และการแบ่ง Bounded Contexts ที่ถูกต้อง

> ขั้นตอนต่อไป: Part 66 - Clean Architecture
