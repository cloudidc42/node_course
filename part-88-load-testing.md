# Part 88 | ขั้นตอนที่ 1561-1580 จาก 1000+

## Load Testing, Stress Testing, และ Capacity Planning

ในส่วนนี้เราจะเรียนรู้การทดสอบประสิทธิภาพระบบด้วย k6, JMeter, และการวางแผน capacity

---

## ขั้นตอนที่ 1561: Load Testing Fundamentals

```javascript
// k6-basics.js
// k6 load test script (รันด้วย: k6 run k6-basics.js)

import http from 'k6/http';
import { check, sleep, group } from 'k6';
import { Rate, Trend, Counter, Gauge } from 'k6/metrics';

// Custom Metrics
const errorRate = new Rate('errors');
const loginDuration = new Trend('login_duration');
const activeUsers = new Gauge('active_users');
const requestCount = new Counter('request_count');

// Test Configuration
export const options = {
  // Stages: define the load pattern
  stages: [
    { duration: '2m', target: 10 },   // Ramp up to 10 users
    { duration: '5m', target: 10 },   // Stay at 10 users
    { duration: '2m', target: 50 },   // Ramp up to 50 users
    { duration: '5m', target: 50 },   // Stay at 50 users
    { duration: '2m', target: 100 },  // Ramp up to 100 users
    { duration: '5m', target: 100 },  // Stay at 100 users
    { duration: '5m', target: 0 },    // Ramp down to 0
  ],
  
  // Thresholds: define pass/fail criteria
  thresholds: {
    http_req_duration: ['p(95)<500', 'p(99)<1000'], // 95% under 500ms, 99% under 1s
    http_req_failed: ['rate<0.01'],                   // Error rate < 1%
    errors: ['rate<0.05'],                            // Custom error rate < 5%
  },
  
  // HTTP settings
  httpDebug: 'full',
  noConnectionReuse: false,
  discardResponseBodies: false,
};

// Base URL
const BASE_URL = __ENV.BASE_URL || 'http://localhost:3000';

// Helper function
function getHeaders(token = null) {
  const headers = {
    'Content-Type': 'application/json',
    'X-Request-ID': `k6-${Date.now()}`
  };
  
  if (token) {
    headers['Authorization'] = `Bearer ${token}`;
  }
  
  return headers;
}

// Setup: runs once per VU (virtual user)
export function setup() {
  // Create test users
  const testUsers = [];
  
  for (let i = 0; i < 10; i++) {
    const res = http.post(`${BASE_URL}/api/users`, JSON.stringify({
      email: `k6-test-${i}-${Date.now()}@example.com`,
      password: 'Test@12345',
      name: `Test User ${i}`
    }), { headers: { 'Content-Type': 'application/json' } });
    
    if (res.status === 201) {
      testUsers.push({ email: `k6-test-${i}@example.com`, password: 'Test@12345' });
    }
  }
  
  return { testUsers };
}

// Main test function - runs for each VU
export default function(data) {
  const { testUsers } = data;
  const user = testUsers[Math.floor(Math.random() * testUsers.length)];
  
  // Scenario 1: User Login
  let authToken;
  
  group('Authentication', () => {
    const loginStart = Date.now();
    
    const loginRes = http.post(`${BASE_URL}/api/auth/login`, JSON.stringify({
      email: user.email,
      password: user.password
    }), { headers: getHeaders() });
    
    loginDuration.add(Date.now() - loginStart);
    
    const loginSuccess = check(loginRes, {
      'login status is 200': (r) => r.status === 200,
      'login has token': (r) => JSON.parse(r.body).token !== undefined,
      'login response time < 500ms': (r) => r.timings.duration < 500
    });
    
    errorRate.add(!loginSuccess);
    requestCount.add(1);
    
    if (loginSuccess) {
      authToken = JSON.parse(loginRes.body).token;
    }
  });
  
  if (!authToken) return;
  
  sleep(1); // Think time between actions
  
  // Scenario 2: Browse Products
  group('Product Browsing', () => {
    const listRes = http.get(`${BASE_URL}/api/products?page=1&limit=20`, {
      headers: getHeaders(authToken)
    });
    
    check(listRes, {
      'products status is 200': (r) => r.status === 200,
      'products response time < 300ms': (r) => r.timings.duration < 300,
      'products has items': (r) => JSON.parse(r.body).items?.length > 0
    });
    
    requestCount.add(1);
    
    // View specific product
    const products = JSON.parse(listRes.body).items || [];
    if (products.length > 0) {
      const product = products[Math.floor(Math.random() * products.length)];
      
      const detailRes = http.get(`${BASE_URL}/api/products/${product.id}`, {
        headers: getHeaders(authToken)
      });
      
      check(detailRes, {
        'product detail status is 200': (r) => r.status === 200,
        'product detail response time < 200ms': (r) => r.timings.duration < 200
      });
      
      requestCount.add(1);
    }
  });
  
  sleep(2);
  
  // Scenario 3: Add to Cart and Checkout
  group('Checkout Flow', () => {
    // Add to cart
    const addToCartRes = http.post(`${BASE_URL}/api/cart/items`, JSON.stringify({
      productId: 'test-product-1',
      quantity: Math.floor(Math.random() * 3) + 1
    }), { headers: getHeaders(authToken) });
    
    check(addToCartRes, {
      'add to cart status is 200 or 201': (r) => [200, 201].includes(r.status),
    });
    
    requestCount.add(1);
    
    sleep(1);
    
    // Checkout
    const checkoutRes = http.post(`${BASE_URL}/api/orders`, JSON.stringify({
      items: [{ productId: 'test-product-1', quantity: 1 }],
      paymentMethod: 'card',
      shippingAddress: {
        street: '123 Test St',
        city: 'Test City',
        country: 'TH'
      }
    }), { headers: getHeaders(authToken) });
    
    check(checkoutRes, {
      'checkout status is 201': (r) => r.status === 201,
      'checkout response time < 2000ms': (r) => r.timings.duration < 2000
    });
    
    errorRate.add(checkoutRes.status !== 201);
    requestCount.add(1);
  });
  
  activeUsers.add(1);
  sleep(Math.random() * 3 + 1); // Think time: 1-4 seconds
}

// Teardown: cleanup after test
export function teardown(data) {
  console.log('Test completed. Cleaning up...');
  // Delete test data
}
```

---

## ขั้นตอนที่ 1562: Stress Testing

```javascript
// k6-stress-test.js
// Stress test ค้นหา breaking point

import http from 'k6/http';
import { check, sleep } from 'k6';
import { Rate } from 'k6/metrics';

const errorRate = new Rate('errors');
const BASE_URL = __ENV.BASE_URL || 'http://localhost:3000';

export const options = {
  // Stress test pattern: keep increasing until failure
  stages: [
    { duration: '2m', target: 100 },   // Normal load
    { duration: '5m', target: 100 },
    { duration: '2m', target: 200 },   // 2x load
    { duration: '5m', target: 200 },
    { duration: '2m', target: 300 },   // 3x load
    { duration: '5m', target: 300 },
    { duration: '2m', target: 400 },   // 4x load
    { duration: '5m', target: 400 },
    { duration: '10m', target: 0 },    // Recovery
  ],
  
  thresholds: {
    http_req_duration: ['p(99)<5000'], // Allow degraded performance
    http_req_failed: ['rate<0.1'],     // Up to 10% errors allowed under stress
  }
};

export default function() {
  const res = http.get(`${BASE_URL}/api/health`);
  
  const success = check(res, {
    'status is 200': (r) => r.status === 200,
    'response time < 5s': (r) => r.timings.duration < 5000
  });
  
  errorRate.add(!success);
  
  sleep(1);
}
```

```javascript
// k6-spike-test.js
// Spike test: sudden traffic surge

import http from 'k6/http';
import { check, sleep } from 'k6';

export const options = {
  stages: [
    { duration: '10s', target: 10 },   // Baseline
    { duration: '1m', target: 10 },
    { duration: '10s', target: 500 },  // SPIKE! 
    { duration: '3m', target: 500 },   // Stay at spike
    { duration: '10s', target: 10 },   // Back to baseline
    { duration: '3m', target: 10 },    // Recovery period
    { duration: '10s', target: 0 },
  ]
};

const BASE_URL = __ENV.BASE_URL || 'http://localhost:3000';

export default function() {
  const res = http.get(`${BASE_URL}/api/products`);
  
  check(res, {
    'status is 200 or 503': (r) => [200, 503].includes(r.status),
  });
  
  sleep(1);
}
```

---

## ขั้นตอนที่ 1563: Capacity Planning

```javascript
// capacity-planning.js
// คำนวณ capacity ที่ต้องการ

class CapacityPlanner {
  constructor(metrics) {
    this.metrics = metrics;
  }

  // Calculate required instances based on traffic
  calculateInstances(params) {
    const {
      peakRPS,              // Requests per second at peak
      avgResponseTimeMs,    // Average response time
      targetCPUUtilization, // Target CPU utilization (0-1)
      instanceCapacity,     // Max RPS per instance
      redundancyFactor,     // Extra instances for redundancy
    } = params;
    
    // Little's Law: N = λ * W
    // N = average number of requests in system
    // λ = arrival rate (RPS)
    // W = average time in system (response time)
    const avgConcurrentRequests = peakRPS * (avgResponseTimeMs / 1000);
    
    // Instances needed for current load
    const rawInstances = Math.ceil(peakRPS / (instanceCapacity * targetCPUUtilization));
    
    // Add redundancy
    const recommendedInstances = Math.ceil(rawInstances * redundancyFactor);
    
    return {
      avgConcurrentRequests: avgConcurrentRequests.toFixed(1),
      rawInstances,
      recommendedInstances,
      analysis: {
        currentLoad: `${peakRPS} RPS`,
        perInstanceCapacity: `${instanceCapacity} RPS`,
        targetUtilization: `${targetCPUUtilization * 100}%`,
        redundancyFactor: `${redundancyFactor}x`
      }
    };
  }

  // Database sizing
  calculateDatabaseCapacity(params) {
    const {
      dailyNewRecords,
      avgRecordSizeBytes,
      retentionDays,
      indexOverhead,      // typically 2-3x
      replicationFactor,  // typically 2-3
    } = params;
    
    const rawDataGB = (dailyNewRecords * avgRecordSizeBytes * retentionDays) / (1024 ** 3);
    const withIndexes = rawDataGB * indexOverhead;
    const withReplication = withIndexes * replicationFactor;
    
    // Add 30% buffer
    const recommended = withReplication * 1.3;
    
    return {
      rawDataGB: rawDataGB.toFixed(2),
      withIndexesGB: withIndexes.toFixed(2),
      withReplicationGB: withReplication.toFixed(2),
      recommendedGB: Math.ceil(recommended),
      monthlyGrowthGB: (dailyNewRecords * avgRecordSizeBytes * 30 / (1024 ** 3) * indexOverhead).toFixed(2)
    };
  }

  // Cache sizing
  calculateCacheCapacity(params) {
    const {
      hotDataPercent,     // % of data that's "hot"
      totalDataGB,        // Total dataset size
      hitRateTarget,      // Target cache hit rate (0-1)
      avgItemSizeBytes,   // Average cached item size
    } = params;
    
    // Working set size
    const workingSetGB = totalDataGB * hotDataPercent;
    
    // Adjust for target hit rate
    // More cache = higher hit rate (diminishing returns)
    const cacheGB = workingSetGB * hitRateTarget;
    
    // Calculate expected hit rate with different cache sizes
    const hitRateCurve = [0.25, 0.5, 0.75, 1.0, 1.5, 2.0].map(multiplier => ({
      cacheGB: (workingSetGB * multiplier).toFixed(1),
      estimatedHitRate: Math.min(0.99, multiplier * 0.6).toFixed(2)
    }));
    
    return {
      workingSetGB: workingSetGB.toFixed(2),
      recommendedCacheGB: cacheGB.toFixed(2),
      hitRateCurve
    };
  }

  // Cost estimation
  estimateCosts(infrastructure) {
    const costs = {
      compute: 0,
      database: 0,
      cache: 0,
      storage: 0,
      network: 0
    };
    
    // EC2/ECS costs (simplified)
    if (infrastructure.compute) {
      const { instances, instanceType } = infrastructure.compute;
      const instanceCosts = {
        't3.micro': 0.0104,
        't3.small': 0.0208,
        't3.medium': 0.0416,
        'm5.large': 0.096,
        'm5.xlarge': 0.192,
        'm5.2xlarge': 0.384
      };
      costs.compute = (instanceCosts[instanceType] || 0.1) * instances * 24 * 30;
    }
    
    // RDS costs
    if (infrastructure.database) {
      const { instanceClass, storageGB, replicaCount } = infrastructure.database;
      costs.database = 200 * replicaCount + storageGB * 0.115; // Simplified
    }
    
    // ElastiCache costs
    if (infrastructure.cache) {
      const { nodeType, nodeCount } = infrastructure.cache;
      costs.cache = 70 * nodeCount; // Simplified
    }
    
    return {
      ...costs,
      total: Object.values(costs).reduce((a, b) => a + b, 0),
      currency: 'USD/month'
    };
  }

  // Generate capacity report
  generateReport(currentMetrics, growthRate) {
    const periods = [
      { months: 3, label: '3 months' },
      { months: 6, label: '6 months' },
      { months: 12, label: '12 months' }
    ];
    
    return periods.map(({ months, label }) => {
      const multiplier = Math.pow(1 + growthRate, months / 12);
      
      return {
        period: label,
        projectedRPS: Math.round(currentMetrics.rps * multiplier),
        projectedUsers: Math.round(currentMetrics.users * multiplier),
        projectedStorage: (currentMetrics.storageGB * multiplier).toFixed(1),
        recommendedInstances: this.calculateInstances({
          peakRPS: currentMetrics.rps * multiplier,
          avgResponseTimeMs: 200,
          targetCPUUtilization: 0.6,
          instanceCapacity: 1000,
          redundancyFactor: 1.5
        }).recommendedInstances
      };
    });
  }
}

// ตัวอย่างการใช้งาน
const planner = new CapacityPlanner({});

// คำนวณ compute capacity
console.log('=== Compute Capacity ===');
console.log(planner.calculateInstances({
  peakRPS: 1000,
  avgResponseTimeMs: 200,
  targetCPUUtilization: 0.6,
  instanceCapacity: 500,
  redundancyFactor: 1.5
}));

// คำนวณ database capacity
console.log('\n=== Database Capacity ===');
console.log(planner.calculateDatabaseCapacity({
  dailyNewRecords: 100000,
  avgRecordSizeBytes: 512,
  retentionDays: 365,
  indexOverhead: 2.5,
  replicationFactor: 2
}));

// Capacity report for 12 months
console.log('\n=== 12-Month Growth Projection ===');
const report = planner.generateReport(
  { rps: 500, users: 10000, storageGB: 100 },
  0.3  // 30% annual growth
);
console.table(report);

module.exports = { CapacityPlanner };
```

---

## ขั้นตอนที่ 1564: Performance Test Automation

```javascript
// performance-automation.js
// CI/CD integration สำหรับ performance tests

const { exec } = require('child_process');
const util = require('util');
const execAsync = util.promisify(exec);

class PerformanceCICD {
  constructor(options = {}) {
    this.baseUrl = options.baseUrl || 'http://localhost:3000';
    this.thresholds = options.thresholds || {
      p95ResponseTime: 500,     // ms
      errorRate: 0.01,          // 1%
      minRPS: 100               // Requests per second
    };
  }

  async runK6Test(testFile, options = {}) {
    const envVars = [
      `BASE_URL=${this.baseUrl}`,
      ...Object.entries(options.env || {}).map(([k, v]) => `${k}=${v}`)
    ].join(' ');
    
    const cmd = `${envVars} k6 run --out json=/tmp/k6-results.json ${testFile}`;
    
    console.log(`Running: ${cmd}`);
    
    try {
      const { stdout, stderr } = await execAsync(cmd, {
        timeout: 600000 // 10 minutes
      });
      
      console.log(stdout);
      if (stderr) console.error(stderr);
      
      const results = require('/tmp/k6-results.json');
      return this.parseResults(results);
      
    } catch (err) {
      throw new Error(`Performance test failed: ${err.message}`);
    }
  }

  parseResults(rawResults) {
    const metrics = rawResults.metrics || {};
    
    return {
      http_req_duration: {
        p95: metrics.http_req_duration?.values?.['p(95)'] || 0,
        p99: metrics.http_req_duration?.values?.['p(99)'] || 0,
        avg: metrics.http_req_duration?.values?.avg || 0
      },
      http_req_failed: {
        rate: metrics.http_req_failed?.values?.rate || 0
      },
      http_reqs: {
        count: metrics.http_reqs?.values?.count || 0,
        rate: metrics.http_reqs?.values?.rate || 0
      }
    };
  }

  checkThresholds(results) {
    const violations = [];
    
    if (results.http_req_duration.p95 > this.thresholds.p95ResponseTime) {
      violations.push({
        metric: 'p95_response_time',
        actual: results.http_req_duration.p95,
        threshold: this.thresholds.p95ResponseTime,
        unit: 'ms'
      });
    }
    
    if (results.http_req_failed.rate > this.thresholds.errorRate) {
      violations.push({
        metric: 'error_rate',
        actual: (results.http_req_failed.rate * 100).toFixed(2) + '%',
        threshold: (this.thresholds.errorRate * 100) + '%'
      });
    }
    
    if (results.http_reqs.rate < this.thresholds.minRPS) {
      violations.push({
        metric: 'requests_per_second',
        actual: results.http_reqs.rate,
        threshold: this.thresholds.minRPS
      });
    }
    
    return {
      passed: violations.length === 0,
      violations
    };
  }

  async compareWithBaseline(currentResults, baselineFile) {
    let baseline;
    
    try {
      baseline = require(baselineFile);
    } catch {
      console.log('No baseline found, saving current results as baseline');
      const fs = require('fs');
      fs.writeFileSync(baselineFile, JSON.stringify(currentResults, null, 2));
      return { isNew: true };
    }
    
    const regressions = [];
    const improvements = [];
    
    // Compare metrics
    const p95Change = ((currentResults.http_req_duration.p95 - baseline.http_req_duration.p95) / baseline.http_req_duration.p95) * 100;
    
    if (p95Change > 10) {
      regressions.push({
        metric: 'p95_response_time',
        baseline: baseline.http_req_duration.p95,
        current: currentResults.http_req_duration.p95,
        change: `+${p95Change.toFixed(1)}%`
      });
    } else if (p95Change < -10) {
      improvements.push({
        metric: 'p95_response_time',
        change: `${p95Change.toFixed(1)}%`
      });
    }
    
    return { regressions, improvements, hasRegression: regressions.length > 0 };
  }

  generateReport(results, thresholdCheck, comparison) {
    return {
      summary: {
        passed: thresholdCheck.passed && !comparison?.hasRegression,
        timestamp: new Date().toISOString()
      },
      metrics: {
        p95ResponseTime: `${results.http_req_duration.p95.toFixed(0)}ms`,
        p99ResponseTime: `${results.http_req_duration.p99.toFixed(0)}ms`,
        avgResponseTime: `${results.http_req_duration.avg.toFixed(0)}ms`,
        errorRate: `${(results.http_req_failed.rate * 100).toFixed(2)}%`,
        requestsPerSecond: results.http_reqs.rate.toFixed(1),
        totalRequests: results.http_reqs.count
      },
      thresholds: thresholdCheck,
      comparison
    };
  }
}

// GitHub Actions integration
const generateGitHubSummary = (report) => {
  const status = report.summary.passed ? '✅ PASSED' : '❌ FAILED';
  
  return `
## Performance Test Results ${status}

### Key Metrics
| Metric | Value | Status |
|--------|-------|--------|
| P95 Response Time | ${report.metrics.p95ResponseTime} | ${parseInt(report.metrics.p95ResponseTime) < 500 ? '✅' : '❌'} |
| Error Rate | ${report.metrics.errorRate} | ${parseFloat(report.metrics.errorRate) < 1 ? '✅' : '❌'} |
| Requests/Second | ${report.metrics.requestsPerSecond} | ✅ |

${report.thresholds.violations.length > 0 ? `
### ❌ Threshold Violations
${report.thresholds.violations.map(v => `- **${v.metric}**: ${v.actual} (threshold: ${v.threshold})`).join('\n')}
` : '### ✅ All thresholds passed'}

${report.comparison?.regressions?.length > 0 ? `
### ⚠️ Performance Regressions vs Baseline
${report.comparison.regressions.map(r => `- **${r.metric}**: ${r.change} worse`).join('\n')}
` : ''}
`;
};

module.exports = { PerformanceCICD, generateGitHubSummary };
```

---

## ขั้นตอนที่ 1565-1580: Advanced Load Testing Scenarios

### JMeter Integration

```javascript
// jmeter-runner.js
// Run JMeter tests programmatically

const { exec } = require('child_process');
const fs = require('fs');
const path = require('path');

class JMeterRunner {
  constructor(options = {}) {
    this.jmeterPath = options.jmeterPath || '/opt/jmeter/bin/jmeter';
    this.testPlansDir = options.testPlansDir || './tests/performance';
    this.resultsDir = options.resultsDir || './performance-results';
    
    fs.mkdirSync(this.resultsDir, { recursive: true });
  }

  async runTest(testPlanFile, properties = {}) {
    const timestamp = Date.now();
    const resultsFile = path.join(this.resultsDir, `results_${timestamp}.jtl`);
    const reportDir = path.join(this.resultsDir, `report_${timestamp}`);
    
    const propsArgs = Object.entries(properties)
      .map(([k, v]) => `-J${k}=${v}`)
      .join(' ');
    
    const cmd = [
      this.jmeterPath,
      '-n',                          // Non-GUI mode
      '-t', testPlanFile,            // Test plan
      '-l', resultsFile,             // Results log
      '-e',                          // Generate HTML report
      '-o', reportDir,               // Report output dir
      propsArgs
    ].join(' ');
    
    return new Promise((resolve, reject) => {
      exec(cmd, { timeout: 3600000 }, (error, stdout, stderr) => {
        if (error && error.code !== 0) {
          reject(new Error(`JMeter failed: ${stderr}`));
          return;
        }
        
        resolve({
          resultsFile,
          reportDir,
          stdout,
          summary: this.parseJTL(resultsFile)
        });
      });
    });
  }

  parseJTL(jtlFile) {
    if (!fs.existsSync(jtlFile)) return null;
    
    const lines = fs.readFileSync(jtlFile, 'utf8').split('\n').filter(Boolean);
    const data = lines.slice(1).map(line => {
      const parts = line.split(',');
      return {
        timestamp: parseInt(parts[0]),
        elapsed: parseInt(parts[1]),
        label: parts[2],
        responseCode: parts[3],
        success: parts[7] === 'true'
      };
    }).filter(r => !isNaN(r.elapsed));
    
    const successful = data.filter(r => r.success);
    const failed = data.filter(r => !r.success);
    const durations = successful.map(r => r.elapsed).sort((a, b) => a - b);
    
    return {
      total: data.length,
      successful: successful.length,
      failed: failed.length,
      errorRate: (failed.length / data.length * 100).toFixed(2) + '%',
      avgResponseTime: (durations.reduce((a, b) => a + b, 0) / durations.length).toFixed(0) + 'ms',
      p95: durations[Math.floor(durations.length * 0.95)] + 'ms',
      p99: durations[Math.floor(durations.length * 0.99)] + 'ms'
    };
  }

  // Generate JMeter test plan programmatically
  generateTestPlan(config) {
    return `<?xml version="1.0" encoding="UTF-8"?>
<jmeterTestPlan version="1.2" properties="5.0">
  <hashTree>
    <TestPlan guiclass="TestPlanGui" testclass="TestPlan" testname="${config.name}">
      <boolProp name="TestPlan.serialize_threadgroups">false</boolProp>
    </TestPlan>
    <hashTree>
      <ThreadGroup guiclass="ThreadGroupGui" testclass="ThreadGroup" testname="Users">
        <intProp name="ThreadGroup.num_threads">${config.users}</intProp>
        <intProp name="ThreadGroup.ramp_time">${config.rampUpSeconds}</intProp>
        <intProp name="ThreadGroup.duration">${config.durationSeconds}</intProp>
      </ThreadGroup>
      <hashTree>
        <HTTPSamplerProxy guiclass="HttpTestSampleGui" testclass="HTTPSamplerProxy" testname="${config.endpoint}">
          <elementProp name="HTTPsampler.Arguments" elementType="Arguments">
            <collectionProp name="Arguments.arguments"/>
          </elementProp>
          <stringProp name="HTTPSampler.domain">${config.host}</stringProp>
          <intProp name="HTTPSampler.port">${config.port || 80}</intProp>
          <stringProp name="HTTPSampler.protocol">${config.protocol || 'http'}</stringProp>
          <stringProp name="HTTPSampler.path">${config.path}</stringProp>
          <stringProp name="HTTPSampler.method">${config.method || 'GET'}</stringProp>
        </HTTPSamplerProxy>
        <hashTree/>
      </hashTree>
    </hashTree>
  </hashTree>
</jmeterTestPlan>`;
  }
}

module.exports = { JMeterRunner };
```

### autocannon สำหรับ Quick Tests

```javascript
// autocannon-tests.js
// Quick performance tests ด้วย autocannon

const autocannon = require('autocannon');
const chalk = require('chalk');

async function runLoadTest(options = {}) {
  const config = {
    url: options.url || 'http://localhost:3000',
    connections: options.connections || 100,
    duration: options.duration || 30,
    headers: options.headers || {},
    pipelining: options.pipelining || 1,
    
    ...options
  };

  console.log(chalk.blue(`\nStarting load test: ${config.url}`));
  console.log(chalk.gray(`${config.connections} connections, ${config.duration}s duration`));

  const result = await autocannon(config);
  
  const passed = checkPerformance(result, options.thresholds);
  
  printResults(result, passed);
  
  return { result, passed };
}

function checkPerformance(result, thresholds = {}) {
  const violations = [];
  
  const p99 = result.latency.p99 || 0;
  const errorRate = (result.errors / result.requests.total) * 100;
  const rps = result.requests.average;
  
  if (thresholds.p99 && p99 > thresholds.p99) {
    violations.push(`P99 latency ${p99}ms > ${thresholds.p99}ms`);
  }
  
  if (thresholds.errorRate && errorRate > thresholds.errorRate) {
    violations.push(`Error rate ${errorRate.toFixed(2)}% > ${thresholds.errorRate}%`);
  }
  
  if (thresholds.minRPS && rps < thresholds.minRPS) {
    violations.push(`RPS ${rps.toFixed(0)} < ${thresholds.minRPS}`);
  }
  
  return violations.length === 0;
}

function printResults(result, passed) {
  const status = passed ? chalk.green('✅ PASSED') : chalk.red('❌ FAILED');
  
  console.log(`\n${status}\n`);
  
  const table = {
    'Requests/sec': result.requests.average.toFixed(0),
    'Latency (avg)': `${result.latency.average.toFixed(2)}ms`,
    'Latency (p95)': `${result.latency.p95}ms`,
    'Latency (p99)': `${result.latency.p99}ms`,
    'Total requests': result.requests.total,
    'Errors': result.errors,
    'Timeouts': result.timeouts,
    'Throughput': `${(result.throughput.average / 1024 / 1024).toFixed(2)} MB/s`
  };
  
  Object.entries(table).forEach(([key, value]) => {
    console.log(`  ${chalk.cyan(key.padEnd(20))} ${value}`);
  });
}

// Test runner สำหรับ multiple scenarios
async function runTestSuite(baseUrl) {
  const scenarios = [
    {
      name: 'Static Content',
      url: `${baseUrl}/`,
      connections: 200,
      duration: 30,
      thresholds: { p99: 100, errorRate: 0.1, minRPS: 1000 }
    },
    {
      name: 'API - List Products',
      url: `${baseUrl}/api/products`,
      connections: 100,
      duration: 30,
      thresholds: { p99: 500, errorRate: 1, minRPS: 200 }
    },
    {
      name: 'API - Search',
      url: `${baseUrl}/api/search?q=test`,
      connections: 50,
      duration: 30,
      thresholds: { p99: 1000, errorRate: 1, minRPS: 100 }
    }
  ];
  
  const allResults = [];
  
  for (const scenario of scenarios) {
    console.log(chalk.yellow(`\n=== ${scenario.name} ===`));
    const { result, passed } = await runLoadTest(scenario);
    allResults.push({ name: scenario.name, passed, result });
  }
  
  // Summary
  console.log(chalk.bold('\n=== Test Suite Summary ==='));
  const passCount = allResults.filter(r => r.passed).length;
  const status = passCount === allResults.length ? chalk.green('✅ ALL PASSED') : chalk.red(`❌ ${allResults.length - passCount} FAILED`);
  
  console.log(status);
  allResults.forEach(({ name, passed }) => {
    console.log(`  ${passed ? '✅' : '❌'} ${name}`);
  });
  
  return allResults;
}

module.exports = { runLoadTest, runTestSuite };
```

---

## แบบฝึกหัด

### Exercise 1: Load Test Pipeline
สร้าง GitHub Actions workflow ที่รัน k6 load tests ทุกครั้งที่ merge PR และ fail build ถ้าผลแย่กว่า baseline

### Exercise 2: Capacity Planning Document
สร้าง capacity planning document สำหรับ e-commerce app ที่คาดว่าจะมี 10x traffic growth ใน 12 เดือน

### Exercise 3: Chaos Testing
ใช้ k6 scenarios ทดสอบว่า system gracefully degrade เมื่อ database slow down

### คำถามทบทวน
1. ความแตกต่างระหว่าง load test, stress test, และ spike test คืออะไร?
2. Little's Law คืออะไร และนำไปใช้ใน capacity planning ได้อย่างไร?
3. p95 response time คืออะไร และสำคัญกว่า average อย่างไร?
4. เมื่อไหรควรรัน performance tests ใน CI/CD pipeline?

---

*ต่อไป: Part 89 - Incident Management*
