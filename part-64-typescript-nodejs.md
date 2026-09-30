# Part 64 | ขั้นตอนที่ 1081-1100 จาก 1000+

# TypeScript กับ Node.js

---

## 🎯 เป้าหมายของ Part นี้

หลังจากเรียนจบ Part นี้ คุณจะสามารถ:
1. ติดตั้งและกำหนดค่า TypeScript สำหรับโปรเจกต์ Node.js
2. เข้าใจ tsconfig.json และตัวเลือกต่างๆ
3. ใช้ Type Definitions และ @types packages
4. เขียน Decorators และ Metadata ใน TypeScript
5. แปลง JavaScript โปรเจกต์เป็น TypeScript
6. ใช้ Advanced Types สำหรับ Express.js

---

## ขั้นตอนที่ 1081: ทำไมต้องใช้ TypeScript กับ Node.js?

TypeScript คือ superset ของ JavaScript ที่เพิ่ม static type checking เข้ามา ช่วยให้โค้ดมีความน่าเชื่อถือมากขึ้นและหาบัคได้ง่ายขึ้นก่อน runtime

```
JavaScript vs TypeScript:
┌─────────────────────────────────────────────────────┐
│  JavaScript                 TypeScript               │
│  ─────────────────────────  ─────────────────────── │
│  Dynamic typing             Static typing            │
│  Runtime errors             Compile-time errors      │
│  No IDE support             Full IntelliSense        │
│  No interfaces              Interfaces & types       │
│  No decorators (legacy)     Modern decorators        │
└─────────────────────────────────────────────────────┘
```

### ประโยชน์ของ TypeScript

1. **Type Safety** - ตรวจสอบ type ก่อน runtime
2. **Better IDE Support** - autocomplete และ refactoring
3. **Clearer Code** - interfaces ทำให้โค้ดอ่านง่ายขึ้น
4. **Easier Refactoring** - เปลี่ยนชื่อตัวแปรได้ปลอดภัย
5. **Modern Features** - ใช้ features ใหม่ๆ ได้เลย

---

## ขั้นตอนที่ 1082: การติดตั้ง TypeScript

```bash
# สร้างโปรเจกต์ใหม่
mkdir typescript-node-app
cd typescript-node-app
npm init -y

# ติดตั้ง TypeScript
npm install --save-dev typescript

# ติดตั้ง Node.js type definitions
npm install --save-dev @types/node

# ติดตั้ง ts-node สำหรับรัน TypeScript โดยตรง
npm install --save-dev ts-node

# ติดตั้ง nodemon สำหรับ development
npm install --save-dev nodemon

# สร้าง tsconfig.json
npx tsc --init
```

### ตรวจสอบการติดตั้ง

```bash
npx tsc --version
# TypeScript 5.x.x

npx ts-node --version
# v10.x.x
```

---

## ขั้นตอนที่ 1083: การกำหนดค่า tsconfig.json

```json
{
  "compilerOptions": {
    // Target JavaScript version
    "target": "ES2022",
    
    // Module system
    "module": "commonjs",
    
    // Module resolution strategy
    "moduleResolution": "node",
    
    // Output directory
    "outDir": "./dist",
    
    // Root directory of source files
    "rootDir": "./src",
    
    // Enable strict type checking
    "strict": true,
    
    // Allow importing JSON files
    "resolveJsonModule": true,
    
    // Generate source maps for debugging
    "sourceMap": true,
    
    // Enable experimental decorators
    "experimentalDecorators": true,
    
    // Emit decorator metadata
    "emitDecoratorMetadata": true,
    
    // Skip type checking of declaration files
    "skipLibCheck": true,
    
    // Allow default imports from modules without default export
    "esModuleInterop": true,
    
    // Allow importing modules with .json extension
    "allowSyntheticDefaultImports": true,
    
    // Include declaration files
    "declaration": true,
    
    // Generate declaration map files
    "declarationMap": true,
    
    // Incremental compilation
    "incremental": true,
    
    // TypeScript cache directory
    "tsBuildInfoFile": "./.tsbuildinfo"
  },
  "include": [
    "src/**/*"
  ],
  "exclude": [
    "node_modules",
    "dist",
    "**/*.test.ts"
  ]
}
```

### tsconfig สำหรับ Production

```json
{
  "extends": "./tsconfig.json",
  "compilerOptions": {
    "sourceMap": false,
    "removeComments": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "noImplicitReturns": true,
    "noFallthroughCasesInSwitch": true
  }
}
```

---

## ขั้นตอนที่ 1084: Basic Types ใน TypeScript

```typescript
// src/basics/types.ts

// Primitive types
const name: string = "John";
const age: number = 30;
const isActive: boolean = true;
const nothing: null = null;
const undef: undefined = undefined;

// Arrays
const numbers: number[] = [1, 2, 3];
const strings: Array<string> = ["a", "b", "c"];

// Tuples
const point: [number, number] = [10, 20];
const namedPoint: [x: number, y: number] = [10, 20];

// Union types
let id: string | number = "abc123";
id = 123; // ได้เช่นกัน

// Intersection types
type Admin = { role: "admin" };
type User = { name: string; email: string };
type AdminUser = Admin & User;

const adminUser: AdminUser = {
  role: "admin",
  name: "Alice",
  email: "alice@example.com"
};

// Type assertions
const someValue: unknown = "hello world";
const strLength: number = (someValue as string).length;

// Literal types
type Direction = "north" | "south" | "east" | "west";
let dir: Direction = "north";

// Optional properties
interface Config {
  host: string;
  port?: number;  // Optional
  ssl?: boolean;  // Optional
}

const config: Config = { host: "localhost" };

// Readonly properties
interface Point {
  readonly x: number;
  readonly y: number;
}

const pt: Point = { x: 10, y: 20 };
// pt.x = 5; // Error! Cannot assign to 'x' because it is a read-only property.

// Record type
const userMap: Record<string, User> = {
  "user1": { name: "Alice", email: "alice@example.com" },
  "user2": { name: "Bob", email: "bob@example.com" }
};

// Partial type
function updateUser(user: User, updates: Partial<User>): User {
  return { ...user, ...updates };
}

// Required type
type RequiredConfig = Required<Config>;

// Pick type
type UserEmail = Pick<User, "email">;

// Omit type
type UserWithoutEmail = Omit<User, "email">;

// Exclude type
type StringOrNumber = string | number | boolean;
type OnlyStringOrNumber = Exclude<StringOrNumber, boolean>;

// Extract type
type OnlyString = Extract<StringOrNumber, string>;

// ReturnType
function getUser(): User {
  return { name: "Alice", email: "alice@example.com" };
}
type GetUserReturn = ReturnType<typeof getUser>; // User

// Parameters type
function createUser(name: string, email: string, age: number): User {
  return { name, email };
}
type CreateUserParams = Parameters<typeof createUser>;
// [name: string, email: string, age: number]
```

---

## ขั้นตอนที่ 1085: Interfaces และ Classes

```typescript
// src/models/user.ts

// Interface definition
interface IUser {
  id: string;
  name: string;
  email: string;
  createdAt: Date;
  updatedAt: Date;
}

// Interface extending
interface IAdminUser extends IUser {
  role: "admin" | "super-admin";
  permissions: string[];
}

// Interface for methods
interface IUserRepository {
  findById(id: string): Promise<IUser | null>;
  findByEmail(email: string): Promise<IUser | null>;
  save(user: IUser): Promise<IUser>;
  delete(id: string): Promise<boolean>;
}

// Class implementing interface
class UserRepository implements IUserRepository {
  private users: Map<string, IUser> = new Map();

  async findById(id: string): Promise<IUser | null> {
    return this.users.get(id) ?? null;
  }

  async findByEmail(email: string): Promise<IUser | null> {
    for (const user of this.users.values()) {
      if (user.email === email) {
        return user;
      }
    }
    return null;
  }

  async save(user: IUser): Promise<IUser> {
    this.users.set(user.id, user);
    return user;
  }

  async delete(id: string): Promise<boolean> {
    return this.users.delete(id);
  }
}

// Abstract class
abstract class BaseModel {
  abstract validate(): boolean;
  
  toJSON(): object {
    return { ...this };
  }
}

// Generic class
class Repository<T extends { id: string }> {
  protected items: Map<string, T> = new Map();

  async findById(id: string): Promise<T | null> {
    return this.items.get(id) ?? null;
  }

  async save(item: T): Promise<T> {
    this.items.set(item.id, item);
    return item;
  }

  async findAll(): Promise<T[]> {
    return Array.from(this.items.values());
  }

  async delete(id: string): Promise<boolean> {
    return this.items.delete(id);
  }
}

// Using generic class
class ProductRepository extends Repository<IProduct> {
  async findByCategory(category: string): Promise<IProduct[]> {
    return (await this.findAll()).filter(p => p.category === category);
  }
}

interface IProduct {
  id: string;
  name: string;
  price: number;
  category: string;
}
```

---

## ขั้นตอนที่ 1086: TypeScript กับ Express.js

```typescript
// src/types/express.d.ts - Extending Express types

import { IUser } from "../models/user";

declare global {
  namespace Express {
    interface Request {
      user?: IUser;
      correlationId?: string;
    }
  }
}

export {};
```

```typescript
// src/middleware/auth.ts

import { Request, Response, NextFunction } from "express";
import jwt from "jsonwebtoken";
import { IUser } from "../models/user";

interface JwtPayload {
  userId: string;
  email: string;
  iat: number;
  exp: number;
}

export const authMiddleware = async (
  req: Request,
  res: Response,
  next: NextFunction
): Promise<void> => {
  try {
    const token = req.headers.authorization?.split(" ")[1];
    
    if (!token) {
      res.status(401).json({ error: "No token provided" });
      return;
    }

    const decoded = jwt.verify(token, process.env.JWT_SECRET!) as JwtPayload;
    
    // Attach user to request
    req.user = {
      id: decoded.userId,
      email: decoded.email,
      name: "",
      createdAt: new Date(),
      updatedAt: new Date()
    };
    
    next();
  } catch (error) {
    res.status(401).json({ error: "Invalid token" });
  }
};
```

```typescript
// src/controllers/user.controller.ts

import { Request, Response } from "express";
import { UserService } from "../services/user.service";

export class UserController {
  constructor(private readonly userService: UserService) {}

  async getUser(req: Request, res: Response): Promise<void> {
    try {
      const { id } = req.params;
      const user = await this.userService.findById(id);
      
      if (!user) {
        res.status(404).json({ error: "User not found" });
        return;
      }
      
      res.json(user);
    } catch (error) {
      res.status(500).json({ error: "Internal server error" });
    }
  }

  async createUser(req: Request, res: Response): Promise<void> {
    try {
      const { name, email } = req.body as { name: string; email: string };
      const user = await this.userService.create({ name, email });
      res.status(201).json(user);
    } catch (error) {
      if (error instanceof ValidationError) {
        res.status(400).json({ error: error.message });
      } else {
        res.status(500).json({ error: "Internal server error" });
      }
    }
  }
}

class ValidationError extends Error {
  constructor(message: string) {
    super(message);
    this.name = "ValidationError";
  }
}
```

---

## ขั้นตอนที่ 1087: Generics ขั้นสูง

```typescript
// src/utils/generics.ts

// Generic function
function first<T>(arr: T[]): T | undefined {
  return arr[0];
}

const firstNumber = first([1, 2, 3]); // number | undefined
const firstString = first(["a", "b"]); // string | undefined

// Generic constraints
function getProperty<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key];
}

const user = { name: "Alice", age: 30, email: "alice@example.com" };
const userName = getProperty(user, "name"); // string
const userAge = getProperty(user, "age");   // number

// Conditional types
type NonNullable<T> = T extends null | undefined ? never : T;

type ApiResponse<T> = {
  data: T;
  status: number;
  message: string;
  timestamp: Date;
};

// Mapped types
type Nullable<T> = { [K in keyof T]: T[K] | null };
type ReadonlyUser = Readonly<IUser>;

// Template literal types
type EventName = "click" | "focus" | "blur";
type EventHandler = `on${Capitalize<EventName>}`;
// "onClick" | "onFocus" | "onBlur"

// Infer keyword
type UnpackPromise<T> = T extends Promise<infer U> ? U : T;

type UserPromise = Promise<IUser>;
type UnpackedUser = UnpackPromise<UserPromise>; // IUser

// Recursive types
type DeepPartial<T> = {
  [K in keyof T]?: T[K] extends object ? DeepPartial<T[K]> : T[K];
};

interface Config {
  server: {
    host: string;
    port: number;
    ssl: {
      enabled: boolean;
      cert: string;
    };
  };
  database: {
    url: string;
    maxConnections: number;
  };
}

const partialConfig: DeepPartial<Config> = {
  server: {
    port: 3000
  }
};

interface IUser {
  id: string;
  name: string;
  email: string;
  createdAt: Date;
  updatedAt: Date;
}
```

---

## ขั้นตอนที่ 1088: Decorators ใน TypeScript

```typescript
// src/decorators/index.ts

// Class Decorator
function Injectable(constructor: Function) {
  Reflect.defineMetadata("injectable", true, constructor);
}

// Method Decorator
function Log(
  target: any,
  propertyKey: string,
  descriptor: PropertyDescriptor
) {
  const originalMethod = descriptor.value;
  
  descriptor.value = async function (...args: any[]) {
    console.log(`Calling ${propertyKey} with args:`, args);
    const result = await originalMethod.apply(this, args);
    console.log(`${propertyKey} returned:`, result);
    return result;
  };
  
  return descriptor;
}

// Property Decorator
function Required(target: any, propertyKey: string) {
  Reflect.defineMetadata("required", true, target, propertyKey);
}

// Parameter Decorator
function Param(
  target: any,
  propertyKey: string | symbol,
  parameterIndex: number
) {
  const existingParams = Reflect.getMetadata("params", target, propertyKey) || [];
  existingParams.push(parameterIndex);
  Reflect.defineMetadata("params", existingParams, target, propertyKey);
}

// Decorator Factory
function Validate(schema: object) {
  return function (
    target: any,
    propertyKey: string,
    descriptor: PropertyDescriptor
  ) {
    const originalMethod = descriptor.value;
    
    descriptor.value = function (...args: any[]) {
      // Validate args against schema
      console.log("Validating with schema:", schema);
      return originalMethod.apply(this, args);
    };
    
    return descriptor;
  };
}

// Route Decorators (like NestJS)
function Controller(prefix: string) {
  return function (constructor: Function) {
    Reflect.defineMetadata("prefix", prefix, constructor);
  };
}

function Get(path: string = "") {
  return function (target: any, propertyKey: string, descriptor: PropertyDescriptor) {
    const routes = Reflect.getMetadata("routes", target.constructor) || [];
    routes.push({ method: "GET", path, handler: propertyKey });
    Reflect.defineMetadata("routes", routes, target.constructor);
    return descriptor;
  };
}

function Post(path: string = "") {
  return function (target: any, propertyKey: string, descriptor: PropertyDescriptor) {
    const routes = Reflect.getMetadata("routes", target.constructor) || [];
    routes.push({ method: "POST", path, handler: propertyKey });
    Reflect.defineMetadata("routes", routes, target.constructor);
    return descriptor;
  };
}

// Using decorators
@Injectable
@Controller("/users")
class UserController {
  @Required
  private userService: UserService;

  constructor(userService: UserService) {
    this.userService = userService;
  }

  @Get("/:id")
  @Log
  async getUser(req: Request, res: Response): Promise<void> {
    const user = await this.userService.findById(req.params.id);
    res.json(user);
  }

  @Post("/")
  @Validate({ name: "string", email: "string" })
  async createUser(req: Request, res: Response): Promise<void> {
    const user = await this.userService.create(req.body);
    res.status(201).json(user);
  }
}

class UserService {
  async findById(id: string): Promise<any> { return null; }
  async create(data: any): Promise<any> { return data; }
}

import { Request, Response } from "express";
```

---

## ขั้นตอนที่ 1089: Type Definitions สำหรับ third-party packages

```typescript
// src/types/custom.d.ts

// เพิ่ม type สำหรับ library ที่ไม่มี @types

declare module "some-untyped-library" {
  export function doSomething(input: string): Promise<string>;
  export class SomeClass {
    constructor(options?: { timeout?: number });
    connect(): void;
    disconnect(): void;
  }
}

// Augment existing types
declare module "express-serve-static-core" {
  interface Request {
    user?: {
      id: string;
      email: string;
      roles: string[];
    };
    startTime?: number;
  }
}

// Global type augmentation
declare global {
  interface Window {
    myApp: {
      version: string;
      config: Record<string, unknown>;
    };
  }
  
  namespace NodeJS {
    interface ProcessEnv {
      NODE_ENV: "development" | "production" | "test";
      PORT: string;
      DATABASE_URL: string;
      JWT_SECRET: string;
      REDIS_URL?: string;
    }
  }
}

export {};
```

```bash
# ติดตั้ง @types packages
npm install --save-dev @types/express
npm install --save-dev @types/node
npm install --save-dev @types/jest
npm install --save-dev @types/bcrypt
npm install --save-dev @types/jsonwebtoken
npm install --save-dev @types/cors
npm install --save-dev @types/morgan
```

---

## ขั้นตอนที่ 1090: การจัดการ Error ด้วย TypeScript

```typescript
// src/errors/index.ts

// Custom error classes
export class AppError extends Error {
  constructor(
    public readonly message: string,
    public readonly statusCode: number,
    public readonly code: string,
    public readonly isOperational: boolean = true
  ) {
    super(message);
    this.name = this.constructor.name;
    Error.captureStackTrace(this, this.constructor);
  }
}

export class NotFoundError extends AppError {
  constructor(resource: string, id?: string) {
    super(
      id ? `${resource} with id ${id} not found` : `${resource} not found`,
      404,
      "NOT_FOUND"
    );
  }
}

export class ValidationError extends AppError {
  constructor(
    message: string,
    public readonly errors: ValidationErrorDetail[]
  ) {
    super(message, 400, "VALIDATION_ERROR");
  }
}

export class UnauthorizedError extends AppError {
  constructor(message: string = "Unauthorized") {
    super(message, 401, "UNAUTHORIZED");
  }
}

export class ForbiddenError extends AppError {
  constructor(message: string = "Forbidden") {
    super(message, 403, "FORBIDDEN");
  }
}

interface ValidationErrorDetail {
  field: string;
  message: string;
}

// Error handler middleware
import { Request, Response, NextFunction } from "express";

export function errorHandler(
  error: Error | AppError,
  req: Request,
  res: Response,
  next: NextFunction
): void {
  if (error instanceof AppError) {
    res.status(error.statusCode).json({
      error: {
        code: error.code,
        message: error.message,
        ...(error instanceof ValidationError && { errors: error.errors })
      }
    });
    return;
  }

  // Unknown errors
  console.error("Unexpected error:", error);
  res.status(500).json({
    error: {
      code: "INTERNAL_ERROR",
      message: "An unexpected error occurred"
    }
  });
}

// Result type pattern
type Result<T, E = Error> = 
  | { success: true; data: T }
  | { success: false; error: E };

async function safeOperation<T>(
  operation: () => Promise<T>
): Promise<Result<T>> {
  try {
    const data = await operation();
    return { success: true, data };
  } catch (error) {
    return { success: false, error: error as Error };
  }
}

// Usage
async function main() {
  const result = await safeOperation(() => 
    fetch("https://api.example.com/users").then(r => r.json())
  );
  
  if (result.success) {
    console.log("Users:", result.data);
  } else {
    console.error("Error:", result.error.message);
  }
}
```

---

## ขั้นตอนที่ 1091: TypeScript กับ Async/Await

```typescript
// src/services/user.service.ts

import { IUser } from "../models/user";
import { NotFoundError, ValidationError } from "../errors";

interface CreateUserDto {
  name: string;
  email: string;
  password: string;
}

interface UpdateUserDto {
  name?: string;
  email?: string;
}

export class UserService {
  constructor(
    private readonly userRepo: IUserRepository,
    private readonly emailService: IEmailService
  ) {}

  async findById(id: string): Promise<IUser> {
    const user = await this.userRepo.findById(id);
    if (!user) {
      throw new NotFoundError("User", id);
    }
    return user;
  }

  async create(dto: CreateUserDto): Promise<IUser> {
    // Validate
    const errors = this.validateCreateDto(dto);
    if (errors.length > 0) {
      throw new ValidationError("Validation failed", errors);
    }

    // Check duplicate email
    const existing = await this.userRepo.findByEmail(dto.email);
    if (existing) {
      throw new ValidationError("Email already exists", [
        { field: "email", message: "Email is already taken" }
      ]);
    }

    // Create user
    const user = await this.userRepo.save({
      id: crypto.randomUUID(),
      name: dto.name,
      email: dto.email,
      createdAt: new Date(),
      updatedAt: new Date()
    });

    // Send welcome email (fire and forget)
    this.emailService.sendWelcome(user.email, user.name).catch(err => {
      console.error("Failed to send welcome email:", err);
    });

    return user;
  }

  async update(id: string, dto: UpdateUserDto): Promise<IUser> {
    const user = await this.findById(id);
    
    const updated = await this.userRepo.save({
      ...user,
      ...dto,
      updatedAt: new Date()
    });
    
    return updated;
  }

  private validateCreateDto(dto: CreateUserDto): Array<{field: string; message: string}> {
    const errors: Array<{field: string; message: string}> = [];
    
    if (!dto.name || dto.name.trim().length < 2) {
      errors.push({ field: "name", message: "Name must be at least 2 characters" });
    }
    
    if (!dto.email || !/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(dto.email)) {
      errors.push({ field: "email", message: "Invalid email format" });
    }
    
    if (!dto.password || dto.password.length < 8) {
      errors.push({ field: "password", message: "Password must be at least 8 characters" });
    }
    
    return errors;
  }
}

interface IUserRepository {
  findById(id: string): Promise<IUser | null>;
  findByEmail(email: string): Promise<IUser | null>;
  save(user: IUser): Promise<IUser>;
}

interface IEmailService {
  sendWelcome(email: string, name: string): Promise<void>;
}
```

---

## ขั้นตอนที่ 1092: Utility Types ขั้นสูง

```typescript
// src/utils/types.ts

// DeepReadonly
type DeepReadonly<T> = {
  readonly [K in keyof T]: T[K] extends object ? DeepReadonly<T[K]> : T[K];
};

// NonEmptyArray
type NonEmptyArray<T> = [T, ...T[]];

function first<T>(arr: NonEmptyArray<T>): T {
  return arr[0];
}

// Awaited type (TypeScript 4.5+)
type UserResult = Awaited<Promise<IUser>>;

// ValueOf
type ValueOf<T> = T[keyof T];

const colors = {
  red: "#FF0000",
  green: "#00FF00", 
  blue: "#0000FF"
} as const;

type ColorValue = ValueOf<typeof colors>; // "#FF0000" | "#00FF00" | "#0000FF"

// Flatten array type
type Flatten<T> = T extends Array<infer U> ? U : T;

type NestedArray = string[][];
type FlatArray = Flatten<NestedArray>; // string[]

// Overloads
function parse(input: string): string[];
function parse(input: number): string;
function parse(input: string | number): string | string[] {
  if (typeof input === "string") {
    return input.split(",");
  }
  return String(input);
}

// Discriminated unions
type Shape =
  | { kind: "circle"; radius: number }
  | { kind: "square"; side: number }
  | { kind: "rectangle"; width: number; height: number };

function area(shape: Shape): number {
  switch (shape.kind) {
    case "circle":
      return Math.PI * shape.radius ** 2;
    case "square":
      return shape.side ** 2;
    case "rectangle":
      return shape.width * shape.height;
  }
}

// Type guards
function isUser(value: unknown): value is IUser {
  return (
    typeof value === "object" &&
    value !== null &&
    "id" in value &&
    "name" in value &&
    "email" in value
  );
}

async function processData(data: unknown) {
  if (isUser(data)) {
    // data is IUser here
    console.log(data.email);
  }
}

interface IUser {
  id: string;
  name: string;
  email: string;
  createdAt: Date;
  updatedAt: Date;
}
```

---

## ขั้นตอนที่ 1093: การสร้าง TypeScript Node.js Application

```typescript
// src/app.ts

import express, { Application } from "express";
import cors from "cors";
import helmet from "helmet";
import morgan from "morgan";
import { errorHandler } from "./errors";
import { userRouter } from "./routes/user.routes";
import { authRouter } from "./routes/auth.routes";

export function createApp(): Application {
  const app = express();

  // Middleware
  app.use(helmet());
  app.use(cors({
    origin: process.env.ALLOWED_ORIGINS?.split(",") ?? ["http://localhost:3000"],
    credentials: true
  }));
  app.use(morgan("combined"));
  app.use(express.json());
  app.use(express.urlencoded({ extended: true }));

  // Routes
  app.use("/api/auth", authRouter);
  app.use("/api/users", userRouter);

  // Health check
  app.get("/health", (req, res) => {
    res.json({ 
      status: "ok", 
      timestamp: new Date().toISOString(),
      version: process.env.npm_package_version
    });
  });

  // Error handler (must be last)
  app.use(errorHandler);

  return app;
}
```

```typescript
// src/server.ts

import { createApp } from "./app";
import { connectDatabase } from "./database";

const PORT = parseInt(process.env.PORT ?? "3000", 10);

async function main(): Promise<void> {
  // Connect to database
  await connectDatabase();
  
  const app = createApp();
  
  const server = app.listen(PORT, () => {
    console.log(`Server running on port ${PORT}`);
  });

  // Graceful shutdown
  const shutdown = async (signal: string) => {
    console.log(`Received ${signal}, shutting down gracefully...`);
    
    server.close(async () => {
      // Cleanup resources
      console.log("Server closed");
      process.exit(0);
    });
    
    // Force close after 30s
    setTimeout(() => {
      console.error("Could not close connections in time, forcefully shutting down");
      process.exit(1);
    }, 30000);
  };

  process.on("SIGTERM", () => shutdown("SIGTERM"));
  process.on("SIGINT", () => shutdown("SIGINT"));
}

main().catch(error => {
  console.error("Failed to start server:", error);
  process.exit(1);
});
```

---

## ขั้นตอนที่ 1094: Dependency Injection Pattern

```typescript
// src/container.ts

import "reflect-metadata";

// Simple DI Container
class Container {
  private bindings = new Map<symbol, any>();
  private singletons = new Map<symbol, any>();
  private isSingleton = new Set<symbol>();

  bind<T>(token: symbol, factory: () => T, singleton: boolean = false): void {
    this.bindings.set(token, factory);
    if (singleton) {
      this.isSingleton.add(token);
    }
  }

  get<T>(token: symbol): T {
    if (this.isSingleton.has(token)) {
      if (!this.singletons.has(token)) {
        const factory = this.bindings.get(token);
        if (!factory) throw new Error(`No binding for token: ${token.toString()}`);
        this.singletons.set(token, factory());
      }
      return this.singletons.get(token);
    }

    const factory = this.bindings.get(token);
    if (!factory) throw new Error(`No binding for token: ${token.toString()}`);
    return factory();
  }
}

// Tokens
const TOKENS = {
  UserRepository: Symbol("UserRepository"),
  UserService: Symbol("UserService"),
  EmailService: Symbol("EmailService"),
  UserController: Symbol("UserController")
} as const;

// Setup container
const container = new Container();

container.bind(TOKENS.EmailService, () => new EmailServiceImpl(), true);
container.bind(TOKENS.UserRepository, () => new UserRepositoryImpl(), true);
container.bind(TOKENS.UserService, () => {
  const repo = container.get<IUserRepository>(TOKENS.UserRepository);
  const email = container.get<IEmailService>(TOKENS.EmailService);
  return new UserService(repo, email);
}, true);
container.bind(TOKENS.UserController, () => {
  const service = container.get<UserService>(TOKENS.UserService);
  return new UserControllerImpl(service);
}, true);

export { container, TOKENS };

class EmailServiceImpl {
  async sendWelcome(email: string, name: string): Promise<void> {
    console.log(`Sending welcome email to ${name} at ${email}`);
  }
}

class UserRepositoryImpl {
  private users = new Map<string, any>();
  async findById(id: string) { return this.users.get(id) ?? null; }
  async findByEmail(email: string) { 
    for (const u of this.users.values()) if (u.email === email) return u;
    return null;
  }
  async save(user: any) { this.users.set(user.id, user); return user; }
}

class UserControllerImpl {
  constructor(private service: any) {}
}

interface IUserRepository {
  findById(id: string): Promise<any>;
  findByEmail(email: string): Promise<any>;
  save(user: any): Promise<any>;
}

interface IEmailService {
  sendWelcome(email: string, name: string): Promise<void>;
}

import { UserService } from "./services/user.service";
```

---

## ขั้นตอนที่ 1095: การ Compile และ Build

```json
// package.json scripts
{
  "scripts": {
    "build": "tsc",
    "build:watch": "tsc --watch",
    "start": "node dist/server.js",
    "dev": "nodemon --exec ts-node src/server.ts",
    "clean": "rm -rf dist",
    "typecheck": "tsc --noEmit",
    "lint": "eslint src/**/*.ts",
    "test": "jest"
  }
}
```

```javascript
// nodemon.json
{
  "watch": ["src"],
  "ext": "ts",
  "ignore": ["src/**/*.test.ts"],
  "exec": "ts-node src/server.ts"
}
```

```bash
# Build for production
npm run build

# ตรวจสอบ type errors
npm run typecheck

# รัน development mode
npm run dev

# โครงสร้างไฟล์หลัง build
# dist/
# ├── server.js
# ├── server.d.ts
# ├── server.js.map
# ├── app.js
# ├── models/
# │   └── user.js
# ├── services/
# │   └── user.service.js
# └── controllers/
#     └── user.controller.js
```

---

## ขั้นตอนที่ 1096: Testing ด้วย TypeScript

```typescript
// src/services/__tests__/user.service.test.ts

import { UserService } from "../user.service";
import { NotFoundError, ValidationError } from "../../errors";

describe("UserService", () => {
  let userService: UserService;
  let mockUserRepo: jest.Mocked<IUserRepository>;
  let mockEmailService: jest.Mocked<IEmailService>;

  beforeEach(() => {
    mockUserRepo = {
      findById: jest.fn(),
      findByEmail: jest.fn(),
      save: jest.fn()
    };
    
    mockEmailService = {
      sendWelcome: jest.fn().mockResolvedValue(undefined)
    };
    
    userService = new UserService(mockUserRepo, mockEmailService);
  });

  describe("findById", () => {
    it("should return user when found", async () => {
      const mockUser = {
        id: "1",
        name: "Alice",
        email: "alice@example.com",
        createdAt: new Date(),
        updatedAt: new Date()
      };
      
      mockUserRepo.findById.mockResolvedValue(mockUser);
      
      const result = await userService.findById("1");
      
      expect(result).toEqual(mockUser);
      expect(mockUserRepo.findById).toHaveBeenCalledWith("1");
    });

    it("should throw NotFoundError when user not found", async () => {
      mockUserRepo.findById.mockResolvedValue(null);
      
      await expect(userService.findById("999")).rejects.toThrow(NotFoundError);
    });
  });

  describe("create", () => {
    it("should create user successfully", async () => {
      mockUserRepo.findByEmail.mockResolvedValue(null);
      mockUserRepo.save.mockImplementation(async (user) => user);
      
      const result = await userService.create({
        name: "Bob",
        email: "bob@example.com",
        password: "password123"
      });
      
      expect(result.name).toBe("Bob");
      expect(result.email).toBe("bob@example.com");
    });

    it("should throw ValidationError for invalid email", async () => {
      await expect(userService.create({
        name: "Bob",
        email: "invalid-email",
        password: "password123"
      })).rejects.toThrow(ValidationError);
    });
  });
});

interface IUserRepository {
  findById(id: string): Promise<any>;
  findByEmail(email: string): Promise<any>;
  save(user: any): Promise<any>;
}

interface IEmailService {
  sendWelcome(email: string, name: string): Promise<void>;
}
```

---

## ขั้นตอนที่ 1097: Path Aliases

```json
// tsconfig.json
{
  "compilerOptions": {
    "baseUrl": "src",
    "paths": {
      "@models/*": ["models/*"],
      "@services/*": ["services/*"],
      "@controllers/*": ["controllers/*"],
      "@middleware/*": ["middleware/*"],
      "@utils/*": ["utils/*"],
      "@types/*": ["types/*"],
      "@config/*": ["config/*"]
    }
  }
}
```

```typescript
// ใช้ path aliases
import { UserService } from "@services/user.service";
import { UserController } from "@controllers/user.controller";
import { authMiddleware } from "@middleware/auth";
import { AppError } from "@utils/errors";
```

```bash
# ติดตั้ง tsconfig-paths สำหรับรันtime
npm install tsconfig-paths

# อัพเดท nodemon.json
{
  "exec": "ts-node -r tsconfig-paths/register src/server.ts"
}

# หรือใช้ module-alias สำหรับ compiled output
npm install module-alias

# package.json
{
  "_moduleAliases": {
    "@models": "dist/models",
    "@services": "dist/services",
    "@controllers": "dist/controllers"
  }
}
```

---

## ขั้นตอนที่ 1098: Zod สำหรับ Runtime Validation

```typescript
// src/schemas/user.schema.ts

import { z } from "zod";

export const CreateUserSchema = z.object({
  name: z.string()
    .min(2, "Name must be at least 2 characters")
    .max(100, "Name must be less than 100 characters")
    .trim(),
  email: z.string()
    .email("Invalid email format")
    .toLowerCase(),
  password: z.string()
    .min(8, "Password must be at least 8 characters")
    .regex(/[A-Z]/, "Password must contain at least one uppercase letter")
    .regex(/[0-9]/, "Password must contain at least one number"),
  age: z.number()
    .int()
    .min(18, "Must be at least 18 years old")
    .max(120)
    .optional()
});

export const UpdateUserSchema = CreateUserSchema.partial().omit({ password: true });

// Infer TypeScript types from Zod schemas
export type CreateUserDto = z.infer<typeof CreateUserSchema>;
export type UpdateUserDto = z.infer<typeof UpdateUserSchema>;

// Validation middleware
import { Request, Response, NextFunction } from "express";
import { ZodSchema } from "zod";

export function validate<T>(schema: ZodSchema<T>) {
  return (req: Request, res: Response, next: NextFunction) => {
    const result = schema.safeParse(req.body);
    
    if (!result.success) {
      return res.status(400).json({
        error: {
          code: "VALIDATION_ERROR",
          message: "Validation failed",
          errors: result.error.errors.map(e => ({
            field: e.path.join("."),
            message: e.message
          }))
        }
      });
    }
    
    req.body = result.data;
    next();
  };
}

// Usage in routes
import { Router } from "express";

const router = Router();

router.post(
  "/users",
  validate(CreateUserSchema),
  async (req: Request<{}, {}, CreateUserDto>, res: Response) => {
    // req.body is now typed as CreateUserDto
    const { name, email, password } = req.body;
    // ...
  }
);
```

---

## ขั้นตอนที่ 1099: Environment Variables ด้วย TypeScript

```typescript
// src/config/env.ts

import { z } from "zod";

const EnvSchema = z.object({
  NODE_ENV: z.enum(["development", "production", "test"]).default("development"),
  PORT: z.string().transform(Number).pipe(z.number().positive()).default("3000"),
  DATABASE_URL: z.string().url(),
  JWT_SECRET: z.string().min(32),
  JWT_EXPIRES_IN: z.string().default("7d"),
  REDIS_URL: z.string().url().optional(),
  EMAIL_SERVICE_URL: z.string().url().optional(),
  ALLOWED_ORIGINS: z.string().transform(s => s.split(",")).default("http://localhost:3000"),
  LOG_LEVEL: z.enum(["error", "warn", "info", "debug"]).default("info"),
  MAX_REQUEST_BODY: z.string().default("10mb")
});

export type Env = z.infer<typeof EnvSchema>;

function validateEnv(): Env {
  const result = EnvSchema.safeParse(process.env);
  
  if (!result.success) {
    console.error("Invalid environment variables:");
    for (const error of result.error.errors) {
      console.error(`  ${error.path.join(".")}: ${error.message}`);
    }
    process.exit(1);
  }
  
  return result.data;
}

export const env = validateEnv();
```

```typescript
// src/config/index.ts

import { env } from "./env";

export const config = {
  app: {
    port: env.PORT,
    env: env.NODE_ENV,
    isProduction: env.NODE_ENV === "production",
    isDevelopment: env.NODE_ENV === "development",
    isTest: env.NODE_ENV === "test"
  },
  database: {
    url: env.DATABASE_URL
  },
  jwt: {
    secret: env.JWT_SECRET,
    expiresIn: env.JWT_EXPIRES_IN
  },
  redis: {
    url: env.REDIS_URL
  },
  security: {
    allowedOrigins: env.ALLOWED_ORIGINS
  },
  logging: {
    level: env.LOG_LEVEL
  }
} as const;
```

---

## ขั้นตอนที่ 1100: โปรเจกต์สมบูรณ์ - TypeScript REST API

```typescript
// src/routes/user.routes.ts

import { Router } from "express";
import { UserController } from "@controllers/user.controller";
import { authMiddleware } from "@middleware/auth";
import { validate } from "@middleware/validate";
import { CreateUserSchema, UpdateUserSchema } from "@schemas/user.schema";
import { container, TOKENS } from "@container";

const router = Router();
const controller = container.get<UserController>(TOKENS.UserController);

router.get("/", authMiddleware, controller.getUsers.bind(controller));
router.get("/:id", authMiddleware, controller.getUser.bind(controller));
router.post("/", validate(CreateUserSchema), controller.createUser.bind(controller));
router.put("/:id", authMiddleware, validate(UpdateUserSchema), controller.updateUser.bind(controller));
router.delete("/:id", authMiddleware, controller.deleteUser.bind(controller));

export { router as userRouter };
```

```bash
# โครงสร้างโปรเจกต์
src/
├── config/
│   ├── env.ts
│   └── index.ts
├── container.ts
├── app.ts
├── server.ts
├── errors/
│   └── index.ts
├── middleware/
│   ├── auth.ts
│   └── validate.ts
├── models/
│   └── user.ts
├── repositories/
│   └── user.repository.ts
├── routes/
│   ├── auth.routes.ts
│   └── user.routes.ts
├── schemas/
│   └── user.schema.ts
├── services/
│   └── user.service.ts
├── controllers/
│   └── user.controller.ts
└── types/
    ├── express.d.ts
    └── custom.d.ts
```

---

## 🏋️ แบบฝึกหัด

### แบบฝึกหัดที่ 1: ติดตั้งและกำหนดค่า TypeScript
1. สร้างโปรเจกต์ Node.js ใหม่ด้วย TypeScript
2. กำหนดค่า tsconfig.json ที่เหมาะสม
3. สร้าง Express server พื้นฐานด้วย TypeScript

### แบบฝึกหัดที่ 2: Type-safe API
1. สร้าง interface สำหรับ User model
2. สร้าง type-safe repository pattern
3. implement validation ด้วย Zod

### แบบฝึกหัดที่ 3: Decorators
1. สร้าง `@Log` decorator สำหรับ logging
2. สร้าง `@Cache` decorator สำหรับ method caching
3. สร้าง `@Retry` decorator สำหรับ auto retry

### แบบฝึกหัดที่ 4: การแปลงโปรเจกต์
1. รับโปรเจกต์ JavaScript ที่ให้มา
2. แปลงเป็น TypeScript ทีละขั้นตอน
3. เพิ่ม type safety ทั้งหมด

---

## 📚 สรุป

ใน Part นี้เราได้เรียนรู้:
- การติดตั้งและกำหนดค่า TypeScript สำหรับ Node.js
- tsconfig.json และตัวเลือกต่างๆ
- Type system ของ TypeScript รวม Generics, Utility Types
- Decorators และ Metadata
- การสร้าง type-safe Express.js application
- การใช้ Zod สำหรับ runtime validation
- Dependency Injection pattern ด้วย TypeScript

**Part ถัดไป**: NestJS Framework
