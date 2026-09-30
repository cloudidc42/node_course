# Part 44: XSS Prevention

> ขั้นตอนที่ 44-44 จาก 1000

---

## สารบัญ

1. [XSS Types](#xss-types)
2. [Output Encoding](#output-encoding)
3. [DOMPurify](#dompurify)
4. [Content Security Policy (CSP)](#content-security-policy)
5. [sanitize-html](#sanitize-html)
6. [Best Practices](#best-practices)
7. [แบบฝึกหัด](#แบบฝึกหัด)

---

## XSS Types

XSS (Cross-Site Scripting) คือ attack ที่ attacker inject malicious scripts ลงใน web pages

### 1. Stored XSS (Persistent)

```
Attack flow:
1. Attacker ส่ง malicious script ไปบันทึกใน DB
   POST /comments { content: "<script>steal_cookies()</script>" }

2. Victim โหลดหน้าเว็บ
   GET /posts/123

3. Server ส่ง comment พร้อม script ไปยัง browser
   <div>...</div>
   <script>steal_cookies()</script>  ← script ถูก render!

4. Script รันใน browser ของ victim
   → ขโมย cookies, redirect ไป phishing site, ฯลฯ
```

### 2. Reflected XSS

```
Attack flow:
1. Attacker ส่ง URL พร้อม script payload:
   https://site.com/search?q=<script>alert(document.cookie)</script>

2. Server reflect parameter กลับใน response:
   <h1>Results for: <script>alert(document.cookie)</script></h1>

3. Script รัน ถ้า victim คลิก URL นี้
```

### 3. DOM-based XSS

```javascript
// ❌ Vulnerable JavaScript
const query = new URLSearchParams(location.search).get('q');
document.getElementById('search-term').innerHTML = query; // DOM XSS!

// Attacker URL:
// https://site.com/search?q=<img src=x onerror="steal_cookies()">
```

---

## Output Encoding

หลักการ: encode ข้อมูลก่อน output ทุกครั้ง

### HTML Encoding

```javascript
// utils/encoding.js

// Encode characters สำหรับ HTML context
function encodeHTML(str) {
  return String(str)
    .replace(/&/g, '&amp;')
    .replace(/</g, '&lt;')
    .replace(/>/g, '&gt;')
    .replace(/"/g, '&quot;')
    .replace(/'/g, '&#x27;')
    .replace(/\//g, '&#x2F;');
}

// Encode สำหรับ HTML attribute
function encodeHTMLAttribute(str) {
  return String(str).replace(/[^a-zA-Z0-9 ,._-]/g, (char) => {
    return `&#x${char.charCodeAt(0).toString(16).toUpperCase()};`;
  });
}

// Encode สำหรับ JavaScript string
function encodeJavaScript(str) {
  return String(str).replace(/[^\w ]/g, (char) => {
    return `\\u${char.charCodeAt(0).toString(16).padStart(4, '0')}`;
  });
}

// Encode สำหรับ URL
function encodeURL(str) {
  return encodeURIComponent(str);
}

// ใช้งาน
const userInput = '<script>alert("XSS")</script>';
const safe = encodeHTML(userInput);
// → &lt;script&gt;alert(&quot;XSS&quot;)&lt;/script&gt;
// Browser จะแสดงเป็น text ไม่ใช่ script
```

### Template Engines ที่ Auto-escape

```javascript
// EJS - auto-escape ด้วย <%=
// <%=  → encode HTML (safe)
// <%-  → raw HTML (dangerous!)

// ❌ Dangerous - ไม่ encode
app.get('/profile', (req, res) => {
  res.render('profile', { name: req.user.name });
});
// profile.ejs:
// <h1>Hello <%- name %></h1>  ← ไม่ encode!

// ✅ Safe - auto-encode
// profile.ejs:
// <h1>Hello <%= name %></h1>  ← encode แล้ว!
```

```javascript
// Handlebars - safe by default
// {{ expression }}  → HTML encoded (safe)
// {{{ expression }}} → raw HTML (dangerous!)

// ✅ Safe
// <h1>{{ name }}</h1>

// ❌ Dangerous
// <h1>{{{ name }}}</h1>
```

### JSON Output

```javascript
// ✅ JSON output ปลอดภัยจาก XSS เสมอ (ถ้าตั้ง Content-Type ถูกต้อง)
app.get('/api/user', (req, res) => {
  res.json({ name: req.user.name }); // OK
});

// ❌ แต่ถ้า embed JSON ใน HTML template:
app.get('/page', (req, res) => {
  const user = { name: '</script><script>alert(1)</script>' };
  
  // ❌ Dangerous - XSS ผ่าน JSON in HTML
  res.send(`
    <script>
      var user = ${JSON.stringify(user)};
    </script>
  `);
  
  // ✅ Safe - escape ก่อน embed ใน HTML
  const safeJson = JSON.stringify(user).replace(/</g, '\\u003c').replace(/>/g, '\\u003e');
  res.send(`<script>var user = ${safeJson};</script>`);
});
```

---

## DOMPurify

DOMPurify ใช้สำหรับ sanitize HTML ที่ต้องการ allow HTML บางส่วน (เช่น rich text editor)

### Client-side DOMPurify

```html
<script src="https://cdnjs.cloudflare.com/ajax/libs/dompurify/3.0.5/purify.min.js"></script>
```

```javascript
// Browser
const dirty = '<p>Hello <script>steal()</script></p>';

// Clean!
const clean = DOMPurify.sanitize(dirty);
// → <p>Hello </p>  (script removed)

// Custom config
const clean2 = DOMPurify.sanitize(dirty, {
  ALLOWED_TAGS: ['b', 'i', 'em', 'strong', 'a', 'p', 'ul', 'li', 'ol'],
  ALLOWED_ATTR: ['href', 'class'],
  ALLOW_DATA_ATTR: false,
  FORCE_BODY: true,
});

// ใช้ใน React
function SafeHtml({ content }) {
  const clean = DOMPurify.sanitize(content);
  return <div dangerouslySetInnerHTML={{ __html: clean }} />;
}
```

### Server-side ด้วย isomorphic-dompurify

```bash
npm install isomorphic-dompurify
```

```javascript
const createDOMPurify = require('isomorphic-dompurify');
const { JSDOM } = require('jsdom');

const window = new JSDOM('').window;
const DOMPurify = createDOMPurify(window);

function sanitizeHtml(dirty) {
  return DOMPurify.sanitize(dirty, {
    ALLOWED_TAGS: [
      'b', 'i', 'em', 'strong', 'a', 'p',
      'ul', 'ol', 'li', 'br', 'h1', 'h2', 'h3',
      'blockquote', 'code', 'pre',
    ],
    ALLOWED_ATTR: ['href', 'class', 'id'],
    FORBID_ATTR: ['style', 'onerror', 'onload', 'onclick'],
    FORBID_TAGS: ['script', 'iframe', 'object', 'embed', 'form'],
  });
}

// ใช้งานก่อนบันทึก
app.post('/posts', async (req, res) => {
  const { title, content } = req.body;
  
  const cleanContent = sanitizeHtml(content);
  
  const post = await Post.create({
    title,
    content: cleanContent,
    author: req.user.id,
  });
  
  res.status(201).json(post);
});
```

---

## Content Security Policy

CSP เป็น Defense-in-Depth ที่สำคัญที่สุดสำหรับ XSS

```javascript
// หยุด XSS ด้วย CSP
app.use(helmet({
  contentSecurityPolicy: {
    directives: {
      defaultSrc: ["'self'"],
      
      // ป้องกัน inline scripts
      scriptSrc: [
        "'self'",
        // ❌ "'unsafe-inline'" - อนุญาต inline scripts (เปิดช่องโหว่)
        // ✅ ใช้ nonce แทน
        (req, res) => `'nonce-${res.locals.nonce}'`,
      ],
      
      // ป้องกัน data: URIs ที่อาจมี scripts
      imgSrc: ["'self'", "https:"],
      
      // ป้องกัน การส่งข้อมูลไปยัง external sites
      connectSrc: ["'self'"],
      
      // ไม่อนุญาต base tag ที่อาจเปลี่ยน URL resolution
      baseUri: ["'none'"],
      
      // ไม่อนุญาต form submission ไป external
      formAction: ["'self'"],
    },
  },
}));
```

---

## sanitize-html

sanitize-html เป็น server-side library สำหรับ sanitize HTML

```bash
npm install sanitize-html
```

```javascript
const sanitizeHtml = require('sanitize-html');

// Default config - เข้มงวดมาก
const clean = sanitizeHtml(dirty);

// Custom config สำหรับ blog posts
const blogConfig = {
  allowedTags: [
    'h1', 'h2', 'h3', 'h4', 'h5', 'h6',
    'b', 'i', 'strong', 'em', 'strike',
    'p', 'br', 'ul', 'ol', 'li',
    'blockquote', 'pre', 'code',
    'a', 'img',
  ],
  
  allowedAttributes: {
    'a': ['href', 'title', 'target'],
    'img': ['src', 'alt', 'width', 'height'],
    'code': ['class'],  // สำหรับ syntax highlighting
    '*': ['class'],
  },
  
  // ป้องกัน javascript: protocol
  allowedSchemes: ['http', 'https', 'mailto'],
  allowedSchemesByTag: {
    img: ['http', 'https', 'data'], // อนุญาต data URLs สำหรับ img
  },
  
  // Custom transformations
  transformTags: {
    'a': (tagName, attribs) => {
      // เพิ่ม rel="noopener noreferrer" สำหรับ external links
      if (attribs.href && !attribs.href.startsWith('/')) {
        return {
          tagName: 'a',
          attribs: {
            ...attribs,
            target: '_blank',
            rel: 'noopener noreferrer',
          },
        };
      }
      return { tagName, attribs };
    },
  },
};

function sanitizeBlogContent(content) {
  return sanitizeHtml(content, blogConfig);
}

// Strict config สำหรับ comments
const commentConfig = {
  allowedTags: ['b', 'i', 'em', 'strong', 'a', 'br'],
  allowedAttributes: {
    'a': ['href'],
  },
  allowedSchemes: ['https'],
};

function sanitizeComment(content) {
  return sanitizeHtml(content, commentConfig);
}
```

---

## Best Practices

### ป้องกัน XSS อย่างครบถ้วน

```javascript
// 1. Validate input type/format
// 2. Sanitize HTML ถ้า allow HTML
// 3. Encode output ตาม context
// 4. ใช้ CSP
// 5. HttpOnly cookies (ป้องกัน script อ่าน cookies)

// HttpOnly cookie
res.cookie('sessionId', token, {
  httpOnly: true,    // ไม่ให้ JavaScript อ่าน cookie
  secure: true,      // HTTPS only
  sameSite: 'strict',
});

// 6. X-Content-Type-Options
app.use(helmet.noSniff());

// 7. ไม่ render user input ใน innerHTML
// ❌
element.innerHTML = userInput;

// ✅
element.textContent = userInput;
```

### Security Headers ที่เกี่ยวข้อง

```javascript
app.use(helmet({
  contentSecurityPolicy: {
    directives: {
      defaultSrc: ["'self'"],
      scriptSrc: ["'self'"],
      // ไม่มี 'unsafe-inline' หรือ 'unsafe-eval'
    },
  },
  
  // X-XSS-Protection (legacy, แต่ยังมีประโยชน์)
  xssFilter: true,
  
  // ป้องกัน MIME type confusion
  noSniff: true,
}));
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Fix XSS Vulnerabilities

```javascript
// ❌ แก้ไขทั้งหมด

// 1. Express route ที่ vulnerable
app.get('/search', (req, res) => {
  const q = req.query.q;
  res.send(`<h1>Results for: ${q}</h1>`); // XSS!
});

// 2. Template ที่ vulnerable
// profile.ejs:
// <div class="bio"><%- user.bio %></div>  // XSS!

// 3. React component ที่ vulnerable
function Comment({ content }) {
  return <div dangerouslySetInnerHTML={{ __html: content }} />; // XSS!
}
```

### แบบฝึกหัดที่ 2: Rich Text Editor Security

สร้าง blog post editor ที่:
1. Allow HTML formatting (bold, italic, links, images)
2. ป้องกัน scripts และ event handlers
3. Validate links (https only)
4. Test ด้วย XSS payloads

### แบบฝึกหัดที่ 3: CSP Implementation

ตั้งค่า CSP สำหรับ SPA (React):
1. ไม่ใช้ `unsafe-inline`
2. ใช้ nonce สำหรับ inline scripts
3. Report violations
4. Test ว่า XSS ถูกบล็อก

---

## สรุป

XSS ป้องกันได้ด้วยหลายชั้น

| ชั้น | วิธี |
|------|------|
| Input | Validate format |
| Storage | Sanitize HTML (DOMPurify, sanitize-html) |
| Output | Encode ตาม context |
| Browser | CSP headers |
| Cookies | HttpOnly flag |

**ถัดไป**: [Part 45: CSRF Protection →](./part-45-csrf-protection.md)
