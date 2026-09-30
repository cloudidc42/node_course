# Part 66 | ขั้นตอนที่ 1121-1140 จาก 1000+

# Advanced NestJS - Interceptors, Guards, Pipes, Microservices

---

## 🎯 เป้าหมายของ Part นี้

หลังจากเรียนจบ Part นี้ คุณจะสามารถ:
1. ใช้ Advanced Interceptors สำหรับ logging และ transformation
2. สร้าง custom Guards สำหรับ authorization
3. ใช้ Pipes สำหรับ data transformation และ validation
4. สร้าง NestJS Microservices
5. ใช้ Message Brokers กับ NestJS
6. implement CQRS pattern

---

## ขั้นตอนที่ 1121: Advanced Interceptors

Interceptors ทำงานทั้งก่อนและหลัง route handler execution ใช้ RxJS observables

```typescript
// src/common/interceptors/cache.interceptor.ts
import {
  Injectable,
  NestInterceptor,
  ExecutionContext,
  CallHandler
} from "@nestjs/common";
import { Observable, of } from "rxjs";
import { tap } from "rxjs/operators";
import { Cache } from "cache-manager";
import { InjectCache } from "@nestjs/cache-manager";
import { Reflector } from "@nestjs/core";

export const CACHE_TTL_KEY = "cacheTtl";

@Injectable()
export class CustomCacheInterceptor implements NestInterceptor {
  constructor(
    @InjectCache() private cacheManager: Cache,
    private reflector: Reflector
  ) {}

  async intercept(
    context: ExecutionContext,
    next: CallHandler
  ): Promise<Observable<any>> {
    const request = context.switchToHttp().getRequest();
    
    // Only cache GET requests
    if (request.method !== "GET") {
      return next.handle();
    }
    
    const ttl = this.reflector.get<number>(CACHE_TTL_KEY, context.getHandler()) ?? 60;
    const cacheKey = `cache:${request.url}:${JSON.stringify(request.query)}`;
    
    const cached = await this.cacheManager.get(cacheKey);
    if (cached) {
      return of(cached);
    }
    
    return next.handle().pipe(
      tap(async (data) => {
        await this.cacheManager.set(cacheKey, data, ttl);
      })
    );
  }
}

// Decorator
export const CacheTTL = (ttl: number) => 
  (target: any, key: string, descriptor: PropertyDescriptor) => {
    Reflect.defineMetadata(CACHE_TTL_KEY, ttl, descriptor.value);
    return descriptor;
  };
```

```typescript
// src/common/interceptors/rate-limit.interceptor.ts
import {
  Injectable,
  NestInterceptor,
  ExecutionContext,
  CallHandler,
  HttpException,
  HttpStatus
} from "@nestjs/common";
import { Observable } from "rxjs";
import { Cache } from "cache-manager";
import { InjectCache } from "@nestjs/cache-manager";

@Injectable()
export class RateLimitInterceptor implements NestInterceptor {
  private readonly WINDOW_SIZE = 60 * 1000; // 1 minute
  private readonly MAX_REQUESTS = 100;

  constructor(@InjectCache() private cacheManager: Cache) {}

  async intercept(
    context: ExecutionContext,
    next: CallHandler
  ): Promise<Observable<any>> {
    const request = context.switchToHttp().getRequest();
    const clientIp = request.ip;
    const key = `rate_limit:${clientIp}`;
    
    const current = (await this.cacheManager.get<number>(key)) ?? 0;
    
    if (current >= this.MAX_REQUESTS) {
      throw new HttpException(
        "Too Many Requests",
        HttpStatus.TOO_MANY_REQUESTS
      );
    }
    
    await this.cacheManager.set(key, current + 1, this.WINDOW_SIZE / 1000);
    
    return next.handle();
  }
}
```

```typescript
// src/common/interceptors/timeout.interceptor.ts
import {
  Injectable,
  NestInterceptor,
  ExecutionContext,
  CallHandler,
  RequestTimeoutException
} from "@nestjs/common";
import { Observable, throwError, TimeoutError } from "rxjs";
import { catchError, timeout } from "rxjs/operators";

@Injectable()
export class TimeoutInterceptor implements NestInterceptor {
  constructor(private readonly timeoutMs: number = 5000) {}

  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    return next.handle().pipe(
      timeout(this.timeoutMs),
      catchError(err => {
        if (err instanceof TimeoutError) {
          return throwError(() => new RequestTimeoutException());
        }
        return throwError(() => err);
      })
    );
  }
}
```

---

## ขั้นตอนที่ 1122: Advanced Guards

```typescript
// src/auth/guards/permission.guard.ts
import {
  Injectable,
  CanActivate,
  ExecutionContext,
  ForbiddenException
} from "@nestjs/common";
import { Reflector } from "@nestjs/core";

export enum Permission {
  READ_USERS = "read:users",
  WRITE_USERS = "write:users",
  DELETE_USERS = "delete:users",
  MANAGE_SYSTEM = "manage:system"
}

export const RequirePermission = (...permissions: Permission[]) =>
  SetMetadata("permissions", permissions);

@Injectable()
export class PermissionGuard implements CanActivate {
  constructor(private reflector: Reflector) {}

  canActivate(context: ExecutionContext): boolean {
    const requiredPermissions = this.reflector.getAllAndOverride<Permission[]>(
      "permissions",
      [context.getHandler(), context.getClass()]
    );
    
    if (!requiredPermissions?.length) return true;
    
    const { user } = context.switchToHttp().getRequest();
    
    if (!user) {
      throw new ForbiddenException("No user found");
    }
    
    const hasAllPermissions = requiredPermissions.every(permission =>
      user.permissions?.includes(permission)
    );
    
    if (!hasAllPermissions) {
      throw new ForbiddenException("Insufficient permissions");
    }
    
    return true;
  }
}

import { SetMetadata } from "@nestjs/common";
```

```typescript
// src/auth/guards/throttle.guard.ts
import { ThrottlerGuard } from "@nestjs/throttler";
import { Injectable } from "@nestjs/common";

@Injectable()
export class CustomThrottleGuard extends ThrottlerGuard {
  protected getTracker(req: Record<string, any>): Promise<string> {
    // Track by user ID if authenticated, otherwise by IP
    const userId = req.user?.id;
    return Promise.resolve(userId ?? req.ip);
  }
}
```

---

## ขั้นตอนที่ 1123: Advanced Pipes

```typescript
// src/common/pipes/file-upload.pipe.ts
import {
  PipeTransform,
  Injectable,
  ArgumentMetadata,
  BadRequestException
} from "@nestjs/common";

interface FileUploadOptions {
  maxSize?: number;         // bytes
  allowedMimeTypes?: string[];
  required?: boolean;
}

@Injectable()
export class FileUploadPipe implements PipeTransform {
  constructor(private readonly options: FileUploadOptions = {}) {}

  transform(file: Express.Multer.File, metadata: ArgumentMetadata) {
    if (!file) {
      if (this.options.required !== false) {
        throw new BadRequestException("File is required");
      }
      return file;
    }
    
    if (this.options.maxSize && file.size > this.options.maxSize) {
      throw new BadRequestException(
        `File size exceeds limit of ${this.options.maxSize / (1024 * 1024)}MB`
      );
    }
    
    if (
      this.options.allowedMimeTypes?.length &&
      !this.options.allowedMimeTypes.includes(file.mimetype)
    ) {
      throw new BadRequestException(
        `File type not allowed. Allowed types: ${this.options.allowedMimeTypes.join(", ")}`
      );
    }
    
    return file;
  }
}
```

```typescript
// src/common/pipes/sanitize.pipe.ts
import { PipeTransform, Injectable } from "@nestjs/common";
import * as sanitizeHtml from "sanitize-html";

@Injectable()
export class SanitizePipe implements PipeTransform {
  transform(value: any) {
    if (typeof value === "string") {
      return sanitizeHtml(value, {
        allowedTags: [],
        allowedAttributes: {}
      });
    }
    
    if (typeof value === "object" && value !== null) {
      return this.sanitizeObject(value);
    }
    
    return value;
  }

  private sanitizeObject(obj: Record<string, any>): Record<string, any> {
    const result: Record<string, any> = {};
    for (const [key, val] of Object.entries(obj)) {
      if (typeof val === "string") {
        result[key] = sanitizeHtml(val, { allowedTags: [], allowedAttributes: {} });
      } else if (typeof val === "object" && val !== null) {
        result[key] = this.sanitizeObject(val);
      } else {
        result[key] = val;
      }
    }
    return result;
  }
}
```

---

## ขั้นตอนที่ 1124: CQRS Pattern

```typescript
// npm install @nestjs/cqrs

// src/users/commands/create-user.command.ts
export class CreateUserCommand {
  constructor(
    public readonly name: string,
    public readonly email: string,
    public readonly password: string
  ) {}
}

// src/users/commands/handlers/create-user.handler.ts
import { CommandHandler, ICommandHandler, EventBus } from "@nestjs/cqrs";
import { InjectRepository } from "@nestjs/typeorm";
import { Repository } from "typeorm";
import { ConflictException } from "@nestjs/common";
import { CreateUserCommand } from "../create-user.command";
import { User } from "../../entities/user.entity";
import { UserCreatedEvent } from "../../events/user-created.event";

@CommandHandler(CreateUserCommand)
export class CreateUserHandler implements ICommandHandler<CreateUserCommand> {
  constructor(
    @InjectRepository(User)
    private readonly usersRepository: Repository<User>,
    private readonly eventBus: EventBus
  ) {}

  async execute(command: CreateUserCommand): Promise<User> {
    const { name, email, password } = command;
    
    const existing = await this.usersRepository.findOne({ where: { email } });
    if (existing) throw new ConflictException("Email already exists");
    
    const user = this.usersRepository.create({ name, email, password });
    const saved = await this.usersRepository.save(user);
    
    this.eventBus.publish(new UserCreatedEvent(saved.id, saved.email, saved.name));
    
    return saved;
  }
}
```

```typescript
// src/users/queries/get-user.query.ts
export class GetUserQuery {
  constructor(public readonly userId: string) {}
}

// src/users/queries/handlers/get-user.handler.ts
import { QueryHandler, IQueryHandler } from "@nestjs/cqrs";
import { InjectRepository } from "@nestjs/typeorm";
import { Repository } from "typeorm";
import { NotFoundException } from "@nestjs/common";
import { GetUserQuery } from "../get-user.query";
import { User } from "../../entities/user.entity";

@QueryHandler(GetUserQuery)
export class GetUserHandler implements IQueryHandler<GetUserQuery> {
  constructor(
    @InjectRepository(User)
    private readonly usersRepository: Repository<User>
  ) {}

  async execute(query: GetUserQuery): Promise<User> {
    const user = await this.usersRepository.findOne({
      where: { id: query.userId }
    });
    
    if (!user) throw new NotFoundException(`User ${query.userId} not found`);
    
    return user;
  }
}
```

```typescript
// src/users/events/user-created.event.ts
export class UserCreatedEvent {
  constructor(
    public readonly userId: string,
    public readonly email: string,
    public readonly name: string
  ) {}
}

// src/users/events/handlers/user-created.handler.ts
import { EventsHandler, IEventHandler } from "@nestjs/cqrs";
import { Logger } from "@nestjs/common";
import { UserCreatedEvent } from "../user-created.event";
import { EmailService } from "../../../email/email.service";

@EventsHandler(UserCreatedEvent)
export class UserCreatedHandler implements IEventHandler<UserCreatedEvent> {
  private readonly logger = new Logger(UserCreatedHandler.name);

  constructor(private readonly emailService: EmailService) {}

  async handle(event: UserCreatedEvent): Promise<void> {
    this.logger.log(`User created: ${event.email}`);
    await this.emailService.sendWelcome(event.email, event.name);
  }
}
```

```typescript
// src/users/users.module.ts (with CQRS)
import { CqrsModule } from "@nestjs/cqrs";
import { CreateUserHandler } from "./commands/handlers/create-user.handler";
import { GetUserHandler } from "./queries/handlers/get-user.handler";
import { UserCreatedHandler } from "./events/handlers/user-created.handler";

@Module({
  imports: [CqrsModule, TypeOrmModule.forFeature([User])],
  controllers: [UsersController],
  providers: [
    UsersService,
    CreateUserHandler,
    GetUserHandler,
    UserCreatedHandler
  ]
})
export class UsersModule {}
```

---

## ขั้นตอนที่ 1125: NestJS Microservices

```typescript
// npm install @nestjs/microservices

// apps/user-service/src/main.ts - Microservice setup
import { NestFactory } from "@nestjs/core";
import { Transport, MicroserviceOptions } from "@nestjs/microservices";
import { UserServiceModule } from "./user-service.module";

async function bootstrap() {
  const app = await NestFactory.createMicroservice<MicroserviceOptions>(
    UserServiceModule,
    {
      transport: Transport.RMQ,
      options: {
        urls: [process.env.RABBITMQ_URL!],
        queue: "user_queue",
        queueOptions: { durable: true }
      }
    }
  );
  
  await app.listen();
  console.log("User microservice is listening");
}
bootstrap();
```

```typescript
// apps/user-service/src/users/users.controller.ts
import { Controller } from "@nestjs/common";
import { MessagePattern, Payload } from "@nestjs/microservices";
import { UsersService } from "./users.service";

@Controller()
export class UsersMicroserviceController {
  constructor(private readonly usersService: UsersService) {}

  @MessagePattern("get_user")
  async getUser(@Payload() data: { userId: string }) {
    return this.usersService.findById(data.userId);
  }

  @MessagePattern("create_user")
  async createUser(@Payload() data: { name: string; email: string; password: string }) {
    return this.usersService.create(data);
  }

  @MessagePattern("update_user")
  async updateUser(@Payload() data: { userId: string; updates: any }) {
    return this.usersService.update(data.userId, data.updates);
  }
}
```

```typescript
// apps/api-gateway/src/users/users.service.ts - API Gateway
import { Injectable } from "@nestjs/common";
import { ClientProxy } from "@nestjs/microservices";
import { InjectClient } from "@nestjs/microservices";
import { firstValueFrom, timeout } from "rxjs";

@Injectable()
export class UsersGatewayService {
  constructor(
    @InjectClient("USER_SERVICE")
    private readonly userClient: ClientProxy
  ) {}

  async findById(userId: string) {
    return firstValueFrom(
      this.userClient.send("get_user", { userId }).pipe(timeout(5000))
    );
  }

  async create(data: { name: string; email: string; password: string }) {
    return firstValueFrom(
      this.userClient.send("create_user", data).pipe(timeout(5000))
    );
  }
}
```

---

## ขั้นตอนที่ 1126: Event-Driven Microservices

```typescript
// apps/user-service/src/users/users.controller.ts (Event patterns)
import { EventPattern, Payload } from "@nestjs/microservices";

@Controller()
export class UsersEventController {
  constructor(private readonly usersService: UsersService) {}

  @EventPattern("order_completed")
  async handleOrderCompleted(@Payload() data: {
    userId: string;
    orderId: string;
    amount: number;
  }) {
    await this.usersService.updateUserStats(data.userId, {
      totalOrders: 1,
      totalSpent: data.amount
    });
  }
}
```

```typescript
// Publishing events from API Gateway
import { Inject } from "@nestjs/common";

@Injectable()
export class OrdersService {
  constructor(
    @InjectClient("USER_SERVICE")
    private readonly userClient: ClientProxy,
    @InjectClient("NOTIFICATION_SERVICE")
    private readonly notificationClient: ClientProxy
  ) {}

  async completeOrder(orderId: string, userId: string, amount: number) {
    // Process order
    const order = await this.ordersRepository.save({
      id: orderId,
      userId,
      amount,
      status: "completed"
    });
    
    // Emit event to multiple services
    this.userClient.emit("order_completed", { userId, orderId, amount });
    this.notificationClient.emit("send_notification", {
      userId,
      message: `Order ${orderId} completed! Amount: $${amount}`
    });
    
    return order;
  }
}
```

---

## ขั้นตอนที่ 1127: gRPC Microservices

```typescript
// npm install @grpc/grpc-js @grpc/proto-loader

// proto/user.proto
// syntax = "proto3";
// package user;
// 
// service UserService {
//   rpc GetUser (GetUserRequest) returns (UserResponse);
//   rpc CreateUser (CreateUserRequest) returns (UserResponse);
//   rpc ListUsers (ListUsersRequest) returns (ListUsersResponse);
// }
// 
// message GetUserRequest { string id = 1; }
// message CreateUserRequest {
//   string name = 1;
//   string email = 2;
// }
// message UserResponse {
//   string id = 1;
//   string name = 2;
//   string email = 3;
// }

// apps/user-service/src/main.ts (gRPC)
import { NestFactory } from "@nestjs/core";
import { Transport, MicroserviceOptions } from "@nestjs/microservices";
import { join } from "path";

async function bootstrap() {
  const app = await NestFactory.createMicroservice<MicroserviceOptions>(
    UserServiceModule,
    {
      transport: Transport.GRPC,
      options: {
        package: "user",
        protoPath: join(__dirname, "../../../proto/user.proto"),
        url: "0.0.0.0:50051"
      }
    }
  );
  
  await app.listen();
}

// Controller for gRPC
import { Controller } from "@nestjs/common";
import { GrpcMethod } from "@nestjs/microservices";

@Controller()
export class UserGrpcController {
  constructor(private readonly usersService: UsersService) {}

  @GrpcMethod("UserService", "GetUser")
  async getUser(data: { id: string }): Promise<any> {
    return this.usersService.findById(data.id);
  }

  @GrpcMethod("UserService", "CreateUser")
  async createUser(data: { name: string; email: string }): Promise<any> {
    return this.usersService.create(data);
  }
}
```

---

## ขั้นตอนที่ 1128: Advanced Error Handling

```typescript
// src/common/filters/all-exceptions.filter.ts
import {
  ExceptionFilter,
  Catch,
  ArgumentsHost,
  HttpException,
  HttpStatus,
  Logger
} from "@nestjs/common";
import { QueryFailedError } from "typeorm";

@Catch()
export class GlobalExceptionFilter implements ExceptionFilter {
  private readonly logger = new Logger("GlobalExceptionFilter");

  catch(exception: unknown, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse();
    const request = ctx.getRequest();
    
    let statusCode = HttpStatus.INTERNAL_SERVER_ERROR;
    let message = "Internal server error";
    let errors: any[] | undefined;
    
    if (exception instanceof HttpException) {
      statusCode = exception.getStatus();
      const exceptionResponse = exception.getResponse();
      
      if (typeof exceptionResponse === "object") {
        const { message: msg, errors: errs } = exceptionResponse as any;
        message = Array.isArray(msg) ? msg.join(", ") : msg;
        errors = errs;
      } else {
        message = exceptionResponse as string;
      }
    } else if (exception instanceof QueryFailedError) {
      // Handle database errors
      if ((exception as any).code === "23505") { // Unique constraint violation
        statusCode = HttpStatus.CONFLICT;
        message = "Resource already exists";
      } else if ((exception as any).code === "23503") { // Foreign key violation
        statusCode = HttpStatus.BAD_REQUEST;
        message = "Referenced resource not found";
      }
    } else if (exception instanceof Error) {
      this.logger.error(exception.message, exception.stack);
    }
    
    const responseBody: Record<string, any> = {
      statusCode,
      timestamp: new Date().toISOString(),
      path: request.url,
      message
    };
    
    if (errors) responseBody.errors = errors;
    
    response.status(statusCode).json(responseBody);
  }
}
```

---

## ขั้นตอนที่ 1129: Request Lifecycle Hooks

```typescript
// src/common/middleware/request-id.middleware.ts
import { Injectable, NestMiddleware } from "@nestjs/common";
import { Request, Response, NextFunction } from "express";
import { v4 as uuidv4 } from "uuid";

@Injectable()
export class RequestIdMiddleware implements NestMiddleware {
  use(req: Request, res: Response, next: NextFunction) {
    req.headers["x-request-id"] = req.headers["x-request-id"] || uuidv4();
    res.setHeader("x-request-id", req.headers["x-request-id"]);
    next();
  }
}

// src/app.module.ts
import { MiddlewareConsumer, Module, NestModule } from "@nestjs/common";

@Module({})
export class AppModule implements NestModule {
  configure(consumer: MiddlewareConsumer) {
    consumer
      .apply(RequestIdMiddleware)
      .forRoutes("*");
  }
}
```

---

## ขั้นตอนที่ 1130: Event Sourcing Pattern

```typescript
// src/common/event-store/event-store.service.ts
import { Injectable } from "@nestjs/common";
import { InjectRepository } from "@nestjs/typeorm";
import { Repository } from "typeorm";
import { EventStore } from "./event-store.entity";

@Injectable()
export class EventStoreService {
  constructor(
    @InjectRepository(EventStore)
    private readonly eventStoreRepository: Repository<EventStore>
  ) {}

  async append(
    aggregateId: string,
    aggregateType: string,
    eventType: string,
    eventData: object,
    version: number
  ): Promise<void> {
    const event = this.eventStoreRepository.create({
      aggregateId,
      aggregateType,
      eventType,
      eventData,
      version,
      createdAt: new Date()
    });
    
    await this.eventStoreRepository.save(event);
  }

  async getEvents(aggregateId: string, fromVersion?: number): Promise<EventStore[]> {
    const qb = this.eventStoreRepository
      .createQueryBuilder("event")
      .where("event.aggregateId = :aggregateId", { aggregateId })
      .orderBy("event.version", "ASC");
    
    if (fromVersion !== undefined) {
      qb.andWhere("event.version > :fromVersion", { fromVersion });
    }
    
    return qb.getMany();
  }
}
```

---

## ขั้นตอนที่ 1131: Saga Pattern

```typescript
// src/orders/sagas/order.saga.ts
import { Injectable } from "@nestjs/common";
import { ICommand, Saga, ofType } from "@nestjs/cqrs";
import { Observable } from "rxjs";
import { map, filter } from "rxjs/operators";
import { OrderCreatedEvent } from "../events/order-created.event";
import { ReserveInventoryCommand } from "../../inventory/commands/reserve-inventory.command";
import { SendOrderConfirmationCommand } from "../../notifications/commands/send-order-confirmation.command";

@Injectable()
export class OrderSaga {
  @Saga()
  orderCreated = (events$: Observable<any>): Observable<ICommand> => {
    return events$.pipe(
      ofType(OrderCreatedEvent),
      map(event => new ReserveInventoryCommand(
        event.orderId,
        event.items
      ))
    );
  };

  @Saga()
  sendConfirmation = (events$: Observable<any>): Observable<ICommand> => {
    return events$.pipe(
      ofType(OrderCreatedEvent),
      map(event => new SendOrderConfirmationCommand(
        event.userId,
        event.orderId,
        event.total
      ))
    );
  };
}
```

---

## ขั้นตอนที่ 1132: Dynamic Modules

```typescript
// src/database/database.module.ts
import { DynamicModule, Module } from "@nestjs/common";
import { TypeOrmModule } from "@nestjs/typeorm";
import { ConfigService } from "@nestjs/config";

interface DatabaseModuleOptions {
  readReplicas?: boolean;
  connectionPoolSize?: number;
}

@Module({})
export class DatabaseModule {
  static forRoot(options: DatabaseModuleOptions = {}): DynamicModule {
    return {
      module: DatabaseModule,
      imports: [
        TypeOrmModule.forRootAsync({
          useFactory: (configService: ConfigService) => ({
            type: "postgres",
            host: configService.get("DB_HOST"),
            port: configService.get<number>("DB_PORT"),
            username: configService.get("DB_USER"),
            password: configService.get("DB_PASS"),
            database: configService.get("DB_NAME"),
            entities: [__dirname + "/../**/*.entity{.ts,.js}"],
            synchronize: configService.get("NODE_ENV") === "development",
            extra: {
              max: options.connectionPoolSize ?? 10
            }
          }),
          inject: [ConfigService]
        })
      ],
      exports: [TypeOrmModule]
    };
  }
}
```

---

## ขั้นตอนที่ 1133: Microservices กับ Kafka

```typescript
// npm install kafkajs @nestjs/microservices

// apps/notification-service/src/main.ts
import { NestFactory } from "@nestjs/core";
import { Transport, MicroserviceOptions } from "@nestjs/microservices";

async function bootstrap() {
  const app = await NestFactory.createMicroservice<MicroserviceOptions>(
    NotificationModule,
    {
      transport: Transport.KAFKA,
      options: {
        client: {
          clientId: "notification-service",
          brokers: [process.env.KAFKA_BROKER!]
        },
        consumer: {
          groupId: "notification-consumer-group"
        }
      }
    }
  );
  
  await app.listen();
}

// Controller
import { Controller } from "@nestjs/common";
import { MessagePattern, Payload } from "@nestjs/microservices";

@Controller()
export class NotificationController {
  @MessagePattern("user.created")
  async handleUserCreated(@Payload() message: {
    userId: string;
    email: string;
    name: string;
  }) {
    console.log(`Sending welcome email to ${message.email}`);
  }

  @MessagePattern("order.completed")
  async handleOrderCompleted(@Payload() message: {
    userId: string;
    orderId: string;
    amount: number;
  }) {
    console.log(`Order ${message.orderId} completed for user ${message.userId}`);
  }
}
```

---

## ขั้นตอนที่ 1134: Service Discovery และ Load Balancing

```typescript
// src/common/decorators/service-client.decorator.ts
import { Inject } from "@nestjs/common";
import { ClientProxy } from "@nestjs/microservices";

export const InjectServiceClient = (serviceName: string) => 
  Inject(serviceName);

// src/common/modules/clients.module.ts
import { Module } from "@nestjs/common";
import { ClientsModule, Transport } from "@nestjs/microservices";

@Module({
  imports: [
    ClientsModule.registerAsync([
      {
        name: "USER_SERVICE",
        useFactory: () => ({
          transport: Transport.RMQ,
          options: {
            urls: [process.env.RABBITMQ_URL!],
            queue: "user_queue",
            queueOptions: { durable: true }
          }
        })
      },
      {
        name: "ORDER_SERVICE",
        useFactory: () => ({
          transport: Transport.RMQ,
          options: {
            urls: [process.env.RABBITMQ_URL!],
            queue: "order_queue",
            queueOptions: { durable: true }
          }
        })
      }
    ])
  ],
  exports: [ClientsModule]
})
export class ClientsRegistryModule {}
```

---

## ขั้นตอนที่ 1135: Advanced Testing

```typescript
// src/users/tests/users.controller.spec.ts - E2E Testing
import { Test, TestingModule } from "@nestjs/testing";
import { INestApplication, ValidationPipe } from "@nestjs/common";
import * as request from "supertest";
import { AppModule } from "../../app.module";
import { getRepositoryToken } from "@nestjs/typeorm";
import { User } from "../entities/user.entity";

describe("UsersController (e2e)", () => {
  let app: INestApplication;
  let jwtToken: string;

  beforeAll(async () => {
    const moduleFixture: TestingModule = await Test.createTestingModule({
      imports: [AppModule]
    })
    .overrideProvider(getRepositoryToken(User))
    .useValue({
      findOne: jest.fn(),
      findAndCount: jest.fn().mockResolvedValue([[], 0]),
      create: jest.fn(),
      save: jest.fn()
    })
    .compile();

    app = moduleFixture.createNestApplication();
    app.useGlobalPipes(new ValidationPipe({ whitelist: true }));
    await app.init();
    
    // Get auth token
    const response = await request(app.getHttpServer())
      .post("/api/v1/auth/login")
      .send({ email: "admin@test.com", password: "Admin123!" });
    
    jwtToken = response.body.accessToken;
  });

  afterAll(async () => {
    await app.close();
  });

  describe("GET /api/v1/users", () => {
    it("should return paginated users", async () => {
      return request(app.getHttpServer())
        .get("/api/v1/users")
        .set("Authorization", `Bearer ${jwtToken}`)
        .expect(200)
        .expect(res => {
          expect(res.body).toHaveProperty("data");
          expect(res.body).toHaveProperty("total");
          expect(res.body).toHaveProperty("page");
        });
    });

    it("should return 401 without auth token", () => {
      return request(app.getHttpServer())
        .get("/api/v1/users")
        .expect(401);
    });
  });
});
```

---

## ขั้นตอนที่ 1136: Monitoring กับ NestJS

```typescript
// npm install @willsoto/nestjs-prometheus prom-client

// src/monitoring/monitoring.module.ts
import { Module } from "@nestjs/common";
import {
  PrometheusModule,
  makeCounterProvider,
  makeHistogramProvider
} from "@willsoto/nestjs-prometheus";

@Module({
  imports: [PrometheusModule.register()],
  providers: [
    makeCounterProvider({
      name: "http_requests_total",
      help: "Total number of HTTP requests",
      labelNames: ["method", "path", "status"]
    }),
    makeHistogramProvider({
      name: "http_request_duration_seconds",
      help: "HTTP request duration in seconds",
      labelNames: ["method", "path"],
      buckets: [0.1, 0.5, 1, 2, 5]
    })
  ],
  exports: [PrometheusModule]
})
export class MonitoringModule {}
```

```typescript
// src/common/interceptors/metrics.interceptor.ts
import {
  Injectable,
  NestInterceptor,
  ExecutionContext,
  CallHandler
} from "@nestjs/common";
import { Observable } from "rxjs";
import { tap } from "rxjs/operators";
import { InjectMetric } from "@willsoto/nestjs-prometheus";
import { Counter, Histogram } from "prom-client";

@Injectable()
export class MetricsInterceptor implements NestInterceptor {
  constructor(
    @InjectMetric("http_requests_total")
    private readonly counter: Counter<string>,
    @InjectMetric("http_request_duration_seconds")
    private readonly histogram: Histogram<string>
  ) {}

  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    const request = context.switchToHttp().getRequest();
    const response = context.switchToHttp().getResponse();
    const { method, path } = request;
    const timer = this.histogram.startTimer({ method, path });
    
    return next.handle().pipe(
      tap({
        next: () => {
          this.counter.labels(method, path, String(response.statusCode)).inc();
          timer();
        },
        error: () => {
          this.counter.labels(method, path, "500").inc();
          timer();
        }
      })
    );
  }
}
```

---

## ขั้นตอนที่ 1137: Feature Modules Best Practices

```typescript
// src/products/products.module.ts - Feature module example
import { Module } from "@nestjs/common";
import { TypeOrmModule } from "@nestjs/typeorm";
import { CqrsModule } from "@nestjs/cqrs";
import { BullModule } from "@nestjs/bull";

import { ProductsController } from "./products.controller";
import { ProductsService } from "./products.service";
import { Product } from "./entities/product.entity";
import { ProductImage } from "./entities/product-image.entity";
import { ProductCategory } from "./entities/product-category.entity";

// CQRS Handlers
import { CreateProductHandler } from "./commands/handlers/create-product.handler";
import { UpdateProductHandler } from "./commands/handlers/update-product.handler";
import { GetProductHandler } from "./queries/handlers/get-product.handler";
import { SearchProductsHandler } from "./queries/handlers/search-products.handler";

// Event Handlers
import { ProductCreatedHandler } from "./events/handlers/product-created.handler";

// Processors
import { ImageResizeProcessor } from "./processors/image-resize.processor";

const CommandHandlers = [CreateProductHandler, UpdateProductHandler];
const QueryHandlers = [GetProductHandler, SearchProductsHandler];
const EventHandlers = [ProductCreatedHandler];

@Module({
  imports: [
    TypeOrmModule.forFeature([Product, ProductImage, ProductCategory]),
    CqrsModule,
    BullModule.registerQueue({ name: "image-processing" })
  ],
  controllers: [ProductsController],
  providers: [
    ProductsService,
    ...CommandHandlers,
    ...QueryHandlers,
    ...EventHandlers,
    ImageResizeProcessor
  ],
  exports: [ProductsService]
})
export class ProductsModule {}
```

---

## ขั้นตอนที่ 1138: Lifecycle Hooks

```typescript
// src/app.service.ts
import {
  Injectable,
  OnModuleInit,
  OnModuleDestroy,
  OnApplicationBootstrap,
  OnApplicationShutdown
} from "@nestjs/common";
import { InjectConnection } from "@nestjs/typeorm";
import { Connection } from "typeorm";

@Injectable()
export class AppService
  implements OnModuleInit, OnModuleDestroy, OnApplicationBootstrap, OnApplicationShutdown {
  constructor(
    @InjectConnection() private readonly connection: Connection
  ) {}

  async onModuleInit() {
    console.log("AppService initialized");
    // Run migrations
    await this.connection.runMigrations();
  }

  async onApplicationBootstrap() {
    console.log("Application fully started");
    // Seed initial data if needed
  }

  async onModuleDestroy() {
    console.log("AppService being destroyed");
  }

  async onApplicationShutdown(signal?: string) {
    console.log(`Application shutting down (signal: ${signal})`);
    await this.connection.close();
  }
}
```

---

## ขั้นตอนที่ 1139: Advanced TypeORM Patterns

```typescript
// src/common/repositories/base.repository.ts
import { Repository, FindManyOptions, FindOneOptions } from "typeorm";
import { NotFoundException } from "@nestjs/common";

export abstract class BaseRepository<T extends { id: string }> {
  constructor(protected readonly repository: Repository<T>) {}

  async findAll(options?: FindManyOptions<T>): Promise<T[]> {
    return this.repository.find(options);
  }

  async findById(id: string, options?: FindOneOptions<T>): Promise<T> {
    const entity = await this.repository.findOne({
      where: { id } as any,
      ...options
    });
    
    if (!entity) {
      throw new NotFoundException(`Entity with id ${id} not found`);
    }
    
    return entity;
  }

  async save(entity: T): Promise<T> {
    return this.repository.save(entity);
  }

  async delete(id: string): Promise<void> {
    await this.findById(id);
    await this.repository.delete(id);
  }

  async exists(id: string): Promise<boolean> {
    return this.repository.exists({ where: { id } as any });
  }
}
```

---

## ขั้นตอนที่ 1140: Production Deployment

```typescript
// src/main.ts (Production configuration)
import { NestFactory } from "@nestjs/core";
import { AppModule } from "./app.module";
import { ValidationPipe, VersioningType } from "@nestjs/common";
import { SwaggerModule, DocumentBuilder } from "@nestjs/swagger";
import * as compression from "compression";
import helmet from "helmet";
import { GlobalExceptionFilter } from "./common/filters/all-exceptions.filter";
import { LoggingInterceptor } from "./common/interceptors/logging.interceptor";
import { TransformInterceptor } from "./common/interceptors/transform.interceptor";

async function bootstrap() {
  const app = await NestFactory.create(AppModule, {
    logger: ["error", "warn", "log"]
  });

  // Security
  app.use(helmet());
  app.use(compression());
  
  // CORS
  app.enableCors({
    origin: process.env.ALLOWED_ORIGINS?.split(","),
    credentials: true,
    methods: ["GET", "POST", "PUT", "DELETE", "PATCH", "OPTIONS"]
  });
  
  // Versioning
  app.enableVersioning({ type: VersioningType.URI });
  
  // Global prefix
  app.setGlobalPrefix("api");
  
  // Global pipes
  app.useGlobalPipes(new ValidationPipe({
    whitelist: true,
    forbidNonWhitelisted: true,
    transform: true,
    transformOptions: { enableImplicitConversion: true }
  }));
  
  // Global filters
  app.useGlobalFilters(new GlobalExceptionFilter());
  
  // Global interceptors
  app.useGlobalInterceptors(
    new LoggingInterceptor(),
    new TransformInterceptor()
  );
  
  // Swagger (disable in production)
  if (process.env.NODE_ENV !== "production") {
    const config = new DocumentBuilder()
      .setTitle("API Documentation")
      .setVersion("1.0")
      .addBearerAuth()
      .build();
    const doc = SwaggerModule.createDocument(app, config);
    SwaggerModule.setup("docs", app, doc);
  }
  
  const port = process.env.PORT ?? 3000;
  await app.listen(port);
  console.log(`Application running on port ${port}`);
}

bootstrap();
```

```dockerfile
# Dockerfile
FROM node:20-alpine AS builder

WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM node:20-alpine AS production

WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY --from=builder /app/dist ./dist

USER node
EXPOSE 3000
CMD ["node", "dist/main"]
```

---

## 🏋️ แบบฝึกหัด

### แบบฝึกหัดที่ 1: CQRS Pattern
1. implement CQRS สำหรับ Orders module
2. สร้าง commands: CreateOrder, UpdateOrderStatus, CancelOrder
3. สร้าง queries: GetOrder, ListOrders, GetOrdersByUser
4. implement sagas สำหรับ order processing flow

### แบบฝึกหัดที่ 2: Microservices
1. สร้าง 3 microservices: user-service, order-service, notification-service
2. เชื่อมต่อด้วย RabbitMQ
3. implement event-driven communication

### แบบฝึกหัดที่ 3: Testing
1. เขียน unit tests ครอบคลุม 90%
2. เขียน integration tests
3. เขียน e2e tests สำหรับ critical flows

---

## 📚 สรุป

ใน Part นี้เราได้เรียนรู้:
- Advanced Interceptors (cache, rate limit, timeout, metrics)
- Advanced Guards (permissions, throttling)
- Advanced Pipes (file upload, sanitization)
- CQRS Pattern (Commands, Queries, Events, Sagas)
- NestJS Microservices (RabbitMQ, Kafka, gRPC)
- Event Sourcing pattern
- Dynamic Modules
- Production deployment

**Part ถัดไป**: Prisma ORM
