# Bluestock API Backend - Implementation Summary

## Project Overview
- **Framework**: Express.js (Node.js)
- **Database**: PostgreSQL with Prisma ORM
- **Deployment**: Vercel (Serverless)
- **Version**: 1.0.0
- **Type**: CommonJS modules
- **Authentication**: JWT + API Keys + 2FA (TOTP)
- **Caching**: Upstash Redis (with in-memory fallback for local dev)

---

## 1. API ROUTES & ENDPOINTS

### Authentication Routes (`/api/v1/auth`)
**File**: [src/routes/auth.js](src/routes/auth.js)

#### User Account Management
- `POST /auth/register` - Register new B2B customer
  - Validates: businessName, email, password (min 8 chars)
  - Returns: JWT token, sanitized user, pending approval message
  - Sends: Email verification link (24hr validity)

- `POST /auth/verify-email` - Verify email address
  - Validates: Verification token from Redis
  - Returns: Sanitized user data
  - Sets: emailVerified = true

- `POST /auth/resend-verification` - Resend verification email
  - Input: email address
  - Returns: Success message

- `POST /auth/login` - Authenticate user
  - Validates: email, password
  - Checks: Email verified, account active
  - Returns: JWT token, sanitized user

- `GET /auth/me` - Get current user (Requires: Bearer token)
  - Returns: Sanitized user profile with all attributes

#### Two-Factor Authentication (2FA / TOTP)
- `POST /auth/2fa/setup` - Initialize 2FA
  - Returns: QR code URL (otpauthUrl), base32 secret
  - Stores: Temporary secret in database

- `POST /auth/2fa/verify` - Confirm 2FA setup
  - Validates: TOTP token against temp secret
  - Activates: 2FA on account
  - Stores: twoFactorSecret (permanent)

- `POST /auth/2fa/disable` - Disable 2FA
  - Validates: TOTP token for confirmation
  - Removes: All 2FA data

#### API Key Management
- `GET /auth/apikeys` - List user's API keys (Requires: Bearer token)
  - Returns: Array of key metadata (id, status, createdAt)
  - Limit: 5 keys per user

- `POST /auth/apikeys` - Generate new API key (Requires: Bearer token)
  - Returns: New API key (format: `ak_[32-char-hex]`)
  - Stores: Fingerprint (SHA256) + bcrypt-hashed secret
  - Visible: Only on creation

- `PATCH /auth/apikeys/:id/revoke` - Revoke an API key (Requires: Bearer token)
  - Status change: ACTIVE → REVOKED
  - Returns: Updated key metadata

- `PATCH /auth/apikeys/:id/rotate` - Rotate an API key (Requires: Bearer token)
  - Action: Revoke old key + generate new one
  - Returns: New API key with metadata

---

### Admin Routes (`/api/v1/admin`)
**File**: [src/routes/admin.js](src/routes/admin.js)

**Middleware Protection**: Requires JWT Bearer token + Admin/SuperAdmin role + 2FA verification

#### Dashboard & Analytics
- `GET /admin/overview` - Dashboard statistics (Admin only)
  - Returns:
    - totalVillages (count)
    - activeUsers, pendingUsers
    - totalRequests (today), successRate
    - planData (distribution by plan)
    - topStates (top 5 by village count)

#### User Management
- `GET /admin/users` - List all users (Admin only)
  - Returns: All users with: email, businessName, role, plan, status, dailyLimit, stateAccess, createdAt, updatedAt
  - Ordered: By creation date (descending)

- `PATCH /admin/users/:id/approve` - Approve user account
  - Status change: PENDING_APPROVAL → ACTIVE
  - Sends: Status email to user

- `PATCH /admin/users/:id/reject` - Reject user account
  - Status change: PENDING_APPROVAL → REJECTED
  - Sends: Status email to user

- `PATCH /admin/users/:id/access` - Set state-level access restrictions
  - Input: stateIds array
  - Stores: JSON array in stateAccess field
  - Effect: Limits API calls to specified states

- `PATCH /admin/users/:id/plan` - Update user's plan & daily limit
  - Input: plan (FREE, PREMIUM, PRO, UNLIMITED), optional dailyLimit
  - Plan limits: FREE(5k), PREMIUM(50k), PRO(300k), UNLIMITED(1M) requests/day
  - If dailyLimit not provided: Uses plan's default

- `PATCH /admin/users/:id/limit` - Override daily request limit
  - Input: dailyLimit (positive integer)
  - Sets: Custom daily request threshold

#### Logging & Monitoring
- `GET /admin/logs` - Query usage logs (Admin only)
  - Query filters:
    - endpoint (substring match, case-insensitive)
    - status (HTTP status code)
    - userEmail (substring match, case-insensitive)
    - fromDate, toDate (ISO datetime)
    - page, limit (pagination, max 100 per page)
  - Returns:
    - id, time (ISO), user, endpoint, status, ms (response time), apiKeyId
    - meta: total, count, page, limit, totalPages

#### Reference Data
- `GET /admin/states` - List all states
  - Returns: id, name, code

---

### Usage Analytics Routes (`/api/v1/usage`)
**File**: [src/routes/usage.js](src/routes/usage.js)

**Middleware Protection**: Requires JWT Bearer token

- `GET /usage/dashboard` - User's usage dashboard (Requires: Bearer token)
  - Returns:
    - dailyLimit (user's limit)
    - todayRequests (requests made today)
    - totalRequests (all-time)
    - remainingToday (quota left)
    - last7Days (array with day & request count)
    - successRate (% of successful requests)
    - averageResponse (avg response time in ms)
    - topEndpoints (most-used endpoints)

- `GET /usage/logs` - User's own usage logs (Requires: Bearer token)
  - Query filters:
    - endpoint (substring)
    - statusCode (HTTP status)
    - page, limit (pagination, max 100/page)
  - Returns: endpoint, method, statusCode, responseTime, date for each log

---

### Data Lookup Routes (Requires: X-API-Key header + Rate Limiting)

#### States Route
**File**: [src/routes/states.js](src/routes/states.js)

- `GET /api/v1/states` - List all states (cached 24h)
  - Returns: id, name, code, districtCount
  - Ordered: By name (ascending)

- `GET /api/v1/states/:id` - Get single state by ID
  - Returns: Full state with district count

#### Districts Route
**File**: [src/routes/districts.js](src/routes/districts.js)

- `GET /api/v1/districts` - List districts with pagination & caching
  - Query filters:
    - stateId (optional, filters to state)
    - page (default 1)
    - limit (default 20, max 100)
  - Cache: 1 minute (in-memory)
  - Returns: id, name, code, stateId, subDistrictCount
  - Pagination: Includes totalCount & totalPages

#### SubDistricts Route
**File**: [src/routes/subdistricts.js](src/routes/subdistricts.js)

- `GET /api/v1/subdistricts` - List subdistricts
  - Query filters:
    - districtId (optional)
  - Returns: id, name, code, districtId, district info, villageCount
  - Ordered: By name (ascending)

- `GET /api/v1/subdistricts/:id` - Get single subdistrict by ID
  - Returns: Full subdistrict with parent district info

#### Villages Route
**File**: [src/routes/villages.js](src/routes/villages.js)

- `GET /api/v1/villages` - List villages with pagination
  - Query filters:
    - subDistrictId (required)
    - page (default 1)
    - limit (default 20)
  - Returns: id, name, code, full address hierarchy
  - Pagination: Includes total & totalPages

- `GET /api/v1/villages/:id` - Get single village by ID
  - Returns: Full village with hierarchy

#### Search Route
**File**: [src/routes/search.js](src/routes/search.js)

- `GET /api/v1/search` - Full-text village search
  - Query filters:
    - q (search query, min 2 chars, max 50 results)
    - stateId (optional, numeric)
    - districtId (optional, numeric)
    - limit (optional, max 50)
  - Returns: Array with:
    - value (village id)
    - label (village name)
    - code (village code)
    - fullAddress
    - hierarchy (village, subDistrict, district, state)
  - Search: Case-insensitive, partial match on village name

- `GET /api/v1/search/autocomplete` - Autocomplete suggestions
  - Query: q (search term)
  - Returns: Suggestions for village names

---

### Health Check
- `GET /` - Root endpoint
  - Returns: `{ success: true, message: 'Bluestock Village API is running!', version: 'v1' }`

---

## 2. MIDDLEWARE COMPONENTS

### Authentication Middleware
**File**: [src/middleware/auth.js](src/middleware/auth.js)

#### `requireApiKey`
- Validates X-API-Key header format: `ak_[0-9a-f]{32}`
- Checks: API key fingerprint (SHA256) exists in database
- Verifies: Key status is ACTIVE
- Compares: API key against bcrypt-hashed secret
- Supports: Demo mode with `DEMO_API_KEY` environment variable
- Populates: req.user, req.apiKey, req.stateAccess, req.demoMode

#### `requireAuth` (within auth.js)
- Validates: Bearer token in Authorization header
- Verifies: JWT signature against JWT_SECRET
- Checks: User exists and status is ACTIVE
- Populates: req.user

#### `requireAdmin`
- Checks: User role is ADMIN or SUPERADMIN
- Throws: 403 if not admin

#### `requireAdmin2FA`
- If 2FA enabled on admin: Validates X-2FA-Code header
- Uses: TOTP verification with speakeasy library
- Window: ±1 time step

---

### Request Logging Middleware
**File**: [src/middleware/logger.js](src/middleware/logger.js)

#### `requestLogger`
- Logs: Every API response (if authenticated)
- Records:
  - userId, apiKeyId
  - endpoint (URL path)
  - method (HTTP verb)
  - statusCode
  - responseTime (ms)
  - date (current timestamp)
- Storage: Asynchronous writes to Usage table
- Cleanup: `flushUsageLogs()` ensures pending logs complete

---

### Rate Limiting Middleware
**File**: [src/middleware/rateLimit.js](src/middleware/rateLimit.js)

#### `rateLimit(plan)`
- Storage: Redis or in-memory map
- Key format: `ratelimit:{apiKey}:{YYYY-MM-DD}`
- Tracks: Daily request count per API key
- Limits by plan:
  - FREE: 5,000 requests/day
  - PREMIUM: 50,000 requests/day
  - PRO: 300,000 requests/day
  - UNLIMITED: 1,000,000 requests/day
- Custom limits: User-specific dailyLimit overrides plan
- Headers set:
  - X-RateLimit-Limit
  - X-RateLimit-Remaining
  - X-RateLimit-Reset
- Alerts: Sends email at 80% and 95% thresholds
- Alert suppression: Prevents duplicate emails within 24h

---

### Global Error Handling
**File**: [src/app.js](src/app.js)

- Express error handler middleware
- Delegates to `handleError` utility
- CORS enabled for ports 5173, 5174, 5175
- Helmet security headers applied
- JSON body parser configured

---

## 3. KEY UTILITIES & HELPER FUNCTIONS

### Error Handler
**File**: [src/utils/errorHandler.js](src/utils/errorHandler.js)

#### `AppError` (Custom Error Class)
- Properties: message, statusCode, isOperational (flag)
- Usage: `throw new AppError('message', 400)`

#### `handleError(res, error)`
- Logs: All errors to console
- Responses:
  - Operational errors: Returns statusCode + error message
  - Prisma errors (code starts with 'P'): Returns 500 "Database error"
  - Unknown errors: Returns 500 "Internal Server Error"

---

### Environment Configuration
**File**: [src/lib/env.js](src/lib/env.js)

#### `getRequiredEnv(key)`
- Throws: Error if key not found
- Returns: Environment variable value

#### `getEnv(key, fallback)`
- Returns: Environment variable or fallback value

#### `getIntEnv(key, fallback)`
- Parses: Integer value from env var
- Throws: Error if not valid integer
- Returns: Parsed int or fallback

---

### Prisma Database Connection
**File**: [src/lib/prisma.js](src/lib/prisma.js)

- Pattern: Singleton connection (prevents connection leaks)
- Logging: Errors and warnings only
- Reuses: Global prisma instance in development
- Datasource: PostgreSQL via DATABASE_URL env var

---

### Redis / Caching
**File**: [src/lib/redis.js](src/lib/redis.js)

#### Upstash Redis Client
- Connection: REST API to Upstash Redis
- Fallback: In-memory Map (development only)

#### Helper Functions

- `redis.get(key)` - Retrieve cached value
- `redis.setex(key, ttl, value)` - Set with TTL (seconds)
- `redis.del(key)` - Delete key

#### `cached(key, fetchFn, ttlSeconds)`
- Purpose: Cache-or-fetch pattern
- Returns: Cached value or executes fetchFn and caches result
- TTL: Default 3600 seconds (1 hour)
- Example: `const states = await cached('states:all', () => prisma.state.findMany(), 86400)`

#### `clearCache()`
- Clears: In-memory cache (no-op if using Upstash)

---

### Email Service
**File**: [src/lib/email.js](src/lib/email.js)

#### `sendEmail({ to, subject, text, html })`
- Transport: Nodemailer with SMTP configuration
- Config: HOST, PORT, USER, PASS from env vars
- Fallback: Logs email to console if SMTP not configured

#### `sendUsageAlert({ user, current, limit, threshold })`
- Triggered: When user reaches 80% or 95% of daily limit
- Sends: Email with usage percentage and quota details

#### `sendUserStatusEmail({ user, status })`
- Triggered: When user account status changes
- Status values: ACTIVE, REJECTED, SUSPENDED, etc.
- Sends: Formatted HTML/text email

---

### Admin Bootstrap
**File**: [src/lib/adminBootstrap.js](src/lib/adminBootstrap.js)

#### `bootstrapAdmin()`
- Purpose: Create initial admin user if none exists
- Checks: 
  - If ADMIN_EMAIL and ADMIN_PASSWORD env vars provided
  - If admin/superadmin role already exists
  - If user with email already exists
- Creates: SUPERADMIN with UNLIMITED plan, 1M daily limit
- Hashes: Password with bcrypt (10 rounds)
- Attributes: businessName, email, phone, gst (from env)

---

## 4. TESTING STRUCTURE

**Framework**: Jest + Supertest

### Test Setup
**File**: [src/__tests__/setup.js](src/__tests__/setup.js)

- **Before All**: 
  - Connects to TEST_DATABASE_URL or DATABASE_URL
  - Cleans database (deletes in foreign-key-safe order)
  - Sets global.prisma

- **After Each**:
  - Flushes pending usage logs
  - Clears cache

- **After All**:
  - Disconnects Prisma

- **Cleanup Order**: Usage → ApiKey → Village → SubDistrict → District → State → Country → User

### Test Files

#### 1. Auth Tests (`auth.test.js`)
- POST /auth/register
  - ✓ Register new user successfully
  - ✓ Reject existing email
  - ✓ Validate required fields
- POST /auth/login
  - ✓ Login with correct credentials
  - ✓ Reject invalid credentials
  - ✓ Require email verification
  - ✓ Check account active status
- GET /auth/me
  - ✓ Return user profile with token
- API Key management
  - ✓ Create API key
  - ✓ List user's API keys
  - ✓ Revoke API key
  - ✓ Rotate API key

#### 2. Search Tests (`search.test.js`)
- GET /api/v1/search
  - ✓ Search villages by name
  - ✓ Filter by stateId
  - ✓ Filter by districtId
  - ✓ Validate minimum query length
  - ✓ Respect result limit

#### 3. States Tests (`states.test.js`)
- GET /api/v1/states
  - ✓ Return all states
  - ✓ Verify caching
- GET /api/v1/states/:id
  - ✓ Return single state
  - ✓ Handle invalid ID

#### 4. Usage Tests (`usage.test.js`)
- GET /api/v1/usage/dashboard
  - ✓ Return usage metrics
  - ✓ Calculate 7-day breakdown
  - ✓ Calculate success rate
  - ✓ Show top endpoints

### Jest Configuration
**File**: [jest.config.js](jest.config.js)

```javascript
{
  testEnvironment: 'node',
  testMatch: ['**/__tests__/**/*.test.js'],
  collectCoverageFrom: ['src/**/*.js', '!src/index.js', '!src/lib/prisma.js', '!src/lib/redis.js'],
  setupFilesAfterEnv: ['<rootDir>/src/__tests__/setup.js'],
  testTimeout: 10000,
  maxWorkers: 1  // Serial execution to avoid DB conflicts
}
```

### Running Tests
```bash
npm test              # Run all tests once
npm run test:watch   # Watch mode
npm run test:coverage # Coverage report
```

---

## 5. ENVIRONMENT CONFIGURATION

### Required Environment Variables
```
DATABASE_URL                    # PostgreSQL connection string
JWT_SECRET                      # Secret for JWT signing
UPSTASH_REDIS_REST_URL         # Upstash Redis endpoint
UPSTASH_REDIS_REST_TOKEN       # Upstash Redis token
SMTP_HOST                       # Email server host
SMTP_PORT                       # Email server port (465 for secure)
SMTP_USER                       # Email account username
SMTP_PASS                       # Email account password
EMAIL_FROM                      # Sender email address
ADMIN_EMAIL                     # Initial admin email
ADMIN_PASSWORD                  # Initial admin password
```

### Optional Environment Variables
```
PORT                            # Server port (default: 3000)
FRONTEND_URL                    # Frontend URL (default: http://localhost:5173)
ADMIN_BUSINESS_NAME            # Admin business name
ADMIN_PHONE                     # Admin phone
ADMIN_GST                       # Admin GST number
DEMO_API_KEY                    # Demo API key for testing (format: ak_[32-hex])
DEMO_STATE_NAME                 # Demo state name (default: Maharashtra)
NODE_ENV                        # Environment (development/production)
TEST_DATABASE_URL              # Test database URL (if different from DATABASE_URL)
```

### Vercel Configuration
**File**: [vercel.json](vercel.json)

```json
{
  "version": 3,
  "functions": {
    "api/[[...all]].js": {
      "runtime": "nodejs18.x",
      "memory": 1024,
      "maxDuration": 10
    }
  },
  "env": {
    "DATABASE_URL": "@DATABASE_URL",
    "JWT_SECRET": "@JWT_SECRET",
    // ... other env vars reference Vercel secrets (@NAME format)
  }
}
```

---

## 6. SWAGGER / OpenAPI DOCUMENTATION

**File**: [src/swagger.js](src/swagger.js)

### OpenAPI Specification
- **Version**: 3.0.1
- **Title**: Bluestock Village API
- **Version**: 1.0.0
- **Description**: B2B API for village lookup, API keys, and usage analytics

### Server
- Development: `http://localhost:3000/api/v1`

### Security Schemes
1. **bearerAuth** (JWT)
   - Type: HTTP Bearer
   - Format: JWT

2. **apiKeyAuth** (X-API-Key)
   - Type: API Key
   - Location: Header
   - Name: X-API-Key

### Endpoint Coverage in Swagger
- Auth Register: POST /auth/register
- Auth Login: POST /auth/login
- States: GET /api/v1/states
- Schemas defined: User, ApiKey

### Exposure
- **Swagger UI**: `/api-docs`
- **OpenAPI JSON**: `GET /openapi.json`

---

## 7. DATABASE SCHEMA (Prisma)

**File**: [prisma/schema.prisma](prisma/schema.prisma)

### Models

#### Geography Models
- **Country**
  - id, name (unique), code (unique), createdAt
  - Relations: states[]

- **State**
  - id, name, code (unique), countryId, createdAt
  - Relations: country, districts[]
  - Indexes: countryId, name

- **District**
  - id, name, code (unique), stateId, createdAt
  - Relations: state, subDistricts[]
  - Indexes: stateId, name

- **SubDistrict**
  - id, name, code (unique), districtId, createdAt
  - Relations: district, villages[]
  - Indexes: districtId, name

- **Village**
  - id, name, code (unique), subDistrictId, createdAt
  - Relations: subDistrict
  - Indexes: subDistrictId, name

#### User & Auth Models
- **User**
  - id, businessName, email (unique), phone, gst
  - passwordHash (bcrypt)
  - role: USER | ADMIN | SUPERADMIN (default: USER)
  - plan: FREE | PREMIUM | PRO | UNLIMITED (default: FREE)
  - status: PENDING_APPROVAL | ACTIVE | SUSPENDED | REJECTED (default: PENDING_APPROVAL)
  - emailVerified: boolean (default: false)
  - dailyLimit: int (default: 5000)
  - stateAccess: JSON (null = unrestricted)
  - twoFactorEnabled: boolean
  - twoFactorSecret, twoFactorTempSecret: for TOTP
  - createdAt, updatedAt
  - Relations: apiKeys[], usages[]
  - Indexes: email, status, role

- **ApiKey**
  - id, fingerprint (unique, SHA256), secretHash (bcrypt)
  - userId, status: ACTIVE | REVOKED (default: ACTIVE)
  - createdAt
  - Relations: user, usages[]
  - Indexes: userId, fingerprint

#### Analytics Model
- **Usage**
  - id, userId, apiKeyId (optional)
  - endpoint, method (GET/POST/etc), statusCode, responseTime (ms)
  - date: DateTime (default: now())
  - Relations: user, apiKey
  - Indexes: userId, apiKeyId, date

---

## 8. FILE STRUCTURE

```
bluestock_api/backend/
├── package.json                 # Dependencies & scripts
├── jest.config.js              # Jest test configuration
├── vercel.json                 # Vercel deployment config
├── README.md                   # Setup & deployment guide
├── api/
│   └── [[...all]].js          # Vercel serverless wrapper
├── prisma/
│   ├── schema.prisma           # Database schema
│   ├── migrations/             # Migration files
│   └── migration_lock.toml
├── src/
│   ├── index.js               # Server entry point
│   ├── app.js                 # Express app setup
│   ├── swagger.js             # OpenAPI specification
│   ├── middleware/
│   │   ├── auth.js            # API key & JWT validation
│   │   ├── logger.js          # Request logging
│   │   └── rateLimit.js       # Rate limiting
│   ├── routes/
│   │   ├── auth.js            # Auth & API key endpoints
│   │   ├── admin.js           # Admin management
│   │   ├── usage.js           # Usage analytics
│   │   ├── search.js          # Village search
│   │   ├── states.js          # State lookup
│   │   ├── districts.js       # District lookup
│   │   ├── subdistricts.js    # SubDistrict lookup
│   │   └── villages.js        # Village lookup
│   ├── lib/
│   │   ├── prisma.js          # Database connection
│   │   ├── redis.js           # Caching layer
│   │   ├── email.js           # Email service
│   │   ├── env.js             # Env var helpers
│   │   └── adminBootstrap.js  # Admin initialization
│   ├── utils/
│   │   └── errorHandler.js    # Error handling
│   ├── validators/
│   │   ├── search.validator.js
│   │   └── village.validator.js
│   └── __tests__/
│       ├── setup.js           # Test setup & cleanup
│       ├── auth.test.js       # Auth tests
│       ├── search.test.js     # Search tests
│       ├── states.test.js     # State tests
│       └── usage.test.js      # Usage tests
└── python_scripts/
    ├── import_villages.py      # Data import script
    └── dataset/
```

---

## 9. SECURITY FEATURES

1. **Authentication**: JWT (8h expiry) + API Keys (bcrypt hashed)
2. **2FA**: TOTP (Time-based One-Time Password) with speakeasy
3. **Password Security**: bcrypt hashing (10 rounds)
4. **API Key Format**: Fingerprint + Secret Hash (not stored in plaintext)
5. **Rate Limiting**: Per-user daily limits by plan
6. **CORS**: Restricted to localhost development ports
7. **Security Headers**: Helmet.js applied
8. **Input Validation**: express-validator on all inputs
9. **Error Handling**: Generic error messages in production
10. **Email Verification**: Required before account activation

---

## 10. DEPLOYMENT & MIGRATION

### Running Locally
```bash
npm install
npm run dev              # Start development server (nodemon)
npm run bootstrap-admin  # Create initial admin
```

### Migrations
```bash
npm run migrate:dev     # Run migrations locally (generates new ones)
npm run migrate:deploy  # Apply migrations in production
npm run migrate:status  # Check migration status
npm run migrate:rollback # Display rollback guidance
```

### Vercel Deployment
- Configuration: `vercel.json` specifies Node 18.x runtime
- Environment: Secrets referenced via @VARIABLE_NAME format
- Auto-deploy: On pushes to main, staging, preview branches
- Requires: VERCEL_TOKEN, VERCEL_ORG_ID, VERCEL_PROJECT_ID secrets

### API Testing Results
- **all-tests.json** - Test results snapshot
- **auth-results*.json** - Authentication endpoint test results
- Tests cover: Registration, login, 2FA, API keys, admin functions

---

## 11. KEY PATTERNS & BEST PRACTICES

1. **Centralized Error Handling**: AppError class + middleware
2. **Singleton Connections**: Prisma & Redis
3. **Async/Await**: Promise-based error handling
4. **Request Logging**: Middleware-based, captured in database
5. **Caching Strategy**: Redis with in-memory fallback
6. **Input Validation**: Express-validator before DB queries
7. **Pagination**: Standard limit/offset with max limits
8. **Rate Limiting**: Per-API-key daily tracking
9. **CORS Configuration**: Explicit whitelist, no wildcard
10. **Admin Bootstrap**: Automatic on first server start

---

## 12. ENVIRONMENT-SPECIFIC BEHAVIOR

### Local Development
- Redis: Falls back to in-memory Map
- Email: Logs to console if SMTP not configured
- Database: Can use TEST_DATABASE_URL for testing
- CORS: Allows ports 5173, 5174, 5175

### Production (Vercel)
- Redis: Uses Upstash REST API
- Email: Sends via configured SMTP
- Database: Uses DATABASE_URL from Vercel secrets
- JWT Secret: Environment-specific from Vercel secrets

