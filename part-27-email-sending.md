# Part 27: Email Sending

> ขั้นตอนที่ 27-30 จาก 1000 — การส่ง Email ด้วย Nodemailer, SendGrid และ Queue

---

## สารบัญ

1. [Nodemailer Setup](#nodemailer-setup)
2. [SMTP Configuration](#smtp-configuration)
3. [HTML Email Templates](#html-email-templates)
4. [Email Attachments](#email-attachments)
5. [SendGrid API](#sendgrid-api)
6. [Email Queue](#email-queue)
7. [Practical: Email Verification, Password Reset](#practical-email-verification-password-reset)
8. [แบบฝึกหัด](#แบบฝึกหัด)

---

## Nodemailer Setup

### ติดตั้ง

```bash
npm install nodemailer
npm install @sendgrid/mail       # SendGrid
npm install bull                 # Redis-based queue
npm install handlebars           # template engine
npm install mjml                 # email HTML framework
```

### Nodemailer Transport

```javascript
// config/email.js
const nodemailer = require('nodemailer');

let transporter;

// ตรวจสอบ environment
if (process.env.NODE_ENV === 'production') {
  // Production: ใช้ SMTP จริง (Mailgun, SendGrid, SES)
  transporter = nodemailer.createTransporter({
    service: process.env.EMAIL_SERVICE,  // 'gmail', 'SendGrid', 'Mailgun'
    auth: {
      user: process.env.EMAIL_USER,
      pass: process.env.EMAIL_PASS,
    },
    pool: true,         // ใช้ connection pool
    maxConnections: 5,
    maxMessages: 100,
  });
} else if (process.env.NODE_ENV === 'test') {
  // Test: ไม่ส่งจริง ใช้ Ethereal.email
  const createTestTransport = async () => {
    const testAccount = await nodemailer.createTestAccount();
    return nodemailer.createTransport({
      host: 'smtp.ethereal.email',
      port: 587,
      secure: false,
      auth: {
        user: testAccount.user,
        pass: testAccount.pass,
      },
    });
  };
  // เรียกแบบ async
  createTestTransport().then((t) => { transporter = t; });
} else {
  // Development: ใช้ Mailhog หรือ MailDev (local SMTP)
  transporter = nodemailer.createTransport({
    host: 'localhost',
    port: 1025,
    secure: false,
    ignoreTLS: true,
  });
}

// ทดสอบการเชื่อมต่อ
const verifyConnection = async () => {
  try {
    await transporter.verify();
    console.log('Email server ready');
  } catch (error) {
    console.error('Email server not available:', error.message);
  }
};

module.exports = { transporter, verifyConnection };
```

---

## SMTP Configuration

### ตัวอย่าง SMTP Providers

```env
# Gmail (ต้องเปิด 2FA และสร้าง App Password)
EMAIL_SERVICE=gmail
EMAIL_USER=yourapp@gmail.com
EMAIL_PASS=your-app-specific-password

# Mailgun
MAILGUN_SMTP_HOST=smtp.mailgun.org
MAILGUN_SMTP_PORT=587
MAILGUN_SMTP_USER=postmaster@mg.yourdomain.com
MAILGUN_SMTP_PASS=your-mailgun-smtp-password

# AWS SES
SES_SMTP_HOST=email-smtp.ap-southeast-1.amazonaws.com
SES_SMTP_PORT=587
SES_SMTP_USER=AKIAIOSFODNN7EXAMPLE
SES_SMTP_PASS=your-ses-smtp-password

# SendGrid via SMTP
SENDGRID_SMTP_HOST=smtp.sendgrid.net
SENDGRID_SMTP_PORT=587
SENDGRID_SMTP_USER=apikey
SENDGRID_SMTP_PASS=SG.your-api-key
```

### Custom SMTP Config

```javascript
// config/email.js (updated)
const nodemailer = require('nodemailer');

const createTransporter = () => {
  const config = {
    host: process.env.SMTP_HOST,
    port: parseInt(process.env.SMTP_PORT) || 587,
    secure: process.env.SMTP_SECURE === 'true',  // true = port 465
    auth: {
      user: process.env.SMTP_USER,
      pass: process.env.SMTP_PASS,
    },
    
    // TLS options
    tls: {
      rejectUnauthorized: process.env.NODE_ENV === 'production',
    },
    
    // Connection pool
    pool: true,
    maxConnections: parseInt(process.env.SMTP_MAX_CONNECTIONS) || 5,
    maxMessages: 100,
    rateDelta: 1000,   // ms
    rateLimit: 10,     // max messages per rateDelta
  };
  
  return nodemailer.createTransport(config);
};

module.exports = { createTransporter };
```

---

## HTML Email Templates

### Template ด้วย Handlebars

```javascript
// utils/emailTemplates.js
const handlebars = require('handlebars');
const fs = require('fs');
const path = require('path');

const TEMPLATES_DIR = path.join(__dirname, '../templates/emails');

// Cache templates
const templateCache = new Map();

/**
 * โหลดและ compile template
 */
const getTemplate = (name) => {
  if (!templateCache.has(name)) {
    const filepath = path.join(TEMPLATES_DIR, `${name}.hbs`);
    const source = fs.readFileSync(filepath, 'utf8');
    templateCache.set(name, handlebars.compile(source));
  }
  return templateCache.get(name);
};

// Register helpers
handlebars.registerHelper('formatDate', (date) => {
  return new Date(date).toLocaleDateString('th-TH', {
    year: 'numeric',
    month: 'long',
    day: 'numeric',
  });
});

handlebars.registerHelper('ifEqual', function (a, b, options) {
  return a === b ? options.fn(this) : options.inverse(this);
});

/**
 * Render template
 */
exports.renderTemplate = (name, data) => {
  const template = getTemplate(name);
  return template(data);
};
```

### Email Template (Handlebars)

```html
<!-- templates/emails/verify-email.hbs -->
<!DOCTYPE html>
<html lang="th">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>ยืนยัน Email ของคุณ</title>
  <style>
    /* Inline CSS เพราะ email clients ไม่รองรับ external CSS */
    body {
      font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif;
      background-color: #f4f4f4;
      margin: 0;
      padding: 0;
    }
    .container {
      max-width: 600px;
      margin: 40px auto;
      background: #ffffff;
      border-radius: 8px;
      overflow: hidden;
      box-shadow: 0 2px 8px rgba(0,0,0,0.1);
    }
    .header {
      background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
      padding: 32px;
      text-align: center;
    }
    .header h1 {
      color: #ffffff;
      margin: 0;
      font-size: 24px;
    }
    .body {
      padding: 32px;
      color: #333333;
      line-height: 1.6;
    }
    .button {
      display: inline-block;
      background: #667eea;
      color: #ffffff !important;
      text-decoration: none;
      padding: 14px 32px;
      border-radius: 6px;
      font-weight: 600;
      font-size: 16px;
      margin: 24px 0;
    }
    .footer {
      padding: 24px 32px;
      background: #f8f8f8;
      color: #999999;
      font-size: 12px;
      text-align: center;
    }
    .expiry-note {
      background: #fff3cd;
      border: 1px solid #ffc107;
      border-radius: 4px;
      padding: 12px 16px;
      font-size: 13px;
      color: #856404;
      margin: 16px 0;
    }
  </style>
</head>
<body>
  <div class="container">
    <div class="header">
      <h1>ยืนยัน Email ของคุณ</h1>
    </div>
    <div class="body">
      <p>สวัสดี <strong>{{name}}</strong>,</p>
      <p>ขอบคุณที่สมัครสมาชิกกับ <strong>{{appName}}</strong>!</p>
      <p>กรุณากดปุ่มด้านล่างเพื่อยืนยัน email ของคุณ:</p>
      
      <div style="text-align: center;">
        <a href="{{verifyUrl}}" class="button">ยืนยัน Email</a>
      </div>
      
      <div class="expiry-note">
        ⚠️ ลิงก์นี้จะหมดอายุใน <strong>24 ชั่วโมง</strong>
      </div>
      
      <p>ถ้าคุณไม่ได้สมัครสมาชิก ไม่ต้องทำอะไร</p>
      
      <p>หรือคัดลอก URL นี้ไปวางในเบราว์เซอร์:</p>
      <p style="word-break: break-all; color: #667eea; font-size: 13px;">
        {{verifyUrl}}
      </p>
    </div>
    <div class="footer">
      <p>© {{year}} {{appName}} — ส่งโดย {{supportEmail}}</p>
      <p>ถ้าไม่ต้องการรับ email กรุณา <a href="{{unsubscribeUrl}}">ยกเลิก</a></p>
    </div>
  </div>
</body>
</html>
```

```html
<!-- templates/emails/password-reset.hbs -->
<!DOCTYPE html>
<html lang="th">
<head>
  <meta charset="UTF-8">
  <title>รีเซ็ตรหัสผ่าน</title>
  <style>
    /* ... base styles ... */
    .warning-box {
      background: #fee2e2;
      border: 1px solid #fca5a5;
      border-radius: 4px;
      padding: 12px 16px;
      color: #991b1b;
      font-size: 13px;
      margin: 16px 0;
    }
  </style>
</head>
<body>
  <div class="container">
    <div class="header" style="background: linear-gradient(135deg, #f59e0b, #d97706);">
      <h1>รีเซ็ตรหัสผ่าน</h1>
    </div>
    <div class="body">
      <p>สวัสดี <strong>{{name}}</strong>,</p>
      <p>เราได้รับคำขอรีเซ็ตรหัสผ่านสำหรับบัญชี <strong>{{email}}</strong></p>
      
      <div style="text-align: center;">
        <a href="{{resetUrl}}" class="button" style="background: #f59e0b;">
          รีเซ็ตรหัสผ่าน
        </a>
      </div>
      
      <div class="warning-box">
        ⚠️ ลิงก์นี้จะหมดอายุใน <strong>1 ชั่วโมง</strong>
      </div>
      
      <p><strong>ถ้าคุณไม่ได้ขอรีเซ็ตรหัสผ่าน</strong> กรุณาเพิกเฉยต่อ email นี้ 
      รหัสผ่านของคุณจะไม่เปลี่ยนแปลง</p>
      
      <p>เพื่อความปลอดภัย ห้ามแชร์ลิงก์นี้กับใคร</p>
    </div>
    <div class="footer">
      <p>© {{year}} {{appName}}</p>
      <p>คำขอมาจาก IP: {{requestIp}}</p>
    </div>
  </div>
</body>
</html>
```

---

## Email Attachments

```javascript
// ส่ง email พร้อม attachment
const sendEmailWithAttachment = async ({ to, subject, template, data, attachments }) => {
  const html = renderTemplate(template, data);
  
  const mailOptions = {
    from: `"${process.env.EMAIL_FROM_NAME}" <${process.env.EMAIL_FROM}>`,
    to,
    subject,
    html,
    
    attachments: attachments?.map((attachment) => {
      if (attachment.path) {
        // ไฟล์จาก disk
        return {
          filename: attachment.filename,
          path: attachment.path,
          contentType: attachment.contentType,
        };
      }
      
      if (attachment.buffer) {
        // Buffer
        return {
          filename: attachment.filename,
          content: attachment.buffer,
          contentType: attachment.contentType,
          encoding: 'base64',
        };
      }
      
      if (attachment.url) {
        // URL
        return {
          filename: attachment.filename,
          href: attachment.url,
        };
      }
    }),
  };
  
  return transporter.sendMail(mailOptions);
};

// ตัวอย่าง: ส่ง invoice PDF
const sendInvoice = async (user, order, pdfBuffer) => {
  await sendEmailWithAttachment({
    to: user.email,
    subject: `ใบเสร็จ Order #${order.orderNumber}`,
    template: 'invoice',
    data: {
      name: user.name,
      order,
      year: new Date().getFullYear(),
      appName: process.env.APP_NAME,
    },
    attachments: [
      {
        filename: `invoice-${order.orderNumber}.pdf`,
        buffer: pdfBuffer,
        contentType: 'application/pdf',
      },
    ],
  });
};
```

---

## SendGrid API

### ตั้งค่า SendGrid

```bash
npm install @sendgrid/mail
```

```javascript
// services/sendgridService.js
const sgMail = require('@sendgrid/mail');

sgMail.setApiKey(process.env.SENDGRID_API_KEY);

class SendGridService {
  constructor() {
    this.from = {
      email: process.env.EMAIL_FROM,
      name: process.env.EMAIL_FROM_NAME || 'MyApp',
    };
  }
  
  /**
   * ส่ง email ด้วย dynamic template
   */
  async sendWithTemplate(options) {
    const { to, templateId, dynamicData, attachments } = options;
    
    const msg = {
      to,
      from: this.from,
      templateId,
      dynamicTemplateData: dynamicData,
    };
    
    if (attachments?.length) {
      msg.attachments = attachments.map((att) => ({
        content: att.buffer.toString('base64'),
        filename: att.filename,
        type: att.contentType,
        disposition: 'attachment',
      }));
    }
    
    try {
      const [response] = await sgMail.send(msg);
      return { success: true, statusCode: response.statusCode };
    } catch (error) {
      console.error('SendGrid error:', error.response?.body?.errors);
      throw error;
    }
  }
  
  /**
   * ส่ง email หลายคนพร้อมกัน (bulk)
   */
  async sendBulk(recipients, options) {
    const { subject, html } = options;
    
    // SendGrid อนุญาต max 1000 recipients ต่อ request
    const batches = [];
    for (let i = 0; i < recipients.length; i += 1000) {
      batches.push(recipients.slice(i, i + 1000));
    }
    
    for (const batch of batches) {
      const messages = batch.map((recipient) => ({
        to: recipient.email,
        from: this.from,
        subject,
        html: html.replace(/\{\{name\}\}/g, recipient.name),
      }));
      
      await sgMail.send(messages);
    }
  }
  
  /**
   * ตรวจสอบ email bounce/unsubscribe
   */
  async checkEmailStatus(email) {
    const response = await fetch(
      `https://api.sendgrid.com/v3/suppression/bounces/${email}`,
      {
        headers: {
          Authorization: `Bearer ${process.env.SENDGRID_API_KEY}`,
        },
      }
    );
    
    return response.ok;
  }
}

module.exports = new SendGridService();
```

---

## Email Queue

### Bull Queue สำหรับ Email

```bash
npm install bull redis
```

```javascript
// queues/emailQueue.js
const Bull = require('bull');
const emailService = require('../services/emailService');

// สร้าง queue
const emailQueue = new Bull('email', {
  redis: {
    host: process.env.REDIS_HOST || 'localhost',
    port: parseInt(process.env.REDIS_PORT) || 6379,
    password: process.env.REDIS_PASSWORD,
  },
  defaultJobOptions: {
    attempts: 3,               // retry 3 ครั้ง
    backoff: {
      type: 'exponential',     // wait 1s, 2s, 4s
      delay: 1000,
    },
    removeOnComplete: 100,     // เก็บ 100 jobs ที่สำเร็จ
    removeOnFail: 200,         // เก็บ 200 jobs ที่ fail
  },
});

// Process jobs
emailQueue.process('send-email', 5, async (job) => {
  const { to, subject, template, data } = job.data;
  
  job.progress(10);
  
  await emailService.send({ to, subject, template, data });
  
  job.progress(100);
  
  return { sent: true, to };
});

// Event handlers
emailQueue.on('completed', (job, result) => {
  console.log(`Email sent to ${result.to} [Job ${job.id}]`);
});

emailQueue.on('failed', (job, err) => {
  console.error(`Email failed [Job ${job.id}]:`, err.message);
});

emailQueue.on('stalled', (job) => {
  console.warn(`Email job stalled [Job ${job.id}]`);
});

// Helper functions
const emailQueueHelpers = {
  /**
   * เพิ่ม email เข้า queue
   */
  async add(to, subject, template, data, options = {}) {
    return emailQueue.add(
      'send-email',
      { to, subject, template, data },
      {
        ...options,
        delay: options.delay || 0,  // delay ก่อนส่ง (ms)
        priority: options.priority || 0,  // ลำดับความสำคัญ (ต่ำ = สำคัญกว่า)
      }
    );
  },
  
  /**
   * Schedule email สำหรับอนาคต
   */
  async schedule(to, subject, template, data, sendAt) {
    const delay = sendAt.getTime() - Date.now();
    return this.add(to, subject, template, data, { delay });
  },
  
  /**
   * ดู queue stats
   */
  async getStats() {
    const [waiting, active, completed, failed] = await Promise.all([
      emailQueue.getWaitingCount(),
      emailQueue.getActiveCount(),
      emailQueue.getCompletedCount(),
      emailQueue.getFailedCount(),
    ]);
    
    return { waiting, active, completed, failed };
  },
};

module.exports = { emailQueue, emailQueueHelpers };
```

### Email Service

```javascript
// services/emailService.js
const { transporter } = require('../config/email');
const { renderTemplate } = require('../utils/emailTemplates');

class EmailService {
  constructor() {
    this.from = `"${process.env.EMAIL_FROM_NAME}" <${process.env.EMAIL_FROM}>`;
  }
  
  async send({ to, subject, template, data, attachments }) {
    const html = renderTemplate(template, {
      ...data,
      year: new Date().getFullYear(),
      appName: process.env.APP_NAME || 'MyApp',
      appUrl: process.env.APP_URL,
      supportEmail: process.env.SUPPORT_EMAIL,
    });
    
    const mailOptions = {
      from: this.from,
      to,
      subject,
      html,
      // Plain text fallback
      text: html.replace(/<[^>]*>/g, '').replace(/\s+/g, ' ').trim(),
    };
    
    if (attachments?.length) {
      mailOptions.attachments = attachments;
    }
    
    const info = await transporter.sendMail(mailOptions);
    
    // ใน development แสดง preview URL
    if (process.env.NODE_ENV !== 'production') {
      const nodemailer = require('nodemailer');
      console.log('Preview URL:', nodemailer.getTestMessageUrl(info));
    }
    
    return info;
  }
  
  // Convenience methods
  async sendVerification(user, token) {
    const verifyUrl = `${process.env.FRONTEND_URL}/verify-email/${token}`;
    
    return this.send({
      to: user.email,
      subject: 'ยืนยัน Email ของคุณ',
      template: 'verify-email',
      data: { name: user.name, verifyUrl },
    });
  }
  
  async sendPasswordReset(user, token, ip) {
    const resetUrl = `${process.env.FRONTEND_URL}/reset-password/${token}`;
    
    return this.send({
      to: user.email,
      subject: 'รีเซ็ตรหัสผ่าน',
      template: 'password-reset',
      data: { name: user.name, email: user.email, resetUrl, requestIp: ip },
    });
  }
  
  async sendWelcome(user) {
    return this.send({
      to: user.email,
      subject: `ยินดีต้อนรับสู่ ${process.env.APP_NAME}!`,
      template: 'welcome',
      data: { name: user.name },
    });
  }
  
  async sendOrderConfirmation(user, order) {
    return this.send({
      to: user.email,
      subject: `ยืนยันคำสั่งซื้อ #${order.orderNumber}`,
      template: 'order-confirmation',
      data: { name: user.name, order },
    });
  }
}

module.exports = new EmailService();
```

---

## Practical: Email Verification, Password Reset

### ใช้ Queue ใน Controller

```javascript
// controllers/authController.js
const { emailQueueHelpers } = require('../queues/emailQueue');

exports.register = async (req, res) => {
  // ... สร้าง user ...
  
  const token = user.generateEmailVerificationToken();
  await user.save({ validateBeforeSave: false });
  
  // เพิ่มเข้า queue แทนการส่งทันที
  await emailQueueHelpers.add(
    user.email,
    'ยืนยัน Email ของคุณ',
    'verify-email',
    { name: user.name, verifyUrl: `${process.env.FRONTEND_URL}/verify-email/${token}` },
    { priority: 1 }  // ความสำคัญสูง
  );
  
  res.status(201).json({
    success: true,
    message: 'สมัครสมาชิกเรียบร้อย กรุณาตรวจสอบ email เพื่อยืนยัน',
  });
};

// Verify email
exports.verifyEmail = async (req, res) => {
  const hashedToken = crypto
    .createHash('sha256')
    .update(req.params.token)
    .digest('hex');
  
  const user = await User.findOne({
    emailVerificationToken: hashedToken,
    emailVerificationExpires: { $gt: Date.now() },
  });
  
  if (!user) {
    return res.status(400).json({
      success: false,
      message: 'Token ไม่ถูกต้องหรือหมดอายุ',
    });
  }
  
  user.isEmailVerified = true;
  user.emailVerificationToken = undefined;
  user.emailVerificationExpires = undefined;
  await user.save({ validateBeforeSave: false });
  
  // ส่ง welcome email
  await emailQueueHelpers.add(
    user.email,
    `ยินดีต้อนรับสู่ ${process.env.APP_NAME}!`,
    'welcome',
    { name: user.name }
  );
  
  res.json({
    success: true,
    message: 'ยืนยัน email เรียบร้อยแล้ว',
  });
};
```

---

## แบบฝึกหัด

### Exercise 1: Email Newsletter

1. สร้าง newsletter subscription system
2. Template สำหรับ newsletter
3. Unsubscribe link ใน footer ทุก email
4. Batch sending สำหรับ subscribers หลายพัน

### Exercise 2: Transactional Emails

สร้าง templates และ functions สำหรับ:
1. Order confirmation
2. Shipping notification
3. Delivery confirmation
4. Return/refund confirmation

### Exercise 3: Email Analytics

1. Track email opens (tracking pixel)
2. Track link clicks
3. Handle bounces และ unsubscribes จาก webhook
4. Dashboard แสดง open rate, click rate

---

## สรุป

- **Nodemailer** สำหรับส่ง email ผ่าน SMTP
- **Handlebars** templates สำหรับ HTML emails
- **SendGrid** API สำหรับ transactional email ใน production
- **Bull Queue** ส่ง email แบบ asynchronous
- **Retry logic** สำหรับ email ที่ล้มเหลว

> **บทถัดไป:** Part 28 — Input Validation
