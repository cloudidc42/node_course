# Part 03: npm และการจัดการ Package
## ขั้นตอนที่ 51-90: ทุกอย่างที่ต้องรู้เกี่ยวกับ npm

---

## 🎯 เป้าหมายของ Part นี้

หลังจากเรียนจบ Part นี้ คุณจะสามารถ:
1. ใช้ npm commands ทั้งหมดได้คล่อง
2. เข้าใจ Semantic Versioning ได้
3. จัดการ dependencies ได้อย่างถูกต้อง
4. สร้าง npm package และ publish ได้
5. ใช้ npx ได้อย่างถูกต้อง
6. จัดการ monorepo ด้วย npm workspaces ได้
7. แก้ปัญหาต่างๆ ที่พบบ่อยกับ npm ได้

---

## ขั้นตอนที่ 51: npm คืออะไร?

```
npm = Node Package Manager

ecosystem:
┌─────────────────────────────────────────────────┐
│                    npm                          │
│                                                 │
│  ┌────────────┐   ┌───────────┐   ┌──────────┐  │
│  │  Registry  │   │    CLI    │   │ Website  │  │
│  │(npmjs.com) │   │  (tool)   │   │(npm.io)  │  │
│  │            │   │           │   │          │  │
│  │ 2M+ packages│  │ install   │   │ search   │  │
│  │            │   │ publish   │   │ browse   │  │
│  │            │   │ run       │   │ docs     │  │
│  └────────────┘   └───────────┘   └──────────┘  │
└─────────────────────────────────────────────────┘
```

### npm vs yarn vs pnpm

```
┌─────────────────────────────────────────────────┐
│  Package Manager Comparison                     │
├──────────┬───────┬──────────┬───────────────────┤
│ Feature  │  npm  │   yarn   │      pnpm         │
├──────────┼───────┼──────────┼───────────────────┤
│ Speed    │  ⭐⭐  │  ⭐⭐⭐   │     ⭐⭐⭐⭐        │
│ Disk use │  ⭐⭐  │  ⭐⭐⭐   │     ⭐⭐⭐⭐⭐      │
│ Workspce │  ✅   │   ✅     │       ✅          │
│ Built-in │  ✅   │   ❌     │       ❌          │
│ Popular  │ ⭐⭐⭐⭐⭐│  ⭐⭐⭐   │      ⭐⭐⭐        │
└──────────┴───────┴──────────┴───────────────────┘

แนะนำ:
- เริ่มต้น: ใช้ npm (built-in กับ Node.js)
- ความเร็ว + ประหยัด disk: ใช้ pnpm
- Enterprise/Yarn Berry: ใช้ yarn
```

---

## ขั้นตอนที่ 52: npm Commands พื้นฐาน

```bash
# ══════════════════════════════════════
# การสร้าง Project
# ══════════════════════════════════════

# สร้าง package.json แบบ interactive
npm init

# สร้างแบบ auto (ใช้ default values)
npm init -y
npm init --yes

# สร้างด้วย template
npm init react-app my-app
npm init vite@latest my-app

# ══════════════════════════════════════
# การติดตั้ง Package
# ══════════════════════════════════════

# ติดตั้ง package (save ใน dependencies)
npm install express
npm install express mongoose dotenv    # ติดตั้งหลาย package พร้อมกัน
npm i express                          # shorthand

# ติดตั้ง devDependency
npm install --save-dev nodemon
npm install -D jest eslint             # shorthand -D

# ติดตั้ง global
npm install -g nodemon
npm install -g create-react-app

# ติดตั้ง specific version
npm install express@4.18.2
npm install express@latest
npm install express@next               # pre-release

# ติดตั้ง package จาก GitHub
npm install github:expressjs/express
npm install expressjs/express#main     # specific branch

# ติดตั้ง package จาก local path
npm install ../my-local-package

# ══════════════════════════════════════
# การลบ Package
# ══════════════════════════════════════

npm uninstall express
npm remove mongoose
npm rm dotenv

# ลบ global
npm uninstall -g nodemon

# ══════════════════════════════════════
# การอัพเดท Package
# ══════════════════════════════════════

# อัพเดท package เดียว
npm update express

# อัพเดท ทั้งหมด
npm update

# ตรวจสอบ package ที่ outdated
npm outdated

# อัพเดทไปยัง major version (ใช้ npx npm-check-updates)
npx npm-check-updates -u
npm install

# ══════════════════════════════════════
# การดู Package Info
# ══════════════════════════════════════

# ดู packages ที่ติดตั้งแล้ว
npm list
npm ls

# ดูแค่ top-level
npm list --depth=0

# ดู global packages
npm list -g --depth=0

# ดูข้อมูล package
npm info express
npm view express
npm show express versions    # ดู versions ทั้งหมด

# ══════════════════════════════════════
# การค้นหา Package
# ══════════════════════════════════════

npm search "http client"
npm search lodash

# ══════════════════════════════════════
# Audit & Security
# ══════════════════════════════════════

# ตรวจสอบ security vulnerabilities
npm audit

# แก้ไข vulnerabilities อัตโนมัติ
npm audit fix

# แก้ไขแบบ force (อาจ break things)
npm audit fix --force
```

---

## ขั้นตอนที่ 53: Semantic Versioning (SemVer)

```
Semantic Versioning Format: MAJOR.MINOR.PATCH

ตัวอย่าง: 4.18.2
          │  │  │
          │  │  └── PATCH: bug fixes (backward compatible)
          │  └───── MINOR: new features (backward compatible)
          └──────── MAJOR: breaking changes

กฎการเพิ่ม version:
- PATCH: แก้ bug โดยไม่เปลี่ยน API
- MINOR: เพิ่ม features ใหม่ แต่ยังใช้ API เดิมได้
- MAJOR: เปลี่ยน API ที่อาจทำให้ code เดิม break
```

### Version Ranges ใน package.json

```json
{
  "dependencies": {
    "exact":     "4.18.2",      // ติดตั้ง version นี้เท่านั้น
    "caret":     "^4.18.2",     // >= 4.18.2 < 5.0.0 (แนะนำ)
    "tilde":     "~4.18.2",     // >= 4.18.2 < 4.19.0
    "greater":   ">4.18.2",     // มากกว่า 4.18.2
    "gte":       ">=4.18.2",    // มากกว่าหรือเท่ากับ
    "range":     "4.0.0 - 5.0.0", // ระหว่าง (inclusive)
    "wildcard":  "4.*",          // ทุก patch ของ 4.x
    "latest":    "latest",       // version ล่าสุดเสมอ
    "any":       "*"             // ทุก version (ไม่แนะนำ)
  }
}
```

```bash
# ตรวจสอบว่า version match กับ range หรือไม่
node -e "const semver = require('semver'); console.log(semver.satisfies('4.18.2', '^4.0.0'))"

# ติดตั้ง semver
npm install semver
```

```javascript
// semver-demo.js
const semver = require('semver');

// ตรวจสอบ version
console.log(semver.valid('1.2.3'));     // '1.2.3'
console.log(semver.valid('not-valid')); // null

// เปรียบเทียบ
console.log(semver.gt('2.0.0', '1.9.9'));   // true
console.log(semver.lt('1.0.0', '2.0.0'));   // true
console.log(semver.eq('1.0.0', '1.0.0'));   // true
console.log(semver.neq('1.0.0', '2.0.0')); // true

// ตรวจสอบ range
console.log(semver.satisfies('4.18.2', '^4.0.0'));  // true
console.log(semver.satisfies('5.0.0', '^4.0.0'));   // false

// เพิ่ม version
console.log(semver.inc('1.2.3', 'major')); // 2.0.0
console.log(semver.inc('1.2.3', 'minor')); // 1.3.0
console.log(semver.inc('1.2.3', 'patch')); // 1.2.4
console.log(semver.inc('1.2.3', 'prerelease', 'beta')); // 1.2.4-beta.0

// หา version ที่เหมาะสมจาก list
const versions = ['1.0.0', '2.0.0', '2.1.0', '3.0.0'];
console.log(semver.maxSatisfying(versions, '^2.0.0')); // 2.1.0
console.log(semver.minSatisfying(versions, '>=2.0.0')); // 2.0.0

// Sort versions
console.log(versions.sort(semver.compare));
console.log(versions.sort(semver.rcompare)); // reverse
```

---

## ขั้นตอนที่ 54: package.json อย่างละเอียด

```json
{
  "name": "my-awesome-package",
  "version": "1.0.0",
  "description": "คำอธิบาย package ของคุณ",
  
  "main": "dist/index.js",
  "module": "dist/index.esm.js",
  "types": "dist/index.d.ts",
  "exports": {
    ".": {
      "require": "./dist/index.cjs.js",
      "import": "./dist/index.esm.js",
      "types": "./dist/index.d.ts"
    },
    "./utils": {
      "require": "./dist/utils.cjs.js",
      "import": "./dist/utils.esm.js"
    }
  },
  
  "scripts": {
    "start": "node dist/index.js",
    "dev": "nodemon src/index.js",
    "build": "npm run clean && npm run compile",
    "compile": "tsc",
    "clean": "rm -rf dist",
    "test": "jest --coverage",
    "test:watch": "jest --watch",
    "test:ci": "jest --ci --coverage --reporters=default --reporters=jest-junit",
    "lint": "eslint src/**/*.ts",
    "lint:fix": "eslint src/**/*.ts --fix",
    "format": "prettier --write src/**/*.ts",
    "prepare": "husky install",
    "prepublishOnly": "npm run build && npm test",
    "version": "npm run format && git add -A src",
    "postversion": "git push && git push --tags"
  },
  
  "keywords": ["nodejs", "express", "api"],
  "author": {
    "name": "สมชาย ใจดี",
    "email": "somchai@example.com",
    "url": "https://somchai.dev"
  },
  "license": "MIT",
  "homepage": "https://github.com/somchai/my-package#readme",
  "bugs": {
    "url": "https://github.com/somchai/my-package/issues"
  },
  "repository": {
    "type": "git",
    "url": "https://github.com/somchai/my-package.git"
  },
  
  "dependencies": {
    "express": "^4.18.2",
    "dotenv": "^16.3.1"
  },
  "devDependencies": {
    "@types/node": "^20.0.0",
    "nodemon": "^3.0.1",
    "jest": "^29.7.0",
    "typescript": "^5.2.0",
    "eslint": "^8.50.0",
    "prettier": "^3.0.3",
    "husky": "^8.0.3",
    "lint-staged": "^14.0.1"
  },
  "peerDependencies": {
    "react": ">=17.0.0",
    "react-dom": ">=17.0.0"
  },
  "optionalDependencies": {
    "fsevents": "^2.3.3"
  },
  
  "engines": {
    "node": ">=18.0.0",
    "npm": ">=9.0.0"
  },
  "os": ["linux", "darwin", "win32"],
  "cpu": ["x64", "arm64"],
  
  "files": [
    "dist",
    "src",
    "!**/*.test.ts",
    "!**/*.spec.ts"
  ],
  
  "publishConfig": {
    "access": "public",
    "registry": "https://registry.npmjs.org"
  },
  
  "config": {
    "port": "3000"
  },
  
  "lint-staged": {
    "*.{ts,tsx}": ["eslint --fix", "prettier --write"],
    "*.{json,md}": ["prettier --write"]
  }
}
```

---

## ขั้นตอนที่ 55: npm Scripts ขั้นสูง

```json
{
  "scripts": {
    "clean": "rimraf dist",
    "prebuild": "npm run clean",
    "build": "tsc",
    "postbuild": "echo Build complete!",
    
    "dev": "concurrently \"npm run dev:api\" \"npm run dev:frontend\"",
    "dev:api": "nodemon src/server.js",
    "dev:frontend": "cd frontend && npm run dev",
    
    "test": "jest",
    "test:unit": "jest --testPathPattern=unit",
    "test:integration": "jest --testPathPattern=integration",
    "test:e2e": "playwright test",
    
    "db:migrate": "node scripts/migrate.js",
    "db:seed": "node scripts/seed.js",
    "db:reset": "npm run db:migrate && npm run db:seed",
    
    "deploy:staging": "npm run build && ssh user@staging 'cd /app && git pull && npm ci && pm2 restart app'",
    "deploy:prod": "npm run test:ci && npm run build && npm run deploy:staging"
  }
}
```

```bash
# รัน scripts
npm run dev          # รัน script ชื่อ dev
npm start            # รัน script ชื่อ start (ไม่ต้องใช้ run)
npm test             # รัน script ชื่อ test

# ส่ง arguments ไปยัง script
npm run test -- --watch
npm run start -- --port 8080

# รัน script แบบ silent (ไม่แสดง npm info)
npm run build --silent
npm run build -s

# ดู environment variables ใน npm scripts
npm run env          # แสดง env ทั้งหมด

# ใช้ cross-platform scripts
npm install cross-env
# "dev": "cross-env NODE_ENV=development node index.js"

# รัน multiple scripts พร้อมกัน
npm install concurrently
# "dev": "concurrently \"npm:server\" \"npm:client\""
```

---

## ขั้นตอนที่ 56: package-lock.json

```
package-lock.json คืออะไร?

- บันทึก exact versions ของทุก package ที่ติดตั้งจริง
- ทำให้ทุกคน install ได้ package เหมือนกัน 100%
- ควร commit ลงใน Git เสมอ
- อย่า edit ด้วยมือ!

ความแตกต่าง:
package.json       → "express": "^4.18.0"  (range)
package-lock.json  → "express": "4.18.2"   (exact)
```

```bash
# npm ci - ติดตั้งจาก package-lock.json (เร็วกว่า, reproducible)
npm ci

# ใช้สำหรับ:
# - CI/CD pipelines
# - Production deployments
# - ต้องการ reproducible builds

# npm install - อาจอัพเดท package-lock.json
npm install

# ล้าง cache และติดตั้งใหม่
rm -rf node_modules package-lock.json
npm install
```

---

## ขั้นตอนที่ 57: npm Workspaces (Monorepo)

```bash
# โครงสร้าง Monorepo
my-monorepo/
├── package.json          # root package.json
├── packages/
│   ├── api/
│   │   ├── package.json
│   │   └── src/
│   ├── web/
│   │   ├── package.json
│   │   └── src/
│   └── shared/
│       ├── package.json
│       └── src/
└── node_modules/         # shared dependencies
```

```json
// root/package.json
{
  "name": "my-monorepo",
  "version": "1.0.0",
  "private": true,
  "workspaces": [
    "packages/*"
  ],
  "scripts": {
    "dev": "npm run dev --workspaces",
    "build": "npm run build --workspaces",
    "test": "npm run test --workspaces"
  }
}
```

```bash
# รัน command ใน workspace เดียว
npm run build -w packages/api
npm run test --workspace=packages/web

# รัน ใน workspaces ทั้งหมด
npm run build --workspaces

# ติดตั้ง package ใน workspace เดียว
npm install express --workspace=packages/api

# ติดตั้ง shared package (link ระหว่าง workspaces)
npm install @my-monorepo/shared --workspace=packages/api
```

---

## ขั้นตอนที่ 58: สร้างและ Publish npm Package

### Step 1: สร้าง package

```bash
mkdir my-string-utils
cd my-string-utils
npm init -y
```

```javascript
// src/index.js

/**
 * แปลง string เป็น camelCase
 * @param {string} str 
 * @returns {string}
 */
function toCamelCase(str) {
  return str
    .toLowerCase()
    .replace(/[-_\s](.)/g, (_, char) => char.toUpperCase());
}

/**
 * แปลง string เป็น snake_case
 * @param {string} str 
 * @returns {string}
 */
function toSnakeCase(str) {
  return str
    .replace(/([A-Z])/g, '_$1')
    .toLowerCase()
    .replace(/^_/, '');
}

/**
 * แปลง string เป็น kebab-case
 * @param {string} str 
 * @returns {string}
 */
function toKebabCase(str) {
  return toSnakeCase(str).replace(/_/g, '-');
}

/**
 * ตัด whitespace และ normalize
 * @param {string} str 
 * @returns {string}
 */
function normalizeWhitespace(str) {
  return str.trim().replace(/\s+/g, ' ');
}

/**
 * ตัดคำให้สั้นลง
 * @param {string} str 
 * @param {number} maxLength 
 * @param {string} suffix 
 * @returns {string}
 */
function truncate(str, maxLength = 100, suffix = '...') {
  if (str.length <= maxLength) return str;
  return str.substring(0, maxLength - suffix.length) + suffix;
}

/**
 * Count words ใน string
 * @param {string} str 
 * @returns {number}
 */
function countWords(str) {
  return str.trim().split(/\s+/).filter(Boolean).length;
}

/**
 * นับจำนวน occurrences ของ substring
 * @param {string} str 
 * @param {string} search 
 * @returns {number}
 */
function countOccurrences(str, search) {
  const regex = new RegExp(search.replace(/[.*+?^${}()|[\]\\]/g, '\\$&'), 'g');
  return (str.match(regex) || []).length;
}

/**
 * แปลง bytes เป็น human-readable string
 * @param {number} bytes 
 * @param {number} decimals 
 * @returns {string}
 */
function formatBytes(bytes, decimals = 2) {
  if (bytes === 0) return '0 Bytes';
  const k = 1024;
  const dm = decimals < 0 ? 0 : decimals;
  const sizes = ['Bytes', 'KB', 'MB', 'GB', 'TB', 'PB'];
  const i = Math.floor(Math.log(bytes) / Math.log(k));
  return parseFloat((bytes / Math.pow(k, i)).toFixed(dm)) + ' ' + sizes[i];
}

/**
 * สร้าง random string
 * @param {number} length 
 * @param {string} charset 
 * @returns {string}
 */
function randomString(length = 10, charset = 'ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789') {
  let result = '';
  for (let i = 0; i < length; i++) {
    result += charset.charAt(Math.floor(Math.random() * charset.length));
  }
  return result;
}

/**
 * ตรวจสอบ email format
 * @param {string} email 
 * @returns {boolean}
 */
function isValidEmail(email) {
  return /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email);
}

/**
 * Escape HTML special characters
 * @param {string} str 
 * @returns {string}
 */
function escapeHTML(str) {
  const map = { '&': '&amp;', '<': '&lt;', '>': '&gt;', '"': '&quot;', "'": '&#039;' };
  return str.replace(/[&<>"']/g, char => map[char]);
}

module.exports = {
  toCamelCase,
  toSnakeCase,
  toKebabCase,
  normalizeWhitespace,
  truncate,
  countWords,
  countOccurrences,
  formatBytes,
  randomString,
  isValidEmail,
  escapeHTML
};
```

### Step 2: เขียน Tests

```bash
npm install --save-dev jest
```

```javascript
// src/__tests__/index.test.js
const {
  toCamelCase,
  toSnakeCase,
  toKebabCase,
  truncate,
  countWords,
  isValidEmail,
  escapeHTML
} = require('../index');

describe('toCamelCase', () => {
  test('converts kebab-case', () => {
    expect(toCamelCase('hello-world')).toBe('helloWorld');
  });
  
  test('converts snake_case', () => {
    expect(toCamelCase('hello_world')).toBe('helloWorld');
  });
  
  test('converts space separated', () => {
    expect(toCamelCase('hello world foo')).toBe('helloWorldFoo');
  });
});

describe('toSnakeCase', () => {
  test('converts camelCase', () => {
    expect(toSnakeCase('helloWorld')).toBe('hello_world');
  });
  
  test('handles consecutive uppercase', () => {
    expect(toSnakeCase('HTMLParser')).toBe('h_t_m_l_parser');
  });
});

describe('truncate', () => {
  test('does not truncate short strings', () => {
    expect(truncate('Hello', 10)).toBe('Hello');
  });
  
  test('truncates long strings', () => {
    expect(truncate('Hello, World!', 8)).toBe('Hello...');
  });
  
  test('uses custom suffix', () => {
    expect(truncate('Hello, World!', 8, '→')).toBe('Hello, W→');
  });
});

describe('isValidEmail', () => {
  test('validates correct email', () => {
    expect(isValidEmail('user@example.com')).toBe(true);
  });
  
  test('rejects invalid email', () => {
    expect(isValidEmail('not-an-email')).toBe(false);
    expect(isValidEmail('@example.com')).toBe(false);
    expect(isValidEmail('user@')).toBe(false);
  });
});

describe('escapeHTML', () => {
  test('escapes HTML characters', () => {
    expect(escapeHTML('<script>alert("XSS")</script>'))
      .toBe('&lt;script&gt;alert(&quot;XSS&quot;)&lt;/script&gt;');
  });
  
  test('escapes ampersand', () => {
    expect(escapeHTML('Tom & Jerry')).toBe('Tom &amp; Jerry');
  });
});
```

### Step 3: สร้าง README

````markdown
# my-string-utils

ชุดฟังก์ชันช่วยจัดการ string สำหรับ Node.js

## ติดตั้ง

```bash
npm install my-string-utils
```

## ใช้งาน

```javascript
const { toCamelCase, truncate, isValidEmail } = require('my-string-utils');

toCamelCase('hello-world');      // 'helloWorld'
truncate('Long text here', 10);  // 'Long text...'
isValidEmail('user@test.com');   // true
```
````

### Step 4: Publish

```bash
# ตั้งค่า npm account
npm login

# ตรวจสอบว่า login แล้ว
npm whoami

# ตรวจสอบ package ก่อน publish
npm pack --dry-run

# Publish
npm publish

# Publish แบบ scoped
npm publish --access public  # สำหรับ @scope/package

# Publish เป็น beta
npm version 1.1.0-beta.0
npm publish --tag beta

# อัพเดท version และ publish
npm version patch   # 1.0.0 → 1.0.1
npm version minor   # 1.0.0 → 1.1.0
npm version major   # 1.0.0 → 2.0.0
npm publish
```

---

## ขั้นตอนที่ 59: .npmrc Configuration

```bash
# ~/.npmrc (global) หรือ .npmrc ใน project folder

# กำหนด registry
registry=https://registry.npmjs.org/

# Private registry สำหรับ organization
@mycompany:registry=https://npm.mycompany.com/
//npm.mycompany.com/:_authToken=${NPM_TOKEN}

# กำหนด default values
init-author-name=สมชาย ใจดี
init-author-email=somchai@example.com
init-license=MIT
init-version=0.1.0

# Save exact versions (ไม่ใช้ ^ หรือ ~)
save-exact=true

# ใช้ legacy peer deps (แก้ conflicts บางอย่าง)
legacy-peer-deps=true

# กำหนด timeout
fetch-timeout=60000
fetch-retries=3

# Audit settings
audit=false  # ปิด auto-audit (ไม่แนะนำ)
```

---

## ขั้นตอนที่ 60: npx

```bash
# npx = รัน packages โดยไม่ต้องติดตั้ง

# รัน package ล่าสุด
npx create-react-app my-app
npx create-next-app@latest my-app

# รัน specific version
npx create-react-app@5.0.1 my-app

# รัน command จาก local node_modules
npx jest                    # รัน jest จาก node_modules
npx tsc --version           # รัน typescript compiler

# รัน script จาก package.json ที่ remote
npx github:user/repo

# ตรวจสอบว่ามี package ใน local ก่อน ถ้าไม่มีค่อย download
npx --no-install jest       # fail ถ้าไม่มีใน local

# รัน package แบบ interactive
npx -p cowsay cowsay "Hello from npm!"
```

---

## ขั้นตอนที่ 61: Packages ยอดนิยมที่ต้องรู้จัก

### HTTP/API

```bash
# Express.js - Web framework
npm install express

# Fastify - เร็วกว่า Express
npm install fastify

# axios - HTTP client
npm install axios

# node-fetch - Fetch API
npm install node-fetch

# got - HTTP client ขั้นสูง
npm install got
```

### Database

```bash
# MongoDB
npm install mongoose

# PostgreSQL
npm install pg

# MySQL
npm install mysql2

# SQLite
npm install better-sqlite3

# ORM - Prisma (แนะนำ)
npm install prisma @prisma/client

# ORM - Sequelize
npm install sequelize
```

### Authentication & Security

```bash
# Password hashing
npm install bcrypt
npm install bcryptjs  # pure JS version

# JWT
npm install jsonwebtoken

# Session
npm install express-session

# Helmet (Security headers)
npm install helmet

# Rate limiting
npm install express-rate-limit

# Input validation
npm install joi
npm install zod
npm install express-validator
```

### Utilities

```bash
# Lodash - utility functions
npm install lodash

# Day.js - date manipulation (เล็กกว่า moment)
npm install dayjs

# UUID
npm install uuid

# dotenv - environment variables
npm install dotenv

# chalk - colored terminal output
npm install chalk

# ora - loading spinners
npm install ora

# inquirer - interactive CLI
npm install inquirer
```

### Testing

```bash
# Jest - Test runner
npm install --save-dev jest

# Vitest - เร็วกว่า Jest (สำหรับ Vite projects)
npm install --save-dev vitest

# Supertest - HTTP testing
npm install --save-dev supertest

# Playwright - E2E testing
npm install --save-dev playwright

# Faker - สร้าง fake data
npm install --save-dev @faker-js/faker
```

### Development Tools

```bash
# Nodemon - auto restart
npm install --save-dev nodemon

# TypeScript
npm install --save-dev typescript @types/node

# ESLint - code linting
npm install --save-dev eslint

# Prettier - code formatting
npm install --save-dev prettier

# Husky - git hooks
npm install --save-dev husky

# lint-staged
npm install --save-dev lint-staged

# concurrently - รัน multiple commands
npm install --save-dev concurrently

# rimraf - cross-platform rm -rf
npm install --save-dev rimraf
```

---

## ขั้นตอนที่ 62: โปรเจกต์จริง - CLI Task Manager พร้อม npm packages

```bash
mkdir task-manager-cli
cd task-manager-cli
npm init -y

npm install chalk@4 inquirer@8 ora@5 cli-table3 dayjs
npm install --save-dev nodemon
```

```javascript
// src/index.js - Task Manager CLI

const fs = require('fs');
const path = require('path');
const chalk = require('chalk');
const inquirer = require('inquirer');
const Table = require('cli-table3');
const dayjs = require('dayjs');
const relativeTime = require('dayjs/plugin/relativeTime');

dayjs.extend(relativeTime);
require('dayjs/locale/th');

const DATA_FILE = path.join(__dirname, '../data/tasks.json');

// Ensure data directory exists
fs.mkdirSync(path.dirname(DATA_FILE), { recursive: true });

// ═══════════════════════════════════════════
// Data Management
// ═══════════════════════════════════════════

function loadTasks() {
  if (!fs.existsSync(DATA_FILE)) return [];
  return JSON.parse(fs.readFileSync(DATA_FILE, 'utf8'));
}

function saveTasks(tasks) {
  fs.writeFileSync(DATA_FILE, JSON.stringify(tasks, null, 2));
}

function generateId() {
  return Date.now().toString(36) + Math.random().toString(36).substr(2);
}

// ═══════════════════════════════════════════
// Display Functions
// ═══════════════════════════════════════════

function displayTasks(tasks, title = 'All Tasks') {
  if (tasks.length === 0) {
    console.log(chalk.yellow('\n  ไม่มีงานในระบบ\n'));
    return;
  }
  
  const table = new Table({
    head: [
      chalk.cyan('#'),
      chalk.cyan('Title'),
      chalk.cyan('Priority'),
      chalk.cyan('Status'),
      chalk.cyan('Due Date'),
      chalk.cyan('Created')
    ],
    colWidths: [5, 30, 10, 12, 15, 20],
    style: { head: [], border: ['gray'] }
  });
  
  tasks.forEach((task, index) => {
    const priorityColor = {
      high: chalk.red,
      medium: chalk.yellow,
      low: chalk.green
    }[task.priority] || chalk.white;
    
    const statusIcon = task.status === 'done' ? chalk.green('✓') : 
                      task.status === 'in-progress' ? chalk.yellow('⚡') : 
                      chalk.gray('○');
    
    const dueDate = task.dueDate 
      ? (dayjs(task.dueDate).isBefore(dayjs()) && task.status !== 'done' 
          ? chalk.red(dayjs(task.dueDate).format('DD/MM/YYYY'))
          : chalk.white(dayjs(task.dueDate).format('DD/MM/YYYY')))
      : chalk.gray('-');
    
    table.push([
      chalk.gray(index + 1),
      task.status === 'done' ? chalk.strikethrough(task.title) : task.title,
      priorityColor(task.priority),
      `${statusIcon} ${task.status}`,
      dueDate,
      chalk.gray(dayjs(task.createdAt).fromNow())
    ]);
  });
  
  console.log(chalk.bold(`\n  📋 ${title}`));
  console.log(table.toString());
  
  const stats = {
    total: tasks.length,
    done: tasks.filter(t => t.status === 'done').length,
    inProgress: tasks.filter(t => t.status === 'in-progress').length,
    todo: tasks.filter(t => t.status === 'todo').length
  };
  
  console.log(
    chalk.gray(`  Total: ${stats.total} | `) +
    chalk.green(`Done: ${stats.done} | `) +
    chalk.yellow(`In Progress: ${stats.inProgress} | `) +
    chalk.gray(`Todo: ${stats.todo}\n`)
  );
}

// ═══════════════════════════════════════════
// Task Operations
// ═══════════════════════════════════════════

async function addTask() {
  const answers = await inquirer.prompt([
    {
      type: 'input',
      name: 'title',
      message: 'Task title:',
      validate: input => input.trim() ? true : 'Title is required'
    },
    {
      type: 'input',
      name: 'description',
      message: 'Description (optional):'
    },
    {
      type: 'list',
      name: 'priority',
      message: 'Priority:',
      choices: [
        { name: '🔴 High', value: 'high' },
        { name: '🟡 Medium', value: 'medium' },
        { name: '🟢 Low', value: 'low' }
      ],
      default: 'medium'
    },
    {
      type: 'input',
      name: 'dueDate',
      message: 'Due date (DD/MM/YYYY, optional):',
      validate: input => {
        if (!input) return true;
        return dayjs(input, 'DD/MM/YYYY').isValid() ? true : 'Invalid date format';
      }
    },
    {
      type: 'input',
      name: 'tags',
      message: 'Tags (comma-separated, optional):'
    }
  ]);
  
  const tasks = loadTasks();
  const newTask = {
    id: generateId(),
    title: answers.title.trim(),
    description: answers.description.trim() || null,
    priority: answers.priority,
    status: 'todo',
    dueDate: answers.dueDate 
      ? dayjs(answers.dueDate, 'DD/MM/YYYY').toISOString()
      : null,
    tags: answers.tags 
      ? answers.tags.split(',').map(t => t.trim()).filter(Boolean)
      : [],
    createdAt: new Date().toISOString(),
    updatedAt: new Date().toISOString()
  };
  
  tasks.push(newTask);
  saveTasks(tasks);
  
  console.log(chalk.green('\n  ✅ Task added successfully!\n'));
}

async function updateTaskStatus() {
  const tasks = loadTasks();
  
  if (tasks.length === 0) {
    console.log(chalk.yellow('\n  ไม่มีงานให้อัพเดท\n'));
    return;
  }
  
  const { taskIndex } = await inquirer.prompt([
    {
      type: 'list',
      name: 'taskIndex',
      message: 'Select task to update:',
      choices: tasks.map((task, index) => ({
        name: `${index + 1}. [${task.status}] ${task.title}`,
        value: index
      }))
    }
  ]);
  
  const { status } = await inquirer.prompt([
    {
      type: 'list',
      name: 'status',
      message: 'New status:',
      choices: [
        { name: '⬜ Todo', value: 'todo' },
        { name: '⚡ In Progress', value: 'in-progress' },
        { name: '✅ Done', value: 'done' },
        { name: '🚫 Cancelled', value: 'cancelled' }
      ]
    }
  ]);
  
  tasks[taskIndex].status = status;
  tasks[taskIndex].updatedAt = new Date().toISOString();
  if (status === 'done') {
    tasks[taskIndex].completedAt = new Date().toISOString();
  }
  
  saveTasks(tasks);
  console.log(chalk.green('\n  ✅ Task updated!\n'));
}

async function deleteTask() {
  const tasks = loadTasks();
  
  if (tasks.length === 0) {
    console.log(chalk.yellow('\n  ไม่มีงานให้ลบ\n'));
    return;
  }
  
  const { taskIndex, confirm } = await inquirer.prompt([
    {
      type: 'list',
      name: 'taskIndex',
      message: 'Select task to delete:',
      choices: tasks.map((task, index) => ({
        name: `${index + 1}. ${task.title}`,
        value: index
      }))
    },
    {
      type: 'confirm',
      name: 'confirm',
      message: (answers) => `Delete "${tasks[answers.taskIndex].title}"?`,
      default: false
    }
  ]);
  
  if (confirm) {
    const deleted = tasks.splice(taskIndex, 1)[0];
    saveTasks(tasks);
    console.log(chalk.green(`\n  ✅ Deleted: "${deleted.title}"\n`));
  } else {
    console.log(chalk.gray('\n  Cancelled\n'));
  }
}

// ═══════════════════════════════════════════
// Main Menu
// ═══════════════════════════════════════════

async function main() {
  console.clear();
  console.log(chalk.bold.cyan('╔════════════════════════════════╗'));
  console.log(chalk.bold.cyan('║      Task Manager CLI v1.0     ║'));
  console.log(chalk.bold.cyan('╚════════════════════════════════╝\n'));
  
  while (true) {
    const tasks = loadTasks();
    const { action } = await inquirer.prompt([
      {
        type: 'list',
        name: 'action',
        message: 'What would you like to do?',
        choices: [
          { name: `📋 View all tasks (${tasks.length})`, value: 'view' },
          { name: '➕ Add new task', value: 'add' },
          { name: '✏️  Update task status', value: 'update' },
          { name: '🔍 Filter tasks', value: 'filter' },
          { name: '🗑️  Delete task', value: 'delete' },
          { name: '📊 Statistics', value: 'stats' },
          new inquirer.Separator(),
          { name: '👋 Exit', value: 'exit' }
        ]
      }
    ]);
    
    switch (action) {
      case 'view':
        displayTasks(loadTasks());
        break;
        
      case 'add':
        await addTask();
        break;
        
      case 'update':
        await updateTaskStatus();
        break;
        
      case 'filter': {
        const { filterBy } = await inquirer.prompt([
          {
            type: 'list',
            name: 'filterBy',
            message: 'Filter by:',
            choices: [
              { name: 'Priority: High', value: { priority: 'high' } },
              { name: 'Priority: Medium', value: { priority: 'medium' } },
              { name: 'Status: Todo', value: { status: 'todo' } },
              { name: 'Status: In Progress', value: { status: 'in-progress' } },
              { name: 'Status: Done', value: { status: 'done' } },
              { name: 'Overdue tasks', value: 'overdue' }
            ]
          }
        ]);
        
        let filtered;
        if (filterBy === 'overdue') {
          filtered = tasks.filter(t => 
            t.dueDate && 
            dayjs(t.dueDate).isBefore(dayjs()) && 
            t.status !== 'done'
          );
          displayTasks(filtered, 'Overdue Tasks');
        } else {
          const [key, value] = Object.entries(filterBy)[0];
          filtered = tasks.filter(t => t[key] === value);
          displayTasks(filtered, `Filtered by ${key}: ${value}`);
        }
        break;
      }
        
      case 'delete':
        await deleteTask();
        break;
        
      case 'stats': {
        const allTasks = loadTasks();
        console.log(chalk.bold('\n  📊 Statistics\n'));
        console.log(`  Total tasks: ${allTasks.length}`);
        console.log(`  Todo: ${allTasks.filter(t => t.status === 'todo').length}`);
        console.log(`  In Progress: ${allTasks.filter(t => t.status === 'in-progress').length}`);
        console.log(`  Done: ${allTasks.filter(t => t.status === 'done').length}`);
        console.log(`  High Priority: ${allTasks.filter(t => t.priority === 'high').length}`);
        
        const overdue = allTasks.filter(t => 
          t.dueDate && dayjs(t.dueDate).isBefore(dayjs()) && t.status !== 'done'
        );
        if (overdue.length > 0) {
          console.log(chalk.red(`  Overdue: ${overdue.length}`));
        }
        console.log('');
        break;
      }
        
      case 'exit':
        console.log(chalk.cyan('\n  Bye! 👋\n'));
        process.exit(0);
    }
  }
}

main().catch(console.error);
```

```json
// package.json
{
  "name": "task-manager-cli",
  "version": "1.0.0",
  "description": "Task Manager ที่ใช้งานผ่าน Terminal",
  "main": "src/index.js",
  "scripts": {
    "start": "node src/index.js",
    "dev": "nodemon src/index.js"
  },
  "bin": {
    "task-manager": "./src/index.js"
  }
}
```

---

## ขั้นตอนที่ 63: แก้ปัญหาที่พบบ่อย

```bash
# ══════════════════════════════════════
# ปัญหา: EACCES permission denied (global install)
# ══════════════════════════════════════

# วิธีแก้บน macOS/Linux: เปลี่ยน npm default directory
mkdir ~/.npm-global
npm config set prefix '~/.npm-global'
# เพิ่มใน ~/.profile หรือ ~/.bashrc:
# export PATH=~/.npm-global/bin:$PATH

# ══════════════════════════════════════
# ปัญหา: ERESOLVE peer dependency conflict
# ══════════════════════════════════════

npm install --legacy-peer-deps
# หรือ
npm install --force  # ใช้เฉพาะเมื่อจำเป็น

# ══════════════════════════════════════
# ปัญหา: node_modules ขนาดใหญ่
# ══════════════════════════════════════

# ล้าง cache
npm cache clean --force

# ลบ node_modules และติดตั้งใหม่
rm -rf node_modules
npm install

# ใช้ pnpm แทน (ประหยัด disk space)
npm install -g pnpm
pnpm install

# ══════════════════════════════════════
# ปัญหา: Package version conflict
# ══════════════════════════════════════

# ดูว่า conflict อยู่ที่ไหน
npm ls package-name

# อัพเดทให้เป็น compatible versions
npm install package-a@latest package-b@latest

# ══════════════════════════════════════
# ปัญหา: Cannot find module
# ══════════════════════════════════════

# ตรวจสอบว่าติดตั้งแล้ว
npm ls express

# ถ้าไม่มี ติดตั้ง
npm install express

# ══════════════════════════════════════
# ปัญหา: package-lock.json conflict ใน Git
# ══════════════════════════════════════

# ลบและสร้างใหม่
rm package-lock.json
npm install
git add package-lock.json
```

---

## ขั้นตอนที่ 64: Security Best Practices

```bash
# ตรวจสอบ vulnerabilities
npm audit

# ดูรายละเอียด
npm audit --json | node -e "
  const data = JSON.parse(require('fs').readFileSync('/dev/stdin', 'utf8'));
  const critical = data.vulnerabilities;
  Object.entries(critical).forEach(([name, info]) => {
    if (info.severity === 'critical' || info.severity === 'high') {
      console.log(info.severity.toUpperCase(), ':', name, info.range);
    }
  });
"

# แก้อัตโนมัติ
npm audit fix

# ดู outdated packages พร้อม security info
npx better-npm-audit audit
```

```javascript
// ตรวจสอบ package ก่อนติดตั้ง

// ดู package info
npm show <package-name>

// ดู download stats
npm show <package-name> downloads

// ดู dependencies
npm show <package-name> dependencies

// ตรวจสอบบน Snyk
npx snyk test

// ตรวจสอบ license
npx license-checker --summary
```

---

## ขั้นตอนที่ 65: สรุป Part 03 และ Exercise

### สิ่งที่เรียนรู้ใน Part 03

```
✅ npm commands ทั้งหมด
✅ Semantic Versioning
✅ package.json อย่างละเอียด
✅ npm Scripts ขั้นสูง
✅ package-lock.json
✅ npm Workspaces (Monorepo)
✅ สร้างและ publish npm package
✅ npx usage
✅ Packages ยอดนิยม
✅ แก้ปัญหา npm ทั่วไป
✅ Security best practices
```

### 📝 Exercise

**Exercise 1: สร้าง npm Package**
```
สร้าง package ชื่อ thai-utils ที่มีฟังก์ชัน:
1. เลข → คำอ่านภาษาไทย (1 → หนึ่ง)
2. แปลง Buddhist Era ↔ CE (2567 ↔ 2024)
3. Format เงินบาท (1234567.89 → 1,234,567.89 บาท)
4. ตรวจสอบเลขบัตรประชาชน
5. แปลงเบอร์โทรเป็น format มาตรฐาน
พร้อม tests และ publish ขึ้น npm
```

**Exercise 2: CLI Tool**
```
สร้าง CLI tool ชื่อ weather-cli ที่:
1. รับ city name จาก argument
2. เรียก OpenWeatherMap API
3. แสดงอุณหภูมิ, ความชื้น, สภาพอากาศ
4. มี --format flag (json/text)
5. มี --cache flag เพื่อ cache ผลลัพธ์
ติดตั้งได้ด้วย npm install -g weather-cli
```

**Exercise 3: Build System**
```
สร้าง build script ที่:
1. อ่านไฟล์ src/
2. TypeScript compile
3. Bundle ด้วย esbuild
4. Generate types
5. Compress output
6. Generate checksums
ทำเป็น npm package ที่ใช้ได้
```

---

## 🔜 Part ถัดไป

**Part 04: File System (fs module)** จะครอบคลุม:
- อ่านและเขียนไฟล์
- Directory operations
- File watching
- Streams สำหรับไฟล์ใหญ่
- glob patterns
- สร้าง file-based database

---

*Part 03 สมบูรณ์ | ขั้นตอนที่ 51-90 จาก 1000*
