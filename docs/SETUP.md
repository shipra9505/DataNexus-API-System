# BLUESTOCK B2B API - Local Development Setup

**Complete guide to set up the BLUESTOCK platform on your local machine.**

---

## 📋 Prerequisites

### System Requirements
- **OS**: Windows, macOS, or Linux
- **Memory**: 4GB RAM minimum (8GB recommended)
- **Disk Space**: 2GB free

### Required Software
- **Node.js**: 18.x or 20.x ([download](https://nodejs.org/))
- **npm**: 9.x or higher (comes with Node.js)
- **PostgreSQL**: 14.x or higher ([download](https://www.postgresql.org/download/))
- **Redis**: 6.x or higher ([download](https://redis.io/download))
- **Git**: Latest version ([download](https://git-scm.com/))
- **Python**: 3.8+ (for data import scripts)

### Verify Installation
```bash
node --version      # Should be v18.x or higher
npm --version       # Should be 9.x or higher
psql --version      # Should be 14.x or higher
redis-server --version  # Should be 6.x or higher
python --version    # Should be 3.8 or higher
```

---

## 🔧 Environment Variables Setup

### 1. Backend Environment (.env)

Navigate to `bluestock_api/backend/` and create a `.env` file:

```bash
cd bluestock_api/backend
```

Create `.env` file:
```env
# Database
DATABASE_URL=postgresql://user:password@localhost:5432/bluestock_dev

# Redis
UPSTASH_REDIS_REST_URL=http://localhost:6379
UPSTASH_REDIS_REST_TOKEN=redis_token

# JWT Secret
JWT_SECRET=your_super_secret_jwt_key_min_32_characters_long_12345

# Frontend URLs (CORS)
FRONTEND_URL=http://localhost:5173
ADMIN_URL=http://localhost:5174
DEMO_URL=http://localhost:5175

# Email Configuration (SMTP)
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your_email@gmail.com
SMTP_PASS=your_app_password
EMAIL_FROM=noreply@bluestock.com

# Admin Credentials
ADMIN_EMAIL=admin@bluestock.com
ADMIN_PASSWORD=SecureAdmin123!
ADMIN_BUSINESS_NAME=Bluestock Admin
ADMIN_PHONE=+91-9999999999
ADMIN_GST=18AABCT1234K1Z5

# Optional
DEBUG=true
NODE_ENV=development
```

### 2. Admin Dashboard Environment (.env.local)

Navigate to `admin-dashboard/` and create `.env.local`:

```bash
cd admin-dashboard
```

Create `.env.local`:
```env
VITE_API_URL=http://localhost:3000/api/v1
VITE_API_KEY=admin_key_here
```

### 3. B2B Portal Environment (.env.local)

Navigate to `b2b-portal/` and create `.env.local`:

```bash
cd b2b-portal
```

Create `.env.local`:
```env
VITE_API_URL=http://localhost:3000/api/v1
VITE_APP_NAME=BLUESTOCK B2B Portal
```

### 4. Demo App Environment (.env.local)

Navigate to `demo-app/` and create `.env.local`:

```bash
cd demo-app
```

Create `.env.local`:
```env
VITE_API_URL=http://localhost:3000/api/v1
VITE_API_KEY=demo_public_key
```

---

## 🗄️ Database Setup

### 1. Create PostgreSQL Database

```bash
# Start PostgreSQL service
# On macOS (with Homebrew)
brew services start postgresql

# On Windows
# Open PostgreSQL pgAdmin or use command line
```

### 2. Create Database and User

```bash
# Connect to PostgreSQL
psql -U postgres

# Inside psql:
CREATE DATABASE bluestock_dev;
CREATE USER bluestock WITH PASSWORD 'bluestock_password';
ALTER ROLE bluestock WITH CREATEDB;
GRANT ALL PRIVILEGES ON DATABASE bluestock_dev TO bluestock;
\q
```

### 3. Update DATABASE_URL in .env

```env
DATABASE_URL=postgresql://bluestock:bluestock_password@localhost:5432/bluestock_dev
```

### 4. Run Prisma Migrations

```bash
cd bluestock_api/backend

# Generate Prisma client
npx prisma generate

# Run migrations
npx prisma migrate dev --name init

# Seed database (optional)
npx prisma db seed
```

---

## 💾 Redis Setup

### Option 1: Local Redis Server

```bash
# Start Redis on default port 6379
redis-server

# Verify connection
redis-cli ping  # Should return: PONG
```

### Option 2: Using Docker

```bash
# Pull and run Redis image
docker run -d -p 6379:6379 redis:latest

# Verify
redis-cli ping
```

### Update .env

```env
UPSTASH_REDIS_REST_URL=http://localhost:6379
UPSTASH_REDIS_REST_TOKEN=optional_token_if_auth_required
```

---

## 📦 Install Dependencies

### Backend

```bash
cd bluestock_api/backend
npm install
```

**Key packages installed:**
- `express` - REST API framework
- `prisma` - ORM
- `redis` - Caching
- `jsonwebtoken` - Authentication
- `bcrypt` - Password hashing
- `express-validator` - Input validation
- `jest` + `supertest` - Testing

### Admin Dashboard

```bash
cd admin-dashboard
npm install
```

**Key packages:**
- `react` 19
- `vite` - Build tool
- `zustand` - State management
- `react-query` - Data fetching
- `recharts` - Charts
- `tailwind` - Styling

### B2B Portal

```bash
cd b2b-portal
npm install
```

**Same as admin dashboard**

### Demo App

```bash
cd demo-app
npm install
```

**Same as other frontends**

---

## 🚀 Running the Application

### Terminal 1: Start Backend API

```bash
cd bluestock_api/backend
npm run dev
```

**Expected output:**
```
Server running on port 3000
Database connected: bluestock_dev
Redis connected: localhost:6379
```

**Access:**
- API: `http://localhost:3000/api/v1`
- Swagger Docs: `http://localhost:3000/api-docs`

### Terminal 2: Start Admin Dashboard

```bash
cd admin-dashboard
npm run dev
```

**Expected output:**
```
  VITE v5.0.0  ready in 200 ms
  ➜  Local:   http://localhost:5174/
  ➜  press h to show help
```

**Access:** `http://localhost:5174`

### Terminal 3: Start B2B Portal

```bash
cd b2b-portal
npm run dev
```

**Expected output:**
```
  VITE v5.0.0  ready in 200 ms
  ➜  Local:   http://localhost:5173/
```

**Access:** `http://localhost:5173`

### Terminal 4: Start Demo App

```bash
cd demo-app
npm run dev
```

**Expected output:**
```
  VITE v5.0.0  ready in 200 ms
  ➜  Local:   http://localhost:5175/
```

**Access:** `http://localhost:5175`

---

## 🔑 Test Credentials

### Admin Login
```
Email: admin@bluestock.com
Password: SecureAdmin123!
URL: http://localhost:5174
```

### B2B User (Self-Register)
1. Go to `http://localhost:5173`
2. Click "Register"
3. Fill in details:
   - Email: `test@company.com`
   - Business Name: `Test Company`
   - GST: `18AABCU1234A1Z0`
   - Phone: `+91-9999999999`
   - Password: `Password123!`
4. Wait for admin approval
5. Login to see dashboard

### Demo App
- Go to `http://localhost:5175`
- No login required
- Fill the contact form
- Search for a village (e.g., "Manibeli")
- Auto-fill address and submit

---

## 📊 Data Import (Optional)

### Import Village Data

```bash
cd bluestock_api/python_scripts

# Install Python dependencies
pip install pandas psycopg2-binary openpyxl

# Run import script
python import_villages.py

# Expected output:
# Processing: States...
# Processing: Districts...
# Processing: Villages (batch 1-5000)...
# Verification: Total villages: 640,867
```

---

## 🧪 Running Tests

### Backend Tests

```bash
cd bluestock_api/backend

# Run all tests
npm test

# Run specific test file
npm test auth.test.js

# Run with coverage
npm test -- --coverage

# Expected output:
# PASS  src/__tests__/auth.test.js
# PASS  src/__tests__/search.test.js
# Tests: 25 passed, 25 total
```

---

## 🔍 Verify Setup

### Checklist

- [ ] **PostgreSQL running**: `psql -U bluestock -d bluestock_dev`
- [ ] **Redis running**: `redis-cli ping` → PONG
- [ ] **Backend started**: `http://localhost:3000/api-docs` loads
- [ ] **Admin dashboard**: `http://localhost:5174` loads
- [ ] **B2B portal**: `http://localhost:5173` loads
- [ ] **Demo app**: `http://localhost:5175` loads
- [ ] **Database has data**: `psql -U bluestock -d bluestock_dev -c "SELECT COUNT(*) FROM Village;"`

### Test API Connection

```bash
# From any terminal
curl -X GET http://localhost:3000/api/v1/states

# Expected response:
# {
#   "success": true,
#   "count": 28,
#   "data": [...]
# }
```

---

## 🐛 Troubleshooting

### Issue: "Port 3000 already in use"
```bash
# Kill the process using port 3000
# On macOS/Linux:
lsof -i :3000 | grep LISTEN | awk '{print $2}' | xargs kill -9

# On Windows:
netstat -ano | findstr :3000
taskkill /PID <PID> /F
```

### Issue: "Database connection failed"
```bash
# Check PostgreSQL is running
psql -U bluestock -d bluestock_dev

# If error, restart PostgreSQL:
brew services restart postgresql  # macOS
# Or manually start on Windows
```

### Issue: "Redis connection refused"
```bash
# Start Redis server
redis-server

# Or use Docker
docker run -d -p 6379:6379 redis:latest
```

### Issue: ".env file not found"
```bash
# Make sure you created .env in correct folder
# Backend: bluestock_api/backend/.env
# Frontends: */src/.env.local
```

### Issue: "VITE module not found"
```bash
# Clear node_modules and reinstall
rm -rf node_modules package-lock.json
npm install
npm run dev
```

---

## 📚 Common Commands

```bash
# Backend development
npm run dev        # Start with hot reload
npm test           # Run tests
npm run lint       # Check code style

# Frontend development
npm run dev        # Start Vite dev server
npm run build      # Build for production
npm run preview    # Preview production build

# Database
npx prisma studio # Open Prisma Studio (visual DB explorer)
npx prisma db seed # Seed database
```

---

## 🎯 Next Steps

After setup, you should:

1. ✅ Login to admin dashboard
2. ✅ Register a B2B user and get API key
3. ✅ Test API endpoints with Postman/curl
4. ✅ Try demo app autocomplete
5. ✅ Run test suite

---

## 📞 Support

If you encounter issues:

1. Check this setup guide
2. Review backend logs: `bluestock_api/backend/README.md`
3. Check database: `npx prisma studio`
4. Review error messages in terminal

---

## 💡 Tips

- **Hot Reload**: Both backend and frontends support hot reload during development
- **Prisma Studio**: Visual database explorer - `npx prisma studio`
- **Network Requests**: Use browser DevTools to debug API calls
- **Redux DevTools**: Install browser extension for Zustand debugging
- **API Testing**: Use Swagger at `http://localhost:3000/api-docs`

---

**Ready to develop? Happy coding! 🚀**

