# Part 76: Prisma ORM
## ขั้นตอนที่ 751-760 จาก 1000

---

## Prisma คืออะไร?

Prisma เป็น next-generation ORM สำหรับ Node.js และ TypeScript ที่มี type-safe database access, auto-generated migrations, และ Prisma Studio GUI

---

## 1. Setup

```bash
# ติดตั้ง
npm install prisma @prisma/client
npx prisma init

# กำหนด database
# .env
DATABASE_URL="postgresql://user:password@localhost:5432/mydb"
```

### schema.prisma

```prisma
// prisma/schema.prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model User {
  id        String   @id @default(cuid())
  email     String   @unique
  name      String
  role      Role     @default(USER)
  profile   Profile?
  posts     Post[]
  orders    Order[]
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt

  @@index([email])
  @@map("users")
}

model Profile {
  id     String  @id @default(cuid())
  bio    String?
  avatar String?
  userId String  @unique
  user   User    @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@map("profiles")
}

model Post {
  id          String     @id @default(cuid())
  title       String
  content     String?
  published   Boolean    @default(false)
  publishedAt DateTime?
  authorId    String
  author      User       @relation(fields: [authorId], references: [id])
  tags        Tag[]
  categories  Category[]
  createdAt   DateTime   @default(now())
  updatedAt   DateTime   @updatedAt

  @@index([authorId, published])
  @@map("posts")
}

model Tag {
  id    String @id @default(cuid())
  name  String @unique
  posts Post[]

  @@map("tags")
}

model Category {
  id       String @id @default(cuid())
  name     String @unique
  parentId String?
  parent   Category?  @relation("CategoryHierarchy", fields: [parentId], references: [id])
  children Category[] @relation("CategoryHierarchy")
  posts    Post[]

  @@map("categories")
}

model Order {
  id        String      @id @default(cuid())
  userId    String
  user      User        @relation(fields: [userId], references: [id])
  items     OrderItem[]
  total     Decimal     @db.Decimal(10, 2)
  status    OrderStatus @default(PENDING)
  createdAt DateTime    @default(now())

  @@map("orders")
}

model OrderItem {
  id        String  @id @default(cuid())
  orderId   String
  order     Order   @relation(fields: [orderId], references: [id])
  productId String
  quantity  Int
  price     Decimal @db.Decimal(10, 2)

  @@map("order_items")
}

enum Role {
  USER
  ADMIN
  MODERATOR
}

enum OrderStatus {
  PENDING
  CONFIRMED
  SHIPPED
  DELIVERED
  CANCELLED
}
```

---

## 2. Migrations

```bash
# สร้าง migration
npx prisma migrate dev --name init
npx prisma migrate dev --name add_user_profile

# Deploy migration ใน production
npx prisma migrate deploy

# Reset database (development only)
npx prisma migrate reset

# ดู migration status
npx prisma migrate status

# Prisma Studio (GUI)
npx prisma studio
```

### Custom Migration

```sql
-- prisma/migrations/20241001_add_full_text_search/migration.sql
-- เพิ่ม full-text search index
CREATE INDEX posts_search_idx ON posts 
USING GIN (to_tsvector('english', title || ' ' || COALESCE(content, '')));
```

---

## 3. CRUD Operations

```javascript
// lib/prisma.js
const { PrismaClient } = require('@prisma/client');

const prisma = new PrismaClient({
  log: ['query', 'info', 'warn', 'error']
});

module.exports = prisma;
```

```javascript
// repositories/user.repository.js
const prisma = require('../lib/prisma');

class UserRepository {
  // Create
  async create(data) {
    return prisma.user.create({
      data: {
        email: data.email,
        name: data.name,
        profile: data.bio ? {
          create: { bio: data.bio }
        } : undefined
      },
      include: { profile: true }
    });
  }

  // Read
  async findById(id) {
    return prisma.user.findUnique({
      where: { id },
      include: {
        profile: true,
        posts: {
          where: { published: true },
          orderBy: { createdAt: 'desc' },
          take: 5
        }
      }
    });
  }

  async findByEmail(email) {
    return prisma.user.findUnique({
      where: { email: email.toLowerCase() }
    });
  }

  async findMany(options = {}) {
    const { page = 1, limit = 20, role, search } = options;
    
    const where = {};
    if (role) where.role = role;
    if (search) {
      where.OR = [
        { name: { contains: search, mode: 'insensitive' } },
        { email: { contains: search, mode: 'insensitive' } }
      ];
    }

    const [users, total] = await Promise.all([
      prisma.user.findMany({
        where,
        skip: (page - 1) * limit,
        take: limit,
        orderBy: { createdAt: 'desc' },
        include: { profile: true },
        select: {
          id: true,
          email: true,
          name: true,
          role: true,
          createdAt: true,
          profile: { select: { avatar: true } }
        }
      }),
      prisma.user.count({ where })
    ]);

    return { users, total, page, limit };
  }

  // Update
  async update(id, data) {
    return prisma.user.update({
      where: { id },
      data: {
        name: data.name,
        profile: {
          upsert: {
            create: { bio: data.bio, avatar: data.avatar },
            update: { bio: data.bio, avatar: data.avatar }
          }
        }
      },
      include: { profile: true }
    });
  }

  // Delete
  async delete(id) {
    return prisma.user.delete({ where: { id } });
  }

  // Soft delete
  async softDelete(id) {
    return prisma.user.update({
      where: { id },
      data: { deletedAt: new Date() }
    });
  }
}

module.exports = new UserRepository();
```

---

## 4. Relations

```javascript
// queries/post.queries.js

// One-to-many
async function getUserWithPosts(userId) {
  return prisma.user.findUnique({
    where: { id: userId },
    include: {
      posts: {
        where: { published: true },
        orderBy: { publishedAt: 'desc' },
        take: 10
      }
    }
  });
}

// Many-to-many (Posts ↔ Tags)
async function createPostWithTags(data) {
  return prisma.post.create({
    data: {
      title: data.title,
      content: data.content,
      authorId: data.authorId,
      tags: {
        connectOrCreate: data.tags.map(tagName => ({
          where: { name: tagName },
          create: { name: tagName }
        }))
      }
    },
    include: { tags: true }
  });
}

// Nested select
async function getPostsWithDetails() {
  return prisma.post.findMany({
    select: {
      id: true,
      title: true,
      publishedAt: true,
      author: {
        select: {
          name: true,
          profile: { select: { avatar: true } }
        }
      },
      tags: { select: { name: true } },
      _count: { select: { comments: true } }
    }
  });
}

// Aggregation
async function getPostStats() {
  return prisma.post.aggregate({
    _count: { _all: true },
    _avg: { viewCount: true },
    where: { published: true }
  });
}

// GroupBy
async function getPostCountByCategory() {
  return prisma.post.groupBy({
    by: ['categoryId'],
    _count: { id: true },
    orderBy: { _count: { id: 'desc' } }
  });
}
```

---

## 5. Transactions

```javascript
// transactions/order.transaction.js

async function createOrder(userId, items) {
  return prisma.$transaction(async (tx) => {
    // ตรวจสอบ inventory
    for (const item of items) {
      const product = await tx.product.findUnique({
        where: { id: item.productId },
        select: { id: true, price: true, stock: true }
      });

      if (!product || product.stock < item.quantity) {
        throw new Error(`Insufficient stock for product ${item.productId}`);
      }
    }

    // คำนวณ total
    const total = items.reduce((sum, item) => {
      return sum + (item.price * item.quantity);
    }, 0);

    // สร้าง order
    const order = await tx.order.create({
      data: {
        userId,
        total,
        items: {
          create: items.map(item => ({
            productId: item.productId,
            quantity: item.quantity,
            price: item.price
          }))
        }
      },
      include: { items: true }
    });

    // ลด stock
    await Promise.all(items.map(item =>
      tx.product.update({
        where: { id: item.productId },
        data: { stock: { decrement: item.quantity } }
      })
    ));

    return order;
  });
}

// Interactive transactions (สำหรับ complex flows)
async function complexTransaction() {
  return prisma.$transaction(async (tx) => {
    const user = await tx.user.create({ data: { name: 'Test', email: 'test@test.com' } });
    
    // เงื่อนไขใน transaction
    if (!user.email.includes('@')) {
      throw new Error('Invalid email');
    }
    
    const profile = await tx.profile.create({
      data: { userId: user.id, bio: 'Hello' }
    });
    
    return { user, profile };
  }, {
    maxWait: 5000,
    timeout: 10000,
    isolationLevel: 'Serializable'
  });
}
```

---

## 6. Advanced Prisma

### Middleware

```javascript
// Prisma middleware สำหรับ soft delete
prisma.$use(async (params, next) => {
  // Soft delete
  if (params.action === 'delete') {
    params.action = 'update';
    params.args['data'] = { deletedAt: new Date() };
  }
  
  if (params.action === 'deleteMany') {
    params.action = 'updateMany';
    if (params.args.data !== undefined) {
      params.args.data['deletedAt'] = new Date();
    } else {
      params.args['data'] = { deletedAt: new Date() };
    }
  }
  
  // Filter soft deleted records
  if (params.action === 'find' || params.action === 'findMany') {
    if (params.args.where) {
      if (params.args.where.deletedAt === undefined) {
        params.args.where['deletedAt'] = null;
      }
    } else {
      params.args['where'] = { deletedAt: null };
    }
  }
  
  return next(params);
});

// Logging middleware
prisma.$use(async (params, next) => {
  const before = Date.now();
  const result = await next(params);
  const after = Date.now();
  
  if (after - before > 1000) {
    console.warn(`Slow query: ${params.model}.${params.action} took ${after - before}ms`);
  }
  
  return result;
});
```

### Raw Queries

```javascript
// Raw SQL เมื่อ Prisma ทำไม่ได้
const users = await prisma.$queryRaw`
  SELECT u.*, COUNT(p.id) as post_count
  FROM users u
  LEFT JOIN posts p ON p.author_id = u.id AND p.published = true
  WHERE u.created_at > ${new Date('2024-01-01')}
  GROUP BY u.id
  ORDER BY post_count DESC
  LIMIT 10
`;

// Execute
await prisma.$executeRaw`
  UPDATE users SET last_seen = NOW() WHERE id = ${userId}
`;
```

---

## แบบฝึกหัด

### ระดับ 1: พื้นฐาน
- ตั้งค่า Prisma กับ PostgreSQL
- สร้าง schema สำหรับ blog
- Migration

### ระดับ 2: กลาง
- CRUD operations ทุก models
- Relations (1:1, 1:N, M:N)
- Pagination + filtering

### ระดับ 3: ขั้นสูง
- Transactions
- Middleware
- Performance optimization

---

## สรุป

Prisma เป็น ORM ที่ทรงพลังสำหรับ type-safe database access ข้อดีหลักคือ auto-completion, type safety, และ readable query syntax Migration system ของ Prisma จัดการ schema changes ได้ดีมาก เหมาะสำหรับทั้ง prototyping และ production

> ขั้นตอนต่อไป: Part 77 - TypeScript กับ Node.js
