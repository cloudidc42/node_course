# Part 17: Form Handling
## ขั้นตอนที่ 17-7 จาก 1000

---

## สารบัญ
1. HTML Forms
2. body-parser
3. express.urlencoded()
4. express.json()
5. Form Validation
6. File Uploads with Multer
7. Practical: User Registration Form
8. Exercise

---

## 1. HTML Forms

HTML Forms ส่งข้อมูลไปยัง server ใน 2 รูปแบบ:

### Form Encoding Types

```html
<!-- 1. application/x-www-form-urlencoded (default) -->
<!-- ข้อมูลถูก encode เป็น key=value&key2=value2 -->
<form action="/submit" method="POST">
  <input type="text" name="username" value="สมชาย">
  <input type="email" name="email" value="test@example.com">
  <button type="submit">ส่ง</button>
</form>
<!-- ส่ง: username=%E0%B8%AA%E0%B8%A1%E0%B8%8A%E0%B8%B2%E0%B8%A2&email=test%40example.com -->

<!-- 2. multipart/form-data -->
<!-- ใช้เมื่อต้องการ upload ไฟล์ -->
<form action="/upload" method="POST" enctype="multipart/form-data">
  <input type="text" name="username">
  <input type="file" name="avatar">
  <button type="submit">ส่ง</button>
</form>

<!-- 3. text/plain (ไม่ค่อยใช้) -->
<form action="/submit" method="POST" enctype="text/plain">
  <input type="text" name="data">
</form>
```

### Form Methods

```html
<!-- GET - ข้อมูลไปอยู่ใน URL query string -->
<!-- ใช้สำหรับ: search, filter, navigation -->
<form action="/search" method="GET">
  <input type="text" name="q" placeholder="ค้นหา...">
  <input type="hidden" name="page" value="1">
  <button type="submit">ค้นหา</button>
</form>
<!-- URL: /search?q=express&page=1 -->

<!-- POST - ข้อมูลอยู่ใน body -->
<!-- ใช้สำหรับ: create, update, login -->
<form action="/users" method="POST">
  <input type="text" name="name">
  <input type="email" name="email">
  <button type="submit">สมัคร</button>
</form>
```

### ตัวอย่าง Form สมบูรณ์

```html
<!DOCTYPE html>
<html lang="th">
<head>
  <meta charset="UTF-8">
  <title>User Registration</title>
</head>
<body>
  <form action="/register" method="POST" id="registrationForm">
    <!-- Text inputs -->
    <div>
      <label for="firstName">ชื่อ:</label>
      <input type="text" id="firstName" name="firstName" required minlength="2">
    </div>
    
    <div>
      <label for="lastName">นามสกุล:</label>
      <input type="text" id="lastName" name="lastName" required>
    </div>
    
    <!-- Email -->
    <div>
      <label for="email">อีเมล:</label>
      <input type="email" id="email" name="email" required>
    </div>
    
    <!-- Password -->
    <div>
      <label for="password">รหัสผ่าน:</label>
      <input type="password" id="password" name="password" 
             required minlength="8" pattern="(?=.*\d)(?=.*[a-z])(?=.*[A-Z]).{8,}">
      <small>ต้องมีตัวเลข ตัวพิมพ์เล็ก และตัวพิมพ์ใหญ่</small>
    </div>
    
    <!-- Number -->
    <div>
      <label for="age">อายุ:</label>
      <input type="number" id="age" name="age" min="18" max="120">
    </div>
    
    <!-- Date -->
    <div>
      <label for="birthdate">วันเกิด:</label>
      <input type="date" id="birthdate" name="birthdate">
    </div>
    
    <!-- Select -->
    <div>
      <label for="gender">เพศ:</label>
      <select id="gender" name="gender">
        <option value="">-- เลือก --</option>
        <option value="male">ชาย</option>
        <option value="female">หญิง</option>
        <option value="other">อื่นๆ</option>
      </select>
    </div>
    
    <!-- Radio buttons -->
    <div>
      <label>ระดับความสามารถ:</label>
      <label><input type="radio" name="level" value="beginner"> ผู้เริ่มต้น</label>
      <label><input type="radio" name="level" value="intermediate"> ปานกลาง</label>
      <label><input type="radio" name="level" value="advanced"> ขั้นสูง</label>
    </div>
    
    <!-- Checkboxes -->
    <div>
      <label>ความสนใจ:</label>
      <label><input type="checkbox" name="interests" value="nodejs"> Node.js</label>
      <label><input type="checkbox" name="interests" value="react"> React</label>
      <label><input type="checkbox" name="interests" value="database"> Database</label>
    </div>
    
    <!-- Textarea -->
    <div>
      <label for="bio">เกี่ยวกับตัวเอง:</label>
      <textarea id="bio" name="bio" rows="4" maxlength="500"></textarea>
    </div>
    
    <!-- Checkbox (agree) -->
    <div>
      <label>
        <input type="checkbox" name="agree" value="yes" required>
        ฉันยอมรับ <a href="/terms">ข้อตกลงการใช้บริการ</a>
      </label>
    </div>
    
    <button type="submit">สมัครสมาชิก</button>
  </form>
</body>
</html>
```

---

## 2. body-parser

`body-parser` เป็น package เก่าที่ปัจจุบัน built-in ใน Express แล้ว:

```javascript
// วิธีเก่า (ก่อน Express 4.16)
const bodyParser = require('body-parser');
app.use(bodyParser.json());
app.use(bodyParser.urlencoded({ extended: true }));

// วิธีใหม่ (Express 4.16+) - แนะนำ
const express = require('express');
const app = express();
app.use(express.json());
app.use(express.urlencoded({ extended: true }));
```

---

## 3. express.urlencoded()

ใช้สำหรับแปลงข้อมูลจาก HTML form ที่ส่งมาเป็น `application/x-www-form-urlencoded`:

```javascript
const express = require('express');
const app = express();

// Options
app.use(express.urlencoded({
  extended: true,    // ใช้ qs library (รองรับ nested objects)
  // extended: false // ใช้ querystring library (simpler)
  limit: '1mb',      // ขนาดสูงสุดของ body
  parameterLimit: 1000 // จำนวน parameters สูงสุด
}));
```

### ความแตกต่างระหว่าง extended: true vs false

```javascript
// extended: true (qs library) - รองรับ nested objects
// Form: user[name]=John&user[age]=25&hobbies[]=reading&hobbies[]=coding
app.use(express.urlencoded({ extended: true }));
// req.body = {
//   user: { name: 'John', age: '25' },
//   hobbies: ['reading', 'coding']
// }

// extended: false (querystring library) - แบนๆ
app.use(express.urlencoded({ extended: false }));
// req.body = {
//   'user[name]': 'John',
//   'user[age]': '25',
//   'hobbies[]': ['reading', 'coding']
// }
```

### รับข้อมูลจาก HTML Form

```javascript
// Route สำหรับแสดง form
app.get('/register', (req, res) => {
  res.render('register');
});

// Route สำหรับรับข้อมูลจาก form
app.post('/register', (req, res) => {
  console.log(req.body);
  // {
  //   firstName: 'สมชาย',
  //   lastName: 'ใจดี',
  //   email: 'somchai@example.com',
  //   password: 'Password123',
  //   age: '25',
  //   gender: 'male',
  //   level: 'intermediate',
  //   interests: ['nodejs', 'react'],  // checkboxes
  //   bio: 'นักพัฒนา Node.js',
  //   agree: 'yes'
  // }
  
  const { firstName, lastName, email } = req.body;
  
  // ค่าทุกอย่างเป็น string! ต้องแปลงเอง
  const age = parseInt(req.body.age);
  const interests = Array.isArray(req.body.interests) 
    ? req.body.interests 
    : req.body.interests ? [req.body.interests] : [];
  
  res.send(`ยินดีต้อนรับ ${firstName} ${lastName}!`);
});
```

---

## 4. express.json()

ใช้สำหรับแปลงข้อมูลที่ส่งมาเป็น JSON (`Content-Type: application/json`):

```javascript
app.use(express.json({
  limit: '10mb',    // ขนาดสูงสุด (default: '100kb')
  strict: true,     // รับเฉพาะ objects และ arrays (default: true)
  type: 'application/json'  // content-type ที่ต้องการ parse
}));

// รับ JSON จาก API client
app.post('/api/users', (req, res) => {
  console.log(req.body);
  // {
  //   name: 'สมชาย',
  //   email: 'somchai@example.com',
  //   age: 25,        // number (ไม่ใช่ string)
  //   active: true    // boolean
  // }
  
  res.json({ message: 'received', data: req.body });
});
```

### รับทั้ง JSON และ Form Data

```javascript
// รองรับทั้งสองรูปแบบ
app.use(express.json());
app.use(express.urlencoded({ extended: true }));

app.post('/submit', (req, res) => {
  // req.body จะมีข้อมูลไม่ว่าจะส่งมาแบบไหน
  const { name, email } = req.body;
  res.json({ name, email });
});
```

---

## 5. Form Validation

### Manual Validation

```javascript
const express = require('express');
const app = express();
app.use(express.urlencoded({ extended: true }));
app.use(express.json());

// Validation helpers
const validators = {
  required: (value) => {
    if (value === undefined || value === null) return false;
    if (typeof value === 'string') return value.trim() !== '';
    return true;
  },
  
  email: (value) => {
    const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
    return emailRegex.test(value);
  },
  
  minLength: (value, min) => {
    return typeof value === 'string' && value.length >= min;
  },
  
  maxLength: (value, max) => {
    return typeof value === 'string' && value.length <= max;
  },
  
  isNumber: (value) => {
    return !isNaN(parseFloat(value)) && isFinite(value);
  },
  
  isInRange: (value, min, max) => {
    const num = parseFloat(value);
    return num >= min && num <= max;
  },
  
  matches: (value, regex) => {
    return new RegExp(regex).test(value);
  }
};

// Validate registration form
const validateRegistration = (data) => {
  const errors = {};
  
  // firstName
  if (!validators.required(data.firstName)) {
    errors.firstName = 'กรุณาระบุชื่อ';
  } else if (!validators.minLength(data.firstName, 2)) {
    errors.firstName = 'ชื่อต้องมีอย่างน้อย 2 ตัวอักษร';
  }
  
  // lastName
  if (!validators.required(data.lastName)) {
    errors.lastName = 'กรุณาระบุนามสกุล';
  }
  
  // email
  if (!validators.required(data.email)) {
    errors.email = 'กรุณาระบุอีเมล';
  } else if (!validators.email(data.email)) {
    errors.email = 'รูปแบบอีเมลไม่ถูกต้อง';
  }
  
  // password
  if (!validators.required(data.password)) {
    errors.password = 'กรุณาระบุรหัสผ่าน';
  } else if (!validators.minLength(data.password, 8)) {
    errors.password = 'รหัสผ่านต้องมีอย่างน้อย 8 ตัวอักษร';
  } else if (!validators.matches(data.password, /(?=.*\d)(?=.*[a-z])(?=.*[A-Z])/)) {
    errors.password = 'รหัสผ่านต้องมีตัวเลข ตัวพิมพ์เล็ก และตัวพิมพ์ใหญ่';
  }
  
  // age (optional)
  if (data.age && !validators.isNumber(data.age)) {
    errors.age = 'อายุต้องเป็นตัวเลข';
  } else if (data.age && !validators.isInRange(data.age, 18, 120)) {
    errors.age = 'อายุต้องอยู่ระหว่าง 18-120';
  }
  
  return {
    isValid: Object.keys(errors).length === 0,
    errors
  };
};

app.post('/register', (req, res) => {
  const { isValid, errors } = validateRegistration(req.body);
  
  if (!isValid) {
    return res.status(400).json({
      success: false,
      errors
    });
  }
  
  // บันทึกข้อมูล...
  res.json({ success: true, message: 'สมัครสมาชิกสำเร็จ' });
});
```

### Validation ด้วย Joi

```bash
npm install joi
```

```javascript
const Joi = require('joi');

// สร้าง schema
const registerSchema = Joi.object({
  firstName: Joi.string()
    .min(2)
    .max(50)
    .required()
    .messages({
      'string.empty': 'กรุณาระบุชื่อ',
      'string.min': 'ชื่อต้องมีอย่างน้อย {#limit} ตัวอักษร',
      'any.required': 'กรุณาระบุชื่อ'
    }),
  
  lastName: Joi.string()
    .min(2)
    .max(50)
    .required()
    .messages({
      'string.empty': 'กรุณาระบุนามสกุล',
      'any.required': 'กรุณาระบุนามสกุล'
    }),
  
  email: Joi.string()
    .email({ tlds: { allow: false } })
    .required()
    .messages({
      'string.email': 'รูปแบบอีเมลไม่ถูกต้อง',
      'any.required': 'กรุณาระบุอีเมล'
    }),
  
  password: Joi.string()
    .min(8)
    .pattern(/^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)/)
    .required()
    .messages({
      'string.min': 'รหัสผ่านต้องมีอย่างน้อย {#limit} ตัวอักษร',
      'string.pattern.base': 'รหัสผ่านต้องมีตัวเลข ตัวพิมพ์เล็ก และตัวพิมพ์ใหญ่',
      'any.required': 'กรุณาระบุรหัสผ่าน'
    }),
  
  age: Joi.number()
    .integer()
    .min(18)
    .max(120)
    .optional(),
  
  gender: Joi.string()
    .valid('male', 'female', 'other')
    .optional(),
  
  interests: Joi.alternatives().try(
    Joi.array().items(Joi.string()),
    Joi.string()
  ).optional(),
  
  bio: Joi.string().max(500).optional(),
  
  agree: Joi.string().valid('yes').required()
    .messages({ 'any.only': 'กรุณายอมรับข้อตกลง' })
});

// Validation middleware
const validate = (schema) => async (req, res, next) => {
  try {
    await schema.validateAsync(req.body, {
      abortEarly: false, // แสดงทุก error ไม่หยุดที่แรก
      stripUnknown: true // ลบ fields ที่ไม่ได้กำหนดใน schema
    });
    next();
  } catch (err) {
    const errors = {};
    err.details.forEach(detail => {
      const key = detail.path.join('.');
      errors[key] = detail.message;
    });
    
    res.status(400).json({
      success: false,
      message: 'ข้อมูลไม่ถูกต้อง',
      errors
    });
  }
};

// ใช้งาน
app.post('/register', validate(registerSchema), async (req, res) => {
  // ถ้าถึงตรงนี้ แปลงว่า validation ผ่านแล้ว
  const userData = req.body;
  
  // บันทึกข้อมูล...
  res.status(201).json({
    success: true,
    message: 'สมัครสมาชิกสำเร็จ'
  });
});
```

---

## 6. File Uploads with Multer

```bash
npm install multer
```

### Upload รูปภาพ Profile

```javascript
const multer = require('multer');
const path = require('path');
const fs = require('fs');
const sharp = require('sharp'); // npm install sharp

// สร้างโฟลเดอร์ถ้ายังไม่มี
const uploadDir = 'uploads/avatars';
if (!fs.existsSync(uploadDir)) {
  fs.mkdirSync(uploadDir, { recursive: true });
}

// Config multer
const storage = multer.memoryStorage(); // เก็บใน memory ก่อน

const fileFilter = (req, file, cb) => {
  const allowedTypes = ['image/jpeg', 'image/jpg', 'image/png', 'image/webp'];
  
  if (allowedTypes.includes(file.mimetype)) {
    cb(null, true);
  } else {
    cb(new Error('อนุญาตเฉพาะไฟล์รูปภาพ jpg, png, webp'), false);
  }
};

const upload = multer({
  storage,
  fileFilter,
  limits: {
    fileSize: 5 * 1024 * 1024, // 5MB
    files: 1
  }
});

// Upload และ resize รูปภาพ
app.post('/upload-avatar',
  upload.single('avatar'),
  async (req, res) => {
    if (!req.file) {
      return res.status(400).json({ error: 'กรุณาเลือกรูปภาพ' });
    }
    
    try {
      const filename = `avatar-${Date.now()}.webp`;
      const outputPath = path.join(uploadDir, filename);
      
      // Resize และแปลงเป็น WebP
      await sharp(req.file.buffer)
        .resize(200, 200, {
          fit: 'cover',
          position: 'center'
        })
        .webp({ quality: 80 })
        .toFile(outputPath);
      
      res.json({
        success: true,
        url: `/uploads/avatars/${filename}`
      });
    } catch (err) {
      console.error('Image processing error:', err);
      res.status(500).json({ error: 'เกิดข้อผิดพลาดในการประมวลผลรูปภาพ' });
    }
  }
);
```

---

## 7. Practical: User Registration Form

### โครงสร้างไฟล์

```
registration-app/
├── public/
│   └── css/
│       └── style.css
├── views/
│   ├── register.ejs
│   └── success.ejs
├── uploads/
│   └── avatars/
└── app.js
```

### views/register.ejs

```html
<!DOCTYPE html>
<html lang="th">
<head>
  <meta charset="UTF-8">
  <title>สมัครสมาชิก</title>
  <style>
    * { box-sizing: border-box; margin: 0; padding: 0; }
    body { font-family: 'Sarabun', sans-serif; background: #f0f2f5; }
    .container { max-width: 600px; margin: 2rem auto; padding: 0 1rem; }
    .card { background: white; border-radius: 1rem; padding: 2rem; box-shadow: 0 2px 10px rgba(0,0,0,0.1); }
    h1 { color: #1e40af; margin-bottom: 2rem; text-align: center; }
    .form-group { margin-bottom: 1.25rem; }
    label { display: block; margin-bottom: 0.5rem; font-weight: 600; color: #374151; }
    input, select, textarea {
      width: 100%; padding: 0.75rem; border: 1px solid #d1d5db;
      border-radius: 0.5rem; font-size: 1rem; font-family: inherit;
      transition: border-color 0.2s;
    }
    input:focus, select:focus, textarea:focus {
      outline: none; border-color: #2563eb; box-shadow: 0 0 0 3px rgba(37,99,235,0.1);
    }
    .error { color: #dc2626; font-size: 0.875rem; margin-top: 0.25rem; }
    .btn {
      width: 100%; padding: 0.875rem; background: #2563eb; color: white;
      border: none; border-radius: 0.5rem; font-size: 1rem; font-weight: 600;
      cursor: pointer; margin-top: 1rem; transition: background 0.2s;
    }
    .btn:hover { background: #1d4ed8; }
    .row { display: grid; grid-template-columns: 1fr 1fr; gap: 1rem; }
    .checkbox-group { display: flex; gap: 1rem; flex-wrap: wrap; }
    .checkbox-label { display: flex; align-items: center; gap: 0.5rem; cursor: pointer; }
    .alert { padding: 1rem; border-radius: 0.5rem; margin-bottom: 1rem; }
    .alert-error { background: #fef2f2; border: 1px solid #fecaca; color: #dc2626; }
    .avatar-preview { width: 100px; height: 100px; border-radius: 50%; object-fit: cover; display: none; }
  </style>
</head>
<body>
  <div class="container">
    <div class="card">
      <h1>สมัครสมาชิก</h1>
      
      <% if (typeof errors !== 'undefined' && Object.keys(errors).length > 0) { %>
        <div class="alert alert-error">
          <strong>กรุณาตรวจสอบข้อมูล:</strong>
          <ul>
            <% Object.values(errors).forEach(error => { %>
              <li><%= error %></li>
            <% }) %>
          </ul>
        </div>
      <% } %>
      
      <form action="/register" method="POST" enctype="multipart/form-data" novalidate>
        <!-- Avatar upload -->
        <div class="form-group">
          <label>รูปโปรไฟล์ (ไม่บังคับ)</label>
          <input type="file" name="avatar" accept="image/*" id="avatarInput">
          <img id="avatarPreview" class="avatar-preview" src="" alt="Preview">
          <% if (typeof errors !== 'undefined' && errors.avatar) { %>
            <span class="error"><%= errors.avatar %></span>
          <% } %>
        </div>
        
        <!-- Name -->
        <div class="row">
          <div class="form-group">
            <label for="firstName">ชื่อ *</label>
            <input type="text" id="firstName" name="firstName" 
                   value="<%= typeof formData !== 'undefined' ? formData.firstName || '' : '' %>"
                   required minlength="2">
            <% if (typeof errors !== 'undefined' && errors.firstName) { %>
              <span class="error"><%= errors.firstName %></span>
            <% } %>
          </div>
          
          <div class="form-group">
            <label for="lastName">นามสกุล *</label>
            <input type="text" id="lastName" name="lastName"
                   value="<%= typeof formData !== 'undefined' ? formData.lastName || '' : '' %>"
                   required>
            <% if (typeof errors !== 'undefined' && errors.lastName) { %>
              <span class="error"><%= errors.lastName %></span>
            <% } %>
          </div>
        </div>
        
        <!-- Email -->
        <div class="form-group">
          <label for="email">อีเมล *</label>
          <input type="email" id="email" name="email"
                 value="<%= typeof formData !== 'undefined' ? formData.email || '' : '' %>"
                 required>
          <% if (typeof errors !== 'undefined' && errors.email) { %>
            <span class="error"><%= errors.email %></span>
          <% } %>
        </div>
        
        <!-- Password -->
        <div class="row">
          <div class="form-group">
            <label for="password">รหัสผ่าน *</label>
            <input type="password" id="password" name="password" required minlength="8">
            <% if (typeof errors !== 'undefined' && errors.password) { %>
              <span class="error"><%= errors.password %></span>
            <% } %>
          </div>
          
          <div class="form-group">
            <label for="confirmPassword">ยืนยันรหัสผ่าน *</label>
            <input type="password" id="confirmPassword" name="confirmPassword" required>
            <% if (typeof errors !== 'undefined' && errors.confirmPassword) { %>
              <span class="error"><%= errors.confirmPassword %></span>
            <% } %>
          </div>
        </div>
        
        <!-- Gender -->
        <div class="form-group">
          <label for="gender">เพศ</label>
          <select id="gender" name="gender">
            <option value="">-- เลือก --</option>
            <option value="male" <%= typeof formData !== 'undefined' && formData.gender === 'male' ? 'selected' : '' %>>ชาย</option>
            <option value="female" <%= typeof formData !== 'undefined' && formData.gender === 'female' ? 'selected' : '' %>>หญิง</option>
            <option value="other" <%= typeof formData !== 'undefined' && formData.gender === 'other' ? 'selected' : '' %>>อื่นๆ</option>
          </select>
        </div>
        
        <!-- Bio -->
        <div class="form-group">
          <label for="bio">เกี่ยวกับตัวเอง</label>
          <textarea id="bio" name="bio" rows="3" maxlength="500"
                    placeholder="บอกเล่าเกี่ยวกับตัวเองสักเล็กน้อย..."><%= typeof formData !== 'undefined' ? formData.bio || '' : '' %></textarea>
        </div>
        
        <!-- Agree -->
        <div class="form-group">
          <label class="checkbox-label">
            <input type="checkbox" name="agree" value="yes">
            ฉันยอมรับ <a href="/terms" target="_blank">ข้อตกลงการใช้บริการ</a>
          </label>
          <% if (typeof errors !== 'undefined' && errors.agree) { %>
            <span class="error"><%= errors.agree %></span>
          <% } %>
        </div>
        
        <button type="submit" class="btn">สมัครสมาชิก</button>
      </form>
    </div>
  </div>
  
  <script>
    // Preview avatar
    document.getElementById('avatarInput').addEventListener('change', function() {
      const file = this.files[0];
      if (file) {
        const reader = new FileReader();
        reader.onload = (e) => {
          const preview = document.getElementById('avatarPreview');
          preview.src = e.target.result;
          preview.style.display = 'block';
        };
        reader.readAsDataURL(file);
      }
    });
  </script>
</body>
</html>
```

### app.js - Registration System

```javascript
// app.js
const express = require('express');
const multer = require('multer');
const path = require('path');
const fs = require('fs');
const app = express();

app.set('view engine', 'ejs');
app.set('views', './views');
app.use(express.urlencoded({ extended: true }));
app.use('/uploads', express.static('uploads'));

// Multer config
const upload = multer({
  storage: multer.diskStorage({
    destination: (req, file, cb) => cb(null, 'uploads/avatars/'),
    filename: (req, file, cb) => {
      const ext = path.extname(file.originalname);
      cb(null, `avatar-${Date.now()}${ext}`);
    }
  }),
  fileFilter: (req, file, cb) => {
    const allowed = ['image/jpeg', 'image/png', 'image/webp'];
    allowed.includes(file.mimetype) ? cb(null, true) : cb(new Error('ไม่รองรับไฟล์นี้'));
  },
  limits: { fileSize: 2 * 1024 * 1024 }
});

// In-memory users store
const users = [];

// Validation function
const validateRegistration = (data) => {
  const errors = {};
  
  if (!data.firstName || data.firstName.trim().length < 2)
    errors.firstName = 'ชื่อต้องมีอย่างน้อย 2 ตัวอักษร';
  
  if (!data.lastName || data.lastName.trim().length < 1)
    errors.lastName = 'กรุณาระบุนามสกุล';
  
  if (!data.email || !/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(data.email))
    errors.email = 'รูปแบบอีเมลไม่ถูกต้อง';
  
  if (users.some(u => u.email === data.email))
    errors.email = 'Email นี้ถูกใช้งานแล้ว';
  
  if (!data.password || data.password.length < 8)
    errors.password = 'รหัสผ่านต้องมีอย่างน้อย 8 ตัวอักษร';
  else if (!/(?=.*\d)(?=.*[a-z])(?=.*[A-Z])/.test(data.password))
    errors.password = 'รหัสผ่านต้องมีตัวเลข ตัวพิมพ์เล็ก และตัวพิมพ์ใหญ่';
  
  if (data.password !== data.confirmPassword)
    errors.confirmPassword = 'รหัสผ่านไม่ตรงกัน';
  
  if (data.agree !== 'yes')
    errors.agree = 'กรุณายอมรับข้อตกลง';
  
  return { isValid: Object.keys(errors).length === 0, errors };
};

// GET form
app.get('/register', (req, res) => {
  res.render('register', { errors: {}, formData: {} });
});

// POST form
app.post('/register', upload.single('avatar'), (req, res) => {
  const { isValid, errors } = validateRegistration(req.body);
  
  if (!isValid) {
    // ลบไฟล์ที่ upload ถ้า validation ไม่ผ่าน
    if (req.file) fs.unlinkSync(req.file.path);
    
    return res.status(400).render('register', {
      errors,
      formData: req.body
    });
  }
  
  const newUser = {
    id: users.length + 1,
    firstName: req.body.firstName.trim(),
    lastName: req.body.lastName.trim(),
    email: req.body.email.toLowerCase(),
    gender: req.body.gender || null,
    bio: req.body.bio || null,
    avatar: req.file ? `/uploads/avatars/${req.file.filename}` : null,
    createdAt: new Date()
  };
  
  users.push(newUser);
  
  res.render('success', { user: newUser });
});

// Ensure upload dir exists
fs.mkdirSync('uploads/avatars', { recursive: true });

app.listen(3000, () => {
  console.log('Registration Form: http://localhost:3000/register');
});
```

---

## 8. Exercise

### Exercise 1: Contact Form

สร้าง Contact Form ที่:
- มีช่อง name, email, subject, message
- Validate ทุก field
- ส่ง email confirmation (จำลองด้วย log)
- แสดง success/error page

### Exercise 2: Survey Form

สร้าง Survey Form ที่:
- มีหลาย steps (step 1, 2, 3)
- เก็บข้อมูลระหว่าง steps ด้วย session
- Summary page ก่อน submit
- บันทึกผลลัพธ์

### Exercise 3: Product Create Form

สร้าง Form สำหรับเพิ่มสินค้า:
- ชื่อ, ราคา, category, description
- Upload รูปภาพหลาย รูป (max 5)
- Resize รูปเป็น thumbnail
- Preview รูปก่อน submit

---

## สรุป Part 17

ในบทนี้เราได้เรียนรู้:

1. **HTML Forms** - GET/POST, encoding types
2. **body-parser** - parse request body
3. **express.urlencoded()** - parse form data
4. **express.json()** - parse JSON body
5. **Form Validation** - manual และ Joi
6. **File Uploads** - multer สำหรับ file handling
7. **Practical** - User Registration Form แบบสมบูรณ์

---

*Part 17 | Node.js/Express.js Course | ขั้นตอนที่ 17 จาก 1000*
