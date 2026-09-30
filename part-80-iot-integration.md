# Part 80 | ขั้นตอนที่ 1401-1420 จาก 1000+

# IoT Integration with Node.js

---

## 🎯 เป้าหมายของ Part นี้

หลังจากเรียนจบ Part นี้ คุณจะสามารถ:
1. เข้าใจ IoT protocols (MQTT, CoAP)
2. สร้าง MQTT broker และ clients
3. process sensor data real-time
4. implement device management
5. สร้าง IoT dashboard
6. implement edge computing patterns

---

## ขั้นตอนที่ 1401: IoT Architecture

```
IoT Architecture:

┌──────────────┐   MQTT/CoAP   ┌─────────────┐   WebSocket   ┌──────────┐
│  IoT Devices │──────────────►│  IoT Gateway│──────────────►│Dashboard │
│              │               │  (Node.js)  │               │          │
│ - Sensors    │               │             │               │ - Charts │
│ - Actuators  │               │ - Filter    │               │ - Alerts │
│ - Cameras    │               │ - Validate  │               │ - Control│
└──────────────┘               │ - Route     │               └──────────┘
                                │ - Store     │
                                └──────┬──────┘
                                       │
                    ┌──────────────────┼──────────────────┐
                    ▼                  ▼                   ▼
              ┌──────────┐     ┌──────────────┐    ┌──────────┐
              │ TimescaleDB│   │ Message Queue│    │  Redis   │
              │ (Time Series│  │   (Kafka)    │    │  Cache   │
              └──────────┘     └──────────────┘    └──────────┘
```

---

## ขั้นตอนที่ 1402: MQTT Protocol

```
MQTT (Message Queuing Telemetry Transport):

Topics:
  /devices/{deviceId}/telemetry    - Sensor data
  /devices/{deviceId}/status       - Device status
  /devices/{deviceId}/commands     - Commands to device
  /alerts/{severity}/{deviceId}    - Alerts

QoS Levels:
  QoS 0: Fire and forget (best effort)
  QoS 1: At least once delivery
  QoS 2: Exactly once delivery

Retained Messages:
  - Last known device state
  - Available for new subscribers
```

```bash
# Install MQTT packages
npm install mqtt aedes  # aedes = MQTT broker
npm install @nestjs/websockets @nestjs/platform-socket.io
npm install timescaledb  # Time-series database
```

---

## ขั้นตอนที่ 1403: MQTT Broker Setup

```typescript
// src/iot/mqtt/mqtt-broker.service.ts

import { Injectable, Logger, OnModuleInit } from "@nestjs/common";
import Aedes from "aedes";
import { createServer } from "net";
import { createServer as createWebSocketServer } from "http";
import * as ws from "websocket-stream";

@Injectable()
export class MQTTBrokerService implements OnModuleInit {
  private readonly logger = new Logger(MQTTBrokerService.name);
  private aedes: Aedes;

  async onModuleInit() {
    await this.startBroker();
  }

  private async startBroker() {
    this.aedes = new Aedes({
      authenticate: this.authenticate.bind(this),
      authorizePublish: this.authorizePublish.bind(this),
      authorizeSubscribe: this.authorizeSubscribe.bind(this)
    });

    // TCP server (port 1883)
    const tcpServer = createServer(this.aedes.handle);
    tcpServer.listen(1883, () => {
      this.logger.log("MQTT broker started on port 1883");
    });

    // WebSocket server (port 8883 for browser clients)
    const httpServer = createWebSocketServer();
    ws.createServer({ server: httpServer }, this.aedes.handle);
    httpServer.listen(8883, () => {
      this.logger.log("MQTT WebSocket broker started on port 8883");
    });

    // Event handlers
    this.aedes.on("clientConnected", (client) => {
      this.logger.log(`Device connected: ${client.id}`);
    });

    this.aedes.on("clientDisconnected", (client) => {
      this.logger.log(`Device disconnected: ${client.id}`);
    });

    this.aedes.on("publish", (packet, client) => {
      if (client) {  // null = broker itself
        this.logger.debug(`Message from ${client.id}: ${packet.topic}`);
      }
    });
  }

  // Authentication - verify device credentials
  private async authenticate(client: any, username: string, password: Buffer, callback: any) {
    try {
      const deviceId = username;
      const token = password?.toString();
      
      const isValid = await this.validateDeviceToken(deviceId, token);
      callback(null, isValid);
    } catch {
      callback(null, false);
    }
  }

  private async authorizePublish(client: any, packet: any, callback: any) {
    // Only allow devices to publish to their own topics
    const allowedTopics = [
      `/devices/${client.id}/telemetry`,
      `/devices/${client.id}/status`
    ];

    const isAllowed = allowedTopics.some(topic => packet.topic === topic);
    callback(isAllowed ? null : new Error("Unauthorized topic"));
  }

  private async authorizeSubscribe(client: any, subscription: any, callback: any) {
    // Devices can subscribe to their command topics
    const allowedTopics = [`/devices/${client.id}/commands`];
    const isAllowed = allowedTopics.includes(subscription.topic);
    callback(null, isAllowed ? subscription : null);
  }

  private async validateDeviceToken(deviceId: string, token: string): Promise<boolean> {
    // Check token against database
    return true;  // Simplified
  }

  // Send command to device
  async sendCommand(deviceId: string, command: object): Promise<void> {
    const topic = `/devices/${deviceId}/commands`;
    const payload = JSON.stringify(command);

    this.aedes.publish({
      topic,
      payload,
      qos: 1,
      retain: false,
      cmd: "publish",
      dup: false
    } as any, () => {
      this.logger.log(`Command sent to device ${deviceId}`);
    });
  }
}
```

---

## ขั้นตอนที่ 1404: MQTT Client for IoT Devices

```typescript
// src/iot/mqtt/device-client.ts
// Client code that runs on IoT device

import mqtt from "mqtt";

interface SensorReading {
  temperature: number;
  humidity: number;
  pressure: number;
  battery: number;
  timestamp: string;
}

class IoTDeviceClient {
  private client: mqtt.MqttClient;
  private readonly deviceId: string;

  constructor(deviceId: string, brokerUrl: string, token: string) {
    this.deviceId = deviceId;
    
    this.client = mqtt.connect(brokerUrl, {
      clientId: deviceId,
      username: deviceId,
      password: token,
      clean: false,  // Persistent session
      reconnectPeriod: 5000,
      connectTimeout: 10000,
      will: {
        topic: `/devices/${deviceId}/status`,
        payload: JSON.stringify({ online: false, timestamp: new Date() }),
        qos: 1,
        retain: true  // Retained message for last known state
      }
    });

    this.setupHandlers();
  }

  private setupHandlers() {
    this.client.on("connect", () => {
      console.log(`Device ${this.deviceId} connected`);
      
      // Publish online status
      this.publishStatus({ online: true });
      
      // Subscribe to commands
      this.client.subscribe(`/devices/${this.deviceId}/commands`, { qos: 1 });
    });

    this.client.on("message", (topic, message) => {
      if (topic === `/devices/${this.deviceId}/commands`) {
        const command = JSON.parse(message.toString());
        this.handleCommand(command);
      }
    });

    this.client.on("offline", () => {
      console.log(`Device ${this.deviceId} went offline`);
    });

    this.client.on("error", (error) => {
      console.error(`MQTT error: ${error.message}`);
    });
  }

  publishTelemetry(reading: SensorReading) {
    const topic = `/devices/${this.deviceId}/telemetry`;
    const payload = JSON.stringify({
      ...reading,
      deviceId: this.deviceId,
      timestamp: new Date().toISOString()
    });

    this.client.publish(topic, payload, {
      qos: 1,
      retain: false
    });
  }

  publishStatus(status: { online: boolean; [key: string]: any }) {
    const topic = `/devices/${this.deviceId}/status`;
    this.client.publish(topic, JSON.stringify({
      ...status,
      deviceId: this.deviceId,
      timestamp: new Date().toISOString()
    }), {
      qos: 1,
      retain: true  // Retained so new subscribers get last status
    });
  }

  private handleCommand(command: any) {
    console.log(`Received command: ${command.action}`);
    
    switch (command.action) {
      case "reboot":
        console.log("Rebooting device...");
        process.exit(0);
        break;
      case "setInterval":
        this.startReporting(command.interval);
        break;
      case "getStatus":
        this.publishStatus({ online: true, uptime: process.uptime() });
        break;
    }
  }

  private startReporting(intervalMs: number = 5000) {
    setInterval(() => {
      // Simulate sensor readings
      this.publishTelemetry({
        temperature: 25 + Math.random() * 5,
        humidity: 60 + Math.random() * 10,
        pressure: 1013 + Math.random() * 5,
        battery: 3.3 + Math.random() * 0.5,
        timestamp: new Date().toISOString()
      });
    }, intervalMs);
  }
}
```

---

## ขั้นตอนที่ 1405: Telemetry Processor

```typescript
// src/iot/processors/telemetry-processor.service.ts

import { Injectable, Logger } from "@nestjs/common";
import { DataSource } from "typeorm";
import * as mqtt from "mqtt";
import Redis from "ioredis";

interface TelemetryData {
  deviceId: string;
  temperature?: number;
  humidity?: number;
  pressure?: number;
  battery?: number;
  [key: string]: any;
  timestamp: string;
}

@Injectable()
export class TelemetryProcessorService {
  private readonly logger = new Logger(TelemetryProcessorService.name);
  private mqttClient: mqtt.MqttClient;

  constructor(
    private readonly dataSource: DataSource,
    private readonly redis: Redis
  ) {
    this.connectToMQTT();
  }

  private connectToMQTT() {
    this.mqttClient = mqtt.connect(
      process.env.MQTT_BROKER_URL ?? "mqtt://localhost:1883",
      {
        username: "server",
        password: process.env.MQTT_SERVER_PASSWORD
      }
    );

    this.mqttClient.on("connect", () => {
      this.logger.log("Telemetry processor connected to MQTT");
      
      // Subscribe to all device telemetry
      this.mqttClient.subscribe("/devices/+/telemetry", { qos: 1 });
      this.mqttClient.subscribe("/devices/+/status", { qos: 1 });
    });

    this.mqttClient.on("message", async (topic, message) => {
      try {
        const data = JSON.parse(message.toString());
        await this.processTelemetry(topic, data);
      } catch (error) {
        this.logger.error(`Error processing message from ${topic}`);
      }
    });
  }

  private async processTelemetry(topic: string, data: any) {
    const [, , deviceId, type] = topic.split("/");

    if (type === "telemetry") {
      await Promise.all([
        this.storeInTimeSeries(data),
        this.updateDeviceCache(deviceId, data),
        this.checkAlerts(deviceId, data)
      ]);
    } else if (type === "status") {
      await this.updateDeviceStatus(deviceId, data);
    }
  }

  private async storeInTimeSeries(data: TelemetryData): Promise<void> {
    // TimescaleDB hypertable for time-series data
    await this.dataSource.query(`
      INSERT INTO device_telemetry (
        device_id, temperature, humidity, pressure, battery,
        raw_data, recorded_at
      )
      VALUES ($1, $2, $3, $4, $5, $6, $7)
    `, [
      data.deviceId,
      data.temperature ?? null,
      data.humidity ?? null,
      data.pressure ?? null,
      data.battery ?? null,
      JSON.stringify(data),
      data.timestamp
    ]);
  }

  private async updateDeviceCache(deviceId: string, data: any): Promise<void> {
    // Cache latest reading for real-time dashboard
    await this.redis.set(
      `device:${deviceId}:latest`,
      JSON.stringify({ ...data, updatedAt: new Date() }),
      "EX",
      300  // 5 minute TTL
    );
  }

  private async checkAlerts(deviceId: string, data: TelemetryData): Promise<void> {
    const alerts: Array<{ type: string; message: string; severity: string }> = [];

    // Temperature threshold
    if (data.temperature !== undefined) {
      if (data.temperature > 40) {
        alerts.push({
          type: "high_temperature",
          message: `Temperature ${data.temperature.toFixed(1)}°C exceeds 40°C threshold`,
          severity: "critical"
        });
      } else if (data.temperature < 0) {
        alerts.push({
          type: "low_temperature",
          message: `Temperature ${data.temperature.toFixed(1)}°C below 0°C threshold`,
          severity: "warning"
        });
      }
    }

    // Battery level
    if (data.battery !== undefined && data.battery < 3.1) {
      alerts.push({
        type: "low_battery",
        message: `Battery voltage ${data.battery.toFixed(2)}V is low`,
        severity: "warning"
      });
    }

    // Publish alerts
    for (const alert of alerts) {
      this.mqttClient.publish(
        `/alerts/${alert.severity}/${deviceId}`,
        JSON.stringify({ ...alert, deviceId, timestamp: new Date() }),
        { qos: 1 }
      );

      // Also save to database
      await this.dataSource.query(`
        INSERT INTO device_alerts (device_id, type, message, severity, created_at)
        VALUES ($1, $2, $3, $4, NOW())
      `, [deviceId, alert.type, alert.message, alert.severity]);
    }
  }

  private async updateDeviceStatus(deviceId: string, status: any): Promise<void> {
    await this.dataSource.query(`
      INSERT INTO device_status (device_id, online, battery, firmware_version, updated_at)
      VALUES ($1, $2, $3, $4, NOW())
      ON CONFLICT (device_id) DO UPDATE SET
        online = $2,
        battery = $3,
        firmware_version = $4,
        updated_at = NOW()
    `, [deviceId, status.online, status.battery, status.firmwareVersion]);
  }
}
```

---

## ขั้นตอนที่ 1406: Device Registry

```typescript
// src/iot/devices/device-registry.service.ts

import { Injectable, Logger, NotFoundException } from "@nestjs/common";
import { DataSource } from "typeorm";
import { v4 as uuidv4 } from "uuid";
import * as crypto from "crypto";

interface Device {
  id: string;
  name: string;
  type: string;
  location: string;
  serialNumber: string;
  firmwareVersion: string;
  token: string;
  online: boolean;
  lastSeen?: Date;
  metadata?: Record<string, any>;
}

@Injectable()
export class DeviceRegistryService {
  private readonly logger = new Logger(DeviceRegistryService.name);

  constructor(private readonly dataSource: DataSource) {}

  async registerDevice(data: {
    name: string;
    type: string;
    location: string;
    serialNumber: string;
    metadata?: Record<string, any>;
  }): Promise<Device> {
    const id = uuidv4();
    const token = crypto.randomBytes(32).toString("hex");
    const tokenHash = crypto.createHash("sha256").update(token).digest("hex");

    await this.dataSource.query(`
      INSERT INTO devices (id, name, type, location, serial_number, token_hash, metadata, registered_at)
      VALUES ($1, $2, $3, $4, $5, $6, $7, NOW())
    `, [id, data.name, data.type, data.location, data.serialNumber, tokenHash, JSON.stringify(data.metadata ?? {})]);

    this.logger.log(`Registered device ${id}: ${data.name}`);

    return {
      id,
      name: data.name,
      type: data.type,
      location: data.location,
      serialNumber: data.serialNumber,
      firmwareVersion: "0.0.0",
      token,  // Only returned once at registration
      online: false
    };
  }

  async getDevice(deviceId: string): Promise<Device> {
    const result = await this.dataSource.query(
      "SELECT * FROM devices WHERE id = $1",
      [deviceId]
    );

    if (!result[0]) throw new NotFoundException(`Device ${deviceId} not found`);

    return this.mapToDevice(result[0]);
  }

  async listDevices(filters?: { type?: string; online?: boolean; location?: string }) {
    let query = "SELECT d.*, s.online, s.updated_at as last_seen FROM devices d LEFT JOIN device_status s ON d.id = s.device_id WHERE 1=1";
    const params: any[] = [];

    if (filters?.type) {
      params.push(filters.type);
      query += ` AND d.type = $${params.length}`;
    }

    if (filters?.online !== undefined) {
      params.push(filters.online);
      query += ` AND s.online = $${params.length}`;
    }

    if (filters?.location) {
      params.push(`%${filters.location}%`);
      query += ` AND d.location ILIKE $${params.length}`;
    }

    query += " ORDER BY d.registered_at DESC";

    const devices = await this.dataSource.query(query, params);
    return devices.map(this.mapToDevice);
  }

  async validateDeviceToken(deviceId: string, token: string): Promise<boolean> {
    const tokenHash = crypto.createHash("sha256").update(token).digest("hex");

    const result = await this.dataSource.query(
      "SELECT 1 FROM devices WHERE id = $1 AND token_hash = $2",
      [deviceId, tokenHash]
    );

    return result.length > 0;
  }

  private mapToDevice(row: any): Device {
    return {
      id: row.id,
      name: row.name,
      type: row.type,
      location: row.location,
      serialNumber: row.serial_number,
      firmwareVersion: row.firmware_version ?? "unknown",
      token: "",  // Never return token
      online: row.online ?? false,
      lastSeen: row.last_seen,
      metadata: row.metadata
    };
  }
}
```

---

## ขั้นตอนที่ 1407: Time-Series Database Setup

```sql
-- TimescaleDB schema for IoT data

-- Enable TimescaleDB
CREATE EXTENSION IF NOT EXISTS timescaledb;

-- Device telemetry table
CREATE TABLE device_telemetry (
  device_id    VARCHAR(36) NOT NULL,
  temperature  FLOAT,
  humidity     FLOAT,
  pressure     FLOAT,
  battery      FLOAT,
  raw_data     JSONB,
  recorded_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Convert to hypertable (partitioned by time)
SELECT create_hypertable('device_telemetry', 'recorded_at',
  chunk_time_interval => INTERVAL '1 day'
);

-- Index for fast queries
CREATE INDEX ON device_telemetry (device_id, recorded_at DESC);

-- Compression policy (compress chunks older than 7 days)
SELECT add_compression_policy('device_telemetry', INTERVAL '7 days');

-- Retention policy (delete data older than 1 year)
SELECT add_retention_policy('device_telemetry', INTERVAL '1 year');

-- Continuous aggregate for hourly averages
CREATE MATERIALIZED VIEW device_hourly_avg
WITH (timescaledb.continuous) AS
SELECT
  device_id,
  time_bucket('1 hour', recorded_at) as hour,
  AVG(temperature) as avg_temperature,
  AVG(humidity) as avg_humidity,
  AVG(battery) as avg_battery,
  COUNT(*) as reading_count
FROM device_telemetry
GROUP BY device_id, hour;

-- Add refresh policy
SELECT add_continuous_aggregate_policy('device_hourly_avg',
  start_offset => INTERVAL '4 hours',
  end_offset => INTERVAL '1 hour',
  schedule_interval => INTERVAL '1 hour'
);
```

---

## ขั้นตอนที่ 1408: Time-Series Queries

```typescript
// src/iot/analytics/time-series.service.ts

import { Injectable, Logger } from "@nestjs/common";
import { DataSource } from "typeorm";

export interface TimeSeriesQuery {
  deviceId: string;
  metric: string;
  from: Date;
  to: Date;
  interval?: string;
  aggregation?: "avg" | "min" | "max" | "sum" | "count";
}

@Injectable()
export class TimeSeriesService {
  private readonly logger = new Logger(TimeSeriesService.name);

  constructor(private readonly dataSource: DataSource) {}

  async query(params: TimeSeriesQuery): Promise<Array<{ time: Date; value: number }>> {
    const bucket = params.interval ?? "5 minutes";
    const agg = params.aggregation ?? "avg";
    const metric = params.metric;

    const result = await this.dataSource.query(`
      SELECT
        time_bucket($1::interval, recorded_at) as time,
        ${agg}(${metric}) as value
      FROM device_telemetry
      WHERE device_id = $2
        AND recorded_at >= $3
        AND recorded_at <= $4
        AND ${metric} IS NOT NULL
      GROUP BY 1
      ORDER BY 1 ASC
    `, [bucket, params.deviceId, params.from, params.to]);

    return result;
  }

  async getLatestReading(deviceId: string): Promise<any | null> {
    const result = await this.dataSource.query(`
      SELECT *
      FROM device_telemetry
      WHERE device_id = $1
      ORDER BY recorded_at DESC
      LIMIT 1
    `, [deviceId]);

    return result[0] ?? null;
  }

  async getDeviceStats(deviceId: string, hours: number = 24): Promise<any> {
    return this.dataSource.query(`
      SELECT
        AVG(temperature) as avg_temperature,
        MIN(temperature) as min_temperature,
        MAX(temperature) as max_temperature,
        AVG(humidity) as avg_humidity,
        AVG(battery) as avg_battery,
        COUNT(*) as reading_count,
        MAX(recorded_at) as last_reading
      FROM device_telemetry
      WHERE device_id = $1
        AND recorded_at > NOW() - ($2 || ' hours')::interval
    `, [deviceId, hours]);
  }

  async getAnomalies(deviceId: string, metric: string, hours: number = 24): Promise<any[]> {
    // Statistical anomaly detection using PostgreSQL
    return this.dataSource.query(`
      WITH stats AS (
        SELECT
          AVG(${metric}) as mean,
          STDDEV(${metric}) as std
        FROM device_telemetry
        WHERE device_id = $1
          AND recorded_at > NOW() - ($2 || ' hours')::interval
      )
      SELECT
        recorded_at as timestamp,
        ${metric} as value,
        (${metric} - stats.mean) / NULLIF(stats.std, 0) as z_score
      FROM device_telemetry, stats
      WHERE device_id = $1
        AND recorded_at > NOW() - ($2 || ' hours')::interval
        AND ABS((${metric} - stats.mean) / NULLIF(stats.std, 0)) > 3
      ORDER BY recorded_at DESC
    `, [deviceId, hours]);
  }
}
```

---

## ขั้นตอนที่ 1409: WebSocket Real-time Dashboard

```typescript
// src/iot/gateway/iot.gateway.ts

import {
  WebSocketGateway,
  WebSocketServer,
  SubscribeMessage,
  MessageBody,
  ConnectedSocket,
  OnGatewayConnection,
  OnGatewayDisconnect
} from "@nestjs/websockets";
import { Server, Socket } from "socket.io";
import { Logger } from "@nestjs/common";
import Redis from "ioredis";

@WebSocketGateway({
  cors: {
    origin: process.env.DASHBOARD_URL ?? "http://localhost:3001",
    credentials: true
  },
  namespace: "iot"
})
export class IoTGateway implements OnGatewayConnection, OnGatewayDisconnect {
  @WebSocketServer()
  server: Server;

  private readonly logger = new Logger(IoTGateway.name);

  constructor(private readonly redis: Redis) {}

  handleConnection(client: Socket) {
    this.logger.log(`Dashboard connected: ${client.id}`);
  }

  handleDisconnect(client: Socket) {
    this.logger.log(`Dashboard disconnected: ${client.id}`);
  }

  // Subscribe to specific device
  @SubscribeMessage("subscribe:device")
  async handleSubscribeDevice(
    @MessageBody() data: { deviceId: string },
    @ConnectedSocket() client: Socket
  ) {
    const room = `device:${data.deviceId}`;
    client.join(room);
    
    // Send latest reading immediately
    const latest = await this.redis.get(`device:${data.deviceId}:latest`);
    if (latest) {
      client.emit("telemetry", JSON.parse(latest));
    }
    
    this.logger.log(`Client ${client.id} subscribed to device ${data.deviceId}`);
    return { success: true };
  }

  // Unsubscribe from device
  @SubscribeMessage("unsubscribe:device")
  handleUnsubscribeDevice(
    @MessageBody() data: { deviceId: string },
    @ConnectedSocket() client: Socket
  ) {
    client.leave(`device:${data.deviceId}`);
    return { success: true };
  }

  // Broadcast telemetry to all subscribers
  broadcastTelemetry(deviceId: string, data: any) {
    this.server
      .to(`device:${deviceId}`)
      .emit("telemetry", data);
  }

  // Broadcast alert to all connected dashboards
  broadcastAlert(alert: any) {
    this.server.emit("alert", alert);
  }
}
```

---

## ขั้นตอนที่ 1410: CoAP Protocol

```typescript
// src/iot/coap/coap-server.ts
// CoAP (Constrained Application Protocol) for low-power devices

import * as coap from "coap";
import { Logger } from "@nestjs/common";

const logger = new Logger("CoapServer");

export function startCoapServer() {
  const server = coap.createServer((req, res) => {
    const path = req.url;
    const method = req.method;
    
    logger.log(`CoAP ${method} ${path} from ${req.rsinfo.address}`);

    if (method === "POST" && path === "/telemetry") {
      try {
        const data = JSON.parse(req.payload.toString());
        logger.debug(`Received telemetry: ${JSON.stringify(data)}`);
        
        // Process data...
        
        res.code = "2.04";  // Changed
        res.end(JSON.stringify({ success: true }));
      } catch {
        res.code = "4.00";  // Bad Request
        res.end("Invalid JSON");
      }
    } else if (method === "GET" && path.startsWith("/devices/")) {
      const deviceId = path.split("/")[2];
      
      // Return device config
      res.code = "2.05";  // Content
      res.setOption("Content-Format", "application/json");
      res.end(JSON.stringify({
        deviceId,
        reportInterval: 60,
        timezone: "UTC+7"
      }));
    } else {
      res.code = "4.04";  // Not Found
      res.end("Not Found");
    }
  });

  server.listen(5683, () => {
    logger.log("CoAP server started on port 5683 (UDP)");
  });

  return server;
}

// CoAP client (for sending commands to devices)
export async function sendCoapCommand(
  deviceIp: string,
  command: object
): Promise<any> {
  return new Promise((resolve, reject) => {
    const req = coap.request({
      hostname: deviceIp,
      pathname: "/commands",
      method: "POST",
      port: 5683
    });

    req.write(JSON.stringify(command));
    
    req.on("response", (res) => {
      const data = res.payload.toString();
      resolve(JSON.parse(data));
    });

    req.on("error", reject);
    
    req.end();
  });
}
```

---

## ขั้นตอนที่ 1411: Device Shadow Pattern

```typescript
// src/iot/shadow/device-shadow.service.ts
// Device shadow maintains last known state

import { Injectable, Logger } from "@nestjs/common";
import Redis from "ioredis";

interface DeviceShadow {
  state: {
    reported: Record<string, any>;   // What device reports
    desired: Record<string, any>;    // What we want to set
    delta?: Record<string, any>;     // Difference
  };
  metadata: {
    reportedAt?: Date;
    updatedAt: Date;
  };
  version: number;
}

@Injectable()
export class DeviceShadowService {
  private readonly logger = new Logger(DeviceShadowService.name);

  constructor(private readonly redis: Redis) {}

  async updateReported(deviceId: string, reported: Record<string, any>): Promise<void> {
    const shadow = await this.getShadow(deviceId);
    
    shadow.state.reported = { ...shadow.state.reported, ...reported };
    shadow.metadata.reportedAt = new Date();
    shadow.version++;
    
    // Calculate delta (desired - reported)
    shadow.state.delta = this.calculateDelta(shadow.state.desired, shadow.state.reported);
    
    await this.saveShadow(deviceId, shadow);
  }

  async updateDesired(deviceId: string, desired: Record<string, any>): Promise<void> {
    const shadow = await this.getShadow(deviceId);
    
    shadow.state.desired = { ...shadow.state.desired, ...desired };
    shadow.metadata.updatedAt = new Date();
    shadow.version++;
    
    // Recalculate delta
    shadow.state.delta = this.calculateDelta(shadow.state.desired, shadow.state.reported);
    
    await this.saveShadow(deviceId, shadow);
    
    // If there's a delta, notify device
    if (Object.keys(shadow.state.delta).length > 0) {
      this.logger.log(`Device ${deviceId} has pending delta: ${JSON.stringify(shadow.state.delta)}`);
    }
  }

  async getShadow(deviceId: string): Promise<DeviceShadow> {
    const key = `shadow:${deviceId}`;
    const data = await this.redis.get(key);
    
    if (data) {
      return JSON.parse(data);
    }

    // Create new shadow
    return {
      state: { reported: {}, desired: {} },
      metadata: { updatedAt: new Date() },
      version: 0
    };
  }

  private async saveShadow(deviceId: string, shadow: DeviceShadow): Promise<void> {
    await this.redis.set(
      `shadow:${deviceId}`,
      JSON.stringify(shadow),
      "EX",
      86400 * 30  // 30 days
    );
  }

  private calculateDelta(
    desired: Record<string, any>,
    reported: Record<string, any>
  ): Record<string, any> {
    const delta: Record<string, any> = {};
    
    for (const [key, desiredValue] of Object.entries(desired)) {
      if (JSON.stringify(reported[key]) !== JSON.stringify(desiredValue)) {
        delta[key] = desiredValue;
      }
    }
    
    return delta;
  }
}
```

---

## ขั้นตอนที่ 1412: OTA Firmware Updates

```typescript
// src/iot/ota/firmware.service.ts
// Over-the-Air firmware updates

import { Injectable, Logger } from "@nestjs/common";
import { DataSource } from "typeorm";
import { MQTTBrokerService } from "../mqtt/mqtt-broker.service";

interface FirmwareVersion {
  version: string;
  deviceType: string;
  fileSize: number;
  checksum: string;
  downloadUrl: string;
  releaseNotes: string;
  mandatory: boolean;
}

@Injectable()
export class FirmwareService {
  private readonly logger = new Logger(FirmwareService.name);

  constructor(
    private readonly dataSource: DataSource,
    private readonly mqttBroker: MQTTBrokerService
  ) {}

  async publishUpdate(deviceId: string, version: string): Promise<void> {
    const firmware = await this.getFirmwareVersion(version);
    
    const command = {
      action: "ota_update",
      version: firmware.version,
      url: firmware.downloadUrl,
      checksum: firmware.checksum,
      size: firmware.fileSize,
      mandatory: firmware.mandatory
    };
    
    await this.mqttBroker.sendCommand(deviceId, command);
    
    this.logger.log(`Sent OTA update command to device ${deviceId}: v${version}`);
    
    await this.dataSource.query(`
      INSERT INTO ota_updates (device_id, target_version, status, started_at)
      VALUES ($1, $2, 'pending', NOW())
      ON CONFLICT (device_id) DO UPDATE SET
        target_version = $2,
        status = 'pending',
        started_at = NOW()
    `, [deviceId, version]);
  }

  async rolloutUpdate(version: string, deviceType: string, percentage: number): Promise<void> {
    const devices = await this.dataSource.query(`
      SELECT id
      FROM devices
      WHERE type = $1
        AND firmware_version != $2
        AND RANDOM() < $3  -- Random sampling for gradual rollout
      LIMIT 100
    `, [deviceType, version, percentage / 100]);

    this.logger.log(
      `Rolling out v${version} to ${devices.length} devices (${percentage}% rollout)`
    );

    for (const device of devices) {
      await this.publishUpdate(device.id, version);
    }
  }

  async checkForUpdate(deviceId: string, currentVersion: string): Promise<FirmwareVersion | null> {
    const result = await this.dataSource.query(`
      SELECT f.*
      FROM firmware_versions f
      JOIN devices d ON d.type = f.device_type
      WHERE d.id = $1
        AND f.version != $2
        AND f.is_latest = true
      LIMIT 1
    `, [deviceId, currentVersion]);

    return result[0] ?? null;
  }

  private async getFirmwareVersion(version: string): Promise<FirmwareVersion> {
    const result = await this.dataSource.query(
      "SELECT * FROM firmware_versions WHERE version = $1",
      [version]
    );
    return result[0];
  }
}
```

---

## ขั้นตอนที่ 1413: IoT Controller

```typescript
// src/iot/iot.controller.ts

import { Controller, Get, Post, Body, Param, Query, Logger } from "@nestjs/common";
import { DeviceRegistryService } from "./devices/device-registry.service";
import { TimeSeriesService } from "./analytics/time-series.service";
import { DeviceShadowService } from "./shadow/device-shadow.service";
import { MQTTBrokerService } from "./mqtt/mqtt-broker.service";
import { FirmwareService } from "./ota/firmware.service";

@Controller("iot")
export class IoTController {
  private readonly logger = new Logger(IoTController.name);

  constructor(
    private readonly devices: DeviceRegistryService,
    private readonly timeSeries: TimeSeriesService,
    private readonly shadow: DeviceShadowService,
    private readonly mqtt: MQTTBrokerService,
    private readonly firmware: FirmwareService
  ) {}

  @Post("devices/register")
  async registerDevice(@Body() data: any) {
    const device = await this.devices.registerDevice(data);
    return { success: true, device };
  }

  @Get("devices")
  async listDevices(@Query("type") type?: string, @Query("online") online?: string) {
    return this.devices.listDevices({
      type,
      online: online ? online === "true" : undefined
    });
  }

  @Get("devices/:id")
  async getDevice(@Param("id") id: string) {
    return this.devices.getDevice(id);
  }

  @Get("devices/:id/telemetry")
  async getDeviceTelemetry(
    @Param("id") id: string,
    @Query("from") from: string,
    @Query("to") to: string,
    @Query("metric") metric: string = "temperature",
    @Query("interval") interval: string = "5 minutes"
  ) {
    return this.timeSeries.query({
      deviceId: id,
      metric,
      from: new Date(from),
      to: new Date(to),
      interval,
      aggregation: "avg"
    });
  }

  @Get("devices/:id/latest")
  async getLatestReading(@Param("id") id: string) {
    return this.timeSeries.getLatestReading(id);
  }

  @Get("devices/:id/shadow")
  async getDeviceShadow(@Param("id") id: string) {
    return this.shadow.getShadow(id);
  }

  @Post("devices/:id/shadow/desired")
  async updateDesiredState(@Param("id") id: string, @Body() state: any) {
    await this.shadow.updateDesired(id, state);
    return { success: true };
  }

  @Post("devices/:id/commands")
  async sendCommand(@Param("id") id: string, @Body() command: any) {
    await this.mqtt.sendCommand(id, command);
    return { success: true };
  }

  @Post("devices/:id/ota")
  async triggerOTA(@Param("id") id: string, @Body() body: { version: string }) {
    await this.firmware.publishUpdate(id, body.version);
    return { success: true };
  }

  @Get("devices/:id/analytics")
  async getDeviceAnalytics(
    @Param("id") id: string,
    @Query("hours") hours: string = "24"
  ) {
    return this.timeSeries.getDeviceStats(id, parseInt(hours));
  }

  @Get("devices/:id/anomalies")
  async getAnomalies(
    @Param("id") id: string,
    @Query("metric") metric: string = "temperature",
    @Query("hours") hours: string = "24"
  ) {
    return this.timeSeries.getAnomalies(id, metric, parseInt(hours));
  }
}
```

---

## ขั้นตอนที่ 1414: Edge Computing Patterns

```typescript
// src/iot/edge/edge-processor.ts
// Process data at the edge before sending to cloud

interface Rule {
  condition: (value: number) => boolean;
  action: string;
  params?: Record<string, any>;
}

export class EdgeProcessor {
  private rules: Map<string, Rule[]> = new Map();
  private buffer: Array<any> = [];
  private lastFlush = Date.now();

  addRule(metric: string, rule: Rule): void {
    if (!this.rules.has(metric)) {
      this.rules.set(metric, []);
    }
    this.rules.get(metric)!.push(rule);
  }

  process(data: any): {
    shouldSend: boolean;
    actions: Array<{ action: string; params: any }>;
  } {
    const actions: Array<{ action: string; params: any }> = [];

    // Check rules for each metric
    for (const [metric, rules] of this.rules.entries()) {
      const value = data[metric];
      if (value === undefined) continue;

      for (const rule of rules) {
        if (rule.condition(value)) {
          actions.push({ action: rule.action, params: rule.params ?? {} });
        }
      }
    }

    // Buffer data
    this.buffer.push(data);

    // Decide whether to send data to cloud
    const shouldSend = this.shouldFlush(data, actions);
    if (shouldSend) {
      this.lastFlush = Date.now();
    }

    return { shouldSend, actions };
  }

  private shouldFlush(data: any, actions: Array<{ action: string }>): boolean {
    // Always send if there are alerts
    if (actions.some(a => a.action === "alert")) return true;
    
    // Send every 60 seconds
    if (Date.now() - this.lastFlush > 60000) return true;
    
    // Send every 10 readings
    if (this.buffer.length >= 10) return true;
    
    return false;
  }

  getBufferedData(): any[] {
    const data = [...this.buffer];
    this.buffer = [];
    return data;
  }
}

// Example edge rules
const processor = new EdgeProcessor();

processor.addRule("temperature", {
  condition: (v) => v > 35,
  action: "alert",
  params: { severity: "critical", message: "High temperature" }
});

processor.addRule("battery", {
  condition: (v) => v < 3.2,
  action: "alert",
  params: { severity: "warning", message: "Low battery" }
});
```

---

## ขั้นตอนที่ 1415: Device Simulator for Testing

```typescript
// src/iot/__tests__/device-simulator.ts

import mqtt from "mqtt";

export class DeviceSimulator {
  private client: mqtt.MqttClient;
  private interval: NodeJS.Timeout | null = null;
  
  constructor(
    private readonly deviceId: string,
    private readonly brokerUrl: string = "mqtt://localhost:1883"
  ) {}

  async connect(): Promise<void> {
    return new Promise((resolve, reject) => {
      this.client = mqtt.connect(this.brokerUrl, {
        clientId: this.deviceId,
        username: this.deviceId,
        password: "test-token",
        connectTimeout: 5000
      });

      this.client.on("connect", () => resolve());
      this.client.on("error", reject);
    });
  }

  startSending(intervalMs: number = 5000, scenario: "normal" | "hot" | "battery_low" = "normal"): void {
    this.interval = setInterval(() => {
      const reading = this.generateReading(scenario);
      
      this.client.publish(
        `/devices/${this.deviceId}/telemetry`,
        JSON.stringify(reading),
        { qos: 1 }
      );
    }, intervalMs);
  }

  stopSending(): void {
    if (this.interval) {
      clearInterval(this.interval);
      this.interval = null;
    }
  }

  disconnect(): void {
    this.stopSending();
    this.client?.end();
  }

  private generateReading(scenario: string) {
    const base = {
      deviceId: this.deviceId,
      timestamp: new Date().toISOString()
    };

    switch (scenario) {
      case "hot":
        return { ...base, temperature: 42 + Math.random() * 3, humidity: 80, battery: 3.5 };
      case "battery_low":
        return { ...base, temperature: 25, humidity: 60, battery: 3.0 + Math.random() * 0.1 };
      default:
        return {
          ...base,
          temperature: 22 + Math.random() * 8,
          humidity: 50 + Math.random() * 20,
          pressure: 1013 + Math.random() * 5,
          battery: 3.3 + Math.random() * 0.5
        };
    }
  }
}
```

---

## ขั้นตอนที่ 1416: IoT Module

```typescript
// src/iot/iot.module.ts

import { Module } from "@nestjs/common";
import { MQTTBrokerService } from "./mqtt/mqtt-broker.service";
import { TelemetryProcessorService } from "./processors/telemetry-processor.service";
import { DeviceRegistryService } from "./devices/device-registry.service";
import { TimeSeriesService } from "./analytics/time-series.service";
import { DeviceShadowService } from "./shadow/device-shadow.service";
import { IoTGateway } from "./gateway/iot.gateway";
import { IoTController } from "./iot.controller";
import { FirmwareService } from "./ota/firmware.service";

@Module({
  providers: [
    MQTTBrokerService,
    TelemetryProcessorService,
    DeviceRegistryService,
    TimeSeriesService,
    DeviceShadowService,
    IoTGateway,
    FirmwareService
  ],
  controllers: [IoTController],
  exports: [
    DeviceRegistryService,
    TimeSeriesService,
    MQTTBrokerService
  ]
})
export class IoTModule {}
```

---

## ขั้นตอนที่ 1417: Data Aggregation

```typescript
// src/iot/analytics/aggregation.service.ts

import { Injectable, Logger } from "@nestjs/common";
import { DataSource } from "typeorm";
import { Cron } from "@nestjs/schedule";

@Injectable()
export class IoTAggregationService {
  private readonly logger = new Logger(IoTAggregationService.name);

  constructor(private readonly dataSource: DataSource) {}

  // Run aggregations every 5 minutes
  @Cron("*/5 * * * *")
  async aggregateRecentData(): Promise<void> {
    await this.dataSource.query(`
      INSERT INTO device_hourly_stats (
        device_id, hour, avg_temperature, avg_humidity, reading_count
      )
      SELECT
        device_id,
        date_trunc('hour', recorded_at) as hour,
        AVG(temperature),
        AVG(humidity),
        COUNT(*)
      FROM device_telemetry
      WHERE recorded_at > NOW() - INTERVAL '2 hours'
      GROUP BY device_id, date_trunc('hour', recorded_at)
      ON CONFLICT (device_id, hour) DO UPDATE SET
        avg_temperature = EXCLUDED.avg_temperature,
        avg_humidity = EXCLUDED.avg_humidity,
        reading_count = EXCLUDED.reading_count
    `);
    
    this.logger.debug("IoT hourly stats updated");
  }

  async getFleetSummary(): Promise<any> {
    return this.dataSource.query(`
      SELECT
        COUNT(DISTINCT device_id) as total_devices,
        COUNT(DISTINCT CASE WHEN s.online THEN d.id END) as online_devices,
        AVG(t.temperature) as fleet_avg_temperature,
        COUNT(CASE WHEN a.severity = 'critical' AND a.resolved = false THEN 1 END) as critical_alerts
      FROM devices d
      LEFT JOIN device_status s ON d.id = s.device_id
      LEFT JOIN LATERAL (
        SELECT temperature
        FROM device_telemetry
        WHERE device_id = d.id
        ORDER BY recorded_at DESC
        LIMIT 1
      ) t ON true
      LEFT JOIN device_alerts a ON a.device_id = d.id AND a.created_at > NOW() - INTERVAL '1 hour'
    `);
  }
}
```

---

## ขั้นตอนที่ 1418: Alert Management

```typescript
// src/iot/alerts/alert-manager.service.ts

import { Injectable, Logger } from "@nestjs/common";
import { DataSource } from "typeorm";
import Redis from "ioredis";

interface Alert {
  id: string;
  deviceId: string;
  type: string;
  message: string;
  severity: "critical" | "warning" | "info";
  resolved: boolean;
  createdAt: Date;
  resolvedAt?: Date;
}

@Injectable()
export class AlertManagerService {
  private readonly logger = new Logger(AlertManagerService.name);
  
  // Debounce: Don't create duplicate alerts within 5 minutes
  private readonly alertDebounce = new Map<string, Date>();

  constructor(
    private readonly dataSource: DataSource,
    private readonly redis: Redis
  ) {}

  async createAlert(
    deviceId: string,
    type: string,
    message: string,
    severity: "critical" | "warning" | "info"
  ): Promise<Alert | null> {
    const debounceKey = `${deviceId}:${type}`;
    const lastAlertTime = this.alertDebounce.get(debounceKey);
    
    if (lastAlertTime && Date.now() - lastAlertTime.getTime() < 5 * 60 * 1000) {
      return null;  // Too soon, skip
    }

    this.alertDebounce.set(debounceKey, new Date());

    const result = await this.dataSource.query(`
      INSERT INTO device_alerts (device_id, type, message, severity, created_at)
      VALUES ($1, $2, $3, $4, NOW())
      RETURNING *
    `, [deviceId, type, message, severity]);

    const alert = result[0];
    
    // Cache active alerts
    await this.redis.sadd("active_alerts", alert.id);
    
    this.logger.warn(`Alert created: [${severity}] Device ${deviceId} - ${message}`);
    
    return alert;
  }

  async resolveAlert(alertId: string): Promise<void> {
    await this.dataSource.query(`
      UPDATE device_alerts
      SET resolved = true, resolved_at = NOW()
      WHERE id = $1
    `, [alertId]);

    await this.redis.srem("active_alerts", alertId);
    
    this.logger.log(`Alert ${alertId} resolved`);
  }

  async getActiveAlerts(deviceId?: string): Promise<Alert[]> {
    const query = deviceId
      ? "SELECT * FROM device_alerts WHERE resolved = false AND device_id = $1 ORDER BY created_at DESC"
      : "SELECT * FROM device_alerts WHERE resolved = false ORDER BY severity DESC, created_at DESC LIMIT 100";
    
    return this.dataSource.query(query, deviceId ? [deviceId] : []);
  }
}
```

---

## ขั้นตอนที่ 1419: Security for IoT

```typescript
// src/iot/security/device-auth.service.ts

import { Injectable, Logger } from "@nestjs/common";
import * as crypto from "crypto";
import { DataSource } from "typeorm";
import Redis from "ioredis";

@Injectable()
export class DeviceAuthService {
  private readonly logger = new Logger(DeviceAuthService.name);

  constructor(
    private readonly dataSource: DataSource,
    private readonly redis: Redis
  ) {}

  // Validate device JWT token
  async validateToken(deviceId: string, token: string): Promise<boolean> {
    // Check rate limiting first
    const rateLimitKey = `auth:${deviceId}`;
    const attempts = await this.redis.incr(rateLimitKey);
    await this.redis.expire(rateLimitKey, 60);  // 1 minute window
    
    if (attempts > 10) {
      this.logger.warn(`Rate limit exceeded for device ${deviceId}`);
      return false;
    }

    // Validate token
    const tokenHash = crypto.createHash("sha256").update(token).digest("hex");
    
    const result = await this.dataSource.query(
      "SELECT 1 FROM devices WHERE id = $1 AND token_hash = $2 AND active = true",
      [deviceId, tokenHash]
    );

    if (result.length > 0) {
      await this.redis.del(rateLimitKey);  // Reset on success
      return true;
    }

    return false;
  }

  // Rotate device token
  async rotateToken(deviceId: string): Promise<string> {
    const newToken = crypto.randomBytes(32).toString("hex");
    const newHash = crypto.createHash("sha256").update(newToken).digest("hex");

    await this.dataSource.query(
      "UPDATE devices SET token_hash = $1 WHERE id = $2",
      [newHash, deviceId]
    );

    this.logger.log(`Token rotated for device ${deviceId}`);
    return newToken;
  }
}
```

---

## ขั้นตอนที่ 1420: Complete IoT Dashboard

```typescript
// Summary: Full IoT application structure

const iotApplicationSummary = {
  features: [
    "MQTT Broker (Aedes) on port 1883",
    "MQTT WebSocket on port 8883",
    "CoAP server on port 5683",
    "Device registry and authentication",
    "Real-time telemetry processing",
    "TimescaleDB time-series storage",
    "Device shadow (desired/reported state)",
    "Alert management",
    "OTA firmware updates",
    "WebSocket dashboard with Socket.IO",
    "Edge computing patterns",
    "Device simulator for testing",
    "Security (rate limiting, token rotation)"
  ],
  
  mqttTopics: {
    "/devices/{id}/telemetry": "Sensor data from device",
    "/devices/{id}/status": "Device online/offline status",
    "/devices/{id}/commands": "Commands to device",
    "/alerts/{severity}/{id}": "Generated alerts"
  },
  
  apis: {
    "POST /iot/devices/register": "Register new device",
    "GET /iot/devices": "List all devices",
    "GET /iot/devices/:id/telemetry": "Historical data",
    "GET /iot/devices/:id/latest": "Latest reading",
    "GET /iot/devices/:id/shadow": "Device shadow",
    "POST /iot/devices/:id/commands": "Send command",
    "POST /iot/devices/:id/ota": "Trigger OTA update"
  },

  database: {
    "device_telemetry": "TimescaleDB hypertable",
    "device_status": "Current device state",
    "device_alerts": "Alert history",
    "devices": "Device registry",
    "firmware_versions": "OTA firmware versions"
  }
};
```

---

## 🏋️ แบบฝึกหัด

### แบบฝึกหัดที่ 1: MQTT Setup
1. ติดตั้ง MQTT broker ด้วย Aedes
2. สร้าง device client
3. Publish และ subscribe messages

### แบบฝึกหัดที่ 2: Telemetry Processing
1. implement TelemetryProcessorService
2. ตั้งค่า thresholds สำหรับ alerts
3. เก็บข้อมูลลง TimescaleDB

### แบบฝึกหัดที่ 3: Dashboard
1. สร้าง WebSocket gateway
2. ทดสอบ real-time data flow
3. แสดง device status บน dashboard

---

## 📚 สรุป

ใน Part นี้เราได้เรียนรู้:
- IoT architecture patterns
- MQTT protocol กับ Node.js (Aedes broker)
- CoAP protocol สำหรับ low-power devices
- Device registry และ authentication
- Real-time telemetry processing
- Time-series data storage (TimescaleDB)
- Device shadow pattern
- Alert management
- OTA firmware updates
- WebSocket dashboard
- Edge computing patterns
- Device simulator

---

## 🎓 จบ Parts 64-80

หลังจากเรียนจบ 17 Parts นี้ คุณได้เรียนรู้เทคโนโลยีขั้นสูงสำหรับ Node.js:

| Part | หัวข้อ | สิ่งที่เรียนรู้ |
|------|--------|----------------|
| 64 | TypeScript | Type safety, decorators, generics |
| 65 | NestJS | Framework, DI, modules |
| 66 | Advanced NestJS | CQRS, microservices, interceptors |
| 67 | Prisma ORM | Schema, migrations, transactions |
| 68 | Elasticsearch | Full-text search, aggregations |
| 69 | Kafka | Event streaming, producers, consumers |
| 70 | Distributed Tracing | OpenTelemetry, Jaeger |
| 71 | Feature Flags | Gradual rollout, A/B testing |
| 72 | API Gateway | Kong, Nginx, custom gateway |
| 73 | Service Mesh | Istio, mTLS, traffic management |
| 74 | Chaos Engineering | Fault injection, resilience testing |
| 75 | Observability | Logs, metrics, traces, SLO/SLA |
| 76 | Cost Optimization | Auto-scaling, right-sizing, caching |
| 77 | Data Pipelines | ETL, streaming, CDC |
| 78 | Machine Learning | TensorFlow.js, sentiment, fraud detection |
| 79 | Blockchain | Ethereum, Web3.js, NFT |
| 80 | IoT | MQTT, CoAP, time-series data |

**ยินดีด้วย! คุณได้เรียนรู้ 80 Parts จากคอร์สนี้แล้ว 🎉**
