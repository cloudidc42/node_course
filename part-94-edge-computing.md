# Part 94 | ขั้นตอนที่ 1681-1700 จาก 1000+

## Edge Computing และ Serverless Edge Functions

ในส่วนนี้เราจะเรียนรู้ Edge Computing ด้วย Cloudflare Workers, Vercel Edge Functions, และการลด latency สำหรับ Node.js applications

---

## ขั้นตอนที่ 1681: Edge Computing Fundamentals

Edge Computing คือการประมวลผลที่ "ขอบ" ของ network ใกล้กับผู้ใช้ แทนที่จะส่งข้อมูลไปยัง central data center

**ข้อดีของ Edge Computing:**
- Latency ต่ำกว่ามาก (5-20ms แทนที่จะเป็น 100-200ms)
- Distributed globally
- Auto-scaling
- Cost-effective สำหรับ simple operations

```javascript
// cloudflare-worker.js
// Simple Cloudflare Worker for edge caching and routing

addEventListener('fetch', event => {
  event.respondWith(handleRequest(event.request));
});

async function handleRequest(request) {
  const url = new URL(request.url);
  
  // Edge routing
  if (url.pathname.startsWith('/api/')) {
    return handleAPI(request, url);
  }
  
  if (url.pathname.startsWith('/static/')) {
    return handleStatic(request, url);
  }
  
  return handleDefault(request);
}

async function handleAPI(request, url) {
  // Try cache first
  const cacheKey = new Request(url.toString(), request);
  const cache = caches.default;
  
  if (request.method === 'GET') {
    let cachedResponse = await cache.match(cacheKey);
    
    if (cachedResponse) {
      return new Response(cachedResponse.body, {
        ...cachedResponse,
        headers: {
          ...Object.fromEntries(cachedResponse.headers),
          'X-Cache': 'HIT',
          'X-Edge-Location': request.cf?.colo || 'unknown'
        }
      });
    }
  }
  
  // Forward to origin
  const originUrl = `https://api.example.com${url.pathname}${url.search}`;
  const response = await fetch(originUrl, {
    method: request.method,
    headers: request.headers,
    body: request.method !== 'GET' ? request.body : undefined
  });
  
  const modifiedResponse = new Response(response.body, {
    status: response.status,
    headers: {
      ...Object.fromEntries(response.headers),
      'X-Cache': 'MISS',
      'X-Edge-Location': request.cf?.colo || 'unknown',
      'X-Country': request.cf?.country || 'unknown'
    }
  });
  
  // Cache successful GET responses
  if (request.method === 'GET' && response.ok) {
    const responseToCache = modifiedResponse.clone();
    event.waitUntil(cache.put(cacheKey, responseToCache));
  }
  
  return modifiedResponse;
}

async function handleStatic(request, url) {
  // Serve static assets from edge cache
  const cache = caches.default;
  let response = await cache.match(request);
  
  if (!response) {
    response = await fetch(`https://assets.example.com${url.pathname}`);
    event.waitUntil(cache.put(request, response.clone()));
  }
  
  return response;
}

async function handleDefault(request) {
  return new Response('Hello from Edge!', {
    headers: { 'content-type': 'text/plain' }
  });
}
```

---

## ขั้นตอนที่ 1682: Cloudflare Workers KV Storage

```javascript
// edge-kv-worker.js
// Cloudflare Workers with KV storage

// In wrangler.toml:
// [[kv_namespaces]]
// binding = "SESSIONS"
// id = "your-namespace-id"

addEventListener('fetch', event => {
  event.respondWith(handleRequest(event.request));
});

const ALLOWED_ORIGINS = ['https://app.example.com', 'https://www.example.com'];

async function handleRequest(request) {
  // CORS
  const origin = request.headers.get('Origin');
  const corsHeaders = ALLOWED_ORIGINS.includes(origin) ? {
    'Access-Control-Allow-Origin': origin,
    'Access-Control-Allow-Methods': 'GET, POST, DELETE, OPTIONS',
    'Access-Control-Allow-Headers': 'Content-Type, Authorization'
  } : {};
  
  if (request.method === 'OPTIONS') {
    return new Response(null, { status: 204, headers: corsHeaders });
  }
  
  const url = new URL(request.url);
  
  // Session management at edge
  if (url.pathname.startsWith('/session/')) {
    const sessionId = url.pathname.split('/')[2];
    return handleSession(request, sessionId, corsHeaders);
  }
  
  // Rate limiting at edge
  const clientIP = request.headers.get('CF-Connecting-IP');
  const rateLimitResult = await checkRateLimit(clientIP);
  
  if (!rateLimitResult.allowed) {
    return new Response(JSON.stringify({ error: 'Rate limit exceeded' }), {
      status: 429,
      headers: {
        'Content-Type': 'application/json',
        'Retry-After': String(rateLimitResult.retryAfter),
        ...corsHeaders
      }
    });
  }
  
  return new Response('OK', { headers: corsHeaders });
}

async function handleSession(request, sessionId, corsHeaders) {
  switch (request.method) {
    case 'GET': {
      const data = await SESSIONS.get(sessionId, 'json');
      if (!data) {
        return new Response(JSON.stringify({ error: 'Session not found' }), {
          status: 404,
          headers: { 'Content-Type': 'application/json', ...corsHeaders }
        });
      }
      return new Response(JSON.stringify(data), {
        headers: { 'Content-Type': 'application/json', ...corsHeaders }
      });
    }
    
    case 'POST': {
      const body = await request.json();
      await SESSIONS.put(sessionId, JSON.stringify({
        ...body,
        updatedAt: new Date().toISOString()
      }), {
        expirationTtl: 3600 // 1 hour
      });
      return new Response(JSON.stringify({ success: true }), {
        headers: { 'Content-Type': 'application/json', ...corsHeaders }
      });
    }
    
    case 'DELETE': {
      await SESSIONS.delete(sessionId);
      return new Response(JSON.stringify({ success: true }), {
        headers: { 'Content-Type': 'application/json', ...corsHeaders }
      });
    }
    
    default:
      return new Response('Method not allowed', { status: 405 });
  }
}

async function checkRateLimit(clientIP) {
  const key = `rate:${clientIP}`;
  const now = Math.floor(Date.now() / 1000);
  const windowStart = now - 60; // 1 minute window
  
  const data = await SESSIONS.get(key, 'json') || { requests: [], window: now };
  
  // Clean old requests
  data.requests = data.requests.filter(t => t > windowStart);
  
  if (data.requests.length >= 100) {
    return { allowed: false, retryAfter: 60 - (now - data.requests[0]) };
  }
  
  data.requests.push(now);
  await SESSIONS.put(key, JSON.stringify(data), { expirationTtl: 120 });
  
  return { allowed: true };
}
```

---

## ขั้นตอนที่ 1683: Vercel Edge Functions

```javascript
// api/edge-function.js
// Vercel Edge Function

export const config = { runtime: 'edge' };

export default async function handler(req) {
  const { searchParams } = new URL(req.url);
  const userId = searchParams.get('userId');
  
  if (!userId) {
    return new Response(JSON.stringify({ error: 'userId required' }), {
      status: 400,
      headers: { 'content-type': 'application/json' }
    });
  }
  
  // Geolocation from Vercel edge
  const country = req.geo?.country || 'unknown';
  const city = req.geo?.city || 'unknown';
  
  // Get user preferences from edge KV
  const userPrefs = await getUserPreferences(userId);
  
  // Personalize response at edge
  const response = {
    userId,
    location: { country, city },
    preferences: userPrefs,
    content: getLocalizedContent(country, userPrefs?.language),
    timestamp: new Date().toISOString()
  };
  
  return new Response(JSON.stringify(response), {
    headers: {
      'content-type': 'application/json',
      'cache-control': 'private, max-age=60',
      'x-edge-region': process.env.VERCEL_REGION || 'unknown'
    }
  });
}

async function getUserPreferences(userId) {
  const kv = process.env.KV_URL;
  // Simplified - would use actual KV store
  return { language: 'en', theme: 'dark' };
}

function getLocalizedContent(country, language = 'en') {
  const content = {
    th: { greeting: 'สวัสดี', message: 'ยินดีต้อนรับ' },
    en: { greeting: 'Hello', message: 'Welcome' },
    ja: { greeting: 'こんにちは', message: 'ようこそ' }
  };
  
  const langMap = { TH: 'th', JP: 'ja' };
  const lang = language || langMap[country] || 'en';
  return content[lang] || content.en;
}
```

---

## ขั้นตอนที่ 1684-1700: Edge Middleware และ A/B Testing

```javascript
// edge-middleware.js
// Next.js Edge Middleware สำหรับ A/B testing

import { NextResponse } from 'next/server';
import type { NextRequest } from 'next/server';

export const config = {
  matcher: ['/landing/:path*', '/pricing']
};

export function middleware(request: NextRequest) {
  const url = request.nextUrl.clone();
  
  // A/B Testing at edge
  const abTestCookie = request.cookies.get('ab-test');
  let variant = abTestCookie?.value;
  
  if (!variant) {
    variant = Math.random() < 0.5 ? 'control' : 'treatment';
  }
  
  // Rewrite URL based on variant
  if (url.pathname.startsWith('/landing')) {
    if (variant === 'treatment') {
      url.pathname = url.pathname.replace('/landing', '/landing-v2');
    }
  }
  
  const response = NextResponse.rewrite(url);
  
  // Set variant cookie
  if (!abTestCookie) {
    response.cookies.set('ab-test', variant, {
      maxAge: 60 * 60 * 24 * 30, // 30 days
      httpOnly: true,
      sameSite: 'lax'
    });
  }
  
  // Track experiment
  response.headers.set('x-ab-variant', variant);
  
  return response;
}

// Edge authentication
export function authMiddleware(request: NextRequest) {
  const token = request.headers.get('Authorization')?.replace('Bearer ', '');
  
  if (!token) {
    return NextResponse.json({ error: 'Unauthorized' }, { status: 401 });
  }
  
  try {
    // Verify JWT at edge (using Web Crypto API)
    const isValid = verifyJWTAtEdge(token, process.env.JWT_SECRET!);
    
    if (!isValid) {
      return NextResponse.json({ error: 'Invalid token' }, { status: 401 });
    }
    
    return NextResponse.next();
  } catch {
    return NextResponse.json({ error: 'Token verification failed' }, { status: 401 });
  }
}

async function verifyJWTAtEdge(token: string, secret: string): Promise<boolean> {
  try {
    const parts = token.split('.');
    if (parts.length !== 3) return false;
    
    const encoder = new TextEncoder();
    const key = await crypto.subtle.importKey(
      'raw',
      encoder.encode(secret),
      { name: 'HMAC', hash: 'SHA-256' },
      false,
      ['verify']
    );
    
    const signature = base64UrlToUint8Array(parts[2]);
    const data = encoder.encode(`${parts[0]}.${parts[1]}`);
    
    return crypto.subtle.verify('HMAC', key, signature, data);
  } catch {
    return false;
  }
}

function base64UrlToUint8Array(base64url: string): Uint8Array {
  const base64 = base64url.replace(/-/g, '+').replace(/_/g, '/');
  const binary = atob(base64);
  const uint8Array = new Uint8Array(binary.length);
  for (let i = 0; i < binary.length; i++) {
    uint8Array[i] = binary.charCodeAt(i);
  }
  return uint8Array;
}
```

---

## แบบฝึกหัด

### Exercise 1: Edge Rate Limiting
สร้าง Cloudflare Worker ที่ implement sliding window rate limiting ต่อ IP

### Exercise 2: Geolocation Routing
สร้าง edge function ที่ redirect ผู้ใช้ไปยัง data center ที่ใกล้ที่สุดตาม geolocation

### Exercise 3: Edge A/B Testing
สร้าง A/B testing system ที่ track conversion rates และ auto-stop poor performing variants

### คำถามทบทวน
1. Edge Computing เหมาะกับ use cases อะไร และไม่เหมาะกับอะไร?
2. ข้อจำกัดของ Edge Functions เมื่อเทียบกับ Node.js แบบเต็ม?
3. Cold start problem ใน edge functions คืออะไร?

---

*ต่อไป: Part 95 - AI/LLM Integration*
