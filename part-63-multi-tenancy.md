# Part 63: Multi-tenancy
## ขั้นตอนที่ 621-630 จาก 1000

---

## Multi-tenancy คืออะไร?

Multi-tenancy คือ software architecture ที่ application เดียวให้บริการแก่ลูกค้าหลายราย (tenants) โดยแต่ละ tenant ใช้งาน instance เดียวกัน แต่ข้อมูลของแต่ละ tenant ถูกแยกออกจากกัน

---

## 1. Multi-tenant Patterns

### Pattern 1: Database per Tenant

```
Tenant A → Database A
Tenant B → Database B
Tenant C → Database C
```

ข้อดี:
- ความเป็น privacy/isolation สูงสุด
- ง่ายต่อการ backup/restore แยกราย tenant
- Performance ไม่กระทบกัน

ข้อเสีย:
- ค่าใช้จ่ายสูง
- ยากต่อการบริหารจัดการ
- Migration ซับซ้อน

### Pattern 2: Schema per Tenant

```
Single Database:
  schema_tenant_a → tables
  schema_tenant_b → tables
  schema_tenant_c → tables
```

ข้อดี:
- สมดุลระหว่าง isolation และ cost
- ใช้ทรัพยากรร่วมกัน

ข้อเสีย:
- ยาก scale ถ้า tenants มาก
- Cross-tenant queries ซับซ้อน

### Pattern 3: Row-level Security (Shared Database)

```
Single Database, Single Schema:
  users table: { id, tenant_id, name, ... }
  orders table: { id, tenant_id, ... }
```

ข้อดี:
- ใช้ทรัพยากรน้อยที่สุด
- ง่ายต่อการ maintenance
- Cross-tenant analytics ง่าย

ข้อเสีย:
- ต้องระวัง data leakage มาก
- Performance อาจมีปัญหาถ้า tenants มาก

---

## 2. Database per Tenant

### Connection Manager

```javascript
// services/database-manager.js
const mongoose = require('mongoose');

class DatabaseManager {
  constructor() {
    this.connections = new Map();
  }

  async getConnection(tenantId) {
    if (this.connections.has(tenantId)) {
      const conn = this.connections.get(tenantId);
      if (conn.readyState === 1) { // Connected
        return conn;
      }
    }

    const dbUri = this.buildConnectionString(tenantId);
    
    const connection = await mongoose.createConnection(dbUri, {
      maxPoolSize: 10,
      minPoolSize: 2,
      serverSelectionTimeoutMS: 5000,
      socketTimeoutMS: 45000
    });

    // ลงทะเบียน models ใน connection
    this.registerModels(connection);
    
    this.connections.set(tenantId, connection);
    
    connection.on('error', (err) => {
      console.error(`DB connection error for tenant ${tenantId}:`, err);
      this.connections.delete(tenantId);
    });

    connection.on('disconnected', () => {
      console.log(`DB disconnected for tenant ${tenantId}`);
      this.connections.delete(tenantId);
    });

    return connection;
  }

  buildConnectionString(tenantId) {
    const baseUri = process.env.MONGODB_BASE_URI; // mongodb://localhost:27017
    return `${baseUri}/tenant_${tenantId}`;
  }

  registerModels(connection) {
    // Register schemas ทุกตัวกับ connection นี้
    const UserSchema = require('../models/schemas/user.schema');
    const OrderSchema = require('../models/schemas/order.schema');
    const ProductSchema = require('../models/schemas/product.schema');

    connection.model('User', UserSchema);
    connection.model('Order', OrderSchema);
    connection.model('Product', ProductSchema);
  }

  async closeConnection(tenantId) {
    const conn = this.connections.get(tenantId);
    if (conn) {
      await conn.close();
      this.connections.delete(tenantId);
    }
  }

  async closeAll() {
    const promises = Array.from(this.connections.keys())
      .map(tenantId => this.closeConnection(tenantId));
    await Promise.all(promises);
  }

  getActiveConnections() {
    return this.connections.size;
  }
}

module.exports = new DatabaseManager();
```

### Tenant Middleware

```javascript
// middleware/tenant.js
const Tenant = require('../models/Tenant');
const dbManager = require('../services/database-manager');

async function tenantMiddleware(req, res, next) {
  try {
    // หา tenant จาก subdomain หรือ header
    const tenantId = getTenantId(req);
    
    if (!tenantId) {
      return res.status(400).json({ error: 'Tenant not specified' });
    }

    // ตรวจสอบว่า tenant มีอยู่
    const tenant = await Tenant.findOne({ 
      slug: tenantId, 
      status: 'active' 
    });
    
    if (!tenant) {
      return res.status(404).json({ error: 'Tenant not found' });
    }

    // เชื่อมต่อกับ database ของ tenant
    const db = await dbManager.getConnection(tenant.id);
    
    req.tenant = tenant;
    req.db = db;
    
    next();
  } catch (error) {
    next(error);
  }
}

function getTenantId(req) {
  // Option 1: จาก subdomain (acme.myapp.com → acme)
  const host = req.hostname;
  const parts = host.split('.');
  if (parts.length > 2) {
    return parts[0];
  }
  
  // Option 2: จาก header
  return req.headers['x-tenant-id'];
  
  // Option 3: จาก JWT token
  // return req.user?.tenantId;
}

module.exports = tenantMiddleware;
```

---

## 3. Schema per Tenant (PostgreSQL)

### PostgreSQL Schema Management

```javascript
// services/schema-manager.js
const { Pool } = require('pg');

class SchemaManager {
  constructor() {
    this.pool = new Pool({
      connectionString: process.env.DATABASE_URL
    });
  }

  async createTenantSchema(tenantId) {
    const schemaName = `tenant_${tenantId}`;
    
    const client = await this.pool.connect();
    
    try {
      await client.query('BEGIN');
      
      // สร้าง schema
      await client.query(`CREATE SCHEMA IF NOT EXISTS "${schemaName}"`);
      
      // สร้าง tables ใน schema
      await this.createTables(client, schemaName);
      
      await client.query('COMMIT');
      console.log(`Schema created: ${schemaName}`);
    } catch (error) {
      await client.query('ROLLBACK');
      throw error;
    } finally {
      client.release();
    }
  }

  async createTables(client, schemaName) {
    // สร้าง tables ทั้งหมดที่ต้องการ
    await client.query(`
      CREATE TABLE IF NOT EXISTS "${schemaName}".users (
        id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
        email VARCHAR(255) UNIQUE NOT NULL,
        name VARCHAR(255) NOT NULL,
        role VARCHAR(50) DEFAULT 'user',
        created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
        updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
      )
    `);

    await client.query(`
      CREATE TABLE IF NOT EXISTS "${schemaName}".products (
        id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
        name VARCHAR(255) NOT NULL,
        price DECIMAL(10,2) NOT NULL,
        stock INTEGER DEFAULT 0,
        created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
      )
    `);

    await client.query(`
      CREATE TABLE IF NOT EXISTS "${schemaName}".orders (
        id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
        user_id UUID REFERENCES "${schemaName}".users(id),
        total DECIMAL(10,2) NOT NULL,
        status VARCHAR(50) DEFAULT 'pending',
        created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
      )
    `);

    // สร้าง indexes
    await client.query(`
      CREATE INDEX IF NOT EXISTS idx_${schemaName.replace('-', '_')}_users_email 
      ON "${schemaName}".users(email)
    `);
  }

  async deleteTenantSchema(tenantId) {
    const schemaName = `tenant_${tenantId}`;
    await this.pool.query(`DROP SCHEMA IF EXISTS "${schemaName}" CASCADE`);
  }

  getSchemaName(tenantId) {
    return `tenant_${tenantId}`;
  }
}

module.exports = new SchemaManager();
```

### Tenant-aware Query Builder

```javascript
// services/tenant-db.js
const { Pool } = require('pg');
const schemaManager = require('./schema-manager');

class TenantDB {
  constructor(tenantId) {
    this.tenantId = tenantId;
    this.schema = schemaManager.getSchemaName(tenantId);
    this.pool = new Pool({
      connectionString: process.env.DATABASE_URL
    });
  }

  async query(text, params = []) {
    // Set search_path ไปยัง tenant schema
    const client = await this.pool.connect();
    try {
      await client.query(`SET search_path TO "${this.schema}", public`);
      const result = await client.query(text, params);
      return result;
    } finally {
      client.release();
    }
  }

  // CRUD methods
  async findAll(table, conditions = {}, options = {}) {
    const { limit = 50, offset = 0, orderBy = 'created_at DESC' } = options;
    
    const keys = Object.keys(conditions);
    const values = Object.values(conditions);
    
    let whereClause = '';
    if (keys.length > 0) {
      whereClause = 'WHERE ' + keys.map((k, i) => `${k} = $${i + 1}`).join(' AND ');
    }

    const result = await this.query(
      `SELECT * FROM ${table} ${whereClause} ORDER BY ${orderBy} LIMIT $${keys.length + 1} OFFSET $${keys.length + 2}`,
      [...values, limit, offset]
    );
    
    return result.rows;
  }

  async findById(table, id) {
    const result = await this.query(
      `SELECT * FROM ${table} WHERE id = $1`,
      [id]
    );
    return result.rows[0] || null;
  }

  async create(table, data) {
    const keys = Object.keys(data);
    const values = Object.values(data);
    const placeholders = keys.map((_, i) => `$${i + 1}`).join(', ');
    
    const result = await this.query(
      `INSERT INTO ${table} (${keys.join(', ')}) VALUES (${placeholders}) RETURNING *`,
      values
    );
    
    return result.rows[0];
  }

  async update(table, id, data) {
    const keys = Object.keys(data);
    const values = Object.values(data);
    const setClause = keys.map((k, i) => `${k} = $${i + 1}`).join(', ');
    
    const result = await this.query(
      `UPDATE ${table} SET ${setClause}, updated_at = NOW() WHERE id = $${keys.length + 1} RETURNING *`,
      [...values, id]
    );
    
    return result.rows[0];
  }

  async delete(table, id) {
    await this.query(`DELETE FROM ${table} WHERE id = $1`, [id]);
  }
}

module.exports = TenantDB;
```

---

## 4. Row-level Security

### RLS Middleware

```javascript
// middleware/rls.js
// Row-Level Security ผ่าน application layer

function createTenantScope(tenantId) {
  return {
    // MongoDB
    mongoScope: { tenantId },
    
    // SQL where clause
    sqlScope: { tenant_id: tenantId },
    
    // Sequelize scope
    sequelizeScope: { where: { tenantId } }
  };
}

// MongoDB middleware
function mongoTenantPlugin(schema) {
  // เพิ่ม tenantId field
  schema.add({ tenantId: { type: String, required: true, index: true } });

  // Intercept ทุก query เพื่อเพิ่ม tenantId filter
  schema.pre('find', function() {
    if (this._tenantId) {
      this.where({ tenantId: this._tenantId });
    }
  });

  schema.pre('findOne', function() {
    if (this._tenantId) {
      this.where({ tenantId: this._tenantId });
    }
  });

  schema.pre('countDocuments', function() {
    if (this._tenantId) {
      this.where({ tenantId: this._tenantId });
    }
  });

  // เพิ่ม tenantId โดยอัตโนมัติเมื่อ save
  schema.pre('save', function() {
    if (!this.tenantId && this._tenantId) {
      this.tenantId = this._tenantId;
    }
  });
}

module.exports = { createTenantScope, mongoTenantPlugin };
```

### PostgreSQL Row-Level Security

```sql
-- เปิด RLS บน table
ALTER TABLE users ENABLE ROW LEVEL SECURITY;
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;
ALTER TABLE products ENABLE ROW LEVEL SECURITY;

-- สร้าง policy
CREATE POLICY tenant_isolation ON users
  USING (tenant_id = current_setting('app.tenant_id')::uuid);

CREATE POLICY tenant_isolation ON orders
  USING (tenant_id = current_setting('app.tenant_id')::uuid);

-- สร้าง function สำหรับ set tenant
CREATE OR REPLACE FUNCTION set_tenant(tenant_id UUID)
RETURNS VOID AS $$
BEGIN
  PERFORM set_config('app.tenant_id', tenant_id::text, true);
END;
$$ LANGUAGE plpgsql;
```

```javascript
// ใช้ PostgreSQL RLS จาก Node.js
async function queryWithTenant(tenantId, queryText, params) {
  const client = await pool.connect();
  
  try {
    // Set tenant context
    await client.query('SELECT set_tenant($1)', [tenantId]);
    
    // Run query (RLS จะ filter อัตโนมัติ)
    const result = await client.query(queryText, params);
    
    return result.rows;
  } finally {
    client.release();
  }
}

// ใช้งาน
const users = await queryWithTenant(
  tenantId,
  'SELECT * FROM users WHERE status = $1',
  ['active']
);
```

### Tenant-aware Mongoose Model

```javascript
// models/base-model.js
const mongoose = require('mongoose');
const { mongoTenantPlugin } = require('../middleware/rls');

function createTenantModel(name, schema) {
  // ใช้ plugin
  schema.plugin(mongoTenantPlugin);
  
  return mongoose.model(name, schema);
}

// Tenant-aware query wrapper
class TenantModel {
  constructor(model, tenantId) {
    this.model = model;
    this.tenantId = tenantId;
  }

  find(conditions = {}) {
    const query = this.model.find({ ...conditions, tenantId: this.tenantId });
    return query;
  }

  findOne(conditions = {}) {
    return this.model.findOne({ ...conditions, tenantId: this.tenantId });
  }

  findById(id) {
    return this.model.findOne({ _id: id, tenantId: this.tenantId });
  }

  create(data) {
    return this.model.create({ ...data, tenantId: this.tenantId });
  }

  updateOne(conditions, update) {
    return this.model.updateOne(
      { ...conditions, tenantId: this.tenantId },
      update
    );
  }

  findByIdAndUpdate(id, update, options = {}) {
    return this.model.findOneAndUpdate(
      { _id: id, tenantId: this.tenantId },
      update,
      options
    );
  }

  deleteOne(conditions) {
    return this.model.deleteOne({ ...conditions, tenantId: this.tenantId });
  }

  countDocuments(conditions = {}) {
    return this.model.countDocuments({ ...conditions, tenantId: this.tenantId });
  }
}
```

---

## 5. Tenant Onboarding

```javascript
// services/tenant-onboarding.js
const Tenant = require('../models/Tenant');
const User = require('../models/User');
const schemaManager = require('./schema-manager');
const dbManager = require('./database-manager');
const emailService = require('./email.service');

class TenantOnboardingService {
  async createTenant(data) {
    const { companyName, adminEmail, adminName, plan } = data;
    
    // สร้าง slug
    const slug = this.generateSlug(companyName);
    
    // ตรวจสอบ slug ซ้ำ
    const existing = await Tenant.findOne({ slug });
    if (existing) {
      throw new Error(`Tenant slug "${slug}" already exists`);
    }

    // สร้าง tenant record
    const tenant = await Tenant.create({
      name: companyName,
      slug,
      plan,
      status: 'active',
      settings: this.getDefaultSettings(plan)
    });

    try {
      // สร้าง database/schema สำหรับ tenant
      await this.setupTenantDatabase(tenant.id);
      
      // สร้าง admin user
      const admin = await this.createAdminUser(tenant.id, {
        email: adminEmail,
        name: adminName
      });

      // ส่ง welcome email
      await emailService.sendWelcome(adminEmail, {
        name: adminName,
        companyName,
        loginUrl: `https://${slug}.myapp.com`
      });

      return { tenant, admin };
    } catch (error) {
      // Rollback ถ้า error
      await Tenant.deleteOne({ _id: tenant.id });
      throw error;
    }
  }

  async setupTenantDatabase(tenantId) {
    // สำหรับ schema-per-tenant strategy
    await schemaManager.createTenantSchema(tenantId);
    
    // Run migrations สำหรับ tenant ใหม่
    await this.runMigrations(tenantId);
  }

  async createAdminUser(tenantId, { email, name }) {
    const db = await dbManager.getConnection(tenantId);
    const User = db.model('User');
    
    const tempPassword = crypto.randomBytes(8).toString('hex');
    const hashedPassword = await bcrypt.hash(tempPassword, 12);
    
    const user = await User.create({
      email,
      name,
      role: 'admin',
      password: hashedPassword,
      mustChangePassword: true
    });

    // ส่ง password reset email
    await emailService.sendPasswordReset(email, tempPassword);
    
    return user;
  }

  generateSlug(name) {
    return name
      .toLowerCase()
      .replace(/[^a-z0-9]+/g, '-')
      .replace(/^-|-$/g, '')
      .substring(0, 50);
  }

  getDefaultSettings(plan) {
    const settings = {
      free: {
        maxUsers: 5,
        maxStorage: '1GB',
        features: ['basic']
      },
      pro: {
        maxUsers: 50,
        maxStorage: '50GB',
        features: ['basic', 'advanced', 'api']
      },
      enterprise: {
        maxUsers: -1, // Unlimited
        maxStorage: '-1',
        features: ['basic', 'advanced', 'api', 'sso', 'audit']
      }
    };
    
    return settings[plan] || settings.free;
  }
}

module.exports = new TenantOnboardingService();
```

---

## 6. Cross-tenant Operations (Admin)

```javascript
// services/admin.service.js
// สำหรับ platform admins เท่านั้น

class AdminService {
  async getMetrics() {
    const metrics = await Promise.all([
      Tenant.countDocuments({ status: 'active' }),
      Tenant.aggregate([
        { $group: { _id: '$plan', count: { $sum: 1 } } }
      ]),
      this.getTotalUsers(),
      this.getTotalRevenue()
    ]);

    return {
      activeTenants: metrics[0],
      tenantsByPlan: metrics[1],
      totalUsers: metrics[2],
      totalRevenue: metrics[3]
    };
  }

  async getTotalUsers() {
    const tenants = await Tenant.find({ status: 'active' });
    let total = 0;
    
    for (const tenant of tenants) {
      const db = await dbManager.getConnection(tenant.id);
      const User = db.model('User');
      const count = await User.countDocuments({ active: true });
      total += count;
    }
    
    return total;
  }

  async migrateTenants(migrationFn) {
    const tenants = await Tenant.find({ status: 'active' });
    const results = [];
    
    for (const tenant of tenants) {
      try {
        await migrationFn(tenant);
        results.push({ tenantId: tenant.id, status: 'success' });
      } catch (error) {
        results.push({
          tenantId: tenant.id,
          status: 'error',
          error: error.message
        });
      }
    }
    
    return results;
  }
}
```

---

## แบบฝึกหัด

### ระดับ 1: พื้นฐาน
สร้าง multi-tenant app ที่:
- ใช้ row-level security
- Tenant resolve จาก subdomain
- ข้อมูลแต่ละ tenant แยกจากกัน

### ระดับ 2: กลาง
สร้าง tenant onboarding flow:
- สร้าง tenant ใหม่พร้อม database
- สร้าง admin user
- ส่ง welcome email

### ระดับ 3: ขั้นสูง
สร้าง complete multi-tenant platform:
- Schema-per-tenant strategy
- Migration system
- Admin dashboard
- Billing integration

---

## สรุป

Multi-tenancy มีหลาย patterns ให้เลือกตามความต้องการ Row-level security เหมาะกับ early-stage apps ส่วน database-per-tenant เหมาะกับ enterprise ที่ต้องการ isolation สูง การเลือก pattern ที่ถูกต้องตั้งแต่ต้นจะช่วยประหยัดเวลา migration ในภายหลัง

> ขั้นตอนต่อไป: Part 64 - CQRS Pattern
