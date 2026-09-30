# Part 91 | ขั้นตอนที่ 1621-1640 จาก 1000+

## Platform Engineering และ Internal Developer Platform (IDP)

ในส่วนนี้เราจะเรียนรู้การสร้าง Internal Developer Platform ที่ช่วยให้ developer ทำงานได้เร็วขึ้น

---

## ขั้นตอนที่ 1621: Platform Engineering Fundamentals

```javascript
// idp-overview.js
// Internal Developer Platform components

/*
Internal Developer Platform (IDP) Components:
1. Self-service infrastructure provisioning
2. Golden paths (opinionated templates)
3. Service catalog
4. Unified observability
5. CI/CD automation
6. Secret management
7. Environment management
*/

// Service Catalog - registry ของ services ทั้งหมด
class ServiceCatalog {
  constructor(db) {
    this.db = db;
  }

  async register(serviceConfig) {
    const service = {
      id: serviceConfig.id || serviceConfig.name.toLowerCase().replace(/\s+/g, '-'),
      name: serviceConfig.name,
      description: serviceConfig.description,
      team: serviceConfig.team,
      tier: serviceConfig.tier || 2, // 1=critical, 2=standard, 3=experimental
      
      // Tech stack
      language: serviceConfig.language || 'nodejs',
      framework: serviceConfig.framework,
      runtime: serviceConfig.runtime || 'node18',
      
      // Infrastructure
      deployments: serviceConfig.deployments || [],
      dependencies: serviceConfig.dependencies || [],
      
      // Documentation
      runbook: serviceConfig.runbook,
      docs: serviceConfig.docs,
      repository: serviceConfig.repository,
      
      // Ownership
      oncall: serviceConfig.oncall,
      slackChannel: serviceConfig.slackChannel,
      
      // Metadata
      tags: serviceConfig.tags || [],
      createdAt: new Date().toISOString(),
      updatedAt: new Date().toISOString()
    };
    
    await this.db.query(`
      INSERT INTO services (id, data)
      VALUES ($1, $2)
      ON CONFLICT (id) DO UPDATE SET data = $2, updated_at = NOW()
    `, [service.id, JSON.stringify(service)]);
    
    return service;
  }

  async findAll(filters = {}) {
    let query = 'SELECT data FROM services WHERE 1=1';
    const params = [];
    
    if (filters.team) {
      params.push(filters.team);
      query += ` AND data->>'team' = $${params.length}`;
    }
    
    if (filters.tier) {
      params.push(filters.tier);
      query += ` AND (data->>'tier')::int = $${params.length}`;
    }
    
    const result = await this.db.query(query, params);
    return result.rows.map(r => r.data);
  }

  async findById(id) {
    const result = await this.db.query(
      "SELECT data FROM services WHERE id = $1",
      [id]
    );
    return result.rows[0]?.data || null;
  }

  async getDependencyGraph() {
    const services = await this.findAll();
    
    const nodes = services.map(s => ({
      id: s.id,
      label: s.name,
      tier: s.tier,
      team: s.team
    }));
    
    const edges = services.flatMap(s => 
      (s.dependencies || []).map(dep => ({
        from: s.id,
        to: dep,
        type: 'depends-on'
      }))
    );
    
    return { nodes, edges };
  }
}

module.exports = { ServiceCatalog };
```

---

## ขั้นตอนที่ 1622: Golden Path Templates

```javascript
// golden-paths.js
// Opinionated templates สำหรับ common patterns

const fs = require('fs');
const path = require('path');
const Handlebars = require('handlebars');

class GoldenPathGenerator {
  constructor(templatesDir = './templates') {
    this.templatesDir = templatesDir;
    this.templates = new Map();
    this.loadTemplates();
  }

  loadTemplates() {
    const templateDirs = fs.readdirSync(this.templatesDir, { withFileTypes: true })
      .filter(d => d.isDirectory())
      .map(d => d.name);
    
    templateDirs.forEach(dir => {
      const manifest = JSON.parse(
        fs.readFileSync(path.join(this.templatesDir, dir, 'manifest.json'), 'utf8')
      );
      this.templates.set(dir, { ...manifest, dir });
    });
  }

  list() {
    return Array.from(this.templates.values()).map(t => ({
      id: t.dir,
      name: t.name,
      description: t.description,
      tags: t.tags,
      requirements: t.requirements
    }));
  }

  async generate(templateId, variables, outputDir) {
    const template = this.templates.get(templateId);
    if (!template) throw new Error(`Template '${templateId}' not found`);
    
    const templateDir = path.join(this.templatesDir, templateId);
    const files = this.getTemplateFiles(templateDir);
    
    for (const file of files) {
      const relativePath = file.replace(templateDir + '/', '');
      const outputPath = path.join(outputDir, this.renderString(relativePath, variables));
      
      const content = fs.readFileSync(file, 'utf8');
      const rendered = this.renderString(content, variables);
      
      fs.mkdirSync(path.dirname(outputPath), { recursive: true });
      fs.writeFileSync(outputPath, rendered);
      
      console.log(`  Generated: ${outputPath}`);
    }
    
    return outputDir;
  }

  renderString(template, variables) {
    const compiled = Handlebars.compile(template);
    return compiled(variables);
  }

  getTemplateFiles(dir) {
    const files = [];
    const walk = (d) => {
      fs.readdirSync(d, { withFileTypes: true }).forEach(entry => {
        const fullPath = path.join(d, entry.name);
        if (entry.isDirectory()) {
          walk(fullPath);
        } else if (!entry.name.endsWith('.manifest.json')) {
          files.push(fullPath);
        }
      });
    };
    walk(dir);
    return files;
  }
}

// Available Golden Path Templates
const goldenPaths = {
  'node-rest-api': {
    name: 'Node.js REST API',
    description: 'Production-ready REST API with Express',
    tags: ['nodejs', 'rest', 'api'],
    variables: ['serviceName', 'port', 'team'],
    includes: [
      'Express + TypeScript setup',
      'OpenAPI documentation',
      'JWT authentication',
      'PostgreSQL integration',
      'Redis caching',
      'Structured logging',
      'Health checks',
      'Docker + K8s configs',
      'CI/CD pipelines',
      'Unit + integration tests'
    ]
  },
  
  'node-event-consumer': {
    name: 'Kafka Event Consumer',
    description: 'Event-driven service consuming Kafka events',
    tags: ['nodejs', 'kafka', 'event-driven'],
    variables: ['serviceName', 'consumerGroup', 'topics'],
    includes: [
      'KafkaJS setup',
      'Event handler pattern',
      'Dead letter queue',
      'Idempotency',
      'Observability'
    ]
  },

  'node-grpc-service': {
    name: 'gRPC Service',
    description: 'High-performance gRPC microservice',
    tags: ['nodejs', 'grpc', 'microservice'],
    variables: ['serviceName', 'protoFile'],
    includes: [
      'gRPC server setup',
      'Protocol Buffer definitions',
      'Service reflection',
      'mTLS setup',
      'Health checking'
    ]
  }
};

module.exports = { GoldenPathGenerator, goldenPaths };
```

---

## ขั้นตอนที่ 1623: Self-Service Infrastructure

```javascript
// infrastructure-portal.js
// Self-service infrastructure provisioning

const { execSync } = require('child_process');
const yaml = require('js-yaml');
const fs = require('fs');

class InfrastructurePortal {
  constructor(options) {
    this.tfDir = options.tfDir || './terraform';
    this.k8sCluster = options.k8sCluster;
    this.gitopsRepo = options.gitopsRepo;
  }

  // Request new database
  async requestDatabase(config) {
    const {
      name,
      engine,        // postgres, mysql, redis
      tier,          // small, medium, large
      environment,   // dev, staging, prod
      requestedBy
    } = config;
    
    const dbConfig = {
      name: `${environment}-${name}`,
      engine,
      tier: this.getInstanceSize(tier, engine),
      backupEnabled: environment === 'production',
      multiAZ: environment === 'production',
      storageGB: this.getStorageSize(tier)
    };
    
    // Create Terraform config
    const tfConfig = this.generateTerraformConfig(dbConfig);
    
    const tfFile = `${this.tfDir}/databases/${dbConfig.name}.tf`;
    fs.writeFileSync(tfFile, tfConfig);
    
    // Create PR to gitops repo
    const prData = await this.createGitOpsPR({
      title: `[Infrastructure] Create database: ${dbConfig.name}`,
      branch: `infra/db-${dbConfig.name}`,
      files: [{ path: tfFile, content: tfConfig }],
      requestedBy,
      approvers: ['platform-team']
    });
    
    return {
      requestId: prData.id,
      database: dbConfig,
      prUrl: prData.url,
      estimatedProvisionTime: '10-15 minutes after approval'
    };
  }

  getInstanceSize(tier, engine) {
    const sizes = {
      postgres: { small: 'db.t3.micro', medium: 'db.t3.medium', large: 'db.r5.large' },
      mysql: { small: 'db.t3.micro', medium: 'db.t3.medium', large: 'db.r5.large' },
      redis: { small: 'cache.t3.micro', medium: 'cache.t3.medium', large: 'cache.r6g.large' }
    };
    return sizes[engine]?.[tier] || sizes[engine]?.small;
  }

  getStorageSize(tier) {
    const sizes = { small: 20, medium: 100, large: 500 };
    return sizes[tier] || 20;
  }

  generateTerraformConfig(dbConfig) {
    return `
resource "aws_db_instance" "${dbConfig.name.replace(/-/g, '_')}" {
  identifier        = "${dbConfig.name}"
  engine            = "${dbConfig.engine}"
  instance_class    = "${dbConfig.tier}"
  allocated_storage = ${dbConfig.storageGB}
  
  backup_retention_period = ${dbConfig.backupEnabled ? 7 : 0}
  multi_az                = ${dbConfig.multiAZ}
  
  tags = {
    Environment = "${dbConfig.name.split('-')[0]}"
    ManagedBy   = "IDP"
    RequestedAt = "${new Date().toISOString()}"
  }
}
`;
  }

  // Request Kubernetes namespace
  async requestNamespace(config) {
    const { team, environment, quotas } = config;
    
    const namespaceManifest = yaml.dump({
      apiVersion: 'v1',
      kind: 'Namespace',
      metadata: {
        name: `${team}-${environment}`,
        labels: {
          team,
          environment,
          'managed-by': 'idp'
        }
      }
    });
    
    const quotaManifest = yaml.dump({
      apiVersion: 'v1',
      kind: 'ResourceQuota',
      metadata: {
        name: 'default-quota',
        namespace: `${team}-${environment}`
      },
      spec: {
        hard: {
          'requests.cpu': quotas.cpu || '4',
          'requests.memory': quotas.memory || '8Gi',
          'limits.cpu': quotas.cpuLimit || '8',
          'limits.memory': quotas.memoryLimit || '16Gi',
          pods: quotas.pods || '20'
        }
      }
    });
    
    // Apply to cluster
    await this.applyK8sManifest(namespaceManifest);
    await this.applyK8sManifest(quotaManifest);
    
    return {
      namespace: `${team}-${environment}`,
      quotas,
      kubeconfig: this.generateKubeconfig(team, environment)
    };
  }

  applyK8sManifest(manifest) {
    const tmpFile = `/tmp/manifest-${Date.now()}.yaml`;
    fs.writeFileSync(tmpFile, manifest);
    
    try {
      execSync(`kubectl apply -f ${tmpFile}`);
    } finally {
      fs.unlinkSync(tmpFile);
    }
  }

  generateKubeconfig(team, namespace) {
    // Generate limited kubeconfig for team
    return `
apiVersion: v1
kind: Config
clusters:
- cluster:
    server: ${this.k8sCluster}
  name: production
contexts:
- context:
    cluster: production
    namespace: ${namespace}
    user: ${team}-user
  name: ${team}-context
current-context: ${team}-context
users:
- name: ${team}-user
  user:
    token: <GENERATED_TOKEN>
`;
  }

  async createGitOpsPR(config) {
    // Create PR in GitOps repo
    return {
      id: `PR-${Date.now()}`,
      url: `https://github.com/company/infra/pull/${Date.now()}`
    };
  }
}

// Express API
const express = require('express');
const app = express();
app.use(express.json());

const portal = new InfrastructurePortal({
  k8sCluster: 'https://k8s.company.com',
  gitopsRepo: 'git@github.com:company/infra.git'
});

app.post('/infra/database', async (req, res) => {
  const request = await portal.requestDatabase({
    ...req.body,
    requestedBy: req.user.email
  });
  res.json(request);
});

app.post('/infra/namespace', async (req, res) => {
  const namespace = await portal.requestNamespace(req.body);
  res.json(namespace);
});

module.exports = { InfrastructurePortal };
```

---

## ขั้นตอนที่ 1624-1640: Platform Observability

```javascript
// platform-observability.js
// Unified observability สำหรับ platform

const express = require('express');
const promClient = require('prom-client');

// Platform-level metrics
const serviceMetrics = {
  deploymentFrequency: new promClient.Counter({
    name: 'platform_deployments_total',
    help: 'Total number of deployments',
    labelNames: ['service', 'environment', 'status']
  }),
  
  leadTime: new promClient.Histogram({
    name: 'platform_lead_time_seconds',
    help: 'Time from commit to production deployment',
    labelNames: ['service'],
    buckets: [300, 600, 1800, 3600, 7200, 86400]
  }),
  
  changeFailureRate: new promClient.Gauge({
    name: 'platform_change_failure_rate',
    help: 'Percentage of deployments that cause failures',
    labelNames: ['service']
  }),
  
  mttr: new promClient.Histogram({
    name: 'platform_mttr_seconds',
    help: 'Mean time to recovery from failures',
    labelNames: ['service', 'severity'],
    buckets: [60, 300, 900, 1800, 3600, 7200]
  })
};

// DORA Metrics Tracker
class DORAMetricsTracker {
  constructor(db) {
    this.db = db;
  }

  // Deployment Frequency
  async recordDeployment(service, environment, status) {
    serviceMetrics.deploymentFrequency.inc({ service, environment, status });
    
    await this.db.query(
      'INSERT INTO deployments (service, environment, status, deployed_at) VALUES ($1, $2, $3, NOW())',
      [service, environment, status]
    );
  }

  // Lead Time for Changes
  async recordLeadTime(service, commitSha, deployedAt) {
    const commit = await this.getCommit(commitSha);
    if (!commit) return;
    
    const leadTimeSeconds = (new Date(deployedAt) - new Date(commit.committedAt)) / 1000;
    
    serviceMetrics.leadTime.observe({ service }, leadTimeSeconds);
    
    await this.db.query(
      'UPDATE deployments SET lead_time_seconds = $1 WHERE service = $2 AND commit_sha = $3',
      [leadTimeSeconds, service, commitSha]
    );
  }

  // Calculate DORA metrics
  async getDORAMetrics(service, days = 30) {
    const since = new Date();
    since.setDate(since.getDate() - days);
    
    const [deployments, incidents] = await Promise.all([
      this.db.query(
        'SELECT * FROM deployments WHERE service = $1 AND deployed_at > $2',
        [service, since]
      ),
      this.db.query(
        'SELECT * FROM incidents WHERE service = $1 AND created_at > $2',
        [service, since]
      )
    ]);
    
    const deploymentsData = deployments.rows;
    const incidentsData = incidents.rows;
    
    // Deployment Frequency
    const deploymentFrequency = deploymentsData.filter(d => d.status === 'success').length / days;
    
    // Lead Time for Changes
    const leadTimes = deploymentsData
      .filter(d => d.lead_time_seconds)
      .map(d => d.lead_time_seconds);
    
    const avgLeadTime = leadTimes.length > 0 
      ? leadTimes.reduce((a, b) => a + b, 0) / leadTimes.length 
      : null;
    
    // Change Failure Rate
    const failedDeployments = deploymentsData.filter(d => d.status === 'failed').length;
    const changeFailureRate = deploymentsData.length > 0
      ? (failedDeployments / deploymentsData.length) * 100
      : 0;
    
    // MTTR
    const resolvedIncidents = incidentsData.filter(i => i.resolved_at);
    const avgMTTR = resolvedIncidents.length > 0
      ? resolvedIncidents.reduce((sum, i) => {
          return sum + (new Date(i.resolved_at) - new Date(i.created_at)) / 60000;
        }, 0) / resolvedIncidents.length
      : null;
    
    return {
      service,
      period: { days, from: since.toISOString() },
      metrics: {
        deploymentFrequency: {
          value: deploymentFrequency.toFixed(2),
          unit: 'deploys/day',
          level: this.classifyDeploymentFrequency(deploymentFrequency)
        },
        leadTimeForChanges: {
          value: avgLeadTime ? `${Math.round(avgLeadTime / 3600)}h` : 'N/A',
          level: this.classifyLeadTime(avgLeadTime)
        },
        changeFailureRate: {
          value: `${changeFailureRate.toFixed(1)}%`,
          level: this.classifyChangeFailureRate(changeFailureRate)
        },
        mttr: {
          value: avgMTTR ? `${Math.round(avgMTTR)}m` : 'N/A',
          level: this.classifyMTTR(avgMTTR)
        }
      },
      doraLevel: this.calculateOverallLevel({
        deploymentFrequency,
        leadTime: avgLeadTime,
        changeFailureRate,
        mttr: avgMTTR
      })
    };
  }

  classifyDeploymentFrequency(freq) {
    if (freq >= 1) return 'Elite';
    if (freq >= 1/7) return 'High';
    if (freq >= 1/30) return 'Medium';
    return 'Low';
  }

  classifyLeadTime(seconds) {
    if (!seconds) return 'Unknown';
    const hours = seconds / 3600;
    if (hours < 1) return 'Elite';
    if (hours < 24) return 'High';
    if (hours < 168) return 'Medium';
    return 'Low';
  }

  classifyChangeFailureRate(rate) {
    if (rate < 5) return 'Elite';
    if (rate < 10) return 'High';
    if (rate < 15) return 'Medium';
    return 'Low';
  }

  classifyMTTR(minutes) {
    if (!minutes) return 'Unknown';
    if (minutes < 60) return 'Elite';
    if (minutes < 720) return 'High';
    if (minutes < 10080) return 'Medium';
    return 'Low';
  }

  calculateOverallLevel(metrics) {
    const levels = ['Elite', 'High', 'Medium', 'Low'];
    const levelValues = {
      Elite: 4, High: 3, Medium: 2, Low: 1, Unknown: 0
    };
    
    const scores = [
      levelValues[this.classifyDeploymentFrequency(metrics.deploymentFrequency)],
      levelValues[this.classifyLeadTime(metrics.leadTime)],
      levelValues[this.classifyChangeFailureRate(metrics.changeFailureRate)],
      levelValues[this.classifyMTTR(metrics.mttr)]
    ];
    
    const avgScore = scores.reduce((a, b) => a + b, 0) / scores.length;
    
    if (avgScore >= 3.5) return 'Elite';
    if (avgScore >= 2.5) return 'High';
    if (avgScore >= 1.5) return 'Medium';
    return 'Low';
  }

  async getCommit(sha) {
    const result = await this.db.query(
      'SELECT * FROM git_commits WHERE sha = $1',
      [sha]
    );
    return result.rows[0] || null;
  }
}

// Platform Dashboard API
const app = express();

app.get('/platform/dora/:service', async (req, res) => {
  const metrics = await doraTracker.getDORAMetrics(
    req.params.service,
    parseInt(req.query.days) || 30
  );
  res.json(metrics);
});

app.get('/platform/dora', async (req, res) => {
  const services = await serviceCatalog.findAll();
  
  const metricsPromises = services.map(service =>
    doraTracker.getDORAMetrics(service.id)
      .catch(err => ({ service: service.id, error: err.message }))
  );
  
  const allMetrics = await Promise.all(metricsPromises);
  
  res.json({
    timestamp: new Date().toISOString(),
    services: allMetrics
  });
});

module.exports = { DORAMetricsTracker };
```

---

## แบบฝึกหัด

### Exercise 1: Service Catalog API
สร้าง full REST API สำหรับ service catalog พร้อม search, filtering, และ dependency visualization

### Exercise 2: Golden Path CLI
เขียน CLI tool ที่ scaffold project ใหม่จาก golden path templates และ auto-register ใน service catalog

### Exercise 3: DORA Metrics Dashboard
สร้าง web dashboard แสดง DORA metrics ของ services ทั้งหมดใน organization

### คำถามทบทวน
1. Internal Developer Platform (IDP) คืออะไร และต่างจาก DevOps platform อย่างไร?
2. Golden Path template ช่วย developer อย่างไร?
3. DORA metrics 4 ตัวคืออะไร และวัดอะไร?
4. Self-service infrastructure provisioning มีข้อดีข้อเสียอะไรบ้าง?

---

*ต่อไป: Part 92 - GitOps*
