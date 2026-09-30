# Part 05: HTTP Module
## ขั้นตอนที่ 141-180: สร้าง Web Server จาก Scratch

---

## 🎯 เป้าหมายของ Part นี้

1. สร้าง HTTP Server ด้วย Node.js built-in http module ได้
2. จัดการ Routes และ Methods ได้
3. Parse request body ได้
4. ส่ง response formats ต่างๆ ได้
5. Handle static files ได้
6. เข้าใจ HTTP Protocol ได้อย่างลึกซึ้ง

---

## ขั้นตอนที่ 141: HTTP Protocol พื้นฐาน

```
HTTP Request Structure:
┌─────────────────────────────────────────────┐
│  GET /api/users?page=1 HTTP/1.1             │ ← Request Line
│  Host: example.com                          │
│  Accept: application/json                  │ ← Headers
│  Authorization: Bearer token123             │
│  Content-Type: application/json            │
│                                             │
│  {"name": "Alice"}                          │ ← Body (optional)
└─────────────────────────────────────────────┘

HTTP Response Structure:
┌─────────────────────────────────────────────┐
│  HTTP/1.1 200 OK                            │ ← Status Line
│  Content-Type: application/json             │
│  Content-Length: 45                         │ ← Headers
│  X-Request-Id: abc123                       │
│                                             │
│  {"id": 1, "name": "Alice", "age": 28}     │ ← Body
└─────────────────────────────────────────────┘

HTTP Methods:
┌──────────┬──────────────────────────────────┐
│  GET     │ ดึงข้อมูล                        │
│  POST    │ สร้างข้อมูลใหม่                  │
│  PUT     │ แก้ไขข้อมูลทั้งหมด               │
│  PATCH   │ แก้ไขข้อมูลบางส่วน              │
│  DELETE  │ ลบข้อมูล                         │
│  HEAD    │ เหมือน GET แต่ไม่มี body         │
│  OPTIONS │ ดู methods ที่รองรับ              │
└──────────┴──────────────────────────────────┘

Common Status Codes:
┌──────┬─────────────────────────────────────┐
│  200 │ OK                                  │
│  201 │ Created                             │
│  204 │ No Content                          │
│  301 │ Moved Permanently                   │
│  302 │ Found (Temporary Redirect)          │
│  304 │ Not Modified                        │
│  400 │ Bad Request                         │
│  401 │ Unauthorized                        │
│  403 │ Forbidden                           │
│  404 │ Not Found                           │
│  405 │ Method Not Allowed                  │
│  409 │ Conflict                            │
│  422 │ Unprocessable Entity                │
│  429 │ Too Many Requests                   │
│  500 │ Internal Server Error               │
│  502 │ Bad Gateway                         │
│  503 │ Service Unavailable                 │
└──────┴─────────────────────────────────────┘
```

---

## ขั้นตอนที่ 142: Hello World HTTP Server

```javascript
// simple-server.js
const http = require('http');

const server = http.createServer((req, res) => {
  // req = IncomingMessage (request)
  // res = ServerResponse (response)
  
  // แสดงข้อมูล request
  console.log(`${req.method} ${req.url}`);
  console.log('Headers:', req.headers);
  
  // ส่ง response
  res.statusCode = 200;
  res.setHeader('Content-Type', 'text/plain; charset=utf-8');
  res.end('สวัสดี Node.js HTTP Server!');
});

const PORT = process.env.PORT || 3000;
const HOST = process.env.HOST || 'localhost';

server.listen(PORT, HOST, () => {
  console.log(`Server running at http://${HOST}:${PORT}/`);
});

// Graceful shutdown
process.on('SIGTERM', () => {
  server.close(() => {
    console.log('Server closed');
    process.exit(0);
  });
});

process.on('SIGINT', () => {
  server.close(() => {
    console.log('\nServer closed');
    process.exit(0);
  });
});
```

```bash
# ทดสอบด้วย curl
curl http://localhost:3000/

# ทดสอบด้วย httpie
http GET http://localhost:3000/

# ทดสอบด้วย node
node -e "
  const http = require('http');
  http.get('http://localhost:3000/', res => {
    let data = '';
    res.on('data', chunk => data += chunk);
    res.on('end', () => console.log(data));
  });
"
```

---

## ขั้นตอนที่ 143: HTTP Server พร้อม Routing

```javascript
// server-with-routing.js
const http = require('http');
const url = require('url');

// ฐานข้อมูลจำลอง
let users = [
  { id: 1, name: 'Alice', email: 'alice@example.com', age: 28 },
  { id: 2, name: 'Bob', email: 'bob@example.com', age: 32 },
  { id: 3, name: 'Charlie', email: 'charlie@example.com', age: 25 }
];

let nextId = 4;

// ═══════════════════════════════════════════
// Request Body Parser
// ═══════════════════════════════════════════

async function parseBody(req) {
  return new Promise((resolve, reject) => {
    const chunks = [];
    
    req.on('data', chunk => chunks.push(chunk));
    req.on('end', () => {
      const body = Buffer.concat(chunks).toString();
      
      if (!body) {
        resolve(null);
        return;
      }
      
      const contentType = req.headers['content-type'] || '';
      
      if (contentType.includes('application/json')) {
        try {
          resolve(JSON.parse(body));
        } catch {
          reject(new Error('Invalid JSON'));
        }
      } else if (contentType.includes('application/x-www-form-urlencoded')) {
        const parsed = Object.fromEntries(new URLSearchParams(body));
        resolve(parsed);
      } else {
        resolve(body);
      }
    });
    
    req.on('error', reject);
  });
}

// ═══════════════════════════════════════════
// Response Helpers
// ═══════════════════════════════════════════

function sendJSON(res, data, statusCode = 200) {
  const body = JSON.stringify(data);
  
  res.writeHead(statusCode, {
    'Content-Type': 'application/json; charset=utf-8',
    'Content-Length': Buffer.byteLength(body)
  });
  
  res.end(body);
}

function sendError(res, message, statusCode = 400) {
  sendJSON(res, { error: message, statusCode }, statusCode);
}

function sendNotFound(res) {
  sendError(res, 'Resource not found', 404);
}

// ═══════════════════════════════════════════
// Route Handlers
// ═══════════════════════════════════════════

// GET /api/users - ดู users ทั้งหมด
function getUsers(req, res, parsedUrl) {
  const { query } = parsedUrl;
  let result = [...users];
  
  // Filter
  if (query.name) {
    result = result.filter(u => 
      u.name.toLowerCase().includes(query.name.toLowerCase())
    );
  }
  
  // Pagination
  const page = parseInt(query.page) || 1;
  const limit = parseInt(query.limit) || 10;
  const offset = (page - 1) * limit;
  const paginatedData = result.slice(offset, offset + limit);
  
  sendJSON(res, {
    data: paginatedData,
    total: result.length,
    page,
    limit,
    pages: Math.ceil(result.length / limit)
  });
}

// GET /api/users/:id - ดู user คนเดียว
function getUserById(req, res, id) {
  const user = users.find(u => u.id === parseInt(id));
  
  if (!user) {
    sendNotFound(res);
    return;
  }
  
  sendJSON(res, user);
}

// POST /api/users - สร้าง user ใหม่
async function createUser(req, res) {
  const body = await parseBody(req);
  
  if (!body?.name || !body?.email) {
    sendError(res, 'Name and email are required', 400);
    return;
  }
  
  // ตรวจสอบ email ซ้ำ
  if (users.some(u => u.email === body.email)) {
    sendError(res, 'Email already exists', 409);
    return;
  }
  
  const newUser = {
    id: nextId++,
    name: body.name,
    email: body.email,
    age: body.age || null,
    createdAt: new Date().toISOString()
  };
  
  users.push(newUser);
  sendJSON(res, newUser, 201);
}

// PUT /api/users/:id - แก้ไข user
async function updateUser(req, res, id) {
  const userIndex = users.findIndex(u => u.id === parseInt(id));
  
  if (userIndex === -1) {
    sendNotFound(res);
    return;
  }
  
  const body = await parseBody(req);
  
  users[userIndex] = {
    ...users[userIndex],
    ...body,
    id: users[userIndex].id  // ไม่ให้เปลี่ยน id
  };
  
  sendJSON(res, users[userIndex]);
}

// DELETE /api/users/:id - ลบ user
function deleteUser(req, res, id) {
  const userIndex = users.findIndex(u => u.id === parseInt(id));
  
  if (userIndex === -1) {
    sendNotFound(res);
    return;
  }
  
  const deleted = users.splice(userIndex, 1)[0];
  sendJSON(res, { message: 'User deleted', user: deleted });
}

// ═══════════════════════════════════════════
// Router
// ═══════════════════════════════════════════

const server = http.createServer(async (req, res) => {
  // CORS headers
  res.setHeader('Access-Control-Allow-Origin', '*');
  res.setHeader('Access-Control-Allow-Methods', 'GET, POST, PUT, DELETE, OPTIONS');
  res.setHeader('Access-Control-Allow-Headers', 'Content-Type, Authorization');
  
  // Handle preflight
  if (req.method === 'OPTIONS') {
    res.writeHead(204);
    res.end();
    return;
  }
  
  const parsedUrl = url.parse(req.url, true);
  const { pathname } = parsedUrl;
  
  // Log request
  const startTime = Date.now();
  const originalEnd = res.end.bind(res);
  res.end = function(...args) {
    console.log(`${req.method} ${pathname} ${res.statusCode} ${Date.now() - startTime}ms`);
    return originalEnd(...args);
  };
  
  try {
    // Router matching
    // GET /api/users
    if (req.method === 'GET' && pathname === '/api/users') {
      return getUsers(req, res, parsedUrl);
    }
    
    // GET /api/users/:id
    const getUserMatch = pathname.match(/^\/api\/users\/(\d+)$/);
    if (req.method === 'GET' && getUserMatch) {
      return getUserById(req, res, getUserMatch[1]);
    }
    
    // POST /api/users
    if (req.method === 'POST' && pathname === '/api/users') {
      return await createUser(req, res);
    }
    
    // PUT /api/users/:id
    const putUserMatch = pathname.match(/^\/api\/users\/(\d+)$/);
    if (req.method === 'PUT' && putUserMatch) {
      return await updateUser(req, res, putUserMatch[1]);
    }
    
    // DELETE /api/users/:id
    const deleteUserMatch = pathname.match(/^\/api\/users\/(\d+)$/);
    if (req.method === 'DELETE' && deleteUserMatch) {
      return deleteUser(req, res, deleteUserMatch[1]);
    }
    
    // GET / - Home
    if (req.method === 'GET' && pathname === '/') {
      sendJSON(res, { 
        message: 'Users API',
        version: '1.0.0',
        endpoints: {
          'GET /api/users': 'List all users',
          'GET /api/users/:id': 'Get user by ID',
          'POST /api/users': 'Create user',
          'PUT /api/users/:id': 'Update user',
          'DELETE /api/users/:id': 'Delete user'
        }
      });
      return;
    }
    
    // 404
    sendNotFound(res);
    
  } catch (err) {
    console.error('Server error:', err);
    sendError(res, 'Internal server error', 500);
  }
});

server.listen(3000, () => {
  console.log('API Server running at http://localhost:3000');
});
```

---

## ขั้นตอนที่ 144: Static File Server

```javascript
// static-server.js
const http = require('http');
const fs = require('fs');
const fsPromises = require('fs/promises');
const path = require('path');
const crypto = require('crypto');

// MIME Types
const MIME_TYPES = {
  '.html': 'text/html; charset=utf-8',
  '.htm':  'text/html; charset=utf-8',
  '.css':  'text/css; charset=utf-8',
  '.js':   'application/javascript; charset=utf-8',
  '.mjs':  'application/javascript; charset=utf-8',
  '.json': 'application/json; charset=utf-8',
  '.png':  'image/png',
  '.jpg':  'image/jpeg',
  '.jpeg': 'image/jpeg',
  '.gif':  'image/gif',
  '.svg':  'image/svg+xml',
  '.ico':  'image/x-icon',
  '.webp': 'image/webp',
  '.pdf':  'application/pdf',
  '.txt':  'text/plain; charset=utf-8',
  '.xml':  'application/xml',
  '.woff': 'font/woff',
  '.woff2':'font/woff2',
  '.ttf':  'font/ttf',
  '.mp4':  'video/mp4',
  '.mp3':  'audio/mpeg',
  '.wasm': 'application/wasm'
};

const PUBLIC_DIR = path.join(__dirname, 'public');

// สร้าง ETag จาก file stats
function generateETag(stat) {
  return `"${stat.size}-${stat.mtime.getTime()}"`;
}

// ตรวจสอบว่า path ปลอดภัย (ป้องกัน path traversal)
function isSafePath(filePath, baseDir) {
  const resolvedPath = path.resolve(filePath);
  return resolvedPath.startsWith(baseDir);
}

const server = http.createServer(async (req, res) => {
  if (req.method !== 'GET' && req.method !== 'HEAD') {
    res.writeHead(405, { 'Allow': 'GET, HEAD' });
    res.end('Method Not Allowed');
    return;
  }
  
  // Parse URL
  let urlPath = new URL(req.url, 'http://localhost').pathname;
  
  // Decode URL encoding
  try {
    urlPath = decodeURIComponent(urlPath);
  } catch {
    res.writeHead(400);
    res.end('Bad Request');
    return;
  }
  
  // สร้าง file path
  let filePath = path.join(PUBLIC_DIR, urlPath);
  
  // ตรวจสอบ path traversal
  if (!isSafePath(filePath, PUBLIC_DIR)) {
    res.writeHead(403);
    res.end('Forbidden');
    return;
  }
  
  try {
    let stat = await fsPromises.stat(filePath);
    
    // ถ้าเป็น directory ให้ลอง index.html
    if (stat.isDirectory()) {
      filePath = path.join(filePath, 'index.html');
      try {
        stat = await fsPromises.stat(filePath);
      } catch {
        // ถ้าไม่มี index.html แสดง directory listing
        const entries = await fsPromises.readdir(path.join(PUBLIC_DIR, urlPath));
        const html = generateDirectoryListing(urlPath, entries);
        res.writeHead(200, { 'Content-Type': 'text/html; charset=utf-8' });
        res.end(html);
        return;
      }
    }
    
    // ตรวจสอบ cache (ETag)
    const etag = generateETag(stat);
    const ifNoneMatch = req.headers['if-none-match'];
    
    if (ifNoneMatch === etag) {
      res.writeHead(304);
      res.end();
      return;
    }
    
    // ตรวจสอบ If-Modified-Since
    const ifModifiedSince = req.headers['if-modified-since'];
    if (ifModifiedSince && new Date(ifModifiedSince) >= stat.mtime) {
      res.writeHead(304);
      res.end();
      return;
    }
    
    // ดู extension
    const ext = path.extname(filePath).toLowerCase();
    const contentType = MIME_TYPES[ext] || 'application/octet-stream';
    
    // ตรวจสอบ Range request (video streaming)
    const range = req.headers['range'];
    
    if (range) {
      const parts = range.replace(/bytes=/, '').split('-');
      const start = parseInt(parts[0], 10);
      const end = parts[1] ? parseInt(parts[1], 10) : stat.size - 1;
      const chunkSize = (end - start) + 1;
      
      const headers = {
        'Content-Range': `bytes ${start}-${end}/${stat.size}`,
        'Accept-Ranges': 'bytes',
        'Content-Length': chunkSize,
        'Content-Type': contentType
      };
      
      res.writeHead(206, headers);
      
      if (req.method !== 'HEAD') {
        const stream = fs.createReadStream(filePath, { start, end });
        stream.pipe(res);
      } else {
        res.end();
      }
      return;
    }
    
    // Normal response headers
    const headers = {
      'Content-Type': contentType,
      'Content-Length': stat.size,
      'ETag': etag,
      'Last-Modified': stat.mtime.toUTCString(),
      'Cache-Control': getCacheControl(ext)
    };
    
    // Compression (ถ้า client รองรับ)
    const acceptEncoding = req.headers['accept-encoding'] || '';
    if (acceptEncoding.includes('gzip') && stat.size > 1024) {
      const zlib = require('zlib');
      headers['Content-Encoding'] = 'gzip';
      delete headers['Content-Length'];  // ไม่รู้ compressed size ล่วงหน้า
      
      res.writeHead(200, headers);
      
      if (req.method !== 'HEAD') {
        const gzip = zlib.createGzip();
        fs.createReadStream(filePath).pipe(gzip).pipe(res);
      } else {
        res.end();
      }
      return;
    }
    
    res.writeHead(200, headers);
    
    if (req.method !== 'HEAD') {
      fs.createReadStream(filePath).pipe(res);
    } else {
      res.end();
    }
    
  } catch (err) {
    if (err.code === 'ENOENT') {
      res.writeHead(404, { 'Content-Type': 'text/html' });
      res.end('<h1>404 Not Found</h1>');
    } else {
      console.error(err);
      res.writeHead(500);
      res.end('Internal Server Error');
    }
  }
});

function getCacheControl(ext) {
  const immutable = ['.woff', '.woff2', '.ttf', '.ico'];
  const long = ['.jpg', '.jpeg', '.png', '.gif', '.webp', '.svg'];
  const medium = ['.css', '.js', '.mjs'];
  
  if (immutable.includes(ext)) return 'public, max-age=31536000, immutable';
  if (long.includes(ext)) return 'public, max-age=86400';
  if (medium.includes(ext)) return 'public, max-age=3600';
  return 'no-cache';
}

function generateDirectoryListing(urlPath, entries) {
  const items = entries.map(name => 
    `<li><a href="${urlPath}${urlPath.endsWith('/') ? '' : '/'}${name}">${name}</a></li>`
  ).join('\n');
  
  return `<!DOCTYPE html>
<html>
<head><title>Directory: ${urlPath}</title></head>
<body>
  <h1>Directory: ${urlPath}</h1>
  <ul>
    <li><a href="${path.dirname(urlPath)}">.. (parent)</a></li>
    ${items}
  </ul>
</body>
</html>`;
}

server.listen(3000, () => {
  console.log('Static server running at http://localhost:3000');
  console.log(`Serving files from: ${PUBLIC_DIR}`);
});
```

---

## ขั้นตอนที่ 145: HTTP Client

```javascript
// http-client.js
const http = require('http');
const https = require('https');
const { URL } = require('url');

// ═══════════════════════════════════════════
// Simple HTTP GET
// ═══════════════════════════════════════════

function get(url) {
  return new Promise((resolve, reject) => {
    const parsedUrl = new URL(url);
    const protocol = parsedUrl.protocol === 'https:' ? https : http;
    
    const req = protocol.get(url, (res) => {
      const chunks = [];
      
      res.on('data', chunk => chunks.push(chunk));
      res.on('end', () => {
        const body = Buffer.concat(chunks).toString();
        
        if (res.statusCode >= 200 && res.statusCode < 300) {
          try {
            resolve({ 
              status: res.statusCode,
              headers: res.headers,
              data: JSON.parse(body)
            });
          } catch {
            resolve({ status: res.statusCode, headers: res.headers, data: body });
          }
        } else {
          reject(new Error(`HTTP Error: ${res.statusCode} ${body}`));
        }
      });
    });
    
    req.on('error', reject);
    req.setTimeout(10000, () => {
      req.destroy(new Error('Request timeout'));
    });
  });
}

// ═══════════════════════════════════════════
// Full HTTP Client
// ═══════════════════════════════════════════

class HTTPClient {
  constructor(baseURL = '', options = {}) {
    this.baseURL = baseURL;
    this.defaultHeaders = {
      'User-Agent': 'NodeJS-HTTP-Client/1.0',
      'Accept': 'application/json',
      ...options.headers
    };
    this.timeout = options.timeout || 30000;
  }
  
  async request(method, path, { data, headers, params } = {}) {
    const url = new URL(this.baseURL + path);
    
    // เพิ่ม query params
    if (params) {
      Object.entries(params).forEach(([key, value]) => {
        url.searchParams.set(key, value);
      });
    }
    
    const protocol = url.protocol === 'https:' ? https : http;
    const body = data ? JSON.stringify(data) : null;
    
    const requestHeaders = {
      ...this.defaultHeaders,
      ...headers
    };
    
    if (body) {
      requestHeaders['Content-Type'] = 'application/json';
      requestHeaders['Content-Length'] = Buffer.byteLength(body);
    }
    
    const options = {
      hostname: url.hostname,
      port: url.port || (url.protocol === 'https:' ? 443 : 80),
      path: url.pathname + url.search,
      method: method.toUpperCase(),
      headers: requestHeaders
    };
    
    return new Promise((resolve, reject) => {
      const req = protocol.request(options, (res) => {
        const chunks = [];
        
        res.on('data', chunk => chunks.push(chunk));
        res.on('end', () => {
          const rawBody = Buffer.concat(chunks).toString();
          const contentType = res.headers['content-type'] || '';
          
          let parsedBody;
          if (contentType.includes('application/json')) {
            try {
              parsedBody = JSON.parse(rawBody);
            } catch {
              parsedBody = rawBody;
            }
          } else {
            parsedBody = rawBody;
          }
          
          const response = {
            status: res.statusCode,
            statusText: res.statusMessage,
            headers: res.headers,
            data: parsedBody,
            ok: res.statusCode >= 200 && res.statusCode < 300
          };
          
          if (!response.ok) {
            const error = new Error(`HTTP Error: ${res.statusCode} ${res.statusMessage}`);
            error.response = response;
            reject(error);
          } else {
            resolve(response);
          }
        });
      });
      
      req.on('error', reject);
      
      // Timeout
      req.setTimeout(this.timeout, () => {
        req.destroy(new Error(`Request timeout after ${this.timeout}ms`));
      });
      
      if (body) {
        req.write(body);
      }
      
      req.end();
    });
  }
  
  get(path, options) { return this.request('GET', path, options); }
  post(path, data, options) { return this.request('POST', path, { data, ...options }); }
  put(path, data, options) { return this.request('PUT', path, { data, ...options }); }
  patch(path, data, options) { return this.request('PATCH', path, { data, ...options }); }
  delete(path, options) { return this.request('DELETE', path, options); }
}

// ═══════════════════════════════════════════
// ตัวอย่างใช้งาน
// ═══════════════════════════════════════════

async function demo() {
  const client = new HTTPClient('https://jsonplaceholder.typicode.com');
  
  // GET request
  const users = await client.get('/users');
  console.log('Users:', users.data.length);
  
  // POST request
  const newPost = await client.post('/posts', {
    title: 'My Post',
    body: 'Content here',
    userId: 1
  });
  console.log('Created post ID:', newPost.data.id);
  
  // GET with params
  const filtered = await client.get('/posts', {
    params: { userId: 1 }
  });
  console.log('Posts by user 1:', filtered.data.length);
}

demo();
```

---

## ขั้นตอนที่ 146: สร้าง Mini Framework

```javascript
// mini-framework.js - สร้าง Express-like framework เอง

const http = require('http');
const url = require('url');

class Router {
  constructor() {
    this.routes = [];
    this.middlewares = [];
  }
  
  use(path, ...handlers) {
    if (typeof path === 'function') {
      handlers = [path, ...handlers];
      path = '/';
    }
    
    this.middlewares.push({ path, handlers });
    return this;
  }
  
  route(method, pattern, ...handlers) {
    this.routes.push({
      method: method.toUpperCase(),
      pattern: this.pathToRegex(pattern),
      params: this.extractParams(pattern),
      handlers
    });
    return this;
  }
  
  get(pattern, ...handlers) { return this.route('GET', pattern, ...handlers); }
  post(pattern, ...handlers) { return this.route('POST', pattern, ...handlers); }
  put(pattern, ...handlers) { return this.route('PUT', pattern, ...handlers); }
  patch(pattern, ...handlers) { return this.route('PATCH', pattern, ...handlers); }
  delete(pattern, ...handlers) { return this.route('DELETE', pattern, ...handlers); }
  
  pathToRegex(pattern) {
    const regexString = pattern
      .replace(/:[^/]+/g, '([^/]+)')
      .replace(/\*/g, '.*');
    return new RegExp(`^${regexString}$`);
  }
  
  extractParams(pattern) {
    const params = [];
    const matches = pattern.matchAll(/:([^/]+)/g);
    for (const match of matches) {
      params.push(match[1]);
    }
    return params;
  }
  
  async handle(req, res) {
    const parsedUrl = url.parse(req.url, true);
    const pathname = parsedUrl.pathname;
    
    // เพิ่ม utility methods ให้ res
    res.json = (data, status = 200) => {
      const body = JSON.stringify(data);
      res.writeHead(status, {
        'Content-Type': 'application/json',
        'Content-Length': Buffer.byteLength(body)
      });
      res.end(body);
    };
    
    res.status = (code) => {
      res.statusCode = code;
      return res;
    };
    
    res.send = (data, status = 200) => {
      const body = typeof data === 'string' ? data : JSON.stringify(data);
      const contentType = typeof data === 'string' ? 'text/plain' : 'application/json';
      res.writeHead(status, { 'Content-Type': contentType });
      res.end(body);
    };
    
    // เพิ่ม utility methods ให้ req
    req.params = {};
    req.query = parsedUrl.query;
    req.pathname = pathname;
    
    // Parse body
    if (['POST', 'PUT', 'PATCH'].includes(req.method)) {
      req.body = await this.parseBody(req);
    }
    
    // รัน middlewares
    const middlewareStack = [
      ...this.middlewares.filter(m => pathname.startsWith(m.path)),
      ...this.findRoute(req.method, pathname)
    ].flatMap(r => r.handlers || []);
    
    let index = 0;
    
    const next = async (err) => {
      if (err) {
        // Error handling
        const errorHandlers = this.middlewares
          .filter(m => m.handlers.length === 4)
          .flatMap(m => m.handlers);
        
        for (const handler of errorHandlers) {
          await handler(err, req, res, () => {});
        }
        return;
      }
      
      const handler = middlewareStack[index++];
      if (handler) {
        try {
          await handler(req, res, next);
        } catch (error) {
          await next(error);
        }
      }
    };
    
    await next();
    
    // ถ้าไม่มี handler ตอบ
    if (!res.writableEnded) {
      res.json({ error: 'Not Found' }, 404);
    }
  }
  
  findRoute(method, pathname) {
    for (const route of this.routes) {
      if (route.method !== method) continue;
      
      const match = pathname.match(route.pattern);
      if (!match) continue;
      
      // Extract path params
      const params = {};
      route.params.forEach((param, index) => {
        params[param] = match[index + 1];
      });
      
      return [{ ...route, extractedParams: params }];
    }
    return [];
  }
  
  async parseBody(req) {
    return new Promise((resolve, reject) => {
      const chunks = [];
      req.on('data', chunk => chunks.push(chunk));
      req.on('end', () => {
        const body = Buffer.concat(chunks).toString();
        if (!body) return resolve(null);
        
        const contentType = req.headers['content-type'] || '';
        try {
          if (contentType.includes('application/json')) {
            resolve(JSON.parse(body));
          } else {
            resolve(body);
          }
        } catch {
          resolve(body);
        }
      });
      req.on('error', reject);
    });
  }
}

class App extends Router {
  constructor() {
    super();
    this.server = null;
  }
  
  listen(port, host = 'localhost', callback) {
    this.server = http.createServer((req, res) => {
      this.handle(req, res);
    });
    
    this.server.listen(port, host, () => {
      if (callback) callback();
      else console.log(`Server listening at http://${host}:${port}`);
    });
    
    return this.server;
  }
  
  close(callback) {
    if (this.server) this.server.close(callback);
  }
}

// ═══════════════════════════════════════════
// ทดสอบ Mini Framework
// ═══════════════════════════════════════════

const app = new App();

// Global middlewares
app.use((req, res, next) => {
  const start = Date.now();
  console.log(`→ ${req.method} ${req.pathname}`);
  res.on('finish', () => {
    console.log(`← ${req.method} ${req.pathname} ${res.statusCode} (${Date.now()-start}ms)`);
  });
  next();
});

app.use((req, res, next) => {
  res.setHeader('X-Powered-By', 'Mini-Framework');
  res.setHeader('Access-Control-Allow-Origin', '*');
  next();
});

// Routes
app.get('/', (req, res) => {
  res.json({ message: 'Welcome to Mini Framework!', version: '1.0' });
});

app.get('/users', (req, res) => {
  const users = [
    { id: 1, name: 'Alice' },
    { id: 2, name: 'Bob' }
  ];
  res.json(users);
});

app.get('/users/:id', (req, res) => {
  const { id } = req.params;
  res.json({ id: parseInt(id), name: 'User ' + id });
});

app.post('/users', (req, res) => {
  const user = { id: Date.now(), ...req.body };
  res.json(user, 201);
});

// Error handler
app.use((err, req, res, next) => {
  console.error(err);
  res.json({ error: err.message }, 500);
});

app.listen(3000, 'localhost', () => {
  console.log('Mini Framework server at http://localhost:3000');
});
```

---

## ขั้นตอนที่ 147: Streaming HTTP Response

```javascript
// streaming-response.js

const http = require('http');
const { Readable } = require('stream');

const server = http.createServer(async (req, res) => {
  const url = req.url;
  
  // ═══════════════════════════════════════
  // Server-Sent Events (SSE)
  // ═══════════════════════════════════════
  if (url === '/sse') {
    res.writeHead(200, {
      'Content-Type': 'text/event-stream',
      'Cache-Control': 'no-cache',
      'Connection': 'keep-alive',
      'Access-Control-Allow-Origin': '*'
    });
    
    let count = 0;
    const interval = setInterval(() => {
      count++;
      
      // SSE format: data: <payload>\n\n
      res.write(`id: ${count}\n`);
      res.write(`event: update\n`);
      res.write(`data: ${JSON.stringify({ count, time: new Date().toISOString() })}\n\n`);
      
      if (count >= 10) {
        res.write('event: close\ndata: Stream ended\n\n');
        res.end();
        clearInterval(interval);
      }
    }, 1000);
    
    // ถ้า client disconnect
    req.on('close', () => {
      clearInterval(interval);
    });
    
    return;
  }
  
  // ═══════════════════════════════════════
  // Chunked Transfer (streaming large data)
  // ═══════════════════════════════════════
  if (url === '/stream') {
    res.writeHead(200, {
      'Content-Type': 'application/json',
      'Transfer-Encoding': 'chunked',
      'X-Content-Type-Options': 'nosniff'
    });
    
    // ส่งข้อมูลทีละ chunk
    res.write('[');
    
    for (let i = 0; i < 100; i++) {
      if (i > 0) res.write(',');
      
      const item = JSON.stringify({
        id: i + 1,
        name: `Item ${i + 1}`,
        value: Math.random()
      });
      
      res.write(item);
      
      // หยุดพักระหว่าง chunks
      await new Promise(resolve => setTimeout(resolve, 10));
    }
    
    res.write(']');
    res.end();
    return;
  }
  
  // ═══════════════════════════════════════
  // Readable Stream as response
  // ═══════════════════════════════════════
  if (url === '/readable') {
    let count = 0;
    
    const readable = new Readable({
      read() {
        if (count < 5) {
          count++;
          this.push(JSON.stringify({ id: count, data: 'chunk ' + count }) + '\n');
        } else {
          this.push(null);  // end of stream
        }
      }
    });
    
    res.writeHead(200, { 'Content-Type': 'text/plain' });
    readable.pipe(res);
    return;
  }
  
  // Default
  res.writeHead(200, { 'Content-Type': 'text/html' });
  res.end(`
    <html>
      <body>
        <h2>Streaming Demos</h2>
        <ul>
          <li><a href="/sse">Server-Sent Events</a></li>
          <li><a href="/stream">Chunked Transfer</a></li>
          <li><a href="/readable">Readable Stream</a></li>
        </ul>
        
        <h2>SSE Demo</h2>
        <div id="messages"></div>
        <script>
          const es = new EventSource('/sse');
          es.addEventListener('update', e => {
            const div = document.getElementById('messages');
            const data = JSON.parse(e.data);
            div.innerHTML += '<p>' + JSON.stringify(data) + '</p>';
          });
          es.addEventListener('close', () => {
            es.close();
          });
        </script>
      </body>
    </html>
  `);
});

server.listen(3000, () => {
  console.log('Streaming server at http://localhost:3000');
});
```

---

## ขั้นตอนที่ 148: สรุป Part 05 และ Exercise

### สิ่งที่เรียนรู้ใน Part 05

```
✅ HTTP Protocol - Methods, Status Codes, Headers
✅ สร้าง HTTP Server ด้วย http module
✅ Routing พร้อม Regex patterns
✅ Request Body parsing (JSON, Form)
✅ Static File Server พร้อม caching
✅ HTTP Client
✅ สร้าง Mini Express Framework
✅ Streaming Responses (SSE, Chunked)
✅ Security: Path traversal prevention
```

### 📝 Exercise

**Exercise 1: REST API Server**
```
สร้าง REST API สำหรับ Blog ด้วย http module เท่านั้น:
- GET    /api/posts         - แสดง posts ทั้งหมด (+ pagination)
- GET    /api/posts/:id     - แสดง post เดียว
- POST   /api/posts         - สร้าง post
- PUT    /api/posts/:id     - แก้ไข post
- DELETE /api/posts/:id     - ลบ post
- GET    /api/posts/search  - ค้นหา
เก็บข้อมูลใน JSON file
```

**Exercise 2: Proxy Server**
```
สร้าง HTTP proxy ที่:
1. รับ request
2. Forward ไปยัง target server
3. แก้ไข response headers
4. Cache response
5. Rate limit
6. Logging
```

**Exercise 3: Chat Server ด้วย SSE**
```
สร้าง simple chat server:
- Client ส่ง message ด้วย POST /messages
- Server broadcast ด้วย SSE /events
- เก็บ history 100 messages
- Support multiple rooms
```

---

## 🔜 Part ถัดไป

**Part 06: Events และ EventEmitter** จะครอบคลุม:
- EventEmitter class
- Custom events
- Node.js built-in events
- Event patterns ใน production
- Memory leak prevention

---

*Part 05 สมบูรณ์ | ขั้นตอนที่ 141-180 จาก 1000*
