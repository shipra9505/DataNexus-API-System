# BLUESTOCK B2B API - Architecture Documentation

**Complete system design and architecture overview.**

---

## 🏗️ High-Level System Architecture

```
┌────────────────────────────────────────────────────────────────────┐
│                     B2B CLIENTS & END USERS                         │
│  (Swiggy, Zomato, Logistics, KYC Platforms, etc.)                 │
└────────────────────┬─────────────────────────────────────────────┘
                     │
        ┌────────────┼────────────┐
        │            │            │
    ┌───▼──┐   ┌────▼────┐   ┌──▼────┐
    │ B2B  │   │ Admin   │   │ Demo  │
    │Port  │   │ Port    │   │ App   │
    │5173  │   │ 5174    │   │ 5175  │
    └───┬──┘   └────┬────┘   └──┬────┘
        │           │           │
        └───────────┼───────────┘
                    │
        ┌───────────▼────────────┐
        │   Express.js REST API   │
        │   (Port 3000)           │
        │   Vercel Serverless     │
        │                         │
        │  • Authentication       │
        │  • Rate Limiting        │
        │  • Geographic API       │
        │  • Admin Endpoints      │
        │  • Usage Analytics      │
        └───────────┬────────────┘
                    │
      ┌─────────────┼─────────────┐
      │             │             │
  ┌───▼────┐  ┌────▼───┐   ┌────▼────┐
  │ NeonDB │  │ Upstash│   │ SMTP    │
  │PostgreSQL │ Redis   │   │Service  │
  │           │         │   │(SendGrid)
  │           │         │   │         │
  │ • Country │ • Cache │   │Email    │
  │ • State   │ • Rate  │   │Verify   │
  │ • District│ Limits  │   │Alerts   │
  │ • SubDist │ • Session   │         │
  │ • Village │         │   │         │
  │ (640K+)   │ (1min,  │   │         │
  │           │  24h)   │   │         │
  │ Indexes:  │         │   │         │
  │ • name    │         │   │         │
  │ • FK      │         │   │         │
  └───────────┘ └─────────┘ └──────────┘

DEPLOYMENT: Vercel Edge Network (Global CDN)
MONITORING: Sentry (Error tracking)
CI/CD: GitHub Actions
```

---

## 📊 Request Processing Flow

### 1. API Request Flow (with API Key)

```
Client Request
     │
     │ GET /api/v1/search?q=village_name
     │ Headers: X-API-Key: ak_xxxxx
     │
     ▼
┌─────────────────────────┐
│ Vercel Edge Gateway     │
│ • Route to serverless   │
│ • Add request ID        │
└──────────────┬──────────┘
               │
               ▼
         ┌──────────────────────┐
         │ Express App          │
         │ • Request logging    │
         │ • CORS check         │
         └──────────┬───────────┘
                    │
                    ▼
         ┌──────────────────────┐
         │ API Key Middleware   │
         │ • Verify X-API-Key   │
         │ • Find user          │
         │ • Get plan           │
         └──────────┬───────────┘
                    │ ✅ Valid?
                    ▼
         ┌──────────────────────┐
         │ Rate Limit Check     │
         │ • Check Redis        │
         │ • User daily quota   │
         │ • Per-minute limit   │
         └──────────┬───────────┘
                    │ ✅ Under limit?
                    ▼
         ┌──────────────────────┐
         │ Route Handler        │
         │ /api/v1/search       │
         └──────────┬───────────┘
                    │
                    ▼
         ┌──────────────────────┐
         │ Check Redis Cache    │
         │ • Search results     │
         │ • State data         │
         └──────────┬───────────┘
                    │
           ┌────────┴────────┐
           │                 │
        ✅ Cache Hit      ❌ Miss
           │                 │
           │                 ▼
           │          ┌──────────────────────┐
           │          │ Query PostgreSQL     │
           │          │ • Trigram search     │
           │          │ • Join hierarchy     │
           │          │ • Order & limit      │
           │          └──────────┬───────────┘
           │                     │
           │                     ▼
           │          ┌──────────────────────┐
           │          │ Cache in Redis       │
           │          │ (1 min TTL)          │
           │          └──────────┬───────────┘
           │                     │
           └─────────┬───────────┘
                     │
                     ▼
         ┌──────────────────────┐
         │ Log to Usage Table   │
         │ • User ID            │
         │ • Endpoint           │
         │ • Response time      │
         │ • Status code        │
         └──────────┬───────────┘
                    │
                    ▼
         ┌──────────────────────┐
         │ Format Response      │
         │ {                    │
         │  success, count,     │
         │  data, meta          │
         │ }                    │
         └──────────┬───────────┘
                    │
                    ▼
              ▼ Response to Client
```

### 2. JWT Authentication Flow (User Login)

```
User Registration/Login
     │
     ▼
┌──────────────────────────┐
│ POST /auth/register      │
│ • Email validation       │
│ • GST validation         │
│ • Password hash (bcrypt) │
│ • Create user record     │
└───────────┬──────────────┘
            │
            ▼
      ┌─────────────────┐
      │ Send Email      │
      │ Verification    │
      │ Link            │
      └────────┬────────┘
               │
               ▼
        ┌──────────────┐
        │ User Status: │
        │ PENDING_     │
        │ APPROVAL     │
        └──────┬───────┘
               │
        [Admin Approves]
               │
               ▼
        ┌──────────────┐
        │ User Status: │
        │ ACTIVE       │
        └──────┬───────┘
               │
               ▼
┌──────────────────────────┐
│ POST /auth/login         │
│ • Email verification     │
│ • Password hash check    │
│ • Generate JWT token     │
│ • 8-hour expiry          │
└───────────┬──────────────┘
            │
            ▼
     Response with JWT
     {
       "token": "eyJhbGc...",
       "expiresIn": 28800
     }
            │
            ▼
  [Stored in localStorage]
            │
            ▼
┌──────────────────────────┐
│ Future Requests          │
│ Header:                  │
│ Authorization:           │
│ Bearer eyJhbGc...        │
└──────────────────────────┘
```

### 3. Admin Approval Workflow

```
B2B User Registers
        │
        ▼
   ┌─────────────┐
   │ Email Sent  │
   │ Confirmation│
   └─────┬───────┘
         │
         ▼
   Status: PENDING_APPROVAL
         │
         ▼
   Admin Notified
         │
         ▼
┌──────────────────────────┐
│ Admin Dashboard          │
│ • Review user details    │
│ • Check GST validity     │
│ • Check email domain     │
└────────────┬─────────────┘
             │
      ┌──────┴──────┐
      │             │
   ✅ Approve    ❌ Reject
      │             │
      ▼             ▼
┌──────────────┐ ┌──────────────┐
│Status:ACTIVE │ │Status:REJECTED
│ User gets:   │ │ User notified
│ • API key    │ │ • Can re-apply
│ • Dashboard  │ │ • Reason sent
└──────────────┘ └──────────────┘
```

---

## 🗄️ Database Schema (3NF Normalization)

### Entity Relationship Diagram

```
┌─────────────┐
│  Country    │
├─────────────┤
│ id (PK)     │
│ name (UK)   │ "India"
│ code (UK)   │ "IN"
│ createdAt   │
└────────┬────┘
         │ 1:N
         │
         ▼
┌─────────────┐
│    State    │
├─────────────┤
│ id (PK)     │
│ name        │ "Maharashtra"
│ code (UK)   │ "MH"
│ countryId   │ (FK→Country)
│ createdAt   │
│ @@index[countryId] ◄──── Performance
└────────┬────┘
         │ 1:N
         │
         ▼
┌─────────────────┐
│   District      │
├─────────────────┤
│ id (PK)         │
│ name            │ "Nandurbar"
│ code (UK)       │ "ND001"
│ stateId (FK)    │ (FK→State)
│ createdAt       │
│ @@index[stateId]│ ◄──── Performance
└────────┬────────┘
         │ 1:N
         │
         ▼
┌──────────────────┐
│  SubDistrict     │
├──────────────────┤
│ id (PK)          │
│ name             │ "Akkalkuwa"
│ code (UK)        │ "AK001"
│ districtId (FK)  │ (FK→District)
│ createdAt        │
│ @@index[districtId] ◄── Performance
└────────┬─────────┘
         │ 1:N
         │
         ▼
┌──────────────────┐
│   Village        │
├──────────────────┤
│ id (PK)          │
│ name             │ "Manibeli"
│ code (UK)        │ "MB001"
│ subDistrictId    │ (FK→SubDistrict)
│ createdAt        │
│ @@index[name]    │ ◄── Search
│ @@index[FK]      │ ◄── Join
└──────────────────┘

GEOGRAPHIC DATA: 640,867 villages across India

┌──────────────────┐
│     User         │
├──────────────────┤
│ id (PK)          │
│ email (UK)       │
│ businessName     │
│ passwordHash     │ (bcrypt)
│ role             │ (USER/ADMIN)
│ plan             │ (FREE/PREMIUM/PRO)
│ status           │ (PENDING_APPROVAL/ACTIVE)
│ stateAccess      │ (JSON array or null)
│ dailyLimit       │ (5000/50000/300000/1M)
│ twoFactorEnabled │
│ createdAt        │
└────────┬─────────┘
         │ 1:N
         ▼
┌──────────────────┐
│    ApiKey        │
├──────────────────┤
│ id (PK)          │
│ fingerprint (UK) │ ak_a1b2c3d4...
│ secretHash       │ (bcrypt)
│ userId (FK)      │ (FK→User)
│ status           │ (ACTIVE/REVOKED)
│ createdAt        │
└──────────────────┘

┌──────────────────┐
│    Usage         │
├──────────────────┤
│ id (PK)          │
│ userId (FK)      │ (FK→User)
│ apiKeyId (FK)    │ (FK→ApiKey)
│ endpoint         │ "/search"
│ method           │ "GET"
│ statusCode       │ 200
│ responseTime     │ 47 (ms)
│ date             │ 2026-05-05T10:30:00Z
└──────────────────┘
```

---

## 🔐 Middleware Stack (Request Processing)

```
Request
   │
   ▼
┌─────────────────────────────┐
│ 1. Express Middleware       │
│    • bodyParser (JSON)      │
│    • cors                   │
│    • helmet (security)      │
│    • morgan (logging)       │
└──────────────┬──────────────┘
               │
               ▼
┌─────────────────────────────┐
│ 2. Route Validation         │
│    • Parameter validation   │
│    • Query string parsing   │
└──────────────┬──────────────┘
               │
      ┌────────┴────────┐
      │                 │
   Public API       Protected
   Endpoints        (JWT/API Key)
      │                 │
      │                 ▼
      │        ┌─────────────────────┐
      │        │ 3. Authentication   │
      │        │    • JWT validation │
      │        │    • API key lookup │
      │        │    • User context   │
      │        └────────┬────────────┘
      │                 │
      │                 ▼
      │        ┌─────────────────────┐
      │        │ 4. Rate Limiting    │
      │        │    • Check Redis    │
      │        │    • Daily quota    │
      │        │    • Per-minute cap │
      │        └────────┬────────────┘
      │                 │
      └────────┬────────┘
               │
               ▼
    ┌──────────────────────┐
    │ 5. Route Handler     │
    │    (Business Logic)  │
    └──────┬───────────────┘
           │
           ▼
    ┌──────────────────────┐
    │ 6. Async Logging     │
    │    • Usage table     │
    │    • Non-blocking    │
    └──────┬───────────────┘
           │
           ▼
    ┌──────────────────────┐
    │ 7. Response Format   │
    │ {                    │
    │  success, count,     │
    │  data, meta, error   │
    │ }                    │
    └──────┬───────────────┘
           │
           ▼
    ┌──────────────────────┐
    │ 8. Error Handler     │
    │    • AppError class  │
    │    • Status codes    │
    │    • Error messages  │
    └──────┬───────────────┘
           │
           ▼
         Response
```

---

## 💾 Data Flow - API Request to Response

```
┌─────────────────────────────────────────────────────────┐
│ FLOW: GET /search?q=village_name&stateId=2              │
└─────────────────────────────────────────────────────────┘

1️⃣ CLIENT REQUEST
   ├─ API Key: X-API-Key: ak_xxxxx
   ├─ Query: q=village_name, stateId=2
   └─ Method: GET

2️⃣ VERCEL GATEWAY
   ├─ Route to serverless function
   ├─ Generate request ID: req_xxxxx
   └─ Timestamp: 2026-05-05T10:30:00Z

3️⃣ EXPRESS MIDDLEWARE
   ├─ CORS: ✅ Port 5173 allowed
   ├─ Helmet: ✅ Security headers
   └─ Morgan: ✅ Log request

4️⃣ API KEY VALIDATION
   ├─ Extract API key from header
   ├─ Query ApiKey table
   ├─ Verify fingerprint
   ├─ Check bcrypt secret hash
   ├─ Lookup User record
   └─ Get plan: "PREMIUM" (50K daily limit)

5️⃣ RATE LIMIT CHECK
   ├─ Query Redis: "user:2:usage:2026-05-05"
   ├─ Current count: 12,450 requests
   ├─ Daily limit: 50,000
   ├─ Remaining: 37,550 ✅
   └─ Set response headers:
      X-RateLimit-Limit: 50000
      X-RateLimit-Remaining: 37550
      X-RateLimit-Reset: 2026-05-06T00:00:00Z

6️⃣ BUSINESS LOGIC
   ├─ Parse query: q="village_name"
   ├─ Check Redis cache (1 min TTL)
   └─ Cache MISS → Query database

7️⃣ DATABASE QUERY
   ├─ Table: Village
   ├─ Trigram search: name ILIKE '%village%'
   ├─ Filter: subDistrict.district.stateId = 2
   ├─ Limit: 20 results
   ├─ Join hierarchy:
   │  └─ Village
   │     └─ SubDistrict
   │        └─ District
   │           └─ State
   │              └─ Country
   └─ Result: 12 villages matching query

8️⃣ CACHE STORAGE
   ├─ Key: "search:village_name:state_2"
   ├─ Value: JSON results
   ├─ TTL: 60 seconds
   └─ Stored in Redis

9️⃣ ASYNC USAGE LOGGING
   ├─ Insert Usage record:
   │  ├─ userId: 2
   │  ├─ apiKeyId: 5
   │  ├─ endpoint: "/search"
   │  ├─ method: "GET"
   │  ├─ statusCode: 200
   │  ├─ responseTime: 47ms
   │  └─ date: 2026-05-05T10:30:00Z
   └─ Non-blocking (fire and forget)

🔟 FORMAT RESPONSE
   {
     "success": true,
     "count": 12,
     "data": [
       {
         "id": 525002,
         "name": "Manibeli",
         "code": "MB001",
         "fullAddress": "Manibeli, Akkalkuwa, Nandurbar, Maharashtra, India",
         "hierarchy": {
           "village": "Manibeli",
           "subDistrict": "Akkalkuwa",
           "district": "Nandurbar",
           "state": "Maharashtra",
           "country": "India"
         }
       },
       ...
     ],
     "meta": {
       "requestId": "req_xxxxx",
       "responseTime": 47,
       "rateLimit": {
         "remaining": 37550,
         "limit": 50000,
         "reset": "2026-05-06T00:00:00Z"
       }
     }
   }

1️⃣1️⃣ SEND RESPONSE
   ├─ Status: 200 OK
   ├─ Headers: Security headers + Rate limits
   └─ Body: JSON response

TOTAL RESPONSE TIME: 47ms ✅ (Target: <100ms)
```

---

## 🎨 Frontend Architecture

### Admin Dashboard

```
                App.jsx
                   │
        ┌──────────┼──────────┐
        │          │          │
     Layout    Navigation  Pages
        │          │          │
        │          │    ┌─────┴─────┐
        │          │    │           │
        │      Sidebar  Overview  Users
        │      Profile     │       Logs
        │      Theme       │
        │                  │
        ▼                  ▼
    Zustand Store      TanStack Query
    (UI State)         (Server State)
        │                  │
        ├─ currentPage     ├─ Overview data
        ├─ sidebarOpen     ├─ Users list
        ├─ darkMode        └─ Logs
        └─ notifications

    Components:
    ├─ KPI Cards
    ├─ Line Chart (7-day requests)
    ├─ Pie Chart (users by plan)
    ├─ Data Table (users/logs)
    └─ Modal Dialogs
```

### B2B Portal

```
                App.jsx
                   │
        ┌──────────┼──────────┐
        │          │          │
    Auth Pages  Dashboard  Settings
        │          │          │
     Login      API Keys     Profile
     Register   Usage Chart   Theme
               Stats Cards
                  │
                  ▼
    Zustand Auth Store (JWT)
        │
    localStorage ◄──── Persist
        │
    TanStack Query
        ├─ Fetch dashboard data
        ├─ API key mutations
        └─ Usage analytics
```

### Demo App

```
              App.jsx
                 │
            Layout
                 │
            Contact Form
                 │
        ┌────────┼────────┐
        │        │        │
     Input   Address   Actions
     Fields  Autocomplete Submit
        │        │        │
        │        ▼        │
        │   Debounce(300ms)
        │        │        │
        │   API Call      │
        │   /search       │
        │        │        │
        │   setState      │
        │        │        │
        └────────┼────────┘
                 │
            Form Submit
```

---

## 🚀 Deployment Architecture

### Development
```
Your Machine
├─ Backend: localhost:3000
├─ Admin: localhost:5174
├─ B2B: localhost:5173
├─ Demo: localhost:5175
├─ PostgreSQL: localhost:5432
└─ Redis: localhost:6379
```

### Production
```
Vercel Edge Network
├─ Backend serverless: api.bluestock.com
├─ Admin dashboard: admin.bluestock.com
├─ B2B portal: portal.bluestock.com
└─ Demo: demo.bluestock.com

Services
├─ PostgreSQL: NeonDB
├─ Redis: Upstash
├─ Email: SendGrid SMTP
├─ Monitoring: Sentry
└─ CI/CD: GitHub Actions
```

### Environments
```
Git Branches
├─ main → Production
│   └─ Triggers: Deploy to api.bluestock.com
├─ develop → Staging
│   └─ Triggers: Deploy to staging.bluestock.com
└─ feature/* → Preview
    └─ Triggers: PR preview deployments
```

---

## 📈 Scalability Considerations

### Caching Strategy
```
Frequently Accessed (24h TTL)
├─ All states (28 records)
├─ Admin overview stats
└─ User profile data

Occasionally Accessed (1 min TTL)
├─ Districts by state
├─ SubDistricts by district
└─ Search results

Never Cached
├─ Village details (varies by query)
├─ Usage logs
└─ Admin operations
```

### Database Optimization
```
Indexes Applied:
├─ name (VARCHAR) → Trigram index for fast text search
├─ Foreign keys (INT) → B-tree index for joins
└─ status (ENUM) → Hash index for filtering

Query Optimization:
├─ Pagination (limit, offset)
├─ Selected fields (not SELECT *)
├─ Batch operations (bulk inserts)
└─ Connection pooling (Prisma)
```

### Load Handling
```
Single Request: 47ms average
1,000 concurrent: No degradation (Redis cache hits)
10,000 daily: 500MB database queries
1M monthly: 15GB data transferred

Bottlenecks & Solutions:
├─ Database: Caching + Indexes + Pagination
├─ Network: Vercel edge + CDN
├─ Rate limits: Redis + User quotas
└─ Storage: Compression + Archiving logs
```

---

## 🔄 CI/CD Pipeline

```
GitHub Push
    │
    ├─ PR to develop
    │  └─ Trigger: GitHub Actions
    │     ├─ Run tests
    │     ├─ Lint code
    │     ├─ Check coverage
    │     └─ Preview deploy
    │
    └─ Merge to main
       └─ Trigger: GitHub Actions
          ├─ Run all tests
          ├─ Build apps
          ├─ Deploy to Vercel
          ├─ Run smoke tests
          └─ Notify team
```

---

## 📚 Key Design Decisions

| Decision | Rationale | Benefit |
|---|---|---|
| **Serverless (Vercel)** | No infrastructure management | Fast deployment, auto-scaling |
| **PostgreSQL** | ACID compliance, complex queries | Data integrity, 3NF possible |
| **Redis caching** | In-memory speed | Sub-100ms response times |
| **JWT + API Keys** | Industry standard | Secure, scalable auth |
| **Separate frontends** | Role-based UI | Optimized UX for each role |
| **Prisma ORM** | Type safety, migrations | Fewer bugs, easy versioning |
| **Zustand + React Query** | Lightweight state | No boilerplate, easy debugging |
| **Tailwind CSS** | Utility-first | Fast UI development |

---

## 🎯 Architecture Goals Met

✅ **Scalability**: Handles 1M+ daily requests  
✅ **Performance**: Sub-100ms response time (95th percentile)  
✅ **Security**: JWT + API keys + rate limiting  
✅ **Reliability**: Error tracking + monitoring  
✅ **Maintainability**: Clean code, proper documentation  
✅ **Cost-effective**: Serverless + managed services  

---

**Ready to build? This architecture supports your vision! 🚀**

