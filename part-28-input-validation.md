# Part 28: Input Validation

> ขั้นตอนที่ 28-30 จาก 1000 — การตรวจสอบและ Sanitize ข้อมูล Input ด้วย Joi, Zod และ express-validator

---

## สารบัญ

1. [Joi Validation](#joi-validation)
2. [Zod Validation](#zod-validation)
3. [express-validator](#express-validator)
4. [Custom Validators](#custom-validators)
5. [Sanitization](#sanitization)
6. [Error Formatting](#error-formatting)
7. [Practical: API with Full Validation](#practical-api-with-full-validation)
8. [แบบฝึกหัด](#แบบฝึกหัด)

---

## Joi Validation

### ติดตั้ง

```bash
npm install joi
npm install express-joi-validation  # middleware wrapper
```

### Joi Schema พื้นฐาน

```javascript
// validation/schemas/userSchemas.js
const Joi = require('joi');

// Schema สำหรับ Register
exports.registerSchema = Joi.object({
  name: Joi.string()
    .min(2)
    .max(100)
    .required()
    .messages({
      'string.min': 'ชื่อต้องมีอย่างน้อย {#limit} ตัวอักษร',
      'string.max': 'ชื่อต้องไม่เกิน {#limit} ตัวอักษร',
      'any.required': 'กรุณาระบุชื่อ',
    }),
  
  email: Joi.string()
    .email({ tlds: { allow: false } })
    .required()
    .messages({
      'string.email': 'กรุณาระบุ email ที่ถูกต้อง',
      'any.required': 'กรุณาระบุ email',
    }),
  
  password: Joi.string()
    .min(8)
    .max(128)
    .pattern(/^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)/)
    .required()
    .messages({
      'string.min': 'รหัสผ่านต้องมีอย่างน้อย 8 ตัวอักษร',
      'string.pattern.base': 'รหัสผ่านต้องมีตัวพิมพ์เล็ก พิมพ์ใหญ่ และตัวเลข',
      'any.required': 'กรุณาระบุรหัสผ่าน',
    }),
  
  confirmPassword: Joi.valid(Joi.ref('password'))
    .required()
    .messages({
      'any.only': 'รหัสผ่านไม่ตรงกัน',
      'any.required': 'กรุณายืนยันรหัสผ่าน',
    }),
  
  role: Joi.string()
    .valid('user', 'admin')
    .default('user'),
  
  age: Joi.number()
    .integer()
    .min(0)
    .max(150)
    .optional(),
  
  phone: Joi.string()
    .pattern(/^0[6-9]\d{8}$/)
    .optional()
    .messages({
      'string.pattern.base': 'เบอร์โทรศัพท์ไม่ถูกต้อง',
    }),
});

// Schema สำหรับ Login
exports.loginSchema = Joi.object({
  email: Joi.string().email().required(),
  password: Joi.string().required(),
  rememberMe: Joi.boolean().default(false),
});

// Schema สำหรับ Product
exports.productSchema = Joi.object({
  name: Joi.string().min(3).max(200).required(),
  description: Joi.string().max(5000).optional(),
  price: Joi.number().positive().precision(2).required(),
  comparePrice: Joi.number().positive().precision(2)
    .greater(Joi.ref('price'))
    .optional()
    .messages({
      'number.greater': 'ราคาเปรียบเทียบต้องสูงกว่าราคาจริง',
    }),
  sku: Joi.string()
    .uppercase()
    .pattern(/^[A-Z0-9-]+$/)
    .required(),
  stock: Joi.number().integer().min(0).default(0),
  categoryId: Joi.number().integer().positive().required(),
  images: Joi.array()
    .items(Joi.string().uri())
    .max(10)
    .default([]),
  attributes: Joi.object().default({}),
  isActive: Joi.boolean().default(true),
});

// Schema สำหรับ Query parameters
exports.paginationSchema = Joi.object({
  page: Joi.number().integer().min(1).default(1),
  limit: Joi.number().integer().min(1).max(100).default(10),
  sort: Joi.string().pattern(/^-?[a-zA-Z_]+$/).default('-createdAt'),
  search: Joi.string().max(100).optional(),
});
```

### Joi Middleware

```javascript
// middleware/joiValidate.js
const Joi = require('joi');

/**
 * Validate request ด้วย Joi schema
 * @param {Object} schemas - { body, query, params }
 */
exports.validate = (schemas) => {
  return (req, res, next) => {
    const errors = [];
    
    for (const [source, schema] of Object.entries(schemas)) {
      if (!schema) continue;
      
      const { error, value } = schema.validate(req[source], {
        abortEarly: false,    // รวบรวม errors ทั้งหมด ไม่หยุดที่ error แรก
        allowUnknown: source === 'query',  // อนุญาต query params เพิ่มเติม
        stripUnknown: true,   // ลบ fields ที่ไม่รู้จัก
      });
      
      if (error) {
        errors.push(
          ...error.details.map((d) => ({
            field: d.path.join('.'),
            message: d.message,
            source,
          }))
        );
      } else {
        req[source] = value;  // ใช้ค่าที่ผ่าน validation แล้ว (พร้อม default values)
      }
    }
    
    if (errors.length > 0) {
      return res.status(422).json({
        success: false,
        message: 'ข้อมูลไม่ถูกต้อง',
        errors,
      });
    }
    
    next();
  };
};

// ใช้งาน
// router.post('/users', validate({ body: registerSchema }), userController.create);
// router.get('/products', validate({ query: paginationSchema }), productController.list);
```

### Joi Advanced Features

```javascript
// Custom types
const customJoi = Joi.extend((joi) => ({
  type: 'objectId',
  base: joi.string(),
  messages: {
    'objectId.invalid': 'ID ไม่ถูกต้อง',
  },
  validate(value, helpers) {
    if (!/^[0-9a-fA-F]{24}$/.test(value)) {
      return { value, errors: helpers.error('objectId.invalid') };
    }
  },
}));

// Conditional validation
const orderSchema = Joi.object({
  type: Joi.string().valid('delivery', 'pickup').required(),
  
  address: Joi.when('type', {
    is: 'delivery',
    then: Joi.object({
      street: Joi.string().required(),
      city: Joi.string().required(),
      zip: Joi.string().pattern(/^\d{5}$/).required(),
    }).required(),
    otherwise: Joi.forbidden(),
  }),
  
  pickupTime: Joi.when('type', {
    is: 'pickup',
    then: Joi.date().min('now').required(),
    otherwise: Joi.optional(),
  }),
});

// Array validation
const bulkCreateSchema = Joi.object({
  items: Joi.array()
    .items(
      Joi.object({
        productId: Joi.number().required(),
        quantity: Joi.number().min(1).max(100).required(),
      })
    )
    .min(1)
    .max(50)
    .unique('productId')  // ไม่อนุญาต productId ซ้ำ
    .required(),
});
```

---

## Zod Validation

Zod เป็น TypeScript-first validation library ที่ได้รับความนิยมมาก

```bash
npm install zod
```

### Zod Schema

```javascript
// validation/zodSchemas.js
const { z } = require('zod');

// String validations
const nameSchema = z
  .string()
  .min(2, 'ชื่อต้องมีอย่างน้อย 2 ตัวอักษร')
  .max(100, 'ชื่อต้องไม่เกิน 100 ตัวอักษร')
  .trim();

const emailSchema = z
  .string()
  .email('กรุณาระบุ email ที่ถูกต้อง')
  .toLowerCase();

const passwordSchema = z
  .string()
  .min(8, 'รหัสผ่านต้องมีอย่างน้อย 8 ตัวอักษร')
  .regex(/[a-z]/, 'ต้องมีตัวพิมพ์เล็กอย่างน้อย 1 ตัว')
  .regex(/[A-Z]/, 'ต้องมีตัวพิมพ์ใหญ่อย่างน้อย 1 ตัว')
  .regex(/\d/, 'ต้องมีตัวเลขอย่างน้อย 1 ตัว');

// Register Schema
exports.registerSchema = z
  .object({
    name: nameSchema,
    email: emailSchema,
    password: passwordSchema,
    confirmPassword: z.string(),
    age: z.number().int().min(0).max(150).optional(),
    role: z.enum(['user', 'admin']).default('user'),
  })
  .refine((data) => data.password === data.confirmPassword, {
    message: 'รหัสผ่านไม่ตรงกัน',
    path: ['confirmPassword'],
  });

// Product Schema
exports.productSchema = z.object({
  name: z.string().min(3).max(200),
  description: z.string().max(5000).optional(),
  price: z.number().positive().multipleOf(0.01),
  comparePrice: z.number().positive().optional(),
  sku: z.string().regex(/^[A-Z0-9-]+$/, 'SKU ต้องเป็น A-Z, 0-9, -'),
  stock: z.number().int().min(0).default(0),
  categoryId: z.number().int().positive(),
  images: z.array(z.string().url()).max(10).default([]),
  attributes: z.record(z.unknown()).default({}),
  isActive: z.boolean().default(true),
}).refine(
  (data) => !data.comparePrice || data.comparePrice > data.price,
  {
    message: 'ราคาเปรียบเทียบต้องสูงกว่าราคาจริง',
    path: ['comparePrice'],
  }
);

// Pagination
exports.paginationSchema = z.object({
  page: z.coerce.number().int().min(1).default(1),
  limit: z.coerce.number().int().min(1).max(100).default(10),
  sort: z.string().optional().default('-createdAt'),
  search: z.string().max(100).optional(),
});
```

### Zod Middleware

```javascript
// middleware/zodValidate.js
const { ZodError } = require('zod');

exports.validateZod = (schemas) => {
  return async (req, res, next) => {
    try {
      for (const [source, schema] of Object.entries(schemas)) {
        if (!schema) continue;
        req[source] = await schema.parseAsync(req[source]);
      }
      next();
    } catch (error) {
      if (error instanceof ZodError) {
        return res.status(422).json({
          success: false,
          message: 'ข้อมูลไม่ถูกต้อง',
          errors: error.errors.map((e) => ({
            field: e.path.join('.'),
            message: e.message,
            code: e.code,
          })),
        });
      }
      next(error);
    }
  };
};
```

---

## express-validator

```bash
npm install express-validator
```

### Chain Validators

```javascript
// middleware/validators/authValidators.js
const { body, query, param, validationResult } = require('express-validator');

// Validator chains
exports.registerValidation = [
  body('name')
    .notEmpty().withMessage('กรุณาระบุชื่อ')
    .isLength({ min: 2, max: 100 }).withMessage('ชื่อต้องมี 2-100 ตัวอักษร')
    .trim()
    .escape(),  // HTML encode ป้องกัน XSS
  
  body('email')
    .notEmpty().withMessage('กรุณาระบุ email')
    .isEmail().withMessage('รูปแบบ email ไม่ถูกต้อง')
    .normalizeEmail()
    .toLowerCase(),
  
  body('password')
    .notEmpty().withMessage('กรุณาระบุรหัสผ่าน')
    .isLength({ min: 8, max: 128 }).withMessage('รหัสผ่านต้องมี 8-128 ตัวอักษร')
    .isStrongPassword({
      minLowercase: 1,
      minUppercase: 1,
      minNumbers: 1,
      minSymbols: 0,
    }).withMessage('รหัสผ่านต้องมีตัวพิมพ์เล็ก พิมพ์ใหญ่ และตัวเลข'),
  
  body('confirmPassword')
    .notEmpty().withMessage('กรุณายืนยันรหัสผ่าน')
    .custom((value, { req }) => {
      if (value !== req.body.password) {
        throw new Error('รหัสผ่านไม่ตรงกัน');
      }
      return true;
    }),
];

exports.loginValidation = [
  body('email').notEmpty().isEmail().normalizeEmail(),
  body('password').notEmpty().withMessage('กรุณาระบุรหัสผ่าน'),
];

exports.productValidation = [
  body('name')
    .notEmpty().withMessage('กรุณาระบุชื่อสินค้า')
    .isLength({ min: 3, max: 200 })
    .trim(),
  
  body('price')
    .notEmpty().withMessage('กรุณาระบุราคา')
    .isFloat({ min: 0 }).withMessage('ราคาต้องไม่ติดลบ')
    .toFloat(),
  
  body('sku')
    .notEmpty().withMessage('กรุณาระบุ SKU')
    .matches(/^[A-Z0-9-]+$/).withMessage('SKU ต้องเป็น A-Z, 0-9, -')
    .toUpperCase(),
  
  body('stock')
    .optional()
    .isInt({ min: 0 }).withMessage('สต็อกต้องไม่ติดลบ')
    .toInt(),
  
  body('images')
    .optional()
    .isArray({ max: 10 }).withMessage('รูปภาพสูงสุด 10 ไฟล์'),
  
  body('images.*')
    .optional()
    .isURL().withMessage('URL รูปภาพไม่ถูกต้อง'),
];

// Pagination validators
exports.paginationValidation = [
  query('page').optional().isInt({ min: 1 }).toInt().default(1),
  query('limit').optional().isInt({ min: 1, max: 100 }).toInt().default(10),
  query('sort').optional().matches(/^-?[a-zA-Z_]+$/),
];

// ObjectId validator
exports.validateId = [
  param('id')
    .isMongoId().withMessage('ID ไม่ถูกต้อง'),
];

// Middleware ตรวจสอบ validation results
exports.handleValidationErrors = (req, res, next) => {
  const errors = validationResult(req);
  
  if (!errors.isEmpty()) {
    return res.status(422).json({
      success: false,
      message: 'ข้อมูลไม่ถูกต้อง',
      errors: errors.array().map((err) => ({
        field: err.path,
        message: err.msg,
        value: err.value,
      })),
    });
  }
  
  next();
};
```

---

## Custom Validators

```javascript
// validation/customValidators.js
const mongoose = require('mongoose');

/**
 * ตรวจสอบว่า ObjectId มีอยู่ใน collection
 */
exports.existsInDB = (Model) => async (value, { req }) => {
  const doc = await Model.findById(value);
  if (!doc) {
    throw new Error(`ไม่พบข้อมูลที่มี ID: ${value}`);
  }
  return true;
};

/**
 * ตรวจสอบ unique field
 */
exports.isUnique = (Model, field, excludeId = null) =>
  async (value, { req }) => {
    const query = { [field]: value };
    if (excludeId) {
      query._id = { $ne: excludeId || req.params.id };
    }
    const exists = await Model.findOne(query);
    if (exists) {
      throw new Error(`${field} นี้มีอยู่แล้ว`);
    }
    return true;
  };

/**
 * ตรวจสอบเบอร์โทรไทย
 */
exports.isThaPhone = (value) => {
  if (!value) return true;
  const cleaned = value.replace(/[-\s]/g, '');
  return /^(0[6-9]\d{8}|0[2-8]\d{7}|\+66[6-9]\d{8})$/.test(cleaned);
};

/**
 * ตรวจสอบรหัสไปรษณีย์ไทย
 */
exports.isThaiZip = (value) => /^\d{5}$/.test(value);

/**
 * ตรวจสอบ Thai National ID
 */
exports.isThaiNationalId = (value) => {
  if (!/^\d{13}$/.test(value)) return false;
  
  let sum = 0;
  for (let i = 0; i < 12; i++) {
    sum += parseInt(value[i]) * (13 - i);
  }
  const checkDigit = (11 - (sum % 11)) % 10;
  return checkDigit === parseInt(value[12]);
};
```

---

## Sanitization

```javascript
// utils/sanitize.js
const DOMPurify = require('isomorphic-dompurify');
const he = require('he');  // npm install he

/**
 * ลบ HTML tags ทั้งหมด
 */
exports.stripHtml = (value) => {
  if (typeof value !== 'string') return value;
  return value.replace(/<[^>]*>/g, '').trim();
};

/**
 * ทำให้ HTML ปลอดภัย (เก็บ tags บางอย่างได้)
 */
exports.sanitizeHtml = (value) => {
  return DOMPurify.sanitize(value, {
    ALLOWED_TAGS: ['p', 'br', 'b', 'i', 'em', 'strong', 'ul', 'ol', 'li', 'a'],
    ALLOWED_ATTR: ['href', 'target'],
  });
};

/**
 * Escape HTML entities
 */
exports.escapeHtml = (value) => {
  if (typeof value !== 'string') return value;
  return he.escape(value);
};

/**
 * Sanitize object recursively
 */
exports.sanitizeObject = (obj, options = {}) => {
  if (typeof obj !== 'object' || obj === null) {
    if (typeof obj === 'string') {
      return options.stripHtml ? exports.stripHtml(obj) : obj.trim();
    }
    return obj;
  }
  
  if (Array.isArray(obj)) {
    return obj.map((item) => exports.sanitizeObject(item, options));
  }
  
  return Object.fromEntries(
    Object.entries(obj).map(([key, value]) => [
      key,
      exports.sanitizeObject(value, options),
    ])
  );
};

/**
 * Prevent MongoDB injection
 */
exports.sanitizeMongoQuery = (obj) => {
  if (typeof obj !== 'object' || obj === null) return obj;
  
  for (const key of Object.keys(obj)) {
    if (key.startsWith('$')) {
      delete obj[key];
    } else if (typeof obj[key] === 'object') {
      exports.sanitizeMongoQuery(obj[key]);
    }
  }
  
  return obj;
};

// Middleware
exports.sanitizeBody = (req, res, next) => {
  if (req.body) {
    req.body = exports.sanitizeObject(req.body, { stripHtml: true });
    req.body = exports.sanitizeMongoQuery(req.body);
  }
  next();
};
```

---

## Error Formatting

### Standardized Error Response

```javascript
// utils/validationErrorFormatter.js

/**
 * แปลง Joi errors เป็น standard format
 */
exports.formatJoiErrors = (error) => {
  return error.details.map((detail) => ({
    field: detail.path.join('.') || 'unknown',
    message: detail.message.replace(/['"]/g, ''),
    code: detail.type,
  }));
};

/**
 * แปลง Zod errors เป็น standard format
 */
exports.formatZodErrors = (error) => {
  return error.errors.map((e) => ({
    field: e.path.join('.') || 'unknown',
    message: e.message,
    code: e.code,
  }));
};

/**
 * แปลง express-validator errors
 */
exports.formatExpressValidatorErrors = (errors) => {
  return errors.array().map((e) => ({
    field: e.path || e.param,
    message: e.msg,
    value: process.env.NODE_ENV !== 'production' ? e.value : undefined,
  }));
};

/**
 * Validation error response
 */
exports.validationErrorResponse = (res, errors) => {
  return res.status(422).json({
    success: false,
    message: 'ข้อมูลไม่ถูกต้อง',
    errors,
  });
};
```

---

## Practical: API with Full Validation

### Routes พร้อม Validation

```javascript
// routes/products.js
const express = require('express');
const router = express.Router();
const { authenticate, requireRole } = require('../middleware/auth');
const { validate } = require('../middleware/joiValidate');
const { productSchema, paginationSchema } = require('../validation/schemas/productSchemas');
const { sanitizeBody } = require('../utils/sanitize');
const productController = require('../controllers/productController');

// GET /products — list พร้อม pagination
router.get(
  '/',
  validate({ query: paginationSchema }),
  productController.getProducts
);

// GET /products/:id
router.get(
  '/:id',
  validate({
    params: Joi.object({ id: Joi.string().pattern(/^[0-9a-fA-F]{24}$/).required() }),
  }),
  productController.getProduct
);

// POST /products — สร้าง product (admin only)
router.post(
  '/',
  authenticate,
  requireRole('admin'),
  sanitizeBody,
  validate({ body: productSchema }),
  productController.createProduct
);

// PUT /products/:id — อัปเดต
router.put(
  '/:id',
  authenticate,
  requireRole('admin'),
  sanitizeBody,
  validate({ body: productSchema.fork(['name', 'price', 'sku', 'categoryId'], (s) => s.optional()) }),
  productController.updateProduct
);

module.exports = router;
```

### Controller พร้อม Validation

```javascript
// controllers/productController.js
const Product = require('../models/Product');
const Category = require('../models/Category');
const { AppError } = require('../utils/errors');

exports.createProduct = async (req, res, next) => {
  try {
    const { name, price, sku, categoryId, ...rest } = req.body;
    
    // ตรวจสอบ category มีอยู่
    const category = await Category.findById(categoryId);
    if (!category) {
      return next(new AppError('ไม่พบหมวดหมู่สินค้า', 404));
    }
    
    // ตรวจสอบ SKU ซ้ำ
    const existing = await Product.findOne({ sku });
    if (existing) {
      return res.status(400).json({
        success: false,
        errors: [{ field: 'sku', message: 'SKU นี้มีอยู่แล้ว' }],
      });
    }
    
    const product = await Product.create({ name, price, sku, categoryId, ...rest });
    
    res.status(201).json({
      success: true,
      data: product,
    });
  } catch (error) {
    next(error);
  }
};
```

---

## แบบฝึกหัด

### Exercise 1: Form Validation Library

สร้าง validation middleware ที่รองรับ:
1. Required fields
2. Custom error messages ภาษาไทย
3. Nested object validation
4. Array validation
5. Cross-field validation (เช่น confirmPassword)

### Exercise 2: API Schema Versioning

1. สร้าง schemas แยกตาม version (v1, v2)
2. Middleware ตรวจสอบ version จาก header
3. ใช้ schema ที่เหมาะสม
4. Document API changes

### Exercise 3: Runtime Type Checking

1. สร้าง TypeScript-like runtime checks
2. Validate response data ก่อนส่งกลับ
3. Log validation failures
4. Integration tests สำหรับ validation

---

## สรุป

- **Joi** — mature, feature-rich validation
- **Zod** — TypeScript-first, type inference
- **express-validator** — chain-style, ยืดหยุ่น
- **Sanitization** — ทำ input ปลอดภัยก่อนใช้
- **Consistent error format** — UX ที่ดีสำหรับ client

> **บทถัดไป:** Part 29 — Advanced Error Handling
