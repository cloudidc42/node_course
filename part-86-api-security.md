# Part 86 | ขั้นตอนที่ 1521-1540 จาก 1000+

## Advanced API Security สำหรับ Node.js

ในส่วนนี้เราจะเรียนรู้เรื่อง Zero-Trust security, mTLS, API key management, OAuth2/OIDC อย่างละเอียด

---

## ขั้นตอนที่ 1521: Zero-Trust Security Model

```javascript
// zero-trust.js
// "Never trust, always verify" - ตรวจสอบทุก request

const jwt = require('jsonwebtoken');
const express = require('express');

// Zero-Trust Middleware Stack
class ZeroTrustMiddleware {
  constructor(options = {}) {
    this.options = {
      requireTLS: options.requireTLS !== false,
      requireAuth: options.requireAuth !== false,
      requireMFA: options.requireMFA || false,
      ipAllowlist: options.ipAllowlist || null,
      maxRequestAge: options.maxRequestAge || 300, // 5 minutes
      ...options
    };
  }

  // 1. Verify TLS
  verifyTLS() {
    return (req, res, next) => {
      if (this.options.requireTLS && !req.secure && req.headers['x-forwarded-proto'] !== 'https') {
        return res.status(403).json({
          error: 'HTTPS required',
          code: 'TLS_REQUIRED'
        });
      }
      next();
    };
  }

  // 2. Verify request freshness (prevent replay attacks)
  verifyRequestFreshness() {
    return (req, res, next) => {
      const timestamp = req.headers['x-request-timestamp'];
      
      if (!timestamp) {
        return res.status(400).json({
          error: 'Missing request timestamp',
          code: 'MISSING_TIMESTAMP'
        });
      }
      
      const requestTime = parseInt(timestamp);
      const now = Math.floor(Date.now() / 1000);
      const age = Math.abs(now - requestTime);
      
      if (age > this.options.maxRequestAge) {
        return res.status(400).json({
          error: `Request too old: ${age}s (max: ${this.options.maxRequestAge}s)`,
          code: 'REQUEST_EXPIRED'
        });
      }
      
      next();
    };
  }

  // 3. Verify request signature (HMAC)
  verifySignature(secretKey) {
    return (req, res, next) => {
      const signature = req.headers['x-signature'];
      const timestamp = req.headers['x-request-timestamp'];
      
      if (!signature) {
        return res.status(401).json({
          error: 'Missing signature',
          code: 'MISSING_SIGNATURE'
        });
      }
      
      const crypto = require('crypto');
      const body = req.rawBody || '';
      const stringToSign = `${req.method}\n${req.path}\n${timestamp}\n${body}`;
      
      const expectedSignature = crypto
        .createHmac('sha256', secretKey)
        .update(stringToSign)
        .digest('hex');
      
      // Constant time comparison to prevent timing attacks
      const sigBuffer = Buffer.from(signature, 'hex');
      const expectedBuffer = Buffer.from(expectedSignature, 'hex');
      
      if (sigBuffer.length !== expectedBuffer.length || 
          !crypto.timingSafeEqual(sigBuffer, expectedBuffer)) {
        return res.status(401).json({
          error: 'Invalid signature',
          code: 'INVALID_SIGNATURE'
        });
      }
      
      next();
    };
  }

  // 4. IP allowlist verification
  verifyIPAllowlist() {
    return (req, res, next) => {
      if (!this.options.ipAllowlist) return next();
      
      const clientIP = req.ip || 
                       req.headers['x-forwarded-for']?.split(',')[0]?.trim() ||
                       req.socket?.remoteAddress;
      
      if (!this.options.ipAllowlist.includes(clientIP)) {
        // Log suspicious access
        console.warn(`Blocked access from unauthorized IP: ${clientIP}`);
        
        return res.status(403).json({
          error: 'Access denied',
          code: 'IP_NOT_ALLOWED'
        });
      }
      
      next();
    };
  }

  // 5. Verify JWT with additional claims
  verifyJWT(options = {}) {
    return async (req, res, next) => {
      const token = this.extractToken(req);
      
      if (!token) {
        return res.status(401).json({
          error: 'No authentication token',
          code: 'NO_TOKEN'
        });
      }
      
      try {
        const decoded = jwt.verify(token, process.env.JWT_PUBLIC_KEY, {
          algorithms: ['RS256'], // Use asymmetric signing
          issuer: process.env.JWT_ISSUER,
          audience: process.env.JWT_AUDIENCE
        });
        
        // Additional claim verification
        if (options.requireMFA && !decoded.mfa_verified) {
          return res.status(403).json({
            error: 'MFA required',
            code: 'MFA_REQUIRED'
          });
        }
        
        // Check token hasn't been revoked
        if (await this.isTokenRevoked(decoded.jti)) {
          return res.status(401).json({
            error: 'Token has been revoked',
            code: 'TOKEN_REVOKED'
          });
        }
        
        req.user = decoded;
        next();
        
      } catch (err) {
        if (err.name === 'TokenExpiredError') {
          return res.status(401).json({
            error: 'Token expired',
            code: 'TOKEN_EXPIRED'
          });
        }
        
        return res.status(401).json({
          error: 'Invalid token',
          code: 'INVALID_TOKEN'
        });
      }
    };
  }

  extractToken(req) {
    const authHeader = req.headers.authorization;
    if (authHeader?.startsWith('Bearer ')) {
      return authHeader.substring(7);
    }
    return null;
  }

  async isTokenRevoked(jti) {
    if (!jti) return false;
    // Check token revocation list in Redis
    const revoked = await redis.get(`revoked:${jti}`);
    return revoked !== null;
  }
}

module.exports = { ZeroTrustMiddleware };
```

---

## ขั้นตอนที่ 1522: Mutual TLS (mTLS)

```javascript
// mtls-server.js
// Mutual TLS - ทั้ง client และ server ยืนยันตัวตนซึ่งกันและกัน

const https = require('https');
const fs = require('fs');
const express = require('express');
const tls = require('tls');

const app = express();

// mTLS Server Setup
const serverOptions = {
  // Server certificate
  cert: fs.readFileSync('/certs/server.crt'),
  key: fs.readFileSync('/certs/server.key'),
  
  // CA certificate (used to verify client certificates)
  ca: fs.readFileSync('/certs/ca.crt'),
  
  // Require client certificate
  requestCert: true,
  rejectUnauthorized: true,  // Reject if cert not valid
  
  // TLS version
  minVersion: 'TLSv1.3'
};

// Middleware to extract client certificate info
app.use((req, res, next) => {
  const cert = req.socket.getPeerCertificate();
  
  if (!cert || Object.keys(cert).length === 0) {
    return res.status(401).json({
      error: 'Client certificate required',
      code: 'NO_CLIENT_CERT'
    });
  }
  
  req.clientCert = {
    subject: cert.subject,
    issuer: cert.issuer,
    serialNumber: cert.serialNumber,
    validFrom: cert.valid_from,
    validTo: cert.valid_to,
    fingerprint: cert.fingerprint
  };
  
  // Verify certificate is in allow list
  const allowedCertSubjects = process.env.ALLOWED_CERT_SUBJECTS?.split(',') || [];
  if (allowedCertSubjects.length > 0) {
    const clientCN = cert.subject.CN;
    if (!allowedCertSubjects.includes(clientCN)) {
      return res.status(403).json({
        error: 'Unauthorized client certificate',
        code: 'UNAUTHORIZED_CERT',
        cn: clientCN
      });
    }
  }
  
  next();
});

// Service-to-Service authentication route
app.get('/api/secure-data', (req, res) => {
  res.json({
    message: 'Secure data for verified client',
    clientCN: req.clientCert.subject.CN,
    timestamp: new Date().toISOString()
  });
});

const server = https.createServer(serverOptions, app);

server.listen(443, () => {
  console.log('mTLS server running on port 443');
});

// mTLS Client
const mtlsClient = () => {
  const options = {
    hostname: 'api.example.com',
    port: 443,
    path: '/api/secure-data',
    method: 'GET',
    
    // Client certificate
    cert: fs.readFileSync('/certs/client.crt'),
    key: fs.readFileSync('/certs/client.key'),
    
    // Trust server CA
    ca: fs.readFileSync('/certs/ca.crt'),
    
    // Enforce TLS 1.3
    minVersion: 'TLSv1.3'
  };
  
  return new Promise((resolve, reject) => {
    const req = https.request(options, (res) => {
      let data = '';
      res.on('data', chunk => data += chunk);
      res.on('end', () => {
        resolve({ statusCode: res.statusCode, body: JSON.parse(data) });
      });
    });
    
    req.on('error', reject);
    req.end();
  });
};

module.exports = { server, mtlsClient };
```

---

## ขั้นตอนที่ 1523: API Key Management

```javascript
// api-key-management.js
// Enterprise API key system

const crypto = require('crypto');
const { Pool } = require('pg');

class APIKeyManager {
  constructor(db, cache) {
    this.db = db;
    this.cache = cache;
    this.keyPrefix = 'sk_'; // Secret key prefix
  }

  // Generate new API key
  async generateKey(options = {}) {
    const {
      name,
      userId,
      scopes = ['read'],
      expiresAt = null,
      rateLimit = { requests: 1000, windowMs: 3600000 },
      metadata = {}
    } = options;

    // Generate cryptographically secure key
    const keyBytes = crypto.randomBytes(32);
    const keySecret = `${this.keyPrefix}${keyBytes.toString('base64url')}`;
    
    // Hash for storage (never store plaintext)
    const keyHash = crypto.createHash('sha256').update(keySecret).digest('hex');
    
    const result = await this.db.query(`
      INSERT INTO api_keys 
        (id, name, key_hash, user_id, scopes, expires_at, rate_limit, metadata, created_at)
      VALUES 
        ($1, $2, $3, $4, $5, $6, $7, $8, NOW())
      RETURNING id, name, scopes, expires_at, created_at
    `, [
      crypto.randomUUID(),
      name,
      keyHash,
      userId,
      JSON.stringify(scopes),
      expiresAt,
      JSON.stringify(rateLimit),
      JSON.stringify(metadata)
    ]);

    // Return the key ONCE - user must save it
    return {
      ...result.rows[0],
      key: keySecret,  // This is shown only ONCE
      warning: 'Store this key securely - it will not be shown again'
    };
  }

  // Validate API key
  async validateKey(rawKey) {
    if (!rawKey || !rawKey.startsWith(this.keyPrefix)) {
      return null;
    }

    const keyHash = crypto.createHash('sha256').update(rawKey).digest('hex');

    // Check cache first
    const cacheKey = `apikey:${keyHash}`;
    const cached = await this.cache.get(cacheKey);
    
    if (cached) {
      return cached.value.valid ? cached.value.data : null;
    }

    // Lookup in database
    const result = await this.db.query(`
      SELECT 
        id, user_id, name, scopes, expires_at, rate_limit,
        revoked, revoked_at, last_used_at
      FROM api_keys 
      WHERE key_hash = $1
    `, [keyHash]);

    if (result.rows.length === 0) {
      await this.cache.set(cacheKey, { valid: false }, { ttl: 300 });
      return null;
    }

    const keyData = result.rows[0];

    // Check if revoked
    if (keyData.revoked) {
      await this.cache.set(cacheKey, { valid: false }, { ttl: 300 });
      return null;
    }

    // Check expiration
    if (keyData.expires_at && new Date(keyData.expires_at) < new Date()) {
      await this.cache.set(cacheKey, { valid: false }, { ttl: 300 });
      return null;
    }

    // Update last used
    this.db.query('UPDATE api_keys SET last_used_at = NOW() WHERE id = $1', [keyData.id])
      .catch(() => {});

    await this.cache.set(cacheKey, { valid: true, data: keyData }, { ttl: 60 });
    
    return keyData;
  }

  // Check scope
  hasScope(keyData, requiredScope) {
    const scopes = JSON.parse(keyData.scopes);
    return scopes.includes(requiredScope) || scopes.includes('*');
  }

  // Revoke key
  async revokeKey(keyId, userId) {
    const result = await this.db.query(`
      UPDATE api_keys 
      SET revoked = true, revoked_at = NOW() 
      WHERE id = $1 AND user_id = $2
      RETURNING id
    `, [keyId, userId]);

    if (result.rows.length === 0) {
      throw new Error('API key not found or not authorized');
    }

    // Clear cache
    const key = await this.db.query(
      'SELECT key_hash FROM api_keys WHERE id = $1',
      [keyId]
    );
    
    if (key.rows[0]) {
      await this.cache.delete(`apikey:${key.rows[0].key_hash}`);
    }

    return { revoked: true, keyId };
  }

  // List keys for user
  async listKeys(userId) {
    const result = await this.db.query(`
      SELECT id, name, scopes, expires_at, created_at, last_used_at, revoked
      FROM api_keys
      WHERE user_id = $1
      ORDER BY created_at DESC
    `, [userId]);

    return result.rows;
  }
}

// API Key Middleware
function createAPIKeyMiddleware(keyManager) {
  return {
    // Basic key validation
    authenticate: async (req, res, next) => {
      const rawKey = req.headers['x-api-key'] ||
                     req.query.api_key;  // Support query param (less secure)
      
      if (!rawKey) {
        return res.status(401).json({
          error: 'API key required',
          code: 'API_KEY_REQUIRED'
        });
      }
      
      const keyData = await keyManager.validateKey(rawKey);
      
      if (!keyData) {
        return res.status(401).json({
          error: 'Invalid or expired API key',
          code: 'INVALID_API_KEY'
        });
      }
      
      req.apiKey = keyData;
      next();
    },

    // Scope requirement
    requireScope: (scope) => (req, res, next) => {
      if (!req.apiKey) {
        return res.status(401).json({ error: 'Not authenticated' });
      }
      
      if (!keyManager.hasScope(req.apiKey, scope)) {
        return res.status(403).json({
          error: `Insufficient scope. Required: ${scope}`,
          code: 'INSUFFICIENT_SCOPE'
        });
      }
      
      next();
    }
  };
}

module.exports = { APIKeyManager, createAPIKeyMiddleware };
```

---

## ขั้นตอนที่ 1524: OAuth2 และ OIDC Implementation

```javascript
// oauth2-server.js
// OAuth2 Authorization Server

const express = require('express');
const jwt = require('jsonwebtoken');
const crypto = require('crypto');

const app = express();
app.use(express.json());
app.use(express.urlencoded({ extended: true }));

// OAuth2 Client Registry
const clients = new Map([
  ['client_id_1', {
    clientId: 'client_id_1',
    clientSecret: 'hashed_secret_1',
    redirectUris: ['https://app.example.com/callback'],
    allowedGrantTypes: ['authorization_code', 'refresh_token'],
    allowedScopes: ['read', 'write', 'openid', 'profile', 'email'],
    pkceRequired: true  // Require PKCE for public clients
  }]
]);

// Authorization Code Store
const authCodes = new Map();
const refreshTokens = new Map();

// Authorization Endpoint
app.get('/oauth/authorize', async (req, res) => {
  const {
    response_type,
    client_id,
    redirect_uri,
    scope,
    state,
    code_challenge,        // PKCE
    code_challenge_method  // PKCE
  } = req.query;

  // Validate client
  const client = clients.get(client_id);
  if (!client) {
    return res.status(400).json({ error: 'invalid_client' });
  }

  // Validate redirect URI
  if (!client.redirectUris.includes(redirect_uri)) {
    return res.status(400).json({ error: 'invalid_redirect_uri' });
  }

  // Validate response type
  if (response_type !== 'code') {
    return redirectError(res, redirect_uri, 'unsupported_response_type', state);
  }

  // Validate PKCE (required for public clients)
  if (client.pkceRequired && !code_challenge) {
    return redirectError(res, redirect_uri, 'invalid_request', state, 'PKCE required');
  }

  // Render consent screen
  res.send(`
    <html>
      <body>
        <h1>Authorization Request</h1>
        <p>Application "${client.clientId}" is requesting access to:</p>
        <ul>
          ${scope.split(' ').map(s => `<li>${s}</li>`).join('')}
        </ul>
        <form method="POST" action="/oauth/authorize">
          <input type="hidden" name="client_id" value="${client_id}" />
          <input type="hidden" name="redirect_uri" value="${redirect_uri}" />
          <input type="hidden" name="scope" value="${scope}" />
          <input type="hidden" name="state" value="${state}" />
          <input type="hidden" name="code_challenge" value="${code_challenge || ''}" />
          <input type="hidden" name="code_challenge_method" value="${code_challenge_method || ''}" />
          <button type="submit" name="action" value="allow">Allow</button>
          <button type="submit" name="action" value="deny">Deny</button>
        </form>
      </body>
    </html>
  `);
});

// Handle consent
app.post('/oauth/authorize', (req, res) => {
  const { client_id, redirect_uri, scope, state, code_challenge, code_challenge_method, action } = req.body;

  if (action === 'deny') {
    return redirectError(res, redirect_uri, 'access_denied', state);
  }

  // Generate authorization code
  const code = crypto.randomBytes(32).toString('base64url');
  
  authCodes.set(code, {
    clientId: client_id,
    userId: req.user?.id || 'user_1', // From session
    redirectUri: redirect_uri,
    scope: scope.split(' '),
    codeChallenge: code_challenge,
    codeChallengeMethod: code_challenge_method,
    expiresAt: Date.now() + 600000, // 10 minutes
    used: false
  });

  // Redirect with code
  const redirectUrl = new URL(redirect_uri);
  redirectUrl.searchParams.set('code', code);
  if (state) redirectUrl.searchParams.set('state', state);
  
  res.redirect(redirectUrl.toString());
});

// Token Endpoint
app.post('/oauth/token', async (req, res) => {
  const { grant_type } = req.body;

  switch (grant_type) {
    case 'authorization_code':
      return handleAuthCode(req, res);
    case 'refresh_token':
      return handleRefreshToken(req, res);
    case 'client_credentials':
      return handleClientCredentials(req, res);
    default:
      return res.status(400).json({ error: 'unsupported_grant_type' });
  }
});

async function handleAuthCode(req, res) {
  const { code, redirect_uri, code_verifier, client_id, client_secret } = req.body;

  // Validate client
  const client = clients.get(client_id);
  if (!client || !verifyClientSecret(client.clientSecret, client_secret)) {
    return res.status(401).json({ error: 'invalid_client' });
  }

  // Get and validate auth code
  const authCode = authCodes.get(code);
  if (!authCode || authCode.used || authCode.expiresAt < Date.now()) {
    authCodes.delete(code);
    return res.status(400).json({ error: 'invalid_grant' });
  }

  // Verify PKCE
  if (authCode.codeChallenge) {
    if (!code_verifier) {
      return res.status(400).json({ error: 'invalid_grant', description: 'code_verifier required' });
    }
    
    const verifierHash = crypto
      .createHash('sha256')
      .update(code_verifier)
      .digest('base64url');
    
    if (verifierHash !== authCode.codeChallenge) {
      return res.status(400).json({ error: 'invalid_grant', description: 'PKCE verification failed' });
    }
  }

  // Verify redirect URI
  if (authCode.redirectUri !== redirect_uri) {
    return res.status(400).json({ error: 'invalid_grant' });
  }

  // Mark code as used
  authCode.used = true;
  authCodes.set(code, authCode);

  // Generate tokens
  const accessToken = generateAccessToken(authCode.userId, authCode.scope, client_id);
  const refreshToken = generateRefreshToken(authCode.userId, authCode.scope, client_id);
  
  // Store refresh token
  refreshTokens.set(refreshToken, {
    userId: authCode.userId,
    scope: authCode.scope,
    clientId: client_id,
    expiresAt: Date.now() + 30 * 24 * 60 * 60 * 1000 // 30 days
  });

  // Build response
  const response = {
    access_token: accessToken,
    token_type: 'Bearer',
    expires_in: 3600,
    refresh_token: refreshToken
  };

  // Add ID token if openid scope requested
  if (authCode.scope.includes('openid')) {
    response.id_token = await generateIDToken(authCode.userId, client_id, authCode.scope);
  }

  res.json(response);
}

function generateAccessToken(userId, scopes, clientId) {
  return jwt.sign({
    sub: userId,
    client_id: clientId,
    scope: scopes.join(' '),
    iat: Math.floor(Date.now() / 1000)
  }, process.env.JWT_PRIVATE_KEY, {
    algorithm: 'RS256',
    expiresIn: '1h',
    issuer: process.env.JWT_ISSUER,
    audience: process.env.JWT_AUDIENCE,
    jwtid: crypto.randomUUID()
  });
}

function generateRefreshToken(userId, scopes, clientId) {
  return crypto.randomBytes(64).toString('base64url');
}

async function generateIDToken(userId, clientId, scopes) {
  const user = await getUserById(userId);
  
  const claims = {
    sub: userId,
    iss: process.env.JWT_ISSUER,
    aud: clientId,
    iat: Math.floor(Date.now() / 1000),
    exp: Math.floor(Date.now() / 1000) + 3600
  };

  if (scopes.includes('profile')) {
    claims.name = user.name;
    claims.picture = user.avatarUrl;
    claims.updated_at = user.updatedAt;
  }

  if (scopes.includes('email')) {
    claims.email = user.email;
    claims.email_verified = user.emailVerified;
  }

  return jwt.sign(claims, process.env.JWT_PRIVATE_KEY, { algorithm: 'RS256' });
}

function redirectError(res, redirectUri, error, state, description = null) {
  const url = new URL(redirectUri);
  url.searchParams.set('error', error);
  if (state) url.searchParams.set('state', state);
  if (description) url.searchParams.set('error_description', description);
  return res.redirect(url.toString());
}

function verifyClientSecret(storedHash, provided) {
  if (!provided) return false;
  const hash = crypto.createHash('sha256').update(provided).digest('hex');
  return crypto.timingSafeEqual(Buffer.from(storedHash), Buffer.from(hash));
}

async function getUserById(userId) {
  return { id: userId, name: 'User', email: 'user@example.com', emailVerified: true };
}

// OIDC Discovery endpoint
app.get('/.well-known/openid-configuration', (req, res) => {
  const baseUrl = `${req.protocol}://${req.host}`;
  
  res.json({
    issuer: baseUrl,
    authorization_endpoint: `${baseUrl}/oauth/authorize`,
    token_endpoint: `${baseUrl}/oauth/token`,
    userinfo_endpoint: `${baseUrl}/oauth/userinfo`,
    jwks_uri: `${baseUrl}/.well-known/jwks.json`,
    response_types_supported: ['code'],
    grant_types_supported: ['authorization_code', 'refresh_token'],
    subject_types_supported: ['public'],
    id_token_signing_alg_values_supported: ['RS256'],
    scopes_supported: ['openid', 'profile', 'email'],
    token_endpoint_auth_methods_supported: ['client_secret_basic', 'client_secret_post'],
    claims_supported: ['sub', 'iss', 'aud', 'exp', 'iat', 'name', 'email', 'email_verified'],
    code_challenge_methods_supported: ['S256']
  });
});

module.exports = app;
```

---

## ขั้นตอนที่ 1525: Security Headers และ CORS

```javascript
// security-headers.js
// Comprehensive security headers middleware

const helmet = require('helmet');
const cors = require('cors');

function setupSecurityHeaders(app, options = {}) {
  const { allowedOrigins = [], environment = 'production' } = options;

  // Helmet สำหรับ security headers
  app.use(helmet({
    // Content Security Policy
    contentSecurityPolicy: {
      directives: {
        defaultSrc: ["'self'"],
        scriptSrc: ["'self'", "'strict-dynamic'"],
        styleSrc: ["'self'", "'unsafe-inline'", 'https://fonts.googleapis.com'],
        fontSrc: ["'self'", 'https://fonts.gstatic.com'],
        imgSrc: ["'self'", 'data:', 'https:'],
        connectSrc: ["'self'", 'https://api.example.com'],
        frameSrc: ["'none'"],
        objectSrc: ["'none'"],
        upgradeInsecureRequests: environment === 'production' ? [] : null
      },
      reportOnly: environment !== 'production'
    },
    
    // HTTP Strict Transport Security
    hsts: {
      maxAge: 31536000,        // 1 year
      includeSubDomains: true,
      preload: true
    },
    
    // X-Frame-Options
    frameguard: { action: 'deny' },
    
    // X-Content-Type-Options
    noSniff: true,
    
    // Referrer Policy
    referrerPolicy: { policy: 'strict-origin-when-cross-origin' },
    
    // X-XSS-Protection
    xssFilter: true,
    
    // Hide X-Powered-By
    hidePoweredBy: true,
    
    // Permissions Policy
    permittedCrossDomainPolicies: false
  }));

  // Additional security headers
  app.use((req, res, next) => {
    // Permissions Policy
    res.set('Permissions-Policy', 
      'geolocation=(), camera=(), microphone=(), payment=()');
    
    // Cross-Origin headers
    res.set('Cross-Origin-Opener-Policy', 'same-origin');
    res.set('Cross-Origin-Embedder-Policy', 'require-corp');
    res.set('Cross-Origin-Resource-Policy', 'same-origin');
    
    next();
  });

  // CORS Configuration
  const corsOptions = {
    origin: (origin, callback) => {
      // Allow requests with no origin (mobile apps, curl)
      if (!origin) return callback(null, true);
      
      if (allowedOrigins.includes(origin) || 
          allowedOrigins.some(allowed => 
            allowed instanceof RegExp && allowed.test(origin)
          )) {
        callback(null, true);
      } else {
        callback(new Error(`Origin ${origin} not allowed by CORS`));
      }
    },
    
    methods: ['GET', 'POST', 'PUT', 'PATCH', 'DELETE', 'OPTIONS'],
    
    allowedHeaders: [
      'Content-Type',
      'Authorization', 
      'X-Request-ID',
      'X-API-Key',
      'X-Requested-With'
    ],
    
    exposedHeaders: [
      'X-Request-ID',
      'X-RateLimit-Limit',
      'X-RateLimit-Remaining',
      'X-RateLimit-Reset'
    ],
    
    credentials: true,
    
    maxAge: 86400,  // 24 hours preflight cache
    
    preflightContinue: false,
    optionsSuccessStatus: 204
  };

  app.use(cors(corsOptions));
  app.options('*', cors(corsOptions)); // Handle preflight
}

module.exports = { setupSecurityHeaders };
```

---

## ขั้นตอนที่ 1526-1540: Advanced Security Patterns

### Input Validation และ Sanitization

```javascript
// input-security.js
// Comprehensive input validation

const { body, param, query, validationResult } = require('express-validator');
const DOMPurify = require('isomorphic-dompurify');
const sqlstring = require('sqlstring');

// Validation chains
const userValidators = {
  create: [
    body('email')
      .isEmail()
      .normalizeEmail()
      .withMessage('Valid email required'),
    
    body('password')
      .isLength({ min: 8 })
      .matches(/^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)(?=.*[@$!%*?&])[A-Za-z\d@$!%*?&]/)
      .withMessage('Password must contain uppercase, lowercase, number, and special character'),
    
    body('name')
      .trim()
      .isLength({ min: 2, max: 100 })
      .matches(/^[a-zA-Z\s'-]+$/)
      .withMessage('Name must contain only letters, spaces, hyphens, and apostrophes'),
    
    body('phone')
      .optional()
      .isMobilePhone('any')
      .withMessage('Invalid phone number format')
  ],

  update: [
    param('id').isUUID(4).withMessage('Invalid user ID'),
    
    body('bio')
      .optional()
      .isLength({ max: 500 })
      .customSanitizer(value => {
        // Sanitize HTML to prevent XSS
        return DOMPurify.sanitize(value, { ALLOWED_TAGS: ['b', 'i', 'em', 'strong'] });
      })
  ]
};

// Validation middleware
const validate = (validations) => {
  return async (req, res, next) => {
    await Promise.all(validations.map(validation => validation.run(req)));
    
    const errors = validationResult(req);
    if (!errors.isEmpty()) {
      return res.status(422).json({
        error: 'Validation failed',
        fields: errors.array().reduce((acc, err) => {
          acc[err.path] = err.msg;
          return acc;
        }, {})
      });
    }
    
    next();
  };
};

// SQL Injection Prevention
const SafeQuery = {
  // Always use parameterized queries
  findUser: (db, userId) => {
    // ✅ Safe: parameterized query
    return db.query('SELECT * FROM users WHERE id = $1', [userId]);
  },
  
  // Dynamic queries must also use params
  searchUsers: (db, searchParams) => {
    const allowedColumns = ['email', 'name', 'status'];
    const column = allowedColumns.includes(searchParams.column) 
      ? searchParams.column 
      : 'name';  // Default to safe column
    
    // Whitelist approach for dynamic column names
    return db.query(
      `SELECT * FROM users WHERE ${column} ILIKE $1`,
      [`%${searchParams.value}%`]
    );
  }
};

// Path Traversal Prevention
const safePath = (userInput, basePath) => {
  const path = require('path');
  const resolved = path.resolve(basePath, userInput);
  
  if (!resolved.startsWith(basePath)) {
    throw new Error('Path traversal attempt detected');
  }
  
  return resolved;
};

// File Upload Security
const multer = require('multer');

const secureUpload = multer({
  storage: multer.memoryStorage(),  // Don't save to disk immediately
  limits: {
    fileSize: 10 * 1024 * 1024,    // 10MB max
    files: 5                         // Max 5 files
  },
  fileFilter: (req, file, cb) => {
    const allowedMimeTypes = [
      'image/jpeg', 'image/png', 'image/gif', 'image/webp',
      'application/pdf',
      'text/csv'
    ];
    
    if (!allowedMimeTypes.includes(file.mimetype)) {
      return cb(new Error(`File type ${file.mimetype} not allowed`));
    }
    
    // Check file extension matches mimetype
    const ext = file.originalname.split('.').pop()?.toLowerCase();
    const validExtensions = {
      'image/jpeg': ['jpg', 'jpeg'],
      'image/png': ['png'],
      'image/gif': ['gif'],
      'image/webp': ['webp'],
      'application/pdf': ['pdf'],
      'text/csv': ['csv']
    };
    
    if (!validExtensions[file.mimetype]?.includes(ext)) {
      return cb(new Error('File extension does not match content type'));
    }
    
    cb(null, true);
  }
});

// Verify file content (not just extension/MIME)
async function verifyFileContent(buffer, expectedType) {
  const fileType = await import('file-type');
  const detected = await fileType.fileTypeFromBuffer(buffer);
  
  if (!detected || !expectedType.includes(detected.mime)) {
    throw new Error(`File content does not match expected type`);
  }
  
  return detected;
}

// Express routes with security
const express = require('express');
const app = express();
app.use(express.json({ limit: '10mb' }));

app.post('/users', validate(userValidators.create), async (req, res) => {
  // Input is now validated and sanitized
  const { email, password, name } = req.body;
  
  // Hash password
  const bcrypt = require('bcrypt');
  const hashedPassword = await bcrypt.hash(password, 12);
  
  // Save to database safely
  const result = await db.query(
    'INSERT INTO users (email, password_hash, name) VALUES ($1, $2, $3) RETURNING id',
    [email, hashedPassword, name]
  );
  
  res.status(201).json({ id: result.rows[0].id });
});

app.post('/upload', secureUpload.single('file'), async (req, res) => {
  if (!req.file) {
    return res.status(400).json({ error: 'No file provided' });
  }
  
  try {
    // Verify actual file content
    await verifyFileContent(req.file.buffer, [req.file.mimetype]);
    
    // Process file...
    res.json({ uploaded: true, size: req.file.size });
    
  } catch (err) {
    res.status(400).json({ error: err.message });
  }
});

module.exports = { validate, userValidators, SafeQuery, secureUpload };
```

---

## แบบฝึกหัด

### Exercise 1: Implement OAuth2 PKCE Flow
สร้าง React app ที่ใช้ PKCE flow สำหรับ authentication กับ OAuth2 server ที่สร้างไว้

### Exercise 2: API Key Rotation
สร้าง system ที่ support API key rotation โดยไม่ทำให้ existing clients พัง

### Exercise 3: Security Audit Tool
สร้าง script ที่ตรวจสอบ API endpoints ทั้งหมดว่ามี security middleware ครบถ้วนหรือไม่

### คำถามทบทวน
1. Zero-Trust security model แตกต่างจาก perimeter security อย่างไร?
2. mTLS ป้องกันอะไรที่ TLS ธรรมดาป้องกันไม่ได้?
3. PKCE ใน OAuth2 แก้ปัญหา authorization code interception ได้อย่างไร?
4. ทำไมจึงต้อง hash API keys ก่อนเก็บใน database?

---

*ต่อไป: Part 87 - Performance Engineering*
