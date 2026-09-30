# Part 47: Search & Filtering (การค้นหาและกรองข้อมูล)
## ขั้นตอนที่ 47-47 จาก 1000

---

## บทนำ

การค้นหาและกรองข้อมูลเป็นฟีเจอร์หลักของ API ทุกตัว บทนี้จะครอบคลุมตั้งแต่การค้นหาพื้นฐานไปจนถึง Full-text Search และ Elasticsearch

---

## 47.1 Text Search พื้นฐาน

### LIKE Search ใน SQL

```javascript
// src/controllers/searchController.js
const { Op } = require('sequelize');
const Product = require('../models/Product');

// Basic LIKE search
const searchProducts = async (req, res) => {
  try {
    const { q } = req.query;
    
    if (!q || q.trim().length < 2) {
      return res.status(400).json({
        success: false,
        message: 'กรุณาระบุคำค้นหาอย่างน้อย 2 ตัวอักษร'
      });
    }
    
    const searchTerm = q.trim();
    
    const products = await Product.findAll({
      where: {
        [Op.or]: [
          { name: { [Op.iLike]: `%${searchTerm}%` } },
          { description: { [Op.iLike]: `%${searchTerm}%` } },
          { sku: { [Op.iLike]: `%${searchTerm}%` } }
        ]
      },
      order: [['name', 'ASC']],
      limit: 50
    });
    
    res.json({
      success: true,
      query: searchTerm,
      count: products.length,
      data: products
    });
  } catch (error) {
    res.status(500).json({ success: false, message: error.message });
  }
};

module.exports = { searchProducts };
```

### Multi-field Search ที่ชาญฉลาดกว่า

```javascript
// src/utils/searchBuilder.js
const { Op, literal, fn, col } = require('sequelize');

/**
 * สร้าง search condition จาก query string
 * รองรับ: exact match, partial match, multi-word
 */
const buildSearchCondition = (searchTerm, fields, options = {}) => {
  const { caseSensitive = false, matchType = 'partial' } = options;
  
  // แยกคำหลายคำด้วย space
  const words = searchTerm.trim().split(/\s+/).filter(w => w.length > 0);
  
  if (words.length === 0) return null;
  
  const operator = caseSensitive ? Op.like : Op.iLike;
  
  if (matchType === 'exact') {
    // ต้องตรงทุกคำ
    return {
      [Op.and]: words.map(word => ({
        [Op.or]: fields.map(field => ({
          [field]: { [operator]: `%${word}%` }
        }))
      }))
    };
  } else {
    // ตรงแค่บางคำก็ได้ (OR logic)
    return {
      [Op.or]: words.flatMap(word =>
        fields.map(field => ({
          [field]: { [operator]: `%${word}%` }
        }))
      )
    };
  }
};

/**
 * คำนวณ relevance score (จำลอง)
 */
const addRelevanceScore = (item, searchTerm, fields) => {
  const term = searchTerm.toLowerCase();
  let score = 0;
  
  fields.forEach((field, index) => {
    const value = (item[field] || '').toLowerCase();
    const weight = fields.length - index; // field แรกมี weight สูงสุด
    
    if (value === term) score += 100 * weight;         // exact match
    else if (value.startsWith(term)) score += 50 * weight;  // prefix match
    else if (value.includes(term)) score += 20 * weight;    // contains
    
    // นับจำนวนครั้งที่ปรากฏ
    const occurrences = (value.match(new RegExp(term, 'g')) || []).length;
    score += occurrences * 5 * weight;
  });
  
  return score;
};

module.exports = { buildSearchCondition, addRelevanceScore };
```

---

## 47.2 Full-Text Search ใน PostgreSQL

PostgreSQL มี built-in full-text search ที่ทรงพลังมาก

### ตั้งค่า Full-Text Search

```sql
-- สร้าง tsvector column
ALTER TABLE products ADD COLUMN search_vector tsvector;

-- อัพเดท search_vector จาก name และ description
UPDATE products SET search_vector = 
  setweight(to_tsvector('english', coalesce(name, '')), 'A') ||
  setweight(to_tsvector('english', coalesce(description, '')), 'B') ||
  setweight(to_tsvector('english', coalesce(tags::text, '')), 'C');

-- สร้าง index สำหรับ full-text search
CREATE INDEX idx_products_search ON products USING GIN(search_vector);

-- สร้าง trigger เพื่ออัพเดทอัตโนมัติ
CREATE FUNCTION update_search_vector() RETURNS trigger AS $$
BEGIN
  NEW.search_vector := 
    setweight(to_tsvector('english', coalesce(NEW.name, '')), 'A') ||
    setweight(to_tsvector('english', coalesce(NEW.description, '')), 'B');
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER products_search_vector_update
  BEFORE INSERT OR UPDATE ON products
  FOR EACH ROW EXECUTE FUNCTION update_search_vector();
```

### Full-Text Search ด้วย Sequelize

```javascript
// src/utils/fullTextSearch.js
const { sequelize } = require('../config/database');
const { QueryTypes, literal, fn, col } = require('sequelize');

/**
 * Full-text search ด้วย PostgreSQL tsvector
 */
const fullTextSearch = async (Model, searchQuery, options = {}) => {
  const {
    language = 'english',
    limit = 20,
    offset = 0,
    fields = ['name', 'description'],
    additionalWhere = {}
  } = options;
  
  // สร้าง tsquery จาก search terms
  const tsQuery = searchQuery
    .trim()
    .split(/\s+/)
    .filter(w => w.length > 0)
    .map(w => `${w}:*`) // prefix search
    .join(' & ');
  
  if (!tsQuery) return { rows: [], count: 0 };
  
  // Raw query สำหรับ full-text search พร้อม ranking
  const results = await sequelize.query(`
    SELECT 
      *,
      ts_rank(search_vector, plainto_tsquery(:language, :query)) AS rank,
      ts_headline(
        :language, 
        name, 
        plainto_tsquery(:language, :query),
        'StartSel=<mark>, StopSel=</mark>, MaxWords=50, MinWords=5'
      ) AS name_highlight,
      ts_headline(
        :language,
        description,
        plainto_tsquery(:language, :query),
        'StartSel=<mark>, StopSel=</mark>, MaxWords=100, MinWords=10'
      ) AS description_highlight
    FROM products
    WHERE search_vector @@ plainto_tsquery(:language, :query)
    ORDER BY rank DESC
    LIMIT :limit OFFSET :offset
  `, {
    replacements: {
      language,
      query: searchQuery,
      limit,
      offset
    },
    type: QueryTypes.SELECT
  });
  
  const countResult = await sequelize.query(`
    SELECT COUNT(*) FROM products
    WHERE search_vector @@ plainto_tsquery(:language, :query)
  `, {
    replacements: { language, query: searchQuery },
    type: QueryTypes.SELECT
  });
  
  return {
    rows: results,
    count: parseInt(countResult[0].count)
  };
};

module.exports = { fullTextSearch };
```

### Controller สำหรับ Full-Text Search

```javascript
// src/controllers/fullTextSearchController.js
const { fullTextSearch } = require('../utils/fullTextSearch');

const searchWithFullText = async (req, res) => {
  try {
    const {
      q,
      page = 1,
      limit = 20,
      language = 'english'
    } = req.query;
    
    if (!q || q.trim().length < 2) {
      return res.status(400).json({
        success: false,
        message: 'Search query must be at least 2 characters'
      });
    }
    
    const offset = (parseInt(page) - 1) * parseInt(limit);
    
    const startTime = Date.now();
    const { rows, count } = await fullTextSearch(null, q, {
      language,
      limit: parseInt(limit),
      offset
    });
    const searchTime = Date.now() - startTime;
    
    res.json({
      success: true,
      query: q,
      results: {
        total: count,
        data: rows,
        searchTimeMs: searchTime
      },
      pagination: {
        currentPage: parseInt(page),
        totalPages: Math.ceil(count / parseInt(limit)),
        itemsPerPage: parseInt(limit)
      }
    });
  } catch (error) {
    res.status(500).json({ success: false, message: error.message });
  }
};

module.exports = { searchWithFullText };
```

---

## 47.3 Filter Patterns (รูปแบบการกรองข้อมูล)

### Filter Builder Pattern

```javascript
// src/utils/filterBuilder.js
const { Op } = require('sequelize');

/**
 * Filter Builder - สร้าง Sequelize where clause จาก query params
 */
class FilterBuilder {
  constructor() {
    this.conditions = {};
  }
  
  /**
   * กรองด้วย exact value
   */
  exact(field, value) {
    if (value !== undefined && value !== null && value !== '') {
      this.conditions[field] = value;
    }
    return this;
  }
  
  /**
   * กรองด้วย array of values (IN)
   */
  in(field, values) {
    if (values && values.length > 0) {
      const valueArray = Array.isArray(values) 
        ? values 
        : values.split(',').map(v => v.trim());
      this.conditions[field] = { [Op.in]: valueArray };
    }
    return this;
  }
  
  /**
   * กรองด้วย range (BETWEEN)
   */
  range(field, min, max) {
    if (min !== undefined || max !== undefined) {
      this.conditions[field] = {};
      if (min !== undefined && min !== '') {
        this.conditions[field][Op.gte] = min;
      }
      if (max !== undefined && max !== '') {
        this.conditions[field][Op.lte] = max;
      }
    }
    return this;
  }
  
  /**
   * กรองด้วย date range
   */
  dateRange(field, startDate, endDate) {
    if (startDate || endDate) {
      this.conditions[field] = {};
      if (startDate) {
        this.conditions[field][Op.gte] = new Date(startDate);
      }
      if (endDate) {
        // เพิ่ม 1 วันเพื่อให้รวมวันสุดท้ายด้วย
        const end = new Date(endDate);
        end.setDate(end.getDate() + 1);
        this.conditions[field][Op.lt] = end;
      }
    }
    return this;
  }
  
  /**
   * กรองด้วย text search (LIKE)
   */
  like(field, value, position = 'both') {
    if (value && value.trim()) {
      let pattern;
      switch (position) {
        case 'start': pattern = `${value.trim()}%`; break;
        case 'end': pattern = `%${value.trim()}`; break;
        default: pattern = `%${value.trim()}%`;
      }
      this.conditions[field] = { [Op.iLike]: pattern };
    }
    return this;
  }
  
  /**
   * กรองด้วย boolean
   */
  boolean(field, value) {
    if (value !== undefined && value !== '') {
      this.conditions[field] = value === 'true' || value === true;
    }
    return this;
  }
  
  /**
   * เพิ่ม custom condition
   */
  custom(condition) {
    Object.assign(this.conditions, condition);
    return this;
  }
  
  /**
   * Build และ return where clause
   */
  build() {
    return this.conditions;
  }
}

module.exports = FilterBuilder;
```

### ตัวอย่างใช้งาน Filter Builder

```javascript
// src/controllers/productController.js
const { Op } = require('sequelize');
const Product = require('../models/Product');
const FilterBuilder = require('../utils/filterBuilder');

const getFilteredProducts = async (req, res) => {
  try {
    const {
      // Text search
      search,
      name,
      
      // Exact match
      category,
      status,
      brand,
      
      // Range filters
      minPrice,
      maxPrice,
      minStock,
      maxStock,
      
      // Date range
      createdFrom,
      createdTo,
      
      // Boolean
      inStock,
      featured,
      
      // Sorting
      sortBy = 'createdAt',
      sortOrder = 'DESC',
      
      // Pagination
      page = 1,
      limit = 20
    } = req.query;
    
    // Build filters
    const where = new FilterBuilder()
      .like('name', search)  // search ใน name
      .exact('category', category)
      .exact('status', status)
      .exact('brand', brand)
      .range('price', minPrice ? parseFloat(minPrice) : undefined, 
                      maxPrice ? parseFloat(maxPrice) : undefined)
      .range('stock', minStock ? parseInt(minStock) : undefined,
                      maxStock ? parseInt(maxStock) : undefined)
      .dateRange('createdAt', createdFrom, createdTo)
      .boolean('inStock', inStock)
      .boolean('featured', featured)
      .build();
    
    // Validate sort fields
    const allowedSortFields = ['name', 'price', 'stock', 'createdAt', 'rating'];
    const finalSort = allowedSortFields.includes(sortBy) ? sortBy : 'createdAt';
    const finalOrder = sortOrder.toUpperCase() === 'ASC' ? 'ASC' : 'DESC';
    
    const pageNum = Math.max(1, parseInt(page));
    const limitNum = Math.min(100, Math.max(1, parseInt(limit)));
    
    const { count, rows } = await Product.findAndCountAll({
      where,
      limit: limitNum,
      offset: (pageNum - 1) * limitNum,
      order: [[finalSort, finalOrder]]
    });
    
    res.json({
      success: true,
      data: rows,
      meta: {
        total: count,
        page: pageNum,
        limit: limitNum,
        totalPages: Math.ceil(count / limitNum),
        appliedFilters: Object.keys(req.query).filter(k => 
          !['page', 'limit', 'sortBy', 'sortOrder'].includes(k) && req.query[k]
        )
      }
    });
  } catch (error) {
    res.status(500).json({ success: false, message: error.message });
  }
};

module.exports = { getFilteredProducts };
```

---

## 47.4 Advanced Sorting

```javascript
// src/utils/sortBuilder.js

/**
 * สร้าง Sequelize order clause จาก query params
 * รองรับ: single sort, multi-column sort
 */
const buildSortClause = (sortParam, allowedFields, defaultSort = [['createdAt', 'DESC']]) => {
  if (!sortParam) return defaultSort;
  
  // รองรับ format: "price:asc,name:desc" หรือ "-price,name"
  const sortParts = sortParam.split(',').map(part => {
    part = part.trim();
    
    // Format: "-field" หรือ "+field"
    if (part.startsWith('-')) {
      return [part.slice(1), 'DESC'];
    }
    if (part.startsWith('+')) {
      return [part.slice(1), 'ASC'];
    }
    
    // Format: "field:direction"
    const [field, direction = 'ASC'] = part.split(':');
    return [field, direction.toUpperCase()];
  });
  
  // Filter เฉพาะ allowed fields
  const validSort = sortParts.filter(([field]) => 
    allowedFields.includes(field)
  );
  
  return validSort.length > 0 ? validSort : defaultSort;
};

module.exports = { buildSortClause };
```

### ตัวอย่างการใช้ Multi-column Sort

```javascript
// การใช้งาน
const { buildSortClause } = require('../utils/sortBuilder');

// GET /products?sort=-price,name
// → ORDER BY price DESC, name ASC

// GET /products?sort=category:asc,price:desc
// → ORDER BY category ASC, price DESC

const order = buildSortClause(
  req.query.sort,
  ['name', 'price', 'stock', 'createdAt', 'rating'],
  [['createdAt', 'DESC']]
);
```

---

## 47.5 Elasticsearch Integration

### ติดตั้ง Elasticsearch

```bash
# ติดตั้ง Docker image
docker pull elasticsearch:8.11.0
docker run -d \
  --name elasticsearch \
  -p 9200:9200 \
  -e "discovery.type=single-node" \
  -e "xpack.security.enabled=false" \
  elasticsearch:8.11.0

# ติดตั้ง client library
npm install @elastic/elasticsearch
```

### ตั้งค่า Elasticsearch Client

```javascript
// src/config/elasticsearch.js
const { Client } = require('@elastic/elasticsearch');

const client = new Client({
  node: process.env.ELASTICSEARCH_URL || 'http://localhost:9200',
  maxRetries: 5,
  requestTimeout: 60000,
  sniffOnStart: false
});

// ทดสอบ connection
const testConnection = async () => {
  try {
    const info = await client.info();
    console.log('✅ Elasticsearch connected:', info.version.number);
  } catch (error) {
    console.error('❌ Elasticsearch connection failed:', error.message);
  }
};

module.exports = { client, testConnection };
```

### สร้าง Index และ Mapping

```javascript
// src/elasticsearch/productIndex.js
const { client } = require('../config/elasticsearch');

const INDEX_NAME = 'products';

/**
 * สร้าง index พร้อม mapping
 */
const createProductIndex = async () => {
  const exists = await client.indices.exists({ index: INDEX_NAME });
  
  if (!exists) {
    await client.indices.create({
      index: INDEX_NAME,
      body: {
        settings: {
          number_of_shards: 1,
          number_of_replicas: 0,
          analysis: {
            analyzer: {
              thai_analyzer: {
                type: 'custom',
                tokenizer: 'standard',
                filter: ['lowercase', 'stop']
              }
            }
          }
        },
        mappings: {
          properties: {
            id: { type: 'integer' },
            name: {
              type: 'text',
              analyzer: 'standard',
              fields: {
                keyword: { type: 'keyword' },
                suggest: { type: 'completion' }
              }
            },
            description: {
              type: 'text',
              analyzer: 'standard'
            },
            category: { type: 'keyword' },
            brand: { type: 'keyword' },
            price: { type: 'float' },
            stock: { type: 'integer' },
            rating: { type: 'float' },
            tags: { type: 'keyword' },
            status: { type: 'keyword' },
            createdAt: { type: 'date' }
          }
        }
      }
    });
    
    console.log(`✅ Index '${INDEX_NAME}' created`);
  }
};

/**
 * Index product document
 */
const indexProduct = async (product) => {
  return await client.index({
    index: INDEX_NAME,
    id: product.id.toString(),
    document: {
      id: product.id,
      name: product.name,
      description: product.description,
      category: product.category,
      brand: product.brand,
      price: product.price,
      stock: product.stock,
      rating: product.rating,
      tags: product.tags || [],
      status: product.status,
      createdAt: product.createdAt
    }
  });
};

/**
 * Bulk index products
 */
const bulkIndexProducts = async (products) => {
  const operations = products.flatMap(product => [
    { index: { _index: INDEX_NAME, _id: product.id.toString() } },
    {
      id: product.id,
      name: product.name,
      description: product.description,
      category: product.category,
      brand: product.brand,
      price: product.price,
      stock: product.stock,
      rating: product.rating,
      tags: product.tags || [],
      status: product.status,
      createdAt: product.createdAt
    }
  ]);
  
  const result = await client.bulk({ operations });
  
  if (result.errors) {
    const errorItems = result.items.filter(item => item.index?.error);
    console.error('Bulk indexing errors:', errorItems);
  }
  
  return {
    total: products.length,
    successful: result.items.filter(item => !item.index?.error).length,
    failed: result.items.filter(item => item.index?.error).length
  };
};

module.exports = { createProductIndex, indexProduct, bulkIndexProducts, INDEX_NAME };
```

### Elasticsearch Search Service

```javascript
// src/services/elasticsearchService.js
const { client } = require('../config/elasticsearch');
const { INDEX_NAME } = require('../elasticsearch/productIndex');

/**
 * Multi-match search ใน Elasticsearch
 */
const searchProducts = async (searchParams) => {
  const {
    query,
    category,
    brand,
    minPrice,
    maxPrice,
    minRating,
    sortBy = '_score',
    sortOrder = 'desc',
    page = 1,
    limit = 20,
    fuzzy = true
  } = searchParams;
  
  const from = (parseInt(page) - 1) * parseInt(limit);
  
  // สร้าง query
  const esQuery = {
    bool: {
      must: [],
      filter: []
    }
  };
  
  // Full-text search
  if (query && query.trim()) {
    if (fuzzy) {
      esQuery.bool.must.push({
        multi_match: {
          query: query.trim(),
          fields: ['name^3', 'description^1', 'category^2', 'brand^2', 'tags^1'],
          fuzziness: 'AUTO',
          type: 'best_fields'
        }
      });
    } else {
      esQuery.bool.must.push({
        multi_match: {
          query: query.trim(),
          fields: ['name^3', 'description^1', 'category^2', 'brand^2'],
          type: 'phrase_prefix'
        }
      });
    }
  } else {
    esQuery.bool.must.push({ match_all: {} });
  }
  
  // Filters
  if (category) {
    esQuery.bool.filter.push({ term: { category } });
  }
  
  if (brand) {
    esQuery.bool.filter.push({ term: { brand } });
  }
  
  if (minPrice || maxPrice) {
    const priceRange = {};
    if (minPrice) priceRange.gte = parseFloat(minPrice);
    if (maxPrice) priceRange.lte = parseFloat(maxPrice);
    esQuery.bool.filter.push({ range: { price: priceRange } });
  }
  
  if (minRating) {
    esQuery.bool.filter.push({ range: { rating: { gte: parseFloat(minRating) } } });
  }
  
  // Execute search
  const result = await client.search({
    index: INDEX_NAME,
    body: {
      query: esQuery,
      sort: [
        sortBy === '_score' ? { _score: sortOrder } : { [sortBy]: { order: sortOrder } }
      ],
      from,
      size: parseInt(limit),
      highlight: {
        fields: {
          name: { pre_tags: ['<mark>'], post_tags: ['</mark>'] },
          description: { 
            pre_tags: ['<mark>'], 
            post_tags: ['</mark>'],
            number_of_fragments: 3,
            fragment_size: 150
          }
        }
      },
      aggs: {
        categories: { terms: { field: 'category', size: 20 } },
        brands: { terms: { field: 'brand', size: 20 } },
        price_stats: { stats: { field: 'price' } },
        price_ranges: {
          range: {
            field: 'price',
            ranges: [
              { key: 'under_500', to: 500 },
              { key: '500_1000', from: 500, to: 1000 },
              { key: '1000_5000', from: 1000, to: 5000 },
              { key: 'over_5000', from: 5000 }
            ]
          }
        }
      }
    }
  });
  
  return {
    hits: result.hits.hits.map(hit => ({
      ...hit._source,
      _score: hit._score,
      highlights: hit.highlight || {}
    })),
    total: result.hits.total.value,
    aggregations: {
      categories: result.aggregations?.categories?.buckets || [],
      brands: result.aggregations?.brands?.buckets || [],
      priceStats: result.aggregations?.price_stats,
      priceRanges: result.aggregations?.price_ranges?.buckets || []
    }
  };
};

/**
 * Auto-suggest / Autocomplete
 */
const getSuggestions = async (prefix) => {
  const result = await client.search({
    index: INDEX_NAME,
    body: {
      suggest: {
        product_suggest: {
          prefix,
          completion: {
            field: 'name.suggest',
            size: 10,
            skip_duplicates: true
          }
        }
      }
    }
  });
  
  return result.suggest.product_suggest[0].options.map(opt => ({
    text: opt.text,
    score: opt._score
  }));
};

module.exports = { searchProducts, getSuggestions };
```

### Controller สำหรับ Elasticsearch

```javascript
// src/controllers/esSearchController.js
const { searchProducts, getSuggestions } = require('../services/elasticsearchService');

const search = async (req, res) => {
  try {
    const { q, ...filters } = req.query;
    
    const results = await searchProducts({
      query: q,
      ...filters
    });
    
    res.json({
      success: true,
      query: q,
      total: results.total,
      data: results.hits,
      facets: results.aggregations,
      pagination: {
        page: parseInt(req.query.page) || 1,
        limit: parseInt(req.query.limit) || 20,
        totalPages: Math.ceil(results.total / (parseInt(req.query.limit) || 20))
      }
    });
  } catch (error) {
    res.status(500).json({ success: false, message: error.message });
  }
};

const autocomplete = async (req, res) => {
  try {
    const { q } = req.query;
    
    if (!q || q.length < 2) {
      return res.json({ success: true, suggestions: [] });
    }
    
    const suggestions = await getSuggestions(q);
    res.json({ success: true, suggestions });
  } catch (error) {
    res.status(500).json({ success: false, message: error.message });
  }
};

module.exports = { search, autocomplete };
```

---

## 47.6 Sync Database กับ Elasticsearch

```javascript
// src/services/syncService.js
const Product = require('../models/Product');
const { indexProduct, bulkIndexProducts } = require('../elasticsearch/productIndex');

/**
 * Sync ข้อมูลทั้งหมดจาก DB ไป Elasticsearch
 */
const syncAllProducts = async () => {
  console.log('Starting full sync...');
  
  const batchSize = 100;
  let offset = 0;
  let totalSynced = 0;
  
  while (true) {
    const products = await Product.findAll({
      limit: batchSize,
      offset,
      raw: true
    });
    
    if (products.length === 0) break;
    
    const result = await bulkIndexProducts(products);
    totalSynced += result.successful;
    offset += batchSize;
    
    console.log(`Synced ${totalSynced} products...`);
  }
  
  console.log(`✅ Sync complete. Total: ${totalSynced} products`);
  return totalSynced;
};

/**
 * Hook สำหรับ sync เมื่อมีการ create/update
 */
const setupSyncHooks = (Model) => {
  Model.afterCreate(async (instance) => {
    try {
      await indexProduct(instance.toJSON());
    } catch (error) {
      console.error('ES index error on create:', error.message);
    }
  });
  
  Model.afterUpdate(async (instance) => {
    try {
      await indexProduct(instance.toJSON());
    } catch (error) {
      console.error('ES index error on update:', error.message);
    }
  });
  
  Model.afterDestroy(async (instance) => {
    try {
      const { client } = require('../config/elasticsearch');
      const { INDEX_NAME } = require('../elasticsearch/productIndex');
      await client.delete({
        index: INDEX_NAME,
        id: instance.id.toString()
      });
    } catch (error) {
      console.error('ES delete error:', error.message);
    }
  });
};

module.exports = { syncAllProducts, setupSyncHooks };
```

---

## แบบฝึกหัดที่ 47

### แบบฝึกหัดพื้นฐาน

**1. สร้าง Search API พื้นฐาน**

สร้าง API ค้นหาบทความ (Articles) รองรับ:
- ค้นหาจาก title, content, tags
- กรองด้วย category, author, date range
- Sort by: date, views, likes
- Pagination

**2. PostgreSQL Full-Text Search**

ปรับแต่ง search จากข้อ 1 ให้ใช้ PostgreSQL full-text search:
- ตั้งค่า tsvector column
- สร้าง trigger สำหรับ auto-update
- Highlight matching text ใน response

### แบบฝึกหัดขั้นสูง

**3. Elasticsearch Product Search**

สร้าง e-commerce search ด้วย Elasticsearch:
- Multi-field search พร้อม boosting
- Faceted search (category, price range, brand)
- Fuzzy search สำหรับ typo tolerance
- Autocomplete suggestions

**4. Search Analytics**

เก็บ log การค้นหาและวิเคราะห์:
- บันทึก search queries ทั้งหมด
- นับจำนวนครั้งที่ค้นหา
- ระบุ zero-result searches
- สร้าง API สำหรับ trending searches

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **Text Search พื้นฐาน** - LIKE query, multi-field search
2. **PostgreSQL Full-Text Search** - tsvector, tsquery, ranking
3. **Filter Builder Pattern** - สร้าง flexible filter system
4. **Advanced Sorting** - multi-column sort, custom sort
5. **Elasticsearch** - index, mapping, full-text search, facets

| วิธีค้นหา | ใช้เมื่อ |
|-----------|---------|
| LIKE | ข้อมูลน้อย, ค้นหาพื้นฐาน |
| PostgreSQL FTS | ข้อมูลมาก, ต้องการ ranking |
| Elasticsearch | ข้อมูลมาก, ต้องการ facets, analytics |

---

**ถัดไป:** Part 48 - Image Processing (การประมวลผลรูปภาพ)
