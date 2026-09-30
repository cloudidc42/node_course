# Part 22: PostgreSQL และ Sequelize

> ขั้นตอนที่ 22-30 จาก 1000 — Relational Database ด้วย PostgreSQL และ Sequelize ORM

---

## สารบัญ

1. [PostgreSQL คืออะไร](#postgresql-คืออะไร)
2. [Connection Pool](#connection-pool)
3. [Sequelize Setup](#sequelize-setup)
4. [Models และ Migrations](#models-และ-migrations)
5. [Associations](#associations)
6. [Transactions](#transactions)
7. [Raw Queries](#raw-queries)
8. [Practical: E-commerce กับ PostgreSQL](#practical-e-commerce-กับ-postgresql)
9. [แบบฝึกหัด](#แบบฝึกหัด)

---

## PostgreSQL คืออะไร

PostgreSQL (Postgres) เป็น **Relational Database Management System (RDBMS)** แบบ open-source ที่มีความสามารถสูง รองรับ ACID transactions และมี feature มากมาย

### ข้อดีของ PostgreSQL

- **ACID Compliant** — รับประกันความถูกต้องของข้อมูล
- **Strong Type System** — JSON, Array, UUID, ENUM และอื่นๆ
- **Full-text Search** — built-in support
- **Window Functions** — การวิเคราะห์ข้อมูลขั้นสูง
- **Extensible** — สร้าง custom types, functions
- **Performance** — query planner ที่ดีเยี่ยม

### การติดตั้ง

**Docker (แนะนำ):**
```bash
docker run -d \
  --name postgres \
  -e POSTGRES_USER=admin \
  -e POSTGRES_PASSWORD=secret \
  -e POSTGRES_DB=ecommerce \
  -p 5432:5432 \
  -v pgdata:/var/lib/postgresql/data \
  postgres:16

# เชื่อมต่อด้วย psql
docker exec -it postgres psql -U admin -d ecommerce
```

**Ubuntu:**
```bash
sudo apt-get install postgresql postgresql-contrib
sudo service postgresql start
sudo -u postgres createuser --interactive
sudo -u postgres createdb ecommerce
```

### ติดตั้ง dependencies

```bash
npm install sequelize pg pg-hstore
npm install --save-dev sequelize-cli
```

---

## Connection Pool

Connection pool ช่วยให้ใช้ connection ซ้ำได้ แทนที่จะสร้างใหม่ทุกครั้ง

```javascript
// config/database.js
const { Sequelize } = require('sequelize');

const sequelize = new Sequelize(
  process.env.DB_NAME,     // database name
  process.env.DB_USER,     // username
  process.env.DB_PASSWORD, // password
  {
    host: process.env.DB_HOST || 'localhost',
    port: process.env.DB_PORT || 5432,
    dialect: 'postgres',
    
    // Connection pool configuration
    pool: {
      max: 10,        // จำนวน connection สูงสุด
      min: 2,         // จำนวน connection ขั้นต่ำ
      acquire: 30000, // ms รอ connection ก่อน error
      idle: 10000,    // ms ก่อน connection ถูก release
    },
    
    // Logging
    logging: process.env.NODE_ENV === 'development'
      ? (sql) => console.log('\x1b[36m%s\x1b[0m', sql)
      : false,
    
    // Timezone
    timezone: '+07:00',
    
    // SSL (production)
    ...(process.env.NODE_ENV === 'production' && {
      dialectOptions: {
        ssl: {
          require: true,
          rejectUnauthorized: false,
        },
      },
    }),
  }
);

// ทดสอบการเชื่อมต่อ
const connectDB = async () => {
  try {
    await sequelize.authenticate();
    console.log('PostgreSQL connection established successfully');
  } catch (error) {
    console.error('Unable to connect to PostgreSQL:', error.message);
    process.exit(1);
  }
};

module.exports = { sequelize, connectDB };
```

---

## Sequelize Setup

### ใช้ Sequelize CLI

```bash
# เริ่มต้น project
npx sequelize-cli init

# สร้าง model
npx sequelize-cli model:generate --name User --attributes name:string,email:string,password:string

# รัน migration
npx sequelize-cli db:migrate

# ย้อน migration
npx sequelize-cli db:migrate:undo

# สร้าง seed
npx sequelize-cli seed:generate --name demo-users
npx sequelize-cli db:seed:all
```

### การตั้งค่า Sequelize CLI

```javascript
// .sequelizerc
const path = require('path');

module.exports = {
  config: path.resolve('config', 'sequelize.js'),
  'models-path': path.resolve('models'),
  'seeders-path': path.resolve('seeders'),
  'migrations-path': path.resolve('migrations'),
};
```

```javascript
// config/sequelize.js
require('dotenv').config();

module.exports = {
  development: {
    username: process.env.DB_USER || 'admin',
    password: process.env.DB_PASSWORD || 'secret',
    database: process.env.DB_NAME || 'ecommerce_dev',
    host: process.env.DB_HOST || 'localhost',
    dialect: 'postgres',
    logging: console.log,
  },
  test: {
    username: process.env.DB_USER,
    password: process.env.DB_PASSWORD,
    database: process.env.DB_NAME + '_test',
    host: process.env.DB_HOST,
    dialect: 'postgres',
    logging: false,
  },
  production: {
    use_env_variable: 'DATABASE_URL',
    dialect: 'postgres',
    logging: false,
    dialectOptions: {
      ssl: { rejectUnauthorized: false },
    },
  },
};
```

---

## Models และ Migrations

### สร้าง Migration

```javascript
// migrations/20240115000001-create-users.js
'use strict';

module.exports = {
  async up(queryInterface, Sequelize) {
    await queryInterface.createTable('Users', {
      id: {
        allowNull: false,
        autoIncrement: true,
        primaryKey: true,
        type: Sequelize.INTEGER,
      },
      name: {
        type: Sequelize.STRING(100),
        allowNull: false,
      },
      email: {
        type: Sequelize.STRING(255),
        allowNull: false,
        unique: true,
      },
      password: {
        type: Sequelize.STRING(255),
        allowNull: false,
      },
      role: {
        type: Sequelize.ENUM('user', 'admin', 'moderator'),
        defaultValue: 'user',
      },
      isActive: {
        type: Sequelize.BOOLEAN,
        defaultValue: true,
      },
      lastLoginAt: {
        type: Sequelize.DATE,
        allowNull: true,
      },
      createdAt: {
        allowNull: false,
        type: Sequelize.DATE,
        defaultValue: Sequelize.literal('CURRENT_TIMESTAMP'),
      },
      updatedAt: {
        allowNull: false,
        type: Sequelize.DATE,
        defaultValue: Sequelize.literal('CURRENT_TIMESTAMP'),
      },
    });
    
    // เพิ่ม indexes
    await queryInterface.addIndex('Users', ['email'], { unique: true });
    await queryInterface.addIndex('Users', ['role', 'isActive']);
  },
  
  async down(queryInterface, Sequelize) {
    await queryInterface.dropTable('Users');
  },
};
```

```javascript
// migrations/20240115000002-create-products.js
'use strict';

module.exports = {
  async up(queryInterface, Sequelize) {
    await queryInterface.createTable('Products', {
      id: {
        allowNull: false,
        autoIncrement: true,
        primaryKey: true,
        type: Sequelize.INTEGER,
      },
      name: {
        type: Sequelize.STRING(200),
        allowNull: false,
      },
      description: {
        type: Sequelize.TEXT,
        allowNull: true,
      },
      price: {
        type: Sequelize.DECIMAL(10, 2),
        allowNull: false,
        defaultValue: 0,
      },
      comparePrice: {
        type: Sequelize.DECIMAL(10, 2),
        allowNull: true,
      },
      sku: {
        type: Sequelize.STRING(50),
        unique: true,
        allowNull: false,
      },
      stock: {
        type: Sequelize.INTEGER,
        defaultValue: 0,
      },
      categoryId: {
        type: Sequelize.INTEGER,
        references: {
          model: 'Categories',
          key: 'id',
        },
        onUpdate: 'CASCADE',
        onDelete: 'SET NULL',
      },
      images: {
        type: Sequelize.ARRAY(Sequelize.STRING),  // PostgreSQL array
        defaultValue: [],
      },
      attributes: {
        type: Sequelize.JSONB,  // JSON binary สำหรับ indexing
        defaultValue: {},
      },
      isActive: {
        type: Sequelize.BOOLEAN,
        defaultValue: true,
      },
      createdAt: {
        allowNull: false,
        type: Sequelize.DATE,
      },
      updatedAt: {
        allowNull: false,
        type: Sequelize.DATE,
      },
    });
    
    await queryInterface.addIndex('Products', ['sku'], { unique: true });
    await queryInterface.addIndex('Products', ['categoryId']);
    await queryInterface.addIndex('Products', ['price']);
    
    // GIN index สำหรับ JSONB search
    await queryInterface.addIndex('Products', ['attributes'], {
      using: 'GIN',
    });
  },
  
  async down(queryInterface, Sequelize) {
    await queryInterface.dropTable('Products');
  },
};
```

### สร้าง Model

```javascript
// models/User.js
'use strict';

const { Model, DataTypes } = require('sequelize');
const bcrypt = require('bcryptjs');

module.exports = (sequelize) => {
  class User extends Model {
    // Instance methods
    async comparePassword(candidatePassword) {
      return bcrypt.compare(candidatePassword, this.password);
    }
    
    toSafeObject() {
      const { password, ...safe } = this.toJSON();
      return safe;
    }
    
    // Static methods
    static async findByEmail(email) {
      return User.findOne({ where: { email: email.toLowerCase() } });
    }
    
    // Associations
    static associate(models) {
      User.hasMany(models.Order, { foreignKey: 'userId', as: 'orders' });
      User.hasMany(models.Review, { foreignKey: 'userId', as: 'reviews' });
      User.hasOne(models.Cart, { foreignKey: 'userId', as: 'cart' });
    }
  }
  
  User.init(
    {
      id: {
        type: DataTypes.INTEGER,
        autoIncrement: true,
        primaryKey: true,
      },
      name: {
        type: DataTypes.STRING(100),
        allowNull: false,
        validate: {
          notEmpty: { msg: 'กรุณาระบุชื่อ' },
          len: {
            args: [2, 100],
            msg: 'ชื่อต้องมี 2-100 ตัวอักษร',
          },
        },
      },
      email: {
        type: DataTypes.STRING(255),
        allowNull: false,
        unique: { msg: 'Email นี้มีผู้ใช้แล้ว' },
        validate: {
          isEmail: { msg: 'กรุณาระบุ email ที่ถูกต้อง' },
          notEmpty: true,
        },
        set(value) {
          this.setDataValue('email', value.toLowerCase().trim());
        },
      },
      password: {
        type: DataTypes.STRING(255),
        allowNull: false,
        validate: {
          len: {
            args: [8, 255],
            msg: 'รหัสผ่านต้องมีอย่างน้อย 8 ตัวอักษร',
          },
        },
      },
      role: {
        type: DataTypes.ENUM('user', 'admin', 'moderator'),
        defaultValue: 'user',
      },
      isActive: {
        type: DataTypes.BOOLEAN,
        defaultValue: true,
      },
      lastLoginAt: DataTypes.DATE,
    },
    {
      sequelize,
      modelName: 'User',
      tableName: 'Users',
      timestamps: true,
      
      hooks: {
        // Hash password ก่อน create/update
        beforeCreate: async (user) => {
          if (user.password) {
            user.password = await bcrypt.hash(user.password, 12);
          }
        },
        beforeUpdate: async (user) => {
          if (user.changed('password')) {
            user.password = await bcrypt.hash(user.password, 12);
          }
        },
      },
      
      scopes: {
        active: { where: { isActive: true } },
        admins: { where: { role: 'admin' } },
        withoutPassword: {
          attributes: { exclude: ['password'] },
        },
      },
    }
  );
  
  return User;
};
```

```javascript
// models/Product.js
'use strict';

const { Model, DataTypes } = require('sequelize');

module.exports = (sequelize) => {
  class Product extends Model {
    static associate(models) {
      Product.belongsTo(models.Category, {
        foreignKey: 'categoryId',
        as: 'category',
      });
      Product.hasMany(models.OrderItem, {
        foreignKey: 'productId',
        as: 'orderItems',
      });
      Product.hasMany(models.Review, {
        foreignKey: 'productId',
        as: 'reviews',
      });
    }
    
    // คำนวณ discount percentage
    get discountPercent() {
      if (!this.comparePrice || this.comparePrice <= this.price) return 0;
      return Math.round(((this.comparePrice - this.price) / this.comparePrice) * 100);
    }
  }
  
  Product.init(
    {
      name: {
        type: DataTypes.STRING(200),
        allowNull: false,
        validate: {
          notEmpty: { msg: 'กรุณาระบุชื่อสินค้า' },
        },
      },
      description: DataTypes.TEXT,
      price: {
        type: DataTypes.DECIMAL(10, 2),
        allowNull: false,
        validate: {
          min: { args: [0], msg: 'ราคาต้องไม่ติดลบ' },
          isDecimal: true,
        },
        get() {
          // แปลงจาก string เป็น number
          return parseFloat(this.getDataValue('price'));
        },
      },
      comparePrice: {
        type: DataTypes.DECIMAL(10, 2),
        get() {
          const val = this.getDataValue('comparePrice');
          return val ? parseFloat(val) : null;
        },
      },
      sku: {
        type: DataTypes.STRING(50),
        allowNull: false,
        unique: { msg: 'SKU นี้มีอยู่แล้ว' },
        validate: {
          notEmpty: true,
        },
      },
      stock: {
        type: DataTypes.INTEGER,
        defaultValue: 0,
        validate: {
          min: { args: [0], msg: 'สต็อกต้องไม่ติดลบ' },
          isInt: true,
        },
      },
      categoryId: {
        type: DataTypes.INTEGER,
        allowNull: true,
      },
      images: {
        type: DataTypes.ARRAY(DataTypes.STRING),
        defaultValue: [],
      },
      attributes: {
        type: DataTypes.JSONB,
        defaultValue: {},
      },
      isActive: {
        type: DataTypes.BOOLEAN,
        defaultValue: true,
      },
    },
    {
      sequelize,
      modelName: 'Product',
      tableName: 'Products',
      timestamps: true,
      scopes: {
        active: { where: { isActive: true } },
        inStock: {
          where: sequelize.literal('"stock" > 0'),
        },
      },
    }
  );
  
  return Product;
};
```

### models/index.js

```javascript
// models/index.js
'use strict';

const fs = require('fs');
const path = require('path');
const { Sequelize } = require('sequelize');
const config = require('../config/sequelize.js')[process.env.NODE_ENV || 'development'];

let sequelize;
if (config.use_env_variable) {
  sequelize = new Sequelize(process.env[config.use_env_variable], config);
} else {
  sequelize = new Sequelize(
    config.database,
    config.username,
    config.password,
    config
  );
}

const db = {};

// โหลด models ทั้งหมดจากโฟลเดอร์
fs.readdirSync(__dirname)
  .filter((file) => file !== 'index.js' && file.endsWith('.js'))
  .forEach((file) => {
    const model = require(path.join(__dirname, file))(sequelize, Sequelize.DataTypes);
    db[model.name] = model;
  });

// สร้าง associations
Object.keys(db).forEach((modelName) => {
  if (db[modelName].associate) {
    db[modelName].associate(db);
  }
});

db.sequelize = sequelize;
db.Sequelize = Sequelize;

module.exports = db;
```

---

## Associations

### ประเภทของ Associations

```javascript
// 1. hasOne — User มี 1 Profile
User.hasOne(Profile, { foreignKey: 'userId' });
Profile.belongsTo(User, { foreignKey: 'userId' });

// 2. hasMany — User มีหลาย Orders
User.hasMany(Order, { foreignKey: 'userId', as: 'orders' });
Order.belongsTo(User, { foreignKey: 'userId', as: 'user' });

// 3. belongsToMany — Product มีหลาย Tags (many-to-many)
Product.belongsToMany(Tag, {
  through: 'ProductTags',  // junction table
  foreignKey: 'productId',
  otherKey: 'tagId',
  as: 'tags',
});
Tag.belongsToMany(Product, {
  through: 'ProductTags',
  foreignKey: 'tagId',
  otherKey: 'productId',
  as: 'products',
});
```

### ตัวอย่าง E-commerce Associations

```javascript
// models/Order.js
module.exports = (sequelize) => {
  class Order extends Model {
    static associate(models) {
      Order.belongsTo(models.User, { foreignKey: 'userId', as: 'user' });
      Order.hasMany(models.OrderItem, { foreignKey: 'orderId', as: 'items' });
      Order.belongsTo(models.Address, { foreignKey: 'addressId', as: 'shippingAddress' });
    }
  }
  
  Order.init(
    {
      userId: {
        type: DataTypes.INTEGER,
        allowNull: false,
      },
      status: {
        type: DataTypes.ENUM(
          'pending', 'confirmed', 'processing', 'shipped', 'delivered', 'cancelled'
        ),
        defaultValue: 'pending',
      },
      subtotal: {
        type: DataTypes.DECIMAL(10, 2),
        allowNull: false,
      },
      shippingFee: {
        type: DataTypes.DECIMAL(10, 2),
        defaultValue: 0,
      },
      discount: {
        type: DataTypes.DECIMAL(10, 2),
        defaultValue: 0,
      },
      total: {
        type: DataTypes.DECIMAL(10, 2),
        allowNull: false,
      },
      notes: DataTypes.TEXT,
      addressId: DataTypes.INTEGER,
      paidAt: DataTypes.DATE,
      shippedAt: DataTypes.DATE,
      deliveredAt: DataTypes.DATE,
    },
    { sequelize, modelName: 'Order', timestamps: true }
  );
  
  return Order;
};
```

### Eager Loading (include)

```javascript
// controller ดึง orders พร้อม items และ user
const getOrders = async (req, res) => {
  const { sequelize } = require('../models');
  const { Order, OrderItem, Product, User } = require('../models');
  
  try {
    const orders = await Order.findAll({
      where: { userId: req.user.id },
      include: [
        {
          model: OrderItem,
          as: 'items',
          include: [
            {
              model: Product,
              as: 'product',
              attributes: ['id', 'name', 'sku', 'images'],
            },
          ],
        },
        {
          model: User,
          as: 'user',
          attributes: ['id', 'name', 'email'],
        },
      ],
      order: [['createdAt', 'DESC']],
    });
    
    res.json({ success: true, data: orders });
  } catch (error) {
    res.status(500).json({ success: false, message: error.message });
  }
};
```

---

## Transactions

Transaction รับประกันว่า operations หลายอย่างจะ succeed หรือ fail พร้อมกัน

```javascript
const { sequelize, Order, OrderItem, Product, Cart } = require('../models');

// Managed transaction (auto commit/rollback)
const createOrder = async (req, res) => {
  try {
    const result = await sequelize.transaction(async (t) => {
      const { items, addressId, notes } = req.body;
      
      // 1. ตรวจสอบสต็อก
      const products = await Product.findAll({
        where: { id: items.map((i) => i.productId) },
        lock: t.LOCK.UPDATE,  // Pessimistic locking
        transaction: t,
      });
      
      for (const item of items) {
        const product = products.find((p) => p.id === item.productId);
        if (!product) {
          throw new Error(`ไม่พบสินค้า ID: ${item.productId}`);
        }
        if (product.stock < item.quantity) {
          throw new Error(`สินค้า "${product.name}" มีสต็อกไม่เพียงพอ`);
        }
      }
      
      // 2. คำนวณราคา
      let subtotal = 0;
      const orderItems = items.map((item) => {
        const product = products.find((p) => p.id === item.productId);
        const price = product.price;
        subtotal += price * item.quantity;
        return {
          productId: item.productId,
          quantity: item.quantity,
          price,
          total: price * item.quantity,
        };
      });
      
      const shippingFee = subtotal >= 500 ? 0 : 50;
      const total = subtotal + shippingFee;
      
      // 3. สร้าง order
      const order = await Order.create(
        {
          userId: req.user.id,
          subtotal,
          shippingFee,
          total,
          addressId,
          notes,
        },
        { transaction: t }
      );
      
      // 4. สร้าง order items
      await OrderItem.bulkCreate(
        orderItems.map((item) => ({ ...item, orderId: order.id })),
        { transaction: t }
      );
      
      // 5. ลดสต็อก
      for (const item of items) {
        await Product.decrement('stock', {
          by: item.quantity,
          where: { id: item.productId },
          transaction: t,
        });
      }
      
      // 6. ล้าง cart
      await Cart.destroy({
        where: { userId: req.user.id },
        transaction: t,
      });
      
      return order;
    });
    
    res.status(201).json({ success: true, data: result });
  } catch (error) {
    // Transaction rollback อัตโนมัติเมื่อ throw error
    res.status(400).json({ success: false, message: error.message });
  }
};

// Unmanaged transaction (manual commit/rollback)
const transferFunds = async (fromUserId, toUserId, amount) => {
  const t = await sequelize.transaction();
  
  try {
    const fromUser = await User.findByPk(fromUserId, {
      lock: t.LOCK.UPDATE,
      transaction: t,
    });
    
    if (fromUser.balance < amount) {
      throw new Error('ยอดเงินไม่เพียงพอ');
    }
    
    await fromUser.decrement('balance', { by: amount, transaction: t });
    await User.increment('balance', {
      by: amount,
      where: { id: toUserId },
      transaction: t,
    });
    
    await t.commit();
    return { success: true };
  } catch (error) {
    await t.rollback();
    throw error;
  }
};
```

---

## Raw Queries

เมื่อต้องการ query ที่ซับซ้อนเกินกว่า ORM จะจัดการได้

```javascript
const { sequelize } = require('../models');
const { QueryTypes } = require('sequelize');

// SELECT query
const getTopProducts = async () => {
  const results = await sequelize.query(
    `
    SELECT
      p.id,
      p.name,
      p.price,
      c.name AS category_name,
      COUNT(DISTINCT oi.order_id) AS order_count,
      SUM(oi.quantity) AS total_sold,
      AVG(r.rating) AS avg_rating,
      COUNT(r.id) AS review_count
    FROM "Products" p
    LEFT JOIN "Categories" c ON c.id = p."categoryId"
    LEFT JOIN "OrderItems" oi ON oi."productId" = p.id
    LEFT JOIN "Reviews" r ON r."productId" = p.id
    WHERE p."isActive" = true
    GROUP BY p.id, c.name
    ORDER BY total_sold DESC NULLS LAST
    LIMIT :limit
    `,
    {
      replacements: { limit: 10 },  // ป้องกัน SQL injection
      type: QueryTypes.SELECT,
    }
  );
  
  return results;
};

// Query ด้วย window functions
const getSalesReport = async (year) => {
  const results = await sequelize.query(
    `
    SELECT
      DATE_TRUNC('month', o."createdAt") AS month,
      COUNT(o.id) AS order_count,
      SUM(o.total) AS revenue,
      SUM(SUM(o.total)) OVER (
        ORDER BY DATE_TRUNC('month', o."createdAt")
        ROWS UNBOUNDED PRECEDING
      ) AS cumulative_revenue,
      LAG(SUM(o.total)) OVER (
        ORDER BY DATE_TRUNC('month', o."createdAt")
      ) AS prev_month_revenue
    FROM "Orders" o
    WHERE
      EXTRACT(YEAR FROM o."createdAt") = :year
      AND o.status = 'delivered'
    GROUP BY DATE_TRUNC('month', o."createdAt")
    ORDER BY month
    `,
    {
      replacements: { year },
      type: QueryTypes.SELECT,
    }
  );
  
  return results;
};

// INSERT/UPDATE/DELETE
const bulkUpdatePrices = async (categoryId, priceIncrease) => {
  const [affectedRows] = await sequelize.query(
    `
    UPDATE "Products"
    SET
      price = price * (1 + :increase / 100),
      "updatedAt" = NOW()
    WHERE "categoryId" = :categoryId
      AND "isActive" = true
    `,
    {
      replacements: { increase: priceIncrease, categoryId },
      type: QueryTypes.UPDATE,
    }
  );
  
  return affectedRows;
};
```

---

## Practical: E-commerce กับ PostgreSQL

### โครงสร้างโปรเจค

```
ecommerce-api/
├── config/
│   └── sequelize.js
├── migrations/
│   ├── 20240115000001-create-users.js
│   ├── 20240115000002-create-categories.js
│   ├── 20240115000003-create-products.js
│   ├── 20240115000004-create-orders.js
│   └── 20240115000005-create-order-items.js
├── models/
│   ├── index.js
│   ├── User.js
│   ├── Category.js
│   ├── Product.js
│   ├── Order.js
│   └── OrderItem.js
├── controllers/
│   ├── productController.js
│   └── orderController.js
├── routes/
│   ├── products.js
│   └── orders.js
├── seeders/
│   └── 20240115000001-demo-data.js
└── app.js
```

### Product Controller

```javascript
// controllers/productController.js
const { Op } = require('sequelize');
const { Product, Category, Review, sequelize } = require('../models');

// GET /api/products
exports.getProducts = async (req, res) => {
  try {
    const {
      page = 1,
      limit = 12,
      search,
      category,
      minPrice,
      maxPrice,
      sort = 'createdAt',
      order = 'DESC',
      inStock,
    } = req.query;
    
    const offset = (parseInt(page) - 1) * parseInt(limit);
    
    // Build where clause
    const where = { isActive: true };
    
    if (search) {
      where[Op.or] = [
        { name: { [Op.iLike]: `%${search}%` } },    // case-insensitive LIKE
        { description: { [Op.iLike]: `%${search}%` } },
      ];
    }
    
    if (category) where.categoryId = category;
    
    if (minPrice || maxPrice) {
      where.price = {};
      if (minPrice) where.price[Op.gte] = parseFloat(minPrice);
      if (maxPrice) where.price[Op.lte] = parseFloat(maxPrice);
    }
    
    if (inStock === 'true') where.stock = { [Op.gt]: 0 };
    
    // Valid sort columns
    const validSort = ['name', 'price', 'createdAt', 'stock'];
    const sortColumn = validSort.includes(sort) ? sort : 'createdAt';
    const sortOrder = order.toUpperCase() === 'ASC' ? 'ASC' : 'DESC';
    
    const { count, rows } = await Product.findAndCountAll({
      where,
      include: [
        {
          model: Category,
          as: 'category',
          attributes: ['id', 'name', 'slug'],
        },
      ],
      attributes: {
        include: [
          [
            sequelize.literal(
              '(SELECT COALESCE(AVG(rating), 0) FROM "Reviews" WHERE "productId" = "Product".id)'
            ),
            'avgRating',
          ],
          [
            sequelize.literal(
              '(SELECT COUNT(*) FROM "Reviews" WHERE "productId" = "Product".id)'
            ),
            'reviewCount',
          ],
        ],
      },
      order: [[sortColumn, sortOrder]],
      limit: parseInt(limit),
      offset,
      distinct: true,  // จำเป็นเมื่อ include hasMany
    });
    
    res.json({
      success: true,
      data: rows,
      pagination: {
        total: count,
        page: parseInt(page),
        limit: parseInt(limit),
        totalPages: Math.ceil(count / parseInt(limit)),
      },
    });
  } catch (error) {
    res.status(500).json({ success: false, message: error.message });
  }
};

// GET /api/products/:id
exports.getProduct = async (req, res) => {
  try {
    const product = await Product.findByPk(req.params.id, {
      include: [
        { model: Category, as: 'category', attributes: ['id', 'name', 'slug'] },
        {
          model: Review,
          as: 'reviews',
          limit: 10,
          order: [['createdAt', 'DESC']],
          include: [{ model: User, as: 'user', attributes: ['id', 'name', 'avatar'] }],
        },
      ],
    });
    
    if (!product) {
      return res.status(404).json({ success: false, message: 'ไม่พบสินค้า' });
    }
    
    res.json({ success: true, data: product });
  } catch (error) {
    res.status(500).json({ success: false, message: error.message });
  }
};
```

### Seeder

```javascript
// seeders/20240115000001-demo-data.js
'use strict';
const bcrypt = require('bcryptjs');

module.exports = {
  async up(queryInterface, Sequelize) {
    // สร้าง users
    const hashedPassword = await bcrypt.hash('password123', 12);
    
    await queryInterface.bulkInsert('Users', [
      {
        name: 'Admin User',
        email: 'admin@example.com',
        password: hashedPassword,
        role: 'admin',
        isActive: true,
        createdAt: new Date(),
        updatedAt: new Date(),
      },
      {
        name: 'สมชาย ใจดี',
        email: 'somchai@example.com',
        password: hashedPassword,
        role: 'user',
        isActive: true,
        createdAt: new Date(),
        updatedAt: new Date(),
      },
    ]);
    
    // สร้าง categories
    await queryInterface.bulkInsert('Categories', [
      { name: 'Electronics', slug: 'electronics', createdAt: new Date(), updatedAt: new Date() },
      { name: 'Clothing', slug: 'clothing', createdAt: new Date(), updatedAt: new Date() },
      { name: 'Books', slug: 'books', createdAt: new Date(), updatedAt: new Date() },
    ]);
    
    // สร้าง products
    await queryInterface.bulkInsert('Products', [
      {
        name: 'iPhone 15 Pro',
        description: 'สมาร์ทโฟนรุ่นล่าสุดจาก Apple',
        price: 45000.00,
        sku: 'IPHONE-15-PRO',
        stock: 50,
        categoryId: 1,
        images: JSON.stringify(['https://example.com/iphone15.jpg']),
        attributes: JSON.stringify({ brand: 'Apple', storage: '256GB', color: 'Space Black' }),
        isActive: true,
        createdAt: new Date(),
        updatedAt: new Date(),
      },
    ]);
  },
  
  async down(queryInterface, Sequelize) {
    await queryInterface.bulkDelete('Products', null, {});
    await queryInterface.bulkDelete('Categories', null, {});
    await queryInterface.bulkDelete('Users', null, {});
  },
};
```

---

## แบบฝึกหัด

### Exercise 1: สร้าง Order Management System

สร้าง API ที่มี:
1. CRUD สำหรับ Orders
2. Transaction สำหรับ checkout process
3. Order status tracking (pending → confirmed → shipped → delivered)
4. Report: ยอดขายรายวัน/รายเดือน

### Exercise 2: Migration Strategy

1. เพิ่ม column `phone` ใน Users table
2. เพิ่ม `discountCode` column ใน Orders table
3. สร้าง index ใหม่
4. เขียน rollback สำหรับทุก migration

### Exercise 3: Query Performance

1. ใช้ `EXPLAIN ANALYZE` วิเคราะห์ slow queries
2. เพิ่ม indexes ที่เหมาะสม
3. ใช้ pagination แทน `findAll` ที่ไม่มี limit
4. Benchmark ก่อน-หลัง optimization

---

## สรุป

ในบทนี้เราได้เรียนรู้:

- **PostgreSQL** และข้อดีของ Relational Database
- **Connection Pool** เพื่อประสิทธิภาพ
- **Sequelize ORM** การตั้งค่าและใช้งาน
- **Models และ Migrations** การจัดการ schema
- **Associations** ทุกประเภท
- **Transactions** เพื่อความสมบูรณ์ของข้อมูล
- **Raw Queries** สำหรับ complex operations

> **บทถัดไป:** Part 23 — Authentication Basics
