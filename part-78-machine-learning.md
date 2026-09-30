# Part 78 | ขั้นตอนที่ 1361-1380 จาก 1000+

# Machine Learning Integration with Node.js

---

## 🎯 เป้าหมายของ Part นี้

หลังจากเรียนจบ Part นี้ คุณจะสามารถ:
1. integrate TensorFlow.js กับ Node.js
2. implement model serving
3. สร้าง recommendation system
4. ทำ sentiment analysis
5. implement anomaly detection
6. optimize ML inference performance

---

## ขั้นตอนที่ 1361: TensorFlow.js Setup

```bash
# Install TensorFlow.js
npm install @tensorflow/tfjs-node

# For GPU support (NVIDIA)
npm install @tensorflow/tfjs-node-gpu

# Python-based models via tfjs-converter
npm install @tensorflow/tfjs

# Install utilities
npm install sharp  # Image processing
npm install natural  # NLP utilities
```

```typescript
// src/ml/tensorflow-init.ts

import * as tf from "@tensorflow/tfjs-node";
import { Logger } from "@nestjs/common";

const logger = new Logger("TensorFlow");

export async function initTensorFlow() {
  // Set memory growth to avoid allocating all GPU memory
  await tf.ready();
  
  logger.log(`TensorFlow.js backend: ${tf.getBackend()}`);
  logger.log(`TensorFlow.js version: ${tf.version.tfjs}`);
  
  // Configure memory
  tf.env().set("WEBGL_CPU_FORWARD", false);
  tf.env().set("WEBGL_PACK", true);
  
  // Warm up with a small operation
  const warmup = tf.zeros([1, 10]);
  warmup.dispose();
  
  logger.log("TensorFlow.js initialized");
}
```

---

## ขั้นตอนที่ 1362: Model Loading and Caching

```typescript
// src/ml/model-manager.ts

import * as tf from "@tensorflow/tfjs-node";
import { Injectable, Logger, OnModuleInit } from "@nestjs/common";

interface ModelConfig {
  name: string;
  path: string;
  warmupInput?: tf.Tensor;
}

@Injectable()
export class ModelManager implements OnModuleInit {
  private readonly logger = new Logger(ModelManager.name);
  private models = new Map<string, tf.LayersModel | tf.GraphModel>();

  private readonly modelConfigs: ModelConfig[] = [
    {
      name: "sentiment",
      path: "file://./models/sentiment/model.json"
    },
    {
      name: "recommendation",
      path: "file://./models/recommendation/model.json"
    },
    {
      name: "fraud-detection",
      path: "file://./models/fraud/model.json"
    }
  ];

  async onModuleInit() {
    await this.loadAllModels();
  }

  async loadAllModels() {
    for (const config of this.modelConfigs) {
      try {
        await this.loadModel(config);
      } catch (error) {
        this.logger.error(`Failed to load model ${config.name}: ${(error as Error).message}`);
      }
    }
  }

  async loadModel(config: ModelConfig): Promise<void> {
    const startTime = Date.now();
    
    const model = await tf.loadLayersModel(config.path);
    
    // Warm up the model to trigger JIT compilation
    if (config.warmupInput) {
      const warmup = model.predict(config.warmupInput) as tf.Tensor;
      warmup.dispose();
    }
    
    this.models.set(config.name, model);
    this.logger.log(`Model '${config.name}' loaded in ${Date.now() - startTime}ms`);
  }

  getModel(name: string): tf.LayersModel | tf.GraphModel {
    const model = this.models.get(name);
    if (!model) throw new Error(`Model '${name}' not loaded`);
    return model;
  }

  async reloadModel(name: string): Promise<void> {
    const config = this.modelConfigs.find(c => c.name === name);
    if (!config) throw new Error(`Config for model '${name}' not found`);
    
    // Dispose old model to free memory
    const old = this.models.get(name);
    if (old) old.dispose();
    
    await this.loadModel(config);
  }

  getMemoryInfo() {
    return tf.memory();
  }
}
```

---

## ขั้นตอนที่ 1363: Sentiment Analysis

```typescript
// src/ml/sentiment-analysis.service.ts

import * as tf from "@tensorflow/tfjs-node";
import { Injectable, Logger } from "@nestjs/common";
import { ModelManager } from "./model-manager";

interface SentimentResult {
  text: string;
  sentiment: "positive" | "negative" | "neutral";
  confidence: number;
  score: number;  // -1 to 1
}

@Injectable()
export class SentimentAnalysisService {
  private readonly logger = new Logger(SentimentAnalysisService.name);
  private vocab: Map<string, number> = new Map();
  private readonly maxLen = 200;

  constructor(private readonly modelManager: ModelManager) {}

  async analyze(text: string): Promise<SentimentResult> {
    // Tokenize text
    const tokens = this.tokenize(text);
    
    // Convert to tensor
    const padded = this.pad(tokens);
    const input = tf.tensor2d([padded], [1, this.maxLen]);
    
    try {
      const model = this.modelManager.getModel("sentiment");
      const output = model.predict(input) as tf.Tensor;
      
      const scores = await output.data() as Float32Array;
      
      // Model outputs [negative, neutral, positive] probabilities
      const [negative, neutral, positive] = Array.from(scores);
      
      const maxScore = Math.max(negative, neutral, positive);
      let sentiment: "positive" | "negative" | "neutral";
      
      if (positive === maxScore) sentiment = "positive";
      else if (negative === maxScore) sentiment = "negative";
      else sentiment = "neutral";
      
      // Score: positive=1, negative=-1, weighted
      const score = positive - negative;
      
      output.dispose();
      
      return {
        text: text.substring(0, 100),
        sentiment,
        confidence: maxScore,
        score
      };
    } finally {
      input.dispose();
    }
  }

  async analyzeBatch(texts: string[]): Promise<SentimentResult[]> {
    const batchSize = 32;
    const results: SentimentResult[] = [];

    for (let i = 0; i < texts.length; i += batchSize) {
      const batch = texts.slice(i, i + batchSize);
      
      const tokenized = batch.map(t => this.pad(this.tokenize(t)));
      const input = tf.tensor2d(tokenized, [batch.length, this.maxLen]);
      
      try {
        const model = this.modelManager.getModel("sentiment");
        const output = model.predict(input) as tf.Tensor;
        const scores = await output.array() as number[][];
        
        for (let j = 0; j < batch.length; j++) {
          const [neg, neu, pos] = scores[j];
          const maxScore = Math.max(neg, neu, pos);
          
          results.push({
            text: batch[j].substring(0, 100),
            sentiment: pos === maxScore ? "positive" : neg === maxScore ? "negative" : "neutral",
            confidence: maxScore,
            score: pos - neg
          });
        }
        
        output.dispose();
      } finally {
        input.dispose();
      }
    }

    return results;
  }

  private tokenize(text: string): number[] {
    return text
      .toLowerCase()
      .replace(/[^a-z0-9\s]/g, " ")
      .split(/\s+/)
      .filter(w => w.length > 0)
      .map(word => this.vocab.get(word) ?? 1);  // 1 = OOV token
  }

  private pad(tokens: number[]): number[] {
    const padded = new Array(this.maxLen).fill(0);
    const start = Math.max(0, this.maxLen - tokens.length);
    tokens.slice(-this.maxLen).forEach((t, i) => {
      padded[start + i] = t;
    });
    return padded;
  }
}
```

---

## ขั้นตอนที่ 1364: Product Recommendation System

```typescript
// src/ml/recommendation.service.ts

import * as tf from "@tensorflow/tfjs-node";
import { Injectable, Logger } from "@nestjs/common";
import { DataSource } from "typeorm";

interface RecommendationResult {
  productId: string;
  score: number;
  reason: string;
}

@Injectable()
export class RecommendationService {
  private readonly logger = new Logger(RecommendationService.name);
  
  // Simple collaborative filtering using embeddings
  private userEmbeddings?: tf.Variable;
  private itemEmbeddings?: tf.Variable;
  private numUsers = 0;
  private numItems = 0;

  constructor(private readonly dataSource: DataSource) {}

  async trainModel(): Promise<void> {
    this.logger.log("Training recommendation model...");
    
    // Load training data
    const interactions = await this.dataSource.query(`
      SELECT user_id, product_id, 
        CASE WHEN purchased THEN 5.0
             WHEN added_to_cart THEN 3.0
             WHEN viewed THEN 1.0
             ELSE 0 END as rating
      FROM user_product_interactions
      WHERE created_at > NOW() - INTERVAL '90 days'
    `);

    // Create user/item mappings
    const userIds = [...new Set(interactions.map((r: any) => r.user_id))];
    const itemIds = [...new Set(interactions.map((r: any) => r.product_id))];
    
    this.numUsers = userIds.length;
    this.numItems = itemIds.length;
    
    const userMap = new Map(userIds.map((id, i) => [id, i]));
    const itemMap = new Map(itemIds.map((id, i) => [id, i]));

    // Initialize embeddings
    const embeddingDim = 32;
    this.userEmbeddings = tf.variable(
      tf.randomNormal([this.numUsers, embeddingDim], 0, 0.1)
    );
    this.itemEmbeddings = tf.variable(
      tf.randomNormal([this.numItems, embeddingDim], 0, 0.1)
    );

    // Training data
    const userIndices = interactions.map((r: any) => userMap.get(r.user_id)!);
    const itemIndices = interactions.map((r: any) => itemMap.get(r.product_id)!);
    const ratings = interactions.map((r: any) => r.rating / 5.0);  // Normalize

    const userTensor = tf.tensor1d(userIndices, "int32");
    const itemTensor = tf.tensor1d(itemIndices, "int32");
    const ratingTensor = tf.tensor1d(ratings);

    // Train with SGD
    const optimizer = tf.train.adam(0.01);
    const epochs = 100;
    
    for (let epoch = 0; epoch < epochs; epoch++) {
      const loss = optimizer.minimize(() => {
        const userEmb = tf.gather(this.userEmbeddings!, userTensor);
        const itemEmb = tf.gather(this.itemEmbeddings!, itemTensor);
        
        // Dot product for predicted rating
        const predicted = tf.sum(tf.mul(userEmb, itemEmb), 1);
        
        // MSE loss
        return tf.mean(tf.square(tf.sub(predicted, ratingTensor))) as tf.Scalar;
      }, true);

      if (epoch % 10 === 0) {
        const lossValue = await loss!.data();
        this.logger.debug(`Epoch ${epoch}: loss = ${lossValue[0].toFixed(4)}`);
      }
      loss?.dispose();
    }

    userTensor.dispose();
    itemTensor.dispose();
    ratingTensor.dispose();

    this.logger.log("Recommendation model trained");
  }

  async getRecommendations(
    userId: string,
    limit: number = 10
  ): Promise<RecommendationResult[]> {
    if (!this.userEmbeddings || !this.itemEmbeddings) {
      throw new Error("Model not trained");
    }

    // This is simplified - in production, maintain userId → index mapping
    const userIndex = 0;  // Placeholder

    const userEmb = tf.gather(this.userEmbeddings, tf.scalar(userIndex, "int32"));
    
    // Compute scores for all items
    const scores = tf.sum(
      tf.mul(tf.expandDims(userEmb, 0), this.itemEmbeddings),
      1
    );
    
    const scoreData = await scores.data() as Float32Array;
    
    // Get top-N items
    const indexed = Array.from(scoreData).map((score, idx) => ({ score, idx }));
    indexed.sort((a, b) => b.score - a.score);
    
    const topItems = indexed.slice(0, limit);
    
    userEmb.dispose();
    scores.dispose();

    return topItems.map(({ score, idx }) => ({
      productId: `product-${idx}`,  // In production, reverse map
      score,
      reason: "Based on your purchase history"
    }));
  }
}
```

---

## ขั้นตอนที่ 1365: Fraud Detection

```typescript
// src/ml/fraud-detection.service.ts

import * as tf from "@tensorflow/tfjs-node";
import { Injectable, Logger } from "@nestjs/common";
import { ModelManager } from "./model-manager";

interface TransactionFeatures {
  amount: number;
  merchantCategory: string;
  country: string;
  deviceFingerprint: string;
  hour: number;
  dayOfWeek: number;
  velocityLast1h: number;   // Transactions in last hour
  velocityLast24h: number;  // Transactions in last 24h
  isNewDevice: boolean;
  isNewCountry: boolean;
  avgTransactionAmount: number;
  amountDeviation: number;  // z-score from user average
}

interface FraudPrediction {
  isFraud: boolean;
  confidence: number;
  riskScore: number;  // 0-100
  riskFactors: string[];
}

@Injectable()
export class FraudDetectionService {
  private readonly logger = new Logger(FraudDetectionService.name);
  
  // Category encoding
  private readonly merchantCategories = new Map<string, number>([
    ["retail", 0], ["restaurant", 1], ["travel", 2],
    ["entertainment", 3], ["crypto", 4], ["gambling", 5]
  ]);

  constructor(private readonly modelManager: ModelManager) {}

  async detectFraud(features: TransactionFeatures): Promise<FraudPrediction> {
    const tensor = this.featuresToTensor(features);
    
    try {
      const model = this.modelManager.getModel("fraud-detection");
      const output = model.predict(tensor) as tf.Tensor;
      const [fraudProbability] = await output.data() as Float32Array;
      
      output.dispose();
      
      const riskScore = fraudProbability * 100;
      const isFraud = riskScore > 70;
      const riskFactors = this.identifyRiskFactors(features);
      
      return {
        isFraud,
        confidence: fraudProbability,
        riskScore,
        riskFactors
      };
    } finally {
      tensor.dispose();
    }
  }

  private featuresToTensor(features: TransactionFeatures): tf.Tensor {
    const vec = [
      // Normalize amount (log scale)
      Math.log1p(features.amount) / 10,
      
      // Category one-hot (simplified to index)
      (this.merchantCategories.get(features.merchantCategory) ?? 7) / 7,
      
      // Time features
      features.hour / 24,
      features.dayOfWeek / 7,
      
      // Velocity features (normalized)
      Math.min(features.velocityLast1h / 10, 1),
      Math.min(features.velocityLast24h / 50, 1),
      
      // Boolean features
      features.isNewDevice ? 1 : 0,
      features.isNewCountry ? 1 : 0,
      
      // Statistical features
      Math.min(Math.abs(features.amountDeviation) / 5, 1)  // Clamp at 5 std deviations
    ];

    return tf.tensor2d([vec], [1, vec.length]);
  }

  private identifyRiskFactors(features: TransactionFeatures): string[] {
    const factors: string[] = [];

    if (features.isNewDevice) factors.push("New device");
    if (features.isNewCountry) factors.push("New country");
    if (features.velocityLast1h > 5) factors.push("High transaction velocity");
    if (features.amountDeviation > 3) factors.push("Unusual amount (>3σ)");
    if (features.merchantCategory === "crypto") factors.push("Cryptocurrency merchant");
    if (features.hour >= 0 && features.hour <= 5) factors.push("Unusual time (midnight)");

    return factors;
  }
}
```

---

## ขั้นตอนที่ 1366: Image Classification

```typescript
// src/ml/image-classifier.service.ts

import * as tf from "@tensorflow/tfjs-node";
import * as sharp from "sharp";
import { Injectable, Logger } from "@nestjs/common";
import { ModelManager } from "./model-manager";

interface ClassificationResult {
  label: string;
  confidence: number;
  topK: Array<{ label: string; confidence: number }>;
}

@Injectable()
export class ImageClassifierService {
  private readonly logger = new Logger(ImageClassifierService.name);
  private labels: string[] = [];

  constructor(private readonly modelManager: ModelManager) {}

  async classify(
    imageBuffer: Buffer,
    topK: number = 5
  ): Promise<ClassificationResult> {
    // Preprocess image
    const tensor = await this.preprocessImage(imageBuffer);
    
    try {
      const model = this.modelManager.getModel("image-classifier");
      const output = model.predict(tensor) as tf.Tensor;
      
      const probabilities = await output.data() as Float32Array;
      
      // Get top K predictions
      const predictions = Array.from(probabilities)
        .map((conf, idx) => ({ label: this.labels[idx] ?? `class-${idx}`, confidence: conf }))
        .sort((a, b) => b.confidence - a.confidence);

      output.dispose();

      return {
        label: predictions[0].label,
        confidence: predictions[0].confidence,
        topK: predictions.slice(0, topK)
      };
    } finally {
      tensor.dispose();
    }
  }

  private async preprocessImage(buffer: Buffer): Promise<tf.Tensor> {
    // Resize to 224x224 (standard for MobileNet/ResNet)
    const resized = await sharp(buffer)
      .resize(224, 224, { fit: "cover" })
      .raw()
      .toBuffer();

    // Convert to tensor [1, 224, 224, 3] normalized to [-1, 1]
    return tf.tidy(() => {
      const pixelData = new Float32Array(resized);
      const tensor = tf.tensor3d(Array.from(pixelData), [224, 224, 3]);
      
      // Normalize: [0, 255] → [-1, 1]
      return tensor.div(127.5).sub(1).expandDims(0);
    });
  }
}
```

---

## ขั้นตอนที่ 1367: Text Embeddings with Node.js

```typescript
// src/ml/text-embedding.service.ts

import { Injectable, Logger } from "@nestjs/common";
import Redis from "ioredis";

@Injectable()
export class TextEmbeddingService {
  private readonly logger = new Logger(TextEmbeddingService.name);

  constructor(private readonly redis: Redis) {}

  // Use OpenAI embeddings API or local model
  async embed(text: string): Promise<number[]> {
    const cacheKey = `embedding:${this.hashText(text)}`;
    
    const cached = await this.redis.get(cacheKey);
    if (cached) return JSON.parse(cached);

    // Option 1: Use OpenAI API
    const embedding = await this.callOpenAIEmbeddings(text);
    
    // Cache for 24 hours
    await this.redis.set(cacheKey, JSON.stringify(embedding), "EX", 86400);
    
    return embedding;
  }

  async embedBatch(texts: string[]): Promise<number[][]> {
    const { default: pLimit } = await import("p-limit");
    const limit = pLimit(10);  // Max 10 concurrent API calls
    
    return Promise.all(
      texts.map(text => limit(() => this.embed(text)))
    );
  }

  // Cosine similarity between two vectors
  cosineSimilarity(a: number[], b: number[]): number {
    if (a.length !== b.length) throw new Error("Vectors must have same length");
    
    let dotProduct = 0, normA = 0, normB = 0;
    
    for (let i = 0; i < a.length; i++) {
      dotProduct += a[i] * b[i];
      normA += a[i] * a[i];
      normB += b[i] * b[i];
    }
    
    return dotProduct / (Math.sqrt(normA) * Math.sqrt(normB));
  }

  // Find most similar texts
  findMostSimilar(
    queryEmbedding: number[],
    candidates: Array<{ text: string; embedding: number[] }>,
    topK: number = 5
  ): Array<{ text: string; similarity: number }> {
    return candidates
      .map(c => ({
        text: c.text,
        similarity: this.cosineSimilarity(queryEmbedding, c.embedding)
      }))
      .sort((a, b) => b.similarity - a.similarity)
      .slice(0, topK);
  }

  private async callOpenAIEmbeddings(text: string): Promise<number[]> {
    const response = await fetch("https://api.openai.com/v1/embeddings", {
      method: "POST",
      headers: {
        "Authorization": `Bearer ${process.env.OPENAI_API_KEY}`,
        "Content-Type": "application/json"
      },
      body: JSON.stringify({
        model: "text-embedding-3-small",
        input: text
      })
    });

    const data = await response.json() as any;
    return data.data[0].embedding;
  }

  private hashText(text: string): string {
    const crypto = require("crypto");
    return crypto.createHash("md5").update(text).digest("hex");
  }
}
```

---

## ขั้นตอนที่ 1368: Anomaly Detection

```typescript
// src/ml/anomaly-detection.service.ts

import { Injectable, Logger } from "@nestjs/common";

interface AnomalyResult {
  isAnomaly: boolean;
  score: number;       // 0-1, higher = more anomalous
  zScore: number;
  message?: string;
}

@Injectable()
export class AnomalyDetectionService {
  private readonly logger = new Logger(AnomalyDetectionService.name);
  
  // Running statistics (Welford's algorithm for memory efficiency)
  private readonly stats = new Map<string, RunningStats>();

  // Detect anomaly using statistical approach
  detect(
    key: string,
    value: number,
    threshold: number = 3  // z-score threshold
  ): AnomalyResult {
    const stats = this.getOrCreateStats(key);
    const zScore = this.calculateZScore(value, stats);
    
    // Update running stats
    this.updateStats(stats, value);
    
    const isAnomaly = Math.abs(zScore) > threshold;
    const score = Math.min(Math.abs(zScore) / (threshold * 2), 1);

    return {
      isAnomaly,
      score,
      zScore,
      message: isAnomaly ? `Value ${value} is ${zScore.toFixed(1)} std deviations from mean` : undefined
    };
  }

  // Detect anomalies in time series
  detectTimeSeries(
    values: number[],
    windowSize: number = 10
  ): boolean[] {
    const anomalies: boolean[] = new Array(values.length).fill(false);
    
    for (let i = windowSize; i < values.length; i++) {
      const window = values.slice(i - windowSize, i);
      const mean = window.reduce((a, b) => a + b, 0) / window.length;
      const std = Math.sqrt(
        window.reduce((sum, v) => sum + Math.pow(v - mean, 2), 0) / window.length
      );
      
      const zScore = std > 0 ? (values[i] - mean) / std : 0;
      anomalies[i] = Math.abs(zScore) > 3;
    }
    
    return anomalies;
  }

  // IQR-based anomaly detection (robust to non-normal distributions)
  detectIQR(values: number[]): boolean[] {
    const sorted = [...values].sort((a, b) => a - b);
    const q1 = sorted[Math.floor(sorted.length * 0.25)];
    const q3 = sorted[Math.floor(sorted.length * 0.75)];
    const iqr = q3 - q1;
    
    const lower = q1 - 1.5 * iqr;
    const upper = q3 + 1.5 * iqr;
    
    return values.map(v => v < lower || v > upper);
  }

  private getOrCreateStats(key: string): RunningStats {
    if (!this.stats.has(key)) {
      this.stats.set(key, { count: 0, mean: 0, m2: 0 });
    }
    return this.stats.get(key)!;
  }

  private calculateZScore(value: number, stats: RunningStats): number {
    if (stats.count < 2) return 0;
    const variance = stats.m2 / (stats.count - 1);
    const std = Math.sqrt(variance);
    return std > 0 ? (value - stats.mean) / std : 0;
  }

  private updateStats(stats: RunningStats, value: number): void {
    stats.count++;
    const delta = value - stats.mean;
    stats.mean += delta / stats.count;
    const delta2 = value - stats.mean;
    stats.m2 += delta * delta2;
  }
}

interface RunningStats {
  count: number;
  mean: number;
  m2: number;  // Sum of squared deviations (for Welford's algorithm)
}
```

---

## ขั้นตอนที่ 1369: ML Model Serving API

```typescript
// src/ml/ml-prediction.controller.ts

import { Controller, Post, Body, UseGuards, Logger } from "@nestjs/common";
import { SentimentAnalysisService } from "./sentiment-analysis.service";
import { FraudDetectionService } from "./fraud-detection.service";
import { RecommendationService } from "./recommendation.service";

@Controller("ml")
export class MLPredictionController {
  private readonly logger = new Logger(MLPredictionController.name);

  constructor(
    private readonly sentiment: SentimentAnalysisService,
    private readonly fraudDetection: FraudDetectionService,
    private readonly recommendations: RecommendationService
  ) {}

  @Post("sentiment")
  async analyzeSentiment(@Body() body: { text: string }) {
    const result = await this.sentiment.analyze(body.text);
    return { success: true, prediction: result };
  }

  @Post("sentiment/batch")
  async analyzeSentimentBatch(@Body() body: { texts: string[] }) {
    if (body.texts.length > 100) {
      return { error: "Maximum 100 texts per batch" };
    }
    
    const results = await this.sentiment.analyzeBatch(body.texts);
    return { success: true, predictions: results };
  }

  @Post("fraud/detect")
  async detectFraud(@Body() transaction: any) {
    const result = await this.fraudDetection.detectFraud(transaction);
    
    if (result.isFraud) {
      this.logger.warn(`Fraud detected: score=${result.riskScore}, factors=${result.riskFactors.join(", ")}`);
    }
    
    return { success: true, prediction: result };
  }

  @Post("recommendations")
  async getRecommendations(@Body() body: { userId: string; limit?: number }) {
    const results = await this.recommendations.getRecommendations(
      body.userId,
      body.limit ?? 10
    );
    return { success: true, recommendations: results };
  }
}
```

---

## ขั้นตอนที่ 1370: Feature Engineering

```typescript
// src/ml/feature-engineering.ts

export class FeatureEngineer {
  
  // Date/time features
  extractTimeFeatures(date: Date): Record<string, number> {
    return {
      hour: date.getHours(),
      dayOfWeek: date.getDay(),
      dayOfMonth: date.getDate(),
      month: date.getMonth(),
      isWeekend: date.getDay() === 0 || date.getDay() === 6 ? 1 : 0,
      isBusinessHours: (date.getHours() >= 9 && date.getHours() <= 17) ? 1 : 0
    };
  }

  // Text features
  extractTextFeatures(text: string): Record<string, number> {
    const words = text.toLowerCase().split(/\s+/);
    return {
      wordCount: words.length,
      charCount: text.length,
      avgWordLength: text.replace(/\s/g, "").length / (words.length || 1),
      exclamationCount: (text.match(/!/g) ?? []).length,
      questionCount: (text.match(/\?/g) ?? []).length,
      uppercaseRatio: (text.match(/[A-Z]/g) ?? []).length / (text.length || 1)
    };
  }

  // Statistical features from history
  extractStatisticalFeatures(values: number[]): Record<string, number> {
    if (values.length === 0) return {};
    
    const sorted = [...values].sort((a, b) => a - b);
    const mean = values.reduce((a, b) => a + b, 0) / values.length;
    const variance = values.reduce((sum, v) => sum + Math.pow(v - mean, 2), 0) / values.length;
    
    return {
      mean,
      std: Math.sqrt(variance),
      min: sorted[0],
      max: sorted[sorted.length - 1],
      median: sorted[Math.floor(sorted.length / 2)],
      q25: sorted[Math.floor(sorted.length * 0.25)],
      q75: sorted[Math.floor(sorted.length * 0.75)]
    };
  }

  // One-hot encoding
  oneHot(value: string, categories: string[]): number[] {
    return categories.map(cat => value === cat ? 1 : 0);
  }

  // Min-max normalization
  normalize(value: number, min: number, max: number): number {
    if (max === min) return 0;
    return (value - min) / (max - min);
  }

  // Log transform for skewed distributions
  logTransform(value: number): number {
    return Math.log1p(Math.max(0, value));
  }

  // Z-score normalization
  standardize(value: number, mean: number, std: number): number {
    if (std === 0) return 0;
    return (value - mean) / std;
  }
}
```

---

## ขั้นตอนที่ 1371: A/B Testing for ML Models

```typescript
// src/ml/model-ab-testing.ts

import { Injectable, Logger } from "@nestjs/common";
import { createHash } from "crypto";

interface ModelVariant {
  name: string;
  weight: number;  // Traffic percentage
  modelFn: (input: any) => Promise<any>;
}

@Injectable()
export class MLModelABTesting {
  private readonly logger = new Logger(MLModelABTesting.name);
  
  private variants: ModelVariant[] = [];
  private metrics = new Map<string, VariantMetrics>();

  addVariant(variant: ModelVariant): void {
    this.variants.push(variant);
    this.metrics.set(variant.name, {
      requests: 0,
      errors: 0,
      totalLatency: 0
    });
  }

  async predict(userId: string, input: any): Promise<{ result: any; variant: string }> {
    const variant = this.assignVariant(userId);
    const metrics = this.metrics.get(variant.name)!;
    
    metrics.requests++;
    const startTime = Date.now();
    
    try {
      const result = await variant.modelFn(input);
      metrics.totalLatency += Date.now() - startTime;
      
      return { result, variant: variant.name };
    } catch (error) {
      metrics.errors++;
      throw error;
    }
  }

  private assignVariant(userId: string): ModelVariant {
    // Consistent assignment based on user ID
    const hash = parseInt(
      createHash("md5").update(userId).digest("hex").slice(0, 8),
      16
    );
    const bucket = (hash % 100) + 1;
    
    let cumulative = 0;
    for (const variant of this.variants) {
      cumulative += variant.weight * 100;
      if (bucket <= cumulative) return variant;
    }
    
    return this.variants[this.variants.length - 1];
  }

  getMetrics(): Record<string, VariantMetrics> {
    const result: Record<string, VariantMetrics> = {};
    this.metrics.forEach((metrics, name) => {
      result[name] = {
        ...metrics,
        avgLatency: metrics.requests > 0 ? metrics.totalLatency / metrics.requests : 0,
        errorRate: metrics.requests > 0 ? metrics.errors / metrics.requests : 0
      };
    });
    return result;
  }
}

interface VariantMetrics {
  requests: number;
  errors: number;
  totalLatency: number;
  avgLatency?: number;
  errorRate?: number;
}
```

---

## ขั้นตอนที่ 1372: Model Performance Monitoring

```typescript
// src/ml/model-monitor.ts

import * as promClient from "prom-client";
import { Injectable, Logger } from "@nestjs/common";

@Injectable()
export class MLModelMonitor {
  private readonly logger = new Logger(MLModelMonitor.name);

  private readonly predictionCounter = new promClient.Counter({
    name: "ml_predictions_total",
    help: "Total ML predictions made",
    labelNames: ["model", "result"]
  });

  private readonly predictionLatency = new promClient.Histogram({
    name: "ml_prediction_latency_ms",
    help: "ML prediction latency",
    labelNames: ["model"],
    buckets: [5, 10, 25, 50, 100, 250, 500, 1000, 2500]
  });

  private readonly modelScore = new promClient.Gauge({
    name: "ml_model_score",
    help: "ML model performance score",
    labelNames: ["model", "metric"]
  });

  recordPrediction(
    model: string,
    result: string,
    latencyMs: number
  ) {
    this.predictionCounter.inc({ model, result });
    this.predictionLatency.observe({ model }, latencyMs);
  }

  updateModelScore(model: string, metric: string, value: number) {
    this.modelScore.set({ model, metric }, value);
    
    if (metric === "accuracy" && value < 0.8) {
      this.logger.warn(`Model ${model} accuracy dropped to ${(value * 100).toFixed(1)}%`);
    }
  }

  // Track prediction confidence distribution
  private confidenceHistogram = new promClient.Histogram({
    name: "ml_prediction_confidence",
    help: "Distribution of prediction confidence scores",
    labelNames: ["model"],
    buckets: [0.1, 0.2, 0.3, 0.4, 0.5, 0.6, 0.7, 0.8, 0.9, 1.0]
  });

  recordConfidence(model: string, confidence: number) {
    this.confidenceHistogram.observe({ model }, confidence);
  }
}
```

---

## ขั้นตอนที่ 1373: Feature Store

```typescript
// src/ml/feature-store.ts
// Centralized store for ML features

import { Injectable, Logger } from "@nestjs/common";
import Redis from "ioredis";
import { DataSource } from "typeorm";

interface Feature {
  name: string;
  value: number | string | number[];
  computedAt: Date;
}

@Injectable()
export class FeatureStore {
  private readonly logger = new Logger(FeatureStore.name);

  constructor(
    private readonly redis: Redis,
    private readonly dataSource: DataSource
  ) {}

  async getFeatures(entityId: string, featureNames: string[]): Promise<Record<string, any>> {
    const cacheKey = `features:${entityId}`;
    
    // Try Redis first (online store)
    const cached = await this.redis.hgetall(cacheKey);
    
    const result: Record<string, any> = {};
    const missing: string[] = [];
    
    for (const name of featureNames) {
      if (cached[name] !== undefined) {
        result[name] = JSON.parse(cached[name]);
      } else {
        missing.push(name);
      }
    }

    // Fetch missing from database (offline store)
    if (missing.length > 0) {
      const dbFeatures = await this.fetchFromDB(entityId, missing);
      Object.assign(result, dbFeatures);
      
      // Cache in Redis
      const pipeline = this.redis.multi();
      for (const [name, value] of Object.entries(dbFeatures)) {
        pipeline.hset(cacheKey, name, JSON.stringify(value));
      }
      pipeline.expire(cacheKey, 3600);  // 1 hour TTL
      await pipeline.exec();
    }

    return result;
  }

  async updateFeatures(entityId: string, features: Record<string, any>): Promise<void> {
    const cacheKey = `features:${entityId}`;
    
    const pipeline = this.redis.multi();
    for (const [name, value] of Object.entries(features)) {
      pipeline.hset(cacheKey, name, JSON.stringify(value));
    }
    pipeline.expire(cacheKey, 3600);
    await pipeline.exec();
  }

  private async fetchFromDB(entityId: string, featureNames: string[]): Promise<Record<string, any>> {
    const features = await this.dataSource.query(`
      SELECT name, value FROM feature_store
      WHERE entity_id = $1 AND name = ANY($2)
    `, [entityId, featureNames]);

    return features.reduce((result: any, f: any) => {
      result[f.name] = JSON.parse(f.value);
      return result;
    }, {});
  }
}
```

---

## ขั้นตอนที่ 1374: Model Training Pipeline

```typescript
// src/ml/training/training-pipeline.ts

import * as tf from "@tensorflow/tfjs-node";
import { Injectable, Logger } from "@nestjs/common";
import { DataSource } from "typeorm";

@Injectable()
export class ModelTrainingPipeline {
  private readonly logger = new Logger(ModelTrainingPipeline.name);

  constructor(private readonly dataSource: DataSource) {}

  async trainSentimentModel(): Promise<tf.LayersModel> {
    this.logger.log("Training sentiment model...");

    // Load training data
    const trainingData = await this.dataSource.query(`
      SELECT text, label FROM product_reviews
      WHERE label IS NOT NULL
      ORDER BY RANDOM()
      LIMIT 50000
    `);

    const texts: string[] = trainingData.map((r: any) => r.text);
    const labels: number[] = trainingData.map((r: any) => parseInt(r.label));

    // Build model
    const model = this.buildSentimentModel(10000, 200, 32);
    
    model.compile({
      optimizer: tf.train.adam(0.001),
      loss: "categoricalCrossentropy",
      metrics: ["accuracy"]
    });

    // Prepare data (simplified - would normally tokenize properly)
    const X = tf.zeros([texts.length, 200]);
    const y = tf.oneHot(tf.tensor1d(labels, "int32"), 3);

    // Train
    await model.fit(X, y, {
      epochs: 20,
      batchSize: 32,
      validationSplit: 0.1,
      callbacks: {
        onEpochEnd: (epoch, logs) => {
          this.logger.log(
            `Epoch ${epoch + 1}: loss=${logs?.loss.toFixed(4)}, ` +
            `accuracy=${((logs?.acc ?? 0) * 100).toFixed(1)}%`
          );
        }
      }
    });

    X.dispose();
    y.dispose();

    // Save model
    await model.save("file://./models/sentiment/model.json");
    this.logger.log("Sentiment model saved");

    return model;
  }

  private buildSentimentModel(
    vocabSize: number,
    maxLen: number,
    embeddingDim: number
  ): tf.LayersModel {
    const model = tf.sequential({
      layers: [
        tf.layers.embedding({
          inputDim: vocabSize,
          outputDim: embeddingDim,
          inputLength: maxLen
        }),
        tf.layers.globalAveragePooling1d(),
        tf.layers.dense({ units: 64, activation: "relu" }),
        tf.layers.dropout({ rate: 0.3 }),
        tf.layers.dense({ units: 32, activation: "relu" }),
        tf.layers.dense({ units: 3, activation: "softmax" })
      ]
    });

    return model;
  }
}
```

---

## ขั้นตอนที่ 1375: Batch Predictions

```typescript
// src/ml/batch-prediction.service.ts

import { Injectable, Logger } from "@nestjs/common";
import { Cron } from "@nestjs/schedule";
import { DataSource } from "typeorm";
import { SentimentAnalysisService } from "./sentiment-analysis.service";

@Injectable()
export class BatchPredictionService {
  private readonly logger = new Logger(BatchPredictionService.name);

  constructor(
    private readonly dataSource: DataSource,
    private readonly sentiment: SentimentAnalysisService
  ) {}

  // Run batch predictions nightly
  @Cron("0 3 * * *")
  async runBatchSentimentAnalysis() {
    this.logger.log("Starting batch sentiment analysis...");
    
    const batchSize = 100;
    let offset = 0;
    let totalProcessed = 0;

    while (true) {
      const reviews = await this.dataSource.query(`
        SELECT id, text
        FROM product_reviews
        WHERE sentiment IS NULL
        ORDER BY created_at DESC
        LIMIT $1 OFFSET $2
      `, [batchSize, offset]);

      if (reviews.length === 0) break;

      const texts = reviews.map((r: any) => r.text);
      const results = await this.sentiment.analyzeBatch(texts);

      // Bulk update
      const values = results.map((r, i) =>
        `('${reviews[i].id}', '${r.sentiment}', ${r.score})`
      ).join(",");
      
      await this.dataSource.query(`
        INSERT INTO review_sentiments (review_id, sentiment, score)
        VALUES ${values}
        ON CONFLICT (review_id) DO UPDATE SET
          sentiment = EXCLUDED.sentiment,
          score = EXCLUDED.score,
          analyzed_at = NOW()
      `);

      totalProcessed += reviews.length;
      offset += batchSize;
      
      this.logger.debug(`Processed ${totalProcessed} reviews`);
    }

    this.logger.log(`Batch analysis complete: ${totalProcessed} reviews`);
  }
}
```

---

## ขั้นตอนที่ 1376: Real-time Predictions

```typescript
// src/ml/realtime-prediction.service.ts

import { Injectable, Logger } from "@nestjs/common";
import Redis from "ioredis";
import { SentimentAnalysisService } from "./sentiment-analysis.service";
import { FraudDetectionService } from "./fraud-detection.service";

@Injectable()
export class RealtimePredictionService {
  private readonly logger = new Logger(RealtimePredictionService.name);

  constructor(
    private readonly redis: Redis,
    private readonly sentiment: SentimentAnalysisService,
    private readonly fraud: FraudDetectionService
  ) {}

  // Review is submitted - analyze sentiment immediately
  async analyzeReview(reviewId: string, text: string): Promise<void> {
    const result = await this.sentiment.analyze(text);
    
    // Cache result
    await this.redis.setex(
      `sentiment:${reviewId}`,
      3600,
      JSON.stringify(result)
    );
    
    // Publish event for downstream consumers
    await this.redis.publish("sentiment-analyzed", JSON.stringify({
      reviewId,
      sentiment: result.sentiment,
      score: result.score
    }));
  }

  // Transaction is made - check for fraud immediately
  async analyzeTransaction(transactionId: string, features: any): Promise<boolean> {
    const result = await this.fraud.detectFraud(features);
    
    if (result.isFraud) {
      // Publish fraud alert
      await this.redis.publish("fraud-alert", JSON.stringify({
        transactionId,
        riskScore: result.riskScore,
        riskFactors: result.riskFactors,
        timestamp: new Date()
      }));
      
      return true;
    }
    
    return false;
  }

  async getCachedSentiment(reviewId: string): Promise<any | null> {
    const cached = await this.redis.get(`sentiment:${reviewId}`);
    return cached ? JSON.parse(cached) : null;
  }
}
```

---

## ขั้นตอนที่ 1377: ML Model Registry

```typescript
// src/ml/model-registry.ts

import { Injectable, Logger } from "@nestjs/common";
import { DataSource } from "typeorm";

interface ModelVersion {
  id: string;
  name: string;
  version: string;
  path: string;
  metrics: Record<string, number>;
  tags: string[];
  status: "staging" | "production" | "archived";
  createdAt: Date;
  promotedAt?: Date;
}

@Injectable()
export class ModelRegistry {
  private readonly logger = new Logger(ModelRegistry.name);

  constructor(private readonly dataSource: DataSource) {}

  async registerModel(model: Omit<ModelVersion, "id" | "createdAt">): Promise<string> {
    const result = await this.dataSource.query(`
      INSERT INTO ml_model_registry (name, version, path, metrics, tags, status, created_at)
      VALUES ($1, $2, $3, $4, $5, $6, NOW())
      RETURNING id
    `, [
      model.name, model.version, model.path,
      JSON.stringify(model.metrics), model.tags.join(","), model.status
    ]);

    const id = result[0].id;
    this.logger.log(`Registered model ${model.name} v${model.version} (${id})`);
    return id;
  }

  async promoteToProduction(modelId: string): Promise<void> {
    // Archive current production model
    await this.dataSource.query(`
      UPDATE ml_model_registry
      SET status = 'archived'
      WHERE name = (
        SELECT name FROM ml_model_registry WHERE id = $1
      ) AND status = 'production'
    `, [modelId]);

    // Promote new model
    await this.dataSource.query(`
      UPDATE ml_model_registry
      SET status = 'production', promoted_at = NOW()
      WHERE id = $1
    `, [modelId]);

    this.logger.log(`Model ${modelId} promoted to production`);
  }

  async getProductionModel(name: string): Promise<ModelVersion | null> {
    const result = await this.dataSource.query(`
      SELECT * FROM ml_model_registry
      WHERE name = $1 AND status = 'production'
      LIMIT 1
    `, [name]);

    return result[0] ?? null;
  }
}
```

---

## ขั้นตอนที่ 1378: ML in NestJS Module

```typescript
// src/ml/ml.module.ts

import { Module, OnModuleInit } from "@nestjs/common";
import { ScheduleModule } from "@nestjs/schedule";
import { ModelManager } from "./model-manager";
import { SentimentAnalysisService } from "./sentiment-analysis.service";
import { FraudDetectionService } from "./fraud-detection.service";
import { RecommendationService } from "./recommendation.service";
import { AnomalyDetectionService } from "./anomaly-detection.service";
import { MLModelMonitor } from "./model-monitor";
import { FeatureStore } from "./feature-store";
import { ModelRegistry } from "./model-registry";
import { BatchPredictionService } from "./batch-prediction.service";
import { RealtimePredictionService } from "./realtime-prediction.service";
import { MLPredictionController } from "./ml-prediction.controller";
import { initTensorFlow } from "./tensorflow-init";

@Module({
  imports: [ScheduleModule.forRoot()],
  providers: [
    ModelManager,
    SentimentAnalysisService,
    FraudDetectionService,
    RecommendationService,
    AnomalyDetectionService,
    MLModelMonitor,
    FeatureStore,
    ModelRegistry,
    BatchPredictionService,
    RealtimePredictionService
  ],
  controllers: [MLPredictionController],
  exports: [
    SentimentAnalysisService,
    FraudDetectionService,
    RecommendationService,
    AnomalyDetectionService,
    RealtimePredictionService
  ]
})
export class MLModule implements OnModuleInit {
  async onModuleInit() {
    await initTensorFlow();
  }
}
```

---

## ขั้นตอนที่ 1379: Testing ML Components

```typescript
// src/ml/__tests__/fraud-detection.spec.ts

import { Test, TestingModule } from "@nestjs/testing";
import { FraudDetectionService } from "../fraud-detection.service";
import * as tf from "@tensorflow/tfjs-node";

describe("FraudDetectionService", () => {
  let service: FraudDetectionService;

  const mockModelManager = {
    getModel: jest.fn().mockReturnValue({
      predict: (tensor: tf.Tensor) => {
        const batch = tensor.shape[0];
        return tf.tensor2d(
          Array.from({ length: batch }, () => [Math.random()]),
          [batch, 1]
        );
      }
    })
  };

  beforeEach(async () => {
    const module: TestingModule = await Test.createTestingModule({
      providers: [
        FraudDetectionService,
        { provide: "ModelManager", useValue: mockModelManager }
      ]
    }).compile();

    service = module.get<FraudDetectionService>(FraudDetectionService);
  });

  it("should return fraud prediction", async () => {
    const features = {
      amount: 5000,
      merchantCategory: "crypto",
      country: "US",
      deviceFingerprint: "abc123",
      hour: 3,
      dayOfWeek: 1,
      velocityLast1h: 8,
      velocityLast24h: 20,
      isNewDevice: true,
      isNewCountry: false,
      avgTransactionAmount: 100,
      amountDeviation: 4.5
    };

    const result = await service.detectFraud(features);

    expect(result).toHaveProperty("isFraud");
    expect(result).toHaveProperty("confidence");
    expect(result.riskFactors).toContain("Unusual time (midnight)");
    expect(result.riskFactors).toContain("Cryptocurrency merchant");
  });
});
```

---

## ขั้นตอนที่ 1380: ML Performance Optimization

```typescript
// src/ml/optimization/inference-optimizer.ts

import * as tf from "@tensorflow/tfjs-node";

export class InferenceOptimizer {
  
  // Batch requests to maximize GPU utilization
  static createBatcher<T, R>(
    predict: (batch: T[]) => Promise<R[]>,
    options: {
      maxBatchSize: number;
      maxWaitMs: number;
    }
  ) {
    const queue: Array<{
      input: T;
      resolve: (value: R) => void;
      reject: (error: Error) => void;
    }> = [];
    let timer: NodeJS.Timeout | null = null;

    const flush = async () => {
      if (queue.length === 0) return;
      if (timer) { clearTimeout(timer); timer = null; }
      
      const batch = queue.splice(0, options.maxBatchSize);
      
      try {
        const inputs = batch.map(b => b.input);
        const results = await predict(inputs);
        batch.forEach((b, i) => b.resolve(results[i]));
      } catch (error) {
        batch.forEach(b => b.reject(error as Error));
      }
    };

    return (input: T): Promise<R> => {
      return new Promise<R>((resolve, reject) => {
        queue.push({ input, resolve, reject });
        
        if (queue.length >= options.maxBatchSize) {
          flush();
        } else if (!timer) {
          timer = setTimeout(flush, options.maxWaitMs);
        }
      });
    };
  }

  // Use tf.tidy to automatically dispose tensors
  static withTensorCleanup<T>(fn: () => T): T {
    return tf.tidy(fn);
  }
  
  // Convert model to int8 for faster inference
  static async quantizeModel(
    model: tf.LayersModel
  ): Promise<void> {
    // TensorFlow.js supports quantized models via converter
    // Example: converting from float32 to int8 reduces model size ~4x
    // and improves inference speed 2-3x on CPU
    console.log("Model quantization requires the TF.js model converter tool");
  }
}
```

---

## 🏋️ แบบฝึกหัด

### แบบฝึกหัดที่ 1: Sentiment Analysis
1. implement SentimentAnalysisService
2. ทดสอบ batch analysis
3. Cache predictions ใน Redis

### แบบฝึกหัดที่ 2: Fraud Detection
1. implement feature extraction
2. สร้าง simple fraud detection model
3. ทดสอบ false positive rate

### แบบฝึกหัดที่ 3: Recommendation System
1. สร้าง collaborative filtering model
2. implement top-N recommendations
3. A/B test สอง models

---

## 📚 สรุป

ใน Part นี้เราได้เรียนรู้:
- TensorFlow.js setup กับ Node.js
- Model loading and caching
- Sentiment analysis
- Fraud detection
- Product recommendations
- Anomaly detection
- Feature engineering
- Model serving API
- Batch and real-time predictions
- Model monitoring

**Part ถัดไป**: Blockchain Integration
