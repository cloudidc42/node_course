# Part 92 | ขั้นตอนที่ 1641-1660 จาก 1000+

## GitOps Principles และ Infrastructure as Code

ในส่วนนี้เราจะเรียนรู้ GitOps, ArgoCD, Flux, และ Infrastructure as Code สำหรับ Node.js applications

---

## ขั้นตอนที่ 1641: GitOps Fundamentals

GitOps คือ methodology ที่ใช้ Git เป็น single source of truth สำหรับทั้ง application code และ infrastructure configuration

**4 หลักการของ GitOps:**
1. **Declarative** - Describe desired state, not imperative steps
2. **Versioned and Immutable** - Git is the single source of truth
3. **Pulled Automatically** - Changes are automatically applied
4. **Continuously Reconciled** - Actual state matches desired state

```yaml
# gitops-structure/
# ├── apps/
# │   ├── node-api/
# │   │   ├── deployment.yaml
# │   │   ├── service.yaml
# │   │   ├── ingress.yaml
# │   │   └── kustomization.yaml
# │   └── worker/
# │       └── ...
# ├── infrastructure/
# │   ├── databases/
# │   ├── messaging/
# │   └── monitoring/
# └── environments/
#     ├── development/
#     ├── staging/
#     └── production/

# apps/node-api/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: node-api
  namespace: production
  labels:
    app: node-api
    version: "1.0.0"
spec:
  replicas: 3
  selector:
    matchLabels:
      app: node-api
  template:
    metadata:
      labels:
        app: node-api
        version: "1.0.0"
    spec:
      containers:
      - name: node-api
        image: registry.company.com/node-api:1.0.0
        ports:
        - containerPort: 3000
        env:
        - name: NODE_ENV
          value: production
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: node-api-secrets
              key: database-url
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
          limits:
            cpu: "500m"
            memory: "512Mi"
        livenessProbe:
          httpGet:
            path: /health/live
            port: 3000
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /health/ready
            port: 3000
          initialDelaySeconds: 5
          periodSeconds: 5
```

---

## ขั้นตอนที่ 1642: ArgoCD Setup

```yaml
# argocd/application.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: node-api
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  
  source:
    repoURL: https://github.com/company/gitops-config
    targetRevision: main
    path: apps/node-api
    
    # Kustomize overlay
    kustomize:
      images:
        - registry.company.com/node-api=registry.company.com/node-api:1.2.3
  
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  
  syncPolicy:
    automated:
      prune: true         # Remove resources not in git
      selfHeal: true      # Fix manual changes
      allowEmpty: false
    
    syncOptions:
      - Validate=true
      - CreateNamespace=false
      - PrunePropagationPolicy=foreground
      - RespectIgnoreDifferences=true
    
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
  
  # Health check override
  ignoreDifferences:
    - group: apps
      kind: Deployment
      jsonPointers:
        - /spec/replicas  # Ignore HPA-managed replicas

---
# argocd/app-project.yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: production-apps
  namespace: argocd
spec:
  description: Production applications
  
  sourceRepos:
    - https://github.com/company/gitops-config
  
  destinations:
    - namespace: production
      server: https://kubernetes.default.svc
    - namespace: production-*
      server: https://kubernetes.default.svc
  
  clusterResourceWhitelist:
    - group: ''
      kind: Namespace
  
  namespaceResourceWhitelist:
    - group: apps
      kind: Deployment
    - group: ''
      kind: Service
    - group: networking.k8s.io
      kind: Ingress
  
  roles:
    - name: developer
      description: Developer access
      policies:
        - p, proj:production-apps:developer, applications, get, production-apps/*, allow
        - p, proj:production-apps:developer, applications, sync, production-apps/*, allow
      groups:
        - developers
```

---

## ขั้นตอนที่ 1643: Automated GitOps Deployment

```javascript
// gitops-deployer.js
// Automated GitOps deployment pipeline

const { Octokit } = require('@octokit/rest');
const yaml = require('js-yaml');
const crypto = require('crypto');

class GitOpsDeployer {
  constructor(config) {
    this.octokit = new Octokit({ auth: config.githubToken });
    this.owner = config.owner;
    this.repo = config.gitopsRepo;
    this.defaultBranch = config.defaultBranch || 'main';
  }

  async deployToEnvironment(service, version, environment) {
    const branchName = `deploy/${service}-${version}-${environment}-${Date.now()}`;
    
    try {
      // 1. Get current state
      const currentFile = await this.getDeploymentFile(service, environment);
      
      // 2. Update image version
      const updatedContent = this.updateImageVersion(
        currentFile.content,
        service,
        version
      );
      
      // 3. Create branch
      await this.createBranch(branchName);
      
      // 4. Commit changes
      await this.commitChanges(
        branchName,
        `apps/${environment}/${service}/deployment.yaml`,
        updatedContent,
        `deploy: ${service} v${version} to ${environment}`,
        currentFile.sha
      );
      
      // 5. Create PR (for production) or auto-merge (for dev/staging)
      if (environment === 'production') {
        const pr = await this.createPR(branchName, service, version, environment);
        return { type: 'pr', url: pr.data.html_url };
      } else {
        await this.mergePR(branchName, service, version);
        return { type: 'auto-merged', branch: branchName };
      }
      
    } catch (err) {
      throw new Error(`GitOps deploy failed: ${err.message}`);
    }
  }

  async getDeploymentFile(service, environment) {
    const filePath = `apps/${environment}/${service}/deployment.yaml`;
    
    const { data } = await this.octokit.repos.getContent({
      owner: this.owner,
      repo: this.repo,
      path: filePath
    });
    
    return {
      content: Buffer.from(data.content, 'base64').toString('utf8'),
      sha: data.sha
    };
  }

  updateImageVersion(yamlContent, service, version) {
    const doc = yaml.load(yamlContent);
    
    // Update image in deployment
    if (doc.spec?.template?.spec?.containers) {
      doc.spec.template.spec.containers = doc.spec.template.spec.containers.map(container => {
        if (container.name === service) {
          const imageName = container.image.split(':')[0];
          container.image = `${imageName}:${version}`;
        }
        return container;
      });
    }
    
    // Update version label
    if (doc.metadata?.labels) {
      doc.metadata.labels.version = version;
    }
    if (doc.spec?.template?.metadata?.labels) {
      doc.spec.template.metadata.labels.version = version;
    }
    
    return yaml.dump(doc);
  }

  async createBranch(branchName) {
    // Get main branch SHA
    const { data: mainRef } = await this.octokit.git.getRef({
      owner: this.owner,
      repo: this.repo,
      ref: `heads/${this.defaultBranch}`
    });
    
    await this.octokit.git.createRef({
      owner: this.owner,
      repo: this.repo,
      ref: `refs/heads/${branchName}`,
      sha: mainRef.object.sha
    });
  }

  async commitChanges(branch, filePath, content, message, sha) {
    await this.octokit.repos.createOrUpdateFileContents({
      owner: this.owner,
      repo: this.repo,
      path: filePath,
      message,
      content: Buffer.from(content).toString('base64'),
      branch,
      sha
    });
  }

  async createPR(branch, service, version, environment) {
    return this.octokit.pulls.create({
      owner: this.owner,
      repo: this.repo,
      title: `[Deploy] ${service} v${version} → ${environment}`,
      head: branch,
      base: this.defaultBranch,
      body: this.generatePRDescription(service, version, environment)
    });
  }

  generatePRDescription(service, version, environment) {
    return `
## Deployment Request

**Service:** ${service}
**Version:** ${version}
**Target Environment:** ${environment}
**Requested by:** CI/CD Pipeline

## Checklist
- [ ] Tests passing
- [ ] Staging deployment verified
- [ ] Rollback plan documented

## What changed
See the diff below for configuration changes.

---
*Auto-generated by GitOps Deployer*
`;
  }

  async mergePR(branch, service, version) {
    const { data: pr } = await this.octokit.pulls.create({
      owner: this.owner,
      repo: this.repo,
      title: `[Deploy] ${service} v${version} (auto)`,
      head: branch,
      base: this.defaultBranch
    });
    
    await this.octokit.pulls.merge({
      owner: this.owner,
      repo: this.repo,
      pull_number: pr.number,
      merge_method: 'squash'
    });
    
    // Clean up branch
    await this.octokit.git.deleteRef({
      owner: this.owner,
      repo: this.repo,
      ref: `heads/${branch}`
    });
  }
}

// Deployment webhook handler
const express = require('express');
const app = express();
app.use(express.json());

const deployer = new GitOpsDeployer({
  githubToken: process.env.GITHUB_TOKEN,
  owner: 'company',
  gitopsRepo: 'gitops-config'
});

// CI/CD pipeline webhook
app.post('/deploy', async (req, res) => {
  const { service, version, environment, triggeredBy } = req.body;
  
  console.log(`Deploying ${service}:${version} to ${environment}`);
  
  try {
    const result = await deployer.deployToEnvironment(service, version, environment);
    
    res.json({
      success: true,
      deployment: { service, version, environment },
      result
    });
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
});

module.exports = { GitOpsDeployer };
```

---

## ขั้นตอนที่ 1644: Kustomize Overlays

```yaml
# kustomization base: apps/node-api/base/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - deployment.yaml
  - service.yaml
  - hpa.yaml

commonLabels:
  app: node-api
  managed-by: kustomize

---
# Staging overlay: apps/node-api/overlays/staging/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

bases:
  - ../../base

namespace: staging

patches:
  - target:
      kind: Deployment
      name: node-api
    patch: |-
      - op: replace
        path: /spec/replicas
        value: 1
      - op: replace
        path: /spec/template/spec/containers/0/resources/requests/cpu
        value: "50m"
      - op: replace
        path: /spec/template/spec/containers/0/resources/requests/memory
        value: "64Mi"

images:
  - name: registry.company.com/node-api
    newTag: latest

configMapGenerator:
  - name: node-api-config
    literals:
      - NODE_ENV=staging
      - LOG_LEVEL=debug

---
# Production overlay: apps/node-api/overlays/production/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

bases:
  - ../../base

namespace: production

patches:
  - target:
      kind: Deployment
      name: node-api
    patch: |-
      - op: replace
        path: /spec/replicas
        value: 3

images:
  - name: registry.company.com/node-api
    newTag: 1.2.3  # Pinned version in production!

configMapGenerator:
  - name: node-api-config
    literals:
      - NODE_ENV=production
      - LOG_LEVEL=info
```

---

## ขั้นตอนที่ 1645-1660: Advanced GitOps Patterns

### Image Update Automation

```javascript
// image-updater.js
// Automatically update image tags in GitOps repo

const { GitOpsDeployer } = require('./gitops-deployer');
const semver = require('semver');

class ImageUpdateAutomation {
  constructor(config) {
    this.deployer = new GitOpsDeployer(config);
    this.registry = config.registry;
    this.updatePolicies = config.policies || {};
  }

  // Check registry for new images
  async checkForUpdates(service) {
    const tags = await this.registry.getTags(service);
    const policy = this.updatePolicies[service] || 'semver';
    
    return this.applyPolicy(tags, policy);
  }

  applyPolicy(tags, policy) {
    switch (policy) {
      case 'semver':
        // Latest stable semver version
        const semverTags = tags.filter(t => semver.valid(t));
        return semver.maxSatisfying(semverTags, '*');
      
      case 'latest':
        return tags.find(t => t === 'latest');
      
      case 'semver-patch':
        // Only auto-update patch versions
        return semver.maxSatisfying(tags.filter(t => semver.valid(t)), '~x.x.x');
      
      default:
        return null;
    }
  }

  async runUpdateCycle() {
    const services = await this.getServicesWithAutoUpdate();
    
    for (const service of services) {
      const latestTag = await this.checkForUpdates(service.name);
      
      if (!latestTag || latestTag === service.currentTag) continue;
      
      console.log(`New version available for ${service.name}: ${service.currentTag} -> ${latestTag}`);
      
      // Deploy to dev automatically
      await this.deployer.deployToEnvironment(service.name, latestTag, 'development');
      
      // Notify team
      await this.notifyUpdate(service, latestTag);
    }
  }

  async getServicesWithAutoUpdate() {
    return [
      { name: 'node-api', currentTag: '1.2.2', autoUpdate: true },
    ];
  }

  async notifyUpdate(service, newTag) {
    console.log(`Notifying team about ${service.name} update to ${newTag}`);
  }
}

// Flux - alternative to ArgoCD
const fluxConfig = `
# flux/kustomize-controller.yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1beta2
kind: Kustomization
metadata:
  name: node-api
  namespace: flux-system
spec:
  interval: 5m
  path: ./apps/node-api/overlays/production
  prune: true
  sourceRef:
    kind: GitRepository
    name: gitops-config
  healthChecks:
    - apiVersion: apps/v1
      kind: Deployment
      name: node-api
      namespace: production

---
# flux/image-policy.yaml
apiVersion: image.toolkit.fluxcd.io/v1beta1
kind: ImagePolicy
metadata:
  name: node-api
  namespace: flux-system
spec:
  imageRepositoryRef:
    name: node-api
  policy:
    semver:
      range: '>=1.0.0 <2.0.0'

---
# flux/image-update-automation.yaml
apiVersion: image.toolkit.fluxcd.io/v1beta1
kind: ImageUpdateAutomation
metadata:
  name: node-api
  namespace: flux-system
spec:
  interval: 5m
  sourceRef:
    kind: GitRepository
    name: gitops-config
  git:
    checkout:
      ref:
        branch: main
    commit:
      author:
        email: fluxcdbot@users.noreply.github.com
        name: fluxcdbot
      messageTemplate: 'chore: update image to {{range .Updated.Images}}{{println .}}{{end}}'
    push:
      branch: main
  update:
    path: ./apps
    strategy: Setters
`;

module.exports = { ImageUpdateAutomation };
```

---

## แบบฝึกหัด

### Exercise 1: GitOps Pipeline
สร้าง GitHub Actions workflow ที่ auto-deploy ไปยัง GitOps repo เมื่อ Docker image build สำเร็จ

### Exercise 2: Multi-Environment Kustomize
สร้าง Kustomize overlays สำหรับ dev, staging, production ที่มี config ต่างกัน

### Exercise 3: Deployment Rollback
สร้าง script ที่ rollback deployment ใน GitOps repo โดยการ revert commit และ trigger re-sync

### คำถามทบทวน
1. GitOps ต่างจาก traditional CI/CD อย่างไร?
2. Declarative vs Imperative configuration คืออะไร?
3. ArgoCD vs Flux มีข้อดีข้อเสียต่างกันอย่างไร?
4. Image Update Automation มีความเสี่ยงอะไร และป้องกันได้อย่างไร?

---

*ต่อไป: Part 93 - Multi-Cloud Strategies*
