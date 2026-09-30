# Part 72: Real-World E-commerce API
## ขั้นตอนที่ 711-720 จาก 1000

---

## E-commerce API Design

สร้าง complete e-commerce backend ด้วย Node.js/Express.js รองรับ:
- Product catalog
- Shopping cart
- Order management
- Payment integration (Stripe)
- Inventory management

---

## 1. Database Schema

```javascript
// models/Product.js
const mongoose = require('mongoose');

const productSchema = new mongoose.Schema({
  name: { type: String, required: true, trim: true },
  slug: { type: String, unique: true },
  description: String,
  price: { type: Number, required: true, min: 0 },
  comparePrice: Number,
  sku: { type: String, unique: true, sparse: true },
  images: [{ url: String, alt: String }],
  category: { type: mongoose.Schema.Types.ObjectId, ref: 'Category' },
  tags: [String],
  attributes: mongoose.Schema.Types.Mixed,
  
  inventory: {
    quantity: { type: Number, default: 0, min: 0 },
    reserved: { type: Number, default: 0, min: 0 },
    available: { type: Number, default: 0 },
    trackInventory: { type: Boolean, default: true }
  },
  
  status: { 
    type: String, 
    enum: ['active', 'inactive', 'archived'],
    default: 'active'
  },
  
  seo: {
    title: String,
    description: String,
    keywords: [String]
  },
  
  ratings: {
    average: { type: Number, default: 0, min: 0, max: 5 },
    count: { type: Number, default: 0 }
  },
  
  weight: Number,
  dimensions: { length: Number, width: Number, height: Number }
}, {
  timestamps: true
});

productSchema.index({ name: 'text', description: 'text' });
productSchema.index({ category: 1, status: 1 });
productSchema.index({ slug: 1 });
productSchema.index({ 'inventory.available': 1 });

productSchema.virtual('inStock').get(function() {
  if (!this.inventory.trackInventory) return true;
  return this.inventory.available > 0;
});

productSchema.pre('save', function(next) {
  if (!this.slug) {
    this.slug = this.name
      .toLowerCase()
      .replace(/[^a-z0-9]+/g, '-')
      .replace(/^-|-$/g, '');
  }
  
  this.inventory.available = Math.max(0, 
    this.inventory.quantity - this.inventory.reserved
  );
  
  next();
});

module.exports = mongoose.model('Product', productSchema);
```

```javascript
// models/Order.js
const orderItemSchema = new mongoose.Schema({
  product: { type: mongoose.Schema.Types.ObjectId, ref: 'Product', required: true },
  name: String,
  sku: String,
  price: { type: Number, required: true },
  quantity: { type: Number, required: true, min: 1 },
  subtotal: Number,
  image: String
});

const orderSchema = new mongoose.Schema({
  orderNumber: { type: String, unique: true },
  user: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  guestEmail: String,
  
  items: [orderItemSchema],
  
  subtotal: { type: Number, required: true },
  discount: { type: Number, default: 0 },
  tax: { type: Number, default: 0 },
  shipping: { type: Number, default: 0 },
  total: { type: Number, required: true },
  
  couponCode: String,
  
  status: {
    type: String,
    enum: ['pending', 'confirmed', 'processing', 'shipped', 'delivered', 'cancelled', 'refunded'],
    default: 'pending'
  },
  
  payment: {
    method: String,
    provider: String,
    transactionId: String,
    status: { type: String, enum: ['pending', 'paid', 'failed', 'refunded'] },
    paidAt: Date,
    amount: Number
  },
  
  shippingAddress: {
    name: String,
    phone: String,
    street: String,
    city: String,
    state: String,
    zipCode: String,
    country: String
  },
  
  billingAddress: {
    name: String,
    street: String,
    city: String,
    state: String,
    zipCode: String,
    country: String
  },
  
  notes: String,
  
  statusHistory: [{
    status: String,
    timestamp: Date,
    note: String,
    changedBy: mongoose.Schema.Types.ObjectId
  }]
}, { timestamps: true });

orderSchema.pre('save', function(next) {
  if (!this.orderNumber) {
    const prefix = 'ORD';
    const timestamp = Date.now().toString(36).toUpperCase();
    const random = Math.random().toString(36).substring(2, 5).toUpperCase();
    this.orderNumber = `${prefix}-${timestamp}-${random}`;
  }
  next();
});

module.exports = mongoose.model('Order', orderSchema);
```

---

## 2. Product Catalog API

```javascript
// controllers/product.controller.js
const Product = require('../models/Product');
const Category = require('../models/Category');
const { uploadImage } = require('../services/storage.service');

exports.list = async (req, res) => {
  const {
    page = 1,
    limit = 20,
    category,
    minPrice,
    maxPrice,
    search,
    sort = '-createdAt',
    inStock,
    tags
  } = req.query;

  const filter = { status: 'active' };
  
  if (category) filter.category = category;
  if (inStock === 'true') filter['inventory.available'] = { $gt: 0 };
  if (tags) filter.tags = { $in: tags.split(',') };
  
  if (minPrice || maxPrice) {
    filter.price = {};
    if (minPrice) filter.price.$gte = parseFloat(minPrice);
    if (maxPrice) filter.price.$lte = parseFloat(maxPrice);
  }

  const query = search
    ? Product.find({ ...filter, $text: { $search: search } }, { score: { $meta: 'textScore' } })
        .sort({ score: { $meta: 'textScore' }, [sort.replace('-', '')]: sort.startsWith('-') ? -1 : 1 })
    : Product.find(filter).sort(sort);

  const [products, total] = await Promise.all([
    query.skip((page - 1) * limit).limit(parseInt(limit))
      .populate('category', 'name slug')
      .select('-__v'),
    Product.countDocuments(filter)
  ]);

  res.json({
    data: products,
    pagination: {
      page: parseInt(page),
      limit: parseInt(limit),
      total,
      pages: Math.ceil(total / limit)
    }
  });
};

exports.getById = async (req, res) => {
  const product = await Product.findOne({
    $or: [{ _id: req.params.id }, { slug: req.params.id }],
    status: 'active'
  }).populate('category', 'name slug');
  
  if (!product) return res.status(404).json({ error: 'Product not found' });
  
  res.json(product);
};

exports.create = async (req, res) => {
  const product = await Product.create(req.body);
  res.status(201).json(product);
};

exports.update = async (req, res) => {
  const product = await Product.findByIdAndUpdate(
    req.params.id,
    req.body,
    { new: true, runValidators: true }
  );
  
  if (!product) return res.status(404).json({ error: 'Product not found' });
  
  res.json(product);
};

exports.uploadImages = async (req, res) => {
  const product = await Product.findById(req.params.id);
  if (!product) return res.status(404).json({ error: 'Product not found' });

  const uploadPromises = req.files.map(file => uploadImage(file, 'products'));
  const uploadedImages = await Promise.all(uploadPromises);

  product.images.push(...uploadedImages);
  await product.save();

  res.json({ images: product.images });
};
```

---

## 3. Shopping Cart

```javascript
// services/cart.service.js
const Cart = require('../models/Cart');
const Product = require('../models/Product');
const Coupon = require('../models/Coupon');

class CartService {
  async getCart(userId) {
    let cart = await Cart.findOne({ user: userId, status: 'active' })
      .populate('items.product', 'name price images inventory slug');
    
    if (!cart) {
      cart = new Cart({ user: userId });
      await cart.save();
    }
    
    // ตรวจสอบ inventory
    await this.validateCartInventory(cart);
    
    return cart;
  }

  async addItem(userId, productId, quantity, options = {}) {
    const product = await Product.findById(productId);
    
    if (!product || product.status !== 'active') {
      throw new Error('Product not available');
    }
    
    if (product.inventory.trackInventory && product.inventory.available < quantity) {
      throw new Error(`Only ${product.inventory.available} items available`);
    }

    let cart = await Cart.findOne({ user: userId, status: 'active' });
    
    if (!cart) {
      cart = new Cart({ user: userId });
    }

    const existingItem = cart.items.find(
      item => item.product.toString() === productId
    );

    if (existingItem) {
      existingItem.quantity += quantity;
    } else {
      cart.items.push({
        product: productId,
        quantity,
        price: product.price,
        name: product.name,
        image: product.images[0]?.url,
        options
      });
    }

    await this.recalculate(cart);
    await cart.save();
    
    return cart.populate('items.product', 'name price images inventory');
  }

  async removeItem(userId, itemId) {
    const cart = await Cart.findOne({ user: userId, status: 'active' });
    if (!cart) throw new Error('Cart not found');

    cart.items = cart.items.filter(item => item._id.toString() !== itemId);
    
    await this.recalculate(cart);
    await cart.save();
    
    return cart;
  }

  async applyCoupon(userId, couponCode) {
    const cart = await Cart.findOne({ user: userId, status: 'active' });
    if (!cart) throw new Error('Cart not found');

    const coupon = await Coupon.findOne({
      code: couponCode.toUpperCase(),
      status: 'active',
      expiresAt: { $gt: new Date() }
    });

    if (!coupon) throw new Error('Invalid or expired coupon');

    // ตรวจสอบ minimum order amount
    if (coupon.minOrderAmount && cart.subtotal < coupon.minOrderAmount) {
      throw new Error(`Minimum order amount is ${coupon.minOrderAmount} THB`);
    }

    // ตรวจสอบจำนวนครั้งที่ใช้
    if (coupon.maxUses && coupon.usedCount >= coupon.maxUses) {
      throw new Error('Coupon usage limit reached');
    }

    cart.coupon = coupon._id;
    cart.couponCode = couponCode.toUpperCase();
    
    await this.recalculate(cart);
    await cart.save();
    
    return cart;
  }

  async recalculate(cart) {
    cart.subtotal = cart.items.reduce(
      (sum, item) => sum + (item.price * item.quantity), 0
    );

    if (cart.coupon) {
      const coupon = await Coupon.findById(cart.coupon);
      if (coupon) {
        if (coupon.type === 'percentage') {
          cart.discount = (cart.subtotal * coupon.value) / 100;
        } else {
          cart.discount = Math.min(coupon.value, cart.subtotal);
        }
      }
    } else {
      cart.discount = 0;
    }

    cart.tax = (cart.subtotal - cart.discount) * 0.07; // 7% VAT
    cart.shipping = cart.subtotal > 1000 ? 0 : 50; // Free shipping over 1000 THB
    cart.total = cart.subtotal - cart.discount + cart.tax + cart.shipping;
  }

  async validateCartInventory(cart) {
    let hasChanges = false;
    
    for (const item of cart.items) {
      const product = await Product.findById(item.product._id || item.product);
      
      if (!product || product.status !== 'active') {
        cart.items = cart.items.filter(i => i !== item);
        hasChanges = true;
        continue;
      }

      if (product.inventory.trackInventory && product.inventory.available < item.quantity) {
        item.quantity = product.inventory.available;
        if (item.quantity === 0) {
          cart.items = cart.items.filter(i => i !== item);
        }
        hasChanges = true;
      }
    }

    if (hasChanges) {
      await this.recalculate(cart);
      await cart.save();
    }

    return cart;
  }
}

module.exports = new CartService();
```

---

## 4. Stripe Payment Integration

```bash
npm install stripe
```

```javascript
// services/payment.service.js
const Stripe = require('stripe');
const stripe = new Stripe(process.env.STRIPE_SECRET_KEY);

class PaymentService {
  async createPaymentIntent(order) {
    const paymentIntent = await stripe.paymentIntents.create({
      amount: Math.round(order.total * 100), // Stripe ใช้ satang/cents
      currency: 'thb',
      metadata: {
        orderId: order._id.toString(),
        orderNumber: order.orderNumber,
        userId: order.user?.toString()
      },
      receipt_email: order.guestEmail || order.user?.email,
      description: `Order #${order.orderNumber}`
    });

    return {
      clientSecret: paymentIntent.client_secret,
      paymentIntentId: paymentIntent.id
    };
  }

  async confirmPayment(paymentIntentId) {
    const paymentIntent = await stripe.paymentIntents.retrieve(paymentIntentId);
    
    if (paymentIntent.status !== 'succeeded') {
      throw new Error(`Payment not succeeded: ${paymentIntent.status}`);
    }

    return {
      status: 'paid',
      transactionId: paymentIntent.id,
      amount: paymentIntent.amount / 100,
      paidAt: new Date(paymentIntent.created * 1000)
    };
  }

  async refund(paymentIntentId, amount) {
    const refund = await stripe.refunds.create({
      payment_intent: paymentIntentId,
      amount: Math.round(amount * 100)
    });

    return {
      refundId: refund.id,
      amount: refund.amount / 100,
      status: refund.status
    };
  }

  async handleWebhook(payload, signature) {
    const event = stripe.webhooks.constructEvent(
      payload,
      signature,
      process.env.STRIPE_WEBHOOK_SECRET
    );

    return event;
  }
}

module.exports = new PaymentService();
```

```javascript
// controllers/checkout.controller.js
const Order = require('../models/Order');
const cartService = require('../services/cart.service');
const paymentService = require('../services/payment.service');
const inventoryService = require('../services/inventory.service');
const emailService = require('../services/email.service');

exports.checkout = async (req, res) => {
  const session = await mongoose.startSession();
  session.startTransaction();
  
  try {
    const { shippingAddress, billingAddress, paymentMethod } = req.body;
    
    // ดึง cart
    const cart = await cartService.getCart(req.user.id);
    
    if (cart.items.length === 0) {
      throw new Error('Cart is empty');
    }

    // Reserve inventory
    await inventoryService.reserve(cart.items, session);

    // สร้าง order
    const order = new Order({
      user: req.user.id,
      items: cart.items.map(item => ({
        product: item.product._id,
        name: item.name,
        price: item.price,
        quantity: item.quantity,
        subtotal: item.price * item.quantity,
        image: item.image
      })),
      subtotal: cart.subtotal,
      discount: cart.discount,
      tax: cart.tax,
      shipping: cart.shipping,
      total: cart.total,
      couponCode: cart.couponCode,
      shippingAddress,
      billingAddress: billingAddress || shippingAddress,
      payment: { method: paymentMethod, status: 'pending' }
    });

    await order.save({ session });

    // สร้าง payment intent
    let paymentData = {};
    if (paymentMethod === 'stripe') {
      paymentData = await paymentService.createPaymentIntent(order);
    }

    await session.commitTransaction();

    // ยกเลิก cart
    cart.status = 'converted';
    await cart.save();

    res.status(201).json({
      order: {
        id: order._id,
        orderNumber: order.orderNumber,
        total: order.total,
        status: order.status
      },
      payment: paymentData
    });
  } catch (error) {
    await session.abortTransaction();
    throw error;
  } finally {
    session.endSession();
  }
};

// Stripe Webhook
exports.stripeWebhook = async (req, res) => {
  const sig = req.headers['stripe-signature'];
  
  let event;
  try {
    event = await paymentService.handleWebhook(req.rawBody, sig);
  } catch (err) {
    return res.status(400).json({ error: err.message });
  }

  switch (event.type) {
    case 'payment_intent.succeeded':
      await handlePaymentSuccess(event.data.object);
      break;
    case 'payment_intent.payment_failed':
      await handlePaymentFailure(event.data.object);
      break;
  }

  res.json({ received: true });
};

async function handlePaymentSuccess(paymentIntent) {
  const orderId = paymentIntent.metadata.orderId;
  const order = await Order.findById(orderId);
  
  if (!order) return;

  order.payment.status = 'paid';
  order.payment.transactionId = paymentIntent.id;
  order.payment.paidAt = new Date();
  order.status = 'confirmed';
  
  order.statusHistory.push({
    status: 'confirmed',
    timestamp: new Date(),
    note: 'Payment received'
  });
  
  await order.save();
  
  // ส่ง confirmation email
  await emailService.sendOrderConfirmation(order);
}
```

---

## 5. Inventory Management

```javascript
// services/inventory.service.js
class InventoryService {
  async reserve(items, session) {
    const bulkOps = items.map(item => ({
      updateOne: {
        filter: {
          _id: item.product._id || item.product,
          'inventory.available': { $gte: item.quantity }
        },
        update: {
          $inc: {
            'inventory.reserved': item.quantity,
            'inventory.available': -item.quantity
          }
        }
      }
    }));

    const result = await Product.bulkWrite(bulkOps, { session });
    
    if (result.matchedCount !== items.length) {
      throw new Error('Some items are out of stock');
    }
  }

  async release(items) {
    const bulkOps = items.map(item => ({
      updateOne: {
        filter: { _id: item.product },
        update: {
          $inc: {
            'inventory.reserved': -item.quantity,
            'inventory.available': item.quantity
          }
        }
      }
    }));

    await Product.bulkWrite(bulkOps);
  }

  async deduct(items) {
    const bulkOps = items.map(item => ({
      updateOne: {
        filter: { _id: item.product },
        update: {
          $inc: {
            'inventory.quantity': -item.quantity,
            'inventory.reserved': -item.quantity
          }
        }
      }
    }));

    await Product.bulkWrite(bulkOps);
  }

  async getLowStockProducts(threshold = 5) {
    return Product.find({
      'inventory.trackInventory': true,
      'inventory.available': { $lte: threshold, $gt: 0 }
    }).select('name sku inventory');
  }

  async getOutOfStockProducts() {
    return Product.find({
      'inventory.trackInventory': true,
      'inventory.available': 0
    }).select('name sku inventory');
  }
}

module.exports = new InventoryService();
```

---

## 6. Order Management

```javascript
// controllers/order.controller.js
exports.getUserOrders = async (req, res) => {
  const { page = 1, limit = 10, status } = req.query;
  
  const filter = { user: req.user.id };
  if (status) filter.status = status;

  const [orders, total] = await Promise.all([
    Order.find(filter)
      .sort('-createdAt')
      .skip((page - 1) * limit)
      .limit(parseInt(limit))
      .populate('items.product', 'name slug images'),
    Order.countDocuments(filter)
  ]);

  res.json({
    data: orders,
    pagination: { page: parseInt(page), limit: parseInt(limit), total }
  });
};

exports.cancelOrder = async (req, res) => {
  const order = await Order.findOne({
    _id: req.params.id,
    user: req.user.id
  });
  
  if (!order) return res.status(404).json({ error: 'Order not found' });
  
  const cancellableStatuses = ['pending', 'confirmed'];
  if (!cancellableStatuses.includes(order.status)) {
    return res.status(400).json({ error: `Cannot cancel order in ${order.status} status` });
  }

  const session = await mongoose.startSession();
  session.startTransaction();
  
  try {
    // Release inventory
    await inventoryService.release(order.items);
    
    // Refund payment
    if (order.payment.status === 'paid') {
      const refund = await paymentService.refund(
        order.payment.transactionId,
        order.total
      );
      order.payment.status = 'refunded';
    }

    order.status = 'cancelled';
    order.statusHistory.push({
      status: 'cancelled',
      timestamp: new Date(),
      note: req.body.reason || 'Cancelled by customer'
    });

    await order.save({ session });
    await session.commitTransaction();
    
    res.json({ message: 'Order cancelled successfully', order });
  } catch (error) {
    await session.abortTransaction();
    throw error;
  } finally {
    session.endSession();
  }
};
```

---

## แบบฝึกหัด

สร้าง complete e-commerce API:
1. Product CRUD + search + filtering
2. Cart management (add/remove/coupon)
3. Checkout flow
4. Stripe payment integration
5. Order management
6. Inventory tracking
7. Admin dashboard endpoints

```bash
# Start project
npm init -y
npm install express mongoose stripe ioredis jsonwebtoken bcrypt multer
npm install -D nodemon jest supertest
```

---

## สรุป

E-commerce API ต้องการการออกแบบที่รอบคอบ โดยเฉพาะเรื่อง inventory management (ป้องกัน overselling) และ payment processing (transaction integrity) ใช้ MongoDB transactions เพื่อความ consistent ของข้อมูล และ Stripe สำหรับ payment ที่ปลอดภัย

> ขั้นตอนต่อไป: Part 73 - Real-World Social Media API
