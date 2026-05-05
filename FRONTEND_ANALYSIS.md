# Frontend Projects Analysis - Bluestock B2B API

## Executive Summary

Three React frontend applications with different purposes, all built with React 19 + Vite + Tailwind CSS:

1. **Admin Dashboard** (5174) - Platform monitoring & management
2. **B2B Portal** (5173) - User account & API key management  
3. **Demo App** (5175) - Customer-facing village search demo

---

## 1. ADMIN DASHBOARD

### Purpose
Internal platform monitoring dashboard for admins to track platform health and manage users.

### File Structure
```
admin-dashboard/
├── src/
│   ├── App.tsx                    # Main layout with page navigation
│   ├── pages/
│   │   ├── Overview.tsx/.jsx      # KPI cards + weekly traffic visualization
│   │   ├── Users.tsx/.jsx         # User directory with search filter
│   │   └── Logs.tsx/.jsx          # API request logs table
│   ├── lib/
│   │   ├── api.ts                 # Axios client + 3 admin endpoints
│   │   ├── store.ts               # Zustand: page state (overview|users|logs)
│   │   └── queryClient.ts         # TanStack Query configuration
│   ├── App.css, index.css          # Tailwind styles
│   └── assets/                     # Images (hero.png, react.svg, vite.svg)
├── package.json                    # React 19, Vite, Recharts, Zustand, Query
├── tsconfig.json, vite.config.js   # TypeScript & Vite configuration
└── tailwind.config.js, postcss.config.js
```

### Pages & Features

#### Overview Page
**KPI Cards (4):**
- Total users (e.g., 471)
- Active users (e.g., 471)
- Pending approvals (awaiting review)
- Total requests (all-time)

**Visualizations:**
- **LineChart**: 7-day API requests trend (Mon-Sun)
- **PieChart**: Users by plan distribution (Free/Premium/Pro/Unlimited)
- **Progress Bars**: Top 5 states by village count

#### Users Page
- **List**: All registered business users
- **Search**: Filter by business name or email
- **Display Fields**: Business name, email, status (Active/Pending/Suspended), plan
- **Pagination**: None (shows all on load)

#### Logs Page
- **Table**: Last 10 API requests
- **Columns**: Endpoint, Method (GET/POST/etc), Status (200/400/500), Latency (ms), Date (timestamp)
- **No filtering**: Shows last 10 only

### API Endpoints Used
```
GET /api/v1/admin/overview      → OverviewSummary
GET /api/v1/admin/users         → AdminUser[]
GET /api/v1/admin/logs?limit=10 → UsageLog[]
```

### Visualizations (Recharts)
- **LineChart**: Request volume over time
- **PieChart**: Plan distribution (donut chart with legend)
- **Manual Progress Bars**: Comparative village counts

### State Management
- **Zustand**: Single `useAdminPageStore()` for page selection
- **TanStack Query**: Data fetching with `useQuery()`
  - Auto-retry on error (default: 1 retry)
  - Manual refetch via loading state
  - Cached responses

### Authentication
- **None** - All requests made without auth headers
- Backend assumes admin-only middleware protection
- No login/logout flow in UI

### Technology Stack
```
React 19.2.4 (TypeScript)
├── Vite 8.0.4
├── Recharts 3.8.1 (charts)
├── Zustand 5.0.12 (page state)
├── @tanstack/react-query 5.99.2 (data fetching)
├── axios 1.15.2 (HTTP client)
├── Tailwind CSS 3.4.4
└── react-router-dom 7.14.2
```

### Design
- **Theme**: Dark mode (bg-slate-950, text-slate-100)
- **Components**: Rounded borders, card layout, blue accents (#3b82f6)
- **Responsive**: Grid layouts for mobile/tablet/desktop

---

## 2. B2B PORTAL

### Purpose
User-facing portal for B2B customers to manage API access, track usage, and generate API keys.

### File Structure
```
b2b-portal/
├── src/
│   ├── App.tsx                    # Auth flow controller & layout
│   ├── pages/
│   │   ├── Login.tsx/.jsx         # Email/password login form
│   │   ├── Register.tsx/.jsx      # Business registration (5 fields)
│   │   └── Dashboard.tsx/.jsx     # Main dashboard (KPIs + usage + API keys)
│   ├── lib/
│   │   ├── api.ts                 # Axios client + 7 auth/usage endpoints
│   │   ├── store.ts               # Zustand: auth state (token, user, setAuth, clearAuth)
│   │   └── queryClient.ts         # TanStack Query configuration
│   ├── App.css, index.css
│   └── assets/
├── package.json                    # React 19, Recharts, lucide-react, Zustand
├── tsconfig.json, vite.config.js
└── tailwind.config.js, postcss.config.js
```

### Pages & Features

#### Login Page
- **Form**: Email (required) + Password (required)
- **Actions**: Login button, "Create account" link to Register
- **Error Handling**: Red alert box for failed attempts
- **Loading**: Button disabled during submission

#### Register Page
- **Form Fields**:
  - Business name (required)
  - Email (required)
  - Phone (optional)
  - GST (optional)
  - Password (required)
- **Response**: Success message → "You can now sign in"
- **Error Handling**: Red alert for validation/server errors

#### Dashboard Page (Main)

**Layout**: Split sidebar (nav) + main content area

**Summary Section (4 KPI Cards)**:
- Today's requests (e.g., 1,245 / 50,000 daily limit)
- Daily remaining quota
- Avg response time (ms)
- Success rate (%)

**Usage Chart Section**:
- **LineChart**: 7-day request volume
- Updates from `dashboard.last7Days` API data
- Includes success rate & average response metrics
- Top 4 endpoints listed

**API Key Management Section**:
- **Create Key**: Generates new key (shown once), copy-to-clipboard
- **List Keys**: Shows all keys with ID, status, created date
- **Actions per Key**:
  - **Rotate**: Generate replacement key
  - **Revoke**: Delete/disable key
- **Key Display**: Masked by default, toggle visibility

**Navigation Sidebar**:
- Dashboard (selected)
- API Keys (nav item, no page implemented)
- Analytics (nav item, no page implemented)
- Settings (nav item, no page implemented)
- Sign out button

**Dark Mode Toggle**: Sun/Moon icons in header

### API Endpoints Used
```
POST /api/v1/auth/login                    → {token, user}
POST /api/v1/auth/register                 → {success}
GET  /api/v1/auth/me                       → {user}
GET  /api/v1/auth/apikeys                  → {ApiKeyListItem[]}
POST /api/v1/auth/apikeys                  → {apiKey: string}
PATCH /api/v1/auth/apikeys/{id}/revoke     → {}
PATCH /api/v1/auth/apikeys/{id}/rotate     → {apiKey: string}
GET  /api/v1/usage/dashboard               → {UsageDashboardData}
```

### Visualizations (Recharts)
- **LineChart**: API requests per day over 7 days
  - Data from `dashboard.last7Days`
  - X-axis: day names
  - Y-axis: request count
  - Stroke: indigo (#6366f1)

### State Management

**Zustand Auth Store**:
```typescript
{
  token: string | null              // JWT from login, stored in localStorage
  user: B2BUser | null              // User profile from API
  setAuth(token, user)              // Update both + persist token
  clearAuth()                       // Clear both + remove from localStorage
}
```

**Session Restoration**:
- On App mount: if token exists, fetch `/auth/me` to restore user
- Prevents re-login on page refresh
- Clears auth if token is invalid

**TanStack Query**:
- `useQuery(['usageDashboard'])` for dashboard data
- `useQuery(['apiKeys'])` for API keys list
- Mutations for create/rotate/revoke with optimistic UI updates
- Cache invalidation after mutations

### Authentication
- **Type**: JWT (Bearer token)
- **Flow**:
  1. User submits login → POST `/auth/login`
  2. Receives `{token, user}` 
  3. Token saved to localStorage (`b2b_token`)
  4. User object stored in Zustand
  5. All subsequent requests include `Authorization: Bearer {token}`
- **Session Persistence**: Token recovered from localStorage on app load
- **Plan-based Limits**: `user.dailyLimit` determines quota
- **State Filtering**: `user.stateAccess` available (not used in current UI)

### Technology Stack
```
React 19.2.4 (TypeScript)
├── Vite 8.0.4
├── Recharts 3.8.1 (LineChart only)
├── lucide-react 1.8.0 (icons)
├── Zustand 5.0.12 (auth state)
├── @tanstack/react-query 5.99.2 (server state)
├── axios 1.15.2 (HTTP client)
├── Tailwind CSS 3.4.4
└── react-router-dom 7.14.2
```

### Design
- **Theme**: Light & dark modes (toggle via header button)
- **Components**: Rounded cards, grid layouts, blue accents
- **Responsive**: Mobile-first grid system
- **Icons**: LayoutDashboard, Key, BarChart3, Settings, LogOut, Sun, Moon, Copy, Eye, EyeOff

---

## 3. DEMO APP

### Purpose
Simple public demo showing how end customers can integrate with Bluestock API for village geolocation searches.

### File Structure
```
demo-app/
├── src/
│   ├── App.jsx                    # Single page: contact form with village search
│   ├── App.css, index.css
│   └── assets/
├── package.json                    # React 19, Axios (minimal dependencies)
├── vite.config.js
├── tailwind.config.js
└── postcss.config.js
```

### Features

**Single Contact Form Page**:

**Input Fields**:
- Full Name (required, text)
- Email (required, email)
- Phone (optional, tel)
- Village Search (required, autocomplete)

**Auto-filled Location Fields** (read-only after village selection):
- Village Name
- Sub-District
- District
- State
- Country (hardcoded: "India")

**Village Search Autocomplete**:
- Debounced input (300ms delay)
- Queries: `/api/v1/search/autocomplete?q={query}`
- Shows dropdown suggestions as user types
- Loading spinner during fetch
- Error messages for network/API failures
- "No results" message if no matches

**Village Selection Flow**:
1. User types village name (min 2 chars)
2. Debounced API call fetches suggestions
3. User clicks suggestion from dropdown
4. Second API call: `/api/v1/villages/{id}` fetches full hierarchy
5. Form auto-fills location fields from hierarchy response

**Form Validation**:
- Required fields: fullName, email, villageId
- Error alert shown if incomplete

**Form Submission**:
- Logs data to browser console
- Shows success screen with submitted details
- "Submit another" button to reset and submit again

**Configuration**:
- API URL: env var `VITE_API_URL` (default: localhost:3000)
- Demo API Key: env var `VITE_DEMO_API_KEY`
- Warning banner if API key not configured

### API Endpoints Used
```
GET /api/v1/search/autocomplete?q={query}    → {data: [{label, value}]}
GET /api/v1/villages/{id}                    → {data: {hierarchy}}
```

### Visualizations
- **None** - Pure form application

### State Management
- **React Hooks Only** - No external state managers
- `useState` for form, search, loading states
- `useRef` for debounce timeout tracking
- No global state or context

**State Variables**:
```javascript
form                   // Contact form + location fields
query                  // Search input value
suggestions            // Dropdown suggestions
loading                // Search loading state
detailsLoading         // Village details loading state
submitted              // Form submission success flag
error                  // Error messages
```

### Authentication
- **Type**: API Key (X-API-Key header)
- **Source**: Environment variable `VITE_DEMO_API_KEY`
- **No User Auth**: Public demo, hardcoded key in Vite config
- **Axios Configuration**:
  ```javascript
  axios.create({
    baseURL: 'http://localhost:3000',
    headers: DEMO_KEY ? { 'X-API-Key': DEMO_KEY } : {}
  })
  ```

### Technology Stack
```
React 19.2.5 (JSX only, no TypeScript)
├── Vite 8.0.4
├── Axios 1.15.2 (HTTP client)
├── Tailwind CSS 3.4.4
└── (No charting, no state management libraries)
```

### Design
- **Theme**: Light theme (gray-50 background, white cards)
- **Components**: Simple form layout, rounded inputs
- **Responsive**: Single column on mobile, max 2xl width
- **No libraries**: All styling via Tailwind

---

## Comparison Matrix

| Feature | Admin Dashboard | B2B Portal | Demo App |
|---------|-----------------|-----------|----------|
| **Purpose** | Admin monitoring | B2B customer | Public demo |
| **Pages** | 3 | 3 | 1 |
| **Authentication** | None | JWT + localStorage | API Key |
| **Data Fetching** | TanStack Query | TanStack Query | Fetch hooks |
| **State Management** | Zustand (UI) | Zustand (Auth) | React hooks |
| **Charts** | 3 types | 1 type | 0 |
| **Dark Mode** | No (dark only) | Yes (toggle) | No (light only) |
| **Icons** | None | lucide-react (10+) | None |
| **Recharts** | ✓ | ✓ | ✗ |
| **TypeScript** | ✓ (tsx) | ✓ (tsx) | ✗ (jsx) |

---

## Component Hierarchy

### Admin Dashboard
```
App.tsx
├── Header (title)
├── Navigation (3 buttons)
└── Suspense
    ├── Overview.tsx
    │   ├── KPI Cards (4)
    │   ├── LineChart (requests over time)
    │   ├── PieChart (plan distribution)
    │   └── Progress bars (top states)
    ├── Users.tsx
    │   ├── Search input
    │   └── User list cards
    └── Logs.tsx
        └── Logs table
```

### B2B Portal
```
App.tsx (Auth flow controller)
├── Header
├── Suspense
│   ├── Login.tsx
│   │   ├── Email input
│   │   ├── Password input
│   │   └── Login button
│   ├── Register.tsx
│   │   ├── Business name input
│   │   ├── Email input
│   │   ├── Phone input
│   │   ├── GST input
│   │   ├── Password input
│   │   └── Register button
│   └── Dashboard.tsx
│       ├── Sidebar nav
│       ├── Header (title + dark mode)
│       ├── KPI cards (4)
│       ├── LineChart (usage)
│       ├── API key creation
│       └── API keys list
```

### Demo App
```
App.jsx (single page)
├── Warning banner (if no API key)
├── Title
├── Error messages
└── Form
    ├── Full name input
    ├── Email input
    ├── Phone input
    ├── Village search input (autocomplete)
    ├── Suggestions dropdown
    ├── Location fields (auto-filled)
    └── Submit button
    
[After submission]
└── Success screen
    ├── Submitted data summary
    └── "Submit another" button
```

---

## API Integration Summary

### Admin Dashboard
- **Base URL**: `http://localhost:3000/api/v1`
- **Auth**: None (admin-only)
- **Endpoints**: 3
- **Data Flow**: `useQuery` → axios → JSON response

### B2B Portal  
- **Base URL**: `http://localhost:3000/api/v1`
- **Auth**: JWT Bearer token in Authorization header
- **Endpoints**: 7
- **Data Flow**: 
  - Login → token saved to localStorage
  - Subsequent requests include token header
  - `useQuery`/`useMutation` manage request lifecycle

### Demo App
- **Base URL**: `http://localhost:3000/api/v1` (or env override)
- **Auth**: X-API-Key header (demo key from env)
- **Endpoints**: 2
- **Data Flow**: Debounced fetch → axios → state update

---

## Missing Features vs Requirements

### Admin Dashboard
**Implemented**:
- ✅ Overview page with KPIs and charts
- ✅ User directory with search
- ✅ API logs viewing

**Not Implemented**:
- ❌ User action buttons (approve/suspend/restore)
- ❌ Log filtering/search by endpoint, status, date
- ❌ Pagination (shows only last 10 logs)
- ❌ Export functionality (CSV/PDF)
- ❌ Real-time updates (WebSocket)
- ❌ Analytics page (nav placeholder)
- ❌ Settings page (nav placeholder)

### B2B Portal
**Implemented**:
- ✅ Login/Register flows
- ✅ Dashboard with KPIs
- ✅ 7-day usage chart
- ✅ API key management (create/rotate/revoke)
- ✅ Dark mode toggle
- ✅ Session persistence (localStorage)

**Not Implemented**:
- ❌ State-level access control (field exists, not used)
- ❌ Monthly/yearly usage views
- ❌ Advanced analytics (endpoint breakdowns)
- ❌ Billing/subscription management
- ❌ Email verification flow
- ❌ 2FA setup (backend supports it)
- ❌ Settings page (nav placeholder)
- ❌ API documentation embed
- ❌ Pagination for API keys

### Demo App
**Implemented**:
- ✅ Contact form with validation
- ✅ Village search autocomplete
- ✅ Auto-fill location hierarchy
- ✅ Error handling
- ✅ Success confirmation

**Not Implemented**:
- ❌ Result persistence (database)
- ❌ Email notification on submit
- ❌ Map visualization
- ❌ Address copy-to-clipboard
- ❌ Export form data
- ❌ Multi-step/wizard form
- ❌ File upload support

---

## Performance Optimizations

### Implemented
- **Debouncing**: Demo app village search (300ms)
- **Lazy Loading**: React.lazy() for pages in admin dashboard
- **Caching**: TanStack Query (auto-refetch on stale)
- **Responsive Containers**: Recharts ResponsiveContainer
- **Code Splitting**: Vite bundling

### Potential Improvements
- Pagination for user/log lists in admin
- Virtual scrolling for large lists
- Image optimization (hero.png)
- Component memoization (React.memo for list items)
- API response caching strategies
- Progressive enhancement

---

## Environment Configuration

### Admin Dashboard
```
VITE_API_BASE_URL (default: http://localhost:3000/api/v1)
```

### B2B Portal
```
VITE_API_BASE_URL (default: http://localhost:3000/api/v1)
```

### Demo App
```
VITE_API_URL (default: http://localhost:3000)
VITE_DEMO_API_KEY (optional, shows warning if missing)
```

---

## Deployment Ports

- **Admin Dashboard**: `npm run dev` → `http://localhost:5174`
- **B2B Portal**: `npm run dev` → `http://localhost:5173`
- **Demo App**: `npm run dev` → `http://localhost:5175`
- **Backend API**: `http://localhost:3000`

---

## Key Takeaways

1. **Admin Dashboard**: Monitoring-focused, minimal interactivity, no user auth
2. **B2B Portal**: Full-featured customer portal with JWT auth and key management
3. **Demo App**: Lightweight integration example showing API usage patterns
4. All three use React 19 + Vite for modern development experience
5. No charting in demo app; admin and portal use Recharts for visualizations
6. Consistent Tailwind CSS styling across all projects
7. State management chosen based on needs (Zustand for light, hooks for simplest)
