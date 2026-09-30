# Part 74 | ขั้นตอนที่ 1281-1300 จาก 1000+

# Chaos Engineering

---

## 🎯 เป้าหมายของ Part นี้

หลังจากเรียนจบ Part นี้ คุณจะสามารถ:
1. เข้าใจ Chaos Engineering principles
2. implement Fault Injection patterns
3. ใช้ Chaos Monkey concepts
4. ทดสอบ resilience ของระบบ
5. สร้าง Game Day สำหรับทีม
6. วิเคราะห์ failure modes

---

## ขั้นตอนที่ 1281: Chaos Engineering คืออะไร?

Chaos Engineering เป็นการทดสอบระบบโดยการจงใจสร้าง failures เพื่อค้นหาจุดอ่อน

```
Chaos Engineering Principles:
┌──────────────────────────────────────────────────────┐
│  1. Define Steady State                               │
│     "System is working normally when..."             │
│                                                      │
│  2. Hypothesize                                      │
│     "If X fails, system should still work"           │
│                                                      │
│  3. Introduce Variables                              │
│     - Server crashes                                 │
│     - Network latency                                │
│     - Database failures                              │
│     - Dependencies unavailable                       │
│                                                      │
│  4. Disprove Hypothesis                              │
│     Run experiment in production (safely)            │
│                                                      │
│  5. Automate & Run Continuously                      │
│     Regular chaos experiments                        │
└──────────────────────────────────────────────────────┘
```

---

## ขั้นตอนที่ 1282: Chaos Monkey Pattern

```typescript
// src/chaos/chaos-monkey.ts
// Chaos Monkey randomly terminates services to test resilience

import { Injectable, Logger } from "@nestjs/common";
import { Cron } from "@nestjs/schedule";
import * as k8s from "@kubernetes/client-node";

@Injectable()
export class ChaosMonkeyService {
  private readonly logger = new Logger(ChaosMonkeyService.name);
  private readonly kc = new k8s.KubeConfig();
  private readonly k8sApi: k8s.CoreV1Api;
  
  private isEnabled = process.env.CHAOS_MONKEY_ENABLED === "true";
  private namespace = process.env.NAMESPACE ?? "default";

  constructor() {
    this.kc.loadFromDefault();
    this.k8sApi = this.kc.makeApiClient(k8s.CoreV1Api);
  }

  // Run every business hour (Mon-Fri 9am-5pm)
  @Cron("0 9-17 * * 1-5")
  async runChaos() {
    if (!this.isEnabled) {
      this.logger.log("Chaos Monkey is disabled");
      return;
    }

    try {
      // Get all pods in namespace
      const pods = await this.k8sApi.listNamespacedPod(this.namespace);
      const activePods = pods.body.items.filter(
        pod => pod.status?.phase === "Running"
      );

      if (activePods.length === 0) {
        this.logger.warn("No running pods found");
        return;
      }

      // Pick a random pod to terminate
      const randomPod = activePods[Math.floor(Math.random() * activePods.length)];
      const podName = randomPod.metadata?.name;
      
      this.logger.warn(`CHAOS: Terminating pod ${podName}`);
      
      await this.k8sApi.deleteNamespacedPod(podName!, this.namespace);
      
      this.logger.log(`Pod ${podName} terminated by Chaos Monkey`);
      
    } catch (error) {
      this.logger.error(`Chaos Monkey error: ${(error as Error).message}`);
    }
  }

  async enableChaos() {
    this.isEnabled = true;
    this.logger.warn("Chaos Monkey ENABLED");
  }

  async disableChaos() {
    this.isEnabled = false;
    this.logger.log("Chaos Monkey disabled");
  }
}
```

---

## ขั้นตอนที่ 1283: Fault Injection Middleware

```typescript
// src/chaos/fault-injection.middleware.ts

import { Injectable, NestMiddleware, Logger } from "@nestjs/common";
import { Request, Response, NextFunction } from "express";

interface FaultConfig {
  enabled: boolean;
  failureRate: number;       // 0-1, percentage of requests to fail
  latencyRate: number;       // 0-1, percentage to add latency
  minLatencyMs: number;
  maxLatencyMs: number;
  errorCodes: number[];
  targetPaths: string[];     // Only inject faults for these paths
}

@Injectable()
export class FaultInjectionMiddleware implements NestMiddleware {
  private readonly logger = new Logger(FaultInjectionMiddleware.name);
  
  private config: FaultConfig = {
    enabled: process.env.FAULT_INJECTION_ENABLED === "true",
    failureRate: parseFloat(process.env.FAULT_INJECTION_FAILURE_RATE ?? "0"),
    latencyRate: parseFloat(process.env.FAULT_INJECTION_LATENCY_RATE ?? "0"),
    minLatencyMs: parseInt(process.env.FAULT_INJECTION_MIN_LATENCY ?? "100"),
    maxLatencyMs: parseInt(process.env.FAULT_INJECTION_MAX_LATENCY ?? "2000"),
    errorCodes: [500, 503, 429],
    targetPaths: ["/api/products", "/api/orders"]
  };

  async use(req: Request, res: Response, next: NextFunction) {
    if (!this.config.enabled) {
      return next();
    }

    // Check if path should be targeted
    const isTargeted = this.config.targetPaths.some(p => req.path.startsWith(p));
    if (!isTargeted) {
      return next();
    }

    // Inject latency
    if (Math.random() < this.config.latencyRate) {
      const latency = Math.floor(
        Math.random() * (this.config.maxLatencyMs - this.config.minLatencyMs) 
        + this.config.minLatencyMs
      );
      
      this.logger.warn(`CHAOS: Injecting ${latency}ms latency for ${req.path}`);
      await new Promise(r => setTimeout(r, latency));
    }

    // Inject failure
    if (Math.random() < this.config.failureRate) {
      const errorCode = this.config.errorCodes[
        Math.floor(Math.random() * this.config.errorCodes.length)
      ];
      
      this.logger.warn(`CHAOS: Injecting ${errorCode} error for ${req.path}`);
      
      return res.status(errorCode).json({
        error: "Chaos fault injected",
        code: errorCode
      });
    }

    next();
  }

  updateConfig(newConfig: Partial<FaultConfig>) {
    this.config = { ...this.config, ...newConfig };
    this.logger.log(`Chaos config updated: ${JSON.stringify(this.config)}`);
  }
}
```

---

## ขั้นตอนที่ 1284: Chaos Engineering Experiments

```typescript
// src/chaos/experiments/database-failure.ts
// Simulate database connection failures

import { Injectable, Logger } from "@nestjs/common";
import { DataSource } from "typeorm";

@Injectable()
export class DatabaseChaosExperiment {
  private readonly logger = new Logger(DatabaseChaosExperiment.name);
  
  constructor(private readonly dataSource: DataSource) {}

  async runConnectionDropExperiment(
    durationMs: number = 5000
  ): Promise<ExperimentResult> {
    const startTime = Date.now();
    const errors: Error[] = [];
    
    this.logger.warn("CHAOS: Starting database connection drop experiment");
    
    try {
      // Destroy the connection pool
      await this.dataSource.destroy();
      
      // Wait for the experiment duration
      await new Promise(r => setTimeout(r, durationMs));
      
      // Observe system behavior (should use cached data / error gracefully)
      
    } catch (error) {
      errors.push(error as Error);
    } finally {
      // Reconnect
      this.logger.log("CHAOS: Restoring database connection");
      await this.dataSource.initialize();
    }
    
    const result: ExperimentResult = {
      name: "Database Connection Drop",
      startTime: new Date(startTime),
      endTime: new Date(),
      durationMs: Date.now() - startTime,
      errors,
      success: errors.length === 0,
      metrics: await this.collectMetrics()
    };
    
    this.logger.log(`Experiment completed: ${JSON.stringify(result.metrics)}`);
    return result;
  }

  async runSlowQueryExperiment() {
    // Inject slow queries using query hijacking
    const originalQuery = this.dataSource.query.bind(this.dataSource);
    
    (this.dataSource as any).query = async (query: string, parameters?: any[]) => {
      // Add random delay to all queries
      const delay = Math.random() * 3000;
      await new Promise(r => setTimeout(r, delay));
      return originalQuery(query, parameters);
    };

    // Run for 30 seconds
    await new Promise(r => setTimeout(r, 30000));
    
    // Restore original
    (this.dataSource as any).query = originalQuery;
  }

  private async collectMetrics() {
    return {
      timestamp: new Date(),
      errorRate: 0, // Would collect from monitoring
      p99Latency: 0,
      successRate: 0
    };
  }
}

interface ExperimentResult {
  name: string;
  startTime: Date;
  endTime: Date;
  durationMs: number;
  errors: Error[];
  success: boolean;
  metrics: any;
}
```

---

## ขั้นตอนที่ 1285: Circuit Breaker Testing

```typescript
// src/chaos/circuit-breaker-test.ts

import axios from "axios";
import { Logger } from "@nestjs/common";

export class CircuitBreakerChaosTest {
  private readonly logger = new Logger(CircuitBreakerChaosTest.name);

  async testCircuitBreakerTrips(serviceUrl: string): Promise<TestReport> {
    const report: TestReport = {
      testName: "Circuit Breaker Trip Test",
      phases: []
    };

    // Phase 1: Steady state (service is healthy)
    this.logger.log("Phase 1: Measuring steady state...");
    const steadyStateMetrics = await this.measureErrorRate(serviceUrl, 100, 0);
    report.phases.push({
      name: "Steady State",
      successRate: steadyStateMetrics.successRate,
      avgLatency: steadyStateMetrics.avgLatency,
      circuitOpen: false
    });

    // Phase 2: Inject failures (trigger circuit breaker)
    this.logger.warn("Phase 2: Injecting failures to trip circuit breaker...");
    // Enable fault injection
    await axios.post(`${serviceUrl}/chaos/fault-injection`, {
      enabled: true,
      failureRate: 0.8  // 80% failure rate
    });
    
    const failureMetrics = await this.measureErrorRate(serviceUrl, 100, 0);
    report.phases.push({
      name: "Failure Injection",
      successRate: failureMetrics.successRate,
      avgLatency: failureMetrics.avgLatency,
      circuitOpen: failureMetrics.circuitOpen
    });

    // Phase 3: Circuit open (requests fail fast)
    this.logger.log("Phase 3: Measuring circuit breaker open state...");
    const circuitOpenMetrics = await this.measureErrorRate(serviceUrl, 20, 500);
    report.phases.push({
      name: "Circuit Open",
      successRate: circuitOpenMetrics.successRate,
      avgLatency: circuitOpenMetrics.avgLatency,
      circuitOpen: true
    });

    // Phase 4: Restore service (circuit should close)
    this.logger.log("Phase 4: Restoring service, waiting for circuit to close...");
    await axios.post(`${serviceUrl}/chaos/fault-injection`, {
      enabled: false
    });
    
    // Wait for circuit breaker reset timeout
    await new Promise(r => setTimeout(r, 30000));
    
    const recoveryMetrics = await this.measureErrorRate(serviceUrl, 100, 0);
    report.phases.push({
      name: "Recovery",
      successRate: recoveryMetrics.successRate,
      avgLatency: recoveryMetrics.avgLatency,
      circuitOpen: false
    });

    report.passed = recoveryMetrics.successRate > 0.95;
    return report;
  }

  private async measureErrorRate(
    url: string, 
    requests: number,
    delayMs: number
  ): Promise<MetricsSnapshot> {
    let successCount = 0;
    let totalLatency = 0;
    let circuitOpen = false;

    for (let i = 0; i < requests; i++) {
      if (delayMs > 0) {
        await new Promise(r => setTimeout(r, delayMs));
      }
      
      const startTime = Date.now();
      try {
        await axios.get(`${url}/health`, { timeout: 5000 });
        successCount++;
      } catch (error: any) {
        if (error.response?.status === 503 && 
            error.response?.data?.reason === "circuit_open") {
          circuitOpen = true;
        }
      }
      totalLatency += Date.now() - startTime;
    }

    return {
      successRate: successCount / requests,
      avgLatency: totalLatency / requests,
      circuitOpen
    };
  }
}

interface TestReport {
  testName: string;
  phases: Phase[];
  passed?: boolean;
}

interface Phase {
  name: string;
  successRate: number;
  avgLatency: number;
  circuitOpen: boolean;
}

interface MetricsSnapshot {
  successRate: number;
  avgLatency: number;
  circuitOpen: boolean;
}
```

---

## ขั้นตอนที่ 1286: Network Chaos

```typescript
// src/chaos/network-chaos.ts
// Simulate network issues

import { exec } from "child_process";
import { promisify } from "util";

const execAsync = promisify(exec);

export class NetworkChaosService {
  
  // Add network latency using tc (Linux traffic control)
  async addLatency(
    interfaceName: string = "eth0",
    latencyMs: number = 200,
    jitterMs: number = 50
  ): Promise<void> {
    const command = `tc qdisc add dev ${interfaceName} root netem delay ${latencyMs}ms ${jitterMs}ms`;
    await execAsync(command);
    console.log(`Added ${latencyMs}ms +/- ${jitterMs}ms latency to ${interfaceName}`);
  }

  // Remove network chaos
  async removeLatency(interfaceName: string = "eth0"): Promise<void> {
    await execAsync(`tc qdisc del dev ${interfaceName} root`);
    console.log(`Removed network chaos from ${interfaceName}`);
  }

  // Simulate packet loss
  async addPacketLoss(
    interfaceName: string = "eth0",
    lossPercent: number = 10
  ): Promise<void> {
    const command = `tc qdisc add dev ${interfaceName} root netem loss ${lossPercent}%`;
    await execAsync(command);
    console.log(`Added ${lossPercent}% packet loss to ${interfaceName}`);
  }

  // Simulate bandwidth throttling
  async throttleBandwidth(
    interfaceName: string = "eth0",
    rateMbit: number = 1   // 1 Mbit/s
  ): Promise<void> {
    const commands = [
      `tc qdisc add dev ${interfaceName} root handle 1: htb default 12`,
      `tc class add dev ${interfaceName} parent 1:1 classid 1:12 htb rate ${rateMbit}mbit ceil ${rateMbit}mbit`
    ];
    
    for (const cmd of commands) {
      await execAsync(cmd);
    }
    
    console.log(`Throttled bandwidth to ${rateMbit}Mbit/s on ${interfaceName}`);
  }
}
```

---

## ขั้นตอนที่ 1287: Steady State Monitoring

```typescript
// src/chaos/steady-state.ts
// Define and monitor system steady state

import * as promClient from "prom-client";

export interface SteadyState {
  name: string;
  description: string;
  check: () => Promise<SteadyStateCheck>;
  tolerance: number;  // Acceptable deviation percentage
}

export interface SteadyStateCheck {
  name: string;
  value: number;
  baselineValue: number;
  withinTolerance: boolean;
  unit: string;
}

export class SteadyStateMonitor {
  private baseline: Map<string, number> = new Map();
  
  private steadyStates: SteadyState[] = [
    {
      name: "error_rate",
      description: "HTTP error rate should be below 1%",
      tolerance: 1,  // 1% tolerance
      check: async () => {
        const errorRate = await this.getMetricValue(
          'rate(http_requests_total{status=~"5.."}[5m]) / rate(http_requests_total[5m])'
        );
        const baseline = this.baseline.get("error_rate") ?? 0;
        return {
          name: "error_rate",
          value: errorRate * 100,
          baselineValue: baseline,
          withinTolerance: errorRate < 0.01,
          unit: "%"
        };
      }
    },
    {
      name: "p99_latency",
      description: "99th percentile latency should be under 500ms",
      tolerance: 50,  // 50ms tolerance
      check: async () => {
        const p99 = await this.getMetricValue(
          "histogram_quantile(0.99, rate(http_request_duration_ms_bucket[5m]))"
        );
        const baseline = this.baseline.get("p99_latency") ?? 200;
        return {
          name: "p99_latency",
          value: p99,
          baselineValue: baseline,
          withinTolerance: p99 < 500,
          unit: "ms"
        };
      }
    },
    {
      name: "success_rate",
      description: "Order success rate should be above 99%",
      tolerance: 1,
      check: async () => {
        const successRate = await this.getMetricValue(
          'rate(orders_total{status="success"}[5m]) / rate(orders_total[5m])'
        );
        const baseline = this.baseline.get("success_rate") ?? 0.99;
        return {
          name: "success_rate",
          value: successRate * 100,
          baselineValue: baseline * 100,
          withinTolerance: successRate > 0.99,
          unit: "%"
        };
      }
    }
  ];

  async recordBaseline(): Promise<void> {
    console.log("Recording baseline steady state...");
    
    for (const state of this.steadyStates) {
      const check = await state.check();
      this.baseline.set(state.name, check.value);
      console.log(`Baseline ${state.name}: ${check.value}${check.unit}`);
    }
  }

  async checkSteadyState(): Promise<SteadyStateReport> {
    const results = await Promise.all(
      this.steadyStates.map(state => state.check())
    );

    const allOk = results.every(r => r.withinTolerance);

    return {
      timestamp: new Date(),
      isInSteadyState: allOk,
      checks: results
    };
  }

  private async getMetricValue(query: string): Promise<number> {
    // Would query Prometheus here
    // For demo, return mock values
    return Math.random() * 100;
  }
}

interface SteadyStateReport {
  timestamp: Date;
  isInSteadyState: boolean;
  checks: SteadyStateCheck[];
}
```

---

## ขั้นตอนที่ 1288: Chaos Experiment Runner

```typescript
// src/chaos/experiment-runner.ts

import { Logger } from "@nestjs/common";
import { SteadyStateMonitor } from "./steady-state";

export interface ChaosExperiment {
  name: string;
  description: string;
  hypothesis: string;
  setup?: () => Promise<void>;
  inject: () => Promise<void>;
  rollback: () => Promise<void>;
  duration: number;  // ms
  abortOnSteadyStateViolation?: boolean;
}

export interface ExperimentReport {
  experiment: string;
  hypothesis: string;
  startTime: Date;
  endTime: Date;
  steadyStateBeforeExperiment: boolean;
  steadyStateDuringExperiment: boolean;
  steadyStateAfterExperiment: boolean;
  hypothesisProven: boolean;
  observations: string[];
  conclusion: string;
}

export class ChaosExperimentRunner {
  private readonly logger = new Logger(ChaosExperimentRunner.name);
  private readonly steadyState = new SteadyStateMonitor();

  async runExperiment(experiment: ChaosExperiment): Promise<ExperimentReport> {
    const observations: string[] = [];
    const startTime = new Date();

    this.logger.warn(`Starting chaos experiment: ${experiment.name}`);
    this.logger.log(`Hypothesis: ${experiment.hypothesis}`);

    // Setup phase
    if (experiment.setup) {
      await experiment.setup();
    }

    // Verify steady state before experiment
    await this.steadyState.recordBaseline();
    const beforeState = await this.steadyState.checkSteadyState();
    
    if (!beforeState.isInSteadyState) {
      this.logger.error("System is NOT in steady state before experiment - aborting!");
      return {
        experiment: experiment.name,
        hypothesis: experiment.hypothesis,
        startTime,
        endTime: new Date(),
        steadyStateBeforeExperiment: false,
        steadyStateDuringExperiment: false,
        steadyStateAfterExperiment: false,
        hypothesisProven: false,
        observations: ["Aborted: system not in steady state before experiment"],
        conclusion: "Experiment aborted - system not healthy"
      };
    }
    
    observations.push(`Before: System in steady state ✓`);

    // Inject chaos
    this.logger.warn("Injecting chaos...");
    await experiment.inject();

    // Monitor during chaos
    let duringStateOk = true;
    const monitoringInterval = setInterval(async () => {
      const state = await this.steadyState.checkSteadyState();
      if (!state.isInSteadyState) {
        duringStateOk = false;
        observations.push(
          `During: Steady state violation detected at ${new Date().toISOString()}`
        );
        
        if (experiment.abortOnSteadyStateViolation) {
          this.logger.warn("Steady state violation - rolling back!");
          clearInterval(monitoringInterval);
          await experiment.rollback();
        }
      }
    }, 5000);

    // Wait for experiment duration
    await new Promise(r => setTimeout(r, experiment.duration));
    clearInterval(monitoringInterval);

    // Rollback chaos
    this.logger.log("Rolling back chaos...");
    await experiment.rollback();

    // Verify steady state after experiment
    await new Promise(r => setTimeout(r, 10000));  // Wait for recovery
    const afterState = await this.steadyState.checkSteadyState();
    
    const hypothesisProven = duringStateOk && afterState.isInSteadyState;
    
    observations.push(
      afterState.isInSteadyState 
        ? "After: System returned to steady state ✓" 
        : "After: System did NOT return to steady state ✗"
    );

    const report: ExperimentReport = {
      experiment: experiment.name,
      hypothesis: experiment.hypothesis,
      startTime,
      endTime: new Date(),
      steadyStateBeforeExperiment: beforeState.isInSteadyState,
      steadyStateDuringExperiment: duringStateOk,
      steadyStateAfterExperiment: afterState.isInSteadyState,
      hypothesisProven,
      observations,
      conclusion: hypothesisProven
        ? "Hypothesis proven: system is resilient to this failure"
        : "Hypothesis disproven: system has weakness that needs to be addressed"
    };

    this.logger.log(`Experiment completed: ${report.conclusion}`);
    return report;
  }
}
```

---

## ขั้นตอนที่ 1289: Predefined Experiments

```typescript
// src/chaos/experiments/predefined.ts

import { ChaosExperiment } from "../experiment-runner";
import axios from "axios";

const FAULT_INJECTION_URL = process.env.FAULT_INJECTION_URL ?? "http://localhost:3000";

export const experiments: ChaosExperiment[] = [
  {
    name: "Database Connection Loss",
    description: "Test system resilience when database connection is lost",
    hypothesis: "When database is unavailable, API returns 503 with cached data",
    duration: 30000,  // 30 seconds
    inject: async () => {
      await axios.post(`${FAULT_INJECTION_URL}/chaos/database`, { enabled: true });
    },
    rollback: async () => {
      await axios.post(`${FAULT_INJECTION_URL}/chaos/database`, { enabled: false });
    }
  },
  
  {
    name: "High Memory Pressure",
    description: "Simulate memory pressure to test OOM behavior",
    hypothesis: "Under memory pressure, system degrades gracefully",
    duration: 60000,
    inject: async () => {
      // Allocate large chunks of memory
      (global as any).__chaosMemory = Buffer.alloc(500 * 1024 * 1024); // 500MB
    },
    rollback: async () => {
      delete (global as any).__chaosMemory;
      if (global.gc) global.gc();  // Force GC if available
    }
  },
  
  {
    name: "High CPU Load",
    description: "Simulate CPU spike",
    hypothesis: "Under high CPU load, response times increase but requests don't fail",
    duration: 30000,
    inject: async () => {
      // Start CPU-intensive background work
      (global as any).__chaosWorkers = [];
      for (let i = 0; i < 4; i++) {
        const worker = setInterval(() => {
          let x = 0;
          for (let j = 0; j < 1000000; j++) {
            x = Math.sqrt(j) * Math.sqrt(j);
          }
          return x;
        }, 0);
        (global as any).__chaosWorkers.push(worker);
      }
    },
    rollback: async () => {
      const workers = (global as any).__chaosWorkers || [];
      workers.forEach((w: NodeJS.Timeout) => clearInterval(w));
      delete (global as any).__chaosWorkers;
    }
  },
  
  {
    name: "Third Party API Timeout",
    description: "Payment service timeout",
    hypothesis: "When payment API times out, order creation returns proper error",
    duration: 30000,
    inject: async () => {
      await axios.post(`${FAULT_INJECTION_URL}/chaos/fault-injection`, {
        enabled: true,
        targetPaths: ["/api/payments"],
        latencyRate: 1.0,
        minLatencyMs: 30000,  // 30s delay (longer than timeout)
        maxLatencyMs: 35000
      });
    },
    rollback: async () => {
      await axios.post(`${FAULT_INJECTION_URL}/chaos/fault-injection`, {
        enabled: false
      });
    }
  }
];
```

---

## ขั้นตอนที่ 1290: Chaos API Controller

```typescript
// src/chaos/chaos.controller.ts

import { Controller, Post, Get, Body, Logger } from "@nestjs/common";
import { FaultInjectionMiddleware } from "./fault-injection.middleware";
import { ChaosExperimentRunner } from "./experiment-runner";
import { experiments } from "./experiments/predefined";

@Controller("chaos")
export class ChaosController {
  private readonly logger = new Logger(ChaosController.name);
  
  constructor(
    private readonly faultInjection: FaultInjectionMiddleware,
    private readonly experimentRunner: ChaosExperimentRunner
  ) {}

  @Post("fault-injection")
  async configureFaultInjection(@Body() config: any) {
    if (process.env.NODE_ENV === "production") {
      // Only allow in specific environments
      this.logger.warn("Fault injection attempted in production - BLOCKED");
      return { error: "Not allowed in production" };
    }
    
    this.faultInjection.updateConfig(config);
    return { message: "Fault injection configured", config };
  }

  @Get("experiments")
  listExperiments() {
    return experiments.map(e => ({
      name: e.name,
      description: e.description,
      hypothesis: e.hypothesis,
      duration: e.duration
    }));
  }

  @Post("experiments/run")
  async runExperiment(@Body() body: { experimentName: string }) {
    const experiment = experiments.find(e => e.name === body.experimentName);
    
    if (!experiment) {
      return { error: `Experiment '${body.experimentName}' not found` };
    }

    this.logger.warn(`Running chaos experiment: ${body.experimentName}`);
    
    const report = await this.experimentRunner.runExperiment(experiment);
    return report;
  }
}
```

---

## ขั้นตอนที่ 1291: Game Day Runbook

```typescript
// src/chaos/game-day.ts
// Game Day - structured chaos engineering session for team

export interface GameDayScenario {
  id: string;
  title: string;
  description: string;
  prerequisites: string[];
  steps: string[];
  expectedOutcome: string;
  rollbackPlan: string;
  successCriteria: string[];
  timeBoxMinutes: number;
}

export const gameDayRunbook: GameDayScenario[] = [
  {
    id: "gd-001",
    title: "Database Failover",
    description: "Primary database fails over to replica",
    prerequisites: [
      "Database replica is running",
      "Monitoring dashboards are visible",
      "All team members are present",
      "On-call engineer is identified"
    ],
    steps: [
      "1. Verify steady state - all green dashboards",
      "2. Check current DB primary: kubectl get pods -n database",
      "3. Announce experiment start in Slack #incidents channel",
      "4. Terminate primary DB pod: kubectl delete pod db-primary-0 -n database",
      "5. Monitor: watch for automatic failover in 30-60 seconds",
      "6. Check error rates in Grafana dashboard",
      "7. Verify application continues to serve requests",
      "8. Document observations in real-time"
    ],
    expectedOutcome: "System detects failure and routes to replica within 30s",
    rollbackPlan: "kubectl apply -f k8s/database/primary-restore.yaml",
    successCriteria: [
      "Failover occurs within 30 seconds",
      "Error rate stays below 5%",
      "No data loss occurs",
      "System returns to full operation within 2 minutes"
    ],
    timeBoxMinutes: 30
  },
  
  {
    id: "gd-002",
    title: "Payment Service Unavailable",
    description: "Payment service becomes completely unavailable",
    prerequisites: [
      "Cart and checkout services are running",
      "Fallback behavior is implemented",
      "Error tracking is visible"
    ],
    steps: [
      "1. Verify steady state",
      "2. Scale payment service to 0: kubectl scale deployment payment-service --replicas=0",
      "3. Attempt to complete a purchase",
      "4. Verify error handling - should see user-friendly error message",
      "5. Check if cart is preserved",
      "6. Restore payment service: kubectl scale deployment payment-service --replicas=3"
    ],
    expectedOutcome: "User sees friendly error, cart is preserved, no data lost",
    rollbackPlan: "kubectl scale deployment payment-service --replicas=3",
    successCriteria: [
      "User sees 'Payment service unavailable' not 500 error",
      "Cart contents preserved",
      "Orders in PENDING state, not ERROR",
      "Automatic retry succeeds when service returns"
    ],
    timeBoxMinutes: 20
  }
];
```

---

## ขั้นตอนที่ 1292: Resilience Testing Framework

```typescript
// src/chaos/resilience-tests.ts
// Automated resilience tests

import axios from "axios";

export class ResilienceTestSuite {
  
  async runAll(baseUrl: string): Promise<TestSuiteReport> {
    const tests = [
      this.testRetryBehavior(baseUrl),
      this.testCircuitBreaker(baseUrl),
      this.testTimeoutHandling(baseUrl),
      this.testDegradedMode(baseUrl),
      this.testCacheFailover(baseUrl)
    ];

    const results = await Promise.allSettled(tests);

    return {
      total: tests.length,
      passed: results.filter(r => r.status === "fulfilled" && (r.value as any).passed).length,
      failed: results.filter(r => r.status === "rejected" || !(r as any).value?.passed).length,
      results: results.map((r, i) => ({
        test: ["Retry", "Circuit Breaker", "Timeout", "Degraded Mode", "Cache Failover"][i],
        passed: r.status === "fulfilled" && (r.value as any).passed,
        details: r.status === "fulfilled" ? r.value : (r as PromiseRejectedResult).reason
      }))
    };
  }

  async testRetryBehavior(baseUrl: string): Promise<{ passed: boolean }> {
    // Enable 50% failure rate
    await axios.post(`${baseUrl}/chaos/fault-injection`, {
      enabled: true,
      failureRate: 0.5
    });

    let successCount = 0;
    for (let i = 0; i < 20; i++) {
      try {
        await axios.get(`${baseUrl}/api/products`, { timeout: 10000 });
        successCount++;
      } catch {}
    }

    // Disable fault injection
    await axios.post(`${baseUrl}/chaos/fault-injection`, { enabled: false });

    // With retries, should succeed more than 80% despite 50% failure rate
    const successRate = successCount / 20;
    return { passed: successRate > 0.8 };
  }

  async testCircuitBreaker(baseUrl: string): Promise<{ passed: boolean }> {
    // Enable 100% failure to trigger circuit breaker
    await axios.post(`${baseUrl}/chaos/fault-injection`, {
      enabled: true,
      failureRate: 1.0
    });

    // Make enough requests to trip circuit breaker
    const latencies: number[] = [];
    for (let i = 0; i < 30; i++) {
      const start = Date.now();
      try {
        await axios.get(`${baseUrl}/api/products`, { timeout: 10000 });
      } catch {}
      latencies.push(Date.now() - start);
    }

    // After circuit opens, failures should be fast (< 100ms)
    const lastTenAvgLatency = latencies.slice(-10).reduce((a, b) => a + b, 0) / 10;
    
    await axios.post(`${baseUrl}/chaos/fault-injection`, { enabled: false });
    
    // Circuit breaker should fail fast (not wait for timeout)
    return { passed: lastTenAvgLatency < 100 };
  }

  async testTimeoutHandling(baseUrl: string): Promise<{ passed: boolean }> {
    // Inject 10s latency
    await axios.post(`${baseUrl}/chaos/fault-injection`, {
      enabled: true,
      latencyRate: 1.0,
      minLatencyMs: 10000,
      maxLatencyMs: 10000
    });

    const start = Date.now();
    let timedOut = false;
    
    try {
      await axios.get(`${baseUrl}/api/products`, { timeout: 5000 });
    } catch (error: any) {
      timedOut = error.code === "ECONNABORTED" || error.message.includes("timeout");
    }

    const elapsed = Date.now() - start;
    
    await axios.post(`${baseUrl}/chaos/fault-injection`, { enabled: false });
    
    // Should timeout in roughly 5 seconds (not 10)
    return { passed: timedOut && elapsed < 6000 };
  }

  async testDegradedMode(baseUrl: string): Promise<{ passed: boolean }> {
    // Simulate non-critical service failure
    await axios.post(`${baseUrl}/chaos/database/recommendations`, { enabled: true });
    
    try {
      const response = await axios.get(`${baseUrl}/api/products`);
      // Products should still return, just without recommendations
      const hasProducts = response.data.products?.length > 0;
      const hasWarning = response.data.degraded === true;
      
      return { passed: hasProducts }; // Core feature works
    } finally {
      await axios.post(`${baseUrl}/chaos/database/recommendations`, { enabled: false });
    }
  }

  async testCacheFailover(baseUrl: string): Promise<{ passed: boolean }> {
    // First, warm up cache
    await axios.get(`${baseUrl}/api/products`);
    
    // Kill Redis
    await axios.post(`${baseUrl}/chaos/redis`, { enabled: true });
    
    try {
      const response = await axios.get(`${baseUrl}/api/products`);
      return { passed: response.status === 200 };  // Should work without cache
    } finally {
      await axios.post(`${baseUrl}/chaos/redis`, { enabled: false });
    }
  }
}

interface TestSuiteReport {
  total: number;
  passed: number;
  failed: number;
  results: Array<{
    test: string;
    passed: boolean;
    details: any;
  }>;
}
```

---

## ขั้นตอนที่ 1293: Chaos Engineering in CI/CD

```yaml
# .github/workflows/chaos-tests.yml

name: Chaos Engineering Tests

on:
  schedule:
    - cron: "0 2 * * 1"  # Every Monday at 2am
  workflow_dispatch:
    inputs:
      environment:
        description: "Environment to run chaos tests"
        required: true
        default: "staging"
        type: choice
        options: [staging, production-lite]

jobs:
  chaos-tests:
    runs-on: ubuntu-latest
    environment: ${{ github.event.inputs.environment || 'staging' }}
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Configure kubectl
        uses: azure/k8s-set-context@v3
        with:
          kubeconfig: ${{ secrets.KUBECONFIG }}
      
      - name: Check system health before chaos
        run: |
          ./scripts/check-steady-state.sh
          if [ $? -ne 0 ]; then
            echo "System not in steady state - aborting chaos tests"
            exit 1
          fi
      
      - name: Run chaos experiments
        run: |
          node scripts/run-chaos-experiments.js \
            --env ${{ github.event.inputs.environment || 'staging' }} \
            --duration 300 \
            --output chaos-report.json
      
      - name: Upload chaos report
        uses: actions/upload-artifact@v3
        with:
          name: chaos-report
          path: chaos-report.json
      
      - name: Comment on experiment results
        if: always()
        run: |
          node scripts/post-chaos-summary.js chaos-report.json
      
      - name: Restore system
        if: always()
        run: |
          ./scripts/chaos-rollback.sh
          ./scripts/check-steady-state.sh
```

---

## ขั้นตอนที่ 1294: Bulkhead Pattern

```typescript
// src/resilience/bulkhead.ts
// Isolate failures to prevent cascading

export class Bulkhead {
  private currentCount = 0;
  private readonly queue: Array<() => void> = [];

  constructor(
    private readonly name: string,
    private readonly maxConcurrent: number,
    private readonly maxQueueSize: number = 10
  ) {}

  async execute<T>(fn: () => Promise<T>): Promise<T> {
    // Check if we can execute immediately
    if (this.currentCount < this.maxConcurrent) {
      return this.run(fn);
    }

    // Check if we can queue
    if (this.queue.length >= this.maxQueueSize) {
      throw new Error(`Bulkhead '${this.name}' queue full (max: ${this.maxQueueSize})`);
    }

    // Queue the request
    return new Promise<T>((resolve, reject) => {
      this.queue.push(() => {
        this.run(fn).then(resolve).catch(reject);
      });
    });
  }

  private async run<T>(fn: () => Promise<T>): Promise<T> {
    this.currentCount++;
    
    try {
      return await fn();
    } finally {
      this.currentCount--;
      
      // Process next queued request
      const next = this.queue.shift();
      if (next) next();
    }
  }

  getStats() {
    return {
      name: this.name,
      concurrent: this.currentCount,
      queued: this.queue.length,
      available: this.maxConcurrent - this.currentCount
    };
  }
}

// Usage with NestJS services
const paymentBulkhead = new Bulkhead("payment-service", 10, 20);
const notificationBulkhead = new Bulkhead("notifications", 50, 100);

async function processOrder(orderId: string) {
  // Payment: critical, limited concurrency
  const payment = await paymentBulkhead.execute(async () => {
    return await processPayment(orderId);
  });

  // Notification: non-critical, higher concurrency allowed
  notificationBulkhead.execute(async () => {
    await sendNotification(orderId);
  }).catch(e => console.log("Notification failed (non-critical):", e));

  return payment;
}

async function processPayment(orderId: string) {}
async function sendNotification(orderId: string) {}
```

---

## ขั้นตอนที่ 1295: Chaos Engineering Metrics

```typescript
// src/chaos/metrics.ts

import * as promClient from "prom-client";

// Track chaos experiment results
const chaosExperimentsTotal = new promClient.Counter({
  name: "chaos_experiments_total",
  help: "Total chaos experiments run",
  labelNames: ["experiment_name", "result"]  // result: passed/failed
});

const chaosExperimentDuration = new promClient.Histogram({
  name: "chaos_experiment_duration_seconds",
  help: "Duration of chaos experiments",
  labelNames: ["experiment_name"],
  buckets: [30, 60, 120, 300, 600]
});

const systemResilienceScore = new promClient.Gauge({
  name: "system_resilience_score",
  help: "Overall system resilience score (0-100)"
});

export class ChaosMetricsCollector {
  
  recordExperimentResult(
    name: string, 
    passed: boolean, 
    durationSeconds: number
  ) {
    chaosExperimentsTotal.inc({
      experiment_name: name,
      result: passed ? "passed" : "failed"
    });
    
    chaosExperimentDuration.observe({ experiment_name: name }, durationSeconds);
  }

  updateResilienceScore(score: number) {
    systemResilienceScore.set(score);
  }

  async calculateResilienceScore(
    experimentResults: Array<{ passed: boolean }>
  ): Promise<number> {
    const totalExperiments = experimentResults.length;
    if (totalExperiments === 0) return 0;
    
    const passedExperiments = experimentResults.filter(r => r.passed).length;
    const score = (passedExperiments / totalExperiments) * 100;
    
    this.updateResilienceScore(score);
    return score;
  }
}
```

---

## ขั้นตอนที่ 1296: Chaos Engineering Tools

```bash
# Tools สำหรับ Chaos Engineering

# 1. Chaos Toolkit (open source)
pip install chaostoolkit
pip install chaostoolkit-kubernetes
pip install chaostoolkit-prometheus

# Create experiment file
cat > experiment.json << 'EOF'
{
  "title": "Database pod termination experiment",
  "description": "Terminate database pod and check recovery",
  "steady-states": {
    "before": {
      "title": "System is healthy",
      "probes": [
        {
          "type": "probe",
          "name": "healthy-nodes",
          "tolerance": true,
          "provider": {
            "type": "http",
            "url": "http://api.example.com/health"
          }
        }
      ]
    }
  },
  "method": [
    {
      "type": "action",
      "name": "terminate-db-pod",
      "provider": {
        "type": "python",
        "module": "chaosk8s.pod.actions",
        "func": "terminate_pods",
        "arguments": {
          "label_selector": "app=database",
          "ns": "default"
        }
      },
      "pauses": {
        "after": 30
      }
    }
  ]
}
EOF

# Run experiment
chaos run experiment.json

# 2. Litmus Chaos (Kubernetes-native)
kubectl apply -f https://litmuschaos.github.io/litmus/litmus-operator-v2.x.yaml

# 3. Chaos Mesh
kubectl apply -f https://mirrors.chaos-mesh.org/v2.x.x/crd.yaml
```

---

## ขั้นตอนที่ 1297: Monitoring During Chaos

```typescript
// src/chaos/chaos-monitor.ts
// Real-time monitoring during chaos experiments

import { EventEmitter } from "events";
import axios from "axios";

export class ChaosMonitor extends EventEmitter {
  private monitoring = false;
  private interval: NodeJS.Timeout | null = null;
  private metrics: MetricSnapshot[] = [];

  async startMonitoring(targetUrl: string, intervalMs: number = 1000) {
    this.monitoring = true;
    this.metrics = [];

    this.interval = setInterval(async () => {
      if (!this.monitoring) return;

      const snapshot = await this.collectSnapshot(targetUrl);
      this.metrics.push(snapshot);

      // Emit events for real-time alerting
      if (snapshot.errorRate > 0.05) {
        this.emit("high-error-rate", snapshot);
      }
      if (snapshot.avgLatencyMs > 1000) {
        this.emit("high-latency", snapshot);
      }
    }, intervalMs);
  }

  stopMonitoring() {
    this.monitoring = false;
    if (this.interval) {
      clearInterval(this.interval);
      this.interval = null;
    }
  }

  getReport(): MonitorReport {
    const errorRates = this.metrics.map(m => m.errorRate);
    const latencies = this.metrics.map(m => m.avgLatencyMs);

    return {
      totalSnapshots: this.metrics.length,
      avgErrorRate: errorRates.reduce((a, b) => a + b, 0) / errorRates.length,
      maxErrorRate: Math.max(...errorRates),
      avgLatencyMs: latencies.reduce((a, b) => a + b, 0) / latencies.length,
      maxLatencyMs: Math.max(...latencies),
      timeline: this.metrics
    };
  }

  private async collectSnapshot(url: string): Promise<MetricSnapshot> {
    const start = Date.now();
    let success = false;

    try {
      await axios.get(`${url}/health`, { timeout: 5000 });
      success = true;
    } catch {}

    return {
      timestamp: new Date(),
      success,
      errorRate: success ? 0 : 1,
      avgLatencyMs: Date.now() - start
    };
  }
}

interface MetricSnapshot {
  timestamp: Date;
  success: boolean;
  errorRate: number;
  avgLatencyMs: number;
}

interface MonitorReport {
  totalSnapshots: number;
  avgErrorRate: number;
  maxErrorRate: number;
  avgLatencyMs: number;
  maxLatencyMs: number;
  timeline: MetricSnapshot[];
}
```

---

## ขั้นตอนที่ 1298: Chaos in Production Safely

```typescript
// src/chaos/safety-guards.ts
// Safety guards before running chaos in production

export class ChaosSafetyGuards {
  
  async verifyCanRunInProduction(): Promise<{ allowed: boolean; reasons: string[] }> {
    const issues: string[] = [];

    // Check 1: Business hours guard
    const hour = new Date().getHours();
    const day = new Date().getDay();
    const isBusinessHours = day >= 1 && day <= 5 && hour >= 9 && hour < 17;
    
    if (!isBusinessHours) {
      issues.push("Outside business hours - chaos should run 9am-5pm Mon-Fri");
    }

    // Check 2: Recent deployments
    const lastDeploymentTime = await this.getLastDeploymentTime();
    const minutesSinceDeployment = (Date.now() - lastDeploymentTime.getTime()) / 60000;
    
    if (minutesSinceDeployment < 30) {
      issues.push(`Recent deployment ${Math.round(minutesSinceDeployment)}m ago - wait 30m before chaos`);
    }

    // Check 3: Active incidents
    const hasActiveIncidents = await this.checkForActiveIncidents();
    if (hasActiveIncidents) {
      issues.push("Active incidents detected - resolve before running chaos");
    }

    // Check 4: Team availability
    const oncallEngineerAvailable = await this.verifyOncallAvailability();
    if (!oncallEngineerAvailable) {
      issues.push("No on-call engineer available for chaos experiments");
    }

    // Check 5: Error rate baseline
    const currentErrorRate = await this.getCurrentErrorRate();
    if (currentErrorRate > 0.01) {
      issues.push(`Current error rate ${(currentErrorRate * 100).toFixed(2)}% is too high (>1%)`);
    }

    return {
      allowed: issues.length === 0,
      reasons: issues
    };
  }

  private async getLastDeploymentTime(): Promise<Date> {
    return new Date(Date.now() - 2 * 60 * 60 * 1000);  // 2 hours ago
  }

  private async checkForActiveIncidents(): Promise<boolean> {
    return false;
  }

  private async verifyOncallAvailability(): Promise<boolean> {
    return true;
  }

  private async getCurrentErrorRate(): Promise<number> {
    return 0.001;
  }
}
```

---

## ขั้นตอนที่ 1299: Learning from Chaos

```typescript
// src/chaos/postmortem.ts
// Post-experiment analysis and learning

export interface PostmortemReport {
  experimentId: string;
  experimentName: string;
  date: Date;
  timeline: TimelineEvent[];
  rootCause: string;
  impact: ImpactAssessment;
  lessonsLearned: string[];
  actionItems: ActionItem[];
  followUpDate: Date;
}

export interface TimelineEvent {
  timestamp: Date;
  event: string;
  severity: "info" | "warning" | "critical";
}

export interface ImpactAssessment {
  usersAffected: number;
  requestsAffected: number;
  dataLoss: boolean;
  durationMinutes: number;
}

export interface ActionItem {
  description: string;
  owner: string;
  dueDate: Date;
  priority: "high" | "medium" | "low";
  status: "open" | "in-progress" | "done";
}

export function generatePostmortem(
  experimentData: any,
  teamObservations: string[]
): PostmortemReport {
  return {
    experimentId: experimentData.id,
    experimentName: experimentData.name,
    date: new Date(),
    timeline: experimentData.timeline ?? [],
    rootCause: "To be determined by team analysis",
    impact: {
      usersAffected: 0,
      requestsAffected: experimentData.affectedRequests ?? 0,
      dataLoss: false,
      durationMinutes: experimentData.durationMs / 60000
    },
    lessonsLearned: teamObservations,
    actionItems: [],
    followUpDate: new Date(Date.now() + 7 * 24 * 60 * 60 * 1000)  // 1 week
  };
}
```

---

## ขั้นตอนที่ 1300: Chaos Engineering Checklist

```markdown
# Chaos Engineering Checklist

## Before Starting
- [ ] System is in steady state
- [ ] Monitoring dashboards are visible
- [ ] On-call engineer is present
- [ ] No active incidents
- [ ] No recent deployments (< 30 min)
- [ ] Team has been notified
- [ ] Rollback plan is ready
- [ ] Experiment has been approved

## During Experiment
- [ ] Monitor error rates
- [ ] Monitor latency
- [ ] Watch for cascading failures
- [ ] Document observations in real-time
- [ ] Be ready to rollback at any moment

## After Experiment
- [ ] Rollback any injected faults
- [ ] Verify system returns to steady state
- [ ] Write postmortem
- [ ] Create action items for weaknesses found
- [ ] Share learnings with team
- [ ] Update runbooks if needed

## Weekly/Monthly
- [ ] Review resilience score trend
- [ ] Run new experiment scenarios
- [ ] Verify previous action items resolved
- [ ] Update Game Day runbooks
```

---

## 🏋️ แบบฝึกหัด

### แบบฝึกหัดที่ 1: Fault Injection
1. implement FaultInjectionMiddleware
2. ทดสอบ retry behavior ด้วย 50% failure rate
3. ตรวจสอบว่า circuit breaker ทำงาน

### แบบฝึกหัดที่ 2: Steady State Definition
1. กำหนด steady state สำหรับ API ของคุณ
2. implement monitoring
3. สร้าง alerting เมื่อ steady state ถูกละเมิด

### แบบฝึกหัดที่ 3: Game Day
1. ออกแบบ game day scenario
2. ทดสอบกับทีม
3. เขียน postmortem

---

## 📚 สรุป

ใน Part นี้เราได้เรียนรู้:
- Chaos Engineering principles
- Fault injection patterns
- Circuit breaker testing
- Network chaos simulation
- Steady state monitoring
- Experiment runner framework
- Safety guards สำหรับ production
- Post-mortem analysis

**Part ถัดไป**: Observability
