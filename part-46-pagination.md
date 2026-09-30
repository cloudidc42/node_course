# Part 46: Pagination (การแบ่งหน้าข้อมูล)
## ขั้นตอนที่ 46-46 จาก 1000

---

## บทนำ

Pagination หรือการแบ่งหน้าข้อมูล เป็นเทคนิคสำคัญในการพัฒนา API ที่ช่วยให้เราสามารถจัดการกับข้อมูลจำนวนมากได้อย่างมีประสิทธิภาพ แทนที่จะดึงข้อมูลทั้งหมดในครั้งเดียว เราแบ่งข้อมูลออกเป็นส่วนย่อย ๆ

---

## 46.1 ทำไมต้องใช้ Pagination?

### ปัญหาของการดึงข้อมูลทั้งหมด

```javascript
// ❌ ไม่ดี - ดึงข้อมูลทั้งหมดในครั้งเดียว
app.get('/users', async (req, res) => {
  const users = await User.findAll(); // ถ้ามี 1 ล้าน records จะเกิดปัญหา!
  res.json(users);
});
```

ปัญหาที่เกิดขึ้น:
- **Memory usage สูง** - โหลดข้อมูลทั้งหมดเข้า RAM
- **Response time ช้า** - ต้องรอข้อมูลทั้งหมดก่อนส่ง
- **Network bandwidth สูง** - ส่งข้อมูลมากเกินความจำเป็น
- **User experience แย่** - ผู้ใช้ต้องรอนาน

### วิธีแก้ไข - ใช้ Pagination

```javascript
// ✅ ดี - ดึงข้อมูลทีละหน้า
app.get('/users', async (req, res) => {
  const page = parseInt(req.query.page) || 1;
  const limit = parseInt(req.query.limit) || 10;
  
  const users = await User.findAll({
    limit: limit,
    offset: (page - 1) * limit
  });
  
  res.json(users);
});
```

---

## 46.2 Offset Pagination (การแบ่งหน้าแบบ Offset)

Offset Pagination เป็นวิธีที่นิยมใช้มากที่สุด ง่ายต่อการเข้าใจและนำไปใช้

### หลักการทำงาน

```
หน้า 1: OFFSET 0, LIMIT 10  → records 1-10
หน้า 2: OFFSET 10, LIMIT 10 → records 11-20
หน้า 3: OFFSET 20, LIMIT 10 → records 21-30
```

### การติดตั้งและตั้งค่าโปรเจค

```bash
mkdir pagination-demo
cd pagination-demo
npm init -y
npm install express sequelize pg pg-hstore dotenv
```

### โครงสร้างโปรเจค

```
pagination-demo/
├── src/
│   ├── controllers/
│   │   └── userController.js
│   ├── models/
│   │   └── User.js
│   ├── middleware/
│   │   └── pagination.js
│   ├── utils/
│   │   └── paginationHelper.js
│   └── app.js
├── .env
└── package.json
```

### 1. สร้าง Model

```javascript
// src/models/User.js
const { DataTypes } = require('sequelize');
const sequelize = require('../config/database');

const User = sequelize.define('User', {
  id: {
    type: DataTypes.INTEGER,
    primaryKey: true,
    autoIncrement: true
  },
  name: {
    type: DataTypes.STRING,
    allowNull: false
  },
  email: {
    type: DataTypes.STRING,
    unique: true,
    allowNull: false
  },
  age: {
    type: DataTypes.INTEGER
  },
  createdAt: {
    type: DataTypes.DATE,
    field: 'created_at'
  }
}, {
  tableName: 'users',
  timestamps: true
});

module.exports = User;
```

### 2. Pagination Middleware

```javascript
// src/middleware/pagination.js
const pagination = (req, res, next) => {
  // ดึงค่าจาก query string
  const page = Math.max(1, parseInt(req.query.page) || 1);
  const limit = Math.min(100, Math.max(1, parseInt(req.query.limit) || 10));
  const offset = (page - 1) * limit;
  
  // เพิ่มค่าเข้า request object
  req.pagination = {
    page,
    limit,
    offset
  };
  
  next();
};

module.exports = pagination;
```

### 3. Pagination Helper

```javascript
// src/utils/paginationHelper.js

/**
 * สร้าง pagination metadata
 * @param {number} total - จำนวนข้อมูลทั้งหมด
 * @param {number} page - หน้าปัจจุบัน
 * @param {number} limit - จำนวนข้อมูลต่อหน้า
 * @param {string} baseUrl - URL พื้นฐาน
 */
const createPaginationMeta = (total, page, limit, baseUrl) => {
  const totalPages = Math.ceil(total / limit);
  const hasNextPage = page < totalPages;
  const hasPrevPage = page > 1;
  
  return {
    pagination: {
      currentPage: page,
      totalPages,
      totalItems: total,
      itemsPerPage: limit,
      hasNextPage,
      hasPrevPage,
    },
    links: {
      self: `${baseUrl}?page=${page}&limit=${limit}`,
      first: `${baseUrl}?page=1&limit=${limit}`,
      last: `${baseUrl}?page=${totalPages}&limit=${limit}`,
      next: hasNextPage ? `${baseUrl}?page=${page + 1}&limit=${limit}` : null,
      prev: hasPrevPage ? `${baseUrl}?page=${page - 1}&limit=${limit}` : null,
    }
  };
};

module.exports = { createPaginationMeta };
```

### 4. Controller

```javascript
// src/controllers/userController.js
const User = require('../models/User');
const { createPaginationMeta } = require('../utils/paginationHelper');

const getUsers = async (req, res) => {
  try {
    const { page, limit, offset } = req.pagination;
    
    // ดึงข้อมูลและนับจำนวนทั้งหมดพร้อมกัน
    const { count, rows } = await User.findAndCountAll({
      limit,
      offset,
      order: [['createdAt', 'DESC']]
    });
    
    const baseUrl = `${req.protocol}://${req.get('host')}${req.path}`;
    const meta = createPaginationMeta(count, page, limit, baseUrl);
    
    res.json({
      success: true,
      data: rows,
      ...meta
    });
  } catch (error) {
    res.status(500).json({
      success: false,
      message: error.message
    });
  }
};

module.exports = { getUsers };
```

### 5. Routes

```javascript
// src/routes/userRoutes.js
const express = require('express');
const router = express.Router();
const { getUsers } = require('../controllers/userController');
const pagination = require('../middleware/pagination');

router.get('/', pagination, getUsers);

module.exports = router;
```

### ตัวอย่าง Response

```json
{
  "success": true,
  "data": [
    { "id": 1, "name": "สมชาย ใจดี", "email": "somchai@example.com" },
    { "id": 2, "name": "สมหญิง รักดี", "email": "somying@example.com" }
  ],
  "pagination": {
    "currentPage": 1,
    "totalPages": 50,
    "totalItems": 500,
    "itemsPerPage": 10,
    "hasNextPage": true,
    "hasPrevPage": false
  },
  "links": {
    "self": "http://localhost:3000/api/users?page=1&limit=10",
    "first": "http://localhost:3000/api/users?page=1&limit=10",
    "last": "http://localhost:3000/api/users?page=50&limit=10",
    "next": "http://localhost:3000/api/users?page=2&limit=10",
    "prev": null
  }
}
```

### ข้อดีและข้อเสียของ Offset Pagination

**ข้อดี:**
- ง่ายต่อการเข้าใจและนำไปใช้
- สามารถกระโดดไปหน้าใดก็ได้โดยตรง
- รองรับการแสดงผลแบบ "หน้า X จาก Y"

**ข้อเสีย:**
- ประสิทธิภาพลดลงเมื่อ offset สูง (ต้องสแกนข้อมูลจำนวนมาก)
- ข้อมูลอาจเลื่อนไปเมื่อมีการเพิ่ม/ลบข้อมูล (Data shifting)

---

## 46.3 Cursor Pagination (การแบ่งหน้าแบบ Cursor)

Cursor Pagination แก้ปัญหาของ Offset Pagination โดยใช้ cursor (ตัวชี้) แทนการใช้ offset

### หลักการทำงาน

```
ดึงหน้าแรก: SELECT * FROM users ORDER BY id LIMIT 10
cursor = id ของ record สุดท้าย (เช่น id=10)

ดึงหน้าถัดไป: SELECT * FROM users WHERE id > 10 ORDER BY id LIMIT 10
cursor = id ของ record สุดท้ายใหม่ (เช่น id=20)
```

### Cursor Pagination Implementation

```javascript
// src/utils/cursorPagination.js
const { Op } = require('sequelize');

/**
 * Encode cursor เป็น base64
 */
const encodeCursor = (data) => {
  return Buffer.from(JSON.stringify(data)).toString('base64');
};

/**
 * Decode cursor จาก base64
 */
const decodeCursor = (cursor) => {
  try {
    return JSON.parse(Buffer.from(cursor, 'base64').toString('utf8'));
  } catch {
    return null;
  }
};

/**
 * ดึงข้อมูลแบบ cursor pagination
 */
const getCursorPage = async (Model, options = {}) => {
  const {
    cursor,
    limit = 10,
    orderField = 'id',
    orderDirection = 'ASC',
    where = {},
    include = []
  } = options;
  
  let whereClause = { ...where };
  
  // ถ้ามี cursor ให้เพิ่ม condition
  if (cursor) {
    const decodedCursor = decodeCursor(cursor);
    if (decodedCursor) {
      const op = orderDirection === 'ASC' ? Op.gt : Op.lt;
      whereClause[orderField] = { [op]: decodedCursor[orderField] };
    }
  }
  
  // ดึงข้อมูลเพิ่ม 1 รายการเพื่อตรวจสอบว่ามีหน้าถัดไป
  const items = await Model.findAll({
    where: whereClause,
    limit: limit + 1,
    order: [[orderField, orderDirection]],
    include
  });
  
  const hasNextPage = items.length > limit;
  const edges = hasNextPage ? items.slice(0, limit) : items;
  
  // สร้าง cursor สำหรับหน้าถัดไป
  const endCursor = edges.length > 0 
    ? encodeCursor({ [orderField]: edges[edges.length - 1][orderField] })
    : null;
  
  return {
    edges: edges.map(item => ({
      node: item,
      cursor: encodeCursor({ [orderField]: item[orderField] })
    })),
    pageInfo: {
      hasNextPage,
      hasPreviousPage: !!cursor,
      startCursor: edges.length > 0 
        ? encodeCursor({ [orderField]: edges[0][orderField] })
        : null,
      endCursor
    },
    totalCount: await Model.count({ where })
  };
};

module.exports = { encodeCursor, decodeCursor, getCursorPage };
```

### Controller สำหรับ Cursor Pagination

```javascript
// src/controllers/cursorController.js
const User = require('../models/User');
const { getCursorPage } = require('../utils/cursorPagination');

const getUsersWithCursor = async (req, res) => {
  try {
    const { cursor, limit = 10 } = req.query;
    
    const result = await getCursorPage(User, {
      cursor,
      limit: parseInt(limit),
      orderField: 'id',
      orderDirection: 'ASC'
    });
    
    res.json({
      success: true,
      ...result
    });
  } catch (error) {
    res.status(500).json({
      success: false,
      message: error.message
    });
  }
};

module.exports = { getUsersWithCursor };
```

### ตัวอย่าง Response ของ Cursor Pagination

```json
{
  "success": true,
  "edges": [
    {
      "node": { "id": 1, "name": "สมชาย", "email": "somchai@example.com" },
      "cursor": "eyJpZCI6MX0="
    },
    {
      "node": { "id": 2, "name": "สมหญิง", "email": "somying@example.com" },
      "cursor": "eyJpZCI6Mn0="
    }
  ],
  "pageInfo": {
    "hasNextPage": true,
    "hasPreviousPage": false,
    "startCursor": "eyJpZCI6MX0=",
    "endCursor": "eyJpZCI6MTB9"
  },
  "totalCount": 500
}
```

---

## 46.4 Keyset Pagination (การแบ่งหน้าแบบ Keyset)

Keyset Pagination คล้ายกับ Cursor Pagination แต่ใช้ค่าจริงของ key แทน encoded cursor

### หลักการทำงาน

```sql
-- หน้าแรก
SELECT * FROM users 
ORDER BY created_at DESC, id DESC 
LIMIT 10;

-- หน้าถัดไป (ใช้ค่าจากรายการสุดท้ายของหน้าก่อน)
SELECT * FROM users 
WHERE (created_at, id) < ('2024-01-15 10:00:00', 100)
ORDER BY created_at DESC, id DESC 
LIMIT 10;
```

### Keyset Pagination Implementation

```javascript
// src/utils/keysetPagination.js
const { Op, Sequelize } = require('sequelize');

/**
 * ดึงข้อมูลแบบ keyset pagination
 * รองรับการ sort หลาย column
 */
const getKeysetPage = async (Model, options = {}) => {
  const {
    afterKey,
    limit = 10,
    sortBy = [['id', 'ASC']],
    where = {},
    include = []
  } = options;
  
  let whereClause = { ...where };
  
  if (afterKey) {
    // สร้าง condition สำหรับ keyset
    // ตัวอย่างสำหรับ sort by created_at DESC, id DESC
    const conditions = buildKeysetCondition(afterKey, sortBy);
    whereClause = { ...whereClause, ...conditions };
  }
  
  const items = await Model.findAll({
    where: whereClause,
    limit: limit + 1,
    order: sortBy,
    include
  });
  
  const hasMore = items.length > limit;
  const data = hasMore ? items.slice(0, limit) : items;
  
  // สร้าง key สำหรับหน้าถัดไป
  let nextKey = null;
  if (hasMore && data.length > 0) {
    const lastItem = data[data.length - 1];
    nextKey = sortBy.reduce((acc, [field]) => {
      acc[field] = lastItem[field];
      return acc;
    }, {});
  }
  
  return {
    data,
    hasMore,
    nextKey,
    count: data.length
  };
};

const buildKeysetCondition = (afterKey, sortBy) => {
  // สร้าง WHERE clause สำหรับ multi-column keyset
  if (sortBy.length === 1) {
    const [field, direction] = sortBy[0];
    const op = direction === 'ASC' ? Op.gt : Op.lt;
    return { [field]: { [op]: afterKey[field] } };
  }
  
  // Multi-column keyset condition (ซับซ้อนกว่า)
  // (a, b) > (x, y) เทียบเท่า: a > x OR (a = x AND b > y)
  const conditions = [];
  for (let i = 0; i < sortBy.length; i++) {
    const condition = {};
    for (let j = 0; j <= i; j++) {
      const [field, direction] = sortBy[j];
      const op = direction === 'ASC' ? Op.gt : Op.lt;
      if (j < i) {
        condition[field] = afterKey[field]; // equal condition
      } else {
        condition[field] = { [op]: afterKey[field] }; // greater/less than
      }
    }
    conditions.push(condition);
  }
  
  return { [Op.or]: conditions };
};

module.exports = { getKeysetPage };
```

---

## 46.5 Page Metadata (ข้อมูล Metadata ของหน้า)

### ออกแบบ Response Format มาตรฐาน

```javascript
// src/utils/responseFormatter.js

/**
 * สร้าง standardized API response
 */
const createResponse = ({
  data,
  pagination = null,
  message = 'Success',
  statusCode = 200
}) => {
  const response = {
    success: statusCode < 400,
    message,
    data
  };
  
  if (pagination) {
    response.meta = {
      pagination,
      timestamp: new Date().toISOString(),
      version: '1.0'
    };
  }
  
  return response;
};

/**
 * สร้าง pagination metadata แบบครบถ้วน
 */
const createFullPaginationMeta = ({
  total,
  page,
  limit,
  sortBy = 'id',
  sortOrder = 'ASC',
  filters = {}
}) => {
  const totalPages = Math.ceil(total / limit);
  
  return {
    currentPage: page,
    totalPages,
    totalItems: total,
    itemsPerPage: limit,
    itemsOnCurrentPage: null, // จะถูก set จาก data.length
    hasNextPage: page < totalPages,
    hasPreviousPage: page > 1,
    firstPage: 1,
    lastPage: totalPages,
    sortBy,
    sortOrder,
    appliedFilters: filters,
    // คำนวณ range ของข้อมูลในหน้านี้
    range: {
      from: (page - 1) * limit + 1,
      to: Math.min(page * limit, total)
    }
  };
};

module.exports = { createResponse, createFullPaginationMeta };
```

### Advanced Controller พร้อม Metadata

```javascript
// src/controllers/advancedUserController.js
const { Op } = require('sequelize');
const User = require('../models/User');
const { createResponse, createFullPaginationMeta } = require('../utils/responseFormatter');

const getUsersAdvanced = async (req, res) => {
  try {
    const {
      page = 1,
      limit = 10,
      sortBy = 'createdAt',
      sortOrder = 'DESC',
      search,
      minAge,
      maxAge
    } = req.query;
    
    const pageNum = Math.max(1, parseInt(page));
    const limitNum = Math.min(100, Math.max(1, parseInt(limit)));
    const offset = (pageNum - 1) * limitNum;
    
    // Build filters
    const filters = {};
    const appliedFilters = {};
    
    if (search) {
      filters[Op.or] = [
        { name: { [Op.iLike]: `%${search}%` } },
        { email: { [Op.iLike]: `%${search}%` } }
      ];
      appliedFilters.search = search;
    }
    
    if (minAge) {
      filters.age = { ...filters.age, [Op.gte]: parseInt(minAge) };
      appliedFilters.minAge = minAge;
    }
    
    if (maxAge) {
      filters.age = { ...filters.age, [Op.lte]: parseInt(maxAge) };
      appliedFilters.maxAge = maxAge;
    }
    
    // Validate sort fields
    const allowedSortFields = ['id', 'name', 'email', 'age', 'createdAt'];
    const validSortBy = allowedSortFields.includes(sortBy) ? sortBy : 'createdAt';
    const validSortOrder = ['ASC', 'DESC'].includes(sortOrder.toUpperCase()) 
      ? sortOrder.toUpperCase() 
      : 'DESC';
    
    const { count, rows } = await User.findAndCountAll({
      where: filters,
      limit: limitNum,
      offset,
      order: [[validSortBy, validSortOrder]],
      attributes: { exclude: ['password'] }
    });
    
    const paginationMeta = createFullPaginationMeta({
      total: count,
      page: pageNum,
      limit: limitNum,
      sortBy: validSortBy,
      sortOrder: validSortOrder,
      filters: appliedFilters
    });
    
    paginationMeta.itemsOnCurrentPage = rows.length;
    
    res.json(createResponse({
      data: rows,
      pagination: paginationMeta,
      message: `Found ${count} users`
    }));
    
  } catch (error) {
    res.status(500).json(createResponse({
      data: null,
      message: error.message,
      statusCode: 500
    }));
  }
};

module.exports = { getUsersAdvanced };
```

---

## 46.6 Pagination กับ MongoDB (Mongoose)

```javascript
// src/utils/mongoPagination.js
const mongoose = require('mongoose');

/**
 * Offset pagination สำหรับ MongoDB
 */
const paginateQuery = async (Model, query = {}, options = {}) => {
  const {
    page = 1,
    limit = 10,
    sort = { createdAt: -1 },
    select = '',
    populate = []
  } = options;
  
  const skip = (page - 1) * limit;
  
  // รัน query และ count พร้อมกัน
  const [data, total] = await Promise.all([
    Model.find(query)
      .select(select)
      .sort(sort)
      .skip(skip)
      .limit(limit)
      .populate(populate),
    Model.countDocuments(query)
  ]);
  
  const totalPages = Math.ceil(total / limit);
  
  return {
    data,
    pagination: {
      currentPage: page,
      totalPages,
      totalItems: total,
      itemsPerPage: limit,
      hasNextPage: page < totalPages,
      hasPreviousPage: page > 1
    }
  };
};

/**
 * Cursor pagination สำหรับ MongoDB ใช้ _id
 */
const paginateWithCursor = async (Model, query = {}, options = {}) => {
  const {
    cursor,
    limit = 10,
    sort = { _id: 1 }
  } = options;
  
  let finalQuery = { ...query };
  
  if (cursor) {
    const direction = Object.values(sort)[0];
    const op = direction === 1 ? '$gt' : '$lt';
    finalQuery._id = { [op]: new mongoose.Types.ObjectId(cursor) };
  }
  
  const items = await Model.find(finalQuery)
    .sort(sort)
    .limit(limit + 1);
  
  const hasMore = items.length > limit;
  const data = hasMore ? items.slice(0, limit) : items;
  
  return {
    data,
    hasMore,
    nextCursor: hasMore && data.length > 0 
      ? data[data.length - 1]._id.toString() 
      : null
  };
};

module.exports = { paginateQuery, paginateWithCursor };
```

---

## 46.7 Frontend Integration

```javascript
// ตัวอย่าง React hook สำหรับ pagination
// hooks/usePagination.js

import { useState, useEffect } from 'react';
import axios from 'axios';

const usePagination = (endpoint, initialPage = 1, initialLimit = 10) => {
  const [data, setData] = useState([]);
  const [pagination, setPagination] = useState(null);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState(null);
  const [page, setPage] = useState(initialPage);
  const [limit, setLimit] = useState(initialLimit);
  
  const fetchData = async () => {
    setLoading(true);
    try {
      const response = await axios.get(endpoint, {
        params: { page, limit }
      });
      
      setData(response.data.data);
      setPagination(response.data.pagination);
    } catch (err) {
      setError(err.message);
    } finally {
      setLoading(false);
    }
  };
  
  useEffect(() => {
    fetchData();
  }, [page, limit]);
  
  return {
    data,
    pagination,
    loading,
    error,
    page,
    setPage,
    limit,
    setLimit,
    refetch: fetchData
  };
};

export default usePagination;
```

### Infinite Scroll ด้วย Cursor Pagination

```javascript
// hooks/useInfiniteScroll.js
import { useState, useEffect, useRef, useCallback } from 'react';
import axios from 'axios';

const useInfiniteScroll = (endpoint) => {
  const [items, setItems] = useState([]);
  const [cursor, setCursor] = useState(null);
  const [hasMore, setHasMore] = useState(true);
  const [loading, setLoading] = useState(false);
  const observer = useRef();
  
  const lastItemRef = useCallback(node => {
    if (loading) return;
    
    if (observer.current) observer.current.disconnect();
    
    observer.current = new IntersectionObserver(entries => {
      if (entries[0].isIntersecting && hasMore) {
        loadMore();
      }
    });
    
    if (node) observer.current.observe(node);
  }, [loading, hasMore]);
  
  const loadMore = async () => {
    if (loading || !hasMore) return;
    
    setLoading(true);
    try {
      const params = { limit: 20 };
      if (cursor) params.cursor = cursor;
      
      const response = await axios.get(endpoint, { params });
      const { edges, pageInfo } = response.data;
      
      setItems(prev => [...prev, ...edges.map(e => e.node)]);
      setCursor(pageInfo.endCursor);
      setHasMore(pageInfo.hasNextPage);
    } catch (error) {
      console.error('Error loading more:', error);
    } finally {
      setLoading(false);
    }
  };
  
  useEffect(() => {
    loadMore();
  }, []);
  
  return { items, loading, hasMore, lastItemRef };
};
```

---

## 46.8 Performance Optimization

### Caching Pagination Results

```javascript
// src/middleware/cacheMiddleware.js
const redis = require('redis');
const client = redis.createClient(process.env.REDIS_URL);

const cachePagination = (ttl = 60) => {
  return async (req, res, next) => {
    const cacheKey = `pagination:${req.path}:${JSON.stringify(req.query)}`;
    
    try {
      const cached = await client.get(cacheKey);
      
      if (cached) {
        return res.json({
          ...JSON.parse(cached),
          fromCache: true
        });
      }
      
      // Override res.json เพื่อ cache ก่อนส่ง
      const originalJson = res.json.bind(res);
      res.json = async (data) => {
        await client.setEx(cacheKey, ttl, JSON.stringify(data));
        return originalJson(data);
      };
      
      next();
    } catch (error) {
      // ถ้า cache ไม่ทำงาน ให้ผ่านต่อได้
      next();
    }
  };
};

module.exports = { cachePagination };
```

### Index Optimization สำหรับ Pagination

```sql
-- สร้าง index สำหรับ offset pagination
CREATE INDEX idx_users_created_at ON users(created_at DESC);

-- Composite index สำหรับ filtered pagination
CREATE INDEX idx_users_status_created ON users(status, created_at DESC);

-- สำหรับ cursor pagination
CREATE INDEX idx_users_id ON users(id);
CREATE INDEX idx_users_cursor ON users(created_at DESC, id DESC);
```

---

## แบบฝึกหัดที่ 46

### แบบฝึกหัดพื้นฐาน

**1. สร้าง Offset Pagination API**

สร้าง REST API สำหรับ Product catalog ที่รองรับ:
- Pagination (page, limit)
- Sorting (price, name, createdAt)
- Filtering (category, minPrice, maxPrice)
- Full metadata response

**2. สร้าง Cursor Pagination API**

แปลง API จากข้อ 1 ให้ใช้ cursor pagination เหมาะสำหรับ infinite scroll

### แบบฝึกหัดขั้นสูง

**3. สร้าง Pagination Middleware ที่ใช้ซ้ำได้**

```javascript
// เป้าหมาย: ใช้งานได้แบบนี้
router.get('/products', 
  paginateMiddleware({
    defaultLimit: 20,
    maxLimit: 100,
    allowedSortFields: ['price', 'name', 'createdAt'],
    defaultSort: { field: 'createdAt', order: 'DESC' }
  }),
  productController.getAll
);
```

**4. Benchmark Offset vs Cursor**

สร้าง script ที่:
- Insert ข้อมูล 1 ล้าน records
- ทดสอบ offset pagination ที่หน้า 1, 100, 1000, 10000
- ทดสอบ cursor pagination ที่ตำแหน่งเดียวกัน
- เปรียบเทียบ performance

**5. สร้าง Pagination Component (Frontend)**

สร้าง React component ที่รองรับทั้ง:
- Classic pagination (หน้า 1, 2, 3...)
- Infinite scroll
- Load more button

### เฉลยแบบฝึกหัด 1

```javascript
// solution/productController.js
const { Op } = require('sequelize');
const Product = require('./models/Product');

const getProducts = async (req, res) => {
  try {
    const {
      page = 1,
      limit = 20,
      sortBy = 'createdAt',
      sortOrder = 'DESC',
      category,
      minPrice,
      maxPrice,
      search
    } = req.query;
    
    const pageNum = Math.max(1, parseInt(page));
    const limitNum = Math.min(100, Math.max(1, parseInt(limit)));
    const offset = (pageNum - 1) * limitNum;
    
    // Build where clause
    const where = {};
    if (category) where.category = category;
    if (minPrice || maxPrice) {
      where.price = {};
      if (minPrice) where.price[Op.gte] = parseFloat(minPrice);
      if (maxPrice) where.price[Op.lte] = parseFloat(maxPrice);
    }
    if (search) {
      where[Op.or] = [
        { name: { [Op.iLike]: `%${search}%` } },
        { description: { [Op.iLike]: `%${search}%` } }
      ];
    }
    
    // Validate sort
    const allowedFields = ['price', 'name', 'createdAt', 'stock'];
    const finalSortBy = allowedFields.includes(sortBy) ? sortBy : 'createdAt';
    const finalSortOrder = ['ASC', 'DESC'].includes(sortOrder.toUpperCase())
      ? sortOrder.toUpperCase() : 'DESC';
    
    const { count, rows } = await Product.findAndCountAll({
      where,
      limit: limitNum,
      offset,
      order: [[finalSortBy, finalSortOrder]]
    });
    
    const totalPages = Math.ceil(count / limitNum);
    
    res.json({
      success: true,
      data: rows,
      meta: {
        pagination: {
          currentPage: pageNum,
          totalPages,
          totalItems: count,
          itemsPerPage: limitNum,
          itemsOnCurrentPage: rows.length,
          hasNextPage: pageNum < totalPages,
          hasPreviousPage: pageNum > 1,
          range: {
            from: offset + 1,
            to: Math.min(offset + limitNum, count)
          }
        },
        filters: { category, minPrice, maxPrice, search },
        sort: { field: finalSortBy, order: finalSortOrder }
      }
    });
    
  } catch (error) {
    res.status(500).json({ success: false, message: error.message });
  }
};

module.exports = { getProducts };
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **Offset Pagination** - วิธีที่ง่ายที่สุด ใช้ page และ limit
2. **Cursor Pagination** - ประสิทธิภาพดีกว่า เหมาะกับ infinite scroll
3. **Keyset Pagination** - คล้าย cursor แต่ใช้ค่าจริงของ key
4. **Page Metadata** - ข้อมูลเสริมที่ client ต้องการ
5. **Performance Optimization** - caching, indexing

### เมื่อไหร่ควรใช้แต่ละแบบ?

| สถานการณ์ | Pagination ที่แนะนำ |
|-----------|---------------------|
| Admin panel ที่ต้องกระโดดหน้า | Offset |
| Social feed / infinite scroll | Cursor |
| Real-time data | Cursor / Keyset |
| ข้อมูลน้อยกว่า 10,000 records | Offset |
| ข้อมูลมากกว่า 1 ล้าน records | Cursor / Keyset |

---

**ถัดไป:** Part 47 - Search & Filtering (การค้นหาและกรองข้อมูล)
