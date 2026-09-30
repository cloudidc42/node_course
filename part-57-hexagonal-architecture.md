# Part 57 | ขั้นตอนที่ 1001-1020 จาก 1000

## Hexagonal Architecture (Ports & Adapters)

Hexagonal Architecture หรือ Ports & Adapters เป็น pattern ที่แยก application core ออกจาก external world โดยใช้ Ports (interfaces) และ Adapters (implementations)

---

## ขั้นตอนที่ 1001: Hexagonal Architecture คืออะไร

```
                    ┌─────────────────────────────┐
  UI Adapter        │                             │   DB Adapter
  (REST/GraphQL) ───┤    Application Core         ├───  (MongoDB/Postgres)
                    │    (Domain Logic)           │
  Test Adapter  ────┤                             ├───  Email Adapter
  (Jest/Mocha)      │    Ports (Interfaces)       │     (SendGrid/SMTP)
                    └─────────────────────────────┘
                         ↑             ↑
                    Driving Port   Driven Port
                    (Primary)      (Secondary)
```

**Driving Ports (Primary/Inbound)**: พวก UI, REST, CLI, Tests ที่ drive application
**Driven Ports (Secondary/Outbound)**: Database, Email, SMS ที่ถูก application drive

---

## ขั้นตอนที่ 1002: Application Core

```javascript
// core/domain/Product.js

class Product {
  constructor({ id, name, price, stock, categoryId }) {
    this.id = id;
    this.name = name;
    this.price = price;
    this.stock = stock;
    this.categoryId = categoryId;
    this._isActive = true;
  }

  get isActive() { return this._isActive; }
  get isAvailable() { return this._isActive && this.stock > 0; }

  decreaseStock(quantity) {
    if (quantity > this.stock) {
      throw new Error(`Cannot decrease stock: ${this.stock} < ${quantity}`);
    }
    this.stock -= quantity;
    return this;
  }

  increaseStock(quantity) {
    this.stock += quantity;
    return this;
  }

  updatePrice(newPrice) {
    if (newPrice < 0) throw new Error('Price cannot be negative');
    this.price = newPrice;
    return this;
  }

  deactivate() {
    this._isActive = false;
    return this;
  }
}

module.exports = Product;
```

---

## ขั้นตอนที่ 1003: Driven Ports (Outbound Interfaces)

```javascript
// core/ports/driven/ProductRepositoryPort.js

class ProductRepositoryPort {
  async findById(id) { throw new Error('Not implemented'); }
  async findAll(filters, pagination) { throw new Error('Not implemented'); }
  async save(product) { throw new Error('Not implemented'); }
  async update(id, changes) { throw new Error('Not implemented'); }
  async delete(id) { throw new Error('Not implemented'); }
  async findLowStock(threshold) { throw new Error('Not implemented'); }
}

// core/ports/driven/NotificationPort.js
class NotificationPort {
  async sendLowStockAlert(product, currentStock) { throw new Error('Not implemented'); }
  async sendOrderConfirmation(order) { throw new Error('Not implemented'); }
}

// core/ports/driven/CachePort.js
class CachePort {
  async get(key) { throw new Error('Not implemented'); }
  async set(key, value, ttl) { throw new Error('Not implemented'); }
  async delete(key) { throw new Error('Not implemented'); }
  async deletePattern(pattern) { throw new Error('Not implemented'); }
}

module.exports = { ProductRepositoryPort, NotificationPort, CachePort };
```

---

## ขั้นตอนที่ 1004: Driving Ports (Inbound Interfaces)

```javascript
// core/ports/driving/ProductServicePort.js
// Primary port: defines what the application offers

class ProductServicePort {
  async createProduct(data) { throw new Error('Not implemented'); }
  async updateProduct(id, data) { throw new Error('Not implemented'); }
  async getProduct(id) { throw new Error('Not implemented'); }
  async listProducts(filters, pagination) { throw new Error('Not implemented'); }
  async deleteProduct(id) { throw new Error('Not implemented'); }
  async adjustStock(id, quantity) { throw new Error('Not implemented'); }
}

module.exports = ProductServicePort;
```

---

## ขั้นตอนที่ 1005: Application Core Service

```javascript
// core/application/ProductService.js

const ProductServicePort = require('../ports/driving/ProductServicePort');
const Product = require('../domain/Product');

class ProductService extends ProductServicePort {
  constructor({
    productRepository,
    notificationPort,
    cachePort
  }) {
    super();
    this.productRepository = productRepository;
    this.notificationPort = notificationPort;
    this.cachePort = cachePort;
    this.LOW_STOCK_THRESHOLD = 5;
  }

  async createProduct(data) {
    const product = new Product({
      id: require('crypto').randomUUID(),
      name: data.name,
      price: data.price,
      stock: data.stock || 0,
      categoryId: data.categoryId
    });
    
    await this.productRepository.save(product);
    await this.cachePort.deletePattern('products:*');
    
    return product;
  }

  async getProduct(id) {
    const cacheKey = `products:${id}`;
    
    const cached = await this.cachePort.get(cacheKey);
    if (cached) return cached;
    
    const product = await this.productRepository.findById(id);
    if (!product) throw new Error(`Product not found: ${id}`);
    
    await this.cachePort.set(cacheKey, product, 300);
    return product;
  }

  async listProducts(filters = {}, pagination = {}) {
    const cacheKey = `products:list:${JSON.stringify({ filters, pagination })}`;
    
    const cached = await this.cachePort.get(cacheKey);
    if (cached) return cached;
    
    const result = await this.productRepository.findAll(filters, pagination);
    
    await this.cachePort.set(cacheKey, result, 60);
    return result;
  }

  async adjustStock(id, quantity) {
    const product = await this.productRepository.findById(id);
    if (!product) throw new Error(`Product not found: ${id}`);
    
    if (quantity < 0) {
      product.decreaseStock(Math.abs(quantity));
    } else {
      product.increaseStock(quantity);
    }
    
    await this.productRepository.update(id, { stock: product.stock });
    await this.cachePort.delete(`products:${id}`);
    
    // Check low stock
    if (product.stock <= this.LOW_STOCK_THRESHOLD) {
      await this.notificationPort.sendLowStockAlert(product, product.stock);
    }
    
    return product;
  }

  async updateProduct(id, data) {
    const product = await this.productRepository.findById(id);
    if (!product) throw new Error(`Product not found: ${id}`);
    
    if (data.price !== undefined) product.updatePrice(data.price);
    if (data.name !== undefined) product.name = data.name;
    
    await this.productRepository.update(id, data);
    await this.cachePort.delete(`products:${id}`);
    await this.cachePort.deletePattern('products:list:*');
    
    return product;
  }

  async deleteProduct(id) {
    const product = await this.productRepository.findById(id);
    if (!product) throw new Error(`Product not found: ${id}`);
    
    product.deactivate();
    await this.productRepository.delete(id);
    await this.cachePort.delete(`products:${id}`);
    
    return true;
  }
}

module.exports = ProductService;
```

---

## ขั้นตอนที่ 1006: Driven Adapters - Database

```javascript
// adapters/driven/MongoProductRepository.js

const { ProductRepositoryPort } = require('../../core/ports/driven/ProductRepositoryPort');
const Product = require('../../core/domain/Product');
const ProductModel = require('../database/ProductModel');

class MongoProductRepository extends ProductRepositoryPort {
  async findById(id) {
    const doc = await ProductModel.findOne({ _id: id, deletedAt: null }).lean();
    return doc ? this._toDomain(doc) : null;
  }

  async findAll(filters = {}, pagination = {}) {
    const { page = 1, limit = 20 } = pagination;
    const query = this._buildQuery(filters);
    
    const [docs, total] = await Promise.all([
      ProductModel.find(query)
        .skip((page - 1) * limit)
        .limit(limit)
        .sort({ createdAt: -1 })
        .lean(),
      ProductModel.countDocuments(query)
    ]);
    
    return {
      items: docs.map(d => this._toDomain(d)),
      total,
      page,
      totalPages: Math.ceil(total / limit)
    };
  }

  async save(product) {
    await ProductModel.create({
      _id: product.id,
      name: product.name,
      price: product.price,
      stock: product.stock,
      categoryId: product.categoryId,
      isActive: product.isActive
    });
    return product;
  }

  async update(id, changes) {
    await ProductModel.findByIdAndUpdate(id, {
      $set: { ...changes, updatedAt: new Date() }
    });
  }

  async delete(id) {
    await ProductModel.findByIdAndUpdate(id, {
      $set: { deletedAt: new Date(), isActive: false }
    });
  }

  async findLowStock(threshold = 5) {
    const docs = await ProductModel.find({
      stock: { $lte: threshold },
      isActive: true
    }).lean();
    return docs.map(d => this._toDomain(d));
  }

  _toDomain(doc) {
    return new Product({
      id: doc._id.toString(),
      name: doc.name,
      price: doc.price,
      stock: doc.stock,
      categoryId: doc.categoryId?.toString()
    });
  }

  _buildQuery(filters) {
    const query = { deletedAt: null, isActive: true };
    if (filters.categoryId) query.categoryId = filters.categoryId;
    if (filters.minPrice) query.price = { $gte: filters.minPrice };
    if (filters.maxPrice) query.price = { ...query.price, $lte: filters.maxPrice };
    if (filters.search) query.name = new RegExp(filters.search, 'i');
    return query;
  }
}

module.exports = MongoProductRepository;
```

---

## ขั้นตอนที่ 1007: Driven Adapters - Cache

```javascript
// adapters/driven/RedisCache.js

const Redis = require('ioredis');
const { CachePort } = require('../../core/ports/driven/CachePort');

class RedisCache extends CachePort {
  constructor(connectionString) {
    super();
    this.client = new Redis(connectionString);
    
    this.client.on('error', err => {
      console.error('Redis error:', err);
    });
  }

  async get(key) {
    const value = await this.client.get(key);
    if (!value) return null;
    
    try {
      return JSON.parse(value);
    } catch {
      return value;
    }
  }

  async set(key, value, ttl) {
    const serialized = JSON.stringify(value);
    
    if (ttl) {
      await this.client.setex(key, ttl, serialized);
    } else {
      await this.client.set(key, serialized);
    }
  }

  async delete(key) {
    await this.client.del(key);
  }

  async deletePattern(pattern) {
    const keys = await this.client.keys(pattern);
    if (keys.length > 0) {
      await this.client.del(...keys);
    }
  }

  async disconnect() {
    await this.client.quit();
  }
}

// In-memory cache สำหรับ development/testing
class InMemoryCache extends CachePort {
  constructor() {
    super();
    this.store = new Map();
    this.expiries = new Map();
  }

  async get(key) {
    const expiry = this.expiries.get(key);
    if (expiry && Date.now() > expiry) {
      this.store.delete(key);
      this.expiries.delete(key);
      return null;
    }
    return this.store.get(key) || null;
  }

  async set(key, value, ttl) {
    this.store.set(key, value);
    if (ttl) {
      this.expiries.set(key, Date.now() + ttl * 1000);
    }
  }

  async delete(key) {
    this.store.delete(key);
    this.expiries.delete(key);
  }

  async deletePattern(pattern) {
    const regex = new RegExp('^' + pattern.replace(/\*/g, '.*') + '$');
    for (const key of this.store.keys()) {
      if (regex.test(key)) {
        this.store.delete(key);
        this.expiries.delete(key);
      }
    }
  }
}

module.exports = { RedisCache, InMemoryCache };
```

---

## ขั้นตอนที่ 1008: Driven Adapters - Notification

```javascript
// adapters/driven/EmailNotification.js

const { NotificationPort } = require('../../core/ports/driven/NotificationPort');
const nodemailer = require('nodemailer');

class EmailNotification extends NotificationPort {
  constructor(config) {
    super();
    this.transporter = nodemailer.createTransport(config);
    this.fromEmail = config.from;
    this.adminEmail = config.adminEmail;
  }

  async sendLowStockAlert(product, currentStock) {
    await this.transporter.sendMail({
      from: this.fromEmail,
      to: this.adminEmail,
      subject: `Low Stock Alert: ${product.name}`,
      html: `
        <h2>Low Stock Warning</h2>
        <p>Product <strong>${product.name}</strong> is running low on stock.</p>
        <p>Current stock: <strong>${currentStock}</strong></p>
        <p>Please reorder soon.</p>
      `
    });
  }

  async sendOrderConfirmation(order) {
    // ส่ง email ยืนยัน order
    await this.transporter.sendMail({
      from: this.fromEmail,
      to: order.customerEmail,
      subject: `Order Confirmation #${order.id}`,
      html: `<p>Your order has been confirmed.</p>`
    });
  }
}

// Slack Notification (alternative adapter)
class SlackNotification extends NotificationPort {
  constructor(webhookUrl) {
    super();
    this.webhookUrl = webhookUrl;
  }

  async sendLowStockAlert(product, currentStock) {
    const https = require('https');
    const payload = JSON.stringify({
      text: `*Low Stock Alert*: ${product.name} - Only ${currentStock} remaining!`
    });
    
    // ส่งผ่าน Slack webhook
    const url = new URL(this.webhookUrl);
    const options = {
      hostname: url.hostname,
      path: url.pathname,
      method: 'POST',
      headers: { 'Content-Type': 'application/json' }
    };
    
    return new Promise((resolve, reject) => {
      const req = https.request(options, res => {
        res.on('end', resolve);
        res.resume();
      });
      req.on('error', reject);
      req.write(payload);
      req.end();
    });
  }

  async sendOrderConfirmation(order) {
    // Slack notification สำหรับ order
  }
}

module.exports = { EmailNotification, SlackNotification };
```

---

## ขั้นตอนที่ 1009: Driving Adapters - REST API

```javascript
// adapters/driving/rest/ProductController.js

const ProductServicePort = require('../../../core/ports/driving/ProductServicePort');

class ProductRESTAdapter {
  constructor(productService) {
    if (!(productService instanceof ProductServicePort)) {
      throw new Error('productService must implement ProductServicePort');
    }
    this.productService = productService;
  }

  listProducts = async (req, res) => {
    try {
      const filters = {
        categoryId: req.query.categoryId,
        search: req.query.q,
        minPrice: req.query.minPrice ? Number(req.query.minPrice) : undefined,
        maxPrice: req.query.maxPrice ? Number(req.query.maxPrice) : undefined
      };
      
      const pagination = {
        page: parseInt(req.query.page) || 1,
        limit: Math.min(parseInt(req.query.limit) || 20, 100)
      };
      
      const result = await this.productService.listProducts(filters, pagination);
      
      res.json({
        success: true,
        data: result.items,
        meta: {
          total: result.total,
          page: result.page,
          totalPages: result.totalPages
        }
      });
    } catch (error) {
      this._handleError(error, res);
    }
  }

  getProduct = async (req, res) => {
    try {
      const product = await this.productService.getProduct(req.params.id);
      res.json({ success: true, data: product });
    } catch (error) {
      this._handleError(error, res);
    }
  }

  createProduct = async (req, res) => {
    try {
      const product = await this.productService.createProduct(req.body);
      res.status(201).json({ success: true, data: product });
    } catch (error) {
      this._handleError(error, res);
    }
  }

  updateProduct = async (req, res) => {
    try {
      const product = await this.productService.updateProduct(
        req.params.id,
        req.body
      );
      res.json({ success: true, data: product });
    } catch (error) {
      this._handleError(error, res);
    }
  }

  adjustStock = async (req, res) => {
    try {
      const { quantity } = req.body;
      
      if (!quantity || isNaN(quantity)) {
        return res.status(400).json({ error: 'Invalid quantity' });
      }
      
      const product = await this.productService.adjustStock(
        req.params.id,
        Number(quantity)
      );
      
      res.json({ success: true, data: product });
    } catch (error) {
      this._handleError(error, res);
    }
  }

  deleteProduct = async (req, res) => {
    try {
      await this.productService.deleteProduct(req.params.id);
      res.json({ success: true });
    } catch (error) {
      this._handleError(error, res);
    }
  }

  _handleError(error, res) {
    if (error.message.includes('not found')) {
      return res.status(404).json({ error: error.message });
    }
    if (error.message.includes('Cannot') || error.message.includes('Invalid')) {
      return res.status(400).json({ error: error.message });
    }
    console.error(error);
    res.status(500).json({ error: 'Internal server error' });
  }
}

module.exports = ProductRESTAdapter;
```

---

## ขั้นตอนที่ 1010: Driving Adapters - GraphQL

```javascript
// adapters/driving/graphql/productResolvers.js

const productResolvers = {
  Query: {
    product: async (_, { id }, { productService }) => {
      return productService.getProduct(id);
    },
    
    products: async (_, { filters, pagination }, { productService }) => {
      return productService.listProducts(filters || {}, pagination || {});
    }
  },
  
  Mutation: {
    createProduct: async (_, { input }, { productService }) => {
      return productService.createProduct(input);
    },
    
    updateProduct: async (_, { id, input }, { productService }) => {
      return productService.updateProduct(id, input);
    },
    
    adjustStock: async (_, { id, quantity }, { productService }) => {
      return productService.adjustStock(id, quantity);
    },
    
    deleteProduct: async (_, { id }, { productService }) => {
      return productService.deleteProduct(id);
    }
  }
};

// GraphQL Schema
const typeDefs = `
  type Product {
    id: ID!
    name: String!
    price: Float!
    stock: Int!
    categoryId: ID
    isActive: Boolean!
    isAvailable: Boolean!
  }

  type ProductList {
    items: [Product!]!
    total: Int!
    page: Int!
    totalPages: Int!
  }

  input ProductFilters {
    categoryId: ID
    search: String
    minPrice: Float
    maxPrice: Float
  }

  input PaginationInput {
    page: Int
    limit: Int
  }

  input CreateProductInput {
    name: String!
    price: Float!
    stock: Int
    categoryId: ID
  }

  input UpdateProductInput {
    name: String
    price: Float
  }

  type Query {
    product(id: ID!): Product
    products(filters: ProductFilters, pagination: PaginationInput): ProductList!
  }

  type Mutation {
    createProduct(input: CreateProductInput!): Product!
    updateProduct(id: ID!, input: UpdateProductInput!): Product!
    adjustStock(id: ID!, quantity: Int!): Product!
    deleteProduct(id: ID!): Boolean!
  }
`;

module.exports = { typeDefs, productResolvers };
```

---

## ขั้นตอนที่ 1011: Driving Adapters - CLI

```javascript
// adapters/driving/cli/ProductCLI.js
const readline = require('readline');

class ProductCLI {
  constructor(productService) {
    this.productService = productService;
    this.rl = readline.createInterface({
      input: process.stdin,
      output: process.stdout
    });
  }

  async run() {
    console.log('Product Management CLI');
    
    const command = await this._prompt('Command (list/get/create/stock): ');
    
    switch (command) {
      case 'list':
        await this._listProducts();
        break;
      case 'get':
        await this._getProduct();
        break;
      case 'create':
        await this._createProduct();
        break;
      case 'stock':
        await this._adjustStock();
        break;
      default:
        console.log('Unknown command');
    }
    
    this.rl.close();
  }

  async _listProducts() {
    const result = await this.productService.listProducts();
    console.table(result.items.map(p => ({
      ID: p.id.substring(0, 8),
      Name: p.name,
      Price: p.price,
      Stock: p.stock
    })));
  }

  async _getProduct() {
    const id = await this._prompt('Product ID: ');
    const product = await this.productService.getProduct(id);
    console.log(product);
  }

  async _createProduct() {
    const name = await this._prompt('Name: ');
    const price = Number(await this._prompt('Price: '));
    const stock = Number(await this._prompt('Stock: '));
    
    const product = await this.productService.createProduct({ name, price, stock });
    console.log('Created:', product);
  }

  async _adjustStock() {
    const id = await this._prompt('Product ID: ');
    const quantity = Number(await this._prompt('Quantity (+/-): '));
    
    const product = await this.productService.adjustStock(id, quantity);
    console.log('Updated stock:', product.stock);
  }

  _prompt(question) {
    return new Promise(resolve => {
      this.rl.question(question, resolve);
    });
  }
}

module.exports = ProductCLI;
```

---

## ขั้นตอนที่ 1012: Composition Root

```javascript
// config/composition.js
// ประกอบ hexagon ทั้งหมด

const ProductService = require('../core/application/ProductService');
const MongoProductRepository = require('../adapters/driven/MongoProductRepository');
const { RedisCache, InMemoryCache } = require('../adapters/driven/RedisCache');
const { EmailNotification, SlackNotification } = require('../adapters/driven/EmailNotification');
const ProductRESTAdapter = require('../adapters/driving/rest/ProductController');

function compose(config) {
  // Driven Adapters (secondary)
  const productRepository = new MongoProductRepository();
  
  const cache = config.redisUrl
    ? new RedisCache(config.redisUrl)
    : new InMemoryCache();
  
  const notification = config.slackWebhook
    ? new SlackNotification(config.slackWebhook)
    : new EmailNotification({
        host: config.smtp.host,
        port: config.smtp.port,
        from: config.smtp.from,
        adminEmail: config.adminEmail
      });
  
  // Application Core
  const productService = new ProductService({
    productRepository,
    notificationPort: notification,
    cachePort: cache
  });
  
  // Driving Adapters (primary)
  const productRESTAdapter = new ProductRESTAdapter(productService);
  
  return {
    productService,     // ใช้สำหรับ GraphQL/CLI
    productRESTAdapter  // ใช้สำหรับ REST routes
  };
}

module.exports = compose;
```

---

## ขั้นตอนที่ 1013: Express Integration

```javascript
// adapters/driving/rest/routes.js

const express = require('express');

function createRoutes({ productRESTAdapter }) {
  const router = express.Router();
  
  // Product routes
  router.get('/products', productRESTAdapter.listProducts);
  router.get('/products/:id', productRESTAdapter.getProduct);
  router.post('/products', productRESTAdapter.createProduct);
  router.put('/products/:id', productRESTAdapter.updateProduct);
  router.patch('/products/:id/stock', productRESTAdapter.adjustStock);
  router.delete('/products/:id', productRESTAdapter.deleteProduct);
  
  return router;
}

module.exports = createRoutes;
```

```javascript
// app.js

const express = require('express');
const mongoose = require('mongoose');
const compose = require('./config/composition');
const createRoutes = require('./adapters/driving/rest/routes');

async function createApp(config) {
  await mongoose.connect(config.mongoUri);
  
  const app = express();
  app.use(express.json());
  
  const { productRESTAdapter } = compose(config);
  
  app.use('/api/v1', createRoutes({ productRESTAdapter }));
  
  return app;
}

module.exports = createApp;
```

---

## ขั้นตอนที่ 1014: Testing with Test Adapters

```javascript
// tests/adapters/driven/InMemoryProductRepository.js
// Test Double: In-memory implementation for testing

const { ProductRepositoryPort } = require('../../../core/ports/driven/ProductRepositoryPort');
const Product = require('../../../core/domain/Product');

class InMemoryProductRepository extends ProductRepositoryPort {
  constructor() {
    super();
    this.products = new Map();
  }

  async findById(id) {
    return this.products.get(id) || null;
  }

  async findAll(filters = {}, pagination = {}) {
    let items = Array.from(this.products.values()).filter(p => p.isActive);
    
    if (filters.categoryId) {
      items = items.filter(p => p.categoryId === filters.categoryId);
    }
    
    const { page = 1, limit = 20 } = pagination;
    const start = (page - 1) * limit;
    
    return {
      items: items.slice(start, start + limit),
      total: items.length,
      page,
      totalPages: Math.ceil(items.length / limit)
    };
  }

  async save(product) {
    this.products.set(product.id, product);
    return product;
  }

  async update(id, changes) {
    const product = this.products.get(id);
    if (product) {
      Object.assign(product, changes);
    }
  }

  async delete(id) {
    const product = this.products.get(id);
    if (product) product.deactivate();
  }

  async findLowStock(threshold) {
    return Array.from(this.products.values())
      .filter(p => p.stock <= threshold && p.isActive);
  }
  
  // Test helpers
  seed(products) {
    products.forEach(p => this.products.set(p.id, p));
    return this;
  }
  
  clear() {
    this.products.clear();
    return this;
  }
}

module.exports = InMemoryProductRepository;
```

```javascript
// tests/core/ProductService.test.js

const ProductService = require('../../core/application/ProductService');
const Product = require('../../core/domain/Product');
const InMemoryProductRepository = require('../adapters/driven/InMemoryProductRepository');
const { InMemoryCache } = require('../../adapters/driven/RedisCache');

describe('ProductService (Core)', () => {
  let productService;
  let repository;
  let cache;
  let notificationSpy;

  beforeEach(() => {
    repository = new InMemoryProductRepository();
    cache = new InMemoryCache();
    
    notificationSpy = {
      sendLowStockAlert: jest.fn().mockResolvedValue(undefined),
      sendOrderConfirmation: jest.fn().mockResolvedValue(undefined)
    };
    
    productService = new ProductService({
      productRepository: repository,
      notificationPort: notificationSpy,
      cachePort: cache
    });
  });

  test('should create product', async () => {
    const product = await productService.createProduct({
      name: 'Test Product',
      price: 99.99,
      stock: 10
    });
    
    expect(product.name).toBe('Test Product');
    expect(product.price).toBe(99.99);
    expect(product.stock).toBe(10);
    
    // ตรวจสอบว่าถูกบันทึก
    const saved = await repository.findById(product.id);
    expect(saved).toBeTruthy();
  });

  test('should send low stock alert when stock falls below threshold', async () => {
    const product = new Product({
      id: 'test-id',
      name: 'Low Stock Product',
      price: 50,
      stock: 6
    });
    
    await repository.save(product);
    
    // ลด stock ให้ต่ำกว่า threshold (5)
    await productService.adjustStock('test-id', -2);
    
    expect(notificationSpy.sendLowStockAlert).toHaveBeenCalledWith(
      expect.objectContaining({ id: 'test-id' }),
      4
    );
  });

  test('should cache product on get', async () => {
    const product = new Product({
      id: 'cache-test',
      name: 'Cached Product',
      price: 100,
      stock: 5
    });
    
    await repository.save(product);
    
    // First get: ดึงจาก DB
    await productService.getProduct('cache-test');
    
    // Second get: ควรมาจาก cache
    await productService.getProduct('cache-test');
    
    const cached = await cache.get('products:cache-test');
    expect(cached).toBeTruthy();
  });
});
```

---

## ขั้นตอนที่ 1015: Hexagonal Architecture Structure

```
src/
├── core/                           (Application Hexagon)
│   ├── domain/
│   │   ├── Product.js              (Domain entities)
│   │   ├── Order.js
│   │   └── errors/
│   │       └── DomainError.js
│   ├── application/
│   │   └── ProductService.js       (Application services)
│   └── ports/
│       ├── driving/                (Primary/Inbound ports)
│       │   └── ProductServicePort.js
│       └── driven/                 (Secondary/Outbound ports)
│           ├── ProductRepositoryPort.js
│           ├── NotificationPort.js
│           └── CachePort.js
│
├── adapters/                       (Outside the hexagon)
│   ├── driven/                     (Secondary adapters)
│   │   ├── MongoProductRepository.js
│   │   ├── RedisCache.js
│   │   └── EmailNotification.js
│   └── driving/                    (Primary adapters)
│       ├── rest/
│       │   ├── ProductController.js
│       │   └── routes.js
│       ├── graphql/
│       │   └── productResolvers.js
│       └── cli/
│           └── ProductCLI.js
│
├── config/
│   └── composition.js              (Wiring)
│
└── tests/
    ├── core/                       (Unit tests for core)
    │   └── ProductService.test.js
    ├── adapters/
    │   └── driven/
    │       └── InMemoryProductRepository.js (Test doubles)
    └── integration/                (Integration tests)
        └── ProductService.integration.test.js
```

---

## ขั้นตอนที่ 1016: Comparing Clean vs Hexagonal Architecture

```
Clean Architecture:
  - 4 layers ชัดเจน (Entities, Use Cases, Adapters, Frameworks)
  - Dependency Rule: inward only
  - เน้น layer separation

Hexagonal Architecture:
  - มีแนวคิด "inside" (application core) และ "outside" (adapters)
  - มีแนวคิด "driving" (inbound) และ "driven" (outbound) adapters
  - Ports เป็น interfaces ที่ชัดเจน
  - เหมาะกับการ test โดยใช้ Test Adapters

ความเหมือน:
  - ทั้งคู่แยก business logic ออกจาก infrastructure
  - ทั้งคู่ใช้ dependency inversion
  - ทั้งคู่ทดสอบได้ง่าย

ในทางปฏิบัติ: มักใช้ร่วมกัน
  - Clean Architecture สำหรับ layer structure
  - Hexagonal สำหรับ port/adapter naming convention
```

---

## ขั้นตอนที่ 1017: Event-Driven Hexagonal Architecture

```javascript
// core/ports/driven/EventBusPort.js

class EventBusPort {
  async publish(event) { throw new Error('Not implemented'); }
  async subscribe(eventType, handler) { throw new Error('Not implemented'); }
}

// adapters/driven/InProcessEventBus.js
class InProcessEventBus extends EventBusPort {
  constructor() {
    super();
    this.handlers = new Map();
  }

  async publish(event) {
    const handlers = this.handlers.get(event.type) || [];
    
    await Promise.all(handlers.map(handler => handler(event)));
  }

  async subscribe(eventType, handler) {
    if (!this.handlers.has(eventType)) {
      this.handlers.set(eventType, []);
    }
    this.handlers.get(eventType).push(handler);
  }
}

// core/application/ProductService.js (with events)
class ProductServiceWithEvents extends ProductService {
  constructor({ productRepository, notificationPort, cachePort, eventBus }) {
    super({ productRepository, notificationPort, cachePort });
    this.eventBus = eventBus;
  }

  async createProduct(data) {
    const product = await super.createProduct(data);
    
    await this.eventBus.publish({
      type: 'ProductCreated',
      data: { productId: product.id, name: product.name },
      occurredAt: new Date()
    });
    
    return product;
  }
}

module.exports = { EventBusPort, InProcessEventBus, ProductServiceWithEvents };
```

---

## ขั้นตอนที่ 1018: Async Driven Adapters

```javascript
// adapters/driven/SQSEventBus.js
// AWS SQS adapter สำหรับ production event bus

const { SQSClient, SendMessageCommand, ReceiveMessageCommand } = require('@aws-sdk/client-sqs');
const { EventBusPort } = require('./InProcessEventBus');

class SQSEventBus extends EventBusPort {
  constructor(config) {
    super();
    this.client = new SQSClient({ region: config.region });
    this.queueUrls = config.queueUrls; // { eventType: queueUrl }
  }

  async publish(event) {
    const queueUrl = this.queueUrls[event.type] || this.queueUrls.default;
    
    if (!queueUrl) {
      console.warn(`No queue URL for event type: ${event.type}`);
      return;
    }
    
    await this.client.send(new SendMessageCommand({
      QueueUrl: queueUrl,
      MessageBody: JSON.stringify(event),
      MessageAttributes: {
        EventType: {
          DataType: 'String',
          StringValue: event.type
        }
      }
    }));
  }

  async subscribe(eventType, handler) {
    // SQS: ต้องใช้ polling หรือ Lambda trigger แทน
    // ในกรณีนี้จะ poll
    const queueUrl = this.queueUrls[eventType];
    this._startPolling(queueUrl, handler);
  }

  _startPolling(queueUrl, handler) {
    const poll = async () => {
      const response = await this.client.send(new ReceiveMessageCommand({
        QueueUrl: queueUrl,
        MaxNumberOfMessages: 10,
        WaitTimeSeconds: 20 // Long polling
      }));
      
      for (const message of response.Messages || []) {
        try {
          const event = JSON.parse(message.Body);
          await handler(event);
          // Delete message after processing
          await this.client.send(new DeleteMessageCommand({
            QueueUrl: queueUrl,
            ReceiptHandle: message.ReceiptHandle
          }));
        } catch (error) {
          console.error('Failed to process message:', error);
        }
      }
      
      // Continue polling
      setImmediate(poll);
    };
    
    poll().catch(console.error);
  }
}

module.exports = SQSEventBus;
```

---

## ขั้นตอนที่ 1019: Feature Testing

```javascript
// tests/features/manage-products.feature.test.js
// BDD-style feature tests

const ProductService = require('../../core/application/ProductService');
const InMemoryProductRepository = require('../adapters/InMemoryProductRepository');
const { InMemoryCache } = require('../../adapters/driven/RedisCache');

describe('Feature: Manage Products', () => {
  let productService;
  
  beforeEach(() => {
    const repository = new InMemoryProductRepository();
    const cache = new InMemoryCache();
    const notification = { sendLowStockAlert: jest.fn(), sendOrderConfirmation: jest.fn() };
    
    productService = new ProductService({
      productRepository: repository,
      notificationPort: notification,
      cachePort: cache
    });
  });

  describe('Scenario: Create a new product', () => {
    it('Given I have product data, When I create it, Then it should be retrievable', async () => {
      // Given
      const productData = { name: 'Widget', price: 29.99, stock: 100 };
      
      // When
      const created = await productService.createProduct(productData);
      
      // Then
      const retrieved = await productService.getProduct(created.id);
      expect(retrieved.name).toBe('Widget');
      expect(retrieved.price).toBe(29.99);
    });
  });

  describe('Scenario: Stock management', () => {
    it('Given a product with stock, When I decrease it, Then stock updates', async () => {
      // Given
      const product = await productService.createProduct({
        name: 'Test', price: 10, stock: 50
      });
      
      // When
      await productService.adjustStock(product.id, -10);
      
      // Then
      const updated = await productService.getProduct(product.id);
      expect(updated.stock).toBe(40);
    });

    it('Given insufficient stock, When I decrease below zero, Then error thrown', async () => {
      const product = await productService.createProduct({
        name: 'Test', price: 10, stock: 5
      });
      
      await expect(
        productService.adjustStock(product.id, -10)
      ).rejects.toThrow('Cannot decrease stock');
    });
  });
});
```

---

## ขั้นตอนที่ 1020: Benefits Summary

```
Hexagonal Architecture Benefits:

1. Testability
   - Core logic test ได้โดยไม่ต้องใช้ DB หรือ external services
   - Test Adapters (In-Memory) แทน Real Adapters ได้

2. Technology Independence
   - เปลี่ยน MongoDB เป็น PostgreSQL ได้โดยแก้เฉพาะ adapter
   - เปลี่ยน REST เป็น GraphQL ได้โดยเพิ่ม driving adapter
   - เพิ่ม CLI โดยไม่แก้ core

3. Maintainability
   - Core logic อ่านง่ายไม่มี infrastructure code
   - Adapters แยกชัดเจน เปลี่ยนได้อิสระ

4. Parallel Development
   - Backend dev ทำ core
   - Infrastructure dev ทำ adapters พร้อมกัน

Trade-offs:
   - เพิ่ม abstractions = code มากขึ้น
   - อาจ over-engineered สำหรับโปรเจคเล็ก
   - Learning curve สูง
```

---

## แบบฝึกหัดท้ายบท

### แบบฝึกหัดที่ 1: Add Order Port
สร้าง OrderServicePort พร้อม implementation และ test adapters

### แบบฝึกหัดที่ 2: PostgreSQL Adapter
สร้าง PostgresProductRepository ที่ implements ProductRepositoryPort

### แบบฝึกหัดที่ 3: WebSocket Notification
สร้าง WebSocketNotification adapter สำหรับ real-time alerts

### แบบฝึกหัดที่ 4: Multiple Driving Adapters
เพิ่ม gRPC adapter ควบคู่กับ REST adapter

### แบบฝึกหัดที่ 5: Integration Tests
เขียน integration tests ที่ test ทั้ง core + real MongoDB adapter

---

## สรุป

Hexagonal Architecture ช่วยแยก application core ออกจาก external world โดยใช้ Ports (interfaces) เป็นตัวกลาง ทำให้สามารถเปลี่ยน adapters ได้โดยไม่กระทบ business logic และทดสอบ core ได้ง่ายโดยใช้ in-memory test adapters
