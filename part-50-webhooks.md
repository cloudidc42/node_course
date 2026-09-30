# Part 50: Webhooks
## ขั้นตอนที่ 50-50 จาก 1000

---

## บทนำ

Webhook คือ HTTP callback ที่ส่งข้อมูลแบบ real-time เมื่อมีเหตุการณ์เกิดขึ้น แทนที่จะ poll API ซ้ำ ๆ เราให้ระบบอื่นแจ้งเราเมื่อมีการเปลี่ยนแปลง

---

## 50.1 Webhook Concepts

### การทำงานของ Webhook

```
ปกติ (Polling):
Client → [GET /status?] → Server
Client → [GET /status?] → Server  (ทำซ้ำทุก N วินาที)
Client → [GET /status?] → Server

Webhook (Event-driven):
Client ← [POST /webhook] ← Server (เมื่อมี event เกิดขึ้น)
```

### Use Cases ของ Webhook

- **Payment Gateways** - Stripe, Omise แจ้งเมื่อชำระเงินสำเร็จ
- **CI/CD** - GitHub แจ้งเมื่อ push code
- **E-commerce** - Shopify แจ้งเมื่อมีคำสั่งซื้อ
- **Communication** - LINE, Slack แจ้งเมื่อมีข้อความ

---

## 50.2 Receiving Webhooks (รับ Webhook)

### Basic Webhook Receiver

```javascript
// src/controllers/webhookController.js
const express = require('express');

/**
 * รับ webhook payload
 */
const receiveWebhook = async (req, res) => {
  // ต้องตอบกลับ 200 เร็ว ๆ ก่อน ไม่งั้น provider จะ retry
  res.status(200).json({ received: true });
  
  // Process ใน background
  processWebhookAsync(req.body, req.headers).catch(err => {
    console.error('Webhook processing error:', err);
  });
};

const processWebhookAsync = async (payload, headers) => {
  const { event, data } = payload;
  
  console.log(`📨 Webhook received: ${event}`);
  
  switch (event) {
    case 'payment.success':
      await handlePaymentSuccess(data);
      break;
    case 'payment.failed':
      await handlePaymentFailed(data);
      break;
    case 'order.created':
      await handleOrderCreated(data);
      break;
    case 'user.deleted':
      await handleUserDeleted(data);
      break;
    default:
      console.log(`Unhandled event: ${event}`);
  }
};

const handlePaymentSuccess = async (data) => {
  const { orderId, amount, currency } = data;
  console.log(`✅ Payment success: Order ${orderId}, Amount ${amount} ${currency}`);
  
  // อัพเดท order status
  await Order.update(
    { status: 'paid', paidAt: new Date() },
    { where: { id: orderId } }
  );
  
  // ส่ง email confirmation
  await emailQueue.add('order-confirmation', { orderId });
};

module.exports = { receiveWebhook };
```

---

## 50.3 Signature Verification (การตรวจสอบลายเซ็น)

การตรวจสอบว่า webhook มาจากแหล่งที่น่าเชื่อถือจริง

### HMAC Signature Verification

```javascript
// src/middleware/webhookVerification.js
const crypto = require('crypto');

/**
 * Stripe-style webhook verification
 */
const verifyStripeWebhook = (req, res, next) => {
  const signature = req.headers['stripe-signature'];
  const secret = process.env.STRIPE_WEBHOOK_SECRET;
  
  if (!signature || !secret) {
    return res.status(401).json({ error: 'Missing webhook signature' });
  }
  
  try {
    const stripe = require('stripe')(process.env.STRIPE_SECRET_KEY);
    
    // Stripe ตรวจสอบ timestamp ด้วย เพื่อป้องกัน replay attack
    const event = stripe.webhooks.constructEvent(
      req.rawBody,  // ต้องใช้ raw body ไม่ใช่ parsed JSON
      signature,
      secret
    );
    
    req.webhookEvent = event;
    next();
  } catch (error) {
    console.error('Webhook signature verification failed:', error.message);
    return res.status(400).json({ error: 'Invalid webhook signature' });
  }
};

/**
 * Generic HMAC verification
 */
const verifyHmacSignature = (secret, headerName = 'x-webhook-signature') => {
  return (req, res, next) => {
    const signature = req.headers[headerName];
    
    if (!signature) {
      return res.status(401).json({ error: 'Missing signature header' });
    }
    
    // สร้าง HMAC จาก body
    const hmac = crypto.createHmac('sha256', secret);
    const expectedSignature = hmac
      .update(req.rawBody || JSON.stringify(req.body))
      .digest('hex');
    
    // ใช้ timingSafeEqual เพื่อป้องกัน timing attack
    const providedBuffer = Buffer.from(signature.replace('sha256=', ''), 'hex');
    const expectedBuffer = Buffer.from(expectedSignature, 'hex');
    
    if (providedBuffer.length !== expectedBuffer.length) {
      return res.status(401).json({ error: 'Invalid signature length' });
    }
    
    if (!crypto.timingSafeEqual(providedBuffer, expectedBuffer)) {
      console.warn(`⚠️ Invalid webhook signature from ${req.ip}`);
      return res.status(401).json({ error: 'Invalid webhook signature' });
    }
    
    next();
  };
};

/**
 * GitHub webhook verification
 */
const verifyGitHubWebhook = (req, res, next) => {
  const signature = req.headers['x-hub-signature-256'];
  const secret = process.env.GITHUB_WEBHOOK_SECRET;
  
  if (!signature) {
    return res.status(401).json({ error: 'No signature' });
  }
  
  const hmac = crypto.createHmac('sha256', secret);
  const digest = `sha256=${hmac.update(req.rawBody).digest('hex')}`;
  
  if (!crypto.timingSafeEqual(Buffer.from(digest), Buffer.from(signature))) {
    return res.status(401).json({ error: 'Invalid signature' });
  }
  
  next();
};

module.exports = { verifyStripeWebhook, verifyHmacSignature, verifyGitHubWebhook };
```

### Preserve Raw Body สำหรับ Signature Verification

```javascript
// src/app.js
const express = require('express');
const app = express();

// สำคัญมาก: เก็บ raw body ไว้สำหรับ webhook verification
app.use('/webhooks', (req, res, next) => {
  let rawBody = '';
  req.on('data', chunk => {
    rawBody += chunk.toString();
  });
  req.on('end', () => {
    req.rawBody = rawBody;
    req.body = JSON.parse(rawBody);
    next();
  });
});

// สำหรับ routes ทั่วไป
app.use(express.json());
app.use(express.urlencoded({ extended: true }));
```

---

## 50.4 Retry Handling (การจัดการ Retry)

### Webhook Delivery System (Outgoing Webhooks)

```javascript
// src/services/webhookDeliveryService.js
const axios = require('axios');
const crypto = require('crypto');

const MAX_RETRIES = 5;
const RETRY_DELAYS = [30, 60, 300, 1800, 3600]; // seconds

/**
 * ส่ง webhook พร้อม retry logic
 */
const deliverWebhook = async (webhookConfig, event, payload) => {
  const { url, secret, id: webhookId } = webhookConfig;
  
  let lastError;
  
  for (let attempt = 0; attempt < MAX_RETRIES; attempt++) {
    try {
      const result = await sendWebhookRequest(url, secret, event, payload, attempt);
      
      // บันทึกสำเร็จ
      await logWebhookDelivery({
        webhookId,
        event,
        attempt,
        status: 'success',
        responseCode: result.status,
        deliveredAt: new Date()
      });
      
      return { success: true, attempt, statusCode: result.status };
    } catch (error) {
      lastError = error;
      
      console.warn(`Webhook delivery attempt ${attempt + 1} failed:`, error.message);
      
      // บันทึก attempt ที่ล้มเหลว
      await logWebhookDelivery({
        webhookId,
        event,
        attempt,
        status: 'failed',
        error: error.message,
        responseCode: error.response?.status
      });
      
      // รอก่อน retry (ยกเว้น attempt สุดท้าย)
      if (attempt < MAX_RETRIES - 1) {
        const delaySeconds = RETRY_DELAYS[attempt] || 3600;
        await sleep(delaySeconds * 1000);
      }
    }
  }
  
  // ล้มเหลวทั้งหมด attempts
  await markWebhookFailed(webhookId, event, lastError);
  throw new Error(`Webhook delivery failed after ${MAX_RETRIES} attempts`);
};

const sendWebhookRequest = async (url, secret, event, payload, attempt) => {
  const timestamp = Math.floor(Date.now() / 1000);
  const body = JSON.stringify({
    event,
    data: payload,
    timestamp,
    attempt
  });
  
  // สร้าง signature
  const signature = crypto
    .createHmac('sha256', secret)
    .update(`${timestamp}.${body}`)
    .digest('hex');
  
  const response = await axios.post(url, body, {
    headers: {
      'Content-Type': 'application/json',
      'X-Webhook-Signature': `sha256=${signature}`,
      'X-Webhook-Timestamp': timestamp.toString(),
      'X-Webhook-Event': event,
      'User-Agent': 'MyApp-Webhook/1.0'
    },
    timeout: 10000,  // 10 second timeout
    validateStatus: (status) => status < 500  // retry เฉพาะ 5xx
  });
  
  // ถ้า response ไม่ใช่ 2xx ให้ throw error
  if (response.status >= 400) {
    throw new Error(`HTTP ${response.status}: ${response.statusText}`);
  }
  
  return response;
};

const sleep = (ms) => new Promise(resolve => setTimeout(resolve, ms));

module.exports = { deliverWebhook };
```

### Webhook Management API

```javascript
// src/controllers/webhookManagementController.js
const Webhook = require('../models/Webhook');
const { deliverWebhook } = require('../services/webhookDeliveryService');
const crypto = require('crypto');

/**
 * สร้าง webhook endpoint ใหม่
 */
const createWebhook = async (req, res) => {
  try {
    const { url, events, description } = req.body;
    const userId = req.user.id;
    
    // Validate URL
    try {
      new URL(url);
    } catch {
      return res.status(400).json({ error: 'Invalid URL' });
    }
    
    // ตรวจสอบ events ที่รองรับ
    const supportedEvents = [
      'payment.success', 'payment.failed',
      'order.created', 'order.updated', 'order.cancelled',
      'user.created', 'user.deleted'
    ];
    
    const invalidEvents = events.filter(e => !supportedEvents.includes(e));
    if (invalidEvents.length > 0) {
      return res.status(400).json({
        error: 'Unsupported events',
        invalidEvents
      });
    }
    
    // สร้าง secret สำหรับ signing
    const secret = crypto.randomBytes(32).toString('hex');
    
    const webhook = await Webhook.create({
      userId,
      url,
      events,
      description,
      secret,
      isActive: true
    });
    
    res.status(201).json({
      success: true,
      data: {
        id: webhook.id,
        url: webhook.url,
        events: webhook.events,
        secret,  // แสดงครั้งเดียวตอนสร้าง
        createdAt: webhook.createdAt
      }
    });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
};

/**
 * Test webhook delivery
 */
const testWebhook = async (req, res) => {
  try {
    const webhook = await Webhook.findOne({
      where: { id: req.params.id, userId: req.user.id }
    });
    
    if (!webhook) {
      return res.status(404).json({ error: 'Webhook not found' });
    }
    
    const result = await deliverWebhook(webhook, 'webhook.test', {
      message: 'This is a test webhook',
      timestamp: new Date().toISOString()
    });
    
    res.json({ success: true, result });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
};

module.exports = { createWebhook, testWebhook };
```

---

## 50.5 Complete Webhook Examples

### Stripe Payment Webhook

```javascript
// src/webhooks/stripeWebhook.js
const express = require('express');
const router = express.Router();
const stripe = require('stripe')(process.env.STRIPE_SECRET_KEY);

// ต้องใช้ raw body
router.post('/stripe', express.raw({ type: 'application/json' }), async (req, res) => {
  const sig = req.headers['stripe-signature'];
  
  let event;
  try {
    event = stripe.webhooks.constructEvent(
      req.body,
      sig,
      process.env.STRIPE_WEBHOOK_SECRET
    );
  } catch (err) {
    console.error('Stripe webhook error:', err.message);
    return res.status(400).send(`Webhook Error: ${err.message}`);
  }
  
  // ตอบ 200 ก่อน
  res.json({ received: true });
  
  // Process event
  switch (event.type) {
    case 'payment_intent.succeeded':
      const paymentIntent = event.data.object;
      await handleSuccessfulPayment(paymentIntent);
      break;
      
    case 'payment_intent.payment_failed':
      const failedPayment = event.data.object;
      await handleFailedPayment(failedPayment);
      break;
      
    case 'customer.subscription.created':
      await handleSubscriptionCreated(event.data.object);
      break;
      
    case 'customer.subscription.deleted':
      await handleSubscriptionCancelled(event.data.object);
      break;
      
    default:
      console.log(`Unhandled Stripe event: ${event.type}`);
  }
});

const handleSuccessfulPayment = async (paymentIntent) => {
  const { id, amount, currency, metadata } = paymentIntent;
  const orderId = metadata.orderId;
  
  await Order.update(
    { status: 'paid', stripePaymentIntentId: id },
    { where: { id: orderId } }
  );
  
  await emailQueue.add('order-confirmation', { orderId });
  console.log(`✅ Payment ${id} processed for order ${orderId}`);
};

module.exports = router;
```

### GitHub Webhook for CI/CD

```javascript
// src/webhooks/githubWebhook.js
const express = require('express');
const router = express.Router();
const { verifyGitHubWebhook } = require('../middleware/webhookVerification');
const { exec } = require('child_process');
const util = require('util');
const execAsync = util.promisify(exec);

router.post('/github',
  express.raw({ type: 'application/json' }),
  verifyGitHubWebhook,
  async (req, res) => {
    const event = req.headers['x-github-event'];
    const payload = JSON.parse(req.rawBody);
    
    res.json({ received: true });
    
    if (event === 'push' && payload.ref === 'refs/heads/main') {
      await deployApplication(payload);
    } else if (event === 'pull_request' && payload.action === 'closed' && payload.pull_request.merged) {
      await handleMergedPR(payload);
    }
  }
);

const deployApplication = async (payload) => {
  const { repository, pusher, commits } = payload;
  console.log(`🚀 Deploying ${repository.name} (pushed by ${pusher.name})`);
  
  try {
    await execAsync('git pull origin main');
    await execAsync('npm install --production');
    await execAsync('npm run build');
    await execAsync('pm2 restart app');
    
    console.log('✅ Deployment successful');
  } catch (error) {
    console.error('❌ Deployment failed:', error.message);
    await notifySlack(`Deployment failed: ${error.message}`);
  }
};

module.exports = router;
```

---

## แบบฝึกหัดที่ 50

### แบบฝึกหัดพื้นฐาน

**1. Webhook Receiver**

สร้าง webhook receiver ที่:
- รับ POST requests
- Verify HMAC signature
- บันทึก events ลง database
- Process events async

**2. Webhook Delivery System**

สร้าง system ส่ง webhooks ออก:
- สร้าง webhook endpoints
- ส่ง events พร้อม retry 5 ครั้ง
- Dashboard แสดง delivery history

### แบบฝึกหัดขั้นสูง

**3. Payment Gateway Integration**

Integrate Stripe webhooks:
- Handle payment.succeeded → update order
- Handle payment.failed → notify user
- Handle subscription events

**4. Event Sourcing with Webhooks**

สร้าง event-driven architecture:
- Events: user.created, order.placed, payment.processed
- Distribute events ผ่าน webhooks
- Replay events ถ้าต้องการ

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **Webhook Concepts** - event-driven vs polling
2. **Receiving Webhooks** - raw body, async processing
3. **Signature Verification** - HMAC, timingSafeEqual
4. **Retry Handling** - exponential backoff
5. **Real examples** - Stripe, GitHub webhooks

**ถัดไป:** Part 51 - Microservices Architecture
