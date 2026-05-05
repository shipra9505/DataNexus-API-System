# BLUESTOCK Deployment Guide

**Complete guide to deploying BLUESTOCK to production on Vercel.**

---

## 📋 Table of Contents

1. [Overview](#-overview)
2. [Prerequisites](#-prerequisites)
3. [Environment Setup](#-environment-setup)
4. [Database & Services](#-database--services)
5. [Local Testing](#-local-testing)
6. [Vercel Deployment](#-vercel-deployment)
7. [CI/CD Pipeline](#-cicd-pipeline)
8. [Post-Deployment](#-post-deployment)
9. [Rollback](#-rollback)
10. [Monitoring](#-monitoring)
11. [Troubleshooting](#-troubleshooting)

---

## 🎯 Overview

### Deployment Architecture

```
GitHub Repository
       │
       └─→ GitHub Actions CI/CD
           ├─ Run Tests
           ├─ Build Frontends
           ├─ Security Scan
           └─ Deploy to Vercel
               │
               ├─ Backend (Serverless)
               ├─ Admin Dashboard
               ├─ B2B Portal
               └─ Demo Application
```

### Deployment Flow

| Environment | Branch | Trigger | URL |
|---|---|---|---|
| **Development** | local | Manual | localhost:3000 |
| **Staging** | develop | Auto on push | staging-api.bluestock.com |
| **Production** | main | Auto on push | api.bluestock.com |

---

## 📋 Prerequisites

### Required Services

- ✅ **GitHub Account** - Repository hosting
- ✅ **Vercel Account** - Serverless deployment
- ✅ **PostgreSQL Database** - NeonDB (free tier)
- ✅ **Redis Cache** - Upstash (free tier)
- ✅ **Email Service** - SendGrid/Gmail SMTP
- ✅ **Domain Name** (optional) - For custom domains

### Required Tools

```bash
# Install Vercel CLI
npm i -g vercel

# Install Git
git --version  # Should output git version

# Node.js 18.x+
node --version
npm --version
```

---

## 🔐 Environment Setup

### Step 1: Create Vercel Account

1. Go to [vercel.com](https://vercel.com)
2. Sign up with GitHub
3. Authorize Vercel to access your repos

### Step 2: Create NeonDB PostgreSQL Database

```bash
# Go to https://neon.tech
# 1. Sign up with GitHub
# 2. Create new project
# 3. Create new database
# 4. Copy connection string
# Format: postgresql://user:password@host.neon.tech/database
```

### Step 3: Create Upstash Redis Instance

```bash
# Go to https://upstash.com
# 1. Sign up
# 2. Create Redis database
# 3. Copy REST URL and token
# Format: https://user:token@host.upstash.io
```

### Step 4: Setup SendGrid Email Service

```bash
# Go to https://sendgrid.com
# 1. Create free account
# 2. Create API key
# 3. Add verified sender email
```

---

## 💾 Database & Services

### Create Production Database

```bash
# Connect to NeonDB
psql postgresql://user:password@host.neon.tech/bluestock_prod

# Or run migrations through Prisma
DATABASE_URL=postgresql://... npx prisma migrate deploy
```

### Initialize Database Schema

```bash
# Run Prisma migrations
cd bluestock_api/backend

# Create .env.production
cat > .env.production << 'EOF'
DATABASE_URL=postgresql://user:password@host.neon.tech/bluestock_prod
UPSTASH_REDIS_REST_URL=https://user:token@host.upstash.io
UPSTASH_REDIS_REST_TOKEN=token
EOF

# Apply migrations
ENVIRONMENT=production npx prisma migrate deploy

# Seed admin user
ENVIRONMENT=production npm run seed
```

### Create Admin User (First Time)

The admin user is automatically created via the `adminBootstrap.js` script on first API startup.

```env
# Set in Vercel environment variables:
ADMIN_EMAIL=admin@bluestock.com
ADMIN_PASSWORD=SecureAdmin123!
ADMIN_BUSINESS_NAME=Bluestock Admin
ADMIN_PHONE=+91-9999999999
ADMIN_GST=18AABCU1234K1Z5
```

---

## 🏠 Local Testing

### 1. Test Locally Before Deploying

```bash
# Setup local environment
cp .env.example .env.local

# Add production-like secrets
echo "DATABASE_URL=postgresql://..." >> .env.local
echo "UPSTASH_REDIS_REST_URL=..." >> .env.local

# Run all tests
npm test

# Run frontend builds
cd admin-dashboard && npm run build
cd b2b-portal && npm run build
cd demo-app && npm run build
```

### 2. Test API Locally

```bash
cd bluestock_api/backend
npm run dev

# In another terminal, test endpoints
curl http://localhost:3000/api/v1/states
```

### 3. Test Vercel Build Locally

```bash
# Install Vercel CLI
npm i -g vercel

# Build for production
vercel build

# Test production build locally
vercel start
```

---

## 🚀 Vercel Deployment

### Step 1: Connect Repository to Vercel

```bash
# Option 1: Through Dashboard
# 1. Go to vercel.com/dashboard
# 2. Click "New Project"
# 3. Import GitHub repository
# 4. Configure project settings

# Option 2: Through CLI
vercel

# Follow interactive prompts:
# ✓ Link to existing project or create new
# ✓ Configure build settings
# ✓ Add environment variables
```

### Step 2: Configure Environment Variables

**For Backend Project:**

1. Go to Vercel Dashboard → Project Settings → Environment Variables
2. Add variables for all environments:

```env
# Database
DATABASE_URL=postgresql://user:password@host.neon.tech/bluestock_prod

# Cache
UPSTASH_REDIS_REST_URL=https://user:token@host.upstash.io
UPSTASH_REDIS_REST_TOKEN=token

# Security
JWT_SECRET=your_super_secret_key_min_32_characters_long

# Email
SMTP_HOST=smtp.sendgrid.net
SMTP_PORT=587
SMTP_USER=apikey
SMTP_PASS=SG.xxxxxxxxxxxx
EMAIL_FROM=noreply@bluestock.com

# Frontend URLs (CORS)
FRONTEND_URL=https://portal.bluestock.com
ADMIN_URL=https://admin.bluestock.com
DEMO_URL=https://demo.bluestock.com

# Admin User
ADMIN_EMAIL=admin@bluestock.com
ADMIN_PASSWORD=SecureAdmin123!
ADMIN_BUSINESS_NAME=Bluestock Admin
ADMIN_PHONE=+91-9999999999
ADMIN_GST=18AABCU1234K1Z5

# Deployment
NODE_ENV=production
```

**For Frontend Projects:**

```env
VITE_API_URL=https://api.bluestock.com/api/v1
VITE_APP_NAME=BLUESTOCK
```

### Step 3: Configure Build Settings

**Backend vercel.json:**

Already configured in [bluestock_api/backend/vercel.json](../bluestock_api/backend/vercel.json)

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
    "UPSTASH_REDIS_REST_URL": "@UPSTASH_REDIS_REST_URL",
    "UPSTASH_REDIS_REST_TOKEN": "@UPSTASH_REDIS_REST_TOKEN"
  }
}
```

**Frontend vite.config.js:**

Already configured in each frontend folder

### Step 4: Deploy

```bash
# Deploy backend
cd bluestock_api/backend
vercel deploy --prod

# Deploy admin dashboard
cd admin-dashboard
vercel deploy --prod

# Deploy B2B portal
cd b2b-portal
vercel deploy --prod

# Deploy demo app
cd demo-app
vercel deploy --prod
```

---

## 🔄 CI/CD Pipeline

### GitHub Actions Workflows

Already configured in [.github/workflows/](.github/workflows/)

#### 1. **test.yml** - Runs on every PR

```yaml
- Lint code
- Run tests
- Build frontends
- Security scan
```

#### 2. **deploy-staging.yml** - Runs on push to `develop`

```yaml
- Run all tests
- Build frontends
- Deploy to staging on Vercel
- Run smoke tests
- Notify team
```

#### 3. **deploy-prod.yml** - Runs on push to `main`

```yaml
- Run tests + coverage
- Security scan
- Build frontends
- Deploy to production
- Run smoke tests
- Create GitHub release
- Notify team
```

### Setup GitHub Actions Secrets

```bash
# Go to GitHub Repo Settings → Secrets and variables

# Add these secrets:
VERCEL_TOKEN=your_vercel_token
VERCEL_ORG_ID=your_org_id
VERCEL_PROJECT_ID_BACKEND_PROD=backend_project_id
VERCEL_PROJECT_ID_ADMIN_PROD=admin_project_id
VERCEL_PROJECT_ID_B2B_PROD=b2b_project_id
VERCEL_PROJECT_ID_DEMO_PROD=demo_project_id
VERCEL_PROJECT_ID_BACKEND_STAGING=staging_backend_id
VERCEL_PROJECT_ID_ADMIN_STAGING=staging_admin_id
VERCEL_PROJECT_ID_B2B_STAGING=staging_b2b_id
VERCEL_PROJECT_ID_DEMO_STAGING=staging_demo_id

SLACK_WEBHOOK=your_slack_webhook_url
SNYK_TOKEN=your_snyk_token
SONAR_TOKEN=your_sonarcloud_token
```

---

## 📝 Post-Deployment

### 1. Verify Deployment

```bash
# Check if APIs are responding
curl https://api.bluestock.com/api/v1/states

# Check if frontends are accessible
curl https://admin.bluestock.com
curl https://portal.bluestock.com
curl https://demo.bluestock.com
```

### 2. Test Critical Workflows

**Admin Dashboard:**
- [ ] Login with admin credentials
- [ ] View overview stats
- [ ] Check user list

**B2B Portal:**
- [ ] Register new user
- [ ] Wait for admin approval
- [ ] Login and view API keys
- [ ] Generate new API key

**Demo App:**
- [ ] Search for village
- [ ] Verify autocomplete works
- [ ] Submit contact form

### 3. Check Logs

```bash
# View Vercel deployment logs
vercel logs <url>

# Example:
vercel logs api.bluestock.com
```

### 4. Monitor Performance

```bash
# Check API response times
curl -w "@curl-format.txt" https://api.bluestock.com/api/v1/states

# Check error rate in Sentry
# Go to https://sentry.io/organizations/your-org/
```

### 5. Update DNS (if using custom domain)

```bash
# Add Vercel nameservers to your domain registrar
# Or add CNAME record:
# admin.bluestock.com → admin-bluestock.vercel.app
# portal.bluestock.com → portal-bluestock.vercel.app
# api.bluestock.com → api-bluestock.vercel.app
# demo.bluestock.com → demo-bluestock.vercel.app
```

---

## ↩️ Rollback

### If Deployment Fails

```bash
# Option 1: Revert Git commit
git revert HEAD
git push origin main

# GitHub Actions will auto-deploy previous version

# Option 2: Manual rollback in Vercel
# 1. Go to Vercel Dashboard
# 2. Select project
# 3. Go to "Deployments"
# 4. Find previous successful deployment
# 5. Click "Promote to Production"
```

### If Database Migration Fails

```bash
# Rollback Prisma migration
cd bluestock_api/backend

# List migrations
npx prisma migrate status

# Rollback to previous migration
npx prisma migrate resolve --rolled-back 20260505100859_migration_name
```

---

## 📊 Monitoring

### Setup Error Tracking (Sentry)

```bash
# 1. Go to https://sentry.io
# 2. Create new project (Node.js)
# 3. Copy DSN

# 2. Add to backend .env
SENTRY_DSN=https://xxx@xxx.ingest.sentry.io/xxx

# 3. Initialize in backend
import * as Sentry from "@sentry/node";

Sentry.init({
  dsn: process.env.SENTRY_DSN,
  environment: process.env.NODE_ENV,
  tracesSampleRate: 1.0,
});
```

### Setup Performance Monitoring

```bash
# Use Vercel Analytics
# Automatically enabled on Vercel

# Check metrics at:
# https://vercel.com/dashboard/your-project/analytics
```

### Setup Uptime Monitoring

```bash
# Use free service like UptimeRobot
# 1. Go to https://uptimerobot.com
# 2. Create monitor for API endpoint
# 3. Get alerts if service goes down

# Monitor URLs:
# https://api.bluestock.com/api/v1/states
# https://admin.bluestock.com
# https://portal.bluestock.com
```

### Setup Logging

```bash
# Vercel provides logs automatically
# Access via:
# 1. Vercel Dashboard → Deployments
# 2. Click deployment → Logs tab

# Or via CLI:
vercel logs api.bluestock.com --follow
```

---

## 🐛 Troubleshooting

### Issue: "Build failed on Vercel"

```bash
# Check build output
vercel logs api.bluestock.com --failed

# Common fixes:
# 1. Verify all environment variables are set
# 2. Check Node.js version compatibility
# 3. Ensure Prisma is installed: npm install prisma

# Test locally first
vercel build
```

### Issue: "Database connection timeout"

```bash
# 1. Check DATABASE_URL is correct
echo $DATABASE_URL

# 2. Verify NeonDB is running and accessible
psql $DATABASE_URL -c "SELECT 1"

# 3. Check firewall/IP whitelist settings in NeonDB
```

### Issue: "Redis connection refused"

```bash
# 1. Verify UPSTASH_REDIS_REST_URL is correct
# 2. Test connection:
curl -X GET $UPSTASH_REDIS_REST_URL/ping

# 3. Check Upstash dashboard for service status
```

### Issue: "API returns 500 error"

```bash
# 1. Check Sentry for error details
# https://sentry.io/organizations/your-org/

# 2. Check Vercel logs
vercel logs api.bluestock.com --follow

# 3. Test locally to reproduce
npm run dev

# 4. Roll back to previous version if critical
```

### Issue: "CORS error in frontend"

```bash
# 1. Verify CORS origin is correct in backend
# Check .env: FRONTEND_URL, ADMIN_URL, DEMO_URL

# 2. Verify frontend environment variables
# Check .env.local: VITE_API_URL

# 3. Test API directly with curl
curl -H "Origin: https://admin.bluestock.com" https://api.bluestock.com/api/v1/states
```

---

## 📊 Performance Checklist

Before considering deployment complete:

- [ ] API response time < 100ms (p95)
- [ ] Frontend load time < 2s
- [ ] No console errors in browser
- [ ] All API endpoints responding with 200
- [ ] Rate limiting working correctly
- [ ] Database queries optimized
- [ ] Cache hit rate > 80%
- [ ] Error rate < 0.1%
- [ ] Security headers present
- [ ] SSL certificate valid

---

## 🔐 Security Checklist

- [ ] All secrets in Vercel, not in code
- [ ] Database password rotated
- [ ] JWT secret changed from default
- [ ] HTTPS enforced
- [ ] CORS origins restricted
- [ ] Security headers configured
- [ ] Rate limiting enabled
- [ ] API key authentication working
- [ ] Admin account password strong
- [ ] 2FA enabled for admin account

---

## 📞 Deployment Support

### Get Help

1. **Vercel Docs**: https://vercel.com/docs
2. **Prisma Docs**: https://prisma.io/docs
3. **GitHub Actions**: https://docs.github.com/en/actions
4. **NeonDB Support**: https://neon.tech/docs

### Common Commands

```bash
# Deploy specific project
vercel deploy --prod

# View logs
vercel logs <url>

# List deployments
vercel ls

# Remove local Vercel config
rm -rf .vercel/

# Verify environment
vercel env list
```

---

## ✅ Deployment Checklist

**Before Pushing to main:**

- [ ] All tests pass: `npm test`
- [ ] Frontend builds: `npm run build`
- [ ] No console errors
- [ ] Environment variables documented
- [ ] Database migrations tested
- [ ] API endpoints tested
- [ ] Performance acceptable
- [ ] Security reviewed

**After Deployment:**

- [ ] API is responding
- [ ] Admin can login
- [ ] B2B user can register
- [ ] Demo app works
- [ ] Logs are clean
- [ ] No errors in Sentry
- [ ] Performance metrics good

---

**Ready to deploy? Happy shipping! 🚀**

