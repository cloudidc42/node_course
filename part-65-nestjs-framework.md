# Part 65 | ขั้นตอนที่ 1101-1120 จาก 1000+

# NestJS Framework

---

## 🎯 เป้าหมายของ Part นี้

หลังจากเรียนจบ Part นี้ คุณจะสามารถ:
1. เข้าใจ architecture ของ NestJS
2. สร้าง Modules, Controllers, Services
3. ใช้ Dependency Injection Container
4. สร้าง REST API ด้วย NestJS
5. ใช้ Pipes, Guards, Interceptors พื้นฐาน
6. เชื่อมต่อกับ Database ด้วย TypeORM

---

## ขั้นตอนที่ 1101: NestJS คืออะไร?

NestJS เป็น framework สำหรับสร้าง Node.js server-side applications ที่มีประสิทธิภาพ สามารถ scale ได้ และ maintainable โดยใช้ TypeScript เป็นหลัก

```
NestJS Architecture:
┌─────────────────────────────────────────────────────┐
│                    NestJS App                        │
│                                                      │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐ │
│  │   Module A   │  │   Module B   │  │   Module C   │ │
│  │  ┌────────┐  │  │  ┌────────┐  │  │  ┌────────┐  │ │
│  │  │Controller│  │  │Controller│  │  │Controller│  │ │
│  │  └────────┘  │  │  └────────┘  │  │  └────────┘  │ │
│  │  ┌────────┐  │  │  ┌────────┐  │  │  ┌────────┐  │ │
│  │  │ Service │  │  │  │ Service │  │  │  │ Service │  │ │
│  │  └────────┘  │  │  └────────┘  │  │  └────────┘  │ │
│  └─────────────┘  └─────────────┘  └─────────────┘ │
│                                                      │
│              Dependency Injection Container          │
└─────────────────────────────────────────────────────┘
```

### ทำไมต้องใช้ NestJS?

1. **Structure** - มี architecture ชัดเจน
2. **TypeScript** - รองรับ TypeScript เต็มรูปแบบ
3. **Decorators** - ใช้ decorators ในการกำหนด route, middleware
4. **DI Container** - มี built-in dependency injection
5. **Testing** - รองรับ unit testing ง่าย
6. **Extensible** - รองรับ library อื่นๆ เช่น Express, Fastify

---

## ขั้นตอนที่ 1102: การติดตั้งและสร้างโปรเจกต์

```bash
# ติดตั้ง NestJS CLI
npm install -g @nestjs/cli

# สร้างโปรเจกต์ใหม่
nest new my-nest-app

# เลือก package manager
# ? Which package manager would you like to use?
# ❯ npm
#   yarn
#   pnpm

cd my-nest-app

# โครงสร้างโปรเจกต์
# src/
# ├── app.controller.ts     - Controller หลัก
# ├── app.controller.spec.ts - Test สำหรับ controller
# ├── app.module.ts          - Module หลัก
# ├── app.service.ts         - Service หลัก
# └── main.ts                - Entry point

# รัน development server
npm run start:dev
```

```typescript
// src/main.ts
import { NestFactory } from "@nestjs/core";
import { AppModule } from "./app.module";
import { ValidationPipe } from "@nestjs/common";

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  
  // Global validation pipe
  app.useGlobalPipes(new ValidationPipe({
    whitelist: true,
    forbidNonWhitelisted: true,
    transform: true
  }));
  
  // Enable CORS
  app.enableCors();
  
  // API prefix
  app.setGlobalPrefix("api/v1");
  
  await app.listen(3000);
  console.log("Application is running on: http://localhost:3000");
}

bootstrap();
```

---

## ขั้นตอนที่ 1103: Modules

Module เป็นหน่วยพื้นฐานในการจัดระเบียบโค้ดใน NestJS

```typescript
// src/app.module.ts
import { Module } from "@nestjs/common";
import { TypeOrmModule } from "@nestjs/typeorm";
import { ConfigModule } from "@nestjs/config";
import { UsersModule } from "./users/users.module";
import { AuthModule } from "./auth/auth.module";

@Module({
  imports: [
    ConfigModule.forRoot({ isGlobal: true }),
    TypeOrmModule.forRoot({
      type: "postgres",
      host: process.env.DB_HOST,
      port: parseInt(process.env.DB_PORT!),
      username: process.env.DB_USER,
      password: process.env.DB_PASS,
      database: process.env.DB_NAME,
      entities: [__dirname + "/**/*.entity{.ts,.js}"],
      synchronize: process.env.NODE_ENV === "development"
    }),
    UsersModule,
    AuthModule
  ]
})
export class AppModule {}
```

```typescript
// src/users/users.module.ts
import { Module } from "@nestjs/common";
import { TypeOrmModule } from "@nestjs/typeorm";
import { UsersController } from "./users.controller";
import { UsersService } from "./users.service";
import { User } from "./entities/user.entity";
import { EmailModule } from "../email/email.module";

@Module({
  imports: [
    TypeOrmModule.forFeature([User]),
    EmailModule
  ],
  controllers: [UsersController],
  providers: [UsersService],
  exports: [UsersService]  // Export เพื่อให้ module อื่นใช้ได้
})
export class UsersModule {}
```

---

## ขั้นตอนที่ 1104: Entities และ TypeORM

```typescript
// src/users/entities/user.entity.ts
import {
  Entity,
  PrimaryGeneratedColumn,
  Column,
  CreateDateColumn,
  UpdateDateColumn,
  BeforeInsert,
  Index
} from "typeorm";
import * as bcrypt from "bcrypt";

@Entity("users")
export class User {
  @PrimaryGeneratedColumn("uuid")
  id: string;

  @Column({ length: 100 })
  name: string;

  @Index({ unique: true })
  @Column({ unique: true })
  email: string;

  @Column({ select: false }) // ไม่ include ใน SELECT โดยปริยาย
  password: string;

  @Column({ default: true })
  isActive: boolean;

  @Column({ nullable: true })
  avatarUrl: string;

  @CreateDateColumn()
  createdAt: Date;

  @UpdateDateColumn()
  updatedAt: Date;

  @BeforeInsert()
  async hashPassword() {
    if (this.password) {
      this.password = await bcrypt.hash(this.password, 12);
    }
  }

  async comparePassword(plainPassword: string): Promise<boolean> {
    return bcrypt.compare(plainPassword, this.password);
  }
}
```

---

## ขั้นตอนที่ 1105: DTOs และ Validation

```typescript
// src/users/dto/create-user.dto.ts
import {
  IsString,
  IsEmail,
  MinLength,
  MaxLength,
  IsOptional,
  Matches,
  IsUrl
} from "class-validator";
import { ApiProperty } from "@nestjs/swagger";

export class CreateUserDto {
  @ApiProperty({ example: "John Doe", description: "User full name" })
  @IsString()
  @MinLength(2)
  @MaxLength(100)
  name: string;

  @ApiProperty({ example: "john@example.com" })
  @IsEmail()
  email: string;

  @ApiProperty({ example: "SecurePass123!" })
  @IsString()
  @MinLength(8)
  @Matches(
    /^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)[a-zA-Z\d@$!%*?&]{8,}$/,
    { message: "Password must contain uppercase, lowercase, and number" }
  )
  password: string;

  @IsOptional()
  @IsUrl()
  avatarUrl?: string;
}
```

```typescript
// src/users/dto/update-user.dto.ts
import { PartialType, OmitType } from "@nestjs/mapped-types";
import { CreateUserDto } from "./create-user.dto";

export class UpdateUserDto extends PartialType(
  OmitType(CreateUserDto, ["password"] as const)
) {}
```

```typescript
// src/users/dto/user-response.dto.ts
import { Exclude, Expose } from "class-transformer";

@Exclude()
export class UserResponseDto {
  @Expose()
  id: string;

  @Expose()
  name: string;

  @Expose()
  email: string;

  @Expose()
  isActive: boolean;

  @Expose()
  createdAt: Date;

  constructor(partial: Partial<UserResponseDto>) {
    Object.assign(this, partial);
  }
}
```

---

## ขั้นตอนที่ 1106: Controllers

```typescript
// src/users/users.controller.ts
import {
  Controller,
  Get,
  Post,
  Put,
  Delete,
  Body,
  Param,
  Query,
  ParseUUIDPipe,
  HttpCode,
  HttpStatus,
  UseGuards,
  SerializeOptions,
  ClassSerializerInterceptor,
  UseInterceptors
} from "@nestjs/common";
import { UsersService } from "./users.service";
import { CreateUserDto } from "./dto/create-user.dto";
import { UpdateUserDto } from "./dto/update-user.dto";
import { UserResponseDto } from "./dto/user-response.dto";
import { JwtAuthGuard } from "../auth/guards/jwt-auth.guard";
import { Roles } from "../auth/decorators/roles.decorator";
import { RolesGuard } from "../auth/guards/roles.guard";
import { CurrentUser } from "../auth/decorators/current-user.decorator";

@Controller("users")
@UseInterceptors(ClassSerializerInterceptor)
export class UsersController {
  constructor(private readonly usersService: UsersService) {}

  @Get()
  @UseGuards(JwtAuthGuard)
  async findAll(
    @Query("page") page: number = 1,
    @Query("limit") limit: number = 10,
    @Query("search") search?: string
  ) {
    return this.usersService.findAll({ page, limit, search });
  }

  @Get("me")
  @UseGuards(JwtAuthGuard)
  async getProfile(@CurrentUser() user: any) {
    return new UserResponseDto(user);
  }

  @Get(":id")
  @UseGuards(JwtAuthGuard)
  async findOne(@Param("id", ParseUUIDPipe) id: string) {
    const user = await this.usersService.findById(id);
    return new UserResponseDto(user);
  }

  @Post()
  @HttpCode(HttpStatus.CREATED)
  async create(@Body() createUserDto: CreateUserDto) {
    const user = await this.usersService.create(createUserDto);
    return new UserResponseDto(user);
  }

  @Put(":id")
  @UseGuards(JwtAuthGuard)
  async update(
    @Param("id", ParseUUIDPipe) id: string,
    @Body() updateUserDto: UpdateUserDto,
    @CurrentUser() currentUser: any
  ) {
    const user = await this.usersService.update(id, updateUserDto, currentUser);
    return new UserResponseDto(user);
  }

  @Delete(":id")
  @HttpCode(HttpStatus.NO_CONTENT)
  @UseGuards(JwtAuthGuard, RolesGuard)
  @Roles("admin")
  async remove(@Param("id", ParseUUIDPipe) id: string) {
    await this.usersService.remove(id);
  }
}
```

---

## ขั้นตอนที่ 1107: Services

```typescript
// src/users/users.service.ts
import {
  Injectable,
  NotFoundException,
  ConflictException,
  ForbiddenException
} from "@nestjs/common";
import { InjectRepository } from "@nestjs/typeorm";
import { Repository, Like, FindManyOptions } from "typeorm";
import { User } from "./entities/user.entity";
import { CreateUserDto } from "./dto/create-user.dto";
import { UpdateUserDto } from "./dto/update-user.dto";
import { EmailService } from "../email/email.service";

interface FindAllOptions {
  page: number;
  limit: number;
  search?: string;
}

interface PaginatedResult<T> {
  data: T[];
  total: number;
  page: number;
  limit: number;
  totalPages: number;
}

@Injectable()
export class UsersService {
  constructor(
    @InjectRepository(User)
    private readonly usersRepository: Repository<User>,
    private readonly emailService: EmailService
  ) {}

  async findAll(options: FindAllOptions): Promise<PaginatedResult<User>> {
    const { page, limit, search } = options;
    const skip = (page - 1) * limit;
    
    const where: FindManyOptions<User>["where"] = search
      ? [
          { name: Like(`%${search}%`) },
          { email: Like(`%${search}%`) }
        ]
      : {};
    
    const [data, total] = await this.usersRepository.findAndCount({
      where,
      skip,
      take: limit,
      order: { createdAt: "DESC" }
    });
    
    return {
      data,
      total,
      page,
      limit,
      totalPages: Math.ceil(total / limit)
    };
  }

  async findById(id: string): Promise<User> {
    const user = await this.usersRepository.findOne({ where: { id } });
    if (!user) {
      throw new NotFoundException(`User with id ${id} not found`);
    }
    return user;
  }

  async findByEmail(email: string): Promise<User | null> {
    return this.usersRepository.findOne({ 
      where: { email },
      select: ["id", "email", "password", "name", "isActive"]
    });
  }

  async create(dto: CreateUserDto): Promise<User> {
    const existing = await this.usersRepository.findOne({
      where: { email: dto.email }
    });
    
    if (existing) {
      throw new ConflictException("Email already exists");
    }
    
    const user = this.usersRepository.create(dto);
    const saved = await this.usersRepository.save(user);
    
    // Send welcome email
    await this.emailService.sendWelcome(saved.email, saved.name);
    
    return saved;
  }

  async update(id: string, dto: UpdateUserDto, currentUser: User): Promise<User> {
    const user = await this.findById(id);
    
    if (user.id !== currentUser.id) {
      throw new ForbiddenException("Cannot update another user's profile");
    }
    
    Object.assign(user, dto);
    return this.usersRepository.save(user);
  }

  async remove(id: string): Promise<void> {
    const user = await this.findById(id);
    await this.usersRepository.remove(user);
  }
}
```

---

## ขั้นตอนที่ 1108: Authentication Module

```typescript
// src/auth/auth.module.ts
import { Module } from "@nestjs/common";
import { JwtModule } from "@nestjs/jwt";
import { PassportModule } from "@nestjs/passport";
import { ConfigModule, ConfigService } from "@nestjs/config";
import { AuthController } from "./auth.controller";
import { AuthService } from "./auth.service";
import { JwtStrategy } from "./strategies/jwt.strategy";
import { LocalStrategy } from "./strategies/local.strategy";
import { UsersModule } from "../users/users.module";

@Module({
  imports: [
    PassportModule,
    JwtModule.registerAsync({
      imports: [ConfigModule],
      useFactory: async (configService: ConfigService) => ({
        secret: configService.get<string>("JWT_SECRET"),
        signOptions: { expiresIn: configService.get<string>("JWT_EXPIRES_IN", "7d") }
      }),
      inject: [ConfigService]
    }),
    UsersModule
  ],
  controllers: [AuthController],
  providers: [AuthService, JwtStrategy, LocalStrategy],
  exports: [AuthService]
})
export class AuthModule {}
```

```typescript
// src/auth/auth.service.ts
import { Injectable, UnauthorizedException } from "@nestjs/common";
import { JwtService } from "@nestjs/jwt";
import { UsersService } from "../users/users.service";
import { User } from "../users/entities/user.entity";

@Injectable()
export class AuthService {
  constructor(
    private readonly usersService: UsersService,
    private readonly jwtService: JwtService
  ) {}

  async validateUser(email: string, password: string): Promise<User | null> {
    const user = await this.usersService.findByEmail(email);
    if (!user) return null;
    
    const isValid = await user.comparePassword(password);
    return isValid ? user : null;
  }

  async login(user: User): Promise<{ accessToken: string; user: Partial<User> }> {
    const payload = { sub: user.id, email: user.email };
    
    return {
      accessToken: this.jwtService.sign(payload),
      user: {
        id: user.id,
        name: user.name,
        email: user.email
      }
    };
  }

  async validateToken(token: string): Promise<any> {
    try {
      return this.jwtService.verify(token);
    } catch {
      throw new UnauthorizedException("Invalid token");
    }
  }
}
```

```typescript
// src/auth/strategies/jwt.strategy.ts
import { Injectable, UnauthorizedException } from "@nestjs/common";
import { PassportStrategy } from "@nestjs/passport";
import { ExtractJwt, Strategy } from "passport-jwt";
import { ConfigService } from "@nestjs/config";
import { UsersService } from "../../users/users.service";

@Injectable()
export class JwtStrategy extends PassportStrategy(Strategy) {
  constructor(
    configService: ConfigService,
    private readonly usersService: UsersService
  ) {
    super({
      jwtFromRequest: ExtractJwt.fromAuthHeaderAsBearerToken(),
      ignoreExpiration: false,
      secretOrKey: configService.get<string>("JWT_SECRET")!
    });
  }

  async validate(payload: { sub: string; email: string }) {
    const user = await this.usersService.findById(payload.sub);
    if (!user || !user.isActive) {
      throw new UnauthorizedException();
    }
    return user;
  }
}
```

---

## ขั้นตอนที่ 1109: Guards

```typescript
// src/auth/guards/jwt-auth.guard.ts
import { Injectable, ExecutionContext } from "@nestjs/common";
import { AuthGuard } from "@nestjs/passport";

@Injectable()
export class JwtAuthGuard extends AuthGuard("jwt") {
  canActivate(context: ExecutionContext) {
    return super.canActivate(context);
  }
}
```

```typescript
// src/auth/guards/roles.guard.ts
import { Injectable, CanActivate, ExecutionContext } from "@nestjs/common";
import { Reflector } from "@nestjs/core";

@Injectable()
export class RolesGuard implements CanActivate {
  constructor(private reflector: Reflector) {}

  canActivate(context: ExecutionContext): boolean {
    const requiredRoles = this.reflector.getAllAndOverride<string[]>("roles", [
      context.getHandler(),
      context.getClass()
    ]);
    
    if (!requiredRoles) return true;
    
    const { user } = context.switchToHttp().getRequest();
    return requiredRoles.some(role => user?.roles?.includes(role));
  }
}
```

```typescript
// src/auth/decorators/roles.decorator.ts
import { SetMetadata } from "@nestjs/common";

export const Roles = (...roles: string[]) => SetMetadata("roles", roles);
```

```typescript
// src/auth/decorators/current-user.decorator.ts
import { createParamDecorator, ExecutionContext } from "@nestjs/common";

export const CurrentUser = createParamDecorator(
  (data: unknown, ctx: ExecutionContext) => {
    const request = ctx.switchToHttp().getRequest();
    return request.user;
  }
);
```

---

## ขั้นตอนที่ 1110: Pipes

```typescript
// src/common/pipes/parse-int-with-default.pipe.ts
import { PipeTransform, Injectable, ArgumentMetadata } from "@nestjs/common";

@Injectable()
export class ParseIntWithDefaultPipe implements PipeTransform {
  constructor(private readonly defaultValue: number) {}

  transform(value: string, metadata: ArgumentMetadata): number {
    const parsed = parseInt(value, 10);
    return isNaN(parsed) ? this.defaultValue : parsed;
  }
}
```

```typescript
// src/common/pipes/trim.pipe.ts
import { PipeTransform, Injectable } from "@nestjs/common";

@Injectable()
export class TrimPipe implements PipeTransform {
  transform(value: any) {
    if (typeof value === "string") {
      return value.trim();
    }
    
    if (typeof value === "object" && value !== null) {
      return this.trimObject(value);
    }
    
    return value;
  }

  private trimObject(obj: Record<string, any>): Record<string, any> {
    const result: Record<string, any> = {};
    for (const [key, val] of Object.entries(obj)) {
      result[key] = typeof val === "string" ? val.trim() : val;
    }
    return result;
  }
}
```

---

## ขั้นตอนที่ 1111: Interceptors

```typescript
// src/common/interceptors/logging.interceptor.ts
import {
  Injectable,
  NestInterceptor,
  ExecutionContext,
  CallHandler,
  Logger
} from "@nestjs/common";
import { Observable } from "rxjs";
import { tap } from "rxjs/operators";
import { v4 as uuidv4 } from "uuid";

@Injectable()
export class LoggingInterceptor implements NestInterceptor {
  private readonly logger = new Logger(LoggingInterceptor.name);

  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    const request = context.switchToHttp().getRequest();
    const correlationId = uuidv4();
    request.correlationId = correlationId;
    
    const { method, url, body } = request;
    const startTime = Date.now();
    
    this.logger.log(`[${correlationId}] ${method} ${url}`);
    
    return next.handle().pipe(
      tap({
        next: (data) => {
          const duration = Date.now() - startTime;
          this.logger.log(`[${correlationId}] ${method} ${url} - ${duration}ms`);
        },
        error: (error) => {
          const duration = Date.now() - startTime;
          this.logger.error(`[${correlationId}] ${method} ${url} - ${duration}ms - Error: ${error.message}`);
        }
      })
    );
  }
}
```

```typescript
// src/common/interceptors/transform.interceptor.ts
import {
  Injectable,
  NestInterceptor,
  ExecutionContext,
  CallHandler
} from "@nestjs/common";
import { Observable } from "rxjs";
import { map } from "rxjs/operators";

export interface ApiResponse<T> {
  data: T;
  statusCode: number;
  timestamp: string;
}

@Injectable()
export class TransformInterceptor<T>
  implements NestInterceptor<T, ApiResponse<T>> {
  intercept(
    context: ExecutionContext,
    next: CallHandler
  ): Observable<ApiResponse<T>> {
    const statusCode = context.switchToHttp().getResponse().statusCode;
    
    return next.handle().pipe(
      map(data => ({
        data,
        statusCode,
        timestamp: new Date().toISOString()
      }))
    );
  }
}
```

---

## ขั้นตอนที่ 1112: Exception Filters

```typescript
// src/common/filters/http-exception.filter.ts
import {
  ExceptionFilter,
  Catch,
  ArgumentsHost,
  HttpException,
  HttpStatus,
  Logger
} from "@nestjs/common";
import { Request, Response } from "express";

@Catch(HttpException)
export class HttpExceptionFilter implements ExceptionFilter {
  private readonly logger = new Logger(HttpExceptionFilter.name);

  catch(exception: HttpException, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse<Response>();
    const request = ctx.getRequest<Request>();
    const statusCode = exception.getStatus();
    const exceptionResponse = exception.getResponse();
    
    const error = typeof exceptionResponse === "string"
      ? { message: exceptionResponse }
      : exceptionResponse as object;
    
    this.logger.error(
      `${request.method} ${request.url} ${statusCode}`,
      JSON.stringify(error)
    );
    
    response.status(statusCode).json({
      statusCode,
      timestamp: new Date().toISOString(),
      path: request.url,
      ...error
    });
  }
}

@Catch()
export class AllExceptionsFilter implements ExceptionFilter {
  private readonly logger = new Logger(AllExceptionsFilter.name);

  catch(exception: unknown, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse<Response>();
    const request = ctx.getRequest<Request>();
    
    const statusCode = exception instanceof HttpException
      ? exception.getStatus()
      : HttpStatus.INTERNAL_SERVER_ERROR;
    
    const message = exception instanceof Error
      ? exception.message
      : "Internal server error";
    
    this.logger.error(`Unhandled exception: ${message}`, exception instanceof Error ? exception.stack : "");
    
    response.status(statusCode).json({
      statusCode,
      timestamp: new Date().toISOString(),
      path: request.url,
      message: process.env.NODE_ENV === "production" 
        ? "Internal server error" 
        : message
    });
  }
}
```

---

## ขั้นตอนที่ 1113: Custom Decorators

```typescript
// src/common/decorators/api-paginated-response.decorator.ts
import { applyDecorators, Type } from "@nestjs/common";
import { ApiOkResponse, getSchemaPath } from "@nestjs/swagger";

export function ApiPaginatedResponse<TModel extends Type<any>>(model: TModel) {
  return applyDecorators(
    ApiOkResponse({
      schema: {
        allOf: [
          {
            properties: {
              data: {
                type: "array",
                items: { $ref: getSchemaPath(model) }
              },
              total: { type: "number" },
              page: { type: "number" },
              limit: { type: "number" },
              totalPages: { type: "number" }
            }
          }
        ]
      }
    })
  );
}
```

```typescript
// src/common/decorators/public.decorator.ts
import { SetMetadata } from "@nestjs/common";

export const IS_PUBLIC_KEY = "isPublic";
export const Public = () => SetMetadata(IS_PUBLIC_KEY, true);
```

---

## ขั้นตอนที่ 1114: Configuration Module

```typescript
// src/config/configuration.ts
import { registerAs } from "@nestjs/config";

export const databaseConfig = registerAs("database", () => ({
  host: process.env.DB_HOST ?? "localhost",
  port: parseInt(process.env.DB_PORT ?? "5432", 10),
  username: process.env.DB_USER ?? "postgres",
  password: process.env.DB_PASS ?? "postgres",
  database: process.env.DB_NAME ?? "myapp"
}));

export const jwtConfig = registerAs("jwt", () => ({
  secret: process.env.JWT_SECRET,
  expiresIn: process.env.JWT_EXPIRES_IN ?? "7d"
}));

export const appConfig = registerAs("app", () => ({
  port: parseInt(process.env.PORT ?? "3000", 10),
  env: process.env.NODE_ENV ?? "development",
  apiPrefix: process.env.API_PREFIX ?? "api/v1"
}));
```

```typescript
// src/app.module.ts (updated)
import { Module } from "@nestjs/common";
import { ConfigModule } from "@nestjs/config";
import { databaseConfig, jwtConfig, appConfig } from "./config/configuration";
import * as Joi from "joi";

@Module({
  imports: [
    ConfigModule.forRoot({
      isGlobal: true,
      load: [databaseConfig, jwtConfig, appConfig],
      validationSchema: Joi.object({
        NODE_ENV: Joi.string().valid("development", "production", "test").default("development"),
        PORT: Joi.number().default(3000),
        DB_HOST: Joi.string().required(),
        DB_PORT: Joi.number().default(5432),
        DB_USER: Joi.string().required(),
        DB_PASS: Joi.string().required(),
        DB_NAME: Joi.string().required(),
        JWT_SECRET: Joi.string().min(32).required(),
        JWT_EXPIRES_IN: Joi.string().default("7d")
      })
    })
  ]
})
export class AppModule {}
```

---

## ขั้นตอนที่ 1115: Swagger Documentation

```typescript
// src/main.ts (with Swagger)
import { NestFactory } from "@nestjs/core";
import { SwaggerModule, DocumentBuilder } from "@nestjs/swagger";
import { AppModule } from "./app.module";

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  
  // Swagger setup
  const config = new DocumentBuilder()
    .setTitle("My API")
    .setDescription("The My API description")
    .setVersion("1.0")
    .addBearerAuth(
      { type: "http", scheme: "bearer", bearerFormat: "JWT" },
      "JWT-auth"
    )
    .addTag("users", "User operations")
    .addTag("auth", "Authentication operations")
    .build();
    
  const document = SwaggerModule.createDocument(app, config);
  SwaggerModule.setup("api/docs", app, document, {
    swaggerOptions: {
      persistAuthorization: true
    }
  });
  
  await app.listen(3000);
}
```

```typescript
// src/users/users.controller.ts (with Swagger decorators)
import {
  ApiTags,
  ApiBearerAuth,
  ApiOperation,
  ApiResponse,
  ApiQuery
} from "@nestjs/swagger";

@ApiTags("users")
@ApiBearerAuth("JWT-auth")
@Controller("users")
export class UsersController {
  @Get()
  @ApiOperation({ summary: "Get all users" })
  @ApiQuery({ name: "page", required: false, type: Number })
  @ApiQuery({ name: "limit", required: false, type: Number })
  @ApiQuery({ name: "search", required: false, type: String })
  @ApiResponse({ status: 200, description: "Returns paginated list of users" })
  async findAll() {}

  @Post()
  @ApiOperation({ summary: "Create new user" })
  @ApiResponse({ status: 201, description: "User created successfully" })
  @ApiResponse({ status: 400, description: "Validation error" })
  @ApiResponse({ status: 409, description: "Email already exists" })
  async create() {}
}
```

---

## ขั้นตอนที่ 1116: Testing ใน NestJS

```typescript
// src/users/tests/users.service.spec.ts
import { Test, TestingModule } from "@nestjs/testing";
import { getRepositoryToken } from "@nestjs/typeorm";
import { Repository } from "typeorm";
import { UsersService } from "../users.service";
import { User } from "../entities/user.entity";
import { EmailService } from "../../email/email.service";
import { ConflictException, NotFoundException } from "@nestjs/common";

describe("UsersService", () => {
  let service: UsersService;
  let usersRepository: jest.Mocked<Repository<User>>;
  let emailService: jest.Mocked<EmailService>;

  beforeEach(async () => {
    const module: TestingModule = await Test.createTestingModule({
      providers: [
        UsersService,
        {
          provide: getRepositoryToken(User),
          useValue: {
            findOne: jest.fn(),
            findAndCount: jest.fn(),
            create: jest.fn(),
            save: jest.fn(),
            remove: jest.fn()
          }
        },
        {
          provide: EmailService,
          useValue: {
            sendWelcome: jest.fn().mockResolvedValue(undefined)
          }
        }
      ]
    }).compile();

    service = module.get<UsersService>(UsersService);
    usersRepository = module.get(getRepositoryToken(User));
    emailService = module.get(EmailService);
  });

  describe("create", () => {
    it("should create user and send welcome email", async () => {
      const dto = { name: "Alice", email: "alice@test.com", password: "Pass123!" };
      const mockUser = { id: "1", ...dto, isActive: true, createdAt: new Date(), updatedAt: new Date() };
      
      usersRepository.findOne.mockResolvedValue(null);
      usersRepository.create.mockReturnValue(mockUser as User);
      usersRepository.save.mockResolvedValue(mockUser as User);
      
      const result = await service.create(dto);
      
      expect(result).toEqual(mockUser);
      expect(emailService.sendWelcome).toHaveBeenCalledWith(dto.email, dto.name);
    });

    it("should throw ConflictException for duplicate email", async () => {
      usersRepository.findOne.mockResolvedValue({ id: "1" } as User);
      
      await expect(service.create({
        name: "Alice",
        email: "alice@test.com",
        password: "Pass123!"
      })).rejects.toThrow(ConflictException);
    });
  });
});
```

---

## ขั้นตอนที่ 1117: Health Checks

```typescript
// npm install @nestjs/terminus

// src/health/health.module.ts
import { Module } from "@nestjs/common";
import { TerminusModule } from "@nestjs/terminus";
import { HealthController } from "./health.controller";
import { TypeOrmHealthIndicator } from "@nestjs/terminus";

@Module({
  imports: [TerminusModule],
  controllers: [HealthController]
})
export class HealthModule {}
```

```typescript
// src/health/health.controller.ts
import { Controller, Get } from "@nestjs/common";
import {
  HealthCheck,
  HealthCheckService,
  TypeOrmHealthIndicator,
  MemoryHealthIndicator,
  DiskHealthIndicator
} from "@nestjs/terminus";

@Controller("health")
export class HealthController {
  constructor(
    private health: HealthCheckService,
    private db: TypeOrmHealthIndicator,
    private memory: MemoryHealthIndicator,
    private disk: DiskHealthIndicator
  ) {}

  @Get()
  @HealthCheck()
  check() {
    return this.health.check([
      () => this.db.pingCheck("database"),
      () => this.memory.checkHeap("memory_heap", 300 * 1024 * 1024), // 300MB
      () => this.memory.checkRSS("memory_rss", 300 * 1024 * 1024),
      () => this.disk.checkStorage("storage", {
        thresholdPercent: 0.9,
        path: "/"
      })
    ]);
  }
}
```

---

## ขั้นตอนที่ 1118: Caching

```typescript
// npm install cache-manager @nestjs/cache-manager

// src/app.module.ts
import { CacheModule } from "@nestjs/cache-manager";
import * as redisStore from "cache-manager-redis-store";

@Module({
  imports: [
    CacheModule.registerAsync({
      isGlobal: true,
      useFactory: () => ({
        store: redisStore,
        host: process.env.REDIS_HOST,
        port: parseInt(process.env.REDIS_PORT!),
        ttl: 60 * 5 // 5 minutes default
      })
    })
  ]
})
export class AppModule {}
```

```typescript
// Using cache in service
import { Injectable } from "@nestjs/common";
import { Cache } from "cache-manager";
import { InjectCache } from "@nestjs/cache-manager";

@Injectable()
export class ProductsService {
  constructor(
    @InjectCache() private cacheManager: Cache
  ) {}

  async findById(id: string) {
    const cacheKey = `product:${id}`;
    
    const cached = await this.cacheManager.get(cacheKey);
    if (cached) return cached;
    
    const product = await this.productRepository.findOne({ where: { id } });
    
    await this.cacheManager.set(cacheKey, product, 300); // 5 minutes
    
    return product;
  }
}
```

---

## ขั้นตอนที่ 1119: Queue Processing ด้วย Bull

```typescript
// npm install @nestjs/bull bull

// src/queue/queue.module.ts
import { Module } from "@nestjs/common";
import { BullModule } from "@nestjs/bull";
import { EmailProcessor } from "./processors/email.processor";
import { EmailService } from "../email/email.service";

@Module({
  imports: [
    BullModule.registerQueue({ name: "email" }),
    BullModule.registerQueue({ name: "notifications" })
  ],
  providers: [EmailProcessor, EmailService],
  exports: [BullModule]
})
export class QueueModule {}
```

```typescript
// src/queue/processors/email.processor.ts
import { Processor, Process, OnQueueFailed } from "@nestjs/bull";
import { Job } from "bull";
import { Logger } from "@nestjs/common";
import { EmailService } from "../../email/email.service";

@Processor("email")
export class EmailProcessor {
  private readonly logger = new Logger(EmailProcessor.name);

  constructor(private readonly emailService: EmailService) {}

  @Process("welcome")
  async handleWelcome(job: Job<{ email: string; name: string }>) {
    const { email, name } = job.data;
    this.logger.log(`Processing welcome email for ${email}`);
    await this.emailService.sendWelcome(email, name);
  }

  @Process("password-reset")
  async handlePasswordReset(job: Job<{ email: string; token: string }>) {
    const { email, token } = job.data;
    await this.emailService.sendPasswordReset(email, token);
  }

  @OnQueueFailed()
  onFailed(job: Job, error: Error) {
    this.logger.error(`Job ${job.id} failed: ${error.message}`);
  }
}
```

---

## ขั้นตอนที่ 1120: WebSockets กับ NestJS

```typescript
// npm install @nestjs/websockets @nestjs/platform-socket.io socket.io

// src/events/events.gateway.ts
import {
  WebSocketGateway,
  SubscribeMessage,
  MessageBody,
  WebSocketServer,
  ConnectedSocket,
  OnGatewayInit,
  OnGatewayConnection,
  OnGatewayDisconnect
} from "@nestjs/websockets";
import { Server, Socket } from "socket.io";
import { Logger } from "@nestjs/common";
import { UseGuards } from "@nestjs/common";
import { WsJwtGuard } from "./guards/ws-jwt.guard";

@WebSocketGateway({
  cors: { origin: "*" },
  namespace: "/events"
})
export class EventsGateway implements OnGatewayInit, OnGatewayConnection, OnGatewayDisconnect {
  @WebSocketServer() server: Server;
  private logger = new Logger(EventsGateway.name);

  afterInit(server: Server) {
    this.logger.log("WebSocket Gateway initialized");
  }

  handleConnection(client: Socket) {
    this.logger.log(`Client connected: ${client.id}`);
  }

  handleDisconnect(client: Socket) {
    this.logger.log(`Client disconnected: ${client.id}`);
  }

  @SubscribeMessage("message")
  @UseGuards(WsJwtGuard)
  handleMessage(
    @ConnectedSocket() client: Socket,
    @MessageBody() data: { room: string; content: string }
  ) {
    this.server.to(data.room).emit("message", {
      from: client.id,
      content: data.content,
      timestamp: new Date()
    });
  }

  @SubscribeMessage("join-room")
  handleJoinRoom(
    @ConnectedSocket() client: Socket,
    @MessageBody() room: string
  ) {
    client.join(room);
    client.emit("joined", { room, timestamp: new Date() });
  }

  // Emit to specific user
  async sendToUser(userId: string, event: string, data: any) {
    this.server.to(`user:${userId}`).emit(event, data);
  }
}
```

---

## 🏋️ แบบฝึกหัด

### แบบฝึกหัดที่ 1: สร้าง NestJS CRUD API
1. สร้าง NestJS app พื้นฐาน
2. สร้าง Products module ด้วย TypeORM
3. implement CRUD operations ครบถ้วน

### แบบฝึกหัดที่ 2: Authentication
1. implement JWT authentication
2. สร้าง login/register endpoints
3. protect routes ด้วย JwtAuthGuard

### แบบฝึกหัดที่ 3: Caching และ Queue
1. ติดตั้ง Redis caching
2. สร้าง Bull queue สำหรับ email sending
3. implement cache invalidation

### แบบฝึกหัดที่ 4: Testing
1. เขียน unit tests สำหรับ services
2. เขียน e2e tests สำหรับ endpoints
3. achieve 80% code coverage

---

## 📚 สรุป

ใน Part นี้เราได้เรียนรู้:
- Architecture ของ NestJS (Modules, Controllers, Services)
- Dependency Injection Container
- Authentication ด้วย Passport และ JWT
- Guards, Pipes, Interceptors, Exception Filters
- Swagger Documentation
- Testing patterns ใน NestJS
- Caching ด้วย Redis
- Queue processing ด้วย Bull
- WebSockets

**Part ถัดไป**: Advanced NestJS - Interceptors, Guards, Pipes, Microservices
