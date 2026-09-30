# Part 51 | ขั้นตอนที่ 881-900 จาก 1000

## API Versioning - การจัดการเวอร์ชัน API

API Versioning เป็นกลยุทธ์สำคัญในการพัฒนา API อย่างยั่งยืน ช่วยให้เราสามารถพัฒนาและปรับปรุง API ได้โดยไม่ทำลาย clients ที่มีอยู่

---

## ขั้นตอนที่ 881: ทำความเข้าใจ API Versioning

### ทำไมต้องมี API Versioning?
- **Backward Compatibility**: รักษาความเข้ากันได้กับ clients เก่า
- **Evolution**: สามารถพัฒนา API โดยไม่กระทบ clients
- **Deprecation**: ค่อยๆ เลิกใช้ version เก่า
- **Multiple Clients**: รองรับ clients ที่มีความต้องการต่างกัน

### กลยุทธ์ API Versioning:
1. **URL Path Versioning**: `/api/v1/users`
2. **Query Parameter**: `/api/users?version=1`
3. **Header Versioning**: `Accept: application/vnd.myapi.v1+json`
4. **Subdomain**: `v1.api.example.com`
5. **Content Negotiation**: `Accept: application/json; version=1`

---

## ขั้นตอนที่ 882: URL Path Versioning

```javascript
// routes/index.js
const express = require('express');
const router = express.Router();

// Version 1 routes
const v1Routes = require('./v1');
router.use('/v1', v1Routes);

// Version 2 routes
const v2Routes = require('./v2');
router.use('/v2', v2Routes);

// Default version (latest stable)
router.use('/', v2Routes);

module.exports = router;
```

```javascript
// routes/v1/index.js
const express = require('express');
const router = express.Router();

router.use('/users', require('./users'));
router.use('/products', require('./products'));
router.use('/orders', require('./orders'));

module.exports = router;
```

```javascript
// routes/v1/users.js
const express = require('express');
const router = express.Router();
const UserController = require('../../controllers/v1/UserController');

router.get('/', UserController.getAll);
router.get('/:id', UserController.getById);
router.post('/', UserController.create);
router.put('/:id', UserController.update);
router.delete('/:id', UserController.delete);

module.exports = router;
```

```javascript
// routes/v2/users.js - เวอร์ชันใหม่ที่มีการเปลี่ยนแปลง
const express = require('express');
const router = express.Router();
const UserController = require('../../controllers/v2/UserController');

router.get('/', UserController.getAll);         // เพิ่ม pagination
router.get('/:id', UserController.getById);     // เพิ่ม populate
router.get('/:id/profile', UserController.getProfile); // endpoint ใหม่
router.post('/', UserController.create);
router.patch('/:id', UserController.update);    // เปลี่ยนจาก PUT เป็น PATCH
router.delete('/:id', UserController.delete);

module.exports = router;
```

---

## ขั้นตอนที่ 883: Header-based Versioning

```javascript
// middleware/apiVersioning.js

const API_VERSIONS = ['1', '2', '3'];
const CURRENT_VERSION = '3';
const MINIMUM_VERSION = '1';

const extractVersion = (req) => {
  // ลอง header ก่อน
  const headerVersion = req.headers['api-version'] || 
                        req.headers['x-api-version'];
  
  if (headerVersion) {
    return headerVersion.replace(/^v/, ''); // ลบ 'v' นำหน้า
  }
  
  // ลอง Accept header
  const acceptHeader = req.headers['accept'] || '';
  const versionMatch = acceptHeader.match(/version=(\d+)/);
  if (versionMatch) {
    return versionMatch[1];
  }
  
  // ลอง URL path
  const pathMatch = req.path.match(/^\/v(\d+)\//);
  if (pathMatch) {
    return pathMatch[1];
  }
  
  // Default version
  return CURRENT_VERSION;
};

const apiVersionMiddleware = (req, res, next) => {
  const version = extractVersion(req);
  
  // ตรวจสอบว่า version ถูกต้อง
  if (!API_VERSIONS.includes(version)) {
    return res.status(400).json({
      success: false,
      error: 'INVALID_API_VERSION',
      message: `API version '${version}' is not supported`,
      supportedVersions: API_VERSIONS
    });
  }
  
  // ตรวจสอบ minimum version
  if (parseInt(version) < parseInt(MINIMUM_VERSION)) {
    return res.status(410).json({
      success: false,
      error: 'API_VERSION_DEPRECATED',
      message: `API version '${version}' is no longer supported`,
      minimumVersion: MINIMUM_VERSION
    });
  }
  
  req.apiVersion = version;
  
  // เพิ่ม headers ที่บอก API version
  res.set({
    'X-API-Version': version,
    'X-API-Current-Version': CURRENT_VERSION,
    'X-API-Deprecated': parseInt(version) < parseInt(CURRENT_VERSION) ? 'true' : 'false'
  });
  
  next();
};

module.exports = { apiVersionMiddleware, extractVersion };
```

---

## ขั้นตอนที่ 884: Content Negotiation Versioning

```javascript
// middleware/contentNegotiation.js

/*
  Client ส่ง: Accept: application/vnd.myapi.v2+json
  หรือ:       Content-Type: application/vnd.myapi.v2+json
*/

const VENDOR_MEDIA_TYPE = 'application/vnd.myapi';

const parseVendorVersion = (mediaType) => {
  const regex = new RegExp(`${VENDOR_MEDIA_TYPE}\\.v(\\d+)\\+json`);
  const match = mediaType.match(regex);
  return match ? match[1] : null;
};

const contentNegotiationMiddleware = (req, res, next) => {
  const accept = req.headers['accept'] || '';
  const contentType = req.headers['content-type'] || '';
  
  // ลอง extract version จาก Accept header
  let version = parseVendorVersion(accept);
  
  // ถ้าไม่พบ ลอง Content-Type
  if (!version) {
    version = parseVendorVersion(contentType);
  }
  
  // Default version
  if (!version) {
    version = '2';
  }
  
  req.apiVersion = version;
  res.set('Content-Type', `${VENDOR_MEDIA_TYPE}.v${version}+json`);
  
  next();
};

// Formatter ที่ transform response ตาม version
const createVersionedResponse = (version, data, meta = {}) => {
  if (version === '1') {
    // V1 format เก่า
    return {
      data,
      status: 'success',
      ...meta
    };
  }
  
  if (version === '2') {
    // V2 format ใหม่
    return {
      success: true,
      data,
      meta: {
        version: '2',
        timestamp: new Date().toISOString(),
        ...meta
      }
    };
  }
  
  // V3+ format
  return {
    success: true,
    data,
    meta: {
      version,
      timestamp: new Date().toISOString(),
      requestId: meta.requestId,
      ...meta
    },
    links: meta.links || {}
  };
};

module.exports = { contentNegotiationMiddleware, createVersionedResponse };
```

---

## ขั้นตอนที่ 885: Versioned Controllers

```javascript
// controllers/UserController.js - Base class

class BaseUserController {
  // ตรรกะหลักที่ใช้ร่วมกันทุก version
  async findUser(id) {
    const User = require('../models/User');
    return User.findById(id);
  }

  async getUsers(options = {}) {
    const User = require('../models/User');
    const { page = 1, limit = 20, sort = '-createdAt' } = options;
    
    const users = await User.find()
      .sort(sort)
      .skip((page - 1) * limit)
      .limit(limit);
    
    const total = await User.countDocuments();
    
    return { users, total, page, limit };
  }
}

// controllers/v1/UserController.js
class UserControllerV1 extends BaseUserController {
  async getAll(req, res) {
    try {
      const users = await this.getUsers();
      
      // V1 response format
      res.json({
        status: 'success',
        data: users.users
      });
    } catch (error) {
      res.status(500).json({ status: 'error', message: error.message });
    }
  }

  async getById(req, res) {
    try {
      const user = await this.findUser(req.params.id);
      
      if (!user) {
        return res.status(404).json({ status: 'error', message: 'User not found' });
      }
      
      // V1: ส่งแค่ basic fields
      res.json({
        status: 'success',
        data: {
          id: user._id,
          name: user.name,
          email: user.email
        }
      });
    } catch (error) {
      res.status(500).json({ status: 'error', message: error.message });
    }
  }
}

// controllers/v2/UserController.js
class UserControllerV2 extends BaseUserController {
  async getAll(req, res) {
    try {
      const { page = 1, limit = 20, sort = '-createdAt', search } = req.query;
      
      const User = require('../../models/User');
      const query = {};
      
      if (search) {
        query.$or = [
          { name: { $regex: search, $options: 'i' } },
          { email: { $regex: search, $options: 'i' } }
        ];
      }
      
      const [users, total] = await Promise.all([
        User.find(query)
          .sort(sort)
          .skip((page - 1) * limit)
          .limit(parseInt(limit))
          .select('-password'),
        User.countDocuments(query)
      ]);
      
      // V2 response format พร้อม pagination
      res.json({
        success: true,
        data: users,
        pagination: {
          page: parseInt(page),
          limit: parseInt(limit),
          total,
          pages: Math.ceil(total / limit)
        }
      });
    } catch (error) {
      res.status(500).json({ success: false, error: error.message });
    }
  }

  async getById(req, res) {
    try {
      const User = require('../../models/User');
      const user = await User.findById(req.params.id)
        .select('-password')
        .populate('role', 'name permissions');
      
      if (!user) {
        return res.status(404).json({
          success: false,
          error: { code: 'USER_NOT_FOUND', message: 'User not found' }
        });
      }
      
      // V2: ส่งข้อมูลเพิ่มเติม
      res.json({
        success: true,
        data: user
      });
    } catch (error) {
      res.status(500).json({ success: false, error: error.message });
    }
  }
}

module.exports = { UserControllerV1, UserControllerV2 };
```

---

## ขั้นตอนที่ 886: Version Router Factory

```javascript
// utils/versionRouter.js

class VersionRouter {
  constructor() {
    this.handlers = new Map();
  }

  // ลงทะเบียน handler สำหรับแต่ละ version
  version(version, handler) {
    this.handlers.set(version.toString(), handler);
    return this;
  }

  // สร้าง middleware จาก registered handlers
  build() {
    return async (req, res, next) => {
      const version = req.apiVersion || '1';
      
      // หา handler ที่ตรงกับ version หรือ version ที่ต่ำกว่า
      let handler = this.handlers.get(version);
      
      if (!handler) {
        // หา version ที่ใกล้เคียงที่สุดที่ต่ำกว่า
        const versions = Array.from(this.handlers.keys())
          .map(Number)
          .sort((a, b) => b - a);
        
        const fallbackVersion = versions.find(v => v <= parseInt(version));
        
        if (fallbackVersion) {
          handler = this.handlers.get(fallbackVersion.toString());
        }
      }
      
      if (!handler) {
        return res.status(400).json({
          success: false,
          message: `No handler found for API version ${version}`
        });
      }
      
      try {
        await handler(req, res, next);
      } catch (error) {
        next(error);
      }
    };
  }
}

// ใช้งาน
const getUsersRoute = new VersionRouter()
  .version('1', async (req, res) => {
    // V1 implementation
    const users = await User.find().select('name email');
    res.json({ data: users });
  })
  .version('2', async (req, res) => {
    // V2 implementation พร้อม pagination
    const { page = 1, limit = 20 } = req.query;
    const users = await User.find()
      .skip((page - 1) * limit)
      .limit(parseInt(limit));
    res.json({ success: true, data: users });
  })
  .version('3', async (req, res) => {
    // V3 implementation พร้อม cursor pagination
    const { cursor, limit = 20 } = req.query;
    const query = cursor ? { _id: { $gt: cursor } } : {};
    const users = await User.find(query).limit(parseInt(limit) + 1).lean();
    const hasMore = users.length > limit;
    const items = hasMore ? users.slice(0, limit) : users;
    
    res.json({
      success: true,
      data: items,
      pagination: {
        cursor: items.length > 0 ? items[items.length - 1]._id : null,
        hasMore
      }
    });
  })
  .build();

router.get('/users', getUsersRoute);
```

---

## ขั้นตอนที่ 887: Deprecation Policy

```javascript
// middleware/deprecationWarning.js

const DEPRECATION_SCHEDULE = {
  'v1': {
    deprecated: true,
    deprecatedOn: '2024-01-01',
    sunsetDate: '2024-07-01',
    message: 'API v1 is deprecated. Please migrate to v2.',
    migrationGuide: 'https://docs.example.com/api/migration/v1-to-v2'
  },
  'v2': {
    deprecated: false,
    current: true
  }
};

const deprecationMiddleware = (req, res, next) => {
  const version = req.apiVersion;
  const schedule = DEPRECATION_SCHEDULE[`v${version}`];
  
  if (schedule && schedule.deprecated) {
    // เพิ่ม Deprecation headers ตาม RFC 8594
    res.set({
      'Deprecation': `date="${new Date(schedule.deprecatedOn).toUTCString()}"`,
      'Sunset': new Date(schedule.sunsetDate).toUTCString(),
      'Link': `<${schedule.migrationGuide}>; rel="deprecation"`,
      'Warning': `299 - "${schedule.message}"`
    });
    
    // Log การใช้งาน deprecated API
    console.warn({
      timestamp: new Date().toISOString(),
      type: 'DEPRECATED_API_USAGE',
      version,
      path: req.path,
      method: req.method,
      userAgent: req.headers['user-agent'],
      ip: req.ip,
      userId: req.user?.id
    });
    
    // ตรวจสอบว่าถึง sunset date แล้วหรือไม่
    if (new Date() > new Date(schedule.sunsetDate)) {
      return res.status(410).json({
        success: false,
        error: 'API_SUNSET',
        message: `API version ${version} has been sunset and is no longer available`,
        sunsetDate: schedule.sunsetDate,
        migrationGuide: schedule.migrationGuide
      });
    }
  }
  
  next();
};

module.exports = deprecationMiddleware;
```

---

## ขั้นตอนที่ 888: API Version Registry

```javascript
// config/apiVersions.js

const API_VERSION_REGISTRY = {
  versions: {
    '1': {
      status: 'deprecated',
      releaseDate: '2023-01-01',
      deprecatedDate: '2024-01-01',
      sunsetDate: '2024-07-01',
      features: ['basic-crud', 'authentication'],
      breaking_changes: []
    },
    '2': {
      status: 'stable',
      releaseDate: '2024-01-01',
      deprecatedDate: null,
      sunsetDate: null,
      features: ['pagination', 'filtering', 'sorting', 'full-text-search'],
      breaking_changes: [
        'Response format changed to { success, data, meta }',
        'PUT replaced with PATCH for partial updates',
        'Error responses now include error codes'
      ]
    },
    '3': {
      status: 'beta',
      releaseDate: '2024-06-01',
      deprecatedDate: null,
      sunsetDate: null,
      features: ['cursor-pagination', 'graphql-support', 'websockets'],
      breaking_changes: [
        'Cursor-based pagination replaces offset pagination',
        'New error format with multiple errors support'
      ]
    }
  },
  current: '2',
  minimum: '2',
  beta: '3'
};

// Route สำหรับ API info
const apiInfoRouter = require('express').Router();

apiInfoRouter.get('/versions', (req, res) => {
  res.json({
    success: true,
    data: {
      current: API_VERSION_REGISTRY.current,
      minimum: API_VERSION_REGISTRY.minimum,
      versions: Object.entries(API_VERSION_REGISTRY.versions).map(([v, info]) => ({
        version: v,
        ...info,
        endpoint: `/api/v${v}`
      }))
    }
  });
});

apiInfoRouter.get('/versions/:version', (req, res) => {
  const { version } = req.params;
  const versionInfo = API_VERSION_REGISTRY.versions[version];
  
  if (!versionInfo) {
    return res.status(404).json({
      success: false,
      message: `Version ${version} not found`
    });
  }
  
  res.json({
    success: true,
    data: { version, ...versionInfo }
  });
});

module.exports = { API_VERSION_REGISTRY, apiInfoRouter };
```

---

## ขั้นตอนที่ 889: Response Transformation

```javascript
// middleware/responseTransformer.js

// Transformer สำหรับแปลง response format ตาม version
class ResponseTransformer {
  static transform(data, version, meta = {}) {
    switch (version) {
      case '1':
        return this.v1Format(data, meta);
      case '2':
        return this.v2Format(data, meta);
      default:
        return this.v3Format(data, meta);
    }
  }

  static v1Format(data, meta) {
    return {
      status: 'success',
      data,
      ...meta
    };
  }

  static v2Format(data, meta) {
    return {
      success: true,
      data,
      meta: {
        timestamp: new Date().toISOString(),
        ...meta
      }
    };
  }

  static v3Format(data, meta) {
    const response = {
      success: true,
      data,
      meta: {
        timestamp: new Date().toISOString(),
        version: '3',
        ...meta
      }
    };
    
    if (meta.pagination) {
      response.links = {
        self: meta.self,
        next: meta.pagination.hasNextPage ? meta.next : null,
        prev: meta.pagination.hasPrevPage ? meta.prev : null
      };
    }
    
    return response;
  }

  // แปลง error format ตาม version
  static transformError(error, version) {
    switch (version) {
      case '1':
        return {
          status: 'error',
          message: error.message
        };
      
      case '2':
        return {
          success: false,
          error: {
            code: error.code || 'INTERNAL_ERROR',
            message: error.message
          }
        };
      
      default:
        return {
          success: false,
          errors: [{
            code: error.code || 'INTERNAL_ERROR',
            message: error.message,
            field: error.field,
            details: error.details
          }]
        };
    }
  }
}

// Middleware ที่ inject transformer ไปยัง response
const injectTransformer = (req, res, next) => {
  const version = req.apiVersion || '2';
  
  res.success = (data, meta = {}) => {
    return res.json(ResponseTransformer.transform(data, version, meta));
  };
  
  res.fail = (error, statusCode = 500) => {
    return res.status(statusCode).json(
      ResponseTransformer.transformError(error, version)
    );
  };
  
  next();
};

module.exports = { ResponseTransformer, injectTransformer };
```

---

## ขั้นตอนที่ 890: Feature Flags ร่วมกับ Versioning

```javascript
// services/featureFlags.js
const redis = require('../config/redis');

class FeatureFlagService {
  constructor() {
    this.prefix = 'feature:';
    this.defaultFlags = {
      'pagination': { enabled: true, minVersion: '1' },
      'cursor-pagination': { enabled: true, minVersion: '3' },
      'full-text-search': { enabled: true, minVersion: '2' },
      'websockets': { enabled: false, minVersion: '3' },
      'graphql': { enabled: false, minVersion: '3' }
    };
  }

  async isEnabled(feature, version) {
    const key = `${this.prefix}${feature}`;
    
    // ลอง get จาก Redis ก่อน (dynamic flags)
    const dynamicFlag = await redis.get(key);
    
    if (dynamicFlag !== null) {
      const flag = JSON.parse(dynamicFlag);
      if (!flag.enabled) return false;
      if (flag.minVersion && parseInt(version) < parseInt(flag.minVersion)) {
        return false;
      }
      return true;
    }
    
    // Fallback ไปยัง default flags
    const defaultFlag = this.defaultFlags[feature];
    if (!defaultFlag) return false;
    
    if (!defaultFlag.enabled) return false;
    if (defaultFlag.minVersion && parseInt(version) < parseInt(defaultFlag.minVersion)) {
      return false;
    }
    
    return true;
  }

  async setFlag(feature, config) {
    const key = `${this.prefix}${feature}`;
    await redis.set(key, JSON.stringify(config));
  }

  async getFlags(version) {
    const flags = {};
    
    for (const [feature, config] of Object.entries(this.defaultFlags)) {
      flags[feature] = await this.isEnabled(feature, version);
    }
    
    return flags;
  }
}

const featureFlags = new FeatureFlagService();

// Middleware ที่ inject feature flags
const featureFlagMiddleware = async (req, res, next) => {
  const version = req.apiVersion || '2';
  req.features = await featureFlags.getFlags(version);
  next();
};

module.exports = { featureFlags, featureFlagMiddleware };
```

---

## ขั้นตอนที่ 891: Migration Guide Generator

```javascript
// utils/migrationGuide.js

const BREAKING_CHANGES = {
  'v1-to-v2': [
    {
      type: 'response_format',
      description: 'Response format changed',
      before: {
        example: `{
  "status": "success",
  "data": [...]
}`
      },
      after: {
        example: `{
  "success": true,
  "data": [...],
  "meta": {
    "timestamp": "2024-01-01T00:00:00Z"
  }
}`
      }
    },
    {
      type: 'http_method',
      description: 'PUT changed to PATCH for partial updates',
      before: { method: 'PUT', endpoint: '/api/v1/users/:id' },
      after: { method: 'PATCH', endpoint: '/api/v2/users/:id' }
    },
    {
      type: 'pagination',
      description: 'Pagination now in query params',
      before: {
        example: 'GET /api/v1/users (returns all users)'
      },
      after: {
        example: 'GET /api/v2/users?page=1&limit=20'
      }
    }
  ]
};

// Route สำหรับ migration guide
const migrationRouter = require('express').Router();

migrationRouter.get('/migrate/:from/:to', (req, res) => {
  const { from, to } = req.params;
  const key = `${from}-to-${to}`;
  const guide = BREAKING_CHANGES[key];
  
  if (!guide) {
    return res.status(404).json({
      success: false,
      message: `No migration guide available for ${from} to ${to}`
    });
  }
  
  res.json({
    success: true,
    data: {
      from,
      to,
      breakingChanges: guide,
      summary: `${guide.length} breaking changes`
    }
  });
});

module.exports = migrationRouter;
```

---

## ขั้นตอนที่ 892: API Gateway Versioning

```javascript
// gateway/apiGateway.js
const httpProxy = require('http-proxy-middleware');
const express = require('express');
const app = express();

// Upstream services ตาม version
const UPSTREAM_SERVICES = {
  users: {
    v1: 'http://users-service-v1:3001',
    v2: 'http://users-service-v2:3002',
    v3: 'http://users-service-v3:3003'
  },
  products: {
    v1: 'http://products-service-v1:4001',
    v2: 'http://products-service-v2:4002'
  },
  orders: {
    v1: 'http://orders-service-v1:5001',
    v2: 'http://orders-service-v2:5002'
  }
};

// สร้าง proxy middleware ตาม version
const createVersionedProxy = (service) => {
  return (req, res, next) => {
    const version = req.apiVersion || '2';
    const serviceVersions = UPSTREAM_SERVICES[service];
    
    // หา upstream ที่ตรงกับ version
    const upstream = serviceVersions[`v${version}`] || 
                     serviceVersions[`v${Math.max(...Object.keys(serviceVersions).map(k => parseInt(k.replace('v', ''))))}`];
    
    if (!upstream) {
      return res.status(503).json({
        success: false,
        message: `Service ${service} not available for version ${version}`
      });
    }
    
    const proxy = httpProxy.createProxyMiddleware({
      target: upstream,
      changeOrigin: true,
      pathRewrite: {
        [`^/api/v${version}/${service}`]: `/${service}`
      }
    });
    
    proxy(req, res, next);
  };
};

// Routes
app.use('/api', require('./middleware/apiVersioning').apiVersionMiddleware);
app.use('/api/:version/users', createVersionedProxy('users'));
app.use('/api/:version/products', createVersionedProxy('products'));
app.use('/api/:version/orders', createVersionedProxy('orders'));

module.exports = app;
```

---

## ขั้นตอนที่ 893: Version-aware Validation

```javascript
// middleware/versionedValidation.js
const Joi = require('joi');

// Schema ที่แตกต่างกันตาม version
const USER_SCHEMAS = {
  v1: {
    create: Joi.object({
      name: Joi.string().required(),
      email: Joi.string().email().required(),
      password: Joi.string().min(6).required()
    }),
    update: Joi.object({
      name: Joi.string(),
      email: Joi.string().email()
    })
  },
  v2: {
    create: Joi.object({
      name: Joi.string().min(2).max(100).required(),
      email: Joi.string().email().required(),
      password: Joi.string().min(8).pattern(/^(?=.*[A-Z])(?=.*[0-9])/).required()
        .messages({
          'string.pattern.base': 'Password must contain at least one uppercase letter and one number'
        }),
      phone: Joi.string().pattern(/^\+\d{10,15}$/),
      role: Joi.string().valid('user', 'admin').default('user')
    }),
    update: Joi.object({
      name: Joi.string().min(2).max(100),
      email: Joi.string().email(),
      phone: Joi.string().pattern(/^\+\d{10,15}$/)
    }).min(1) // ต้องมีอย่างน้อย 1 field
  }
};

// Version-aware validation middleware
const validate = (resourceType, action) => {
  return (req, res, next) => {
    const version = req.apiVersion || '2';
    const schemas = USER_SCHEMAS[`v${version}`] || USER_SCHEMAS.v2;
    const schema = schemas[action];
    
    if (!schema) {
      return next();
    }
    
    const { error, value } = schema.validate(req.body, {
      abortEarly: false, // ส่ง errors ทั้งหมด ไม่หยุดที่ error แรก
      stripUnknown: true // ลบ fields ที่ไม่รู้จัก
    });
    
    if (error) {
      const errors = error.details.map(d => ({
        field: d.path.join('.'),
        message: d.message,
        code: d.type
      }));
      
      // Format error ตาม version
      if (version === '1') {
        return res.status(422).json({
          status: 'error',
          message: errors[0].message,
          errors
        });
      }
      
      return res.status(422).json({
        success: false,
        errors
      });
    }
    
    req.body = value; // ใช้ validated/sanitized data
    next();
  };
};

module.exports = validate;
```

---

## ขั้นตอนที่ 894: API Version Testing

```javascript
// tests/apiVersioning.test.js
const request = require('supertest');
const app = require('../app');

describe('API Versioning', () => {
  describe('URL Path Versioning', () => {
    it('should use v1 endpoint', async () => {
      const res = await request(app)
        .get('/api/v1/users')
        .expect(200);
      
      // V1 response format
      expect(res.body).toHaveProperty('status', 'success');
      expect(res.body).toHaveProperty('data');
      expect(res.body).not.toHaveProperty('success'); // V1 ไม่มี success field
    });

    it('should use v2 endpoint', async () => {
      const res = await request(app)
        .get('/api/v2/users')
        .expect(200);
      
      // V2 response format
      expect(res.body).toHaveProperty('success', true);
      expect(res.body).toHaveProperty('data');
      expect(res.body).toHaveProperty('pagination');
    });
  });

  describe('Header Versioning', () => {
    it('should use version from header', async () => {
      const res = await request(app)
        .get('/api/users')
        .set('api-version', 'v2')
        .expect(200);
      
      expect(res.headers['x-api-version']).toBe('2');
      expect(res.body).toHaveProperty('success', true);
    });

    it('should return current version in response header', async () => {
      const res = await request(app)
        .get('/api/v2/users')
        .expect(200);
      
      expect(res.headers['x-api-current-version']).toBeDefined();
    });
  });

  describe('Deprecation', () => {
    it('should return deprecation warning for old versions', async () => {
      const res = await request(app)
        .get('/api/v1/users');
      
      expect(res.headers['deprecation']).toBeDefined();
      expect(res.headers['sunset']).toBeDefined();
    });
  });

  describe('Invalid Version', () => {
    it('should reject unknown versions', async () => {
      const res = await request(app)
        .get('/api/v99/users')
        .expect(400);
      
      expect(res.body).toHaveProperty('error', 'INVALID_API_VERSION');
    });
  });
});
```

---

## ขั้นตอนที่ 895: Changelog Management

```javascript
// config/changelog.js

const CHANGELOG = {
  '3.0.0': {
    date: '2024-06-01',
    type: 'major',
    changes: [
      {
        type: 'breaking',
        description: 'Cursor-based pagination replaces offset pagination',
        affected_endpoints: ['/api/v3/users', '/api/v3/products']
      },
      {
        type: 'feature',
        description: 'Added GraphQL support at /graphql'
      },
      {
        type: 'feature',
        description: 'Real-time updates via WebSockets'
      }
    ]
  },
  '2.5.0': {
    date: '2024-03-01',
    type: 'minor',
    changes: [
      {
        type: 'feature',
        description: 'Added full-text search for products and users'
      },
      {
        type: 'improvement',
        description: 'Improved error messages with machine-readable codes'
      }
    ]
  },
  '2.0.0': {
    date: '2024-01-01',
    type: 'major',
    changes: [
      {
        type: 'breaking',
        description: 'Response format standardized to { success, data, meta }'
      },
      {
        type: 'breaking',
        description: 'PUT method replaced by PATCH for partial updates'
      },
      {
        type: 'feature',
        description: 'Added pagination, filtering, and sorting'
      }
    ]
  }
};

// Route สำหรับ changelog
const changelogRouter = require('express').Router();

changelogRouter.get('/', (req, res) => {
  res.json({
    success: true,
    data: CHANGELOG
  });
});

changelogRouter.get('/:version', (req, res) => {
  const changes = CHANGELOG[req.params.version];
  
  if (!changes) {
    return res.status(404).json({
      success: false,
      message: `Version ${req.params.version} not found in changelog`
    });
  }
  
  res.json({
    success: true,
    data: { version: req.params.version, ...changes }
  });
});

module.exports = { CHANGELOG, changelogRouter };
```

---

## ขั้นตอนที่ 896: Complete API Versioning Setup

```javascript
// app.js - Full API versioning setup
const express = require('express');
const app = express();
const { apiVersionMiddleware } = require('./middleware/apiVersioning');
const deprecationMiddleware = require('./middleware/deprecationWarning');
const { injectTransformer } = require('./middleware/responseTransformer');
const { featureFlagMiddleware } = require('./services/featureFlags');
const { apiInfoRouter } = require('./config/apiVersions');
const { changelogRouter } = require('./config/changelog');
const migrationRouter = require('./utils/migrationGuide');

app.use(express.json());

// API version detection
app.use('/api', apiVersionMiddleware);

// Deprecation warnings
app.use('/api', deprecationMiddleware);

// Response transformer
app.use('/api', injectTransformer);

// Feature flags
app.use('/api', featureFlagMiddleware);

// API info routes (no versioning needed)
app.use('/api/info', apiInfoRouter);
app.use('/api/changelog', changelogRouter);
app.use('/api/migrate', migrationRouter);

// Versioned routes
app.use('/api/v1', require('./routes/v1'));
app.use('/api/v2', require('./routes/v2'));
app.use('/api/v3', require('./routes/v3'));

// Default to latest stable
app.use('/api', require('./routes/v2'));

module.exports = app;
```

---

## ขั้นตอนที่ 897: Monitoring API Version Usage

```javascript
// middleware/versionTracking.js
const redis = require('../config/redis');

const trackVersionUsage = async (req, res, next) => {
  const version = req.apiVersion || 'unknown';
  const date = new Date().toISOString().split('T')[0];
  
  // เก็บ stats ใน Redis
  const pipeline = redis.pipeline();
  
  // นับจำนวน requests ต่อ version ต่อวัน
  pipeline.incr(`stats:api:version:${version}:${date}`);
  pipeline.expire(`stats:api:version:${version}:${date}`, 86400 * 30);
  
  // เก็บ user agents ที่ใช้ version เก่า
  if (version === '1') {
    const userAgent = req.headers['user-agent'] || 'unknown';
    pipeline.sadd(`stats:api:v1:user-agents:${date}`, userAgent);
    pipeline.expire(`stats:api:v1:user-agents:${date}`, 86400 * 7);
  }
  
  pipeline.exec().catch(err => console.error('Version tracking error:', err));
  
  next();
};

// API สำหรับดู version usage stats
const versionStatsRouter = require('express').Router();

versionStatsRouter.get('/', async (req, res) => {
  const versions = ['1', '2', '3'];
  const days = parseInt(req.query.days) || 7;
  const stats = {};
  
  for (const version of versions) {
    stats[`v${version}`] = [];
    
    for (let i = 0; i < days; i++) {
      const date = new Date();
      date.setDate(date.getDate() - i);
      const dateStr = date.toISOString().split('T')[0];
      
      const count = await redis.get(`stats:api:version:${version}:${dateStr}`);
      stats[`v${version}`].push({
        date: dateStr,
        requests: parseInt(count) || 0
      });
    }
    
    stats[`v${version}`].reverse();
  }
  
  res.json({ success: true, data: stats });
});

module.exports = { trackVersionUsage, versionStatsRouter };
```

---

## ขั้นตอนที่ 898: Semantic Versioning

```javascript
// config/semver.js
// การจัดการ Semantic Versioning สำหรับ API

class APIVersion {
  constructor(versionString) {
    const parts = versionString.replace(/^v/, '').split('.');
    this.major = parseInt(parts[0]) || 0;
    this.minor = parseInt(parts[1]) || 0;
    this.patch = parseInt(parts[2]) || 0;
  }

  toString() {
    return `${this.major}.${this.minor}.${this.patch}`;
  }

  isCompatibleWith(other) {
    // Same major version = backward compatible
    const otherVersion = other instanceof APIVersion ? other : new APIVersion(other);
    return this.major === otherVersion.major;
  }

  isNewerThan(other) {
    const otherVersion = other instanceof APIVersion ? other : new APIVersion(other);
    
    if (this.major !== otherVersion.major) return this.major > otherVersion.major;
    if (this.minor !== otherVersion.minor) return this.minor > otherVersion.minor;
    return this.patch > otherVersion.patch;
  }

  // สร้าง Range parser
  static satisfies(version, range) {
    const v = new APIVersion(version);
    
    // Handle ">=2.0.0 <3.0.0" format
    const parts = range.split(' ');
    return parts.every(part => {
      if (part.startsWith('>=')) {
        return !new APIVersion(part.slice(2)).isNewerThan(v);
      }
      if (part.startsWith('>')) {
        return new APIVersion(part.slice(1)).isNewerThan(v) || 
               new APIVersion(part.slice(1)).isCompatibleWith(v);
      }
      if (part.startsWith('<=')) {
        return !v.isNewerThan(new APIVersion(part.slice(2)));
      }
      if (part.startsWith('<')) {
        return new APIVersion(part.slice(1)).isNewerThan(v);
      }
      return version === part;
    });
  }
}

module.exports = APIVersion;
```

---

## ขั้นตอนที่ 899: Documentation ร่วมกับ Versioning

```javascript
// swagger/versionedSwagger.js
const swaggerJsdoc = require('swagger-jsdoc');

const createSwaggerSpec = (version) => {
  return swaggerJsdoc({
    definition: {
      openapi: '3.0.0',
      info: {
        title: `My API v${version}`,
        version,
        description: `API documentation for version ${version}`,
        contact: {
          name: 'API Support',
          email: 'api@example.com'
        }
      },
      servers: [
        {
          url: `http://localhost:3000/api/v${version}`,
          description: `v${version} server`
        }
      ],
      components: {
        securitySchemes: {
          bearerAuth: {
            type: 'http',
            scheme: 'bearer',
            bearerFormat: 'JWT'
          }
        }
      }
    },
    apis: [
      `./routes/v${version}/**/*.js`,
      `./models/*.js`
    ]
  });
};

// Route สำหรับ Swagger UI ตาม version
const setupSwagger = (app) => {
  const swaggerUi = require('swagger-ui-express');
  
  ['1', '2', '3'].forEach(version => {
    const spec = createSwaggerSpec(version);
    app.use(`/docs/v${version}`, swaggerUi.serve, swaggerUi.setup(spec, {
      customSiteTitle: `API v${version} Documentation`
    }));
  });
  
  // Redirect default docs ไปยัง latest version
  app.get('/docs', (req, res) => res.redirect('/docs/v2'));
};

module.exports = { createSwaggerSpec, setupSwagger };
```

---

## ขั้นตอนที่ 900: สรุปและ Best Practices

```javascript
// Best Practices สรุป

/*
  1. ใช้ URL Path versioning สำหรับ public APIs (ชัดเจนและง่ายต่อการ debug)
  2. กำหนด deprecation policy ที่ชัดเจนล่วงหน้า (อย่างน้อย 6 เดือน)
  3. ส่ง deprecation warnings ผ่าน HTTP headers ตาม RFC 8594
  4. เก็บ metrics ว่า clients ใช้ version อะไรบ้าง
  5. ทำ documentation สำหรับทุก version
  6. เขียน migration guide สำหรับ breaking changes
  7. ทดสอบ backward compatibility ก่อน release
*/

// ตัวอย่าง Complete Setup
const setupAPIVersioning = (app) => {
  const { apiVersionMiddleware } = require('./middleware/apiVersioning');
  const { trackVersionUsage, versionStatsRouter } = require('./middleware/versionTracking');
  const deprecationMiddleware = require('./middleware/deprecationWarning');
  const { setupSwagger } = require('./swagger/versionedSwagger');
  const { apiInfoRouter } = require('./config/apiVersions');
  
  // Apply middlewares
  app.use('/api', [
    apiVersionMiddleware,
    trackVersionUsage,
    deprecationMiddleware
  ]);
  
  // Info routes
  app.use('/api/info', apiInfoRouter);
  app.use('/api/stats/versions', versionStatsRouter);
  
  // Documentation
  setupSwagger(app);
  
  // Versioned routes
  app.use('/api/v1', require('./routes/v1'));
  app.use('/api/v2', require('./routes/v2'));
  app.use('/api/v3', require('./routes/v3'));
};

module.exports = setupAPIVersioning;
```

---

## แบบฝึกหัดท้ายบท

### แบบฝึกหัดที่ 1: Implement Version Detection
สร้าง middleware ที่รองรับทั้ง URL path, header, และ query parameter versioning

### แบบฝึกหัดที่ 2: Migration Script
เขียน script ที่ช่วย migrate data format จาก V1 ไป V2

### แบบฝึกหัดที่ 3: Deprecation Dashboard
สร้าง dashboard แสดงว่า clients ยังใช้ deprecated versions อยู่กี่ราย

### แบบฝึกหัดที่ 4: Version Compatibility Test
เขียน automated tests ที่ตรวจสอบว่า response format ตรงกับ version specification

### แบบฝึกหัดที่ 5: Gradual Migration
Implement feature flags ที่ช่วยให้สามารถ rollout new API version แบบ gradual

---

## สรุป

การ API versioning ที่ดีต้องคำนึงถึงทั้ง developer experience ของ client และการบำรุงรักษาระยะยาว การกำหนด deprecation policy ที่ชัดเจนและให้เวลา migration ที่เพียงพอเป็นกุญแจสำคัญ
