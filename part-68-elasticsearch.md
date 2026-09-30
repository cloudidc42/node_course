# Part 68 | ขั้นตอนที่ 1161-1180 จาก 1000+

# Elasticsearch กับ Node.js

---

## 🎯 เป้าหมายของ Part นี้

หลังจากเรียนจบ Part นี้ คุณจะสามารถ:
1. ติดตั้งและกำหนดค่า Elasticsearch
2. สร้าง Index และ Mappings
3. ทำ Full-text Search และ Advanced Queries
4. ใช้ Aggregations สำหรับ Analytics
5. Sync ข้อมูลจาก Database ไปยัง Elasticsearch
6. Optimize performance

---

## ขั้นตอนที่ 1161: Elasticsearch คืออะไร?

Elasticsearch เป็น distributed search และ analytics engine ที่สร้างบน Apache Lucene ใช้สำหรับ full-text search, log analytics, และ real-time analytics

```
Elasticsearch Concepts:
┌─────────────────────────────────────────────────────┐
│  Index     ≈ Database table                         │
│  Document  ≈ Row in table                           │
│  Field     ≈ Column                                 │
│  Mapping   ≈ Schema                                 │
│  Shard     = Horizontal partition of index          │
│  Replica   = Copy of shard for redundancy           │
└─────────────────────────────────────────────────────┘
```

---

## ขั้นตอนที่ 1162: การติดตั้ง

```bash
# ติดตั้ง Elasticsearch ด้วย Docker
docker run -d \
  --name elasticsearch \
  -p 9200:9200 \
  -p 9300:9300 \
  -e "discovery.type=single-node" \
  -e "xpack.security.enabled=false" \
  elasticsearch:8.10.0

# ทดสอบ
curl http://localhost:9200

# ติดตั้ง Kibana (UI)
docker run -d \
  --name kibana \
  --link elasticsearch:elasticsearch \
  -p 5601:5601 \
  kibana:8.10.0

# ติดตั้ง Node.js client
npm install @elastic/elasticsearch
```

```typescript
// src/elasticsearch/elasticsearch.service.ts
import { Injectable, OnModuleInit } from "@nestjs/common";
import { ElasticsearchService } from "@nestjs/elasticsearch";
import { Client } from "@elastic/elasticsearch";

@Injectable()
export class ElasticService implements OnModuleInit {
  constructor(private readonly esService: ElasticsearchService) {}

  async onModuleInit() {
    // Test connection
    const health = await this.esService.cluster.health();
    console.log("Elasticsearch status:", health.status);
  }
}
```

---

## ขั้นตอนที่ 1163: Index Mappings

```typescript
// src/search/indices/products.index.ts
import { Injectable, OnModuleInit } from "@nestjs/common";
import { ElasticsearchService } from "@nestjs/elasticsearch";

@Injectable()
export class ProductsIndex implements OnModuleInit {
  readonly INDEX_NAME = "products";
  
  constructor(private readonly esService: ElasticsearchService) {}

  async onModuleInit() {
    await this.createIndex();
  }

  async createIndex() {
    const exists = await this.esService.indices.exists({
      index: this.INDEX_NAME
    });
    
    if (!exists) {
      await this.esService.indices.create({
        index: this.INDEX_NAME,
        body: {
          settings: {
            number_of_shards: 1,
            number_of_replicas: 0,
            analysis: {
              analyzer: {
                // Thai text analyzer
                thai_analyzer: {
                  type: "custom",
                  tokenizer: "thai",
                  filter: ["lowercase", "stop"]
                },
                // Edge n-gram for autocomplete
                autocomplete_analyzer: {
                  type: "custom",
                  tokenizer: "autocomplete_tokenizer",
                  filter: ["lowercase"]
                }
              },
              tokenizer: {
                autocomplete_tokenizer: {
                  type: "edge_ngram",
                  min_gram: 2,
                  max_gram: 20,
                  token_chars: ["letter", "digit"]
                }
              }
            }
          },
          mappings: {
            properties: {
              id: { type: "keyword" },
              name: {
                type: "text",
                analyzer: "standard",
                fields: {
                  keyword: { type: "keyword" },
                  autocomplete: { 
                    type: "text",
                    analyzer: "autocomplete_analyzer",
                    search_analyzer: "standard"
                  }
                }
              },
              description: { type: "text", analyzer: "standard" },
              price: { type: "scaled_float", scaling_factor: 100 },
              stock: { type: "integer" },
              category: {
                type: "object",
                properties: {
                  id: { type: "keyword" },
                  name: { type: "keyword" }
                }
              },
              tags: { type: "keyword" },
              rating: { type: "float" },
              reviewCount: { type: "integer" },
              isActive: { type: "boolean" },
              createdAt: { type: "date" },
              updatedAt: { type: "date" }
            }
          }
        }
      });
      
      console.log(`Index ${this.INDEX_NAME} created`);
    }
  }

  async deleteIndex() {
    await this.esService.indices.delete({ index: this.INDEX_NAME });
  }
}
```

---

## ขั้นตอนที่ 1164: Indexing Documents

```typescript
// src/search/products-search.service.ts
import { Injectable } from "@nestjs/common";
import { ElasticsearchService } from "@nestjs/elasticsearch";

interface ProductDocument {
  id: string;
  name: string;
  description?: string;
  price: number;
  stock: number;
  category?: { id: string; name: string };
  tags?: string[];
  rating?: number;
  reviewCount?: number;
  isActive: boolean;
  createdAt: Date;
  updatedAt: Date;
}

@Injectable()
export class ProductsSearchService {
  private readonly INDEX = "products";
  
  constructor(private readonly esService: ElasticsearchService) {}

  // Index single document
  async index(product: ProductDocument): Promise<void> {
    await this.esService.index({
      index: this.INDEX,
      id: product.id,
      document: {
        ...product,
        createdAt: product.createdAt.toISOString(),
        updatedAt: product.updatedAt.toISOString()
      }
    });
  }

  // Bulk index
  async bulkIndex(products: ProductDocument[]): Promise<void> {
    const operations = products.flatMap(product => [
      { index: { _index: this.INDEX, _id: product.id } },
      {
        ...product,
        createdAt: product.createdAt.toISOString(),
        updatedAt: product.updatedAt.toISOString()
      }
    ]);
    
    const { errors, items } = await this.esService.bulk({
      operations,
      refresh: true
    });
    
    if (errors) {
      const failedItems = items
        .filter(item => item.index?.error)
        .map(item => item.index?.error);
      console.error("Bulk index errors:", failedItems);
    }
  }

  // Update document
  async update(id: string, updates: Partial<ProductDocument>): Promise<void> {
    await this.esService.update({
      index: this.INDEX,
      id,
      doc: {
        ...updates,
        updatedAt: new Date().toISOString()
      }
    });
  }

  // Delete document
  async delete(id: string): Promise<void> {
    await this.esService.delete({ index: this.INDEX, id });
  }
}
```

---

## ขั้นตอนที่ 1165: Full-text Search

```typescript
// src/search/products-search.service.ts (continued)

interface SearchParams {
  query?: string;
  minPrice?: number;
  maxPrice?: number;
  categories?: string[];
  tags?: string[];
  minRating?: number;
  inStock?: boolean;
  page?: number;
  limit?: number;
  sortBy?: string;
  sortOrder?: "asc" | "desc";
}

interface SearchResult<T> {
  hits: T[];
  total: number;
  took: number;
}

async search(params: SearchParams): Promise<SearchResult<ProductDocument>> {
  const {
    query, minPrice, maxPrice, categories, tags,
    minRating, inStock, page = 1, limit = 20,
    sortBy = "_score", sortOrder = "desc"
  } = params;
  
  // Build query
  const must: any[] = [];
  const filter: any[] = [];
  
  // Full-text search
  if (query) {
    must.push({
      multi_match: {
        query,
        fields: ["name^3", "description^1", "tags^2"],
        type: "best_fields",
        fuzziness: "AUTO"
      }
    });
  }
  
  // Price range filter
  if (minPrice !== undefined || maxPrice !== undefined) {
    filter.push({
      range: {
        price: {
          ...(minPrice !== undefined && { gte: minPrice }),
          ...(maxPrice !== undefined && { lte: maxPrice })
        }
      }
    });
  }
  
  // Category filter
  if (categories?.length) {
    filter.push({ terms: { "category.id": categories } });
  }
  
  // Tags filter
  if (tags?.length) {
    filter.push({ terms: { tags } });
  }
  
  // Rating filter
  if (minRating !== undefined) {
    filter.push({ range: { rating: { gte: minRating } } });
  }
  
  // Stock filter
  if (inStock) {
    filter.push({ range: { stock: { gt: 0 } } });
  }
  
  // Active filter
  filter.push({ term: { isActive: true } });
  
  const response = await this.esService.search<ProductDocument>({
    index: this.INDEX,
    from: (page - 1) * limit,
    size: limit,
    query: {
      bool: { must, filter }
    },
    sort: sortBy === "_score"
      ? [{ _score: { order: sortOrder } }]
      : [{ [sortBy]: { order: sortOrder } }],
    highlight: {
      fields: {
        name: { number_of_fragments: 0 },
        description: { number_of_fragments: 3, fragment_size: 150 }
      },
      pre_tags: ["<em>"],
      post_tags: ["</em>"]
    }
  });
  
  const hits = response.hits.hits.map(hit => ({
    ...hit._source!,
    _score: hit._score,
    _highlights: hit.highlight
  }));
  
  return {
    hits,
    total: typeof response.hits.total === "number"
      ? response.hits.total
      : response.hits.total?.value ?? 0,
    took: response.took
  };
}
```

---

## ขั้นตอนที่ 1166: Autocomplete

```typescript
// Autocomplete search
async autocomplete(query: string, limit: number = 5): Promise<string[]> {
  const response = await this.esService.search({
    index: this.INDEX,
    body: {
      size: limit,
      query: {
        multi_match: {
          query,
          fields: ["name.autocomplete"],
          type: "bool_prefix"
        }
      },
      _source: ["name"]
    }
  });
  
  return response.hits.hits.map(hit => (hit._source as any).name);
}

// Suggest with completion field
async suggest(text: string): Promise<string[]> {
  const response = await this.esService.search({
    index: this.INDEX,
    body: {
      suggest: {
        product_suggest: {
          prefix: text,
          completion: {
            field: "name_suggest",
            size: 5,
            fuzzy: { fuzziness: 1 }
          }
        }
      }
    }
  });
  
  const suggestions = response.suggest?.product_suggest?.[0]?.options ?? [];
  return suggestions.map((s: any) => s.text);
}
```

---

## ขั้นตอนที่ 1167: Aggregations

```typescript
// src/search/analytics.service.ts

@Injectable()
export class SearchAnalyticsService {
  constructor(private readonly esService: ElasticsearchService) {}

  async getProductAnalytics() {
    const response = await this.esService.search({
      index: "products",
      size: 0, // Don't return documents, only aggregations
      aggs: {
        // Category breakdown
        by_category: {
          terms: {
            field: "category.name",
            size: 20,
            order: { total_revenue: "desc" }
          },
          aggs: {
            product_count: { value_count: { field: "id" } },
            avg_price: { avg: { field: "price" } },
            price_range: {
              stats: { field: "price" }
            },
            total_revenue: {
              sum: { field: "price" }
            }
          }
        },
        
        // Price histogram
        price_distribution: {
          histogram: {
            field: "price",
            interval: 100,
            min_doc_count: 1
          }
        },
        
        // Rating distribution
        rating_distribution: {
          range: {
            field: "rating",
            ranges: [
              { to: 2, key: "1-2" },
              { from: 2, to: 3, key: "2-3" },
              { from: 3, to: 4, key: "3-4" },
              { from: 4, key: "4-5" }
            ]
          }
        },
        
        // Top tags
        top_tags: {
          terms: {
            field: "tags",
            size: 20
          }
        },
        
        // In stock vs out of stock
        stock_status: {
          filters: {
            filters: {
              in_stock: { range: { stock: { gt: 0 } } },
              out_of_stock: { term: { stock: 0 } }
            }
          }
        },
        
        // Products created over time
        products_over_time: {
          date_histogram: {
            field: "createdAt",
            calendar_interval: "month",
            format: "yyyy-MM"
          }
        }
      }
    });
    
    return response.aggregations;
  }

  async getSearchFacets(query: string) {
    const response = await this.esService.search({
      index: "products",
      body: {
        query: {
          multi_match: {
            query,
            fields: ["name^3", "description"]
          }
        },
        aggs: {
          categories: {
            terms: { field: "category.name", size: 10 }
          },
          price_ranges: {
            range: {
              field: "price",
              ranges: [
                { key: "Under $50", to: 50 },
                { key: "$50-$100", from: 50, to: 100 },
                { key: "$100-$500", from: 100, to: 500 },
                { key: "Over $500", from: 500 }
              ]
            }
          },
          tags: {
            terms: { field: "tags", size: 20 }
          },
          avg_rating: {
            avg: { field: "rating" }
          }
        },
        size: 0
      }
    });
    
    return response.aggregations;
  }
}
```

---

## ขั้นตอนที่ 1168: Data Synchronization

```typescript
// src/sync/sync.service.ts
import { Injectable, Logger } from "@nestjs/common";
import { InjectRepository } from "@nestjs/typeorm";
import { Repository } from "typeorm";
import { Cron, CronExpression } from "@nestjs/schedule";
import { Product } from "../products/entities/product.entity";
import { ProductsSearchService } from "../search/products-search.service";

@Injectable()
export class SyncService {
  private readonly logger = new Logger(SyncService.name);

  constructor(
    @InjectRepository(Product)
    private readonly productRepository: Repository<Product>,
    private readonly productsSearchService: ProductsSearchService
  ) {}

  // Full reindex
  async reindexAll(): Promise<void> {
    this.logger.log("Starting full reindex...");
    
    let page = 0;
    const batchSize = 1000;
    let processedCount = 0;
    
    while (true) {
      const products = await this.productRepository.find({
        skip: page * batchSize,
        take: batchSize,
        relations: ["category"],
        order: { id: "ASC" }
      });
      
      if (products.length === 0) break;
      
      await this.productsSearchService.bulkIndex(
        products.map(p => ({
          id: p.id,
          name: p.name,
          description: p.description,
          price: Number(p.price),
          stock: p.stock,
          category: p.category
            ? { id: p.category.id, name: p.category.name }
            : undefined,
          isActive: true,
          createdAt: p.createdAt,
          updatedAt: p.updatedAt
        }))
      );
      
      processedCount += products.length;
      this.logger.log(`Indexed ${processedCount} products`);
      page++;
    }
    
    this.logger.log(`Reindex completed. Total: ${processedCount} products`);
  }

  // Scheduled incremental sync
  @Cron(CronExpression.EVERY_5_MINUTES)
  async incrementalSync(): Promise<void> {
    const lastSyncTime = await this.getLastSyncTime();
    
    const updatedProducts = await this.productRepository.find({
      where: {
        updatedAt: { $gt: lastSyncTime } as any
      },
      relations: ["category"]
    });
    
    if (updatedProducts.length > 0) {
      await this.productsSearchService.bulkIndex(
        updatedProducts.map(p => this.mapProduct(p))
      );
      this.logger.log(`Synced ${updatedProducts.length} products`);
    }
    
    await this.updateLastSyncTime();
  }

  private mapProduct(product: any) {
    return {
      id: product.id,
      name: product.name,
      description: product.description,
      price: Number(product.price),
      stock: product.stock,
      category: product.category,
      isActive: product.isActive,
      createdAt: product.createdAt,
      updatedAt: product.updatedAt
    };
  }

  private async getLastSyncTime(): Promise<Date> {
    // Store in Redis or database
    return new Date(Date.now() - 5 * 60 * 1000); // 5 minutes ago
  }

  private async updateLastSyncTime(): Promise<void> {
    // Update in Redis or database
  }
}
```

---

## ขั้นตอนที่ 1169: Log Analytics

```typescript
// src/logging/log-analytics.service.ts

interface LogEntry {
  timestamp: Date;
  level: string;
  message: string;
  service: string;
  requestId?: string;
  userId?: string;
  method?: string;
  url?: string;
  statusCode?: number;
  duration?: number;
  error?: {
    message: string;
    stack: string;
  };
  metadata?: Record<string, any>;
}

@Injectable()
export class LogAnalyticsService {
  private readonly INDEX = "app-logs";

  constructor(private readonly esService: ElasticsearchService) {}

  async ingest(log: LogEntry): Promise<void> {
    await this.esService.index({
      index: `${this.INDEX}-${new Date().toISOString().slice(0, 7)}`, // Monthly indices
      document: {
        ...log,
        "@timestamp": log.timestamp.toISOString()
      }
    });
  }

  async queryLogs(params: {
    startTime: Date;
    endTime: Date;
    level?: string;
    service?: string;
    query?: string;
    page?: number;
    limit?: number;
  }) {
    const { startTime, endTime, level, service, query, page = 1, limit = 100 } = params;
    
    const filter: any[] = [
      {
        range: {
          "@timestamp": {
            gte: startTime.toISOString(),
            lte: endTime.toISOString()
          }
        }
      }
    ];
    
    if (level) filter.push({ term: { level } });
    if (service) filter.push({ term: { service } });
    
    const must: any[] = [];
    if (query) {
      must.push({
        multi_match: {
          query,
          fields: ["message", "error.message"]
        }
      });
    }
    
    const response = await this.esService.search({
      index: `${this.INDEX}-*`,
      from: (page - 1) * limit,
      size: limit,
      sort: [{ "@timestamp": "desc" }],
      query: { bool: { must, filter } }
    });
    
    return {
      logs: response.hits.hits.map(h => h._source),
      total: (response.hits.total as any).value
    };
  }

  async getErrorSummary(hours: number = 24) {
    const response = await this.esService.search({
      index: `${this.INDEX}-*`,
      size: 0,
      query: {
        bool: {
          filter: [
            { term: { level: "error" } },
            {
              range: {
                "@timestamp": {
                  gte: `now-${hours}h`,
                  lte: "now"
                }
              }
            }
          ]
        }
      },
      aggs: {
        errors_over_time: {
          date_histogram: {
            field: "@timestamp",
            fixed_interval: "1h"
          }
        },
        top_errors: {
          terms: {
            field: "error.message.keyword",
            size: 10
          }
        },
        by_service: {
          terms: {
            field: "service",
            size: 20
          }
        }
      }
    });
    
    return response.aggregations;
  }
}
```

---

## ขั้นตอนที่ 1170: Elasticsearch กับ NestJS Module

```typescript
// src/search/search.module.ts
import { Module } from "@nestjs/common";
import { ElasticsearchModule } from "@nestjs/elasticsearch";
import { ConfigModule, ConfigService } from "@nestjs/config";
import { ProductsSearchService } from "./products-search.service";
import { SearchAnalyticsService } from "./analytics.service";
import { ProductsIndex } from "./indices/products.index";

@Module({
  imports: [
    ElasticsearchModule.registerAsync({
      imports: [ConfigModule],
      useFactory: async (configService: ConfigService) => ({
        node: configService.get<string>("ELASTICSEARCH_URL", "http://localhost:9200"),
        auth: {
          username: configService.get<string>("ELASTICSEARCH_USERNAME"),
          password: configService.get<string>("ELASTICSEARCH_PASSWORD")
        },
        maxRetries: 10,
        requestTimeout: 60000,
        sniffOnStart: true
      }),
      inject: [ConfigService]
    })
  ],
  providers: [ProductsSearchService, SearchAnalyticsService, ProductsIndex],
  exports: [ProductsSearchService, SearchAnalyticsService]
})
export class SearchModule {}
```

---

## ขั้นตอนที่ 1171: More Query Types

```typescript
// Geo-spatial queries
async findNearby(lat: number, lon: number, distanceKm: number) {
  return this.esService.search({
    index: "stores",
    query: {
      bool: {
        filter: {
          geo_distance: {
            distance: `${distanceKm}km`,
            location: { lat, lon }
          }
        }
      }
    },
    sort: [
      {
        _geo_distance: {
          location: { lat, lon },
          order: "asc",
          unit: "km"
        }
      }
    ]
  });
}

// Nested queries
async searchWithNestedReviews(minRating: number) {
  return this.esService.search({
    index: "products",
    query: {
      nested: {
        path: "reviews",
        query: {
          range: {
            "reviews.rating": { gte: minRating }
          }
        },
        inner_hits: {
          size: 3,
          sort: [{ "reviews.rating": "desc" }]
        }
      }
    }
  });
}

// Percolate query (stored queries)
async storeQuery(queryId: string, query: any) {
  await this.esService.index({
    index: ".percolator",
    id: queryId,
    document: { query }
  });
}

async matchDocument(document: any) {
  return this.esService.search({
    index: "products",
    query: {
      percolate: {
        field: "query",
        document
      }
    }
  });
}
```

---

## ขั้นตอนที่ 1172: Index Aliases และ Zero-downtime Reindex

```typescript
// src/search/index-management.service.ts
@Injectable()
export class IndexManagementService {
  constructor(private readonly esService: ElasticsearchService) {}

  async zeroDowntimeReindex(aliasName: string) {
    const timestamp = Date.now();
    const newIndexName = `${aliasName}-${timestamp}`;
    
    // 1. Create new index
    await this.createIndex(newIndexName);
    
    // 2. Reindex data to new index
    await this.esService.reindex({
      source: { index: aliasName },
      dest: { index: newIndexName }
    });
    
    // 3. Get current index for alias
    const aliasInfo = await this.esService.indices.getAlias({ name: aliasName });
    const currentIndex = Object.keys(aliasInfo)[0];
    
    // 4. Atomically swap alias
    await this.esService.indices.updateAliases({
      actions: [
        { remove: { index: currentIndex, alias: aliasName } },
        { add: { index: newIndexName, alias: aliasName } }
      ]
    });
    
    // 5. Delete old index
    if (currentIndex !== aliasName) {
      await this.esService.indices.delete({ index: currentIndex });
    }
    
    console.log(`Reindexed to ${newIndexName}`);
  }

  private async createIndex(name: string) {
    // Create with mappings
    await this.esService.indices.create({
      index: name,
      mappings: { /* ... */ }
    });
  }
}
```

---

## ขั้นตอนที่ 1173: Performance Tuning

```typescript
// Index settings for better performance
const indexSettings = {
  settings: {
    // Number of shards (set based on data size)
    number_of_shards: 3,
    number_of_replicas: 1,
    
    // Refresh interval (default: 1s, increase for better indexing performance)
    refresh_interval: "5s",
    
    // Max result window (default: 10000)
    max_result_window: 50000
  }
};

// Query optimization
async optimizedSearch(query: string) {
  return this.esService.search({
    index: "products",
    // Use filter context for non-scoring queries (better performance)
    query: {
      bool: {
        must: [
          // Scoring query
          { match: { name: query } }
        ],
        filter: [
          // Non-scoring filter (cached)
          { term: { isActive: true } },
          { range: { stock: { gt: 0 } } }
        ]
      }
    },
    // Use _source to limit returned fields
    _source: ["id", "name", "price", "category"],
    // Request cache
    request_cache: true
  });
}

// Bulk operations with controlled batch size
async bulkIndexWithRetry(documents: any[], batchSize: number = 500) {
  for (let i = 0; i < documents.length; i += batchSize) {
    const batch = documents.slice(i, i + batchSize);
    
    let retries = 3;
    while (retries > 0) {
      try {
        await this.bulkIndex(batch);
        break;
      } catch (error) {
        retries--;
        if (retries === 0) throw error;
        await new Promise(r => setTimeout(r, 1000 * (3 - retries)));
      }
    }
  }
}

async bulkIndex(documents: any[]) {
  const operations = documents.flatMap(doc => [
    { index: { _index: "products", _id: doc.id } },
    doc
  ]);
  
  return this.esService.bulk({ operations, refresh: false });
}
```

---

## ขั้นตอนที่ 1174: Search Controller

```typescript
// src/search/search.controller.ts
import {
  Controller, Get, Query, UseGuards,
  ParseIntPipe, DefaultValuePipe
} from "@nestjs/common";
import { ProductsSearchService } from "./products-search.service";
import { SearchAnalyticsService } from "./analytics.service";
import { JwtAuthGuard } from "../auth/guards/jwt-auth.guard";

@Controller("search")
export class SearchController {
  constructor(
    private readonly productsSearch: ProductsSearchService,
    private readonly analytics: SearchAnalyticsService
  ) {}

  @Get("products")
  async searchProducts(
    @Query("q") query: string,
    @Query("page", new DefaultValuePipe(1), ParseIntPipe) page: number,
    @Query("limit", new DefaultValuePipe(20), ParseIntPipe) limit: number,
    @Query("minPrice") minPrice?: string,
    @Query("maxPrice") maxPrice?: string,
    @Query("categories") categories?: string,
    @Query("sortBy") sortBy?: string,
    @Query("sortOrder") sortOrder?: "asc" | "desc"
  ) {
    return this.productsSearch.search({
      query,
      page,
      limit,
      minPrice: minPrice ? parseFloat(minPrice) : undefined,
      maxPrice: maxPrice ? parseFloat(maxPrice) : undefined,
      categories: categories?.split(","),
      sortBy,
      sortOrder
    });
  }

  @Get("autocomplete")
  async autocomplete(@Query("q") query: string) {
    if (!query || query.length < 2) return [];
    return this.productsSearch.autocomplete(query);
  }

  @Get("analytics")
  @UseGuards(JwtAuthGuard)
  async getAnalytics() {
    return this.analytics.getProductAnalytics();
  }

  @Get("facets")
  async getFacets(@Query("q") query: string) {
    return this.analytics.getSearchFacets(query);
  }
}
```

---

## ขั้นตอนที่ 1175: More Aggregation Types

```typescript
// Advanced aggregations
async complexAggregations() {
  return this.esService.search({
    index: "orders",
    size: 0,
    aggs: {
      // Significant terms (unusual terms)
      trending_products: {
        significant_terms: {
          field: "productName.keyword",
          size: 10
        }
      },
      
      // Percentiles
      price_percentiles: {
        percentiles: {
          field: "total",
          percents: [25, 50, 75, 90, 95, 99]
        }
      },
      
      // Moving average (requires date histogram)
      sales_over_time: {
        date_histogram: {
          field: "createdAt",
          calendar_interval: "day"
        },
        aggs: {
          daily_sales: { sum: { field: "total" } },
          moving_avg: {
            moving_avg: {
              buckets_path: "daily_sales",
              window: 7
            }
          }
        }
      },
      
      // Top hits (get actual documents in aggregations)
      top_customers: {
        terms: {
          field: "userId.keyword",
          size: 10,
          order: { total_spent: "desc" }
        },
        aggs: {
          total_spent: { sum: { field: "total" } },
          last_order: {
            top_hits: {
              size: 1,
              sort: [{ createdAt: "desc" }],
              _source: ["userId", "total", "createdAt"]
            }
          }
        }
      }
    }
  });
}
```

---

## ขั้นตอนที่ 1176: Snapshot และ Backup

```typescript
// src/search/backup.service.ts
@Injectable()
export class ElasticsearchBackupService {
  constructor(private readonly esService: ElasticsearchService) {}

  async createSnapshot(snapshotName: string) {
    // Register repository first
    await this.esService.snapshot.createRepository({
      name: "my-backup-repo",
      type: "fs",
      settings: {
        location: "/mount/backups",
        compress: true
      }
    });
    
    // Create snapshot
    await this.esService.snapshot.create({
      repository: "my-backup-repo",
      snapshot: snapshotName,
      wait_for_completion: false,
      indices: ["products", "orders", "users"],
      include_global_state: false
    });
  }

  async restoreSnapshot(snapshotName: string) {
    await this.esService.snapshot.restore({
      repository: "my-backup-repo",
      snapshot: snapshotName,
      wait_for_completion: true,
      indices: ["products"],
      rename_pattern: "(.+)",
      rename_replacement: "$1-restored"
    });
  }
}
```

---

## ขั้นตอนที่ 1177: Monitoring Elasticsearch

```typescript
// src/search/monitoring.service.ts
@Injectable()
export class ElasticsearchMonitoringService {
  constructor(private readonly esService: ElasticsearchService) {}

  async getClusterHealth() {
    const health = await this.esService.cluster.health();
    return {
      status: health.status,
      numberOfNodes: health.number_of_nodes,
      activeShards: health.active_shards,
      unassignedShards: health.unassigned_shards
    };
  }

  async getIndexStats(indexName: string) {
    const stats = await this.esService.indices.stats({ index: indexName });
    const indexStats = stats.indices?.[indexName];
    
    return {
      documentCount: indexStats?.total?.docs?.count,
      deletedCount: indexStats?.total?.docs?.deleted,
      storeSizeBytes: indexStats?.total?.store?.size_in_bytes,
      searchCount: indexStats?.total?.search?.query_total,
      indexingCount: indexStats?.total?.indexing?.index_total
    };
  }

  async getSlowQueries(thresholdMs: number = 1000) {
    // Parse slow query logs from Elasticsearch
    return this.esService.search({
      index: ".logs-deprecation.elasticsearch-default",
      query: {
        range: {
          "event.duration": { gte: thresholdMs * 1000000 } // ns to ms
        }
      },
      sort: [{ "event.duration": "desc" }],
      size: 20
    });
  }
}
```

---

## ขั้นตอนที่ 1178: Search as You Type

```typescript
// Mapping for search-as-you-type
const mapping = {
  mappings: {
    properties: {
      name: {
        type: "search_as_you_type",
        max_shingle_size: 3
      }
    }
  }
};

// Query for search-as-you-type
async searchAsYouType(query: string) {
  return this.esService.search({
    index: "products",
    query: {
      multi_match: {
        query,
        type: "bool_prefix",
        fields: [
          "name",
          "name._2gram",
          "name._3gram"
        ]
      }
    }
  });
}
```

---

## ขั้นตอนที่ 1179: Custom Scoring

```typescript
// Custom relevance scoring
async searchWithCustomScoring(query: string, userId: string) {
  // Get user preferences from separate service
  const userPreferences = await this.getUserPreferences(userId);
  
  return this.esService.search({
    index: "products",
    query: {
      function_score: {
        query: {
          multi_match: {
            query,
            fields: ["name^3", "description"]
          }
        },
        functions: [
          // Boost by rating
          {
            field_value_factor: {
              field: "rating",
              factor: 1.5,
              modifier: "log1p",
              missing: 3
            }
          },
          // Boost by recency
          {
            gauss: {
              createdAt: {
                origin: "now",
                scale: "30d",
                decay: 0.5
              }
            }
          },
          // Boost preferred categories
          {
            filter: { terms: { "category.id": userPreferences.categories } },
            weight: 2
          },
          // Boost in-stock items
          {
            filter: { range: { stock: { gt: 0 } } },
            weight: 1.5
          }
        ],
        score_mode: "sum",
        boost_mode: "multiply"
      }
    }
  });
}

private async getUserPreferences(userId: string): Promise<any> {
  return { categories: [], brands: [] };
}
```

---

## ขั้นตอนที่ 1180: Production Best Practices

```typescript
// Circuit breaker pattern for Elasticsearch
@Injectable()
export class ElasticsearchCircuitBreaker {
  private failures = 0;
  private readonly maxFailures = 5;
  private readonly timeout = 60000; // 1 minute
  private isOpen = false;
  private lastFailureTime?: number;

  constructor(private readonly esService: ElasticsearchService) {}

  async execute<T>(operation: () => Promise<T>): Promise<T | null> {
    if (this.isOpen) {
      const elapsed = Date.now() - (this.lastFailureTime ?? 0);
      if (elapsed < this.timeout) {
        console.warn("Circuit breaker is OPEN, returning null");
        return null;
      }
      this.isOpen = false;
      this.failures = 0;
    }
    
    try {
      const result = await operation();
      this.failures = 0;
      return result;
    } catch (error) {
      this.failures++;
      this.lastFailureTime = Date.now();
      
      if (this.failures >= this.maxFailures) {
        this.isOpen = true;
        console.error("Circuit breaker OPENED after too many failures");
      }
      
      throw error;
    }
  }
}
```

---

## 🏋️ แบบฝึกหัด

### แบบฝึกหัดที่ 1: Product Search Engine
1. สร้าง Elasticsearch index สำหรับ products
2. implement full-text search ด้วย highlighting
3. เพิ่ม faceted search (categories, price ranges, ratings)

### แบบฝึกหัดที่ 2: Log Analytics Dashboard
1. สร้าง log ingestion pipeline
2. implement aggregations สำหรับ error analysis
3. สร้าง alerting สำหรับ error spikes

### แบบฝึกหัดที่ 3: Data Sync
1. implement real-time sync จาก PostgreSQL ไปยัง Elasticsearch
2. ใช้ Change Data Capture (CDC)
3. handle failed sync attempts

---

## 📚 สรุป

ใน Part นี้เราได้เรียนรู้:
- Elasticsearch concepts: Index, Document, Mapping
- Full-text search, Autocomplete, Suggest
- Aggregations สำหรับ analytics
- Data synchronization strategies
- Custom scoring
- Performance optimization
- Production best practices

**Part ถัดไป**: Apache Kafka
