# Part 90 | ขั้นตอนที่ 1601-1620 จาก 1000+

## Developer Experience (DX) สำหรับ Node.js Teams

ในส่วนนี้เราจะเรียนรู้การสร้าง developer portals, internal tools, และ SDK generation

---

## ขั้นตอนที่ 1601: Developer Portal

```javascript
// developer-portal.js
// Internal developer portal API

const express = require('express');
const app = express();

// API Documentation ด้วย OpenAPI/Swagger
const swaggerJSDoc = require('swagger-jsdoc');
const swaggerUi = require('swagger-ui-express');

const swaggerOptions = {
  definition: {
    openapi: '3.0.0',
    info: {
      title: 'Internal API Portal',
      version: '1.0.0',
      description: 'Developer documentation for all internal APIs',
      contact: {
        name: 'Platform Team',
        email: 'platform@company.com',
        url: 'https://portal.company.com'
      }
    },
    servers: [
      { url: 'https://api.company.com/v1', description: 'Production' },
      { url: 'https://api-staging.company.com/v1', description: 'Staging' },
      { url: 'http://localhost:3000/v1', description: 'Development' }
    ],
    components: {
      securitySchemes: {
        bearerAuth: {
          type: 'http',
          scheme: 'bearer',
          bearerFormat: 'JWT'
        },
        apiKey: {
          type: 'apiKey',
          in: 'header',
          name: 'X-API-Key'
        }
      }
    }
  },
  apis: ['./routes/**/*.js', './models/**/*.js']
};

const swaggerSpec = swaggerJSDoc(swaggerOptions);

// Swagger UI
app.use('/docs', swaggerUi.serve, swaggerUi.setup(swaggerSpec, {
  customSiteTitle: 'Company API Portal',
  customCss: `
    .topbar { background: #1a1a2e; }
    .topbar-wrapper img { display: none; }
    .swagger-ui .btn.authorize { background: #4caf50; }
  `,
  swaggerOptions: {
    persistAuthorization: true,
    tagsSorter: 'alpha',
    operationsSorter: 'alpha'
  }
}));

// OpenAPI JSON endpoint
app.get('/docs/openapi.json', (req, res) => {
  res.json(swaggerSpec);
});

/**
 * @swagger
 * /api/users:
 *   get:
 *     summary: List all users
 *     tags: [Users]
 *     security:
 *       - bearerAuth: []
 *     parameters:
 *       - in: query
 *         name: page
 *         schema:
 *           type: integer
 *           default: 1
 *       - in: query
 *         name: limit
 *         schema:
 *           type: integer
 *           default: 20
 *     responses:
 *       200:
 *         description: List of users
 *         content:
 *           application/json:
 *             schema:
 *               type: object
 *               properties:
 *                 items:
 *                   type: array
 *                   items:
 *                     $ref: '#/components/schemas/User'
 *                 total:
 *                   type: integer
 *                 page:
 *                   type: integer
 */

module.exports = app;
```

---

## ขั้นตอนที่ 1602: SDK Generation

```javascript
// sdk-generator.js
// Generate TypeScript SDK จาก OpenAPI spec

const { generateApi } = require('swagger-typescript-api');
const path = require('path');
const fs = require('fs');

class SDKGenerator {
  constructor(options = {}) {
    this.outputDir = options.outputDir || './sdk';
    this.language = options.language || 'typescript';
  }

  async generateFromSpec(specPath) {
    console.log(`Generating SDK from ${specPath}...`);
    
    const { files } = await generateApi({
      name: 'ApiClient',
      output: path.resolve(this.outputDir),
      input: path.resolve(specPath),
      
      httpClientType: 'axios',
      
      generateClient: true,
      generateRouteTypes: true,
      generateResponses: true,
      
      templates: path.resolve('./templates/sdk'),
      
      hooks: {
        onCreateComponent: (component) => component,
        onCreateRequestParams: (rawType) => rawType,
        onCreateRoute: (routeData) => routeData,
        
        onParseSchema: (originalSchema, parsedSchema) => {
          // Add custom validation
          if (parsedSchema.type === 'Email') {
            parsedSchema.validationPattern = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
          }
          return parsedSchema;
        }
      }
    });
    
    // Post-process generated files
    await this.postProcess(files);
    
    // Generate package.json
    await this.generatePackageJson();
    
    console.log(`SDK generated in ${this.outputDir}`);
    return files;
  }

  async postProcess(files) {
    // Add custom utilities
    const utilsContent = `
// Auto-generated utilities
export const handleApiError = (error: any) => {
  if (error.response) {
    const { status, data } = error.response;
    throw new ApiError(status, data.message || 'API Error', data);
  }
  throw error;
};

export class ApiError extends Error {
  constructor(
    public statusCode: number,
    message: string,
    public data?: any
  ) {
    super(message);
    this.name = 'ApiError';
  }
}
`;
    
    fs.writeFileSync(
      path.join(this.outputDir, 'utils.ts'),
      utilsContent
    );
  }

  async generatePackageJson() {
    const packageJson = {
      name: '@company/api-client',
      version: '1.0.0',
      description: 'Auto-generated API client',
      main: 'dist/index.js',
      types: 'dist/index.d.ts',
      scripts: {
        build: 'tsc',
        prepublish: 'npm run build'
      },
      dependencies: {
        axios: '^1.0.0'
      },
      devDependencies: {
        typescript: '^5.0.0'
      }
    };
    
    fs.writeFileSync(
      path.join(this.outputDir, 'package.json'),
      JSON.stringify(packageJson, null, 2)
    );
  }
}

// ตัวอย่าง SDK ที่ generate แล้ว
const generatedSDKExample = `
// Generated SDK usage example

import { ApiClient } from '@company/api-client';

// Initialize client
const client = new ApiClient({
  baseURL: 'https://api.company.com/v1',
  headers: {
    'X-API-Key': process.env.API_KEY
  }
});

// Type-safe API calls
async function example() {
  // List users - fully typed
  const users = await client.users.list({ page: 1, limit: 20 });
  console.log(users.data.items); // Typed as User[]
  
  // Create user
  const newUser = await client.users.create({
    email: 'john@example.com',
    name: 'John Doe'
  });
  
  // Error handling
  try {
    await client.users.get('nonexistent-id');
  } catch (error) {
    if (error.statusCode === 404) {
      console.log('User not found');
    }
  }
}
`;

module.exports = { SDKGenerator };
```

---

## ขั้นตอนที่ 1603: Developer CLI Tools

```javascript
// dev-cli.js
// Internal developer CLI tool

const { program } = require('commander');
const chalk = require('chalk');
const inquirer = require('inquirer');
const ora = require('ora');

// CLI Tool for developers
class DeveloperCLI {
  constructor() {
    this.program = program;
    this.setupCommands();
  }

  setupCommands() {
    this.program
      .name('devtool')
      .description('Company developer toolkit')
      .version('1.0.0');

    // Environment management
    this.program
      .command('env')
      .description('Manage environments')
      .addCommand(
        program.createCommand('list')
          .description('List all environments')
          .action(() => this.listEnvironments())
      )
      .addCommand(
        program.createCommand('switch')
          .argument('<environment>', 'Environment name')
          .description('Switch active environment')
          .action((env) => this.switchEnvironment(env))
      );

    // Service management
    this.program
      .command('service')
      .description('Manage services')
      .addCommand(
        program.createCommand('list')
          .description('List running services')
          .action(() => this.listServices())
      )
      .addCommand(
        program.createCommand('restart')
          .argument('<service>', 'Service name')
          .action((service) => this.restartService(service))
      )
      .addCommand(
        program.createCommand('logs')
          .argument('<service>', 'Service name')
          .option('-f, --follow', 'Follow log output')
          .option('-n, --lines <number>', 'Number of lines', '100')
          .action((service, opts) => this.showLogs(service, opts))
      );

    // Database management
    this.program
      .command('db')
      .description('Database operations')
      .addCommand(
        program.createCommand('migrate')
          .option('--dry-run', 'Show what would change without applying')
          .action((opts) => this.runMigrations(opts))
      )
      .addCommand(
        program.createCommand('seed')
          .option('--env <environment>', 'Target environment', 'development')
          .action((opts) => this.seedDatabase(opts))
      )
      .addCommand(
        program.createCommand('reset')
          .description('Reset database (development only)')
          .action(() => this.resetDatabase())
      );

    // API testing
    this.program
      .command('api')
      .description('API testing tools')
      .addCommand(
        program.createCommand('test')
          .option('--suite <suite>', 'Test suite to run', 'all')
          .action((opts) => this.runAPITests(opts))
      )
      .addCommand(
        program.createCommand('mock')
          .option('--port <port>', 'Mock server port', '4000')
          .action((opts) => this.startMockServer(opts))
      );

    // Code generation
    this.program
      .command('generate')
      .description('Code generation')
      .addCommand(
        program.createCommand('resource')
          .argument('<name>', 'Resource name (e.g., Product)')
          .option('--with-tests', 'Generate test files')
          .action((name, opts) => this.generateResource(name, opts))
      )
      .addCommand(
        program.createCommand('migration')
          .argument('<name>', 'Migration name')
          .action((name) => this.generateMigration(name))
      );
  }

  async listEnvironments() {
    const spinner = ora('Fetching environments...').start();
    
    const environments = [
      { name: 'development', status: 'active', services: 5 },
      { name: 'staging', status: 'active', services: 12 },
      { name: 'production', status: 'active', services: 12 }
    ];
    
    spinner.stop();
    
    console.log(chalk.bold('\nAvailable Environments\n'));
    environments.forEach(env => {
      const statusColor = env.status === 'active' ? chalk.green : chalk.red;
      console.log(`  ${chalk.cyan(env.name.padEnd(15))} ${statusColor(env.status)} (${env.services} services)`);
    });
  }

  async generateResource(name, options) {
    const lowerName = name.toLowerCase();
    const upperName = name.charAt(0).toUpperCase() + name.slice(1);
    
    const spinner = ora(`Generating ${name} resource...`).start();
    
    const files = [
      {
        path: `src/models/${lowerName}.model.ts`,
        content: this.generateModelTemplate(upperName, lowerName)
      },
      {
        path: `src/services/${lowerName}.service.ts`,
        content: this.generateServiceTemplate(upperName, lowerName)
      },
      {
        path: `src/routes/${lowerName}.routes.ts`,
        content: this.generateRoutesTemplate(upperName, lowerName)
      }
    ];
    
    if (options.withTests) {
      files.push({
        path: `tests/${lowerName}.test.ts`,
        content: this.generateTestTemplate(upperName, lowerName)
      });
    }
    
    const fs = require('fs');
    const path = require('path');
    
    for (const file of files) {
      const dir = path.dirname(file.path);
      fs.mkdirSync(dir, { recursive: true });
      fs.writeFileSync(file.path, file.content);
    }
    
    spinner.succeed(`Generated ${name} resource`);
    
    files.forEach(f => console.log(chalk.green(`  ✓ ${f.path}`)));
  }

  generateModelTemplate(upperName, lowerName) {
    return `// ${upperName} Model
export interface ${upperName} {
  id: string;
  createdAt: Date;
  updatedAt: Date;
}

export interface Create${upperName}Dto {
  // Add fields
}

export interface Update${upperName}Dto {
  // Add fields
}
`;
  }

  generateServiceTemplate(upperName, lowerName) {
    return `import { Injectable } from '../decorators';
import { ${upperName}, Create${upperName}Dto, Update${upperName}Dto } from '../models/${lowerName}.model';
import { db } from '../database';

@Injectable()
export class ${upperName}Service {
  async findAll(page = 1, limit = 20): Promise<{ items: ${upperName}[]; total: number }> {
    const offset = (page - 1) * limit;
    const result = await db.query(
      'SELECT * FROM ${lowerName}s ORDER BY created_at DESC LIMIT $1 OFFSET $2',
      [limit, offset]
    );
    const count = await db.query('SELECT COUNT(*) FROM ${lowerName}s');
    
    return {
      items: result.rows,
      total: parseInt(count.rows[0].count)
    };
  }

  async findById(id: string): Promise<${upperName} | null> {
    const result = await db.query('SELECT * FROM ${lowerName}s WHERE id = $1', [id]);
    return result.rows[0] || null;
  }

  async create(data: Create${upperName}Dto): Promise<${upperName}> {
    const result = await db.query(
      'INSERT INTO ${lowerName}s (id, created_at) VALUES (gen_random_uuid(), NOW()) RETURNING *'
    );
    return result.rows[0];
  }

  async update(id: string, data: Update${upperName}Dto): Promise<${upperName} | null> {
    const result = await db.query(
      'UPDATE ${lowerName}s SET updated_at = NOW() WHERE id = $1 RETURNING *',
      [id]
    );
    return result.rows[0] || null;
  }

  async delete(id: string): Promise<boolean> {
    const result = await db.query('DELETE FROM ${lowerName}s WHERE id = $1', [id]);
    return result.rowCount > 0;
  }
}
`;
  }

  generateRoutesTemplate(upperName, lowerName) {
    return `import { Router } from 'express';
import { ${upperName}Service } from '../services/${lowerName}.service';
import { authenticate } from '../middleware/auth';

const router = Router();
const service = new ${upperName}Service();

/**
 * @swagger
 * /api/${lowerName}s:
 *   get:
 *     summary: List ${lowerName}s
 *     tags: [${upperName}s]
 */
router.get('/', authenticate, async (req, res) => {
  const result = await service.findAll(
    parseInt(req.query.page as string) || 1,
    parseInt(req.query.limit as string) || 20
  );
  res.json(result);
});

router.get('/:id', authenticate, async (req, res) => {
  const item = await service.findById(req.params.id);
  if (!item) return res.status(404).json({ error: '${upperName} not found' });
  res.json(item);
});

router.post('/', authenticate, async (req, res) => {
  const item = await service.create(req.body);
  res.status(201).json(item);
});

router.put('/:id', authenticate, async (req, res) => {
  const item = await service.update(req.params.id, req.body);
  if (!item) return res.status(404).json({ error: '${upperName} not found' });
  res.json(item);
});

router.delete('/:id', authenticate, async (req, res) => {
  const deleted = await service.delete(req.params.id);
  if (!deleted) return res.status(404).json({ error: '${upperName} not found' });
  res.status(204).send();
});

export default router;
`;
  }

  generateTestTemplate(upperName, lowerName) {
    return `import { describe, it, expect, beforeEach, afterEach } from 'vitest';
import request from 'supertest';
import app from '../src/app';
import { db } from '../src/database';

describe('${upperName} API', () => {
  let authToken: string;

  beforeEach(async () => {
    // Setup test data
    await db.query('BEGIN');
    authToken = await getTestToken();
  });

  afterEach(async () => {
    await db.query('ROLLBACK');
  });

  describe('GET /api/${lowerName}s', () => {
    it('should return list of ${lowerName}s', async () => {
      const response = await request(app)
        .get('/api/${lowerName}s')
        .set('Authorization', \`Bearer \${authToken}\`)
        .expect(200);

      expect(response.body).toHaveProperty('items');
      expect(response.body).toHaveProperty('total');
    });
  });

  describe('POST /api/${lowerName}s', () => {
    it('should create a new ${lowerName}', async () => {
      const response = await request(app)
        .post('/api/${lowerName}s')
        .set('Authorization', \`Bearer \${authToken}\`)
        .send({})
        .expect(201);

      expect(response.body).toHaveProperty('id');
    });
  });
});

async function getTestToken(): Promise<string> {
  // Get test JWT token
  return 'test-token';
}
`;
  }

  async runMigrations({ dryRun }) {
    const spinner = ora('Running migrations...').start();
    
    if (dryRun) {
      spinner.info('Dry run mode - no changes will be applied');
    }
    
    // Simulate migration
    await new Promise(resolve => setTimeout(resolve, 1000));
    
    spinner.succeed('Migrations completed');
  }

  async showLogs(service, { follow, lines }) {
    console.log(chalk.cyan(`Showing last ${lines} lines of ${service}...`));
    // Implementation
  }
}

// Run CLI
const cli = new DeveloperCLI();
cli.program.parse();

module.exports = { DeveloperCLI };
```

---

## ขั้นตอนที่ 1604-1620: Local Development Environment

```javascript
// dev-environment.js
// Docker Compose setup สำหรับ local development

const devEnvironmentConfig = {
  services: {
    postgres: {
      image: 'postgres:15-alpine',
      environment: {
        POSTGRES_DB: 'myapp_dev',
        POSTGRES_USER: 'dev',
        POSTGRES_PASSWORD: 'dev_password'
      },
      ports: ['5432:5432'],
      volumes: ['postgres_data:/var/lib/postgresql/data']
    },
    redis: {
      image: 'redis:7-alpine',
      ports: ['6379:6379'],
      command: 'redis-server --save 20 1 --loglevel warning'
    },
    kafka: {
      image: 'confluentinc/cp-kafka:7.4.0',
      ports: ['9092:9092'],
      environment: {
        KAFKA_BROKER_ID: 1,
        KAFKA_ZOOKEEPER_CONNECT: 'zookeeper:2181',
        KAFKA_ADVERTISED_LISTENERS: 'PLAINTEXT://localhost:9092'
      }
    },
    mailhog: {
      image: 'mailhog/mailhog',
      ports: ['1025:1025', '8025:8025']
    }
  }
};

// Development setup script
const setupDevelopmentEnvironment = `
#!/bin/bash
# Development environment setup

echo "Setting up development environment..."

# Check dependencies
check_dependency() {
  if ! command -v $1 &> /dev/null; then
    echo "Error: $1 is not installed"
    exit 1
  fi
}

check_dependency docker
check_dependency node
check_dependency npm

# Start services
echo "Starting services..."
docker compose up -d

# Wait for services to be ready
echo "Waiting for services..."
sleep 5

# Check database
until docker exec myapp-postgres pg_isready -U dev; do
  echo "Waiting for PostgreSQL..."
  sleep 2
done

# Run migrations
echo "Running database migrations..."
npm run db:migrate

# Seed development data
echo "Seeding development data..."
npm run db:seed

# Install dependencies
echo "Installing dependencies..."
npm install

echo "Setup complete! Run 'npm run dev' to start development server"
`;

// Hot reload setup
const hotReloadConfig = `
// nodemon.json
{
  "watch": ["src"],
  "ext": "ts,js,json",
  "ignore": ["src/**/*.test.ts", "node_modules"],
  "exec": "ts-node src/index.ts",
  "env": {
    "NODE_ENV": "development"
  },
  "delay": 500
}
`;

module.exports = { devEnvironmentConfig, setupDevelopmentEnvironment };
```

---

## แบบฝึกหัด

### Exercise 1: API Documentation Generator
สร้าง script ที่ generate Swagger documentation โดยอัตโนมัติจาก TypeScript types

### Exercise 2: Developer CLI Tool
สร้าง CLI tool ที่ช่วย developer ทำ common tasks เช่น create PR, check service status, run tests

### Exercise 3: SDK Test Suite
สร้าง automated test suite ที่ verify SDK ทำงานถูกต้อง และรัน ทุกครั้งที่ API spec เปลี่ยน

### คำถามทบทวน
1. Developer Experience ส่งผลต่อ productivity ของทีมอย่างไร?
2. ควรมี documentation อะไรบ้างใน developer portal?
3. SDK generation ช่วย developer อย่างไรเมื่อเทียบกับ raw HTTP calls?
4. ทำไม local development environment ควรเหมือน production มากที่สุด?

---

*ต่อไป: Part 91 - Platform Engineering*
