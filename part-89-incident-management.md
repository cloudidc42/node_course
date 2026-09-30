# Part 89 | ขั้นตอนที่ 1581-1600 จาก 1000+

## Incident Management สำหรับ Production Systems

ในส่วนนี้เราจะเรียนรู้การจัดการ incidents อย่างมืออาชีพ รวมถึง postmortems, runbooks, และ on-call practices

---

## ขั้นตอนที่ 1581: Incident Response Framework

```javascript
// incident-manager.js
// Automated incident management system

const EventEmitter = require('events');

// Incident Severity Levels
const SEVERITY = {
  SEV1: {
    level: 1,
    name: 'SEV1 - Critical',
    description: 'Complete service outage affecting all users',
    slaResponseTime: 15,   // minutes
    escalationTime: 30,    // minutes
    notifyChannels: ['pagerduty', 'slack-incident', 'email', 'sms']
  },
  SEV2: {
    level: 2,
    name: 'SEV2 - High',
    description: 'Major feature degradation affecting many users',
    slaResponseTime: 30,
    escalationTime: 60,
    notifyChannels: ['pagerduty', 'slack-incident', 'email']
  },
  SEV3: {
    level: 3,
    name: 'SEV3 - Medium',
    description: 'Minor feature issue affecting some users',
    slaResponseTime: 60,
    escalationTime: 240,
    notifyChannels: ['slack-incident', 'email']
  },
  SEV4: {
    level: 4,
    name: 'SEV4 - Low',
    description: 'Minor issue with minimal user impact',
    slaResponseTime: 240,
    escalationTime: 480,
    notifyChannels: ['slack-engineering']
  }
};

class IncidentManager extends EventEmitter {
  constructor(services) {
    super();
    this.services = services; // notifier, ticketing, etc.
    this.incidents = new Map();
    this.onCallSchedule = null;
  }

  async declare(data) {
    const incident = {
      id: `INC-${Date.now()}`,
      title: data.title,
      description: data.description,
      severity: SEVERITY[data.severity] || SEVERITY.SEV3,
      status: 'open',
      
      // Timeline
      declaredAt: new Date().toISOString(),
      detectedAt: data.detectedAt || new Date().toISOString(),
      resolvedAt: null,
      
      // People
      declaredBy: data.declaredBy,
      incidentCommander: null,
      responders: [],
      
      // Impact
      affectedServices: data.affectedServices || [],
      affectedUsers: data.affectedUsers || 'unknown',
      
      // Progress
      updates: [],
      actions: [],
      
      // Metrics
      ttd: null,  // Time to Detect
      ttr: null,  // Time to Respond
      ttrs: null  // Time to Resolve
    };
    
    this.incidents.set(incident.id, incident);
    
    console.log(`INCIDENT DECLARED: ${incident.id} - ${incident.severity.name}`);
    
    // Calculate TTD
    if (data.detectedAt) {
      const detected = new Date(data.detectedAt);
      const declared = new Date(incident.declaredAt);
      incident.ttd = Math.round((declared - detected) / 60000); // minutes
    }
    
    // Notify responders
    await this.notifyResponders(incident);
    
    // Assign incident commander
    await this.assignIncidentCommander(incident);
    
    // Start escalation timer
    this.startEscalationTimer(incident);
    
    // Create tracking ticket
    await this.createTicket(incident);
    
    this.emit('incident:declared', incident);
    
    return incident;
  }

  async update(incidentId, update) {
    const incident = this.incidents.get(incidentId);
    if (!incident) throw new Error(`Incident ${incidentId} not found`);
    
    const updateEntry = {
      timestamp: new Date().toISOString(),
      author: update.author,
      message: update.message,
      action: update.action
    };
    
    incident.updates.push(updateEntry);
    
    if (update.action) {
      incident.actions.push({
        ...updateEntry,
        completed: false
      });
    }
    
    // Notify stakeholders
    await this.notifyUpdate(incident, updateEntry);
    
    this.emit('incident:updated', incident);
    
    return incident;
  }

  async resolve(incidentId, resolution) {
    const incident = this.incidents.get(incidentId);
    if (!incident) throw new Error(`Incident ${incidentId} not found`);
    
    incident.status = 'resolved';
    incident.resolvedAt = new Date().toISOString();
    incident.resolution = resolution;
    
    // Calculate MTTR
    const declared = new Date(incident.declaredAt);
    const resolved = new Date(incident.resolvedAt);
    incident.ttrs = Math.round((resolved - declared) / 60000);
    
    console.log(`INCIDENT RESOLVED: ${incidentId} (MTTR: ${incident.ttrs} minutes)`);
    
    // Notify resolution
    await this.notifyResolution(incident);
    
    // Schedule postmortem if SEV1 or SEV2
    if (incident.severity.level <= 2) {
      await this.schedulePostmortem(incident);
    }
    
    this.emit('incident:resolved', incident);
    
    return incident;
  }

  async notifyResponders(incident) {
    const onCall = await this.getOnCallResponders(incident.severity.level);
    
    for (const channel of incident.severity.notifyChannels) {
      await this.services.notifier.send(channel, {
        type: 'incident_declared',
        incident,
        onCall,
        urgency: incident.severity.level <= 2 ? 'high' : 'normal'
      });
    }
    
    incident.responders = onCall;
  }

  async getOnCallResponders(severityLevel) {
    // ดึง on-call schedule จาก PagerDuty หรือ Opsgenie
    return [
      { role: 'primary', name: 'John Doe', contact: '+1234567890' },
      { role: 'secondary', name: 'Jane Smith', contact: '+0987654321' }
    ];
  }

  async assignIncidentCommander(incident) {
    const onCall = incident.responders[0];
    incident.incidentCommander = onCall;
    
    await this.update(incident.id, {
      author: 'system',
      message: `Incident Commander assigned: ${onCall?.name}`
    });
  }

  startEscalationTimer(incident) {
    const { escalationTime } = incident.severity;
    
    setTimeout(async () => {
      if (incident.status === 'open') {
        console.warn(`ESCALATION: ${incident.id} - no response after ${escalationTime} minutes`);
        
        // Escalate to management
        await this.services.notifier.send('pagerduty', {
          type: 'escalation',
          incident,
          reason: `No response after ${escalationTime} minutes`
        });
        
        this.emit('incident:escalated', incident);
      }
    }, escalationTime * 60 * 1000);
  }

  async createTicket(incident) {
    // Create Jira/Linear ticket
    await this.services.ticketing.create({
      title: `[${incident.severity.name}] ${incident.title}`,
      description: incident.description,
      labels: ['incident', incident.severity.name.toLowerCase()],
      priority: incident.severity.level
    });
  }

  async schedulePostmortem(incident) {
    const postmortemDate = new Date();
    postmortemDate.setDate(postmortemDate.getDate() + 2); // 2 days after resolution
    
    await this.services.calendar.schedule({
      title: `Postmortem: ${incident.title} (${incident.id})`,
      date: postmortemDate,
      attendees: [...incident.responders.map(r => r.email)],
      duration: 60
    });
    
    console.log(`Postmortem scheduled for ${postmortemDate.toDateString()}`);
  }

  async notifyUpdate(incident, update) {
    await this.services.notifier.send('slack-incident', {
      type: 'incident_update',
      incident,
      update
    });
  }

  async notifyResolution(incident) {
    for (const channel of incident.severity.notifyChannels) {
      await this.services.notifier.send(channel, {
        type: 'incident_resolved',
        incident
      });
    }
  }

  getIncident(id) {
    return this.incidents.get(id);
  }

  getActiveIncidents() {
    return Array.from(this.incidents.values())
      .filter(i => i.status === 'open')
      .sort((a, b) => a.severity.level - b.severity.level);
  }

  getMetrics() {
    const incidents = Array.from(this.incidents.values());
    const resolved = incidents.filter(i => i.resolvedAt);
    
    if (resolved.length === 0) return { noData: true };
    
    const mttr = resolved.reduce((sum, i) => sum + (i.ttrs || 0), 0) / resolved.length;
    
    return {
      total: incidents.length,
      active: incidents.filter(i => !i.resolvedAt).length,
      resolved: resolved.length,
      meanTimeToResolve: `${Math.round(mttr)} minutes`,
      bySeverity: Object.keys(SEVERITY).reduce((acc, sev) => {
        acc[sev] = incidents.filter(i => i.severity.name.includes(sev.replace('SEV', 'SEV'))).length;
        return acc;
      }, {})
    };
  }
}

module.exports = { IncidentManager, SEVERITY };
```

---

## ขั้นตอนที่ 1582: Runbook System

```javascript
// runbook-system.js
// Executable runbooks สำหรับ common incidents

class RunbookRegistry {
  constructor() {
    this.runbooks = new Map();
  }

  register(name, runbook) {
    this.runbooks.set(name, runbook);
    return this;
  }

  get(name) {
    const runbook = this.runbooks.get(name);
    if (!runbook) throw new Error(`Runbook '${name}' not found`);
    return runbook;
  }

  list() {
    return Array.from(this.runbooks.entries()).map(([name, rb]) => ({
      name,
      title: rb.title,
      severity: rb.severity,
      automatable: rb.automatable
    }));
  }
}

class Runbook {
  constructor(config) {
    this.title = config.title;
    this.description = config.description;
    this.severity = config.severity;
    this.automatable = config.automatable || false;
    this.steps = config.steps;
    this.preconditions = config.preconditions || [];
    this.postconditions = config.postconditions || [];
  }

  async execute(context, executor = null) {
    const execution = {
      runbookTitle: this.title,
      startedAt: new Date().toISOString(),
      steps: [],
      status: 'running'
    };

    // Check preconditions
    for (const precondition of this.preconditions) {
      const met = await precondition.check(context);
      if (!met) {
        execution.status = 'failed';
        execution.error = `Precondition failed: ${precondition.name}`;
        return execution;
      }
    }

    // Execute steps
    for (let i = 0; i < this.steps.length; i++) {
      const step = this.steps[i];
      const stepResult = {
        step: i + 1,
        title: step.title,
        startedAt: new Date().toISOString()
      };

      try {
        if (step.automated && executor) {
          // Auto-execute if automation is available
          stepResult.result = await step.execute(context);
          stepResult.status = 'completed';
          stepResult.automated = true;
        } else if (executor) {
          // Ask human executor
          const response = await executor.prompt({
            step: i + 1,
            title: step.title,
            instructions: step.instructions,
            options: step.options
          });
          stepResult.response = response;
          stepResult.status = 'completed';
        } else {
          // Just log step for manual execution
          console.log(`\nStep ${i + 1}: ${step.title}`);
          console.log(step.instructions);
        }

      } catch (err) {
        stepResult.status = 'failed';
        stepResult.error = err.message;

        if (step.critical) {
          execution.status = 'failed';
          execution.steps.push(stepResult);
          return execution;
        }
      }

      stepResult.completedAt = new Date().toISOString();
      execution.steps.push(stepResult);
    }

    // Check postconditions
    for (const postcondition of this.postconditions) {
      const verified = await postcondition.verify(context);
      if (!verified) {
        execution.warnings = execution.warnings || [];
        execution.warnings.push(`Postcondition not met: ${postcondition.name}`);
      }
    }

    execution.status = 'completed';
    execution.completedAt = new Date().toISOString();
    return execution;
  }
}

// Common Runbooks

// Database Connection Exhaustion Runbook
const dbConnectionRunbook = new Runbook({
  title: 'Database Connection Pool Exhaustion',
  description: 'Steps to handle connection pool exhaustion',
  severity: 'SEV2',
  automatable: true,
  
  preconditions: [
    {
      name: 'verify_alert_source',
      check: async (ctx) => {
        const metrics = await ctx.getMetric('db_connection_pool_utilization');
        return metrics > 0.9;
      }
    }
  ],
  
  steps: [
    {
      title: 'Identify Connection Usage',
      automated: true,
      execute: async (ctx) => {
        const result = await ctx.db.query(`
          SELECT count(*) as total_connections,
                 state,
                 wait_event_type,
                 wait_event
          FROM pg_stat_activity
          GROUP BY state, wait_event_type, wait_event
          ORDER BY total_connections DESC
        `);
        
        ctx.connectionStats = result.rows;
        return result.rows;
      }
    },
    {
      title: 'Kill Idle Connections',
      automated: true,
      execute: async (ctx) => {
        const result = await ctx.db.query(`
          SELECT pg_terminate_backend(pid)
          FROM pg_stat_activity
          WHERE state = 'idle'
          AND query_start < NOW() - INTERVAL '10 minutes'
          AND pid <> pg_backend_pid()
        `);
        
        return { terminated: result.rowCount };
      }
    },
    {
      title: 'Increase Connection Pool Size',
      instructions: `
        1. Go to application configuration
        2. Increase DB_POOL_MAX from current value to ${ctx => ctx.currentMax * 2}
        3. Rolling restart all instances:
           kubectl rollout restart deployment/api
        4. Monitor connection counts
      `,
      critical: false
    },
    {
      title: 'Monitor Recovery',
      automated: true,
      execute: async (ctx) => {
        const checkInterval = 30000;
        const maxWait = 300000;
        const startTime = Date.now();
        
        while (Date.now() - startTime < maxWait) {
          const metrics = await ctx.getMetric('db_connection_pool_utilization');
          if (metrics < 0.7) {
            return { recovered: true, utilization: metrics };
          }
          await new Promise(resolve => setTimeout(resolve, checkInterval));
        }
        
        throw new Error('Connection pool did not recover within 5 minutes');
      }
    }
  ],
  
  postconditions: [
    {
      name: 'verify_pool_utilization_normal',
      verify: async (ctx) => {
        const utilization = await ctx.getMetric('db_connection_pool_utilization');
        return utilization < 0.7;
      }
    }
  ]
});

// High CPU Runbook
const highCPURunbook = new Runbook({
  title: 'High CPU Usage',
  description: 'Steps to diagnose and resolve high CPU usage',
  severity: 'SEV3',
  
  steps: [
    {
      title: 'Identify CPU-intensive processes',
      automated: true,
      execute: async (ctx) => {
        const { execSync } = require('child_process');
        const output = execSync("ps aux --sort=-%cpu | head -10").toString();
        ctx.topProcesses = output;
        return output;
      }
    },
    {
      title: 'Enable CPU profiling',
      instructions: 'Send SIGUSR1 to the Node.js process to start profiling:\nkill -SIGUSR1 $(pgrep node)',
      automated: true,
      execute: async (ctx) => {
        const pid = parseInt(require('child_process').execSync('pgrep node').toString().split('\n')[0]);
        process.kill(pid, 'SIGUSR1');
        return { pid, profiling: 'started' };
      }
    },
    {
      title: 'Scale up instances',
      automated: true,
      execute: async (ctx) => {
        const currentReplicas = await ctx.k8s.getDeploymentReplicas('api');
        const newReplicas = Math.min(currentReplicas + 2, 20);
        await ctx.k8s.scale('api', newReplicas);
        return { scaled: true, from: currentReplicas, to: newReplicas };
      }
    }
  ]
});

// Create registry
const registry = new RunbookRegistry();
registry.register('db-connection-exhaustion', dbConnectionRunbook);
registry.register('high-cpu', highCPURunbook);

module.exports = { RunbookRegistry, Runbook, registry };
```

---

## ขั้นตอนที่ 1583: Postmortem System

```javascript
// postmortem.js
// Blameless postmortem management

class PostmortemTemplate {
  static generate(incident) {
    return {
      title: `Postmortem: ${incident.title} (${incident.id})`,
      date: new Date().toISOString(),
      severity: incident.severity.name,
      duration: incident.ttrs,
      
      summary: {
        what: '',
        impact: '',
        rootCause: ''
      },
      
      timeline: incident.updates.map(u => ({
        time: u.timestamp,
        event: u.message,
        author: u.author
      })),
      
      analysis: {
        rootCause: '',
        contributingFactors: [],
        whatWorked: [],
        whatDidntWork: []
      },
      
      actionItems: [],
      
      lessons: [],
      
      metrics: {
        ttd: incident.ttd,  // Time to Detect
        ttr: incident.ttr,  // Time to Respond
        ttrs: incident.ttrs // Time to Resolve
      }
    };
  }
}

// API สำหรับ postmortem
const express = require('express');
const router = express.Router();

// Create postmortem
router.post('/incidents/:id/postmortem', async (req, res) => {
  const incident = incidentManager.getIncident(req.params.id);
  if (!incident) return res.status(404).json({ error: 'Incident not found' });
  
  const template = PostmortemTemplate.generate(incident);
  const postmortem = await postmortemService.create({
    ...template,
    ...req.body,
    incidentId: incident.id,
    createdBy: req.user.id
  });
  
  res.json(postmortem);
});

// Add action items
router.post('/postmortems/:id/actions', async (req, res) => {
  const { description, owner, dueDate, priority } = req.body;
  
  const actionItem = await postmortemService.addAction(req.params.id, {
    description,
    owner,
    dueDate,
    priority,
    status: 'open',
    createdAt: new Date().toISOString()
  });
  
  res.json(actionItem);
});

// Track action item completion
router.put('/postmortems/:id/actions/:actionId', async (req, res) => {
  const updated = await postmortemService.updateAction(
    req.params.id,
    req.params.actionId,
    req.body
  );
  res.json(updated);
});

module.exports = { PostmortemTemplate, router };
```

---

## ขั้นตอนที่ 1584: On-Call Best Practices

```javascript
// oncall-system.js
// On-call schedule และ notification management

class OnCallSchedule {
  constructor(config) {
    this.schedule = config.schedule; // Array of shifts
    this.escalationPolicy = config.escalationPolicy;
  }

  getCurrentOnCall() {
    const now = new Date();
    
    for (const shift of this.schedule) {
      const start = new Date(shift.start);
      const end = new Date(shift.end);
      
      if (now >= start && now < end) {
        return {
          primary: shift.primary,
          secondary: shift.secondary,
          shiftEnd: shift.end
        };
      }
    }
    
    // Fallback to default on-call
    return this.escalationPolicy.defaultOnCall;
  }

  getUpcomingShifts(days = 7) {
    const now = new Date();
    const future = new Date(now.getTime() + days * 24 * 60 * 60 * 1000);
    
    return this.schedule.filter(shift => {
      const start = new Date(shift.start);
      return start >= now && start <= future;
    });
  }
}

// Alert Fatigue Prevention
class AlertManager {
  constructor() {
    this.suppressedAlerts = new Map();
    this.alertCounts = new Map();
  }

  shouldAlert(alertKey, options = {}) {
    const {
      minInterval = 300000,      // 5 minutes minimum between same alert
      maxPerHour = 5,            // Max 5 of same alert per hour
      floodProtection = 10       // Stop after 10 alerts in burst
    } = options;
    
    const now = Date.now();
    const key = alertKey;
    
    // Check if suppressed
    if (this.suppressedAlerts.has(key)) {
      const suppressedUntil = this.suppressedAlerts.get(key);
      if (now < suppressedUntil) {
        return false;
      }
      this.suppressedAlerts.delete(key);
    }
    
    // Track alert count
    if (!this.alertCounts.has(key)) {
      this.alertCounts.set(key, []);
    }
    
    const counts = this.alertCounts.get(key);
    const oneHourAgo = now - 3600000;
    
    // Remove old counts
    const recentCounts = counts.filter(t => t > oneHourAgo);
    this.alertCounts.set(key, recentCounts);
    
    // Check rate limit
    if (recentCounts.length >= maxPerHour) {
      // Suppress for next 30 minutes
      this.suppressedAlerts.set(key, now + 1800000);
      console.warn(`Alert suppressed due to rate limit: ${key}`);
      return false;
    }
    
    // Check minimum interval
    if (recentCounts.length > 0) {
      const lastAlert = Math.max(...recentCounts);
      if (now - lastAlert < minInterval) {
        return false;
      }
    }
    
    // Flood protection
    const lastMinute = counts.filter(t => t > now - 60000);
    if (lastMinute.length >= floodProtection) {
      this.suppressedAlerts.set(key, now + 3600000); // Suppress for 1 hour
      console.error(`Alert flood detected for ${key}, suppressing for 1 hour`);
      return false;
    }
    
    // Allow alert
    counts.push(now);
    return true;
  }

  // Group similar alerts
  groupAlerts(alerts) {
    const groups = new Map();
    
    for (const alert of alerts) {
      const groupKey = `${alert.type}:${alert.service}`;
      
      if (!groups.has(groupKey)) {
        groups.set(groupKey, {
          key: groupKey,
          alerts: [],
          firstSeen: alert.timestamp,
          lastSeen: alert.timestamp
        });
      }
      
      const group = groups.get(groupKey);
      group.alerts.push(alert);
      group.lastSeen = Math.max(group.lastSeen, alert.timestamp);
    }
    
    return Array.from(groups.values());
  }
}

// SLO-based alerting
class SLOAlertManager {
  constructor(sloConfig) {
    this.slo = sloConfig;
  }

  // Error Budget Burn Rate Alert
  calculateBurnRate(errorRate, windowHours) {
    const hourlyBudget = (1 - this.slo.target) * this.slo.windowDays * 24;
    const burnRate = (errorRate * windowHours) / hourlyBudget;
    return burnRate;
  }

  shouldAlert(metrics) {
    const alerts = [];
    
    // Fast burn alert (critical)
    const fastBurnRate = this.calculateBurnRate(metrics.errorRate1h, 1);
    if (fastBurnRate > 14.4) {  // Will exhaust budget in 5 days
      alerts.push({
        severity: 'SEV1',
        type: 'slo_fast_burn',
        message: `Fast burn rate: ${fastBurnRate.toFixed(1)}x budget`,
        burnRate: fastBurnRate
      });
    }
    
    // Slow burn alert (warning)
    const slowBurnRate = this.calculateBurnRate(metrics.errorRate6h, 6);
    if (slowBurnRate > 6) {  // Will exhaust budget in 2 weeks
      alerts.push({
        severity: 'SEV3',
        type: 'slo_slow_burn',
        message: `Slow burn rate: ${slowBurnRate.toFixed(1)}x budget`,
        burnRate: slowBurnRate
      });
    }
    
    return alerts;
  }
}

module.exports = { OnCallSchedule, AlertManager, SLOAlertManager };
```

---

## ขั้นตอนที่ 1585-1600: Advanced Monitoring

### SLO Tracking System

```javascript
// slo-tracking.js
// Service Level Objectives tracking

class SLOTracker {
  constructor(db, options = {}) {
    this.db = db;
    this.windowDays = options.windowDays || 30;
  }

  async calculateSLO(service, sloTarget, metricQuery) {
    const windowStart = new Date();
    windowStart.setDate(windowStart.getDate() - this.windowDays);
    
    // Count total requests and errors
    const result = await this.db.query(`
      SELECT
        COUNT(*) FILTER (WHERE status_code < 500) as successful_requests,
        COUNT(*) as total_requests,
        PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY response_time) as p50,
        PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY response_time) as p95,
        PERCENTILE_CONT(0.99) WITHIN GROUP (ORDER BY response_time) as p99
      FROM ${metricQuery.table}
      WHERE service = $1 AND timestamp > $2
    `, [service, windowStart]);
    
    const { successful_requests, total_requests, p50, p95, p99 } = result.rows[0];
    
    const errorRate = 1 - (successful_requests / total_requests);
    const successRate = successful_requests / total_requests;
    
    const currentSLO = successRate * 100;
    const errorBudget = {
      total: (1 - sloTarget) * total_requests,
      consumed: total_requests - successful_requests,
      remaining: Math.max(0, (1 - sloTarget) * total_requests - (total_requests - successful_requests)),
      percentRemaining: Math.max(0, 100 - (errorRate / (1 - sloTarget) * 100))
    };
    
    return {
      service,
      period: {
        start: windowStart.toISOString(),
        end: new Date().toISOString(),
        days: this.windowDays
      },
      slo: {
        target: sloTarget * 100,
        current: currentSLO.toFixed(3),
        met: currentSLO >= sloTarget * 100
      },
      metrics: {
        totalRequests: parseInt(total_requests),
        successfulRequests: parseInt(successful_requests),
        errorRate: (errorRate * 100).toFixed(3) + '%',
        p50ResponseTime: `${Math.round(p50)}ms`,
        p95ResponseTime: `${Math.round(p95)}ms`,
        p99ResponseTime: `${Math.round(p99)}ms`
      },
      errorBudget: {
        ...errorBudget,
        percentRemaining: errorBudget.percentRemaining.toFixed(1) + '%',
        status: errorBudget.percentRemaining > 50 ? 'healthy' :
                errorBudget.percentRemaining > 25 ? 'warning' : 'critical'
      }
    };
  }

  async getDashboard() {
    const slos = [
      { service: 'api', target: 0.999, table: 'api_requests' },
      { service: 'auth', target: 0.9999, table: 'api_requests' },
      { service: 'checkout', target: 0.999, table: 'api_requests' }
    ];
    
    const results = await Promise.all(
      slos.map(slo => this.calculateSLO(slo.service, slo.target, slo))
    );
    
    return {
      timestamp: new Date().toISOString(),
      slos: results,
      overallHealth: results.every(r => r.slo.met) ? 'healthy' : 'degraded'
    };
  }
}

// Express API
const express = require('express');
const app = express();

const sloTracker = new SLOTracker(db);

app.get('/slo/dashboard', async (req, res) => {
  const dashboard = await sloTracker.getDashboard();
  res.json(dashboard);
});

app.get('/slo/:service', async (req, res) => {
  const slo = await sloTracker.calculateSLO(
    req.params.service,
    parseFloat(req.query.target) || 0.999,
    { table: 'api_requests' }
  );
  res.json(slo);
});

module.exports = { SLOTracker };
```

---

## แบบฝึกหัด

### Exercise 1: Incident Bot
สร้าง Slack bot ที่ช่วย manage incidents: declare, update, resolve พร้อม automated runbook execution

### Exercise 2: Postmortem Dashboard
สร้าง dashboard แสดง postmortem trends เช่น MTTR, most common root causes, action item completion rate

### Exercise 3: SLO Error Budget Dashboard
สร้าง real-time dashboard แสดง SLO status และ error budget burn rate

### คำถามทบทวน
1. Blameless postmortem คืออะไร และทำไมสำคัญ?
2. SLO, SLI, SLA ต่างกันอย่างไร?
3. Error budget คืออะไร และนำไปใช้ในการตัดสินใจ release ได้อย่างไร?
4. Alert fatigue คืออะไร และป้องกันได้อย่างไร?

---

*ต่อไป: Part 90 - Developer Experience*
