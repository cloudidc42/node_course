# Part 67 | ขั้นตอนที่ 1141-1160 จาก 1000+

# Prisma ORM

---

## 🎯 เป้าหมายของ Part นี้

หลังจากเรียนจบ Part นี้ คุณจะสามารถ:
1. ติดตั้งและกำหนดค่า Prisma
2. เขียน Prisma Schema และ migrations
3. ใช้ Prisma Client สำหรับ CRUD operations
4. เขียน advanced queries และ relations
5. ใช้ Prisma กับ NestJS
6. Optimize queries และ handle transactions

---

## ขั้นตอนที่ 1141: Prisma คืออะไร?

Prisma คือ next-generation ORM สำหรับ Node.js และ TypeScript ที่ให้ type-safe database access

```
Prisma Architecture:
┌─────────────────────────────────────────────────────┐
│                  Your Application                    │
│                                                      │
│  ┌──────────────────────────────────────────────┐   │
│  │              Prisma Client                    │   │
│  │   (Auto-generated, type-safe query builder)   │   │
│  └──────────────┬────────────────────────────────┘   │
│                 │                                     │
│  ┌──────────────▼────────────────────────────────┐   │
│  │              Prisma Query Engine               │   │
│  └──────────────┬────────────────────────────────┘   │
│                 │                                     │
│  ┌──────────────▼────────────────────────────────┐   │
│  │           Database (PostgreSQL/MySQL/SQLite)   │   │
│  └──────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────┘
```

### Prisma vs TypeORM vs Sequelize

| Feature | Prisma | TypeORM | Sequelize |
|---------|--------|---------|-----------|
| Type Safety | Excellent | Good | Limited |
| Schema Definition | Schema file | Decorators | Models |
| Migration | Auto | Complex | Auto |
| Query Builder | Intuitive | Verbose | Chainable |
| Performance | High | Medium | Medium |

---

## ขั้นตอนที่ 1142: การติดตั้ง Prisma

```bash
# สร้างโปรเจกต์ใหม่
mkdir prisma-app && cd prisma-app
npm init -y
npm install typescript ts-node @types/node --save-dev
npx tsc --init

# ติดตั้ง Prisma
npm install prisma --save-dev
npm install @prisma/client

# Initialize Prisma
npx prisma init

# โครงสร้างที่สร้าง:
# prisma/
# └── schema.prisma
# .env
```

```env
# .env
DATABASE_URL="postgresql://username:password@localhost:5432/mydb?schema=public"
```

---

## ขั้นตอนที่ 1143: Prisma Schema

```prisma
// prisma/schema.prisma

generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

// User model
model User {
  id        String   @id @default(uuid())
  email     String   @unique
  name      String
  password  String
  role      Role     @default(USER)
  isActive  Boolean  @default(true)
  profile   Profile?
  posts     Post[]
  orders    Order[]
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt

  @@index([email])
  @@map("users")
}

// Enum
enum Role {
  USER
  ADMIN
  MODERATOR
}

// Profile (one-to-one)
model Profile {
  id          String   @id @default(uuid())
  bio         String?
  avatarUrl   String?
  phoneNumber String?
  userId      String   @unique
  user        User     @relation(fields: [userId], references: [id], onDelete: Cascade)
  createdAt   DateTime @default(now())
  updatedAt   DateTime @updatedAt

  @@map("profiles")
}

// Post (one-to-many)
model Post {
  id          String     @id @default(uuid())
  title       String
  content     String
  published   Boolean    @default(false)
  viewCount   Int        @default(0)
  authorId    String
  author      User       @relation(fields: [authorId], references: [id])
  categories  Category[] @relation("PostCategory")
  tags        Tag[]
  createdAt   DateTime   @default(now())
  updatedAt   DateTime   @updatedAt

  @@index([authorId])
  @@index([published])
  @@map("posts")
}

// Category (many-to-many)
model Category {
  id    String @id @default(uuid())
  name  String @unique
  posts Post[] @relation("PostCategory")

  @@map("categories")
}

// Tag (implicit many-to-many)
model Tag {
  id    String @id @default(uuid())
  name  String @unique
  posts Post[]

  @@map("tags")
}

// Order
model Order {
  id         String      @id @default(uuid())
  userId     String
  user       User        @relation(fields: [userId], references: [id])
  status     OrderStatus @default(PENDING)
  total      Decimal     @db.Decimal(10, 2)
  items      OrderItem[]
  createdAt  DateTime    @default(now())
  updatedAt  DateTime    @updatedAt

  @@index([userId])
  @@index([status])
  @@map("orders")
}

enum OrderStatus {
  PENDING
  PROCESSING
  SHIPPED
  DELIVERED
  CANCELLED
}

// OrderItem
model OrderItem {
  id        String  @id @default(uuid())
  orderId   String
  order     Order   @relation(fields: [orderId], references: [id])
  productId String
  product   Product @relation(fields: [productId], references: [id])
  quantity  Int
  price     Decimal @db.Decimal(10, 2)

  @@map("order_items")
}

// Product
model Product {
  id          String      @id @default(uuid())
  name        String
  description String?
  price       Decimal     @db.Decimal(10, 2)
  stock       Int         @default(0)
  categoryId  String?
  category    ProductCategory? @relation(fields: [categoryId], references: [id])
  orderItems  OrderItem[]
  createdAt   DateTime    @default(now())
  updatedAt   DateTime    @updatedAt

  @@map("products")
}

// ProductCategory
model ProductCategory {
  id       String    @id @default(uuid())
  name     String    @unique
  products Product[]

  @@map("product_categories")
}
```

---

## ขั้นตอนที่ 1144: Migrations

```bash
# สร้าง migration จาก schema
npx prisma migrate dev --name init

# สร้าง migration โดยไม่ apply
npx prisma migrate dev --create-only --name add_user_profile

# Apply migration ใน production
npx prisma migrate deploy

# Reset database (development only)
npx prisma migrate reset

# ดูสถานะ migrations
npx prisma migrate status

# Generate Prisma Client หลังแก้ไข schema
npx prisma generate
```

```bash
# prisma/migrations/20240101000000_init/migration.sql (generated)
-- CreateEnum
CREATE TYPE "Role" AS ENUM ('USER', 'ADMIN', 'MODERATOR');

-- CreateTable
CREATE TABLE "users" (
    "id" TEXT NOT NULL,
    "email" TEXT NOT NULL,
    "name" TEXT NOT NULL,
    "password" TEXT NOT NULL,
    "role" "Role" NOT NULL DEFAULT 'USER',
    "isActive" BOOLEAN NOT NULL DEFAULT true,
    "createdAt" TIMESTAMP(3) NOT NULL DEFAULT CURRENT_TIMESTAMP,
    "updatedAt" TIMESTAMP(3) NOT NULL,

    CONSTRAINT "users_pkey" PRIMARY KEY ("id")
);

-- CreateIndex
CREATE UNIQUE INDEX "users_email_key" ON "users"("email");
CREATE INDEX "users_email_idx" ON "users"("email");
```

---

## ขั้นตอนที่ 1145: Prisma Client - Basic CRUD

```typescript
// src/database/prisma.service.ts
import { Injectable, OnModuleInit, OnModuleDestroy } from "@nestjs/common";
import { PrismaClient } from "@prisma/client";

@Injectable()
export class PrismaService extends PrismaClient implements OnModuleInit, OnModuleDestroy {
  constructor() {
    super({
      log: process.env.NODE_ENV === "development"
        ? ["query", "info", "warn", "error"]
        : ["error"]
    });
  }

  async onModuleInit() {
    await this.$connect();
  }

  async onModuleDestroy() {
    await this.$disconnect();
  }
}
```

```typescript
// src/users/users.service.ts
import { Injectable, NotFoundException, ConflictException } from "@nestjs/common";
import { PrismaService } from "../database/prisma.service";
import { User, Role } from "@prisma/client";
import * as bcrypt from "bcrypt";

@Injectable()
export class UsersService {
  constructor(private readonly prisma: PrismaService) {}

  // CREATE
  async create(data: {
    name: string;
    email: string;
    password: string;
  }): Promise<User> {
    const existing = await this.prisma.user.findUnique({
      where: { email: data.email }
    });
    
    if (existing) throw new ConflictException("Email already exists");
    
    const hashedPassword = await bcrypt.hash(data.password, 12);
    
    return this.prisma.user.create({
      data: {
        ...data,
        password: hashedPassword,
        profile: {
          create: {} // Create empty profile
        }
      },
      include: { profile: true }
    });
  }

  // READ
  async findAll(params: {
    skip?: number;
    take?: number;
    search?: string;
    role?: Role;
  }) {
    const { skip, take, search, role } = params;
    
    const where = {
      isActive: true,
      ...(role && { role }),
      ...(search && {
        OR: [
          { name: { contains: search, mode: "insensitive" as const } },
          { email: { contains: search, mode: "insensitive" as const } }
        ]
      })
    };
    
    const [users, total] = await this.prisma.$transaction([
      this.prisma.user.findMany({
        where,
        skip,
        take,
        orderBy: { createdAt: "desc" },
        select: {
          id: true,
          name: true,
          email: true,
          role: true,
          isActive: true,
          createdAt: true,
          profile: {
            select: { avatarUrl: true, bio: true }
          },
          _count: {
            select: { posts: true, orders: true }
          }
        }
      }),
      this.prisma.user.count({ where })
    ]);
    
    return { users, total };
  }

  async findById(id: string): Promise<User> {
    const user = await this.prisma.user.findUnique({
      where: { id },
      include: {
        profile: true,
        _count: { select: { posts: true, orders: true } }
      }
    });
    
    if (!user) throw new NotFoundException(`User ${id} not found`);
    
    return user;
  }

  // UPDATE
  async update(id: string, data: Partial<{
    name: string;
    email: string;
    role: Role;
  }>): Promise<User> {
    await this.findById(id);
    
    return this.prisma.user.update({
      where: { id },
      data: {
        ...data,
        updatedAt: new Date()
      }
    });
  }

  // DELETE (soft delete)
  async softDelete(id: string): Promise<void> {
    await this.findById(id);
    
    await this.prisma.user.update({
      where: { id },
      data: { isActive: false }
    });
  }

  // DELETE (hard delete)
  async delete(id: string): Promise<void> {
    await this.findById(id);
    await this.prisma.user.delete({ where: { id } });
  }
}
```

---

## ขั้นตอนที่ 1146: Advanced Queries

```typescript
// src/posts/posts.service.ts
import { Injectable } from "@nestjs/common";
import { PrismaService } from "../database/prisma.service";
import { Prisma } from "@prisma/client";

@Injectable()
export class PostsService {
  constructor(private readonly prisma: PrismaService) {}

  // Complex filtering
  async search(params: {
    query?: string;
    categoryIds?: string[];
    tagNames?: string[];
    authorId?: string;
    published?: boolean;
    page?: number;
    limit?: number;
    sortBy?: "createdAt" | "viewCount" | "title";
    sortOrder?: "asc" | "desc";
  }) {
    const {
      query, categoryIds, tagNames, authorId, published,
      page = 1, limit = 10, sortBy = "createdAt", sortOrder = "desc"
    } = params;
    
    const where: Prisma.PostWhereInput = {
      ...(published !== undefined && { published }),
      ...(authorId && { authorId }),
      ...(query && {
        OR: [
          { title: { contains: query, mode: "insensitive" } },
          { content: { contains: query, mode: "insensitive" } }
        ]
      }),
      ...(categoryIds?.length && {
        categories: {
          some: { id: { in: categoryIds } }
        }
      }),
      ...(tagNames?.length && {
        tags: {
          some: { name: { in: tagNames } }
        }
      })
    };
    
    const skip = (page - 1) * limit;
    
    const [posts, total] = await this.prisma.$transaction([
      this.prisma.post.findMany({
        where,
        skip,
        take: limit,
        orderBy: { [sortBy]: sortOrder },
        include: {
          author: {
            select: { id: true, name: true, profile: { select: { avatarUrl: true } } }
          },
          categories: true,
          tags: true,
          _count: { select: { tags: true } }
        }
      }),
      this.prisma.post.count({ where })
    ]);
    
    return {
      posts,
      total,
      page,
      limit,
      totalPages: Math.ceil(total / limit)
    };
  }

  // Aggregation
  async getStats(authorId: string) {
    const [postStats, viewStats] = await this.prisma.$transaction([
      this.prisma.post.groupBy({
        by: ["published"],
        where: { authorId },
        _count: { id: true }
      }),
      this.prisma.post.aggregate({
        where: { authorId },
        _sum: { viewCount: true },
        _avg: { viewCount: true },
        _max: { viewCount: true }
      })
    ]);
    
    return { postStats, viewStats };
  }

  // Raw query
  async getTopPosts(limit: number = 10) {
    return this.prisma.$queryRaw<Array<{
      id: string;
      title: string;
      viewCount: number;
      authorName: string;
    }>>`
      SELECT 
        p.id,
        p.title,
        p."viewCount",
        u.name as "authorName"
      FROM posts p
      JOIN users u ON p."authorId" = u.id
      WHERE p.published = true
      ORDER BY p."viewCount" DESC
      LIMIT ${limit}
    `;
  }

  // Upsert
  async incrementViewCount(postId: string): Promise<void> {
    await this.prisma.post.update({
      where: { id: postId },
      data: { viewCount: { increment: 1 } }
    });
  }
}
```

---

## ขั้นตอนที่ 1147: Relations

```typescript
// src/orders/orders.service.ts
import { Injectable, BadRequestException } from "@nestjs/common";
import { PrismaService } from "../database/prisma.service";
import { OrderStatus } from "@prisma/client";

interface CreateOrderDto {
  userId: string;
  items: Array<{
    productId: string;
    quantity: number;
  }>;
}

@Injectable()
export class OrdersService {
  constructor(private readonly prisma: PrismaService) {}

  async create(dto: CreateOrderDto) {
    return this.prisma.$transaction(async (tx) => {
      // Get products with stock
      const products = await tx.product.findMany({
        where: { id: { in: dto.items.map(i => i.productId) } }
      });
      
      // Validate stock
      for (const item of dto.items) {
        const product = products.find(p => p.id === item.productId);
        if (!product) {
          throw new BadRequestException(`Product ${item.productId} not found`);
        }
        if (product.stock < item.quantity) {
          throw new BadRequestException(
            `Insufficient stock for product ${product.name}`
          );
        }
      }
      
      // Calculate total
      const total = dto.items.reduce((sum, item) => {
        const product = products.find(p => p.id === item.productId)!;
        return sum + Number(product.price) * item.quantity;
      }, 0);
      
      // Create order with items
      const order = await tx.order.create({
        data: {
          userId: dto.userId,
          total,
          items: {
            create: dto.items.map(item => {
              const product = products.find(p => p.id === item.productId)!;
              return {
                productId: item.productId,
                quantity: item.quantity,
                price: product.price
              };
            })
          }
        },
        include: {
          items: {
            include: { product: true }
          },
          user: {
            select: { id: true, name: true, email: true }
          }
        }
      });
      
      // Update stock
      for (const item of dto.items) {
        await tx.product.update({
          where: { id: item.productId },
          data: { stock: { decrement: item.quantity } }
        });
      }
      
      return order;
    });
  }

  async findByUser(userId: string, status?: OrderStatus) {
    return this.prisma.order.findMany({
      where: {
        userId,
        ...(status && { status })
      },
      include: {
        items: {
          include: {
            product: {
              select: { id: true, name: true, price: true }
            }
          }
        }
      },
      orderBy: { createdAt: "desc" }
    });
  }

  async updateStatus(orderId: string, status: OrderStatus) {
    return this.prisma.order.update({
      where: { id: orderId },
      data: { status }
    });
  }
}
```

---

## ขั้นตอนที่ 1148: Transactions

```typescript
// src/database/transaction.service.ts
import { Injectable } from "@nestjs/common";
import { PrismaService } from "./prisma.service";
import { Prisma } from "@prisma/client";

@Injectable()
export class TransactionService {
  constructor(private readonly prisma: PrismaService) {}

  // Interactive transaction
  async transferBalance(
    fromUserId: string,
    toUserId: string,
    amount: number
  ): Promise<void> {
    await this.prisma.$transaction(async (tx) => {
      const fromUser = await tx.user.findUnique({
        where: { id: fromUserId },
        select: { id: true }
      });
      
      const toUser = await tx.user.findUnique({
        where: { id: toUserId },
        select: { id: true }
      });
      
      if (!fromUser || !toUser) {
        throw new Error("User not found");
      }
      
      // Perform operations
      // ... deduct from sender, add to receiver
    }, {
      maxWait: 5000,    // Maximum time to wait for a transaction slot
      timeout: 10000,   // Maximum time the transaction can run
      isolationLevel: Prisma.TransactionIsolationLevel.Serializable
    });
  }

  // Sequential transactions
  async batchCreate<T>(
    items: T[],
    createFn: (tx: Prisma.TransactionClient, item: T) => Promise<any>
  ) {
    return this.prisma.$transaction(
      items.map(item => createFn(this.prisma, item))
    );
  }
}
```

---

## ขั้นตอนที่ 1149: Middleware และ Soft Delete

```typescript
// prisma/middleware/soft-delete.middleware.ts
import { Prisma } from "@prisma/client";

export function softDeleteMiddleware(): Prisma.Middleware {
  return async (params, next) => {
    // Intercept delete operations
    if (params.action === "delete") {
      params.action = "update";
      params.args.data = { deletedAt: new Date() };
    }
    
    if (params.action === "deleteMany") {
      params.action = "updateMany";
      if (params.args.data !== undefined) {
        params.args.data.deletedAt = new Date();
      } else {
        params.args.data = { deletedAt: new Date() };
      }
    }
    
    // Filter out soft-deleted records
    if (params.action === "findUnique" || params.action === "findFirst") {
      params.action = "findFirst";
      params.args.where = {
        ...params.args.where,
        deletedAt: null
      };
    }
    
    if (params.action === "findMany") {
      if (params.args.where) {
        if (!params.args.where.deletedAt) {
          params.args.where.deletedAt = null;
        }
      } else {
        params.args.where = { deletedAt: null };
      }
    }
    
    return next(params);
  };
}
```

```typescript
// src/database/prisma.service.ts (with middleware)
import { PrismaClient } from "@prisma/client";
import { softDeleteMiddleware } from "../../prisma/middleware/soft-delete.middleware";

@Injectable()
export class PrismaService extends PrismaClient implements OnModuleInit {
  constructor() {
    super();
    this.$use(softDeleteMiddleware());
  }
  
  async onModuleInit() {
    await this.$connect();
  }
}
```

---

## ขั้นตอนที่ 1150: Seeding

```typescript
// prisma/seed.ts
import { PrismaClient, Role } from "@prisma/client";
import * as bcrypt from "bcrypt";

const prisma = new PrismaClient();

async function main() {
  console.log("Seeding database...");
  
  // Create admin user
  const adminPassword = await bcrypt.hash("Admin123!", 12);
  const admin = await prisma.user.upsert({
    where: { email: "admin@example.com" },
    update: {},
    create: {
      name: "Admin User",
      email: "admin@example.com",
      password: adminPassword,
      role: Role.ADMIN,
      profile: { create: { bio: "System Administrator" } }
    }
  });
  console.log("Admin created:", admin.id);
  
  // Create product categories
  const categories = await Promise.all([
    prisma.productCategory.upsert({
      where: { name: "Electronics" },
      update: {},
      create: { name: "Electronics" }
    }),
    prisma.productCategory.upsert({
      where: { name: "Clothing" },
      update: {},
      create: { name: "Clothing" }
    }),
    prisma.productCategory.upsert({
      where: { name: "Books" },
      update: {},
      create: { name: "Books" }
    })
  ]);
  
  // Create products
  const products = await Promise.all([
    prisma.product.create({
      data: {
        name: "Laptop Pro",
        description: "High-performance laptop",
        price: 1299.99,
        stock: 50,
        categoryId: categories[0].id
      }
    }),
    prisma.product.create({
      data: {
        name: "TypeScript Handbook",
        description: "Complete guide to TypeScript",
        price: 49.99,
        stock: 100,
        categoryId: categories[2].id
      }
    })
  ]);
  
  console.log(`Created ${products.length} products`);
  console.log("Seeding completed!");
}

main()
  .catch(console.error)
  .finally(() => prisma.$disconnect());
```

```json
// package.json
{
  "prisma": {
    "seed": "ts-node prisma/seed.ts"
  }
}
```

```bash
# Run seed
npx prisma db seed
```

---

## ขั้นตอนที่ 1151: Prisma กับ NestJS

```typescript
// src/database/database.module.ts
import { Global, Module } from "@nestjs/common";
import { PrismaService } from "./prisma.service";

@Global()
@Module({
  providers: [PrismaService],
  exports: [PrismaService]
})
export class DatabaseModule {}
```

```typescript
// src/products/products.service.ts (Full example)
import { Injectable, NotFoundException } from "@nestjs/common";
import { PrismaService } from "../database/prisma.service";
import { Prisma, Product } from "@prisma/client";

type ProductWithCategory = Prisma.ProductGetPayload<{
  include: { category: true }
}>;

@Injectable()
export class ProductsService {
  constructor(private readonly prisma: PrismaService) {}

  async findAll(params: {
    page?: number;
    limit?: number;
    categoryId?: string;
    minPrice?: number;
    maxPrice?: number;
    inStock?: boolean;
    search?: string;
  }): Promise<{ products: ProductWithCategory[]; total: number }> {
    const {
      page = 1, limit = 20, categoryId,
      minPrice, maxPrice, inStock, search
    } = params;
    
    const where: Prisma.ProductWhereInput = {
      ...(categoryId && { categoryId }),
      ...(inStock && { stock: { gt: 0 } }),
      ...(minPrice !== undefined || maxPrice !== undefined) && {
        price: {
          ...(minPrice !== undefined && { gte: minPrice }),
          ...(maxPrice !== undefined && { lte: maxPrice })
        }
      },
      ...(search && {
        OR: [
          { name: { contains: search, mode: "insensitive" } },
          { description: { contains: search, mode: "insensitive" } }
        ]
      })
    };
    
    const [products, total] = await this.prisma.$transaction([
      this.prisma.product.findMany({
        where,
        skip: (page - 1) * limit,
        take: limit,
        include: { category: true },
        orderBy: { createdAt: "desc" }
      }),
      this.prisma.product.count({ where })
    ]);
    
    return { products, total };
  }

  async findById(id: string): Promise<ProductWithCategory> {
    const product = await this.prisma.product.findUnique({
      where: { id },
      include: { category: true }
    });
    
    if (!product) throw new NotFoundException(`Product ${id} not found`);
    
    return product;
  }

  async create(data: Prisma.ProductCreateInput): Promise<Product> {
    return this.prisma.product.create({ data });
  }

  async update(id: string, data: Prisma.ProductUpdateInput): Promise<Product> {
    await this.findById(id);
    return this.prisma.product.update({ where: { id }, data });
  }

  async delete(id: string): Promise<void> {
    await this.findById(id);
    await this.prisma.product.delete({ where: { id } });
  }
}
```

---

## ขั้นตอนที่ 1152: Query Optimization

```typescript
// src/common/utils/prisma-pagination.ts
import { PrismaService } from "../database/prisma.service";

export interface PaginationParams {
  page?: number;
  limit?: number;
  cursor?: string;  // Cursor-based pagination
}

// Offset pagination
export async function paginateOffset<T>(
  findMany: () => Promise<T[]>,
  count: () => Promise<number>,
  page: number = 1,
  limit: number = 10
) {
  const [data, total] = await Promise.all([findMany(), count()]);
  
  return {
    data,
    meta: {
      total,
      page,
      limit,
      totalPages: Math.ceil(total / limit),
      hasNext: page < Math.ceil(total / limit),
      hasPrev: page > 1
    }
  };
}

// Cursor pagination (better for large datasets)
export async function paginateCursor<T extends { id: string }>(
  model: any,
  params: {
    where?: any;
    include?: any;
    orderBy?: any;
    cursor?: string;
    take?: number;
  }
) {
  const { cursor, take = 20, ...rest } = params;
  
  const results = await model.findMany({
    ...rest,
    take: take + 1,  // Fetch one extra to check if there's next page
    ...(cursor && {
      cursor: { id: cursor },
      skip: 1  // Skip the cursor item
    })
  });
  
  const hasNextPage = results.length > take;
  if (hasNextPage) results.pop();
  
  const nextCursor = hasNextPage ? results[results.length - 1]?.id : undefined;
  
  return {
    data: results,
    meta: {
      hasNextPage,
      nextCursor
    }
  };
}
```

---

## ขั้นตอนที่ 1153: Prisma Studio และ GUI Tools

```bash
# เปิด Prisma Studio (web-based GUI)
npx prisma studio

# รันที่ http://localhost:5555
# สามารถ browse, edit, add, delete data ได้

# Generate ER Diagram
npm install --save-dev prisma-erd-generator
```

```prisma
// prisma/schema.prisma (add ERD generator)
generator erd {
  provider = "prisma-erd-generator"
  output   = "./ERD.md"
}
```

---

## ขั้นตอนที่ 1154: Testing กับ Prisma

```typescript
// src/database/prisma.mock.ts
import { PrismaClient } from "@prisma/client";
import { mockDeep, mockReset, DeepMockProxy } from "jest-mock-extended";

export type Context = {
  prisma: PrismaClient;
};

export type MockContext = {
  prisma: DeepMockProxy<PrismaClient>;
};

export const createMockContext = (): MockContext => {
  return {
    prisma: mockDeep<PrismaClient>()
  };
};
```

```typescript
// src/products/__tests__/products.service.spec.ts
import { MockContext, createMockContext } from "../../database/prisma.mock";
import { ProductsService } from "../products.service";

describe("ProductsService", () => {
  let service: ProductsService;
  let mockCtx: MockContext;

  beforeEach(() => {
    mockCtx = createMockContext();
    service = new ProductsService(mockCtx.prisma as any);
  });

  it("should find product by id", async () => {
    const mockProduct = {
      id: "1",
      name: "Test Product",
      price: 99.99,
      stock: 10,
      description: null,
      categoryId: null,
      createdAt: new Date(),
      updatedAt: new Date(),
      category: null
    };
    
    mockCtx.prisma.product.findUnique.mockResolvedValue(mockProduct as any);
    
    const result = await service.findById("1");
    
    expect(result).toEqual(mockProduct);
    expect(mockCtx.prisma.product.findUnique).toHaveBeenCalledWith({
      where: { id: "1" },
      include: { category: true }
    });
  });
});
```

---

## ขั้นตอนที่ 1155: Multiple Database Support

```prisma
// prisma/schema.prisma (multiple databases)

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

// For read replica
datasource readDb {
  provider = "postgresql"
  url      = env("READ_DATABASE_URL")
}
```

```typescript
// src/database/read-replica.service.ts
import { Injectable } from "@nestjs/common";
import { PrismaClient } from "@prisma/client";

@Injectable()
export class ReadReplicaService extends PrismaClient {
  constructor() {
    super({
      datasources: {
        db: {
          url: process.env.READ_DATABASE_URL
        }
      }
    });
  }
}
```

---

## ขั้นตอนที่ 1156: Full-text Search กับ Prisma

```prisma
// schema.prisma - Adding full text search
model Post {
  id      String @id @default(uuid())
  title   String
  content String
  
  // PostgreSQL full-text search
  @@index([title, content], type: BTree)
}
```

```typescript
// Full text search example
async searchPosts(query: string) {
  // Using raw query for full-text search
  return this.prisma.$queryRaw<Post[]>`
    SELECT *
    FROM posts
    WHERE to_tsvector('english', title || ' ' || content)
    @@ plainto_tsquery('english', ${query})
    ORDER BY ts_rank(
      to_tsvector('english', title || ' ' || content),
      plainto_tsquery('english', ${query})
    ) DESC
    LIMIT 20
  `;
}
```

---

## ขั้นตอนที่ 1157: Database Views กับ Prisma

```sql
-- prisma/migrations/xxx_create_views/migration.sql
CREATE VIEW user_stats AS
SELECT
  u.id,
  u.name,
  u.email,
  COUNT(DISTINCT p.id) as post_count,
  COUNT(DISTINCT o.id) as order_count,
  COALESCE(SUM(o.total), 0) as total_spent
FROM users u
LEFT JOIN posts p ON p."authorId" = u.id
LEFT JOIN orders o ON o."userId" = u.id
GROUP BY u.id;
```

```prisma
// schema.prisma
model UserStats {
  id         String @id
  name       String
  email      String
  postCount  Int
  orderCount Int
  totalSpent Decimal

  @@map("user_stats")
}
```

---

## ขั้นตอนที่ 1158: Prisma Extensions

```typescript
// Prisma Client Extensions (Prisma 4.7+)
import { PrismaClient } from "@prisma/client";

const prisma = new PrismaClient().$extends({
  // Add custom methods
  model: {
    user: {
      async findByEmail(email: string) {
        return prisma.user.findUnique({ where: { email } });
      },
      
      async findActive() {
        return prisma.user.findMany({ where: { isActive: true } });
      }
    }
  },
  
  // Computed fields
  result: {
    user: {
      fullName: {
        needs: { name: true },
        compute(user) {
          return `${user.name}`;
        }
      }
    }
  },
  
  // Query extensions
  query: {
    user: {
      async findMany({ args, query }) {
        // Auto-filter deleted users
        args.where = { ...args.where, isActive: true };
        return query(args);
      }
    }
  }
});

export default prisma;
```

---

## ขั้นตอนที่ 1159: Performance Optimization

```typescript
// Select only needed fields
const users = await prisma.user.findMany({
  select: {
    id: true,
    name: true,
    email: true
    // Omit password and other sensitive fields
  }
});

// Use includes wisely - avoid N+1
const postsWithAuthors = await prisma.post.findMany({
  include: {
    author: {
      select: { id: true, name: true }
    }
  }
});

// Use batch operations
await prisma.$transaction([
  prisma.user.updateMany({ where: { role: "USER" }, data: { isActive: true } }),
  prisma.post.updateMany({ where: { published: false }, data: { viewCount: 0 } })
]);

// Connection pooling
const prisma = new PrismaClient({
  datasources: {
    db: {
      url: `${process.env.DATABASE_URL}?connection_limit=10&pool_timeout=30`
    }
  }
});

// Use count instead of findMany for existence checks
const exists = await prisma.user.count({ where: { email } }) > 0;

// Use cursor pagination for large datasets
const result = await prisma.post.findMany({
  take: 20,
  cursor: { id: lastPostId },
  skip: 1,
  orderBy: { id: "asc" }
});
```

---

## ขั้นตอนที่ 1160: Database Schema Versioning

```bash
# สร้าง schema.prisma สำหรับหลาย environments
# prisma/schema.dev.prisma
# prisma/schema.prod.prisma

# ใช้ environment variable ชี้ไปยัง schema ที่ถูกต้อง
DATABASE_URL="postgresql://..."

# Migration workflow
git checkout -b feature/add-user-profile

# แก้ไข schema.prisma
# เพิ่ม model ใหม่หรือ field ใหม่

npx prisma migrate dev --name add_user_stats

# Review migration
cat prisma/migrations/xxx_add_user_stats/migration.sql

# Commit migration
git add prisma/
git commit -m "feat: add user stats"

# Deploy to production
npx prisma migrate deploy
```

---

## 🏋️ แบบฝึกหัด

### แบบฝึกหัดที่ 1: Schema Design
1. ออกแบบ schema สำหรับ e-commerce platform
2. include: Users, Products, Orders, Reviews, Categories
3. กำหนด relations ที่เหมาะสม

### แบบฝึกหัดที่ 2: Complex Queries
1. สร้าง query หา top 10 products ที่ขายดีที่สุด
2. สร้าง query หา users ที่ active มากที่สุดในช่วง 30 วัน
3. สร้าง aggregation สำหรับ sales report รายเดือน

### แบบฝึกหัดที่ 3: Transactions
1. implement cart checkout flow ด้วย transactions
2. handle race conditions สำหรับ stock management
3. implement rollback scenarios

---

## 📚 สรุป

ใน Part นี้เราได้เรียนรู้:
- การติดตั้งและกำหนดค่า Prisma
- Schema definition และ Migrations
- CRUD operations ด้วย Prisma Client
- Advanced queries, relations, aggregations
- Transactions และ Soft Delete
- Testing patterns
- Performance optimization

**Part ถัดไป**: Elasticsearch
