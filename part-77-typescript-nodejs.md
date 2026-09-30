# Part 77: TypeScript กับ Node.js
## ขั้นตอนที่ 761-770 จาก 1000

---

## ทำไมต้องใช้ TypeScript?

TypeScript เพิ่ม static type checking ให้กับ JavaScript ช่วยตรวจพบ bugs ก่อน runtime ทำให้โค้ดอ่านง่ายขึ้น และ refactor ได้ปลอดภัยกว่า

---

## 1. Setup TypeScript สำหรับ Node.js

```bash
npm init -y
npm install typescript ts-node @types/node
npm install express
npm install @types/express -D
npm install -D nodemon ts-node

npx tsc --init
```

### tsconfig.json

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "commonjs",
    "lib": ["ES2022"],
    "outDir": "./dist",
    "rootDir": "./src",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true,
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true,
    
    "baseUrl": ".",
    "paths": {
      "@/*": ["src/*"],
      "@controllers/*": ["src/controllers/*"],
      "@services/*": ["src/services/*"],
      "@models/*": ["src/models/*"]
    },
    
    "experimentalDecorators": true,
    "emitDecoratorMetadata": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist", "**/*.test.ts"]
}
```

### package.json scripts

```json
{
  "scripts": {
    "dev": "nodemon --exec ts-node src/index.ts",
    "build": "tsc",
    "start": "node dist/index.js",
    "typecheck": "tsc --noEmit"
  }
}
```

---

## 2. TypeScript Types สำหรับ Express

### Request/Response Types

```typescript
// types/express.d.ts (augment Express types)
import { User } from '../models/User';

declare global {
  namespace Express {
    interface Request {
      user?: User;
      tenant?: {
        id: string;
        name: string;
        slug: string;
      };
    }
  }
}

// src/types/api.types.ts
export interface ApiResponse<T = unknown> {
  success: boolean;
  data?: T;
  error?: string;
  message?: string;
  pagination?: Pagination;
}

export interface Pagination {
  page: number;
  limit: number;
  total: number;
  pages: number;
}

export interface QueryOptions {
  page?: number;
  limit?: number;
  sort?: string;
  search?: string;
  [key: string]: string | number | boolean | undefined;
}
```

### Typed Controllers

```typescript
// src/controllers/user.controller.ts
import { Request, Response, NextFunction } from 'express';
import { UserService } from '../services/user.service';
import { CreateUserDto, UpdateUserDto } from '../dto/user.dto';
import { ApiResponse, QueryOptions } from '../types/api.types';

export class UserController {
  constructor(private userService: UserService) {}

  getUsers = async (
    req: Request<{}, {}, {}, QueryOptions>,
    res: Response<ApiResponse<User[]>>,
    next: NextFunction
  ): Promise<void> => {
    try {
      const result = await this.userService.findAll(req.query);
      
      res.json({
        success: true,
        data: result.users,
        pagination: {
          page: result.page,
          limit: result.limit,
          total: result.total,
          pages: Math.ceil(result.total / result.limit)
        }
      });
    } catch (error) {
      next(error);
    }
  };

  getUserById = async (
    req: Request<{ id: string }>,
    res: Response<ApiResponse<User>>,
    next: NextFunction
  ): Promise<void> => {
    try {
      const user = await this.userService.findById(req.params.id);
      
      if (!user) {
        res.status(404).json({ success: false, error: 'User not found' });
        return;
      }

      res.json({ success: true, data: user });
    } catch (error) {
      next(error);
    }
  };

  createUser = async (
    req: Request<{}, {}, CreateUserDto>,
    res: Response<ApiResponse<User>>,
    next: NextFunction
  ): Promise<void> => {
    try {
      const user = await this.userService.create(req.body);
      res.status(201).json({ success: true, data: user });
    } catch (error) {
      next(error);
    }
  };
}
```

---

## 3. DTOs (Data Transfer Objects)

```typescript
// src/dto/user.dto.ts
import { IsEmail, IsString, MinLength, MaxLength, IsOptional, IsEnum } from 'class-validator';
import { Expose, Transform } from 'class-transformer';

export enum UserRole {
  USER = 'user',
  ADMIN = 'admin',
  MODERATOR = 'moderator'
}

export class CreateUserDto {
  @IsEmail()
  @Transform(({ value }) => value.toLowerCase().trim())
  email!: string;

  @IsString()
  @MinLength(2)
  @MaxLength(100)
  name!: string;

  @IsString()
  @MinLength(8)
  password!: string;

  @IsOptional()
  @IsEnum(UserRole)
  role?: UserRole = UserRole.USER;
}

export class UpdateUserDto {
  @IsOptional()
  @IsString()
  @MinLength(2)
  @MaxLength(100)
  name?: string;

  @IsOptional()
  @IsString()
  @MaxLength(160)
  bio?: string;
}

// ใช้งาน class-validator
import { validateOrReject, ValidationError } from 'class-validator';
import { plainToClass } from 'class-transformer';

async function validateDto<T extends object>(
  dtoClass: new () => T,
  data: unknown
): Promise<T> {
  const dto = plainToClass(dtoClass, data);
  
  try {
    await validateOrReject(dto);
  } catch (errors) {
    const messages = (errors as ValidationError[])
      .map(e => Object.values(e.constraints || {}).join(', '))
      .join('; ');
    throw new Error(`Validation failed: ${messages}`);
  }
  
  return dto;
}
```

---

## 4. Services กับ TypeScript

```typescript
// src/services/user.service.ts
import { prisma } from '../lib/prisma';
import { User, Prisma } from '@prisma/client';
import bcrypt from 'bcrypt';
import { CreateUserDto, UpdateUserDto } from '../dto/user.dto';

type UserWithProfile = Prisma.UserGetPayload<{
  include: { profile: true }
}>;

interface FindUsersOptions {
  page?: number;
  limit?: number;
  role?: string;
  search?: string;
}

interface FindUsersResult {
  users: UserWithProfile[];
  total: number;
  page: number;
  limit: number;
}

export class UserService {
  async findAll(options: FindUsersOptions = {}): Promise<FindUsersResult> {
    const { page = 1, limit = 20, role, search } = options;
    
    const where: Prisma.UserWhereInput = {};
    
    if (role) {
      where.role = role as Prisma.EnumRoleFilter;
    }
    
    if (search) {
      where.OR = [
        { name: { contains: search, mode: 'insensitive' } },
        { email: { contains: search, mode: 'insensitive' } }
      ];
    }

    const [users, total] = await prisma.$transaction([
      prisma.user.findMany({
        where,
        include: { profile: true },
        skip: (page - 1) * limit,
        take: limit,
        orderBy: { createdAt: 'desc' }
      }),
      prisma.user.count({ where })
    ]);

    return { users, total, page, limit };
  }

  async findById(id: string): Promise<UserWithProfile | null> {
    return prisma.user.findUnique({
      where: { id },
      include: { profile: true }
    });
  }

  async create(data: CreateUserDto): Promise<User> {
    const existing = await prisma.user.findUnique({
      where: { email: data.email }
    });

    if (existing) {
      throw new Error('Email already in use');
    }

    const hashedPassword = await bcrypt.hash(data.password, 12);

    return prisma.user.create({
      data: {
        email: data.email,
        name: data.name,
        password: hashedPassword,
        role: data.role
      }
    });
  }

  async update(id: string, data: UpdateUserDto): Promise<UserWithProfile> {
    return prisma.user.update({
      where: { id },
      data: {
        name: data.name,
        profile: {
          upsert: {
            create: { bio: data.bio },
            update: { bio: data.bio }
          }
        }
      },
      include: { profile: true }
    });
  }

  async delete(id: string): Promise<void> {
    await prisma.user.delete({ where: { id } });
  }
}
```

---

## 5. Decorators

```typescript
// decorators/validate.decorator.ts
import { Request, Response, NextFunction } from 'express';
import { validateOrReject } from 'class-validator';
import { plainToClass } from 'class-transformer';

export function ValidateBody<T extends object>(dtoClass: new () => T) {
  return function (target: any, propertyKey: string, descriptor: PropertyDescriptor) {
    const originalMethod = descriptor.value;
    
    descriptor.value = async function(
      req: Request,
      res: Response,
      next: NextFunction
    ) {
      try {
        const dto = plainToClass(dtoClass, req.body);
        await validateOrReject(dto);
        req.body = dto;
        return originalMethod.call(this, req, res, next);
      } catch (errors) {
        return res.status(400).json({ errors });
      }
    };
    
    return descriptor;
  };
}

// decorators/auth.decorator.ts
export function Roles(...roles: string[]) {
  return function (target: any, propertyKey: string, descriptor: PropertyDescriptor) {
    const originalMethod = descriptor.value;
    
    descriptor.value = async function(req: Request, res: Response, next: NextFunction) {
      if (!req.user || !roles.includes(req.user.role)) {
        return res.status(403).json({ error: 'Insufficient permissions' });
      }
      return originalMethod.call(this, req, res, next);
    };
    
    return descriptor;
  };
}

// ใช้งาน
class AdminController {
  @Roles('admin')
  @ValidateBody(CreateUserDto)
  async createUser(req: Request, res: Response) {
    // req.body จะเป็น CreateUserDto ที่ validated แล้ว
  }
}
```

---

## 6. Generic Repository Pattern

```typescript
// repositories/base.repository.ts
import { PrismaClient } from '@prisma/client';

interface BaseEntity {
  id: string;
  createdAt: Date;
  updatedAt: Date;
}

export abstract class BaseRepository<T extends BaseEntity, CreateInput, UpdateInput> {
  constructor(
    protected prisma: PrismaClient,
    protected model: keyof PrismaClient
  ) {}

  abstract findById(id: string): Promise<T | null>;
  abstract findAll(options?: Record<string, unknown>): Promise<T[]>;
  abstract create(data: CreateInput): Promise<T>;
  abstract update(id: string, data: UpdateInput): Promise<T>;
  abstract delete(id: string): Promise<void>;
}

// Error handling types
export class NotFoundError extends Error {
  constructor(resource: string, id: string) {
    super(`${resource} with id ${id} not found`);
    this.name = 'NotFoundError';
  }
}

export class ValidationError extends Error {
  constructor(public readonly errors: Record<string, string[]>) {
    super('Validation failed');
    this.name = 'ValidationError';
  }
}

// Type-safe error handler
export function handleError(error: unknown): never {
  if (error instanceof NotFoundError) {
    throw error;
  }
  if (error instanceof ValidationError) {
    throw error;
  }
  if (error instanceof Error) {
    throw new Error(`Unexpected error: ${error.message}`);
  }
  throw new Error('Unknown error occurred');
}
```

---

## แบบฝึกหัด

### ระดับ 1: พื้นฐาน
- Convert Express app ไปเป็น TypeScript
- ตั้งค่า tsconfig.json
- เพิ่ม types ให้ routes

### ระดับ 2: กลาง
- DTOs ด้วย class-validator
- Typed services
- Generic repository

### ระดับ 3: ขั้นสูง
- Decorators
- Type-safe Prisma queries
- Advanced generics

---

## สรุป

TypeScript ทำให้ Node.js development ปลอดภัยและ maintainable มากขึ้น โดยเฉพาะเมื่อ project โตขึ้น ควลทำ strict mode ตั้งแต่ต้น เพราะการเพิ่ม strict ทีหลังจะยุ่งยากมาก การใช้ DTOs ร่วมกับ class-validator ช่วยให้ validation ชัดเจนและ reusable

> ขั้นตอนต่อไป: Part 78 - E2E Testing กับ Playwright
