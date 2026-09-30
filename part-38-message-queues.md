# Part 38 | ขั้นตอนที่ 661-680 จาก 1000

## Message Queues - คิวงานและการประมวลผลแบบ Asynchronous

---

## สารบัญ

1. [Message Queue คืออะไร](#message-queue-คืออะไร)
2. [RabbitMQ พื้นฐาน](#rabbitmq-พื้นฐาน)
3. [Bull Queue](#bull-queue)
4. [Job Processing Patterns](#job-processing-patterns)
5. [Retry Logic และ Error Handling](#retry-logic-และ-error-handling)
6. [Priority Queues](#priority-queues)
7. [Scheduled Jobs](#scheduled-jobs)
8. [Monitoring Queues](#monitoring-queues)
9. [แบบฝึกหัด](#แบบฝึกหัด)

---

## Message Queue คืออะไร

### ขั้นตอนที่ 661: ทำความเข้าใจ Message Queue

```
ปัญหา: tasks ที่ใช้เวลานาน block response

Without Queue:
  User Request → Process Email → Process Image → Process PDF → Response
  (user รอนาน 30 วินาที)

With Queue:
  User Request → Add to Queue → Response (ทันที)
                     ↓
              Background Worker processes tasks
```

```
ประโยชน์ของ Message Queue:
  ✅ Decoupling - ผู้ส่งและผู้รับไม่ต้องรู้จักกัน
  ✅ Reliability - ถ้า worker ล้ม job ยังอยู่ใน queue
  ✅ Scalability - เพิ่ม workers ตามความต้องการ
  ✅ Rate Limiting - ควบคุมความเร็วในการประมวลผล
  ✅ Retry - ลองใหม่อัตโนมัติเมื่อเกิดข้อผิดพลาด
```

---

## RabbitMQ พื้นฐาน

### ขั้นตอนที่ 662: ติดตั้ง RabbitMQ

```bash
# Docker (แนะนำ)
docker run -d \
  --name rabbitmq \
  -p 5672:5672 \
  -p 15672:15672 \
  -e RABBITMQ_DEFAULT_USER=admin \
  -e RABBITMQ_DEFAULT_PASS=password \
  rabbitmq:3-management

# เข้า Management UI: http://localhost:15672
# user: admin, password: password
```

```bash
npm install amqplib
```

### ขั้นตอนที่ 663: RabbitMQ Connection

```javascript
// src/config/rabbitmq.js
const amqp = require('amqplib');

let connection = null;
let channel = null;

async function connect() {
  try {
    connection = await amqp.connect({
      hostname: process.env.RABBITMQ_HOST || 'localhost',
      port: parseInt(process.env.RABBITMQ_PORT) || 5672,
      username: process.env.RABBITMQ_USER || 'admin',
      password: process.env.RABBITMQ_PASS || 'password',
      vhost: process.env.RABBITMQ_VHOST || '/',
    });

    channel = await connection.createChannel();

    // Set prefetch count (จำนวน messages ที่ process พร้อมกัน)
    await channel.prefetch(10);

    console.log('✅ Connected to RabbitMQ');

    // Handle connection errors
    connection.on('error', (err) => {
      console.error('RabbitMQ connection error:', err);
      reconnect();
    });

    connection.on('close', () => {
      console.log('RabbitMQ connection closed, reconnecting...');
      reconnect();
    });

    return channel;
  } catch (error) {
    console.error('Failed to connect to RabbitMQ:', error);
    throw error;
  }
}

async function reconnect() {
  setTimeout(async () => {
    try {
      await connect();
    } catch (err) {
      console.error('Reconnect failed:', err);
      reconnect();
    }
  }, 5000);
}

function getChannel() {
  if (!channel) throw new Error('RabbitMQ channel not initialized');
  return channel;
}

module.exports = { connect, getChannel };
```

### ขั้นตอนที่ 664: Producer - ส่ง Messages

```javascript
// src/producers/emailProducer.js
const { getChannel } = require('../config/rabbitmq');

const QUEUE_NAME = 'email_queue';
const EXCHANGE = 'notifications';

class EmailProducer {
  async setup() {
    const channel = getChannel();

    // สร้าง Exchange
    await channel.assertExchange(EXCHANGE, 'direct', { durable: true });

    // สร้าง Queue
    await channel.assertQueue(QUEUE_NAME, {
      durable: true,       // Queue อยู่รอดหลัง restart
      deadLetterExchange: 'dlx',  // Dead letter exchange
      messageTtl: 86400000,       // 24 ชั่วโมง
    });

    // Bind Queue กับ Exchange
    await channel.bindQueue(QUEUE_NAME, EXCHANGE, 'email');
  }

  async sendEmail(emailData) {
    const channel = getChannel();

    const message = {
      id: generateId(),
      ...emailData,
      createdAt: new Date().toISOString(),
    };

    const buffer = Buffer.from(JSON.stringify(message));

    channel.publish(EXCHANGE, 'email', buffer, {
      persistent: true,          // Message อยู่รอดหลัง restart
      contentType: 'application/json',
      correlationId: message.id,
      timestamp: Date.now(),
      headers: {
        priority: emailData.priority || 'normal',
      },
    });

    console.log(`📧 Email queued: ${message.id}`);
    return message.id;
  }

  async sendWelcomeEmail(userId, email) {
    return this.sendEmail({
      type: 'WELCOME',
      userId,
      to: email,
      template: 'welcome',
      data: { userId },
      priority: 'high',
    });
  }

  async sendPasswordResetEmail(userId, email, resetToken) {
    return this.sendEmail({
      type: 'PASSWORD_RESET',
      userId,
      to: email,
      template: 'password_reset',
      data: { resetToken, expiresIn: '1 ชั่วโมง' },
      priority: 'high',
    });
  }

  async sendNotificationEmail(userId, email, notification) {
    return this.sendEmail({
      type: 'NOTIFICATION',
      userId,
      to: email,
      template: 'notification',
      data: notification,
      priority: 'low',
    });
  }
}

function generateId() {
  return `email_${Date.now()}_${Math.random().toString(36).substr(2, 9)}`;
}

module.exports = new EmailProducer();
```

### ขั้นตอนที่ 665: Consumer - รับและประมวลผล Messages

```javascript
// src/consumers/emailConsumer.js
const { getChannel } = require('../config/rabbitmq');
const emailService = require('../services/emailService');

const QUEUE_NAME = 'email_queue';

class EmailConsumer {
  async start() {
    const channel = getChannel();

    console.log(`👂 Email consumer listening on ${QUEUE_NAME}`);

    channel.consume(QUEUE_NAME, async (msg) => {
      if (!msg) return;

      const startTime = Date.now();
      let emailData;

      try {
        emailData = JSON.parse(msg.content.toString());
        console.log(`📧 Processing email: ${emailData.id} (${emailData.type})`);

        // ประมวลผลตาม type
        await this.processEmail(emailData);

        // Acknowledge - บอก RabbitMQ ว่าประมวลผลสำเร็จ
        channel.ack(msg);

        const duration = Date.now() - startTime;
        console.log(`✅ Email ${emailData.id} processed in ${duration}ms`);

      } catch (error) {
        console.error(`❌ Failed to process email:`, error);

        // ตรวจสอบว่าควร retry หรือไม่
        const retryCount = (msg.properties.headers?.retryCount || 0) + 1;

        if (retryCount <= 3) {
          // Nack และ requeue หลัง delay
          console.log(`🔄 Retrying email (attempt ${retryCount}/3)`);
          channel.nack(msg, false, false); // ส่งไป DLX
        } else {
          // ล้มเหลวหมดแล้ว - ส่งไป dead letter queue
          console.error(`💀 Email ${emailData?.id} failed after 3 retries`);
          channel.nack(msg, false, false);
        }
      }
    }, {
      noAck: false, // Manual acknowledgment
    });
  }

  async processEmail(emailData) {
    switch (emailData.type) {
      case 'WELCOME':
        await emailService.sendWelcomeEmail(emailData.to, emailData.data);
        break;

      case 'PASSWORD_RESET':
        await emailService.sendPasswordResetEmail(emailData.to, emailData.data);
        break;

      case 'NOTIFICATION':
        await emailService.sendNotificationEmail(emailData.to, emailData.data);
        break;

      default:
        throw new Error(`Unknown email type: ${emailData.type}`);
    }
  }
}

module.exports = new EmailConsumer();
```

---

## Bull Queue

### ขั้นตอนที่ 666: ติดตั้งและตั้งค่า Bull

Bull เป็น Redis-based queue library ที่นิยมใช้กับ Node.js

```bash
npm install bull
npm install bull-board  # Dashboard UI
```

```javascript
// src/config/queues.js
const Bull = require('bull');

const defaultOptions = {
  redis: {
    host: process.env.REDIS_HOST || 'localhost',
    port: parseInt(process.env.REDIS_PORT) || 6379,
    password: process.env.REDIS_PASSWORD,
  },
  defaultJobOptions: {
    attempts: 3,
    backoff: {
      type: 'exponential',
      delay: 2000,
    },
    removeOnComplete: 100,  // เก็บ 100 completed jobs สุดท้าย
    removeOnFail: 50,       // เก็บ 50 failed jobs สุดท้าย
  },
};

// สร้าง queues
const emailQueue = new Bull('email', defaultOptions);
const imageQueue = new Bull('image-processing', defaultOptions);
const reportQueue = new Bull('report', {
  ...defaultOptions,
  defaultJobOptions: {
    ...defaultOptions.defaultJobOptions,
    attempts: 1,         // Report ไม่ retry
    timeout: 120000,     // 2 นาที timeout
  },
});
const notificationQueue = new Bull('notification', defaultOptions);

module.exports = { emailQueue, imageQueue, reportQueue, notificationQueue };
```

### ขั้นตอนที่ 667: เพิ่ม Jobs ใน Queue

```javascript
// src/services/queueService.js
const { emailQueue, imageQueue, reportQueue } = require('../config/queues');

class QueueService {
  // Email Jobs
  async queueEmail(data, options = {}) {
    const job = await emailQueue.add(data, {
      priority: options.priority || 2,
      delay: options.delay || 0,
      attempts: options.attempts || 3,
    });

    console.log(`Job added to email queue: ${job.id}`);
    return job;
  }

  async queueWelcomeEmail(user) {
    return this.queueEmail({
      type: 'WELCOME',
      to: user.email,
      data: { name: user.username },
    }, { priority: 1 }); // High priority
  }

  // Image Processing Jobs
  async queueImageResize(imageData) {
    return imageQueue.add({
      type: 'RESIZE',
      ...imageData,
    }, {
      attempts: 2,
      timeout: 30000, // 30 วินาที
    });
  }

  async queueImageOptimize(imageData) {
    return imageQueue.add({
      type: 'OPTIMIZE',
      ...imageData,
    });
  }

  // Report Generation
  async queueReport(reportData) {
    return reportQueue.add({
      type: 'GENERATE',
      ...reportData,
    }, {
      timeout: 300000, // 5 นาที
    });
  }

  // Bulk operations
  async bulkQueueEmails(emails) {
    const jobs = emails.map(email => ({
      data: email,
      opts: { attempts: 3 },
    }));

    return emailQueue.addBulk(jobs);
  }
}

module.exports = new QueueService();
```

### ขั้นตอนที่ 668: Workers - ประมวลผล Jobs

```javascript
// src/workers/emailWorker.js
const { emailQueue } = require('../config/queues');
const nodemailer = require('nodemailer');

// สร้าง email transporter
const transporter = nodemailer.createTransporter({
  host: process.env.SMTP_HOST,
  port: parseInt(process.env.SMTP_PORT) || 587,
  secure: false,
  auth: {
    user: process.env.SMTP_USER,
    pass: process.env.SMTP_PASS,
  },
});

// กำหนด processor
emailQueue.process('*', 5, async (job) => {
  // 5 concurrent jobs

  const { type, to, data } = job.data;
  console.log(`Processing email job ${job.id}: ${type}`);

  // อัปเดต progress
  await job.progress(10);

  try {
    const emailContent = await generateEmailContent(type, data);
    await job.progress(50);

    await transporter.sendMail({
      from: process.env.FROM_EMAIL,
      to,
      subject: emailContent.subject,
      html: emailContent.html,
      text: emailContent.text,
    });

    await job.progress(100);
    console.log(`✅ Email sent to ${to}`);

    return { sent: true, to, type };
  } catch (error) {
    console.error(`❌ Failed to send email to ${to}:`, error.message);
    throw error; // Bull จะ retry อัตโนมัติ
  }
});

// Event handlers
emailQueue.on('completed', (job, result) => {
  console.log(`✅ Job ${job.id} completed:`, result);
});

emailQueue.on('failed', (job, error) => {
  console.error(`❌ Job ${job.id} failed (attempt ${job.attemptsMade}/${job.opts.attempts}):`, error.message);
});

emailQueue.on('stalled', (job) => {
  console.warn(`⚠️ Job ${job.id} stalled`);
});

emailQueue.on('progress', (job, progress) => {
  console.log(`📊 Job ${job.id} progress: ${progress}%`);
});

async function generateEmailContent(type, data) {
  const templates = {
    WELCOME: {
      subject: 'ยินดีต้อนรับสู่ระบบ!',
      html: `<h1>ยินดีต้อนรับ, ${data.name}!</h1><p>ขอบคุณที่สมัครสมาชิก</p>`,
      text: `ยินดีต้อนรับ, ${data.name}! ขอบคุณที่สมัครสมาชิก`,
    },
    PASSWORD_RESET: {
      subject: 'รีเซ็ตรหัสผ่าน',
      html: `<p>คลิกลิงก์นี้เพื่อรีเซ็ตรหัสผ่าน: <a href="${data.resetUrl}">${data.resetUrl}</a></p>`,
      text: `รีเซ็ตรหัสผ่านที่: ${data.resetUrl}`,
    },
  };

  return templates[type] || { subject: 'Notification', html: JSON.stringify(data), text: JSON.stringify(data) };
}

console.log('📧 Email worker started');
```

```javascript
// src/workers/imageWorker.js
const { imageQueue } = require('../config/queues');
const sharp = require('sharp');
const path = require('path');
const fs = require('fs').promises;

imageQueue.process('*', 3, async (job) => {
  const { type, inputPath, outputPath, options } = job.data;

  console.log(`Processing image job ${job.id}: ${type}`);
  await job.progress(10);

  switch (type) {
    case 'RESIZE':
      await resizeImage(inputPath, outputPath, options);
      break;
    case 'OPTIMIZE':
      await optimizeImage(inputPath, outputPath, options);
      break;
    case 'THUMBNAIL':
      await createThumbnail(inputPath, outputPath);
      break;
    default:
      throw new Error(`Unknown image job type: ${type}`);
  }

  await job.progress(100);
  return { processed: true, outputPath };
});

async function resizeImage(input, output, { width, height, fit = 'cover' }) {
  await sharp(input)
    .resize(width, height, { fit })
    .toFile(output);
}

async function optimizeImage(input, output, options = {}) {
  const image = sharp(input);
  const metadata = await image.metadata();

  if (metadata.format === 'jpeg') {
    await image.jpeg({ quality: options.quality || 80 }).toFile(output);
  } else if (metadata.format === 'png') {
    await image.png({ compressionLevel: 8 }).toFile(output);
  } else if (metadata.format === 'webp') {
    await image.webp({ quality: options.quality || 80 }).toFile(output);
  }
}

async function createThumbnail(input, output) {
  await sharp(input)
    .resize(200, 200, { fit: 'cover' })
    .jpeg({ quality: 70 })
    .toFile(output);
}

console.log('🖼️ Image worker started');
```

---

## Job Processing Patterns

### ขั้นตอนที่ 669: Job Chaining

```javascript
// ทำ tasks ต่อเนื่องกัน
async function processUserRegistration(userId) {
  const user = await User.findById(userId);

  // Chain of jobs
  const welcomeEmailJob = await emailQueue.add({
    type: 'WELCOME',
    to: user.email,
    data: { name: user.username },
  });

  // ทำหลังจาก welcome email ส่งสำเร็จ
  welcomeEmailJob.finished().then(async () => {
    await notificationQueue.add({
      type: 'IN_APP',
      userId: user._id,
      message: 'ยินดีต้อนรับ! เริ่มต้นใช้งานได้เลย',
    });
  });

  // สร้าง default profile picture
  await imageQueue.add({
    type: 'GENERATE_AVATAR',
    userId: user._id,
    username: user.username,
  });

  return { queued: true };
}
```

### ขั้นตอนที่ 670: Job Dependencies และ Workflows

```javascript
// src/workflows/orderWorkflow.js
const { emailQueue, notificationQueue } = require('../config/queues');

class OrderWorkflow {
  async processOrder(orderId) {
    // Step 1: ตรวจสอบ inventory
    const inventoryJob = await createJob('inventory:check', { orderId });
    const inventoryResult = await inventoryJob.finished();

    if (!inventoryResult.available) {
      await createJob('email:out-of-stock', {
        orderId,
        userId: inventoryResult.userId,
      });
      return { success: false, reason: 'OUT_OF_STOCK' };
    }

    // Step 2: ประมวลผลการชำระเงิน
    const paymentJob = await createJob('payment:process', { orderId });
    const paymentResult = await paymentJob.finished();

    if (!paymentResult.success) {
      await createJob('email:payment-failed', { orderId });
      return { success: false, reason: 'PAYMENT_FAILED' };
    }

    // Step 3: อัปเดต inventory
    await createJob('inventory:update', { orderId });

    // Step 4: ส่ง confirmation email
    await emailQueue.add({
      type: 'ORDER_CONFIRMATION',
      orderId,
      email: paymentResult.email,
    });

    // Step 5: แจ้งเตือน
    await notificationQueue.add({
      type: 'ORDER_PLACED',
      userId: paymentResult.userId,
      orderId,
    });

    return { success: true };
  }
}
```

---

## Retry Logic และ Error Handling

### ขั้นตอนที่ 671: Exponential Backoff

```javascript
// Retry strategy configurations
const retryStrategies = {
  // Exponential backoff: 2s, 4s, 8s, 16s
  email: {
    attempts: 4,
    backoff: {
      type: 'exponential',
      delay: 2000,
    },
  },

  // Linear backoff: 5s, 10s, 15s
  payment: {
    attempts: 3,
    backoff: {
      type: 'fixed',
      delay: 5000,
    },
  },

  // Custom backoff
  api: {
    attempts: 5,
    backoff: {
      type: 'custom',
      // delay(attemptsMade, err) → ระยะเวลา delay ใน ms
    },
  },
};

// Custom backoff function
emailQueue.process(async (job) => {
  // ...
});

emailQueue.on('failed', async (job, error) => {
  // Custom error handling
  if (error.code === 'RATE_LIMITED') {
    // รอ longer สำหรับ rate limit errors
    await job.update({
      ...job.data,
      retryAfter: Date.now() + 60000,
    });
  }

  // Alert ถ้าล้มเหลวเกิน threshold
  if (job.attemptsMade >= job.opts.attempts) {
    await alertService.send({
      level: 'critical',
      message: `Job ${job.id} failed after all retries`,
      error: error.message,
      data: job.data,
    });
  }
});
```

### ขั้นตอนที่ 672: Dead Letter Queue

```javascript
// Dead Letter Queue - เก็บ jobs ที่ล้มเหลว
const dlQueue = new Bull('dead-letter', redisConfig);

// เพิ่ม jobs ที่ล้มเหลวไปยัง DLQ
emailQueue.on('failed', async (job, error) => {
  if (job.attemptsMade >= job.opts.attempts) {
    console.log(`Moving job ${job.id} to dead letter queue`);

    await dlQueue.add({
      originalQueue: 'email',
      originalJobId: job.id,
      jobData: job.data,
      error: {
        message: error.message,
        stack: error.stack,
      },
      failedAt: new Date().toISOString(),
      attempts: job.attemptsMade,
    });
  }
});

// Reprocess DLQ jobs manually
async function reprocessDeadLetterJobs(queueName) {
  const jobs = await dlQueue.getJobs(['waiting', 'active', 'completed', 'failed']);

  const targetJobs = jobs.filter(j => j.data.originalQueue === queueName);

  for (const dlJob of targetJobs) {
    const targetQueue = getQueue(dlJob.data.originalQueue);
    await targetQueue.add(dlJob.data.jobData);
    await dlJob.remove();
    console.log(`Requeued job ${dlJob.data.originalJobId}`);
  }
}
```

---

## Priority Queues

### ขั้นตอนที่ 673: Job Priorities

```javascript
// Bull priority: 1 (สูงสุด) ถึง MAX_INT (ต่ำสุด)
const PRIORITIES = {
  CRITICAL: 1,
  HIGH: 2,
  NORMAL: 5,
  LOW: 10,
  BACKGROUND: 100,
};

// ส่ง emails ตาม priority
async function queueEmail(emailData) {
  const priority = {
    PASSWORD_RESET: PRIORITIES.CRITICAL,
    WELCOME: PRIORITIES.HIGH,
    ORDER_CONFIRMATION: PRIORITIES.HIGH,
    NEWSLETTER: PRIORITIES.LOW,
    PROMOTION: PRIORITIES.BACKGROUND,
  }[emailData.type] || PRIORITIES.NORMAL;

  return emailQueue.add(emailData, { priority });
}

// Worker จะประมวลผล jobs ตาม priority อัตโนมัติ
```

---

## Scheduled Jobs

### ขั้นตอนที่ 674: Delayed Jobs

```javascript
// ส่งงานหลังจากเวลาที่กำหนด
async function scheduleJobs() {
  // ส่ง email หลัง 1 ชั่วโมง
  await emailQueue.add(
    { type: 'FOLLOW_UP', userId: '123' },
    { delay: 60 * 60 * 1000 }  // 1 ชั่วโมง
  );

  // ส่ง reminder ในวันพรุ่งนี้
  const tomorrow = new Date();
  tomorrow.setDate(tomorrow.getDate() + 1);
  tomorrow.setHours(9, 0, 0, 0);

  const delayMs = tomorrow.getTime() - Date.now();
  await emailQueue.add(
    { type: 'DAILY_REMINDER', userId: '456' },
    { delay: delayMs }
  );
}
```

### ขั้นตอนที่ 675: Repeatable Jobs (Cron)

```javascript
// Repeatable Jobs ด้วย Cron expressions
async function setupScheduledJobs() {
  // ทุกวันตี 2 - cleanup old data
  await reportQueue.add(
    { type: 'CLEANUP' },
    {
      repeat: { cron: '0 2 * * *' }, // ทุกวัน เวลา 02:00
    }
  );

  // ทุก 30 นาที - ส่ง digest emails
  await emailQueue.add(
    { type: 'DIGEST' },
    {
      repeat: { cron: '*/30 * * * *' },
      removeOnComplete: 10,
    }
  );

  // ทุกวันจันทร์ - weekly report
  await reportQueue.add(
    { type: 'WEEKLY_REPORT' },
    {
      repeat: {
        cron: '0 9 * * 1',  // ทุกวันจันทร์ 09:00
      },
    }
  );

  // ทุกชั่วโมง - backup
  await createJob('backup', {}, {
    repeat: {
      every: 60 * 60 * 1000,  // ทุก 1 ชั่วโมง
    },
  });
}

// ดู repeatable jobs
async function listScheduledJobs() {
  const repeatableJobs = await emailQueue.getRepeatableJobs();
  console.log('Scheduled jobs:', repeatableJobs);
}

// ลบ repeatable job
async function removeScheduledJob(jobKey) {
  await emailQueue.removeRepeatable({ key: jobKey });
}
```

---

## Monitoring Queues

### ขั้นตอนที่ 676: Bull Board Dashboard

```javascript
// src/admin/bull-board.js
const { createBullBoard } = require('@bull-board/api');
const { BullAdapter } = require('@bull-board/api/bullAdapter');
const { ExpressAdapter } = require('@bull-board/express');
const { emailQueue, imageQueue, reportQueue } = require('../config/queues');

const serverAdapter = new ExpressAdapter();
serverAdapter.setBasePath('/admin/queues');

const { addQueue, removeQueue, setQueues, replaceQueues } = createBullBoard({
  queues: [
    new BullAdapter(emailQueue),
    new BullAdapter(imageQueue),
    new BullAdapter(reportQueue),
  ],
  serverAdapter,
});

module.exports = { serverAdapter };
```

```javascript
// app.js - เพิ่ม Bull Board
const { serverAdapter } = require('./admin/bull-board');

// เฉพาะ admin
app.use('/admin/queues', requireAdmin, serverAdapter.getRouter());
```

### ขั้นตอนที่ 677: Custom Queue Metrics

```javascript
// src/monitoring/queueMetrics.js
const { emailQueue, imageQueue, reportQueue } = require('../config/queues');

async function getQueueStats() {
  const queues = { emailQueue, imageQueue, reportQueue };
  const stats = {};

  for (const [name, queue] of Object.entries(queues)) {
    const [waiting, active, completed, failed, delayed] = await Promise.all([
      queue.getWaitingCount(),
      queue.getActiveCount(),
      queue.getCompletedCount(),
      queue.getFailedCount(),
      queue.getDelayedCount(),
    ]);

    stats[name] = {
      waiting,
      active,
      completed,
      failed,
      delayed,
      total: waiting + active + completed + failed + delayed,
    };
  }

  return stats;
}

// Metrics endpoint
app.get('/admin/queue-stats', requireAdmin, async (req, res) => {
  const stats = await getQueueStats();
  res.json(stats);
});

// Alert เมื่อ queue มีงานค้างมาก
async function monitorQueues() {
  setInterval(async () => {
    const stats = await getQueueStats();

    for (const [name, stat] of Object.entries(stats)) {
      if (stat.waiting > 1000) {
        console.warn(`⚠️ Queue ${name} has ${stat.waiting} waiting jobs!`);
        // ส่ง alert
      }

      if (stat.failed > 100) {
        console.error(`❌ Queue ${name} has ${stat.failed} failed jobs!`);
        // ส่ง alert
      }
    }
  }, 60000); // ทุก 1 นาที
}
```

---

## Advanced Patterns

### ขั้นตอนที่ 678: Circuit Breaker Pattern

```javascript
// ป้องกัน cascade failures
class CircuitBreaker {
  constructor(failureThreshold = 5, resetTimeout = 60000) {
    this.failureCount = 0;
    this.failureThreshold = failureThreshold;
    this.resetTimeout = resetTimeout;
    this.state = 'CLOSED'; // CLOSED, OPEN, HALF_OPEN
    this.nextAttempt = Date.now();
  }

  async call(fn) {
    if (this.state === 'OPEN') {
      if (Date.now() < this.nextAttempt) {
        throw new Error('Circuit breaker is OPEN');
      }
      this.state = 'HALF_OPEN';
    }

    try {
      const result = await fn();
      this.onSuccess();
      return result;
    } catch (error) {
      this.onFailure();
      throw error;
    }
  }

  onSuccess() {
    this.failureCount = 0;
    this.state = 'CLOSED';
  }

  onFailure() {
    this.failureCount++;
    if (this.failureCount >= this.failureThreshold) {
      this.state = 'OPEN';
      this.nextAttempt = Date.now() + this.resetTimeout;
      console.warn(`⚡ Circuit breaker OPENED after ${this.failureCount} failures`);
    }
  }
}

const emailBreaker = new CircuitBreaker(5, 30000);

// ใช้กับ worker
emailQueue.process(async (job) => {
  return emailBreaker.call(async () => {
    await sendEmail(job.data);
  });
});
```

### ขั้นตอนที่ 679: Batch Processing

```javascript
// ประมวลผล jobs เป็น batch
class BatchProcessor {
  constructor(queue, batchSize = 10, processFn) {
    this.queue = queue;
    this.batchSize = batchSize;
    this.processFn = processFn;
    this.buffer = [];
    this.timer = null;
  }

  async add(item) {
    this.buffer.push(item);

    if (this.buffer.length >= this.batchSize) {
      await this.flush();
    } else if (!this.timer) {
      // Flush หลัง 1 วินาที ถ้ายังไม่ถึง batchSize
      this.timer = setTimeout(() => this.flush(), 1000);
    }
  }

  async flush() {
    if (this.timer) {
      clearTimeout(this.timer);
      this.timer = null;
    }

    const batch = this.buffer.splice(0, this.batchSize);
    if (batch.length === 0) return;

    console.log(`Processing batch of ${batch.length} items`);
    await this.processFn(batch);
  }
}

// ตัวอย่างใช้งาน
const emailBatch = new BatchProcessor(emailQueue, 50, async (emails) => {
  await sendBulkEmails(emails);
});
```

### ขั้นตอนที่ 680: Worker Scaling

```javascript
// src/workers/worker-manager.js
const cluster = require('cluster');
const os = require('os');

if (cluster.isMaster) {
  const numWorkers = parseInt(process.env.NUM_WORKERS) || os.cpus().length;
  console.log(`Starting ${numWorkers} workers...`);

  for (let i = 0; i < numWorkers; i++) {
    cluster.fork();
  }

  cluster.on('exit', (worker, code, signal) => {
    console.log(`Worker ${worker.process.pid} died. Restarting...`);
    cluster.fork();
  });
} else {
  // Worker process
  require('./emailWorker');
  require('./imageWorker');
  console.log(`Worker ${process.pid} started`);
}
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: PDF Generation Queue

```javascript
// TODO: สร้าง PDF generation queue
// - รับ order data
// - สร้าง PDF invoice
// - ส่ง email พร้อม PDF attachment
// ใช้ puppeteer หรือ pdfkit
```

### แบบฝึกหัดที่ 2: Notification System

```javascript
// TODO: สร้าง notification queue ที่:
// - ส่ง push notifications ไปยัง mobile devices
// - ส่ง in-app notifications
// - ส่ง SMS notifications
// ตาม user preferences
```

### แบบฝึกหัดที่ 3: Data Import Queue

```javascript
// TODO: รับ CSV file และประมวลผลเป็น batch
// - อ่าน CSV file
// - Validate แต่ละ row
// - บันทึกลง database เป็น batch (100 rows ต่อครั้ง)
// - รายงานผลเมื่อเสร็จสิ้น
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **Message Queue Concepts** - ทำไมต้องใช้ และข้อดีต่างๆ
2. **RabbitMQ** - Exchange, Queue, Consumer patterns
3. **Bull Queue** - Redis-based queue สำหรับ Node.js
4. **Job Processing** - Workers, progress tracking, chaining
5. **Retry Logic** - Exponential backoff, dead letter queues
6. **Scheduled Jobs** - Delayed jobs, cron jobs
7. **Monitoring** - Bull Board, metrics, alerting
8. **Advanced Patterns** - Circuit breaker, batch processing, scaling

ในบทถัดไปเราจะเรียนรู้ **Microservices Architecture** - service discovery, API gateway, inter-service communication
