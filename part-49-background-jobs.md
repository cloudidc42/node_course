# Part 49: Background Jobs (งานเบื้องหลัง)
## ขั้นตอนที่ 49-49 จาก 1000

---

## บทนำ

Background Jobs คืองานที่ทำในเบื้องหลัง ไม่รบกวน HTTP request-response cycle หลัก ตัวอย่างเช่น การส่งอีเมล, การ resize รูปภาพ, การสร้าง report, หรือการ sync ข้อมูล

---

## 49.1 BullMQ - Job Queue สมัยใหม่

BullMQ เป็น library สำหรับ job queue ที่ใช้ Redis เป็น backend

### ติดตั้ง BullMQ

```bash
npm install bullmq ioredis
npm install @bull-board/express @bull-board/api  # Dashboard
```

### ตั้งค่า Redis Connection

```javascript
// src/config/redis.js
const { Redis } = require('ioredis');

const redisConnection = new Redis({
  host: process.env.REDIS_HOST || 'localhost',
  port: parseInt(process.env.REDIS_PORT) || 6379,
  password: process.env.REDIS_PASSWORD,
  maxRetriesPerRequest: null,  // สำคัญสำหรับ BullMQ
  enableReadyCheck: false
});

redisConnection.on('connect', () => {
  console.log('✅ Redis connected');
});

redisConnection.on('error', (err) => {
  console.error('❌ Redis error:', err.message);
});

module.exports = redisConnection;
```

---

## 49.2 Job Queues (คิวงาน)

### สร้าง Queue

```javascript
// src/queues/emailQueue.js
const { Queue } = require('bullmq');
const redisConnection = require('../config/redis');

// สร้าง queue
const emailQueue = new Queue('email', {
  connection: redisConnection,
  defaultJobOptions: {
    attempts: 3,           // จำนวนครั้งที่ retry
    backoff: {
      type: 'exponential',
      delay: 1000          // เริ่ม delay 1s, 2s, 4s...
    },
    removeOnComplete: {
      count: 100,          // เก็บ completed jobs ล่าสุด 100 ชิ้น
      age: 24 * 3600      // เก็บ 24 ชั่วโมง
    },
    removeOnFail: {
      count: 50
    }
  }
});

// เพิ่มงานเข้า queue
const addEmailJob = async (type, data, options = {}) => {
  const job = await emailQueue.add(type, data, {
    priority: options.priority || 0,
    delay: options.delay || 0,
    ...options
  });
  
  console.log(`📧 Email job added: ${job.id} (${type})`);
  return job;
};

module.exports = { emailQueue, addEmailJob };
```

### Worker - ประมวลผลงาน

```javascript
// src/workers/emailWorker.js
const { Worker, QueueEvents } = require('bullmq');
const redisConnection = require('../config/redis');
const emailService = require('../services/emailService');

// Worker ประมวลผลงาน
const emailWorker = new Worker('email', async (job) => {
  console.log(`Processing job ${job.id}: ${job.name}`);
  
  switch (job.name) {
    case 'welcome':
      await processWelcomeEmail(job.data);
      break;
    case 'password-reset':
      await processPasswordResetEmail(job.data);
      break;
    case 'order-confirmation':
      await processOrderConfirmationEmail(job.data);
      break;
    case 'newsletter':
      await processNewsletterEmail(job.data);
      break;
    default:
      throw new Error(`Unknown email type: ${job.name}`);
  }
  
  return { sent: true, timestamp: new Date().toISOString() };
}, {
  connection: redisConnection,
  concurrency: 5,  // ประมวลผลพร้อมกัน 5 งาน
  limiter: {
    max: 100,    // สูงสุด 100 jobs
    duration: 60000  // ต่อ 1 นาที (rate limiting)
  }
});

// Processors
const processWelcomeEmail = async (data) => {
  const { userId, email, name } = data;
  
  await emailService.send({
    to: email,
    subject: `ยินดีต้อนรับ ${name}!`,
    template: 'welcome',
    context: { name, userId }
  });
};

const processPasswordResetEmail = async (data) => {
  const { email, resetToken, expiresAt } = data;
  
  await emailService.send({
    to: email,
    subject: 'รีเซ็ตรหัสผ่าน',
    template: 'password-reset',
    context: { resetToken, expiresAt }
  });
};

const processOrderConfirmationEmail = async (data) => {
  const { orderId, email, items, total } = data;
  
  await emailService.send({
    to: email,
    subject: `ยืนยันคำสั่งซื้อ #${orderId}`,
    template: 'order-confirmation',
    context: { orderId, items, total }
  });
};

const processNewsletterEmail = async (data) => {
  const { email, content, unsubscribeToken } = data;
  
  // Update progress
  await emailWorker.emit('progress', { percentage: 50 });
  
  await emailService.send({
    to: email,
    subject: 'Newsletter ประจำสัปดาห์',
    template: 'newsletter',
    context: { content, unsubscribeToken }
  });
};

// Event handlers
emailWorker.on('completed', (job, result) => {
  console.log(`✅ Job ${job.id} completed:`, result);
});

emailWorker.on('failed', (job, error) => {
  console.error(`❌ Job ${job?.id} failed:`, error.message);
  // ส่ง alert ถ้า job ล้มเหลวทั้งหมด attempts
  if (job?.attemptsMade >= job?.opts?.attempts) {
    notifyAdmin(job, error);
  }
});

emailWorker.on('progress', (job, progress) => {
  console.log(`📊 Job ${job.id} progress: ${JSON.stringify(progress)}`);
});

const notifyAdmin = async (job, error) => {
  console.error(`🚨 ALERT: Job ${job.id} permanently failed after ${job.attemptsMade} attempts`);
  // ส่งแจ้งเตือนทาง Slack, email ผู้ดูแล ฯลฯ
};

module.exports = emailWorker;
```

### Queue Events

```javascript
// src/workers/emailWorkerEvents.js
const { QueueEvents } = require('bullmq');
const redisConnection = require('../config/redis');

const emailQueueEvents = new QueueEvents('email', {
  connection: redisConnection
});

emailQueueEvents.on('waiting', ({ jobId }) => {
  console.log(`⏳ Job ${jobId} is waiting`);
});

emailQueueEvents.on('active', ({ jobId, prev }) => {
  console.log(`🔄 Job ${jobId} is active (was: ${prev})`);
});

emailQueueEvents.on('completed', ({ jobId, returnvalue }) => {
  console.log(`✅ Job ${jobId} completed:`, returnvalue);
});

emailQueueEvents.on('failed', ({ jobId, failedReason }) => {
  console.error(`❌ Job ${jobId} failed:`, failedReason);
});

emailQueueEvents.on('delayed', ({ jobId, delay }) => {
  console.log(`⏰ Job ${jobId} delayed for ${delay}ms`);
});

emailQueueEvents.on('stalled', ({ jobId }) => {
  console.warn(`⚠️ Job ${jobId} stalled`);
});

module.exports = emailQueueEvents;
```

---

## 49.3 Scheduled Jobs (งานตามเวลา)

```javascript
// src/queues/scheduledQueue.js
const { Queue } = require('bullmq');
const redisConnection = require('../config/redis');

const scheduledQueue = new Queue('scheduled', {
  connection: redisConnection
});

/**
 * Repeatable jobs - ทำงานซ้ำตาม cron schedule
 */
const setupScheduledJobs = async () => {
  // ทำงานทุกวัน เวลา 8:00 น.
  await scheduledQueue.add(
    'daily-report',
    { type: 'daily' },
    {
      repeat: { cron: '0 8 * * *' },  // cron expression
      removeOnComplete: 10
    }
  );
  
  // ทำงานทุกชั่วโมง
  await scheduledQueue.add(
    'hourly-sync',
    { source: 'external-api' },
    {
      repeat: { every: 60 * 60 * 1000 },  // milliseconds
    }
  );
  
  // ทำงานทุกวันจันทร์ เวลา 9:00 น.
  await scheduledQueue.add(
    'weekly-newsletter',
    { type: 'weekly' },
    {
      repeat: { cron: '0 9 * * 1' }  // 9:00 AM ทุกวันจันทร์
    }
  );
  
  console.log('✅ Scheduled jobs set up');
};

/**
 * Delayed job - ทำงานหลังจากหน่วงเวลา
 */
const scheduleDelayedJob = async (type, data, delayMs) => {
  const job = await scheduledQueue.add(type, data, {
    delay: delayMs
  });
  
  const runAt = new Date(Date.now() + delayMs);
  console.log(`⏰ Job scheduled to run at: ${runAt.toISOString()}`);
  
  return job;
};

// ตัวอย่าง: ส่งอีเมล reminder หลัง 3 วัน
const scheduleReminderEmail = async (userId, email) => {
  const threeDaysMs = 3 * 24 * 60 * 60 * 1000;
  
  return await scheduleDelayedJob(
    'reminder-email',
    { userId, email },
    threeDaysMs
  );
};

module.exports = { scheduledQueue, setupScheduledJobs, scheduleDelayedJob, scheduleReminderEmail };
```

### Scheduled Worker

```javascript
// src/workers/scheduledWorker.js
const { Worker } = require('bullmq');
const redisConnection = require('../config/redis');

const scheduledWorker = new Worker('scheduled', async (job) => {
  console.log(`⏰ Running scheduled job: ${job.name}`);
  
  switch (job.name) {
    case 'daily-report':
      return await runDailyReport(job.data);
    case 'hourly-sync':
      return await runHourlySync(job.data);
    case 'weekly-newsletter':
      return await runWeeklyNewsletter(job.data);
    case 'reminder-email':
      return await sendReminderEmail(job.data);
    default:
      throw new Error(`Unknown scheduled job: ${job.name}`);
  }
}, {
  connection: redisConnection,
  concurrency: 1  // Scheduled jobs ทำทีละงาน
});

const runDailyReport = async (data) => {
  console.log('📊 Generating daily report...');
  // สร้าง report และส่งอีเมล
  const report = await generateDailyStats();
  await sendReportEmail(report);
  return { success: true, stats: report.summary };
};

const runHourlySync = async (data) => {
  console.log('🔄 Running hourly sync...');
  const synced = await syncExternalData();
  return { synced: synced.count };
};

const generateDailyStats = async () => {
  // Logic สำหรับสร้าง stats
  return {
    summary: {
      newUsers: 45,
      orders: 128,
      revenue: 50000
    }
  };
};

module.exports = scheduledWorker;
```

---

## 49.4 Retry Logic

```javascript
// src/queues/robustQueue.js
const { Queue } = require('bullmq');
const redisConnection = require('../config/redis');

/**
 * Queue พร้อม retry logic ที่ซับซ้อน
 */
const robustQueue = new Queue('robust-tasks', {
  connection: redisConnection,
  defaultJobOptions: {
    attempts: 5,
    backoff: {
      type: 'exponential',
      delay: 2000  // 2s, 4s, 8s, 16s, 32s
    }
  }
});

/**
 * Custom retry logic ใน Worker
 */
const { Worker, UnrecoverableError } = require('bullmq');

const robustWorker = new Worker('robust-tasks', async (job) => {
  try {
    return await processJob(job);
  } catch (error) {
    // ถ้าเป็น error ที่ไม่ควร retry
    if (error.code === 'INVALID_DATA') {
      throw new UnrecoverableError(error.message);
    }
    
    // ถ้าเป็น rate limit error ให้ delay ก่อน retry
    if (error.code === 'RATE_LIMITED') {
      await job.moveToDelayed(Date.now() + 60000);  // delay 1 นาที
      throw error;
    }
    
    // ส่ง error ต่อเพื่อให้ retry
    throw error;
  }
}, {
  connection: redisConnection,
  concurrency: 3
});

const processJob = async (job) => {
  const { type, data, retryCount } = job.data;
  
  // Log attempt number
  console.log(`Attempt ${job.attemptsMade + 1}/${job.opts.attempts} for job ${job.id}`);
  
  // Update job progress
  await job.updateProgress({ attempt: job.attemptsMade + 1 });
  
  // Simulate work that might fail
  if (Math.random() < 0.3) {  // 30% chance of failure
    throw new Error('Random failure - will retry');
  }
  
  return { success: true, processedAt: new Date().toISOString() };
};

// Dead letter queue - รับงานที่ fail ทั้งหมด attempts
const handleDeadLetter = async (queue) => {
  const failedJobs = await queue.getFailed(0, 50);
  
  for (const job of failedJobs) {
    console.log(`Dead letter job ${job.id}:`, {
      name: job.name,
      data: job.data,
      failedReason: job.failedReason,
      processedOn: job.processedOn
    });
    
    // บันทึกลง database สำหรับ review
    await saveFailedJobToDb(job);
  }
};

module.exports = { robustQueue, robustWorker };
```

---

## 49.5 Job Monitoring (การติดตามสถานะงาน)

### Bull Dashboard

```javascript
// src/app.js
const express = require('express');
const { createBullBoard } = require('@bull-board/api');
const { BullMQAdapter } = require('@bull-board/api/bullMQAdapter');
const { ExpressAdapter } = require('@bull-board/express');
const { emailQueue } = require('./queues/emailQueue');
const { scheduledQueue } = require('./queues/scheduledQueue');

const app = express();

// ตั้งค่า Bull Board Dashboard
const serverAdapter = new ExpressAdapter();
serverAdapter.setBasePath('/admin/queues');

createBullBoard({
  queues: [
    new BullMQAdapter(emailQueue),
    new BullMQAdapter(scheduledQueue)
  ],
  serverAdapter
});

// Protect dashboard ด้วย basic auth
const basicAuth = require('express-basic-auth');
app.use('/admin/queues', 
  basicAuth({
    users: { admin: process.env.ADMIN_PASSWORD || 'password' },
    challenge: true
  }),
  serverAdapter.getRouter()
);

// Queue Status API
app.get('/api/queue-status', async (req, res) => {
  try {
    const [emailCounts, scheduledCounts] = await Promise.all([
      emailQueue.getJobCounts(),
      scheduledQueue.getJobCounts()
    ]);
    
    res.json({
      email: emailCounts,
      scheduled: scheduledCounts,
      timestamp: new Date().toISOString()
    });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

module.exports = app;
```

### Job Status Tracking

```javascript
// src/services/jobTrackerService.js
const { emailQueue } = require('../queues/emailQueue');

/**
 * ติดตาม status ของ job
 */
const trackJob = async (jobId, queueName = 'email') => {
  const queue = getQueue(queueName);
  const job = await queue.getJob(jobId);
  
  if (!job) {
    return { found: false };
  }
  
  const state = await job.getState();
  
  return {
    found: true,
    jobId,
    name: job.name,
    state,
    progress: job.progress,
    data: job.data,
    result: job.returnvalue,
    error: job.failedReason,
    attempts: {
      made: job.attemptsMade,
      total: job.opts?.attempts
    },
    timestamps: {
      added: new Date(job.timestamp).toISOString(),
      started: job.processedOn ? new Date(job.processedOn).toISOString() : null,
      finished: job.finishedOn ? new Date(job.finishedOn).toISOString() : null
    }
  };
};

/**
 * รอจนกว่า job จะเสร็จ (polling)
 */
const waitForJob = async (jobId, queueName = 'email', timeout = 30000) => {
  const queue = getQueue(queueName);
  const job = await queue.getJob(jobId);
  
  if (!job) throw new Error(`Job ${jobId} not found`);
  
  return await job.waitUntilFinished(
    require('../workers/emailWorkerEvents'),
    timeout
  );
};

const getQueue = (name) => {
  const queues = {
    email: require('../queues/emailQueue').emailQueue,
    scheduled: require('../queues/scheduledQueue').scheduledQueue
  };
  
  if (!queues[name]) throw new Error(`Queue ${name} not found`);
  return queues[name];
};

module.exports = { trackJob, waitForJob };
```

---

## 49.6 Real-world Example: Order Processing

```javascript
// src/queues/orderQueue.js
const { Queue, Worker, FlowProducer } = require('bullmq');
const redisConnection = require('../config/redis');

const orderQueue = new Queue('orders', { connection: redisConnection });

/**
 * Flow - ลำดับงานที่ต้องทำ
 */
const flowProducer = new FlowProducer({ connection: redisConnection });

/**
 * สร้าง order processing flow
 * payment → inventory → fulfillment → notification
 */
const createOrderProcessingFlow = async (orderData) => {
  const flow = await flowProducer.add({
    name: 'process-payment',
    queueName: 'orders',
    data: { orderId: orderData.id, amount: orderData.total },
    children: [
      {
        name: 'update-inventory',
        queueName: 'orders',
        data: { orderId: orderData.id, items: orderData.items },
        children: [
          {
            name: 'create-fulfillment',
            queueName: 'orders',
            data: { orderId: orderData.id, address: orderData.address },
            children: [
              {
                name: 'send-confirmation',
                queueName: 'orders',
                data: { 
                  orderId: orderData.id, 
                  email: orderData.customerEmail 
                }
              }
            ]
          }
        ]
      }
    ]
  });
  
  return flow;
};

// Worker สำหรับ order processing
const orderWorker = new Worker('orders', async (job) => {
  switch (job.name) {
    case 'process-payment':
      return await handlePayment(job.data);
    case 'update-inventory':
      return await updateInventory(job.data);
    case 'create-fulfillment':
      return await createFulfillment(job.data);
    case 'send-confirmation':
      return await sendConfirmation(job.data);
    default:
      throw new Error(`Unknown order job: ${job.name}`);
  }
}, { connection: redisConnection });

const handlePayment = async ({ orderId, amount }) => {
  console.log(`💳 Processing payment for order ${orderId}: ฿${amount}`);
  // เรียก payment gateway
  return { paymentId: `PAY-${Date.now()}`, status: 'success' };
};

const updateInventory = async ({ orderId, items }) => {
  console.log(`📦 Updating inventory for order ${orderId}`);
  // อัพเดท stock
  return { updated: items.length };
};

const createFulfillment = async ({ orderId, address }) => {
  console.log(`🚚 Creating fulfillment for order ${orderId}`);
  // สร้างคำสั่ง shipping
  return { trackingNumber: `TH-${Date.now()}` };
};

const sendConfirmation = async ({ orderId, email }) => {
  console.log(`📧 Sending confirmation to ${email} for order ${orderId}`);
  // ส่งอีเมลยืนยัน
  return { emailSent: true };
};

module.exports = { orderQueue, orderWorker, createOrderProcessingFlow };
```

---

## แบบฝึกหัดที่ 49

### แบบฝึกหัดพื้นฐาน

**1. Email Queue System**

สร้าง email notification system ที่:
- Queue สำหรับ welcome email, password reset, notifications
- Worker ประมวลผล 5 emails พร้อมกัน
- Retry 3 ครั้งถ้าส่งไม่สำเร็จ
- Dashboard แสดงสถานะ

**2. Image Processing Queue**

สร้าง queue สำหรับ process รูปภาพ:
- รับ upload → เพิ่มงานใน queue
- Worker: resize, convert to WebP, generate thumbnails
- Return URL เมื่องานเสร็จ

### แบบฝึกหัดขั้นสูง

**3. Scheduled Report System**

สร้าง system ที่:
- สร้าง daily, weekly, monthly reports
- ส่งรายงาน Email อัตโนมัติ
- Dashboard แสดงประวัติการ run

**4. Order Processing Flow**

สร้าง e-commerce order processing:
- Payment → Inventory → Fulfillment → Notification
- Handle failure ในแต่ละ step
- Compensating transactions ถ้ามีขั้นตอนล้มเหลว

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **BullMQ** - Job queue ที่ใช้ Redis
2. **Workers** - ประมวลผลงานแบบ concurrent
3. **Scheduled Jobs** - ใช้ cron expressions
4. **Retry Logic** - exponential backoff, dead letter
5. **Monitoring** - Bull Board dashboard

**ถัดไป:** Part 50 - Webhooks
