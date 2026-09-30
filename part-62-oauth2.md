# Part 62: OAuth2 Authentication
## ขั้นตอนที่ 611-620 จาก 1000

---

## OAuth2 คืออะไร?

OAuth 2.0 เป็น authorization framework ที่ช่วยให้ application สามารถขอ access ไปยัง resources ของ user บน service อื่นได้ โดยไม่ต้องให้ user เปิดเผย credentials (username/password)

---

## 1. OAuth2 Flow

### Authorization Code Flow (แนะนำสุด)

```
Client App → Authorization Server → User (login & consent) 
         ← Authorization Code ←
Client App → Token Endpoint (code + client_secret)
         ← Access Token + Refresh Token ←
Client App → Resource Server (Access Token)
         ← Protected Resource ←
```

```javascript
// ขั้นตอนของ Authorization Code Flow

// 1. Redirect user ไป authorization server
const authUrl = `https://accounts.google.com/o/oauth2/auth?
  client_id=${CLIENT_ID}&
  redirect_uri=${REDIRECT_URI}&
  response_type=code&
  scope=email profile&
  state=${generateState()}`;

res.redirect(authUrl);

// 2. รับ authorization code จาก callback
app.get('/auth/callback', async (req, res) => {
  const { code, state } = req.query;
  
  // Verify state เพื่อป้องกัน CSRF
  if (state !== req.session.oauthState) {
    return res.status(403).json({ error: 'Invalid state' });
  }
  
  // 3. Exchange code สำหรับ tokens
  const tokenResponse = await fetch('https://oauth2.googleapis.com/token', {
    method: 'POST',
    headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
    body: new URLSearchParams({
      code,
      client_id: CLIENT_ID,
      client_secret: CLIENT_SECRET,
      redirect_uri: REDIRECT_URI,
      grant_type: 'authorization_code'
    })
  });
  
  const tokens = await tokenResponse.json();
  // tokens = { access_token, refresh_token, expires_in, token_type }
});
```

### Implicit Flow (เก่า ไม่แนะนำ)

```javascript
// ไม่แนะนำ - ใช้ Authorization Code with PKCE แทน
const authUrl = `https://provider.com/auth?
  response_type=token&  // ได้ token โดยตรง ไม่ปลอดภัย
  client_id=${CLIENT_ID}`;
```

### Client Credentials Flow (สำหรับ server-to-server)

```javascript
// ใช้สำหรับ machine-to-machine authentication
const tokenResponse = await fetch('https://auth.example.com/token', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/x-www-form-urlencoded',
    'Authorization': `Basic ${Buffer.from(`${CLIENT_ID}:${CLIENT_SECRET}`).toString('base64')}`
  },
  body: new URLSearchParams({
    grant_type: 'client_credentials',
    scope: 'api:read api:write'
  })
});

const { access_token } = await tokenResponse.json();
```

### PKCE (Proof Key for Code Exchange)

```javascript
// สำหรับ public clients (mobile apps, SPAs)
const crypto = require('crypto');

function generateCodeVerifier() {
  return crypto.randomBytes(32).toString('base64url');
}

function generateCodeChallenge(verifier) {
  return crypto.createHash('sha256')
    .update(verifier)
    .digest('base64url');
}

const codeVerifier = generateCodeVerifier();
const codeChallenge = generateCodeChallenge(codeVerifier);

// เก็บ codeVerifier ไว้ใน session
req.session.codeVerifier = codeVerifier;

const authUrl = `https://provider.com/auth?
  client_id=${CLIENT_ID}&
  redirect_uri=${REDIRECT_URI}&
  response_type=code&
  code_challenge=${codeChallenge}&
  code_challenge_method=S256`;

// เมื่อ exchange code
const tokenResponse = await fetch('/token', {
  method: 'POST',
  body: new URLSearchParams({
    code,
    client_id: CLIENT_ID,
    redirect_uri: REDIRECT_URI,
    grant_type: 'authorization_code',
    code_verifier: req.session.codeVerifier  // ส่ง verifier ไปด้วย
  })
});
```

---

## 2. Passport.js

### ติดตั้งและตั้งค่าพื้นฐาน

```bash
npm install passport passport-local passport-jwt
npm install passport-google-oauth20 passport-github2
npm install express-session
```

```javascript
// config/passport.js
const passport = require('passport');
const { Strategy: LocalStrategy } = require('passport-local');
const { Strategy: JwtStrategy, ExtractJwt } = require('passport-jwt');
const User = require('../models/User');
const bcrypt = require('bcrypt');

// Local Strategy
passport.use('local', new LocalStrategy(
  {
    usernameField: 'email',
    passwordField: 'password'
  },
  async (email, password, done) => {
    try {
      const user = await User.findOne({ email: email.toLowerCase() });
      
      if (!user) {
        return done(null, false, { message: 'Email not found' });
      }

      const isValid = await bcrypt.compare(password, user.password);
      
      if (!isValid) {
        return done(null, false, { message: 'Wrong password' });
      }

      return done(null, user);
    } catch (error) {
      return done(error);
    }
  }
));

// JWT Strategy
passport.use('jwt', new JwtStrategy(
  {
    jwtFromRequest: ExtractJwt.fromAuthHeaderAsBearerToken(),
    secretOrKey: process.env.JWT_SECRET,
    issuer: 'myapp.com',
    audience: 'myapp.com'
  },
  async (jwtPayload, done) => {
    try {
      const user = await User.findById(jwtPayload.sub);
      
      if (!user) {
        return done(null, false);
      }

      // ตรวจสอบว่า token ถูก revoke หรือไม่
      if (user.tokenVersion !== jwtPayload.tokenVersion) {
        return done(null, false);
      }

      return done(null, user);
    } catch (error) {
      return done(error);
    }
  }
));

// Serialize / Deserialize สำหรับ session
passport.serializeUser((user, done) => {
  done(null, user.id);
});

passport.deserializeUser(async (id, done) => {
  try {
    const user = await User.findById(id);
    done(null, user);
  } catch (error) {
    done(error);
  }
});

module.exports = passport;
```

```javascript
// app.js
const express = require('express');
const session = require('express-session');
const passport = require('./config/passport');

const app = express();

app.use(express.json());
app.use(session({
  secret: process.env.SESSION_SECRET,
  resave: false,
  saveUninitialized: false,
  cookie: {
    secure: process.env.NODE_ENV === 'production',
    httpOnly: true,
    maxAge: 24 * 60 * 60 * 1000  // 1 day
  }
}));

app.use(passport.initialize());
app.use(passport.session());
```

### Auth Routes

```javascript
// routes/auth.js
const router = require('express').Router();
const passport = require('../config/passport');
const jwt = require('jsonwebtoken');
const bcrypt = require('bcrypt');
const User = require('../models/User');

// Register
router.post('/register', async (req, res) => {
  const { name, email, password } = req.body;
  
  const existingUser = await User.findOne({ email });
  if (existingUser) {
    return res.status(400).json({ error: 'Email already in use' });
  }
  
  const hashedPassword = await bcrypt.hash(password, 12);
  const user = await User.create({
    name,
    email: email.toLowerCase(),
    password: hashedPassword
  });
  
  const token = generateToken(user);
  res.status(201).json({ user: sanitizeUser(user), token });
});

// Login with local strategy
router.post('/login', (req, res, next) => {
  passport.authenticate('local', { session: false }, (err, user, info) => {
    if (err) return next(err);
    if (!user) return res.status(401).json({ error: info.message });
    
    const token = generateToken(user);
    res.json({ user: sanitizeUser(user), token });
  })(req, res, next);
});

// Protected route
router.get('/me', 
  passport.authenticate('jwt', { session: false }),
  (req, res) => {
    res.json({ user: sanitizeUser(req.user) });
  }
);

// Logout (invalidate token)
router.post('/logout', 
  passport.authenticate('jwt', { session: false }),
  async (req, res) => {
    // Increment tokenVersion ทำให้ tokens เก่าทั้งหมด invalid
    await User.findByIdAndUpdate(req.user.id, { $inc: { tokenVersion: 1 } });
    res.json({ message: 'Logged out successfully' });
  }
);

function generateToken(user) {
  return jwt.sign(
    {
      sub: user.id,
      email: user.email,
      tokenVersion: user.tokenVersion || 0
    },
    process.env.JWT_SECRET,
    {
      expiresIn: '15m',
      issuer: 'myapp.com',
      audience: 'myapp.com'
    }
  );
}

function sanitizeUser(user) {
  const { password, tokenVersion, __v, ...safe } = user.toObject();
  return safe;
}

module.exports = router;
```

---

## 3. Google OAuth

### Setup Google OAuth

```bash
# ไปที่ https://console.cloud.google.com/
# สร้าง Project → Enable Google+ API / Google OAuth2
# สร้าง OAuth2 Credentials
# กำหนด Authorized redirect URIs: http://localhost:3000/auth/google/callback
```

```javascript
// config/passport.js - เพิ่ม Google Strategy
const { Strategy: GoogleStrategy } = require('passport-google-oauth20');

passport.use('google', new GoogleStrategy(
  {
    clientID: process.env.GOOGLE_CLIENT_ID,
    clientSecret: process.env.GOOGLE_CLIENT_SECRET,
    callbackURL: '/auth/google/callback',
    scope: ['profile', 'email']
  },
  async (accessToken, refreshToken, profile, done) => {
    try {
      // หา user ที่มี googleId นี้
      let user = await User.findOne({ 'google.id': profile.id });
      
      if (user) {
        // อัพเดต access token
        user.google.accessToken = accessToken;
        await user.save();
        return done(null, user);
      }

      // ตรวจสอบ email ซ้ำ
      const email = profile.emails[0]?.value;
      user = await User.findOne({ email });
      
      if (user) {
        // เชื่อม Google account กับ existing user
        user.google = {
          id: profile.id,
          accessToken,
          refreshToken
        };
        await user.save();
        return done(null, user);
      }

      // สร้าง user ใหม่
      user = await User.create({
        name: profile.displayName,
        email,
        avatar: profile.photos[0]?.value,
        google: {
          id: profile.id,
          accessToken,
          refreshToken
        }
      });

      return done(null, user);
    } catch (error) {
      return done(error);
    }
  }
));
```

```javascript
// routes/auth.js - Google OAuth routes
const passport = require('../config/passport');
const jwt = require('jsonwebtoken');

// เริ่ม Google OAuth flow
router.get('/google',
  passport.authenticate('google', {
    scope: ['profile', 'email'],
    accessType: 'offline',  // รับ refresh_token
    prompt: 'consent'       // แสดง consent screen ทุกครั้ง (สำหรับ refresh_token)
  })
);

// Google OAuth callback
router.get('/google/callback',
  passport.authenticate('google', {
    session: false,
    failureRedirect: '/login?error=google_failed'
  }),
  (req, res) => {
    const token = jwt.sign(
      { sub: req.user.id, email: req.user.email },
      process.env.JWT_SECRET,
      { expiresIn: '7d' }
    );
    
    // Redirect ไปยัง frontend พร้อม token
    res.redirect(`${process.env.FRONTEND_URL}/auth/callback?token=${token}`);
  }
);
```

### Google API Calls (ด้วย Access Token)

```javascript
// services/google.service.js
const { google } = require('googleapis');

async function getUserProfile(accessToken) {
  const oauth2 = google.oauth2({ version: 'v2' });
  const auth = new google.auth.OAuth2();
  auth.setCredentials({ access_token: accessToken });
  
  const { data } = await oauth2.userinfo.get({ auth });
  return data;
}

async function getUserCalendar(accessToken) {
  const calendar = google.calendar({ version: 'v3' });
  const auth = new google.auth.OAuth2();
  auth.setCredentials({ access_token: accessToken });
  
  const { data } = await calendar.events.list({
    auth,
    calendarId: 'primary',
    timeMin: new Date().toISOString(),
    maxResults: 10,
    singleEvents: true,
    orderBy: 'startTime'
  });
  
  return data.items;
}

async function refreshAccessToken(refreshToken) {
  const oauth2Client = new google.auth.OAuth2(
    process.env.GOOGLE_CLIENT_ID,
    process.env.GOOGLE_CLIENT_SECRET,
    process.env.GOOGLE_REDIRECT_URI
  );
  
  oauth2Client.setCredentials({ refresh_token: refreshToken });
  const { credentials } = await oauth2Client.refreshAccessToken();
  
  return credentials;
}

module.exports = { getUserProfile, getUserCalendar, refreshAccessToken };
```

---

## 4. GitHub OAuth

### Setup GitHub OAuth

```bash
# ไปที่ GitHub → Settings → Developer settings → OAuth Apps
# สร้าง New OAuth App
# กำหนด Authorization callback URL: http://localhost:3000/auth/github/callback
```

```javascript
// config/passport.js - เพิ่ม GitHub Strategy
const { Strategy: GitHubStrategy } = require('passport-github2');

passport.use('github', new GitHubStrategy(
  {
    clientID: process.env.GITHUB_CLIENT_ID,
    clientSecret: process.env.GITHUB_CLIENT_SECRET,
    callbackURL: '/auth/github/callback',
    scope: ['user:email', 'read:user']
  },
  async (accessToken, refreshToken, profile, done) => {
    try {
      let user = await User.findOne({ 'github.id': profile.id });
      
      if (user) {
        user.github.accessToken = accessToken;
        await user.save();
        return done(null, user);
      }

      const email = profile.emails?.[0]?.value;
      
      user = await User.create({
        name: profile.displayName || profile.username,
        email,
        username: profile.username,
        avatar: profile._json.avatar_url,
        github: {
          id: profile.id,
          username: profile.username,
          accessToken
        }
      });

      return done(null, user);
    } catch (error) {
      return done(error);
    }
  }
));
```

```javascript
// GitHub OAuth routes
router.get('/github',
  passport.authenticate('github', {
    scope: ['user:email', 'repo']
  })
);

router.get('/github/callback',
  passport.authenticate('github', {
    session: false,
    failureRedirect: '/login?error=github_failed'
  }),
  (req, res) => {
    const token = generateToken(req.user);
    res.redirect(`${process.env.FRONTEND_URL}/auth/callback?token=${token}`);
  }
);
```

### GitHub API Calls

```javascript
// services/github.service.js
const { Octokit } = require('@octokit/rest');

async function getUserRepos(accessToken) {
  const octokit = new Octokit({ auth: accessToken });
  
  const { data } = await octokit.repos.listForAuthenticatedUser({
    sort: 'updated',
    per_page: 10
  });
  
  return data;
}

async function createRepository(accessToken, name, description, isPrivate = false) {
  const octokit = new Octokit({ auth: accessToken });
  
  const { data } = await octokit.repos.createForAuthenticatedUser({
    name,
    description,
    private: isPrivate,
    auto_init: true
  });
  
  return data;
}

module.exports = { getUserRepos, createRepository };
```

---

## 5. Token Management

### Refresh Token Strategy

```javascript
// services/token.service.js
const jwt = require('jsonwebtoken');
const crypto = require('crypto');
const RefreshToken = require('../models/RefreshToken');

const ACCESS_TOKEN_EXPIRY = '15m';
const REFRESH_TOKEN_EXPIRY = '7d';

function generateAccessToken(user) {
  return jwt.sign(
    {
      sub: user.id,
      email: user.email,
      role: user.role,
      tokenVersion: user.tokenVersion
    },
    process.env.JWT_SECRET,
    { expiresIn: ACCESS_TOKEN_EXPIRY }
  );
}

async function generateRefreshToken(userId) {
  const token = crypto.randomBytes(40).toString('hex');
  const expiresAt = new Date();
  expiresAt.setDate(expiresAt.getDate() + 7);
  
  await RefreshToken.create({
    token,
    userId,
    expiresAt
  });
  
  return token;
}

async function refreshTokens(refreshToken) {
  const stored = await RefreshToken.findOne({
    token: refreshToken,
    used: false,
    expiresAt: { $gt: new Date() }
  }).populate('userId');
  
  if (!stored) {
    throw new Error('Invalid or expired refresh token');
  }
  
  // Rotate refresh token (เพิ่มความปลอดภัย)
  stored.used = true;
  await stored.save();
  
  const user = stored.userId;
  const newAccessToken = generateAccessToken(user);
  const newRefreshToken = await generateRefreshToken(user.id);
  
  return { accessToken: newAccessToken, refreshToken: newRefreshToken };
}

async function revokeRefreshToken(token) {
  await RefreshToken.updateOne({ token }, { revoked: true, revokedAt: new Date() });
}

async function revokeAllUserTokens(userId) {
  await RefreshToken.updateMany(
    { userId, used: false, revoked: false },
    { revoked: true, revokedAt: new Date() }
  );
}

module.exports = {
  generateAccessToken,
  generateRefreshToken,
  refreshTokens,
  revokeRefreshToken,
  revokeAllUserTokens
};
```

### RefreshToken Model

```javascript
// models/RefreshToken.js
const mongoose = require('mongoose');

const refreshTokenSchema = new mongoose.Schema({
  token: {
    type: String,
    required: true,
    unique: true,
    index: true
  },
  userId: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User',
    required: true
  },
  expiresAt: {
    type: Date,
    required: true
  },
  used: {
    type: Boolean,
    default: false
  },
  revoked: {
    type: Boolean,
    default: false
  },
  revokedAt: Date,
  userAgent: String,
  ipAddress: String
}, {
  timestamps: true
});

// Auto-delete expired tokens
refreshTokenSchema.index({ expiresAt: 1 }, { expireAfterSeconds: 0 });

module.exports = mongoose.model('RefreshToken', refreshTokenSchema);
```

### Token Refresh Endpoint

```javascript
// routes/auth.js
const tokenService = require('../services/token.service');

// Refresh access token
router.post('/refresh', async (req, res) => {
  const { refreshToken } = req.body;
  
  if (!refreshToken) {
    return res.status(400).json({ error: 'Refresh token required' });
  }
  
  try {
    const tokens = await tokenService.refreshTokens(refreshToken);
    res.json(tokens);
  } catch (error) {
    res.status(401).json({ error: error.message });
  }
});

// Revoke refresh token (logout from single device)
router.post('/revoke', async (req, res) => {
  const { refreshToken } = req.body;
  
  await tokenService.revokeRefreshToken(refreshToken);
  res.json({ message: 'Token revoked' });
});

// Logout from all devices
router.post('/logout-all',
  passport.authenticate('jwt', { session: false }),
  async (req, res) => {
    await tokenService.revokeAllUserTokens(req.user.id);
    await User.findByIdAndUpdate(req.user.id, { $inc: { tokenVersion: 1 } });
    res.json({ message: 'Logged out from all devices' });
  }
);
```

### Axios Interceptor สำหรับ Token Refresh (Frontend)

```javascript
// api/axios-client.js
import axios from 'axios';

const api = axios.create({
  baseURL: process.env.REACT_APP_API_URL
});

// ส่ง access token กับทุก request
api.interceptors.request.use((config) => {
  const token = localStorage.getItem('accessToken');
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});

// Handle token refresh
let isRefreshing = false;
let failedQueue = [];

api.interceptors.response.use(
  (response) => response,
  async (error) => {
    const originalRequest = error.config;
    
    if (error.response?.status === 401 && !originalRequest._retry) {
      if (isRefreshing) {
        // Queue requests ขณะ refresh
        return new Promise((resolve, reject) => {
          failedQueue.push({ resolve, reject });
        }).then(token => {
          originalRequest.headers.Authorization = `Bearer ${token}`;
          return api(originalRequest);
        });
      }

      originalRequest._retry = true;
      isRefreshing = true;

      try {
        const refreshToken = localStorage.getItem('refreshToken');
        const { data } = await axios.post('/auth/refresh', { refreshToken });
        
        localStorage.setItem('accessToken', data.accessToken);
        localStorage.setItem('refreshToken', data.refreshToken);
        
        // Retry queued requests
        failedQueue.forEach(req => req.resolve(data.accessToken));
        failedQueue = [];
        
        originalRequest.headers.Authorization = `Bearer ${data.accessToken}`;
        return api(originalRequest);
      } catch (refreshError) {
        failedQueue.forEach(req => req.reject(refreshError));
        failedQueue = [];
        
        // Redirect to login
        localStorage.clear();
        window.location.href = '/login';
        return Promise.reject(refreshError);
      } finally {
        isRefreshing = false;
      }
    }
    
    return Promise.reject(error);
  }
);

export default api;
```

---

## 6. Security Best Practices

```javascript
// security considerations

// 1. ใช้ state parameter ป้องกัน CSRF
function generateState() {
  return crypto.randomBytes(16).toString('hex');
}

router.get('/google', (req, res, next) => {
  const state = generateState();
  req.session.oauthState = state;
  
  passport.authenticate('google', {
    state,
    scope: ['profile', 'email']
  })(req, res, next);
});

router.get('/google/callback', (req, res, next) => {
  if (req.query.state !== req.session.oauthState) {
    return res.status(403).json({ error: 'State mismatch - possible CSRF' });
  }
  
  passport.authenticate('google', { session: false })(req, res, next);
});

// 2. ตั้งค่า Cookie อย่างปลอดภัย
app.use(session({
  secret: process.env.SESSION_SECRET,
  cookie: {
    secure: true,    // HTTPS only
    httpOnly: true,  // ป้องกัน XSS
    sameSite: 'lax', // ป้องกัน CSRF
    maxAge: 10 * 60 * 1000  // 10 minutes สำหรับ OAuth flow
  }
}));

// 3. Rate limiting
const rateLimit = require('express-rate-limit');

const authLimiter = rateLimit({
  windowMs: 15 * 60 * 1000,
  max: 5,
  message: { error: 'Too many login attempts, please try again after 15 minutes' }
});

router.post('/login', authLimiter, loginHandler);
```

---

## แบบฝึกหัด

### ระดับ 1: พื้นฐาน
สร้าง authentication system ที่:
- มี Register/Login endpoints
- ใช้ JWT สำหรับ access
- ใช้ Passport local strategy

### ระดับ 2: กลาง
เพิ่ม OAuth providers:
- Google OAuth
- GitHub OAuth
- Refresh token rotation
- Logout from all devices

### ระดับ 3: ขั้นสูง
สร้าง complete auth system:
- Multiple OAuth providers
- Merge accounts (same email จาก providers ต่างกัน)
- Session management (active sessions list)
- Security audit logs

```javascript
// โค้ด starter
const express = require('express');
const passport = require('passport');
const app = express();

// TODO: Setup passport strategies
// TODO: Create auth routes
// TODO: Implement token management
// TODO: Add security middleware

app.listen(3000);
```

---

## สรุป

OAuth2 เป็น standard สำคัญสำหรับ modern authentication ที่ช่วยให้ users สามารถ login ด้วย social accounts ได้อย่างปลอดภัย Passport.js ทำให้การ implement OAuth2 ง่ายขึ้นมาก ควรจำ:
- ใช้ state parameter ป้องกัน CSRF
- Rotate refresh tokens
- เก็บ access token ใน memory ไม่ใช่ localStorage
- ใช้ PKCE สำหรับ public clients

> ขั้นตอนต่อไป: Part 63 - Multi-tenancy
