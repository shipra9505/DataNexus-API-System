# BLUESTOCK - Contributing Guidelines

**How to contribute code to the BLUESTOCK project.**

---

## 📋 Table of Contents

1. [Getting Started](#-getting-started)
2. [Development Setup](#-development-setup)
3. [Code Standards](#-code-standards)
4. [Git Workflow](#-git-workflow)
5. [Commit Messages](#-commit-messages)
6. [Pull Requests](#-pull-requests)
7. [Code Review Process](#-code-review-process)
8. [Testing](#-testing)
9. [Database Changes](#-database-changes)
10. [Documentation](#-documentation)

---

## 🚀 Getting Started

### Prerequisites
- Node.js 18.x+
- PostgreSQL 14.x+
- Redis 6.x+
- Git

### Join the Project
1. Fork the repository
2. Clone your fork: `git clone https://github.com/YOUR_USERNAME/bluestock.git`
3. Add upstream: `git remote add upstream https://github.com/ORIGINAL_OWNER/bluestock.git`
4. Follow [Development Setup](../docs/SETUP.md)

---

## 🛠️ Development Setup

### 1. Install Dependencies

```bash
# Backend
cd bluestock_api/backend
npm install

# Admin Dashboard
cd admin-dashboard
npm install

# B2B Portal
cd b2b-portal
npm install

# Demo App
cd demo-app
npm install
```

### 2. Start Development Servers

```bash
# Terminal 1: Backend (port 3000)
cd bluestock_api/backend
npm run dev

# Terminal 2: Admin Dashboard (port 5174)
cd admin-dashboard
npm run dev

# Terminal 3: B2B Portal (port 5173)
cd b2b-portal
npm run dev

# Terminal 4: Demo App (port 5175)
cd demo-app
npm run dev
```

### 3. Run Tests

```bash
cd bluestock_api/backend
npm test
```

---

## 📝 Code Standards

### JavaScript/TypeScript Style

#### 1. **Naming Conventions**

```javascript
// ✅ GOOD
const getUserById = (id) => { }
const isActive = true
const MAX_RETRY_COUNT = 3
class UserController { }
function validateEmail(email) { }

// ❌ BAD
const get_user_by_id = (id) => { }
const is_active = true
const max_retry_count = 3
class userController { }
function ValidateEmail(email) { }
```

#### 2. **File Organization**

```
bluestock_api/backend/src/
├── routes/              # Route definitions
│   ├── auth.js
│   ├── admin.js
│   └── search.js
├── middleware/          # Middleware functions
│   ├── auth.js
│   ├── rateLimit.js
│   └── logger.js
├── lib/                 # Utilities & helpers
│   ├── prisma.js       # DB connection
│   ├── redis.js        # Cache
│   └── email.js        # Email service
├── validators/          # Input validation
│   ├── search.validator.js
│   └── user.validator.js
├── utils/              # Common utilities
│   └── errorHandler.js
├── __tests__/          # Tests
│   ├── auth.test.js
│   └── setup.js
├── app.js             # Express app setup
└── index.js           # Server entry point
```

#### 3. **Function Style**

```javascript
// ✅ GOOD: Arrow functions, clear parameters
const getUserData = async (userId) => {
  try {
    const user = await prisma.user.findUnique({
      where: { id: userId }
    });
    return user;
  } catch (error) {
    throw new AppError('User not found', 404);
  }
};

// ❌ BAD: Unclear parameters, no error handling
const getUserData = async (u) => {
  return await prisma.user.findUnique({ where: { id: u } });
};
```

#### 4. **Error Handling**

```javascript
// ✅ GOOD: Use custom AppError class
const validateEmail = (email) => {
  if (!email || !email.includes('@')) {
    throw new AppError('Invalid email format', 400);
  }
};

// ❌ BAD: Generic Error
const validateEmail = (email) => {
  if (!email || !email.includes('@')) {
    throw new Error('Invalid email');
  }
};
```

#### 5. **Comments & Documentation**

```javascript
// ✅ GOOD: Clear, concise comments
// Calculate user's daily request usage from Redis
// Returns: { used: 1250, limit: 5000, remaining: 3750 }
const getUsageStats = async (userId) => {
  const key = `user:${userId}:usage:${new Date().toISOString().split('T')[0]}`;
  const used = await redis.get(key) || 0;
  return { used, limit: user.dailyLimit, remaining: user.dailyLimit - used };
};

// ❌ BAD: Unclear comments
// get the usage
const getUsageStats = async (userId) => {
  const key = `user:${userId}:usage:${new Date().toISOString().split('T')[0]}`;
  return await redis.get(key);
};
```

### React Component Style

#### 1. **Component Structure**

```javascript
// ✅ GOOD: Clear structure with hooks at top
import React, { useState, useEffect } from 'react';
import { useQuery } from '@tanstack/react-query';

const UserDashboard = ({ userId }) => {
  // Hooks first
  const [isLoading, setIsLoading] = useState(false);
  const { data: user } = useQuery(['user', userId], () => fetchUser(userId));
  
  // Effects
  useEffect(() => {
    console.log('User loaded:', user);
  }, [user]);
  
  // Event handlers
  const handleLogout = () => {
    // logout logic
  };
  
  // Render
  if (isLoading) return <div>Loading...</div>;
  
  return (
    <div className="p-4">
      <h1>{user?.name}</h1>
      <button onClick={handleLogout}>Logout</button>
    </div>
  );
};

export default UserDashboard;

// ❌ BAD: Mixed logic and JSX, no structure
const UserDashboard = ({ userId }) => {
  return (
    <div>
      <h1>User Dashboard</h1>
      {/* logic scattered in JSX */}
    </div>
  );
};
```

#### 2. **Props & PropTypes**

```javascript
// ✅ GOOD: Define types (TypeScript)
interface UserCardProps {
  userId: number;
  onSelect: (id: number) => void;
  isActive?: boolean;
}

const UserCard: React.FC<UserCardProps> = ({ userId, onSelect, isActive = false }) => {
  return (
    <div onClick={() => onSelect(userId)}>
      {/* component */}
    </div>
  );
};

// ❌ BAD: No type definition
const UserCard = ({ userId, onSelect, isActive }) => {
  return <div onClick={() => onSelect(userId)}>{/* ... */}</div>;
};
```

#### 3. **Styling**

```javascript
// ✅ GOOD: Tailwind CSS classes
<button className="px-4 py-2 bg-blue-500 text-white rounded hover:bg-blue-600">
  Submit
</button>

// ✅ GOOD: Conditional classes
<div className={`p-4 ${isActive ? 'bg-green-100' : 'bg-gray-100'}`}>
  Content
</div>

// ❌ BAD: Inline styles (performance issue)
<button style={{ padding: '8px 16px', backgroundColor: 'blue' }}>
  Submit
</button>

// ❌ BAD: Magic values
<div style={{ marginTop: '24px' }}>Content</div>
```

### Database/Prisma Style

#### 1. **Schema Conventions**

```prisma
// ✅ GOOD: Descriptive names, proper relationships
model User {
  id            Int       @id @default(autoincrement())
  email         String    @unique
  businessName  String
  passwordHash  String    // Explain why hashed
  status        UserStatus @default(PENDING_APPROVAL)
  createdAt     DateTime  @default(now())
  updatedAt     DateTime  @updatedAt
  apiKeys       ApiKey[]  // 1:N relationship
  
  @@index([email]) // Index on frequently queried field
  @@map("users")   // Database table name
}

// ❌ BAD: Vague names, no relationships
model User {
  id    Int     @id
  name  String
  email String
}
```

#### 2. **Query Best Practices**

```javascript
// ✅ GOOD: Select specific fields
const user = await prisma.user.findUnique({
  where: { id: userId },
  select: {
    id: true,
    email: true,
    businessName: true,
    // Exclude passwordHash for security
  }
});

// ❌ BAD: Select all fields (security risk)
const user = await prisma.user.findUnique({
  where: { id: userId }
});

// ✅ GOOD: Include relations
const villages = await prisma.village.findMany({
  where: { subDistrictId: districtId },
  include: {
    subDistrict: {
      include: { district: { include: { state: true } } }
    }
  },
  take: 20,
  skip: (page - 1) * 20
});

// ❌ BAD: Multiple queries (N+1 problem)
const villages = await prisma.village.findMany({ where: { subDistrictId } });
for (const v of villages) {
  v.district = await prisma.district.findUnique({ where: { id: v.districtId } });
}
```

---

## 🌳 Git Workflow

### 1. **Branch Naming**

Use descriptive branch names following this pattern:

```
feature/description
fix/description
docs/description
refactor/description
test/description

Examples:
feature/add-csv-export           # New feature
fix/rate-limiting-bug            # Bug fix
docs/add-architecture-guide      # Documentation
refactor/simplify-auth-logic     # Code improvement
test/add-e2e-tests               # Testing
```

### 2. **Create a Feature Branch**

```bash
# Update main branch
git fetch upstream
git rebase upstream/main

# Create feature branch
git checkout -b feature/your-feature-name

# Push to your fork
git push origin feature/your-feature-name
```

### 3. **Keep Your Branch Updated**

```bash
# While developing, keep your branch updated
git fetch upstream
git rebase upstream/main

# If conflicts occur, resolve them and continue
git rebase --continue
```

---

## 💬 Commit Messages

### Format

```
<type>(<scope>): <subject>

<body>

<footer>
```

### Examples

```
✅ GOOD:
feat(auth): add 2FA verification endpoint
- Add POST /auth/2fa/verify endpoint
- Implement TOTP validation
- Add rate limiting for 2FA attempts
- Update swagger documentation

fix(database): fix user status migration bug
- Correct SQL migration for ACTIVE status
- Add index on status column for performance

docs(setup): update installation instructions
- Add PostgreSQL setup steps
- Include Redis configuration

refactor(api): simplify error handling middleware
- Consolidate error types
- Reduce middleware stack from 5 to 3

test(search): add autocomplete test cases
- Test empty query handling
- Test result limit validation

❌ BAD:
fix bug
updated stuff
changes
```

### Commit Message Guidelines

- **Type**: feat, fix, docs, refactor, test, chore
- **Scope**: Component name (auth, search, admin-dashboard)
- **Subject**: Lowercase, present tense, no period
- **Body**: Explain what and why (not how)
- **Footer**: Reference issues: `Fixes #123`

---

## 📥 Pull Requests

### Before Creating PR

1. **Run tests**
   ```bash
   npm test
   ```

2. **Lint code**
   ```bash
   npm run lint
   ```

3. **Format code**
   ```bash
   npm run format
   ```

4. **Build locally** (for frontends)
   ```bash
   npm run build
   ```

### PR Template

```markdown
## Description
Brief description of changes

## Type of Change
- [ ] New feature
- [ ] Bug fix
- [ ] Documentation
- [ ] Refactoring
- [ ] Testing

## Related Issue
Fixes #123

## Checklist
- [ ] Code follows style guidelines
- [ ] Tests pass locally
- [ ] No new console errors
- [ ] Documentation updated
- [ ] No breaking changes

## Testing
How to test these changes:
1. Navigate to...
2. Click...
3. Verify...

## Screenshots (if applicable)
[Add screenshots for UI changes]
```

### Create PR

1. Push your branch to your fork
2. Go to GitHub and create a PR against `main` branch
3. Fill out the PR template
4. Request review from maintainers

---

## 🔍 Code Review Process

### For Reviewers

1. **Check functionality**
   - Does it work as intended?
   - Are there edge cases?
   - Is error handling complete?

2. **Check code quality**
   - Follows naming conventions?
   - Tests included?
   - Documentation updated?

3. **Check performance**
   - Any N+1 queries?
   - Unnecessary re-renders?
   - Memory leaks?

4. **Provide feedback**
   - Be constructive and helpful
   - Suggest improvements
   - Praise good practices

### For Contributors

1. Address all review comments
2. Make changes and push
3. Request re-review
4. Wait for approval
5. Merge with "Squash and merge"

---

## 🧪 Testing

### Backend Tests

```bash
# Run all tests
npm test

# Run specific file
npm test auth.test.js

# Run with coverage
npm test -- --coverage

# Watch mode
npm test -- --watch
```

### Test Structure

```javascript
describe('AuthController', () => {
  describe('register', () => {
    it('should create a new user with valid email', async () => {
      // Arrange
      const userData = { email: 'test@company.com', password: 'pwd' };
      
      // Act
      const response = await api.post('/auth/register').send(userData);
      
      // Assert
      expect(response.status).toBe(201);
      expect(response.body.success).toBe(true);
    });

    it('should return 400 for invalid email', async () => {
      const response = await api.post('/auth/register').send({ email: 'invalid' });
      expect(response.status).toBe(400);
    });
  });
});
```

### Frontend Tests (E2E with Cypress - TODO)

```javascript
describe('User Login Flow', () => {
  it('should login successfully with valid credentials', () => {
    cy.visit('http://localhost:5173');
    cy.get('[data-testid="email-input"]').type('user@company.com');
    cy.get('[data-testid="password-input"]').type('Password123!');
    cy.get('[data-testid="login-button"]').click();
    cy.url().should('include', '/dashboard');
  });
});
```

---

## 🗄️ Database Changes

### For Schema Changes

1. **Create migration**
   ```bash
   npx prisma migrate dev --name add_new_field
   ```

2. **Review migration file** (`prisma/migrations/*/migration.sql`)

3. **Test locally**
   ```bash
   npx prisma db push
   ```

4. **Commit migration files**
   ```bash
   git add prisma/migrations/
   git commit -m "db: add new_field to users table"
   ```

### Migration Best Practices

- ✅ One migration per logical change
- ✅ Test migrations on staging before production
- ✅ Write reversible migrations
- ✅ Document complex migrations
- ❌ Don't skip migration files
- ❌ Don't manually edit migration.sql

---

## 📚 Documentation

### Code Documentation

```javascript
/**
 * Calculate user's API usage statistics
 * @param {number} userId - User ID
 * @param {string} date - ISO date string (YYYY-MM-DD)
 * @returns {Promise<Object>} Usage stats { used, limit, remaining }
 * @throws {Error} If user not found
 * @example
 * const stats = await getUsageStats(123, '2026-05-05');
 * console.log(stats); // { used: 1250, limit: 5000, remaining: 3750 }
 */
const getUsageStats = async (userId, date) => {
  // implementation
};
```

### README Updates

When adding features, update relevant README files:
- `README.md` - Main overview
- `docs/API_REFERENCE.md` - New endpoints
- `docs/SETUP.md` - New setup steps
- Component README.md - For frontend components

### API Documentation

Add swagger comments for new endpoints:

```javascript
/**
 * @swagger
 * /api/v1/search:
 *   get:
 *     summary: Search villages
 *     parameters:
 *       - in: query
 *         name: q
 *         required: true
 *         schema:
 *           type: string
 *     responses:
 *       200:
 *         description: Search results
 *       400:
 *         description: Invalid query
 */
```

---

## ✅ Pre-Commit Checklist

Before committing:

- [ ] Code follows style guidelines
- [ ] No `console.log()` in production code
- [ ] No hardcoded credentials (use .env)
- [ ] Tests pass: `npm test`
- [ ] No TypeScript errors
- [ ] Documentation updated
- [ ] No breaking changes (or changelog updated)
- [ ] Commit message follows convention

---

## 🚀 Release Process

### Version Bumping

We follow [Semantic Versioning](https://semver.org/):
- **MAJOR** (1.0.0): Breaking changes
- **MINOR** (0.1.0): New features
- **PATCH** (0.0.1): Bug fixes

```bash
# Update version in package.json
# Create git tag
git tag v1.2.3
git push origin v1.2.3

# GitHub Actions will handle deployment
```

---

## 📞 Getting Help

- **Questions?** Open a GitHub Discussion
- **Bug?** Open a GitHub Issue
- **Feature request?** Start a GitHub Discussion
- **Need help?** Check existing issues/PRs

---

## 📋 Code of Conduct

- Be respectful and inclusive
- No harassment or discrimination
- Constructive feedback only
- Focus on code, not person
- Help others learn

---

## 🎉 Thank You!

Thank you for contributing to BLUESTOCK! Your work helps make this platform better for everyone.

**Happy coding!** 🚀

