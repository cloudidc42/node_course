# Part 99 | ขั้นตอนที่ 1781-1800 จาก 1000+

## Career Path สู่ Staff/Principal Engineer

ในส่วนนี้เราจะเรียนรู้เส้นทางอาชีพ, ทักษะที่ต้องพัฒนา, และการเตรียมตัว interview สำหรับ Senior/Staff Engineer

---

## ขั้นตอนที่ 1781: Engineering Career Ladder

```
Junior Engineer → Mid-level → Senior → Staff → Principal → Distinguished/Fellow
```

**Staff Engineer vs Principal Engineer:**
- **Staff Engineer**: ผู้เชี่ยวชาญระดับ team/org ช่วย align technical direction
- **Principal Engineer**: ผู้เชี่ยวชาญระดับ company-wide ตัดสินใจ architecture หลัก

```javascript
// career-tracker.js
// Track career progress and skills

class CareerTracker {
  constructor() {
    this.levels = ['junior', 'mid', 'senior', 'staff', 'principal'];
    
    this.competencies = {
      technical: {
        'coding': { weight: 0.3, description: 'Code quality, patterns, best practices' },
        'system-design': { weight: 0.3, description: 'Scalable system architecture' },
        'debugging': { weight: 0.2, description: 'Problem diagnosis and resolution' },
        'testing': { weight: 0.1, description: 'Testing strategy and execution' },
        'security': { weight: 0.1, description: 'Security awareness and practices' }
      },
      leadership: {
        'mentoring': { weight: 0.25, description: 'Helping others grow' },
        'communication': { weight: 0.25, description: 'Clear technical communication' },
        'project-management': { weight: 0.25, description: 'Delivering projects on time' },
        'influence': { weight: 0.25, description: 'Technical influence without authority' }
      },
      impact: {
        'team-impact': { weight: 0.33, description: 'Impact on team productivity' },
        'org-impact': { weight: 0.33, description: 'Impact across organization' },
        'business-impact': { weight: 0.34, description: 'Impact on business outcomes' }
      }
    };
  }

  assessLevel(scores) {
    // scores: { technical: 0-5, leadership: 0-5, impact: 0-5 }
    const avg = (scores.technical + scores.leadership + scores.impact) / 3;
    
    if (avg < 1.5) return { level: 'junior', next: 'mid', gap: 1.5 - avg };
    if (avg < 2.5) return { level: 'mid', next: 'senior', gap: 2.5 - avg };
    if (avg < 3.5) return { level: 'senior', next: 'staff', gap: 3.5 - avg };
    if (avg < 4.5) return { level: 'staff', next: 'principal', gap: 4.5 - avg };
    return { level: 'principal', next: 'distinguished', gap: 0 };
  }

  generateGrowthPlan(currentLevel, targetLevel) {
    const plans = {
      'mid-to-senior': {
        focus: 'Technical depth and independent ownership',
        milestones: [
          'Lead a significant feature end-to-end',
          'Mentor a junior engineer successfully',
          'Drive technical design for a medium system',
          'Reduce on-call incidents through improved observability',
          'Complete system design interview prep'
        ],
        timeline: '12-18 months',
        keySkills: ['System Design', 'Technical Writing', 'Code Review Excellence']
      },
      'senior-to-staff': {
        focus: 'Cross-team influence and org-level impact',
        milestones: [
          'Define and drive adoption of a technical standard',
          'Influence architecture across multiple teams',
          'Lead an initiative with 3+ engineers',
          'Write a technical strategy document',
          'Present at engineering all-hands'
        ],
        timeline: '18-36 months',
        keySkills: ['Technical Strategy', 'Stakeholder Management', 'Large-scale System Design']
      }
    };
    
    return plans[`${currentLevel}-to-${targetLevel}`] || null;
  }
}

// System Design Interview Patterns
class SystemDesignPrep {
  getFramework() {
    return {
      steps: [
        {
          step: 1,
          name: 'Clarify Requirements (5 min)',
          questions: [
            'How many users? (DAU/MAU)',
            'Read-heavy or write-heavy?',
            'What are the core features?',
            'Consistency or availability priority?',
            'Any latency SLAs?'
          ]
        },
        {
          step: 2,
          name: 'Capacity Estimation (5 min)',
          items: [
            'QPS (queries per second)',
            'Storage requirements',
            'Bandwidth requirements',
            'Memory requirements'
          ]
        },
        {
          step: 3,
          name: 'High-Level Design (10 min)',
          components: [
            'Client (web/mobile)',
            'Load balancer',
            'Application servers',
            'Database',
            'Cache',
            'CDN',
            'Message queue'
          ]
        },
        {
          step: 4,
          name: 'Deep Dive (20 min)',
          topics: [
            'Database schema and sharding strategy',
            'Caching strategy',
            'API design',
            'Handling failures',
            'Monitoring approach'
          ]
        },
        {
          step: 5,
          name: 'Wrap Up (5 min)',
          items: [
            'Identify bottlenecks',
            'Scaling plan',
            'Trade-offs made',
            'What would you do differently'
          ]
        }
      ]
    };
  }

  getCommonProblems() {
    return [
      {
        name: 'Design Twitter/Instagram Feed',
        keyPoints: [
          'Fan-out on write vs fan-out on read',
          'Timeline denormalization',
          'Celebrities vs regular users',
          'Cache warm-up strategy'
        ]
      },
      {
        name: 'Design URL Shortener',
        keyPoints: [
          'Hash collision handling',
          'Custom aliases',
          'Analytics tracking',
          '302 vs 301 redirect'
        ]
      },
      {
        name: 'Design Rate Limiter',
        keyPoints: [
          'Token bucket vs sliding window',
          'Distributed rate limiting with Redis',
          'Per-user vs per-IP vs per-API-key',
          'Handling race conditions'
        ]
      },
      {
        name: 'Design Notification System',
        keyPoints: [
          'Multiple channels (push, email, SMS)',
          'Priority queues',
          'User preferences',
          'Delivery guarantees'
        ]
      }
    ];
  }
}

module.exports = { CareerTracker, SystemDesignPrep };
```

---

## ขั้นตอนที่ 1782: System Design Practice

```javascript
// system-design-example.js
// Design a URL Shortener - full implementation

const express = require('express');
const Redis = require('ioredis');
const { Pool } = require('pg');

class URLShortener {
  constructor() {
    this.app = express();
    this.redis = new Redis({ host: 'localhost', port: 6379 });
    this.db = new Pool({ connectionString: process.env.DATABASE_URL });
    this.base62Chars = '0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz';
    this.setupRoutes();
  }

  async initialize() {
    await this.db.query(`
      CREATE TABLE IF NOT EXISTS urls (
        id BIGSERIAL PRIMARY KEY,
        short_code VARCHAR(10) UNIQUE NOT NULL,
        original_url TEXT NOT NULL,
        user_id TEXT,
        click_count BIGINT DEFAULT 0,
        expires_at TIMESTAMPTZ,
        created_at TIMESTAMPTZ DEFAULT NOW()
      );
      CREATE INDEX IF NOT EXISTS idx_short_code ON urls(short_code);
    `);
  }

  // Encode number to base62
  encode(num) {
    if (num === 0) return this.base62Chars[0];
    let result = '';
    while (num > 0) {
      result = this.base62Chars[num % 62] + result;
      num = Math.floor(num / 62);
    }
    return result;
  }

  async shortenURL(originalUrl, options = {}) {
    // Input validation
    try { new URL(originalUrl); } catch {
      throw new Error('Invalid URL');
    }

    // Check if already shortened
    const existing = await this.db.query(
      'SELECT short_code FROM urls WHERE original_url = $1 LIMIT 1',
      [originalUrl]
    );
    if (existing.rows[0]) return existing.rows[0].short_code;

    // Use custom alias or generate from ID
    let shortCode = options.customAlias;
    
    if (!shortCode) {
      const { rows } = await this.db.query(
        `INSERT INTO urls (short_code, original_url, user_id, expires_at) 
         VALUES ('TEMP', $1, $2, $3) RETURNING id`,
        [originalUrl, options.userId, options.expiresAt]
      );
      shortCode = this.encode(rows[0].id);
      await this.db.query(
        'UPDATE urls SET short_code = $1 WHERE id = $2',
        [shortCode, rows[0].id]
      );
    } else {
      await this.db.query(
        `INSERT INTO urls (short_code, original_url, user_id, expires_at) 
         VALUES ($1, $2, $3, $4)`,
        [shortCode, originalUrl, options.userId, options.expiresAt]
      );
    }

    // Cache for fast redirects
    await this.redis.setex(`url:${shortCode}`, 3600, originalUrl);
    
    return shortCode;
  }

  async resolveURL(shortCode) {
    // Check cache first
    const cached = await this.redis.get(`url:${shortCode}`);
    if (cached) {
      // Async increment click count
      this.incrementClickCount(shortCode).catch(console.error);
      return cached;
    }

    // Fallback to database
    const { rows } = await this.db.query(
      `SELECT original_url, expires_at FROM urls 
       WHERE short_code = $1 AND (expires_at IS NULL OR expires_at > NOW())`,
      [shortCode]
    );

    if (!rows[0]) return null;

    const url = rows[0].original_url;
    await this.redis.setex(`url:${shortCode}`, 3600, url);
    this.incrementClickCount(shortCode).catch(console.error);
    
    return url;
  }

  async incrementClickCount(shortCode) {
    await this.db.query(
      'UPDATE urls SET click_count = click_count + 1 WHERE short_code = $1',
      [shortCode]
    );
  }

  setupRoutes() {
    this.app.use(express.json());

    // Shorten URL
    this.app.post('/shorten', async (req, res) => {
      const { url, customAlias, expiresAt } = req.body;
      
      if (!url) return res.status(400).json({ error: 'URL is required' });
      
      try {
        const shortCode = await this.shortenURL(url, { customAlias, expiresAt });
        res.json({
          shortUrl: `${process.env.BASE_URL}/${shortCode}`,
          shortCode
        });
      } catch (err) {
        res.status(400).json({ error: err.message });
      }
    });

    // Redirect
    this.app.get('/:shortCode', async (req, res) => {
      const { shortCode } = req.params;
      
      try {
        const originalUrl = await this.resolveURL(shortCode);
        
        if (!originalUrl) {
          return res.status(404).json({ error: 'URL not found or expired' });
        }
        
        // 302 temporary redirect (allows analytics tracking)
        res.redirect(302, originalUrl);
      } catch (err) {
        res.status(500).json({ error: 'Internal server error' });
      }
    });
  }
}

module.exports = { URLShortener };
```

---

## ขั้นตอนที่ 1783-1800: Interview Preparation

```javascript
// coding-patterns.js
// Common coding interview patterns for Node.js engineers

// Pattern 1: Two Pointers
function twoSum(nums, target) {
  const map = new Map();
  for (let i = 0; i < nums.length; i++) {
    const complement = target - nums[i];
    if (map.has(complement)) return [map.get(complement), i];
    map.set(nums[i], i);
  }
  return [];
}

// Pattern 2: Sliding Window
function maxSubarraySum(nums, k) {
  let sum = nums.slice(0, k).reduce((a, b) => a + b, 0);
  let maxSum = sum;
  
  for (let i = k; i < nums.length; i++) {
    sum += nums[i] - nums[i - k];
    maxSum = Math.max(maxSum, sum);
  }
  
  return maxSum;
}

// Pattern 3: Binary Search
function binarySearch(arr, target) {
  let left = 0, right = arr.length - 1;
  
  while (left <= right) {
    const mid = left + Math.floor((right - left) / 2);
    if (arr[mid] === target) return mid;
    if (arr[mid] < target) left = mid + 1;
    else right = mid - 1;
  }
  
  return -1;
}

// Pattern 4: BFS for shortest path
function shortestPath(graph, start, end) {
  const queue = [[start, [start]]];
  const visited = new Set([start]);
  
  while (queue.length > 0) {
    const [node, path] = queue.shift();
    
    if (node === end) return path;
    
    for (const neighbor of graph[node] || []) {
      if (!visited.has(neighbor)) {
        visited.add(neighbor);
        queue.push([neighbor, [...path, neighbor]]);
      }
    }
  }
  
  return null;
}

// Pattern 5: Dynamic Programming - LCS
function longestCommonSubsequence(s1, s2) {
  const m = s1.length, n = s2.length;
  const dp = Array.from({ length: m + 1 }, () => new Array(n + 1).fill(0));
  
  for (let i = 1; i <= m; i++) {
    for (let j = 1; j <= n; j++) {
      if (s1[i - 1] === s2[j - 1]) {
        dp[i][j] = dp[i - 1][j - 1] + 1;
      } else {
        dp[i][j] = Math.max(dp[i - 1][j], dp[i][j - 1]);
      }
    }
  }
  
  return dp[m][n];
}

// Behavioral Interview Framework - STAR
const starExamples = {
  leadership: {
    situation: 'Our team had conflicting opinions on microservices vs monolith',
    task: 'As tech lead, I needed to drive consensus and make a decision',
    action: 'I organized an RFC process, invited everyone to contribute, analyzed trade-offs with data',
    result: 'We chose microservices for 3 specific domains, kept monolith for others. Team alignment improved 40%'
  },
  conflict: {
    situation: 'PM wanted to ship a feature with known security vulnerability to meet deadline',
    task: 'Needed to push back while maintaining good relationship',
    action: 'Documented risks clearly, proposed a phased approach: ship with feature flag, fix security in parallel',
    result: 'Feature shipped on time with security patch 2 weeks later. No security incidents.'
  },
  failure: {
    situation: 'Led migration that caused 2-hour outage affecting 50k users',
    task: 'Needed to recover and prevent recurrence',
    action: 'Immediately rolled back, wrote blameless postmortem, added canary deployment requirement',
    result: 'No similar incidents since. Process became company standard for migrations'
  }
};

module.exports = { twoSum, maxSubarraySum, binarySearch, shortestPath, longestCommonSubsequence, starExamples };
```

---

## แบบฝึกหัด

### Exercise 1: Mock System Design
ออกแบบ "Distributed Task Queue" พร้อม capacity estimation และ database schema

### Exercise 2: Behavioral Questions
เตรียมตอบ 10 คำถาม behavioral interview ด้วย STAR framework

### Exercise 3: Live Coding Practice
แก้ LeetCode problems: Two Sum, LRU Cache, Meeting Rooms II ด้วย JavaScript

### Career Development Plan
1. สร้าง GitHub portfolio ที่มี 2-3 projects แสดง Node.js expertise
2. เขียน technical blog ใน Medium หรือ Dev.to อย่างน้อยเดือนละครั้ง
3. Contribute ใน OSS projects ที่มีผู้ใช้จริง
4. เข้าร่วม meetups และ conferences

### คำถามทบทวน
1. Staff Engineer ต่างจาก Senior Engineer อย่างไร?
2. วิธีพัฒนา "influence without authority"?
3. System Design Interview มี framework อะไรที่ควรใช้?

---

*ต่อไป: Part 100 - Capstone Project!*
