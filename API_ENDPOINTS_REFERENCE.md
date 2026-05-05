# Bluestock API - Complete Endpoints Reference

## Authentication & User Management (`/api/v1/auth`)

### Account Registration & Login
- [ ] `POST /auth/register` - Register new B2B customer
- [ ] `POST /auth/verify-email` - Verify email with token
- [ ] `POST /auth/resend-verification` - Resend verification email
- [ ] `POST /auth/login` - Authenticate user (email + password)
- [ ] `GET /auth/me` - Get current user profile (JWT)

### Two-Factor Authentication
- [ ] `POST /auth/2fa/setup` - Initialize 2FA setup (returns QR code)
- [ ] `POST /auth/2fa/verify` - Confirm 2FA setup with TOTP token
- [ ] `POST /auth/2fa/disable` - Disable 2FA with TOTP token

### API Key Management
- [ ] `GET /auth/apikeys` - List user's API keys (max 5)
- [ ] `POST /auth/apikeys` - Generate new API key
- [ ] `PATCH /auth/apikeys/:id/revoke` - Revoke an API key
- [ ] `PATCH /auth/apikeys/:id/rotate` - Rotate API key (revoke + create new)

---

## Admin Dashboard (`/api/v1/admin`)
*All endpoints require: JWT Bearer token + Admin role + 2FA verification*

### Dashboard & Monitoring
- [ ] `GET /admin/overview` - Dashboard statistics (users, villages, requests, plans)
- [ ] `GET /admin/logs` - Query usage logs (endpoint, status, user, date filters)

### User Management
- [ ] `GET /admin/users` - List all users with full details
- [ ] `PATCH /admin/users/:id/approve` - Approve user account (PENDING_APPROVAL → ACTIVE)
- [ ] `PATCH /admin/users/:id/reject` - Reject user account (PENDING_APPROVAL → REJECTED)
- [ ] `PATCH /admin/users/:id/plan` - Update user plan + daily limit
- [ ] `PATCH /admin/users/:id/limit` - Override daily request limit
- [ ] `PATCH /admin/users/:id/access` - Set state-level access (array of state IDs)

### Reference Data
- [ ] `GET /admin/states` - List all states for admin UI

---

## Usage Analytics (`/api/v1/usage`)
*All endpoints require: JWT Bearer token*

- [ ] `GET /usage/dashboard` - User's usage dashboard (7-day breakdown, success rate, quota)
- [ ] `GET /usage/logs` - User's own usage logs with filters (endpoint, status, pagination)

---

## Data Lookup Routes
*All endpoints require: X-API-Key header + Rate Limiting applied*

### States
- [ ] `GET /states` - List all states (cached 24h)
- [ ] `GET /states/:id` - Get single state by ID

### Districts
- [ ] `GET /districts?stateId={id}&page={n}&limit={n}` - List districts with pagination (1min cache)
- [ ] `GET /districts/:id` - Get single district by ID

### SubDistricts
- [ ] `GET /subdistricts?districtId={id}` - List subdistricts filtered by district
- [ ] `GET /subdistricts/:id` - Get single subdistrict by ID

### Villages
- [ ] `GET /villages?subDistrictId={id}&page={n}&limit={n}` - List villages (paginated)
- [ ] `GET /villages/:id` - Get single village by ID

### Search
- [ ] `GET /search?q={query}&stateId={id}&districtId={id}&limit={n}` - Full-text search villages
- [ ] `GET /search/autocomplete?q={query}` - Autocomplete suggestions

---

## Health Check

- [ ] `GET /` - Health check endpoint

---

## Documentation

- [ ] `GET /api-docs` - Swagger UI
- [ ] `GET /openapi.json` - OpenAPI specification JSON

---

## Authentication Headers Summary

### JWT Bearer Token
```
Authorization: Bearer <JWT_token>
```
Used by: `/auth/me`, `/usage/*`, `/admin/*`

### API Key
```
X-API-Key: ak_<32-hex-characters>
```
Used by: `/states`, `/districts`, `/subdistricts`, `/villages`, `/search`

### 2FA Code (Admin only)
```
X-2FA-Code: <6-digit-TOTP>
```
OR in request body: `{ "twoFactorCode": "123456" }`
Used by: `/admin/*` endpoints

---

## Status Codes

- **200** - Success
- **201** - Created (e.g., new user, API key)
- **400** - Bad request (validation error)
- **401** - Unauthorized (missing/invalid auth)
- **403** - Forbidden (rate limit exceeded, no admin access, 2FA required)
- **404** - Not found
- **409** - Conflict (email already registered)
- **500** - Server error

---

## Common Query Parameters

| Parameter | Type | Example | Used In |
|-----------|------|---------|---------|
| `page` | int | `page=1` | districts, villages, admin/logs, usage/logs |
| `limit` | int | `limit=20` | districts, villages, search, admin/logs, usage/logs |
| `stateId` | int | `stateId=5` | districts, search |
| `districtId` | int | `districtId=12` | subdistricts, search |
| `subDistrictId` | int | `subDistrictId=45` | villages |
| `q` | string | `q=manibeli` | search, search/autocomplete |
| `endpoint` | string | `endpoint=/api/v1/search` | admin/logs |
| `status` | int | `status=200` | admin/logs |
| `userEmail` | string | `userEmail=user@example.com` | admin/logs |
| `fromDate` | ISO date | `fromDate=2025-01-01T00:00:00Z` | admin/logs |
| `toDate` | ISO date | `toDate=2025-01-31T23:59:59Z` | admin/logs |

---

## Response Format

### Success Response
```json
{
  "success": true,
  "data": { /* payload */ },
  "count": 10
}
```

### Error Response
```json
{
  "success": false,
  "error": "Error message"
}
```

### Paginated Response
```json
{
  "success": true,
  "count": 20,
  "total": 1000,
  "page": 1,
  "limit": 20,
  "totalPages": 50,
  "data": [ /* items */ ]
}
```

---

## Rate Limit Response Headers

```
X-RateLimit-Limit: 5000
X-RateLimit-Remaining: 4950
X-RateLimit-Reset: 1704067200
```

---

## Middleware Stack

1. **CORS** - Allows: localhost:5173, :5174, :5175
2. **Body Parser** - JSON only
3. **Request Logger** - Logs all authenticated requests
4. **Route-specific Auth**:
   - Auth routes: No protection
   - Admin routes: JWT + Admin role + 2FA
   - Usage routes: JWT only
   - Data routes: API Key + Rate Limiting

---

## Error Handling Examples

### Invalid API Key
```json
{
  "success": false,
  "error": "Invalid API key format"
}
```

### Rate Limit Exceeded
```json
{
  "success": false,
  "error": "Daily rate limit exceeded"
}
```

### 2FA Required (Admin)
```json
{
  "success": false,
  "error": "2FA code required for admin actions"
}
```

### Validation Error
```json
{
  "success": false,
  "error": "Valid email is required"
}
```

---

## Useful Links

- Swagger UI: http://localhost:3000/api-docs
- OpenAPI: http://localhost:3000/openapi.json
- Root: http://localhost:3000
- Auth: http://localhost:3000/api/v1/auth
- Admin: http://localhost:3000/api/v1/admin (admin only)
- Usage: http://localhost:3000/api/v1/usage (authenticated)
- Data: http://localhost:3000/api/v1/{states|districts|subdistricts|villages|search}

