# Bluestock Backend - Architecture & Implementation Layers

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    CLIENT APPLICATIONS                           │
│         (Admin Dashboard, B2B Portal, Demo App)                 │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│                      CORS & Security Layer                       │
│  (Helmet, CORS whitelist: localhost:5173/5174/5175)            │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│                   Express.js Server                              │
│                   (Port 3000 / Vercel)                           │
└────────────────────────┬────────────────────────────────────────┘
                         │
        ┌────────────────┼────────────────┐
        │                │                │
        ▼                ▼                ▼
    ┌─────────┐    ┌──────────┐    ┌──────────────┐
    │ Routes  │    │Middleware│    │ Error Handler│
    │ (8 sets)│    │ (4 types)│    │              │
    └─────────┘    └──────────┘    └──────────────┘
        │
        ├─► Logger Middleware ──► Database (Usage logs)
        │
        ├─► Auth Middleware ──► JWT validation / API Key validation
        │
        ├─► Rate Limit Middleware ──► Redis / In-Memory Cache
        │
        └─► API Route Handlers
                    │
        ┌───────────┼───────────┬──────────────┬──────────────┐
        │           │           │              │              │
        ▼           ▼           ▼              ▼              ▼
    ┌────────┐ ┌────────┐ ┌─────────┐ ┌──────────┐ ┌──────────────┐
    │Prisma │ │ Redis  │ │ Email   │ │Validators│ │ Utilities    │
    │ ORM   │ │(Cache) │ │(SMTP)   │ │(Express) │ │(Error, Env)  │
    └────┬───┘ └────────┘ └─────────┘ └──────────┘ └──────────────┘
         │
         ▼
    ┌─────────────────────┐
    │  PostgreSQL DB      │
    │  (Vercel / Self)    │
    │                     │
    │  - Countries        │
    │  - States           │
    │  - Districts        │
    │  - SubDistricts     │
    │  - Villages         │
    │  - Users            │
    │  - ApiKeys          │
    │  - Usage            │
    └─────────────────────┘
```

---

## Request Flow Diagram

### Authenticated Request (JWT Bearer Token)

```
┌──────────────────┐
│   Client Request │
│ + JWT Bearer     │
└────────┬─────────┘
         │
         ▼
┌──────────────────────────────┐
│ CORS Check                   │
│ (Allowed origins only)       │
└────────┬─────────────────────┘
         │
         ▼
┌──────────────────────────────┐
│ Body Parser (JSON)           │
└────────┬─────────────────────┘
         │
         ▼
┌──────────────────────────────┐
│ Request Logger               │
│ (Queues log to Usage table)  │
└────────┬─────────────────────┘
         │
         ▼
┌──────────────────────────────┐
│ Route-specific Auth          │
│ (requireAuth middleware)     │
│ - Extract JWT token          │
│ - Verify signature           │
│ - Fetch user from DB         │
│ - Populate req.user          │
└────────┬─────────────────────┘
         │
         ├─ Auth Failed? ──► 401 Error
         │
         ▼
┌──────────────────────────────┐
│ Admin Check (if needed)      │
│ (requireAdmin middleware)    │
│ - Check req.user.role        │
└────────┬─────────────────────┘
         │
         ├─ Not Admin? ──► 403 Error
         │
         ▼
┌──────────────────────────────┐
│ 2FA Check (if needed)        │
│ (requireAdmin2FA middleware) │
│ - Verify TOTP token          │
└────────┬─────────────────────┘
         │
         ├─ Invalid 2FA? ──► 401 Error
         │
         ▼
┌──────────────────────────────┐
│ Route Handler                │
│ - Input validation           │
│ - Database operations        │
│ - Business logic             │
└────────┬─────────────────────┘
         │
         ├─ Error? ──► AppError thrown
         │
         ▼
┌──────────────────────────────┐
│ Async Logger (res.on finish) │
│ - Insert usage record        │
│ - Flush to DB                │
└──────────────────────────────┘
         │
         ▼
┌──────────────────────────────┐
│ Response + Headers           │
│ Success: 200/201             │
│ Error: 400/401/403/404/500   │
└──────────────────────────────┘
```

### API Key Request (Data Lookup)

```
┌──────────────────────────────┐
│   Client Request             │
│ + X-API-Key: ak_[32-hex]    │
└────────┬─────────────────────┘
         │
         ▼
┌──────────────────────────────┐
│ CORS Check                   │
└────────┬─────────────────────┘
         │
         ▼
┌──────────────────────────────┐
│ Body Parser                  │
└────────┬─────────────────────┘
         │
         ▼
┌──────────────────────────────┐
│ Request Logger               │
│ (Queues log to Usage table)  │
└────────┬─────────────────────┘
         │
         ▼
┌──────────────────────────────┐
│ API Key Middleware           │
│ (requireApiKey)              │
│ - Validate format            │
│ - Hash (SHA256)              │
│ - Lookup fingerprint         │
│ - Compare bcrypt secret      │
│ - Check status ACTIVE        │
│ - Populate req.user          │
│ - Set req.demoMode if demo   │
└────────┬─────────────────────┘
         │
         ├─ Key invalid? ──► 401 Error
         │
         ▼
┌──────────────────────────────┐
│ Rate Limit Middleware        │
│ - Get user plan/limit        │
│ - Redis key: ratelimit:{key}:{date}
│ - INCR counter               │
│ - Check limit exceeded       │
│ - Set response headers       │
│ - Check for alert threshold  │
│ - Send alert email if needed │
└────────┬─────────────────────┘
         │
         ├─ Limit exceeded? ──► 403 Error
         │
         ▼
┌──────────────────────────────┐
│ Route Handler                │
│ - Validate query params      │
│ - Check demo mode filters    │
│ - Query database (with cache)│
│ - Format response            │
└────────┬─────────────────────┘
         │
         ▼
┌──────────────────────────────┐
│ Async Logger (res.on finish) │
│ - Insert usage record        │
│ - Flush to DB                │
└──────────────────────────────┘
         │
         ▼
┌──────────────────────────────┐
│ Response + Headers           │
│ X-RateLimit-* headers set    │
│ Success: 200                 │
│ Error: 400/401/403/404       │
└──────────────────────────────┘
```

---

## Middleware Layer Stack

```
┌─────────────────────────────────────────────────────────┐
│ 1. CORS Middleware (cors)                               │
│    - Whitelist: localhost:5173, 5174, 5175             │
│    - Methods: GET, POST, PUT, DELETE                   │
│    - Headers: Content-Type, X-API-Key, Authorization  │
└─────────────────────────────────────────────────────────┘
         ▼
┌─────────────────────────────────────────────────────────┐
│ 2. Body Parser (express.json)                           │
│    - Parses JSON request bodies                         │
└─────────────────────────────────────────────────────────┘
         ▼
┌─────────────────────────────────────────────────────────┐
│ 3. Request Logger (requestLogger)                       │
│    - Fires on response.finish                          │
│    - Async logs to Usage table                         │
│    - Skips unauthenticated requests                    │
└─────────────────────────────────────────────────────────┘
         ▼
┌─────────────────────────────────────────────────────────┐
│ 4. Route-Specific Middleware (conditionally applied)    │
│                                                         │
│    a) requireApiKey (on protected data routes)         │
│       - Validates X-API-Key header format              │
│       - Checks fingerprint in DB                       │
│       - Verifies bcrypt secret                         │
│       - Populates req.user, req.apiKey                 │
│                                                         │
│    b) requireAuth (on admin/usage routes)              │
│       - Extracts Bearer token                          │
│       - Verifies JWT signature                         │
│       - Fetches user from DB                           │
│       - Populates req.user                             │
│                                                         │
│    c) rateLimit (on protected data routes)             │
│       - Reads user plan/dailyLimit                     │
│       - Increments Redis counter                       │
│       - Checks quota exceeded                          │
│       - Sets X-RateLimit headers                       │
│       - Sends alert email at 80%, 95%                  │
│                                                         │
│    d) requireAdmin (on admin routes)                   │
│       - Checks req.user.role in [ADMIN, SUPERADMIN]   │
│                                                         │
│    e) requireAdmin2FA (on sensitive admin routes)      │
│       - If twoFactorEnabled: Validates TOTP token      │
└─────────────────────────────────────────────────────────┘
         ▼
┌─────────────────────────────────────────────────────────┐
│ 5. Route Handlers                                       │
│    - Validation logic                                  │
│    - Business logic                                    │
│    - Database queries                                  │
│    - Response formatting                              │
└─────────────────────────────────────────────────────────┘
         ▼
┌─────────────────────────────────────────────────────────┐
│ 6. Global Error Handler (last middleware)              │
│    - Catches all errors thrown by routes               │
│    - Differentiates: Operational vs Unknown errors     │
│    - Logs errors to console                            │
│    - Returns JSON error response                       │
└─────────────────────────────────────────────────────────┘
```

---

## Route Organization

```
├── Public Routes (no auth required)
│   ├── GET  /                          # Health check
│   ├── GET  /api-docs                  # Swagger UI
│   └── GET  /openapi.json              # OpenAPI spec
│
├── Auth Routes (/api/v1/auth) - Mixed Protection
│   ├── POST   /register                # Public
│   ├── POST   /verify-email            # Public
│   ├── POST   /resend-verification     # Public
│   ├── POST   /login                   # Public
│   ├── GET    /me                      # Requires JWT ✓
│   ├── POST   /2fa/setup               # Requires JWT ✓
│   ├── POST   /2fa/verify              # Requires JWT ✓
│   ├── POST   /2fa/disable             # Requires JWT ✓
│   ├── GET    /apikeys                 # Requires JWT ✓
│   ├── POST   /apikeys                 # Requires JWT ✓
│   ├── PATCH  /apikeys/:id/revoke      # Requires JWT ✓
│   └── PATCH  /apikeys/:id/rotate      # Requires JWT ✓
│
├── Admin Routes (/api/v1/admin) - Full Protection
│   ├── GET    /overview                # JWT + Admin + 2FA ✓✓✓
│   ├── GET    /users                   # JWT + Admin + 2FA ✓✓✓
│   ├── GET    /logs                    # JWT + Admin + 2FA ✓✓✓
│   ├── GET    /states                  # JWT + Admin + 2FA ✓✓✓
│   ├── PATCH  /users/:id/approve       # JWT + Admin + 2FA ✓✓✓
│   ├── PATCH  /users/:id/reject        # JWT + Admin + 2FA ✓✓✓
│   ├── PATCH  /users/:id/access        # JWT + Admin + 2FA ✓✓✓
│   ├── PATCH  /users/:id/plan          # JWT + Admin + 2FA ✓✓✓
│   └── PATCH  /users/:id/limit         # JWT + Admin + 2FA ✓✓✓
│
├── Usage Routes (/api/v1/usage) - JWT Protected
│   ├── GET    /dashboard               # Requires JWT ✓
│   └── GET    /logs                    # Requires JWT ✓
│
└── Data Lookup Routes (/api/v1/*) - API Key + Rate Limit
    ├── GET    /states                  # API Key + RateLimit ✓✓
    ├── GET    /states/:id              # API Key + RateLimit ✓✓
    ├── GET    /districts               # API Key + RateLimit ✓✓
    ├── GET    /districts/:id           # API Key + RateLimit ✓✓
    ├── GET    /subdistricts            # API Key + RateLimit ✓✓
    ├── GET    /subdistricts/:id        # API Key + RateLimit ✓✓
    ├── GET    /villages                # API Key + RateLimit ✓✓
    ├── GET    /villages/:id            # API Key + RateLimit ✓✓
    ├── GET    /search                  # API Key + RateLimit ✓✓
    └── GET    /search/autocomplete     # API Key + RateLimit ✓✓
```

---

## Database Layer

```
┌──────────────────────────────────────────────────────────┐
│ Prisma Client (Singleton)                                │
│ - Cached globally to prevent connection leaks           │
│ - Logging: errors & warnings only                       │
└──────────────┬───────────────────────────────────────────┘
               │
               ▼
┌──────────────────────────────────────────────────────────┐
│ PostgreSQL Database                                      │
│ (Can be local, RDS, or Vercel Postgres)                 │
└──────────────┬───────────────────────────────────────────┘
               │
       ┌───────┼───────┬──────────┬───────────────┐
       │       │       │          │               │
       ▼       ▼       ▼          ▼               ▼
    ┌────┐ ┌────┐ ┌──────┐ ┌─────────┐ ┌──────────────┐
    │User│ │Auth│ │Geo   │ │Analytics│ │Configuration│
    └────┘ └────┘ │      │ └─────────┘ │              │
                  │      │              │              │
        Columns:  │      │   Columns:   │ Columns:     │
        - id      │      │   - id       │ - NODE_ENV   │
        - email   │      │   - userId   │ - PORT       │
        - plan    │      │   - endpoint │ - JWT_SECRET │
        - role    │      │   - status   │ - DB_URL     │
        - status  │      │   - date     │ - REDIS_URL  │
        - limits  └──┬───┴─────────────┘ └──────────────┘
        - 2FA        │
        - apiKeys    │
        - verified   │
        - stateAccess│
                     │
          Relationships:
          ┌──────────────────────────────┐
          │ Country                      │
          │ - code, name                 │
          └──────────────┬───────────────┘
                         │ has many
          ┌──────────────▼───────────────┐
          │ State                        │
          │ - code, name, countryId      │
          └──────────────┬───────────────┘
                         │ has many
          ┌──────────────▼───────────────┐
          │ District                     │
          │ - code, name, stateId        │
          └──────────────┬───────────────┘
                         │ has many
          ┌──────────────▼───────────────┐
          │ SubDistrict                  │
          │ - code, name, districtId     │
          └──────────────┬───────────────┘
                         │ has many
          ┌──────────────▼───────────────┐
          │ Village                      │
          │ - code, name, subDistrictId  │
          └──────────────────────────────┘
```

---

## Caching Strategy

```
┌─────────────────────────────────┐
│ Client Request                  │
└────────────┬────────────────────┘
             │
             ▼
┌─────────────────────────────────┐
│ Check Cache for /states         │
│ Key: states:all                 │
└────────┬────────────────────────┘
         │
    ┌────┴─────┐
    │ YES       │ NO
    ▼           ▼
┌────────┐  ┌─────────────────────────────┐
│Cache   │  │ Query Database              │
│HIT     │  │ (prisma.state.findMany)     │
│Return  │  └────────┬────────────────────┘
│cached  │           │
│data    │           ▼
└────────┘  ┌─────────────────────────────┐
            │ Store in Cache              │
            │ Key: states:all             │
            │ TTL: 86400 seconds (24h)    │
            │ Storage: Redis or In-Memory │
            └────────┬────────────────────┘
                     │
                     ▼
            ┌─────────────────────────────┐
            │ Return to Client            │
            │ + Cache HIT/MISS log        │
            └─────────────────────────────┘

Rate Limiting Cache:
Key: ratelimit:{apiKey}:{YYYY-MM-DD}
TTL: 86400 seconds (resets daily)
Storage: Redis or In-Memory
Value: Request counter (incremented per request)

Alert Cache:
Key: usage-alert:{userId}:{date}
TTL: 86400 seconds
Storage: Redis only
Value: Last alert threshold (0.8 or 0.95)
```

---

## Security Layers

```
┌─────────────────────────────────────────────────────┐
│ Layer 1: Transport Security                         │
│ - HTTPS (via Vercel)                                │
│ - Helmet.js headers                                 │
└─────────────────────────────────────────────────────┘
         ▼
┌─────────────────────────────────────────────────────┐
│ Layer 2: Request Validation                         │
│ - CORS origin whitelist (3 dev ports only)         │
│ - Content-Type: application/json only              │
│ - Body size limits                                  │
└─────────────────────────────────────────────────────┘
         ▼
┌─────────────────────────────────────────────────────┐
│ Layer 3: Authentication                             │
│ Option A: JWT Bearer Token                          │
│  - Signed with HS256 algorithm                     │
│  - Secret: JWT_SECRET (env var)                    │
│  - Expiry: 8 hours                                  │
│  - Verification: Via jsonwebtoken lib              │
│                                                     │
│ Option B: API Key                                   │
│  - Format: ak_[32-char-hex]                        │
│  - Fingerprint: SHA256 hash (indexed)              │
│  - Secret: bcrypt hashed (never stored plaintext)  │
│  - Validation: Via requireApiKey middleware        │
└─────────────────────────────────────────────────────┘
         ▼
┌─────────────────────────────────────────────────────┐
│ Layer 4: Authorization                              │
│ - Role-based: USER / ADMIN / SUPERADMIN            │
│ - State-level: stateAccess JSON array              │
│ - Status-based: PENDING_APPROVAL / ACTIVE / etc.   │
│ - API Key status: ACTIVE / REVOKED                 │
└─────────────────────────────────────────────────────┘
         ▼
┌─────────────────────────────────────────────────────┐
│ Layer 5: Two-Factor Authentication                  │
│ - TOTP (Time-based One-Time Password)              │
│ - Lib: speakeasy                                    │
│ - Window: ±1 time step (30-sec validity)           │
│ - Admin-only requirement                            │
│ - Headers: X-2FA-Code or body.twoFactorCode        │
└─────────────────────────────────────────────────────┘
         ▼
┌─────────────────────────────────────────────────────┐
│ Layer 6: Input Validation                           │
│ - express-validator library                         │
│ - Type checking (string, int, email, etc.)         │
│ - Length limits (min/max)                          │
│ - Format validation (email, UUID, etc.)            │
│ - Whitelist patterns (status, plan, role)          │
│ - Custom validators (API key format)               │
└─────────────────────────────────────────────────────┘
         ▼
┌─────────────────────────────────────────────────────┐
│ Layer 7: Rate Limiting                              │
│ - Per-user daily quotas (by plan)                  │
│ - Redis-backed counter (survives server restart)   │
│ - Automatic alerts at 80%, 95% thresholds          │
│ - Email notification to user                        │
└─────────────────────────────────────────────────────┘
         ▼
┌─────────────────────────────────────────────────────┐
│ Layer 8: Data Protection                            │
│ - Password: bcrypt hashing (10 rounds)             │
│ - API Secret: bcrypt hashing + fingerprinting      │
│ - 2FA Secret: base32 encoded                        │
│ - Sensitive fields: Sanitized in responses         │
│ - PII: Not logged (except for admin dashboard)     │
└─────────────────────────────────────────────────────┘
```

---

## Testing Architecture

```
┌──────────────────────────────────┐
│ Jest Test Runner                 │
│ - Node environment               │
│ - Serial execution (maxWorkers:1)│
│ - 10 second timeout per test     │
└────────┬─────────────────────────┘
         │
┌────────▼─────────────────────────┐
│ Setup File (setup.js)            │
│ beforeAll:                        │
│ - Connect to TEST_DATABASE_URL   │
│ - Clean database (7 tables)      │
│ - Set global.prisma              │
│                                  │
│ afterEach:                        │
│ - Flush pending usage logs       │
│ - Clear cache                    │
│                                  │
│ afterAll:                         │
│ - Disconnect Prisma              │
└────────┬─────────────────────────┘
         │
    ┌────┴──────┬──────────┬──────────┬───────────┐
    │            │          │          │           │
    ▼            ▼          ▼          ▼           ▼
┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐
│Auth    │ │Search  │ │States  │ │Usage   │ │Custom  │
│Tests   │ │Tests   │ │Tests   │ │Tests   │ │Tests   │
│(19)    │ │(8)     │ │(6)     │ │(8)     │ │(N/A)   │
└────────┘ └────────┘ └────────┘ └────────┘ └────────┘
    │            │          │          │
    └────┬───────┴──────────┴──────────┘
         │
         ▼
    ┌─────────────────────────────────┐
    │ Supertest (HTTP assertions)     │
    │ - Makes requests to Express app │
    │ - Tests request/response cycle  │
    │ - Validates status codes        │
    │ - Checks response body          │
    └────────┬────────────────────────┘
             │
             ▼
    ┌─────────────────────────────────┐
    │ Coverage Report (optional)      │
    │ - Excludes: index.js, prisma.js,│
    │           redis.js              │
    │ - Tracks: statements, branches, │
    │           functions, lines      │
    └─────────────────────────────────┘
```

---

## Deployment Architecture

```
┌─────────────────────────────────────┐
│ GitHub Repository                   │
│ (Source Code)                       │
└────────┬────────────────────────────┘
         │
         │ On push to main/staging/preview
         │
         ▼
┌─────────────────────────────────────┐
│ GitHub Actions Workflow             │
│ (.github/workflows/backend-deploy.yml)
│ - Trigger: Push event               │
│ - Run: Jest tests (if configured)   │
│ - Deploy: To Vercel                 │
└────────┬────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────┐
│ Vercel                              │
│ - Retrieves secrets                 │
│   @DATABASE_URL                     │
│   @JWT_SECRET                       │
│   @SMTP_* (email config)            │
│   @UPSTASH_* (cache config)         │
│   etc.                              │
└────────┬────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────┐
│ Node 18.x Runtime                   │
│ - 1024 MB RAM                       │
│ - 10 second max function duration   │
└────────┬────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────┐
│ Serverless Function                 │
│ (api/[[...all]].js wrapper)         │
│ - Wraps Express app                 │
│ - Converts to serverless handler    │
│ - Handles cold starts               │
└────────┬────────────────────────────┘
         │
    ┌────┴────┬──────────┬──────────┐
    │          │          │          │
    ▼          ▼          ▼          ▼
┌────────┐ ┌────────┐ ┌────────┐ ┌──────────┐
│Vercel  │ │Vercel  │ │Upstash │ │SMTP      │
│Postgres│ │Upstash │ │Redis   │ │Service   │
│        │ │(Cache) │ │(REST)  │ │(Email)   │
└────────┘ └────────┘ └────────┘ └──────────┘
```

