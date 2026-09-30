# Part 71 | ขั้นตอนที่ 1221-1240 จาก 1000+

# Feature Flags และ A/B Testing

---

## 🎯 เป้าหมายของ Part นี้

หลังจากเรียนจบ Part นี้ คุณจะสามารถ:
1. implement Feature Flags ใน Node.js
2. สร้าง A/B Testing framework
3. ใช้ LaunchDarkly SDK
4. implement Gradual Rollouts
5. วิเคราะห์ผลลัพธ์ของ experiments
6. สร้าง Feature Flag management system

---

## ขั้นตอนที่ 1221: Feature Flags คืออะไร?

Feature Flags (Feature Toggles) คือ technique ที่ช่วยให้ deploy code ที่ยังไม่พร้อมสำหรับทุกคน และค่อยๆ เปิดให้ผู้ใช้ส่วนหนึ่งก่อน

```
Feature Flag Workflow:
                                                    
Deploy → All Users See Old Feature                  
         │                                          
         ├── Flag = 10% → 10% Users See New Feature  
         │                                          
         ├── Flag = 50% → 50% Users See New Feature  
         │                                          
         ├── Flag = 100% → All Users See New Feature  
         │                                          
         └── Remove Flag from Code                  

Benefits:
- Safe deployments
- A/B testing
- Gradual rollouts
- Kill switch for bugs
- Dark launches
```

---

## ขั้นตอนที่ 1222: Simple Feature Flag Implementation

```typescript
// src/feature-flags/feature-flag.ts
export interface FeatureFlagConfig {
  key: string;
  enabled: boolean;
  rolloutPercentage?: number; // 0-100
  whitelist?: string[];       // User IDs always enabled
  blacklist?: string[];       // User IDs always disabled
  environments?: string[];    // Enabled environments
  startDate?: Date;           // Not enabled before this date
  endDate?: Date;             // Not enabled after this date
}

export class FeatureFlag {
  constructor(private readonly config: FeatureFlagConfig) {}

  isEnabled(context?: {
    userId?: string;
    environment?: string;
    attributes?: Record<string, any>;
  }): boolean {
    // Check if globally disabled
    if (!this.config.enabled) return false;
    
    // Check date range
    const now = new Date();
    if (this.config.startDate && now < this.config.startDate) return false;
    if (this.config.endDate && now > this.config.endDate) return false;
    
    // Check environment
    if (context?.environment && this.config.environments?.length) {
      if (!this.config.environments.includes(context.environment)) return false;
    }
    
    // Check blacklist
    if (context?.userId && this.config.blacklist?.includes(context.userId)) {
      return false;
    }
    
    // Check whitelist
    if (context?.userId && this.config.whitelist?.includes(context.userId)) {
      return true;
    }
    
    // Gradual rollout
    if (this.config.rolloutPercentage !== undefined && context?.userId) {
      return this.isInRollout(context.userId, this.config.rolloutPercentage);
    }
    
    return true;
  }

  // Consistent hashing for gradual rollout
  private isInRollout(userId: string, percentage: number): boolean {
    const hash = this.hashString(`${this.config.key}:${userId}`);
    const bucket = hash % 100;
    return bucket < percentage;
  }

  private hashString(str: string): number {
    let hash = 0;
    for (let i = 0; i < str.length; i++) {
      const char = str.charCodeAt(i);
      hash = ((hash << 5) - hash) + char;
      hash = hash & hash;
    }
    return Math.abs(hash);
  }
}
```

---

## ขั้นตอนที่ 1223: Feature Flag Service

```typescript
// src/feature-flags/feature-flag.service.ts
import { Injectable, Logger, OnModuleInit } from "@nestjs/common";
import { InjectRepository } from "@nestjs/typeorm";
import { Repository } from "typeorm";
import { Cache } from "cache-manager";
import { InjectCache } from "@nestjs/cache-manager";
import { FeatureFlagEntity } from "./entities/feature-flag.entity";
import { FeatureFlag } from "./feature-flag";

@Injectable()
export class FeatureFlagService implements OnModuleInit {
  private readonly logger = new Logger(FeatureFlagService.name);
  private flags = new Map<string, FeatureFlag>();
  private readonly CACHE_TTL = 60; // 1 minute

  constructor(
    @InjectRepository(FeatureFlagEntity)
    private readonly flagRepository: Repository<FeatureFlagEntity>,
    @InjectCache() private readonly cache: Cache
  ) {}

  async onModuleInit() {
    await this.loadFlags();
  }

  async loadFlags(): Promise<void> {
    const configs = await this.flagRepository.find({ where: { isActive: true } });
    
    for (const config of configs) {
      this.flags.set(config.key, new FeatureFlag({
        key: config.key,
        enabled: config.enabled,
        rolloutPercentage: config.rolloutPercentage,
        whitelist: config.whitelist,
        blacklist: config.blacklist,
        environments: config.environments,
        startDate: config.startDate,
        endDate: config.endDate
      }));
    }
    
    this.logger.log(`Loaded ${this.flags.size} feature flags`);
  }

  async isEnabled(
    flagKey: string,
    context?: {
      userId?: string;
      environment?: string;
      attributes?: Record<string, any>;
    }
  ): Promise<boolean> {
    // Check cache first
    const cacheKey = `ff:${flagKey}:${context?.userId ?? "anon"}`;
    const cached = await this.cache.get<boolean>(cacheKey);
    if (cached !== undefined) return cached;
    
    const flag = this.flags.get(flagKey);
    if (!flag) {
      this.logger.warn(`Feature flag "${flagKey}" not found`);
      return false;
    }
    
    const result = flag.isEnabled(context);
    
    // Cache result
    await this.cache.set(cacheKey, result, this.CACHE_TTL);
    
    // Track exposure
    this.trackExposure(flagKey, context?.userId, result);
    
    return result;
  }

  async getVariant(
    flagKey: string,
    userId: string,
    variants: string[]
  ): Promise<string> {
    // Consistent assignment based on user ID
    const hash = this.hashUserId(flagKey, userId);
    return variants[hash % variants.length];
  }

  private hashUserId(flagKey: string, userId: string): number {
    const str = `${flagKey}:${userId}`;
    let hash = 0;
    for (let i = 0; i < str.length; i++) {
      hash = ((hash << 5) - hash) + str.charCodeAt(i);
      hash = hash & hash;
    }
    return Math.abs(hash);
  }

  private async trackExposure(
    flagKey: string,
    userId: string | undefined,
    value: boolean
  ): Promise<void> {
    // Store exposure for analytics
  }

  async createFlag(config: any): Promise<FeatureFlagEntity> {
    const flag = this.flagRepository.create(config);
    const saved = await this.flagRepository.save(flag);
    
    // Reload flags
    await this.loadFlags();
    
    return saved;
  }

  async updateFlag(key: string, updates: any): Promise<void> {
    await this.flagRepository.update({ key }, updates);
    await this.loadFlags();
  }

  async deleteFlag(key: string): Promise<void> {
    await this.flagRepository.update({ key }, { isActive: false });
    this.flags.delete(key);
  }
}
```

---

## ขั้นตอนที่ 1224: Feature Flag Entity

```typescript
// src/feature-flags/entities/feature-flag.entity.ts
import {
  Entity, PrimaryGeneratedColumn, Column,
  CreateDateColumn, UpdateDateColumn
} from "typeorm";

@Entity("feature_flags")
export class FeatureFlagEntity {
  @PrimaryGeneratedColumn("uuid")
  id: string;

  @Column({ unique: true })
  key: string;

  @Column()
  name: string;

  @Column({ type: "text", nullable: true })
  description?: string;

  @Column({ default: false })
  enabled: boolean;

  @Column({ type: "int", nullable: true })
  rolloutPercentage?: number;

  @Column("simple-array", { nullable: true })
  whitelist?: string[];

  @Column("simple-array", { nullable: true })
  blacklist?: string[];

  @Column("simple-array", { nullable: true })
  environments?: string[];

  @Column({ nullable: true })
  startDate?: Date;

  @Column({ nullable: true })
  endDate?: Date;

  @Column({ default: true })
  isActive: boolean;

  @Column({ type: "jsonb", nullable: true })
  metadata?: Record<string, any>;

  @CreateDateColumn()
  createdAt: Date;

  @UpdateDateColumn()
  updatedAt: Date;
}
```

---

## ขั้นตอนที่ 1225: Feature Flag Decorator

```typescript
// src/feature-flags/decorators/feature-flag.decorator.ts
import { SetMetadata } from "@nestjs/common";

export const FEATURE_FLAG_KEY = "featureFlag";

export interface FeatureFlagOptions {
  key: string;
  fallback?: any;  // Return this when flag is disabled
}

export const RequireFlag = (options: FeatureFlagOptions) =>
  SetMetadata(FEATURE_FLAG_KEY, options);
```

```typescript
// src/feature-flags/guards/feature-flag.guard.ts
import {
  Injectable, CanActivate, ExecutionContext,
  NotFoundException
} from "@nestjs/common";
import { Reflector } from "@nestjs/core";
import { FeatureFlagService } from "../feature-flag.service";
import { FEATURE_FLAG_KEY, FeatureFlagOptions } from "../decorators/feature-flag.decorator";

@Injectable()
export class FeatureFlagGuard implements CanActivate {
  constructor(
    private readonly reflector: Reflector,
    private readonly featureFlagService: FeatureFlagService
  ) {}

  async canActivate(context: ExecutionContext): Promise<boolean> {
    const options = this.reflector.get<FeatureFlagOptions>(
      FEATURE_FLAG_KEY,
      context.getHandler()
    );
    
    if (!options) return true;
    
    const request = context.switchToHttp().getRequest();
    const userId = request.user?.id;
    
    const enabled = await this.featureFlagService.isEnabled(options.key, {
      userId,
      environment: process.env.NODE_ENV
    });
    
    if (!enabled) {
      throw new NotFoundException("Feature not available");
    }
    
    return true;
  }
}
```

```typescript
// Usage in controller
import { Controller, Get, UseGuards } from "@nestjs/common";

@Controller("products")
export class ProductsController {
  @Get("new-layout")
  @RequireFlag({ key: "new-product-layout" })
  @UseGuards(FeatureFlagGuard)
  getNewLayout() {
    return { layout: "new" };
  }
}
```

---

## ขั้นตอนที่ 1226: A/B Testing Framework

```typescript
// src/ab-testing/ab-test.service.ts
import { Injectable } from "@nestjs/common";
import { InjectRepository } from "@nestjs/typeorm";
import { Repository } from "typeorm";
import { ExperimentEntity } from "./entities/experiment.entity";
import { ExperimentResultEntity } from "./entities/experiment-result.entity";

interface Experiment {
  id: string;
  name: string;
  variants: Array<{
    key: string;
    name: string;
    weight: number;    // 0-100
    config?: Record<string, any>;
  }>;
  startDate: Date;
  endDate?: Date;
  targetUserPercentage: number;  // 0-100
}

interface Assignment {
  experimentId: string;
  variantKey: string;
  userId: string;
  assignedAt: Date;
}

@Injectable()
export class ABTestService {
  constructor(
    @InjectRepository(ExperimentEntity)
    private readonly experimentRepository: Repository<ExperimentEntity>,
    @InjectRepository(ExperimentResultEntity)
    private readonly resultRepository: Repository<ExperimentResultEntity>
  ) {}

  // Get experiment variant for user
  async getVariant(
    experimentKey: string,
    userId: string
  ): Promise<{ variantKey: string; config?: Record<string, any> } | null> {
    const experiment = await this.experimentRepository.findOne({
      where: { key: experimentKey, isActive: true }
    });
    
    if (!experiment) return null;
    
    // Check if experiment is running
    const now = new Date();
    if (now < experiment.startDate) return null;
    if (experiment.endDate && now > experiment.endDate) return null;
    
    // Check if user should participate
    if (!this.isUserInExperiment(userId, experiment.targetUserPercentage)) {
      return null;
    }
    
    // Get consistent variant assignment
    const variantKey = this.assignVariant(
      userId,
      experimentKey,
      experiment.variants
    );
    
    const variant = experiment.variants.find(v => v.key === variantKey);
    
    // Track exposure
    await this.trackExposure(experiment.id, userId, variantKey);
    
    return variant ? { variantKey, config: variant.config } : null;
  }

  private isUserInExperiment(userId: string, percentage: number): boolean {
    const hash = this.hashString(userId) % 100;
    return hash < percentage;
  }

  private assignVariant(
    userId: string,
    experimentKey: string,
    variants: Array<{ key: string; weight: number }>
  ): string {
    const hash = this.hashString(`${experimentKey}:${userId}`) % 100;
    
    let cumulative = 0;
    for (const variant of variants) {
      cumulative += variant.weight;
      if (hash < cumulative) return variant.key;
    }
    
    return variants[variants.length - 1].key;
  }

  // Track experiment outcome
  async trackConversion(
    experimentKey: string,
    userId: string,
    eventName: string,
    value?: number
  ): Promise<void> {
    await this.resultRepository.save({
      experimentKey,
      userId,
      eventName,
      value,
      timestamp: new Date()
    });
  }

  // Analyze experiment results
  async analyzeResults(experimentKey: string) {
    const results = await this.resultRepository
      .createQueryBuilder("result")
      .where("result.experimentKey = :key", { key: experimentKey })
      .getMany();
    
    const byVariant: Record<string, {
      exposures: number;
      conversions: number;
      conversionRate: number;
      avgValue: number;
    }> = {};
    
    for (const result of results) {
      if (!byVariant[result.variantKey]) {
        byVariant[result.variantKey] = {
          exposures: 0,
          conversions: 0,
          conversionRate: 0,
          avgValue: 0
        };
      }
      
      const variant = byVariant[result.variantKey];
      variant.exposures++;
      if (result.eventName === "conversion") {
        variant.conversions++;
        variant.avgValue = (variant.avgValue * (variant.conversions - 1) + (result.value ?? 0)) / variant.conversions;
      }
      variant.conversionRate = variant.conversions / variant.exposures;
    }
    
    return this.calculateStatisticalSignificance(byVariant);
  }

  private calculateStatisticalSignificance(variants: Record<string, any>) {
    // Chi-square test for statistical significance
    const control = variants["control"];
    const results: Record<string, any> = {};
    
    for (const [key, variant] of Object.entries(variants)) {
      if (key === "control") continue;
      
      const lift = ((variant as any).conversionRate - control.conversionRate) / control.conversionRate;
      const pValue = this.chiSquareTest(
        control.conversions, control.exposures - control.conversions,
        (variant as any).conversions, (variant as any).exposures - (variant as any).conversions
      );
      
      results[key] = {
        ...variant,
        lift: lift * 100,
        pValue,
        isSignificant: pValue < 0.05
      };
    }
    
    return { control, variants: results };
  }

  private chiSquareTest(
    a: number, b: number, c: number, d: number
  ): number {
    const n = a + b + c + d;
    const expected_a = ((a + c) * (a + b)) / n;
    const expected_b = ((b + d) * (a + b)) / n;
    const expected_c = ((a + c) * (c + d)) / n;
    const expected_d = ((b + d) * (c + d)) / n;
    
    const chi2 = 
      Math.pow(a - expected_a, 2) / expected_a +
      Math.pow(b - expected_b, 2) / expected_b +
      Math.pow(c - expected_c, 2) / expected_c +
      Math.pow(d - expected_d, 2) / expected_d;
    
    // Approximate p-value for chi-square with 1 df
    return Math.exp(-0.5 * chi2);
  }

  private hashString(str: string): number {
    let hash = 0;
    for (let i = 0; i < str.length; i++) {
      hash = ((hash << 5) - hash) + str.charCodeAt(i);
      hash = hash & hash;
    }
    return Math.abs(hash);
  }

  private async trackExposure(
    experimentId: string,
    userId: string,
    variantKey: string
  ): Promise<void> {
    await this.resultRepository.save({
      experimentId,
      userId,
      variantKey,
      eventName: "exposure",
      timestamp: new Date()
    });
  }
}
```

---

## ขั้นตอนที่ 1227: LaunchDarkly Integration

```typescript
// npm install launchdarkly-node-server-sdk

// src/feature-flags/launchdarkly.service.ts
import { Injectable, OnModuleInit, OnModuleDestroy, Logger } from "@nestjs/common";
import { init, LDClient, LDContext } from "launchdarkly-node-server-sdk";

@Injectable()
export class LaunchDarklyService implements OnModuleInit, OnModuleDestroy {
  private readonly logger = new Logger(LaunchDarklyService.name);
  private client: LDClient;

  async onModuleInit() {
    this.client = init(process.env.LAUNCHDARKLY_SDK_KEY!);
    
    await this.client.waitForInitialization();
    this.logger.log("LaunchDarkly client initialized");
  }

  async onModuleDestroy() {
    await this.client.close();
  }

  // Check boolean flag
  async isEnabled(
    flagKey: string,
    userId: string,
    userAttributes?: Record<string, any>
  ): Promise<boolean> {
    const context: LDContext = {
      kind: "user",
      key: userId,
      ...userAttributes
    };
    
    return this.client.variation(flagKey, context, false);
  }

  // Get flag variation
  async getVariation<T>(
    flagKey: string,
    userId: string,
    defaultValue: T,
    userAttributes?: Record<string, any>
  ): Promise<T> {
    const context: LDContext = {
      kind: "user",
      key: userId,
      ...userAttributes
    };
    
    return this.client.variation(flagKey, context, defaultValue);
  }

  // Get string variation (A/B test)
  async getStringVariant(
    flagKey: string,
    userId: string,
    defaultVariant: string = "control"
  ): Promise<string> {
    const context: LDContext = { kind: "user", key: userId };
    return this.client.variation(flagKey, context, defaultVariant);
  }

  // Get JSON variation (feature config)
  async getJSONVariant(
    flagKey: string,
    userId: string,
    defaultValue: object = {}
  ): Promise<object> {
    const context: LDContext = { kind: "user", key: userId };
    return this.client.variation(flagKey, context, defaultValue);
  }

  // Track event (for analytics)
  async trackEvent(
    eventName: string,
    userId: string,
    data?: Record<string, any>,
    metricValue?: number
  ): Promise<void> {
    const context: LDContext = { kind: "user", key: userId };
    this.client.track(eventName, context, data, metricValue);
  }

  // Get all flags for user (for bootstrapping client)
  async getAllFlags(userId: string): Promise<Record<string, any>> {
    const context: LDContext = { kind: "user", key: userId };
    const state = await this.client.allFlagsState(context);
    return state.toValuesMap();
  }
}
```

---

## ขั้นตอนที่ 1228: Gradual Rollout

```typescript
// src/feature-flags/rollout.service.ts
import { Injectable } from "@nestjs/common";
import { FeatureFlagService } from "./feature-flag.service";

interface RolloutConfig {
  flagKey: string;
  stages: Array<{
    name: string;
    percentage: number;
    waitDays: number;
    successCriteria: {
      errorRate: number;    // Max acceptable error rate
      latency?: number;     // Max acceptable latency (ms)
    };
  }>;
}

@Injectable()
export class GradualRolloutService {
  private activeRollouts = new Map<string, { currentStage: number; startDate: Date }>();

  constructor(private readonly flagService: FeatureFlagService) {}

  async startRollout(config: RolloutConfig): Promise<void> {
    this.activeRollouts.set(config.flagKey, {
      currentStage: 0,
      startDate: new Date()
    });
    
    // Set initial rollout percentage
    await this.flagService.updateFlag(config.flagKey, {
      enabled: true,
      rolloutPercentage: config.stages[0].percentage
    });
    
    console.log(`Started rollout for ${config.flagKey} at ${config.stages[0].percentage}%`);
  }

  async advanceRollout(config: RolloutConfig): Promise<boolean> {
    const rollout = this.activeRollouts.get(config.flagKey);
    if (!rollout) return false;
    
    const currentStage = config.stages[rollout.currentStage];
    
    // Check success criteria
    const metrics = await this.getMetrics(config.flagKey);
    
    if (metrics.errorRate > currentStage.successCriteria.errorRate) {
      console.error(`Rollout failed: error rate ${metrics.errorRate} > threshold ${currentStage.successCriteria.errorRate}`);
      await this.rollback(config.flagKey);
      return false;
    }
    
    // Advance to next stage
    rollout.currentStage++;
    
    if (rollout.currentStage >= config.stages.length) {
      console.log(`Rollout complete for ${config.flagKey}: 100%`);
      return true;
    }
    
    const nextStage = config.stages[rollout.currentStage];
    await this.flagService.updateFlag(config.flagKey, {
      rolloutPercentage: nextStage.percentage
    });
    
    console.log(`Advanced rollout for ${config.flagKey} to ${nextStage.percentage}%`);
    return false;
  }

  async rollback(flagKey: string): Promise<void> {
    await this.flagService.updateFlag(flagKey, {
      enabled: false,
      rolloutPercentage: 0
    });
    
    this.activeRollouts.delete(flagKey);
    console.log(`Rolled back feature flag: ${flagKey}`);
  }

  private async getMetrics(flagKey: string): Promise<{ errorRate: number; latency: number }> {
    // Get from monitoring system
    return { errorRate: 0.01, latency: 200 };
  }
}
```

---

## ขั้นตอนที่ 1229: Feature Flag Middleware

```typescript
// src/feature-flags/middleware/feature-flag.middleware.ts
import { Injectable, NestMiddleware } from "@nestjs/common";
import { Request, Response, NextFunction } from "express";
import { FeatureFlagService } from "../feature-flag.service";
import { LaunchDarklyService } from "../launchdarkly.service";

@Injectable()
export class FeatureFlagMiddleware implements NestMiddleware {
  constructor(
    private readonly featureFlagService: FeatureFlagService,
    private readonly launchDarkly: LaunchDarklyService
  ) {}

  async use(req: Request, res: Response, next: NextFunction) {
    const userId = (req as any).user?.id;
    
    if (userId) {
      // Preload all flags for this user (for performance)
      const allFlags = await this.launchDarkly.getAllFlags(userId);
      (req as any).featureFlags = allFlags;
    }
    
    next();
  }
}
```

---

## ขั้นตอนที่ 1230: Feature Flag Dashboard API

```typescript
// src/feature-flags/feature-flags.controller.ts
import {
  Controller, Get, Post, Put, Delete,
  Body, Param, Query, UseGuards
} from "@nestjs/common";
import { FeatureFlagService } from "./feature-flag.service";
import { ABTestService } from "../ab-testing/ab-test.service";
import { Roles } from "../auth/decorators/roles.decorator";
import { JwtAuthGuard } from "../auth/guards/jwt-auth.guard";
import { RolesGuard } from "../auth/guards/roles.guard";

@Controller("feature-flags")
@UseGuards(JwtAuthGuard, RolesGuard)
@Roles("admin")
export class FeatureFlagsController {
  constructor(
    private readonly featureFlagService: FeatureFlagService,
    private readonly abTestService: ABTestService
  ) {}

  @Get()
  async getAllFlags() {
    return this.featureFlagService.findAll();
  }

  @Post()
  async createFlag(@Body() dto: any) {
    return this.featureFlagService.createFlag(dto);
  }

  @Put(":key")
  async updateFlag(@Param("key") key: string, @Body() dto: any) {
    await this.featureFlagService.updateFlag(key, dto);
    return { success: true };
  }

  @Delete(":key")
  async deleteFlag(@Param("key") key: string) {
    await this.featureFlagService.deleteFlag(key);
    return { success: true };
  }

  @Post(":key/enable")
  async enableFlag(@Param("key") key: string) {
    await this.featureFlagService.updateFlag(key, { enabled: true });
    return { success: true };
  }

  @Post(":key/disable")
  async disableFlag(@Param("key") key: string) {
    await this.featureFlagService.updateFlag(key, { enabled: false });
    return { success: true };
  }

  @Get("experiments/:key/results")
  async getExperimentResults(@Param("key") key: string) {
    return this.abTestService.analyzeResults(key);
  }

  // Check flag for specific user (for debugging)
  @Get(":key/evaluate")
  async evaluateFlag(
    @Param("key") key: string,
    @Query("userId") userId: string
  ) {
    const enabled = await this.featureFlagService.isEnabled(key, {
      userId,
      environment: process.env.NODE_ENV
    });
    
    return { flagKey: key, userId, enabled };
  }
}
```

---

## ขั้นตอนที่ 1231: Using Feature Flags in Services

```typescript
// src/products/products.service.ts (with feature flags)
import { Injectable } from "@nestjs/common";
import { FeatureFlagService } from "../feature-flags/feature-flag.service";
import { ABTestService } from "../ab-testing/ab-test.service";

@Injectable()
export class ProductsService {
  constructor(
    private readonly flagService: FeatureFlagService,
    private readonly abTestService: ABTestService
  ) {}

  async getProductRecommendations(userId: string) {
    // Check if new recommendation algorithm is enabled
    const useNewAlgorithm = await this.flagService.isEnabled(
      "new-recommendation-algorithm",
      { userId }
    );
    
    if (useNewAlgorithm) {
      return this.getMLRecommendations(userId);
    }
    
    return this.getLegacyRecommendations(userId);
  }

  async getProductLayout(userId: string) {
    // A/B test for product layout
    const variant = await this.abTestService.getVariant(
      "product-layout-test",
      userId
    );
    
    if (!variant) {
      return this.getDefaultLayout();
    }
    
    // Track exposure
    await this.abTestService.trackConversion(
      "product-layout-test",
      userId,
      "page_view"
    );
    
    switch (variant.variantKey) {
      case "grid":
        return this.getGridLayout();
      case "list":
        return this.getListLayout();
      default:
        return this.getDefaultLayout();
    }
  }

  async checkout(userId: string, orderId: string) {
    const useNewCheckout = await this.flagService.isEnabled(
      "new-checkout-flow",
      { userId }
    );
    
    let result;
    if (useNewCheckout) {
      result = await this.newCheckoutFlow(orderId);
      // Track conversion for A/B test
      await this.abTestService.trackConversion(
        "new-checkout-flow",
        userId,
        "checkout_complete",
        result.total
      );
    } else {
      result = await this.legacyCheckoutFlow(orderId);
    }
    
    return result;
  }

  private async getMLRecommendations(userId: string) {
    return [];
  }

  private async getLegacyRecommendations(userId: string) {
    return [];
  }

  private getDefaultLayout() { return { type: "default" }; }
  private getGridLayout() { return { type: "grid" }; }
  private getListLayout() { return { type: "list" }; }
  private async newCheckoutFlow(orderId: string) { return { total: 100 }; }
  private async legacyCheckoutFlow(orderId: string) { return { total: 100 }; }
}
```

---

## ขั้นตอนที่ 1232: Feature Flag Testing

```typescript
// src/feature-flags/__tests__/feature-flag.service.spec.ts
import { Test } from "@nestjs/testing";
import { FeatureFlagService } from "../feature-flag.service";

describe("FeatureFlagService", () => {
  let service: FeatureFlagService;

  beforeEach(async () => {
    // Mock feature flags for testing
    const mockFlags = new Map([
      ["new-feature", {
        isEnabled: (ctx: any) => ctx?.userId === "test-user-1"
      }],
      ["gradual-rollout", {
        isEnabled: (ctx: any) => true // Enable for all in tests
      }]
    ]);
    
    service = new FeatureFlagService(null as any, null as any);
    (service as any).flags = mockFlags;
  });

  it("should return true for whitelisted user", async () => {
    const enabled = await service.isEnabled("new-feature", {
      userId: "test-user-1"
    });
    expect(enabled).toBe(true);
  });

  it("should return false for non-whitelisted user", async () => {
    const enabled = await service.isEnabled("new-feature", {
      userId: "other-user"
    });
    expect(enabled).toBe(false);
  });

  it("should return false for unknown flag", async () => {
    const enabled = await service.isEnabled("unknown-flag");
    expect(enabled).toBe(false);
  });
});

// Helper for tests - override flags
export class TestFeatureFlagService {
  private overrides = new Map<string, boolean>();

  enable(flagKey: string): void {
    this.overrides.set(flagKey, true);
  }

  disable(flagKey: string): void {
    this.overrides.set(flagKey, false);
  }

  async isEnabled(flagKey: string): Promise<boolean> {
    return this.overrides.get(flagKey) ?? false;
  }
}
```

---

## ขั้นตอนที่ 1233: Feature Flags in React (Client-side)

```typescript
// For completeness - client side feature flags
// This would be in a React app

// hooks/useFeatureFlag.ts
import { useState, useEffect } from "react";

interface FlagContext {
  userId: string;
  attributes?: Record<string, any>;
}

async function fetchFlag(flagKey: string, context: FlagContext): Promise<boolean> {
  const response = await fetch(`/api/feature-flags/${flagKey}/evaluate?userId=${context.userId}`);
  const data = await response.json();
  return data.enabled;
}

export function useFeatureFlag(flagKey: string, context: FlagContext) {
  const [enabled, setEnabled] = useState(false);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetchFlag(flagKey, context)
      .then(setEnabled)
      .finally(() => setLoading(false));
  }, [flagKey, context.userId]);

  return { enabled, loading };
}

// Component usage
function ProductPage({ userId }: { userId: string }) {
  const { enabled: newLayout } = useFeatureFlag("new-product-layout", { userId });
  
  return newLayout ? <NewProductLayout /> : <OldProductLayout />;
}

function NewProductLayout() { return null; }
function OldProductLayout() { return null; }
```

---

## ขั้นตอนที่ 1234: Multivariate Testing

```typescript
// src/ab-testing/multivariate.service.ts
// Testing multiple variables simultaneously

interface MultivariateCombination {
  id: string;
  variables: Record<string, string>;
  weight: number;
}

@Injectable()
export class MultivariateTestService {
  async getCombination(
    testKey: string,
    userId: string,
    variables: Record<string, string[]>
  ): Promise<Record<string, string>> {
    // Generate all combinations
    const combinations = this.generateCombinations(variables);
    const equalWeight = 100 / combinations.length;
    
    const weightedCombinations: MultivariateCombination[] = combinations.map(
      (combo, i) => ({
        id: String(i),
        variables: combo,
        weight: equalWeight
      })
    );
    
    // Assign consistent combination to user
    const hash = this.hashString(`${testKey}:${userId}`) % 100;
    let cumulative = 0;
    
    for (const combo of weightedCombinations) {
      cumulative += combo.weight;
      if (hash < cumulative) return combo.variables;
    }
    
    return weightedCombinations[0].variables;
  }

  private generateCombinations(
    variables: Record<string, string[]>
  ): Record<string, string>[] {
    const keys = Object.keys(variables);
    const combinations: Record<string, string>[] = [{}];
    
    for (const key of keys) {
      const newCombinations: Record<string, string>[] = [];
      for (const combo of combinations) {
        for (const value of variables[key]) {
          newCombinations.push({ ...combo, [key]: value });
        }
      }
      combinations.splice(0, combinations.length, ...newCombinations);
    }
    
    return combinations;
  }

  private hashString(str: string): number {
    let hash = 0;
    for (let i = 0; i < str.length; i++) {
      hash = ((hash << 5) - hash) + str.charCodeAt(i);
      hash = hash & hash;
    }
    return Math.abs(hash);
  }
}
```

---

## ขั้นตอนที่ 1235: Kill Switch

```typescript
// src/feature-flags/kill-switch.service.ts
// Emergency feature disable

@Injectable()
export class KillSwitchService {
  constructor(
    private readonly featureFlagService: FeatureFlagService,
    private readonly alertingService: AlertingService
  ) {}

  async activateKillSwitch(
    flagKey: string,
    reason: string,
    activatedBy: string
  ): Promise<void> {
    // Immediately disable feature
    await this.featureFlagService.updateFlag(flagKey, {
      enabled: false,
      rolloutPercentage: 0
    });
    
    // Alert team
    await this.alertingService.sendAlert({
      type: "kill_switch",
      severity: "critical",
      message: `Kill switch activated for ${flagKey}: ${reason}`,
      activatedBy,
      timestamp: new Date()
    });
    
    // Log incident
    console.error(`[KILL SWITCH] ${flagKey} disabled by ${activatedBy}: ${reason}`);
  }
}

class AlertingService {
  async sendAlert(alert: any): Promise<void> {
    console.error("ALERT:", alert);
  }
}
```

---

## ขั้นตอนที่ 1236: Flag Dependencies

```typescript
// Sometimes flags depend on each other
@Injectable()
export class FlagDependencyService {
  constructor(private readonly flagService: FeatureFlagService) {}

  async isEnabled(
    flagKey: string,
    userId: string,
    dependencies: string[] = []
  ): Promise<boolean> {
    // Check all dependencies first
    for (const dep of dependencies) {
      const depEnabled = await this.flagService.isEnabled(dep, { userId });
      if (!depEnabled) return false;
    }
    
    return this.flagService.isEnabled(flagKey, { userId });
  }
}

// Usage
const enabled = await flagDependencyService.isEnabled(
  "advanced-checkout",
  userId,
  ["new-checkout-flow", "payment-v2"] // Must be enabled first
);
```

---

## ขั้นตอนที่ 1237: Feature Flag Audit Log

```typescript
// src/feature-flags/audit.service.ts
@Injectable()
export class FlagAuditService {
  constructor(
    @InjectRepository(FlagAuditEntity)
    private readonly auditRepository: Repository<FlagAuditEntity>
  ) {}

  async log(
    flagKey: string,
    action: "created" | "updated" | "deleted" | "enabled" | "disabled",
    changes: any,
    performedBy: string
  ): Promise<void> {
    await this.auditRepository.save({
      flagKey,
      action,
      changes: JSON.stringify(changes),
      performedBy,
      timestamp: new Date()
    });
  }

  async getHistory(flagKey: string) {
    return this.auditRepository.find({
      where: { flagKey },
      order: { timestamp: "DESC" },
      take: 100
    });
  }
}
```

---

## ขั้นตอนที่ 1238: Integration Testing

```typescript
// src/__tests__/feature-flag-integration.spec.ts
import { Test, TestingModule } from "@nestjs/testing";
import * as request from "supertest";
import { INestApplication } from "@nestjs/common";
import { AppModule } from "../app.module";
import { FeatureFlagService } from "../feature-flags/feature-flag.service";

describe("Feature Flag Integration", () => {
  let app: INestApplication;
  let flagService: FeatureFlagService;

  beforeAll(async () => {
    const module: TestingModule = await Test.createTestingModule({
      imports: [AppModule]
    }).compile();

    app = module.createNestApplication();
    await app.init();
    flagService = module.get(FeatureFlagService);
  });

  it("should serve new layout when flag is enabled", async () => {
    // Enable flag for test user
    await flagService.createFlag({
      key: "test-new-layout",
      enabled: true,
      whitelist: ["test-user-123"]
    });
    
    const response = await request(app.getHttpServer())
      .get("/api/products")
      .set("x-user-id", "test-user-123")
      .expect(200);
    
    expect(response.body.layout).toBe("new");
  });

  it("should serve old layout when flag is disabled", async () => {
    await flagService.updateFlag("test-new-layout", { enabled: false });
    
    const response = await request(app.getHttpServer())
      .get("/api/products")
      .set("x-user-id", "test-user-123")
      .expect(200);
    
    expect(response.body.layout).toBe("default");
  });
});
```

---

## ขั้นตอนที่ 1239: Metrics Tracking

```typescript
// src/ab-testing/metrics.service.ts
@Injectable()
export class ExperimentMetricsService {
  constructor(
    @InjectRepository(ExperimentMetricEntity)
    private readonly metricRepository: Repository<ExperimentMetricEntity>
  ) {}

  async trackMetric(
    experimentKey: string,
    userId: string,
    variantKey: string,
    metric: string,
    value: number
  ): Promise<void> {
    await this.metricRepository.save({
      experimentKey,
      userId,
      variantKey,
      metric,
      value,
      timestamp: new Date()
    });
  }

  async getMetricSummary(
    experimentKey: string,
    metric: string
  ) {
    const results = await this.metricRepository
      .createQueryBuilder("m")
      .select("m.variantKey", "variant")
      .addSelect("COUNT(*)", "count")
      .addSelect("AVG(m.value)", "avg")
      .addSelect("MIN(m.value)", "min")
      .addSelect("MAX(m.value)", "max")
      .where("m.experimentKey = :key", { key: experimentKey })
      .andWhere("m.metric = :metric", { metric })
      .groupBy("m.variantKey")
      .getRawMany();
    
    return results;
  }
}
```

---

## ขั้นตอนที่ 1240: Feature Flag Complete Setup

```typescript
// src/feature-flags/feature-flags.module.ts
import { Module } from "@nestjs/common";
import { TypeOrmModule } from "@nestjs/typeorm";
import { FeatureFlagService } from "./feature-flag.service";
import { FeatureFlagsController } from "./feature-flags.controller";
import { LaunchDarklyService } from "./launchdarkly.service";
import { GradualRolloutService } from "./rollout.service";
import { KillSwitchService } from "./kill-switch.service";
import { FlagAuditService } from "./audit.service";
import { FeatureFlagEntity } from "./entities/feature-flag.entity";
import { FlagAuditEntity } from "./entities/flag-audit.entity";

@Module({
  imports: [TypeOrmModule.forFeature([FeatureFlagEntity, FlagAuditEntity])],
  controllers: [FeatureFlagsController],
  providers: [
    FeatureFlagService,
    LaunchDarklyService,
    GradualRolloutService,
    KillSwitchService,
    FlagAuditService
  ],
  exports: [FeatureFlagService, LaunchDarklyService]
})
export class FeatureFlagsModule {}
```

---

## 🏋️ แบบฝึกหัด

### แบบฝึกหัดที่ 1: Feature Flag System
1. สร้าง Feature Flag CRUD API
2. implement gradual rollout (10% → 50% → 100%)
3. สร้าง kill switch endpoint

### แบบฝึกหัดที่ 2: A/B Testing
1. สร้าง experiment สำหรับ checkout flow
2. assign users to variants consistently
3. analyze results และ statistical significance

### แบบฝึกหัดที่ 3: Integration
1. integrate LaunchDarkly
2. implement client-side flag delivery
3. สร้าง real-time flag updates via WebSocket

---

## 📚 สรุป

ใน Part นี้เราได้เรียนรู้:
- Feature Flag patterns
- Gradual rollout strategies
- A/B Testing framework
- Statistical significance testing
- LaunchDarkly integration
- Kill switch pattern
- Multivariate testing
- Audit logging

**Part ถัดไป**: API Gateway
