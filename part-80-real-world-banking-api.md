# Part 80: Real-World Banking API
## ขั้นตอนที่ 791-800 จาก 1000

---

## Banking API Design

สร้าง banking system backend ที่ครอบคลุม:
- Account management
- Transactions
- Double-entry accounting
- Fraud detection
- Compliance

---

## 1. Database Schema

```javascript
// models/Account.js
const mongoose = require('mongoose');

const accountSchema = new mongoose.Schema({
  accountNumber: { 
    type: String, 
    unique: true, 
    required: true,
    index: true
  },
  userId: { type: mongoose.Schema.Types.ObjectId, ref: 'User', required: true },
  
  type: {
    type: String,
    enum: ['checking', 'savings', 'credit', 'loan'],
    required: true
  },
  
  currency: { type: String, default: 'THB', required: true },
  
  balance: {
    available: { type: Number, default: 0 },  // สำหรับ withdraw
    current: { type: Number, default: 0 },    // รวม pending
    pending: { type: Number, default: 0 }     // รอ settle
  },
  
  limits: {
    dailyTransfer: { type: Number, default: 100000 },
    singleTransaction: { type: Number, default: 50000 },
    overdraft: { type: Number, default: 0 }
  },
  
  status: {
    type: String,
    enum: ['active', 'frozen', 'closed', 'suspended'],
    default: 'active'
  },
  
  metadata: {
    openedAt: { type: Date, default: Date.now },
    closedAt: Date,
    lastActivityAt: Date,
    interestRate: { type: Number, default: 0 }
  }
}, { timestamps: true });

accountSchema.index({ userId: 1 });
accountSchema.index({ accountNumber: 1 }, { unique: true });
accountSchema.index({ 'metadata.lastActivityAt': -1 });

// สร้าง account number อัตโนมัติ
accountSchema.pre('save', function(next) {
  if (!this.accountNumber) {
    this.accountNumber = generateAccountNumber();
  }
  next();
});

function generateAccountNumber() {
  const prefix = '500'; // Bank code
  const timestamp = Date.now().toString().slice(-7);
  const random = Math.floor(Math.random() * 1000).toString().padStart(3, '0');
  const base = `${prefix}${timestamp}${random}`;
  const checkDigit = calculateLuhn(base);
  return `${base}${checkDigit}`;
}

function calculateLuhn(number) {
  let sum = 0;
  let alternate = false;
  
  for (let i = number.length - 1; i >= 0; i--) {
    let n = parseInt(number.charAt(i));
    if (alternate) {
      n *= 2;
      if (n > 9) n -= 9;
    }
    sum += n;
    alternate = !alternate;
  }
  
  return (10 - (sum % 10)) % 10;
}

module.exports = mongoose.model('Account', accountSchema);
```

```javascript
// models/Transaction.js - Double-entry accounting
const transactionSchema = new mongoose.Schema({
  transactionId: { type: String, unique: true, required: true },
  referenceId: String,  // external reference
  
  type: {
    type: String,
    enum: ['transfer', 'deposit', 'withdrawal', 'payment', 'refund', 'fee', 'interest', 'adjustment'],
    required: true
  },
  
  // Double-entry: debit account และ credit account
  entries: [{
    accountId: { type: mongoose.Schema.Types.ObjectId, ref: 'Account', required: true },
    accountNumber: String,
    type: { type: String, enum: ['debit', 'credit'], required: true },
    amount: { type: Number, required: true, min: 0 },
    currency: String,
    balanceBefore: Number,
    balanceAfter: Number
  }],
  
  amount: { type: Number, required: true, min: 0 },
  currency: { type: String, default: 'THB' },
  
  description: String,
  category: String,
  
  status: {
    type: String,
    enum: ['pending', 'processing', 'completed', 'failed', 'reversed'],
    default: 'pending'
  },
  
  metadata: {
    initiatedBy: mongoose.Schema.Types.ObjectId,
    channel: { type: String, enum: ['web', 'mobile', 'api', 'atm', 'branch'] },
    ipAddress: String,
    deviceId: String,
    location: {
      lat: Number,
      lng: Number,
      city: String,
      country: String
    }
  },
  
  fraud: {
    score: { type: Number, min: 0, max: 100 },
    flags: [String],
    reviewed: { type: Boolean, default: false },
    reviewedBy: mongoose.Schema.Types.ObjectId,
    reviewedAt: Date
  },
  
  failureReason: String,
  reversalOf: mongoose.Schema.Types.ObjectId,
  
  settledAt: Date,
  valueDate: Date
}, { timestamps: true });

transactionSchema.index({ transactionId: 1 }, { unique: true });
transactionSchema.index({ 'entries.accountId': 1, createdAt: -1 });
transactionSchema.index({ status: 1, createdAt: -1 });
transactionSchema.index({ 'fraud.score': -1, status: 1 });

module.exports = mongoose.model('Transaction', transactionSchema);
```

---

## 2. Account Management

```javascript
// services/account.service.js
const Account = require('../models/Account');
const Transaction = require('../models/Transaction');
const { v4: uuidv4 } = require('uuid');

class AccountService {
  async createAccount(userId, type, currency = 'THB') {
    const account = await Account.create({
      userId,
      type,
      currency,
      balance: { available: 0, current: 0, pending: 0 }
    });
    
    return account;
  }

  async getBalance(accountId) {
    const account = await Account.findById(accountId);
    if (!account) throw new Error('Account not found');
    
    return {
      available: account.balance.available,
      current: account.balance.current,
      pending: account.balance.pending,
      currency: account.currency
    };
  }

  async getStatement(accountId, options = {}) {
    const { startDate, endDate, page = 1, limit = 50 } = options;
    
    const filter = {
      'entries.accountId': accountId,
      status: 'completed'
    };
    
    if (startDate || endDate) {
      filter.createdAt = {};
      if (startDate) filter.createdAt.$gte = new Date(startDate);
      if (endDate) filter.createdAt.$lte = new Date(endDate);
    }

    const [transactions, total, stats] = await Promise.all([
      Transaction.find(filter)
        .sort('-createdAt')
        .skip((page - 1) * limit)
        .limit(limit),
      Transaction.countDocuments(filter),
      Transaction.aggregate([
        { $match: filter },
        { $unwind: '$entries' },
        { $match: { 'entries.accountId': mongoose.Types.ObjectId(accountId) } },
        { $group: {
          _id: '$entries.type',
          total: { $sum: '$entries.amount' }
        }}
      ])
    ]);

    const statsMap = {};
    stats.forEach(s => { statsMap[s._id] = s.total; });

    return {
      transactions,
      total,
      page,
      limit,
      summary: {
        totalCredits: statsMap.credit || 0,
        totalDebits: statsMap.debit || 0,
        netChange: (statsMap.credit || 0) - (statsMap.debit || 0)
      }
    };
  }
}

module.exports = new AccountService();
```

---

## 3. Double-Entry Transactions

```javascript
// services/transaction.service.js

class TransactionService {
  async transfer({ fromAccountId, toAccountId, amount, description, metadata }) {
    if (amount <= 0) throw new Error('Amount must be positive');

    const [fromAccount, toAccount] = await Promise.all([
      Account.findById(fromAccountId),
      Account.findById(toAccountId)
    ]);

    if (!fromAccount) throw new Error('Source account not found');
    if (!toAccount) throw new Error('Destination account not found');

    if (fromAccount.status !== 'active') throw new Error('Source account is not active');
    if (toAccount.status !== 'active') throw new Error('Destination account is not active');

    // ตรวจสอบ balance เพียงพอ
    if (fromAccount.balance.available < amount) {
      throw new Error(`Insufficient funds. Available: ${fromAccount.balance.available}`);
    }

    // ตรวจสอบ limits
    await this.checkDailyLimit(fromAccountId, amount);
    if (amount > fromAccount.limits.singleTransaction) {
      throw new Error(`Amount exceeds single transaction limit: ${fromAccount.limits.singleTransaction}`);
    }

    // สร้าง transaction ด้วย MongoDB transaction
    const session = await mongoose.startSession();
    session.startTransaction();
    
    try {
      const transactionId = uuidv4();
      
      // Double-entry: debit จาก source, credit ไปยัง destination
      const transaction = await Transaction.create([{
        transactionId,
        type: 'transfer',
        amount,
        currency: fromAccount.currency,
        description,
        entries: [
          {
            accountId: fromAccountId,
            accountNumber: fromAccount.accountNumber,
            type: 'debit',
            amount,
            currency: fromAccount.currency,
            balanceBefore: fromAccount.balance.current,
            balanceAfter: fromAccount.balance.current - amount
          },
          {
            accountId: toAccountId,
            accountNumber: toAccount.accountNumber,
            type: 'credit',
            amount,
            currency: toAccount.currency,
            balanceBefore: toAccount.balance.current,
            balanceAfter: toAccount.balance.current + amount
          }
        ],
        status: 'processing',
        metadata: {
          ...metadata,
          initiatedBy: metadata?.userId
        }
      }], { session });

      // อัพเดต balances
      await Promise.all([
        Account.findByIdAndUpdate(fromAccountId, {
          $inc: {
            'balance.available': -amount,
            'balance.current': -amount
          },
          'metadata.lastActivityAt': new Date()
        }, { session }),
        
        Account.findByIdAndUpdate(toAccountId, {
          $inc: {
            'balance.available': amount,
            'balance.current': amount
          },
          'metadata.lastActivityAt': new Date()
        }, { session })
      ]);

      // อัพเดต status เป็น completed
      await Transaction.findByIdAndUpdate(
        transaction[0]._id,
        { status: 'completed', settledAt: new Date() },
        { session }
      );

      await session.commitTransaction();
      
      // Async: ส่ง notifications
      await notificationService.sendTransactionAlert(fromAccountId, transaction[0]).catch(console.error);
      await notificationService.sendTransactionAlert(toAccountId, transaction[0]).catch(console.error);
      
      return transaction[0];
      
    } catch (error) {
      await session.abortTransaction();
      throw error;
    } finally {
      session.endSession();
    }
  }

  async checkDailyLimit(accountId, amount) {
    const account = await Account.findById(accountId);
    const today = new Date();
    today.setHours(0, 0, 0, 0);

    const dailyTotal = await Transaction.aggregate([
      {
        $match: {
          'entries.accountId': mongoose.Types.ObjectId(accountId),
          'entries.type': 'debit',
          status: { $in: ['completed', 'processing'] },
          createdAt: { $gte: today }
        }
      },
      { $unwind: '$entries' },
      {
        $match: {
          'entries.accountId': mongoose.Types.ObjectId(accountId),
          'entries.type': 'debit'
        }
      },
      { $group: { _id: null, total: { $sum: '$entries.amount' } } }
    ]);

    const totalToday = dailyTotal[0]?.total || 0;
    
    if (totalToday + amount > account.limits.dailyTransfer) {
      throw new Error(
        `Daily transfer limit exceeded. Limit: ${account.limits.dailyTransfer}, Used: ${totalToday}`
      );
    }
  }

  async reverseTransaction(transactionId, reason) {
    const original = await Transaction.findOne({ transactionId });
    
    if (!original) throw new Error('Transaction not found');
    if (original.status !== 'completed') throw new Error('Only completed transactions can be reversed');

    const session = await mongoose.startSession();
    session.startTransaction();
    
    try {
      // สร้าง reversal transaction (สลับ debit/credit)
      const reversalEntries = original.entries.map(entry => ({
        ...entry,
        type: entry.type === 'debit' ? 'credit' : 'debit'
      }));

      const reversal = await Transaction.create([{
        transactionId: uuidv4(),
        type: 'refund',
        amount: original.amount,
        currency: original.currency,
        description: `Reversal of ${original.transactionId}: ${reason}`,
        entries: reversalEntries,
        status: 'completed',
        reversalOf: original._id,
        settledAt: new Date()
      }], { session });

      // อัพเดต balances (สลับทิศทาง)
      for (const entry of original.entries) {
        const increment = entry.type === 'debit' ? entry.amount : -entry.amount;
        await Account.findByIdAndUpdate(entry.accountId, {
          $inc: {
            'balance.available': increment,
            'balance.current': increment
          }
        }, { session });
      }

      // Mark original as reversed
      await Transaction.findByIdAndUpdate(
        original._id,
        { status: 'reversed' },
        { session }
      );

      await session.commitTransaction();
      return reversal[0];
      
    } catch (error) {
      await session.abortTransaction();
      throw error;
    } finally {
      session.endSession();
    }
  }
}

module.exports = new TransactionService();
```

---

## 4. Fraud Detection

```javascript
// services/fraud-detection.service.js
const redis = require('../redis');

class FraudDetectionService {
  async analyzeTransaction(transaction, account) {
    const checks = await Promise.all([
      this.checkAmountAnomaly(transaction, account),
      this.checkVelocity(account._id, transaction.amount),
      this.checkGeolocationAnomaly(account._id, transaction.metadata?.location),
      this.checkUnusualTime(transaction),
      this.checkBlacklist(transaction.metadata?.ipAddress)
    ]);

    const flags = checks.filter(c => c.flag).map(c => c.flag);
    const score = this.calculateScore(checks);

    return { score, flags };
  }

  async checkAmountAnomaly(transaction, account) {
    // ตรวจสอบว่า amount ผิดปกติจาก pattern ปกติ
    const avgAmount = await this.getAverageTransactionAmount(account._id);
    const ratio = transaction.amount / (avgAmount || 1);
    
    if (ratio > 10) {
      return { flag: 'UNUSUAL_AMOUNT', score: 40 };
    }
    if (ratio > 5) {
      return { flag: 'HIGH_AMOUNT', score: 20 };
    }
    return { score: 0 };
  }

  async checkVelocity(accountId, amount) {
    // ตรวจสอบ transactions ใน 1 ชั่วโมงที่ผ่านมา
    const key = `velocity:${accountId}`;
    const count = await redis.incr(key);
    
    if (count === 1) {
      await redis.expire(key, 3600);
    }
    
    if (count > 20) {
      return { flag: 'HIGH_VELOCITY', score: 60 };
    }
    if (count > 10) {
      return { flag: 'ELEVATED_VELOCITY', score: 30 };
    }
    return { score: 0 };
  }

  async checkGeolocationAnomaly(accountId, currentLocation) {
    if (!currentLocation) return { score: 0 };
    
    const lastLocationKey = `last_location:${accountId}`;
    const lastLocation = await redis.get(lastLocationKey);
    
    if (lastLocation) {
      const last = JSON.parse(lastLocation);
      const distance = this.calculateDistance(last, currentLocation);
      
      // ถ้าระยะทางเกิน 1000km ใน 1 ชั่วโมง → suspicious
      if (distance > 1000) {
        return { flag: 'IMPOSSIBLE_TRAVEL', score: 80 };
      }
    }
    
    await redis.set(lastLocationKey, JSON.stringify(currentLocation), 'EX', 3600);
    return { score: 0 };
  }

  async checkUnusualTime(transaction) {
    const hour = new Date().getHours();
    
    // 2am - 6am เป็นช่วงเสี่ยง
    if (hour >= 2 && hour <= 6) {
      return { flag: 'UNUSUAL_HOUR', score: 20 };
    }
    return { score: 0 };
  }

  async checkBlacklist(ipAddress) {
    if (!ipAddress) return { score: 0 };
    
    const isBlacklisted = await redis.sismember('blacklist:ips', ipAddress);
    
    if (isBlacklisted) {
      return { flag: 'BLACKLISTED_IP', score: 100 };
    }
    return { score: 0 };
  }

  calculateScore(checks) {
    const totalScore = checks.reduce((sum, c) => sum + (c.score || 0), 0);
    return Math.min(100, totalScore);
  }

  async getAverageTransactionAmount(accountId) {
    const cacheKey = `avg_amount:${accountId}`;
    const cached = await redis.get(cacheKey);
    
    if (cached) return parseFloat(cached);
    
    const result = await Transaction.aggregate([
      {
        $match: {
          'entries.accountId': mongoose.Types.ObjectId(accountId),
          status: 'completed',
          createdAt: { $gte: new Date(Date.now() - 30 * 24 * 60 * 60 * 1000) }
        }
      },
      { $group: { _id: null, avg: { $avg: '$amount' } } }
    ]);

    const avg = result[0]?.avg || 0;
    await redis.set(cacheKey, avg, 'EX', 3600);
    
    return avg;
  }

  calculateDistance(loc1, loc2) {
    const R = 6371; // Earth's radius in km
    const lat1Rad = (loc1.lat * Math.PI) / 180;
    const lat2Rad = (loc2.lat * Math.PI) / 180;
    const deltaLat = ((loc2.lat - loc1.lat) * Math.PI) / 180;
    const deltaLng = ((loc2.lng - loc1.lng) * Math.PI) / 180;

    const a = Math.sin(deltaLat / 2) ** 2 +
      Math.cos(lat1Rad) * Math.cos(lat2Rad) * Math.sin(deltaLng / 2) ** 2;
    
    return R * 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1 - a));
  }
}

module.exports = new FraudDetectionService();
```

---

## 5. Compliance

```javascript
// services/compliance.service.js

class ComplianceService {
  // KYC Check
  async verifyKYC(userId, documents) {
    // ตรวจสอบ National ID
    // ตรวจสอบ Bank Statement
    // Face matching
    
    return {
      status: 'verified',
      level: 'full',
      verifiedAt: new Date()
    };
  }

  // AML - Anti-Money Laundering
  async checkAML(transaction) {
    const threshold = 1000000; // 1 ล้านบาท ต้องรายงาน
    
    if (transaction.amount >= threshold) {
      await this.reportLargeTransaction(transaction);
    }

    // ตรวจสอบ Suspicious Activity
    const suspicious = await this.detectSuspiciousPattern(transaction);
    if (suspicious) {
      await this.reportSuspiciousActivity(transaction);
    }
  }

  async reportLargeTransaction(transaction) {
    // ส่งรายงานไปยัง BOT (Bank of Thailand)
    await AuditLog.create({
      type: 'LARGE_TRANSACTION',
      transactionId: transaction.transactionId,
      amount: transaction.amount,
      reportedAt: new Date()
    });
  }

  async detectSuspiciousPattern(transaction) {
    // Structuring detection (หลาย transactions เล็กๆ เพื่อเลี่ยง threshold)
    const oneHourAgo = new Date(Date.now() - 3600000);
    
    const recentTx = await Transaction.aggregate([
      {
        $match: {
          'entries.accountId': transaction.entries[0].accountId,
          status: 'completed',
          createdAt: { $gte: oneHourAgo },
          _id: { $ne: transaction._id }
        }
      },
      { $group: { _id: null, total: { $sum: '$amount' }, count: { $sum: 1 } } }
    ]);

    const { total = 0, count = 0 } = recentTx[0] || {};
    
    // ถ้า sum ของ transactions ใน 1 ชั่วโมงเกิน threshold แต่แต่ละอันน้อยกว่า
    return total + transaction.amount >= 1000000 && count > 5;
  }
}

module.exports = new ComplianceService();
```

---

## 6. API Endpoints

```javascript
// routes/banking.js
const router = require('express').Router();
const authenticate = require('../middleware/authenticate');
const { rateLimit } = require('express-rate-limit');

const transferLimiter = rateLimit({
  windowMs: 60 * 1000,
  max: 10,
  message: { error: 'Too many transfer requests' }
});

// Accounts
router.post('/accounts', authenticate, accountController.create);
router.get('/accounts', authenticate, accountController.list);
router.get('/accounts/:id', authenticate, accountController.getById);
router.get('/accounts/:id/balance', authenticate, accountController.getBalance);
router.get('/accounts/:id/statement', authenticate, accountController.getStatement);

// Transactions
router.post('/transfer', authenticate, transferLimiter, transactionController.transfer);
router.post('/deposit', authenticate, transactionController.deposit);
router.post('/withdraw', authenticate, transactionController.withdraw);
router.get('/transactions/:id', authenticate, transactionController.getById);

// Admin
router.post('/transactions/:id/reverse', authenticate, authorize('admin'), transactionController.reverse);
router.get('/fraud/suspicious', authenticate, authorize('compliance'), fraudController.getSuspicious);

module.exports = router;
```

---

## แบบฝึกหัด

สร้าง Banking API ที่มี:
1. Account creation + management
2. Transfers ด้วย double-entry accounting
3. Transaction history
4. Daily limits
5. Basic fraud detection
6. Audit logging

---

## สรุป

Banking API ต้องการความระมัดระวังสูงสุดในเรื่อง data consistency (ใช้ transactions), security (encryption, audit logs), และ compliance (KYC/AML) Double-entry accounting เป็น foundation ที่สำคัญที่ทำให้ balances ถูกต้องเสมอ Fraud detection เป็น layered approach ที่ต้องพัฒนาต่อเนื่องตาม patterns ใหม่ๆ

---

## จบหลักสูตร Node.js/Express.js ขั้นสูง

ยินดีด้วย! คุณได้เรียนรู้ตั้งแต่พื้นฐานจนถึง production-grade Node.js development ครอบคลุม:

- Server-Sent Events, OAuth2, Multi-tenancy
- CQRS, DDD, Clean Architecture
- Monitoring, Distributed Tracing, Load Balancing
- Kubernetes, Serverless
- Real-world projects (E-commerce, Social Media, Banking)
- Security, Testing, TypeScript, Prisma, Kafka

> ขั้นตอนที่ 791-800 เสร็จสมบูรณ์ • เหลือ 200 ขั้นตอนจาก 1000
