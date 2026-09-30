# Part 97 | ขั้นตอนที่ 1741-1760 จาก 1000+

## Technical Leadership และ Architecture Decision Making

ในส่วนนี้เราจะเรียนรู้ทักษะ Technical Leadership, การเขียน Architecture Decision Records (ADRs), และการจัดการ Technical Debt

---

## ขั้นตอนที่ 1741: Architecture Decision Records (ADRs)

ADR คือเอกสารที่บันทึกการตัดสินใจทางสถาปัตยกรรมที่สำคัญ รวมถึงบริบท ตัวเลือก และเหตุผลที่เลือก

```markdown
# ADR-001: Use PostgreSQL as Primary Database

**Date:** 2024-01-15
**Status:** Accepted
**Deciders:** Tech Lead, Backend Team

## Context
Our application needs a reliable, scalable database that supports complex queries, transactions, and JSON data. We evaluated several options.

## Decision Drivers
- Need for ACID transactions
- Complex query requirements
- Team familiarity
- Scalability to 10M users
- Cost constraints

## Considered Options

### Option 1: PostgreSQL
**Pros:**
- Full ACID compliance
- Excellent JSON support with JSONB
- Mature ecosystem
- pgvector for AI features
- Free and open source

**Cons:**
- Vertical scaling complexity
- Not as fast as NoSQL for simple reads

### Option 2: MongoDB
**Pros:**
- Flexible schema
- Horizontal scaling easier
- Good developer experience

**Cons:**
- Eventual consistency concerns
- Complex transactions
- Higher cost at scale

### Option 3: MySQL
**Pros:**
- Very mature, battle-tested
- Good performance

**Cons:**
- JSON support less sophisticated
- Less feature-rich than PostgreSQL

## Decision
Use PostgreSQL as primary database.

## Consequences
**Positive:**
- Strong consistency guarantees
- Rich querying capabilities
- Single technology to master

**Negative:**
- Need to plan for read replicas as we scale
- Schema migrations require careful planning

## Implementation Notes
- Use pg connection pooling
- Set up read replica for analytics queries
- Use JSONB for flexible data fields
```

```javascript
// adr-manager.js
// Manage and query ADRs programmatically

const fs = require('fs').promises;
const path = require('path');

class ADRManager {
  constructor(adrDir) {
    this.adrDir = adrDir;
    this.adrs = new Map();
  }

  async loadAll() {
    const files = await fs.readdir(this.adrDir);
    const adrFiles = files.filter(f => f.match(/^adr-\d+-.+\.md$/));
    
    for (const file of adrFiles) {
      const content = await fs.readFile(path.join(this.adrDir, file), 'utf8');
      const adr = this.parse(content);
      this.adrs.set(adr.id, { ...adr, file });
    }
    
    return this.adrs;
  }

  parse(content) {
    const titleMatch = content.match(/^# (ADR-\d+): (.+)$/m);
    const statusMatch = content.match(/\*\*Status:\*\* (.+)$/m);
    const dateMatch = content.match(/\*\*Date:\*\* (.+)$/m);
    
    return {
      id: titleMatch?.[1],
      title: titleMatch?.[2],
      status: statusMatch?.[1]?.trim(),
      date: dateMatch?.[1]?.trim(),
      content
    };
  }

  async create(title, template = {}) {
    const id = this.nextId();
    const adrNum = id.replace('ADR-', '').padStart(3, '0');
    const slug = title.toLowerCase().replace(/\s+/g, '-').replace(/[^a-z0-9-]/g, '');
    const filename = `adr-${adrNum}-${slug}.md`;
    
    const content = this.generateTemplate(id, title, template);
    
    await fs.writeFile(path.join(this.adrDir, filename), content, 'utf8');
    
    return { id, filename };
  }

  nextId() {
    const ids = Array.from(this.adrs.keys()).map(id => parseInt(id.replace('ADR-', '')));
    const nextNum = ids.length > 0 ? Math.max(...ids) + 1 : 1;
    return `ADR-${String(nextNum).padStart(3, '0')}`;
  }

  generateTemplate(id, title, template) {
    return `# ${id}: ${title}

**Date:** ${new Date().toISOString().split('T')[0]}
**Status:** Proposed
**Deciders:** ${template.deciders || '[Add deciders]'}

## Context
${template.context || '[Describe the context and problem]'}

## Decision Drivers
${template.drivers?.map(d => `- ${d}`).join('\n') || '- [Add decision drivers]'}

## Considered Options
${template.options?.map(o => `### ${o}\n[Add pros/cons]`).join('\n\n') || '### Option 1\n[Add options]'}

## Decision
${template.decision || '[State the decision]'}

## Consequences
**Positive:**
- [Add positive consequences]

**Negative:**
- [Add negative consequences]
`;
  }

  getByStatus(status) {
    return Array.from(this.adrs.values()).filter(adr => adr.status === status);
  }

  async updateStatus(adrId, newStatus) {
    const adr = this.adrs.get(adrId);
    if (!adr) throw new Error(`ADR ${adrId} not found`);
    
    const updated = adr.content.replace(
      /\*\*Status:\*\* .+/,
      `**Status:** ${newStatus}`
    );
    
    await fs.writeFile(path.join(this.adrDir, adr.file), updated, 'utf8');
    adr.status = newStatus;
  }
}

module.exports = { ADRManager };
```

---

## ขั้นตอนที่ 1742: Technical Debt Management

```javascript
// tech-debt-tracker.js
// Track and prioritize technical debt

class TechDebtItem {
  constructor(data) {
    this.id = data.id;
    this.title = data.title;
    this.description = data.description;
    this.type = data.type; // 'code', 'architecture', 'test', 'documentation', 'dependency'
    this.severity = data.severity; // 'low', 'medium', 'high', 'critical'
    this.effort = data.effort; // Story points or days
    this.impact = data.impact; // 1-10 scale
    this.files = data.files || [];
    this.createdAt = data.createdAt || new Date();
    this.deadline = data.deadline;
    this.owner = data.owner;
    this.tags = data.tags || [];
  }

  get priority() {
    const severityScore = { low: 1, medium: 2, high: 3, critical: 4 };
    const urgency = this.deadline ? 
      Math.max(0, 30 - Math.ceil((new Date(this.deadline) - new Date()) / 86400000)) : 0;
    
    return (severityScore[this.severity] * this.impact) + urgency;
  }

  get roi() {
    // Return on Investment: impact per unit effort
    return this.impact / this.effort;
  }
}

class TechDebtTracker {
  constructor(storage) {
    this.storage = storage; // Could be a database
    this.items = new Map();
  }

  async loadItems() {
    const items = await this.storage.findAll('tech_debt');
    items.forEach(item => this.items.set(item.id, new TechDebtItem(item)));
  }

  async addItem(data) {
    const item = new TechDebtItem({
      ...data,
      id: `TD-${Date.now()}`
    });
    
    this.items.set(item.id, item);
    await this.storage.save('tech_debt', item);
    
    return item;
  }

  getPrioritized(limit = 10) {
    return Array.from(this.items.values())
      .sort((a, b) => b.priority - a.priority)
      .slice(0, limit);
  }

  getByType(type) {
    return Array.from(this.items.values()).filter(item => item.type === type);
  }

  getSummary() {
    const items = Array.from(this.items.values());
    
    const bySeverity = items.reduce((acc, item) => {
      acc[item.severity] = (acc[item.severity] || 0) + 1;
      return acc;
    }, {});
    
    const totalEffort = items.reduce((sum, item) => sum + item.effort, 0);
    const criticalCount = bySeverity.critical || 0;
    
    return {
      total: items.length,
      bySeverity,
      totalEffort,
      criticalCount,
      topItems: this.getPrioritized(5).map(i => ({
        id: i.id, title: i.title, severity: i.severity, priority: i.priority
      }))
    };
  }

  // Automated tech debt detection from code analysis
  async detectFromCode(codebaseDir) {
    const detected = [];
    const { execSync } = require('child_process');
    
    // Find TODO/FIXME/HACK comments
    try {
      const output = execSync(
        `grep -rn "TODO\\|FIXME\\|HACK\\|XXX" ${codebaseDir} --include="*.js" --include="*.ts"`,
        { encoding: 'utf8' }
      );
      
      const lines = output.split('\n').filter(Boolean);
      for (const line of lines.slice(0, 50)) { // Limit to first 50
        const match = line.match(/^(.+):(\d+):.+(TODO|FIXME|HACK|XXX):?\s*(.+)$/);
        if (match) {
          detected.push({
            type: 'code',
            title: `${match[3]}: ${match[4].substring(0, 60)}`,
            description: `Found in ${match[1]} line ${match[2]}`,
            severity: match[3] === 'FIXME' ? 'high' : 'medium',
            effort: 1,
            impact: 3,
            files: [match[1]]
          });
        }
      }
    } catch { /* grep returns non-zero when no matches */ }
    
    return detected;
  }
}

module.exports = { TechDebtItem, TechDebtTracker };
```

---

## ขั้นตอนที่ 1743-1760: Engineering Metrics และ Team Health

```javascript
// engineering-metrics.js
// Track engineering team health metrics

class EngineeringMetrics {
  constructor(githubClient, jiraClient) {
    this.github = githubClient;
    this.jira = jiraClient;
  }

  async collectDORAMetrics(repo, period = 30) {
    const endDate = new Date();
    const startDate = new Date(endDate - period * 86400000);
    
    const [deployments, incidents, prMetrics] = await Promise.all([
      this.getDeployments(repo, startDate, endDate),
      this.getIncidents(startDate, endDate),
      this.getPRMetrics(repo, startDate, endDate)
    ]);
    
    return {
      deploymentFrequency: this.calcDeployFrequency(deployments, period),
      leadTime: prMetrics.avgLeadTime,
      changeFailureRate: this.calcChangeFailureRate(deployments, incidents),
      mttr: this.calcMTTR(incidents),
      period,
      generatedAt: new Date().toISOString()
    };
  }

  calcDeployFrequency(deployments, periodDays) {
    const deploymentsPerDay = deployments.length / periodDays;
    
    if (deploymentsPerDay >= 1) return { value: deploymentsPerDay, rating: 'Elite', label: `${deploymentsPerDay.toFixed(1)}/day` };
    if (deploymentsPerDay >= 1/7) return { value: deploymentsPerDay, rating: 'High', label: `${(deploymentsPerDay * 7).toFixed(1)}/week` };
    if (deploymentsPerDay >= 1/30) return { value: deploymentsPerDay, rating: 'Medium', label: `${(deploymentsPerDay * 30).toFixed(1)}/month` };
    return { value: deploymentsPerDay, rating: 'Low', label: 'Less than monthly' };
  }

  calcChangeFailureRate(deployments, incidents) {
    if (deployments.length === 0) return { value: 0, rating: 'Elite' };
    
    const rate = (incidents.length / deployments.length) * 100;
    let rating = 'Low';
    if (rate <= 5) rating = 'Elite';
    else if (rate <= 10) rating = 'High';
    else if (rate <= 15) rating = 'Medium';
    
    return { value: parseFloat(rate.toFixed(1)), rating };
  }

  calcMTTR(incidents) {
    if (incidents.length === 0) return { value: 0, rating: 'Elite' };
    
    const avgHours = incidents
      .filter(i => i.resolvedAt)
      .reduce((sum, i) => {
        const hours = (new Date(i.resolvedAt) - new Date(i.createdAt)) / 3600000;
        return sum + hours;
      }, 0) / incidents.length;
    
    let rating = 'Low';
    if (avgHours < 1) rating = 'Elite';
    else if (avgHours < 24) rating = 'High';
    else if (avgHours < 168) rating = 'Medium';
    
    return { value: parseFloat(avgHours.toFixed(1)), unit: 'hours', rating };
  }

  async getPRMetrics(repo, startDate, endDate) {
    // Simplified - would use actual GitHub API
    const prs = await this.github.listPRs(repo, { startDate, endDate, state: 'closed' });
    
    const leadTimes = prs
      .filter(pr => pr.mergedAt)
      .map(pr => (new Date(pr.mergedAt) - new Date(pr.createdAt)) / 3600000);
    
    const avgLeadTime = leadTimes.reduce((sum, t) => sum + t, 0) / leadTimes.length;
    
    return {
      totalPRs: prs.length,
      avgLeadTime: parseFloat(avgLeadTime.toFixed(1)),
      avgReviewCycles: prs.reduce((sum, pr) => sum + (pr.reviewCount || 1), 0) / prs.length
    };
  }

  // Sprint velocity tracking
  async getSprintVelocity(sprintHistory = []) {
    const velocities = sprintHistory.map(sprint => ({
      sprint: sprint.name,
      committed: sprint.committedPoints,
      completed: sprint.completedPoints,
      velocity: sprint.completedPoints,
      completion: (sprint.completedPoints / sprint.committedPoints * 100).toFixed(0)
    }));
    
    const avgVelocity = velocities.reduce((sum, s) => sum + s.velocity, 0) / velocities.length;
    const trend = this.calculateTrend(velocities.map(v => v.velocity));
    
    return {
      velocities,
      avgVelocity: parseFloat(avgVelocity.toFixed(1)),
      trend: trend > 0 ? 'improving' : trend < 0 ? 'declining' : 'stable'
    };
  }

  calculateTrend(values) {
    if (values.length < 2) return 0;
    const recent = values.slice(-3);
    const older = values.slice(0, -3);
    const recentAvg = recent.reduce((a, b) => a + b, 0) / recent.length;
    const olderAvg = older.reduce((a, b) => a + b, 0) / (older.length || 1);
    return recentAvg - olderAvg;
  }

  async getDeployments(repo, startDate, endDate) {
    // Placeholder - would use actual GitHub Deployments API
    return [];
  }

  async getIncidents(startDate, endDate) {
    // Placeholder - would use PagerDuty/OpsGenie API
    return [];
  }
}

// Architecture Review Checklist
class ArchitectureReview {
  constructor() {
    this.categories = {
      reliability: [
        'Single points of failure identified and mitigated',
        'Graceful degradation strategy defined',
        'Health checks implemented',
        'Circuit breakers in place',
        'Retry logic with backoff'
      ],
      scalability: [
        'Horizontal scaling supported',
        'Database connection pooling',
        'Caching strategy defined',
        'Stateless service design',
        'Performance benchmarks established'
      ],
      security: [
        'Authentication and authorization',
        'Input validation',
        'Secrets management',
        'Encryption at rest and in transit',
        'Security headers configured'
      ],
      operability: [
        'Structured logging',
        'Metrics and alerting',
        'Distributed tracing',
        'Runbooks documented',
        'On-call rotation defined'
      ],
      maintainability: [
        'Code review process',
        'Testing strategy (unit, integration, e2e)',
        'Documentation',
        'ADRs for key decisions',
        'Dependency update strategy'
      ]
    };
  }

  generateReview(systemName) {
    const review = {
      system: systemName,
      date: new Date().toISOString().split('T')[0],
      categories: {}
    };
    
    for (const [category, items] of Object.entries(this.categories)) {
      review.categories[category] = {
        items: items.map(item => ({ item, status: 'pending', notes: '' })),
        score: 0
      };
    }
    
    return review;
  }

  calculateScore(review) {
    let totalItems = 0;
    let passedItems = 0;
    
    for (const category of Object.values(review.categories)) {
      for (const item of category.items) {
        totalItems++;
        if (item.status === 'pass') passedItems++;
      }
    }
    
    return { score: Math.round(passedItems / totalItems * 100), passed: passedItems, total: totalItems };
  }
}

module.exports = { EngineeringMetrics, ArchitectureReview };
```

---

## แบบฝึกหัด

### Exercise 1: ADR Generator
สร้าง CLI tool ที่ generate ADR template และ track status changes

### Exercise 2: Tech Debt Dashboard
สร้าง web dashboard ที่แสดง tech debt ทั้งหมดพร้อม priority scoring

### Exercise 3: DORA Metrics Report
สร้าง automated report ที่ calculate DORA metrics จาก GitHub ทุกสัปดาห์

### คำถามทบทวน
1. ADRs ช่วยทีมอย่างไรในระยะยาว?
2. วิธีจัดการ technical debt โดยไม่ให้กระทบ velocity?
3. DORA metrics ช่วย predict software delivery performance อย่างไร?

---

*ต่อไป: Part 98 - Open Source Development*
