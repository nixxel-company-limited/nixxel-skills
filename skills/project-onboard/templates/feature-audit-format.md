# Feature Audit Output Format (Universal 8-Category)

Templates for Phase 3 (Feature Audit) output files.
Read this file BEFORE running the feature audit.

Each category auto-detects and auto-skips if not found in the project.
Output: 1 file per category in `.context/features/`.

---

## _feature-map.md (read this first)

The feature map is the cross-reference. It connects all 8 categories
into coherent features — frontend, backend, data, and auth in one view.

```markdown
# Feature Map
> Read this file WHEN you need to understand how a feature works end-to-end.
> Read specific feature sections — you rarely need the whole file.

## How to use
Find the feature you're working on. Follow the connections across all layers:
Frontend (Pages/State/Components) -> Backend (API/Jobs) -> Data (DB/Cache) -> Auth.
This tells you what's affected by your change.

## Features

### {Feature Name}

**Frontend:**
- Pages/Screens: /products -> `ProductListPage.tsx`, ProductListScreen (mobile)
- Navigation: MainTabs > ShopStack > ProductList
- State: productApi (RTK Query), cartSlice (Redux)
- Components: ProductCard, FilterBar, PriceRange

**Backend:**
- API: GET /api/products, POST /api/products/:id/review
- Middleware: auth + rateLimit(100/min)
- Jobs: syncProductInventory (BullMQ)
- Integrations: SML sync pulls product data nightly

**Data:**
- DB: Product, ProductCategory, ProductReview
- Cache: product:list (Redis, TTL 60s), product:{id} (Redis, TTL 300s)

**Auth:** public browsing, login required to review (role: user)

### {Feature Name}
...

## Unmapped

### Orphan Endpoints (API exists, no page/screen calls it)
- {method} {path} — {possible reason}

### Orphan Pages/Screens (exists, calls no API)
- {path} — {possible reason: static page? WIP?}

### Orphan State (store/slice defined, no consumer reads it)
- {store} — {possible reason}

### Orphan Jobs (job defined, never queued)
- {jobName} — {possible reason}

### Dead Routes (in router config but component is empty/placeholder)
- {route} — {possible reason}

### Unused DB Tables (not referenced by any endpoint)
- {table} — {possible reason}
```

---

## pages-routing.md

Covers web pages, mobile screens, navigation hierarchy, route guards,
deep links, and layouts.

```markdown
# Pages & Routing
> Read this file BEFORE modifying or creating any page, screen, or route.
> Check which APIs a page uses before changing them.
> Check which pages use an API before changing the API.

## How to use
Find the page/screen you're working on. Check its API dependencies,
auth requirements, and layout nesting. For mobile, check the navigator
hierarchy to understand where the screen lives.

## Page/Screen Index

| Route/Screen | Component | API Calls | Auth | Layout |
|-------------|-----------|-----------|------|--------|
| / | `HomePage` | GET /api/stats | public | MainLayout |
| /products | `ProductListPage` | GET /api/products | public | ShopLayout |
| /products/[id] | `ProductDetail` | GET /api/products/:id | public | ShopLayout |
| /admin/orders | `OrdersPage` | GET /api/admin/orders | admin | AdminLayout |
| (mobile) HomeScreen | `HomeScreen` | GET /api/stats | login | MainTabs |
| ... | ... | ... | ... | ... |

## Page/Screen Details

### / -> HomePage (web)
- **File:** `src/app/page.tsx`
- **API calls:** GET /api/stats, GET /api/featured-products
- **Route params:** none
- **Key components:** HeroBanner, FeaturedGrid, StatCard
- **Auth:** public
- **Layout:** MainLayout (header + footer)
- **Features:** Shows overview stats and featured products

### HomeScreen (mobile)
- **File:** `src/screens/HomeScreen.tsx`
- **API calls:** GET /api/stats, GET /api/featured-products
- **Route params:** none
- **Key widgets/components:** HeroBanner, ProductCarousel
- **Auth:** login required
- **Navigator:** MainTabs > HomeStack
- **Deep link:** myapp://home

### /products -> ProductListPage (web)
...

## Navigation (mobile, if applicable)

### Navigator Hierarchy
```
AppNavigator
├── AuthStack (unauthenticated)
│   ├── LoginScreen
│   └── RegisterScreen
├── MainTabs (authenticated)
│   ├── HomeStack
│   │   ├── HomeScreen
│   │   └── NotificationScreen
│   ├── ShopStack
│   │   ├── ProductListScreen
│   │   └── ProductDetailScreen
│   └── ProfileStack
│       ├── ProfileScreen
│       └── OrderHistoryScreen
```

### Deep Links
| Path | Screen | Params |
|------|--------|--------|
| myapp://product/:id | ProductDetailScreen | { id: string } |
| myapp://order/:id | OrderDetailScreen | { id: string } |

### Auth Guard Logic
- Unauthenticated users -> AuthStack
- Authenticated users -> MainTabs
- Admin role -> shows AdminTab in MainTabs

## Layouts

### MainLayout (web)
- **File:** `src/app/layout.tsx`
- **Contains:** Header, Footer, Sidebar (collapsible)
- **Used by:** all public pages

### AdminLayout (web)
- **File:** `src/app/admin/layout.tsx`
- **Contains:** AdminSidebar, AdminHeader
- **Used by:** all /admin/* pages
- **Auth:** redirects to /login if not admin

## Shared Components
{components used across multiple pages/screens}
- `DataTable` -- used by: /admin/orders, /admin/users, /admin/products
- `SearchFilter` -- used by: /products, /admin/orders
- `ProductCard` -- used by: /, /products, HomeScreen, ShopScreen
```

---

## state-data.md

Covers state management, client caching, API client patterns,
and data fetching strategies.

```markdown
# State & Data Fetching
> Read this file BEFORE modifying app state (stores, slices, providers, queries).
> Cross-reference with pages-routing.md to check which pages consume each store.

## How to use
Find the store/slice/query you need. Check its state shape, actions,
and which screens/pages depend on it. Follow API endpoint references
to api-middleware.md.

## State Management Overview

**Library:** Redux Toolkit + RTK Query (web), Riverpod (mobile)
**Persistence:** redux-persist (AsyncStorage), Hive (mobile offline)

## Stores / Slices

### cartSlice
- **File:** `src/store/cartSlice.ts`
- **State shape:**
  ```typescript
  { items: CartItem[], total: number, loading: boolean }
  ```
- **Key actions:** addItem, removeItem, updateQuantity, clearCart
- **Async operations:** checkout -> POST /api/orders
- **Persistence:** redux-persist (localStorage / AsyncStorage)
- **Consumers:** CartPage, CartScreen, CartIcon (badge count), CheckoutPage

### authSlice
- **File:** `src/store/authSlice.ts`
- **State shape:**
  ```typescript
  { user: User | null, token: string | null, loading: boolean }
  ```
- **Key actions:** login, logout, refreshToken
- **Async operations:** login -> POST /api/auth/login, refresh -> POST /api/auth/refresh
- **Persistence:** redux-persist (secure storage)
- **Consumers:** all authenticated pages, Header (user name), ProfilePage

## API Queries (RTK Query / React Query / SWR)

### productApi (RTK Query)
- **File:** `src/store/productApi.ts`
- **Base URL:** /api/products
- **Endpoints:**
  | Query/Mutation | Method | Path | Cache TTL |
  |---------------|--------|------|-----------|
  | getProducts | GET | /api/products | 60s |
  | getProduct | GET | /api/products/:id | 300s |
  | searchProducts | GET | /api/products/search | 30s |
  | createReview | POST | /api/products/:id/review | invalidates getProduct |
- **Tag invalidation:** createReview invalidates ['Product', id]
- **Consumers:** ProductListPage, ProductDetailPage, SearchPage

### orderApi (React Query)
- **File:** `src/hooks/useOrders.ts`
- **Query keys:** ['orders'], ['orders', id], ['orders', 'stats']
- **Endpoints:**
  | Hook | Method | Path |
  |------|--------|------|
  | useOrders | GET | /api/orders |
  | useOrder | GET | /api/orders/:id |
  | useCreateOrder | POST | /api/orders |
- **Consumers:** OrdersPage, OrderDetailPage, CheckoutPage

## Data Persistence (offline / local)

### Hive (mobile)
- **Box:** `recentSearches` -- stores last 20 search terms
- **Box:** `offlineCart` -- stores cart for offline browsing
- **Sync:** on app foreground, sync offlineCart -> cartSlice

## Patterns
- **Optimistic updates:** cart operations update UI immediately, revert on error
- **Cache invalidation:** tag-based (RTK Query), query key-based (React Query)
- **Error handling:** global error slice, toast notifications
```

---

## design-system.md

Covers component library, theming, forms/validation (client-side),
i18n, and accessibility patterns.

```markdown
# Design System & UI
> Read this file BEFORE modifying UI components, theming, forms, or i18n.
> Check the component inventory before creating new shared components.

## How to use
Check if a component already exists before creating one. Follow the
theming tokens for consistent styling. Check i18n files before adding text.

## Theming

**Library:** Tailwind CSS + custom theme tokens
**Config file:** `tailwind.config.ts`

### Key Tokens
| Token | Value | Usage |
|-------|-------|-------|
| primary | #2563EB | buttons, links, focus rings |
| secondary | #7C3AED | accents, badges |
| destructive | #DC2626 | delete buttons, error states |
| background | #FFFFFF / #0F172A | light/dark mode |
| border-radius | 0.5rem (default) | cards, inputs, buttons |
| font-sans | Inter, system-ui | body text |
| font-mono | JetBrains Mono | code blocks |

### Dark Mode
- Strategy: CSS class-based (`dark:` prefix)
- Toggle: stored in localStorage, respects system preference

## Shared Component Inventory

| Component | File | Props | Used by |
|-----------|------|-------|---------|
| DataTable | `components/DataTable.tsx` | columns, data, pagination, onSort | 8 admin pages |
| SearchFilter | `components/SearchFilter.tsx` | filters, onChange, onReset | 5 pages |
| Modal | `components/Modal.tsx` | open, onClose, title, children | 12 pages |
| FileUpload | `components/FileUpload.tsx` | accept, maxSize, onUpload | ProductForm, BrandForm |
| PriceDisplay | `components/PriceDisplay.tsx` | amount, currency, discount | ProductCard, CartItem |
| StatusBadge | `components/StatusBadge.tsx` | status, variant | OrdersPage, UsersPage |

## Forms & Validation (client-side)

**Library:** React Hook Form + Yup (web), Formik + Yup (mobile)

### Patterns
- Form schemas defined alongside form components
- Server errors mapped to field-level errors
- Validation runs on blur + submit

### Common Schemas
| Schema | File | Used by |
|--------|------|---------|
| loginSchema | `schemas/auth.ts` | LoginForm |
| productSchema | `schemas/product.ts` | ProductForm (admin) |
| addressSchema | `schemas/address.ts` | CheckoutForm, ProfileForm |

## i18n (Internationalization)

**Library:** next-intl (web), react-i18next (mobile)
**Supported locales:** en, th
**Default locale:** th

### File locations
- Web: `messages/en.json`, `messages/th.json`
- Mobile: `src/locales/en.json`, `src/locales/th.json`

### How to add new text
1. Add key to both `en.json` and `th.json`
2. Use `t('namespace.key')` in component
3. Never hardcode user-facing text

### Namespace structure
```json
{
  "common": { "save": "Save", "cancel": "Cancel" },
  "product": { "title": "Products", "addNew": "Add Product" },
  "order": { "title": "Orders", "status": "Status" }
}
```

## Accessibility (a11y)

### Patterns in use
- All interactive elements have ARIA labels
- Focus management on modal open/close
- Keyboard navigation for DataTable (arrow keys)
- Skip-to-content link on main layout
- Color contrast meets WCAG AA

### Screen reader support
- Live regions for toast notifications
- Descriptive alt text on product images
- Form error announcements via aria-live
```

---

## api-middleware.md

Covers API endpoints, middleware pipeline, server-side validation,
and error handling patterns.

```markdown
# API & Middleware
> Read this file BEFORE modifying or creating any API endpoint.
> Check existing endpoints before creating new ones.
> Follow the same patterns for auth, validation, and response shape.

## How to use
Check the endpoint index for existing routes. Check the middleware pipeline
to understand what runs before your handler. Follow error handling patterns
for consistent error responses.

## Endpoint Index

| Method | Path | Auth | Middleware | Summary |
|--------|------|------|-----------|---------|
| GET | /api/products | none | rateLimit | List products |
| GET | /api/products/:id | none | rateLimit | Get product |
| POST | /api/products | JWT+Admin | auth, validate, upload | Create product |
| POST | /api/orders | JWT | auth, validate | Create order |
| GET | /api/admin/users | JWT+Admin | auth, roleCheck | List users |
| ... | ... | ... | ... | ... |

## Endpoint Details

### GET /api/products
- **File:** `src/routes/product.route.ts` -> `src/controllers/product.controller.ts`
- **Auth:** none (public)
- **Middleware chain:** rateLimit(200/min) -> controller
- **Query params:** `page` (number), `limit` (number), `search` (string), `categoryId` (string)
- **Response:** `{ products: Product[], count: number }`
- **Logic:** Paginated product list with optional filters, uses Elasticsearch for search
- **Side effects:** none

### POST /api/orders
- **File:** `src/routes/order.route.ts` -> `src/controllers/order.controller.ts`
- **Auth:** JWT required (role: user)
- **Middleware chain:** auth -> validate(orderSchema) -> controller
- **Body:** `{ items: OrderItem[], addressId: string, paymentMethod: string }`
- **Response:** `{ order: Order, message: "Order created" }`
- **Logic:** Validates stock, calculates total, creates order, queues payment job
- **Side effects:** Queues `processPayment` job, sends order confirmation email

## Middleware Pipeline

### Execution Order (global)
1. `cors` -- CORS headers
2. `helmet` -- security headers
3. `express.json()` -- body parsing (limit: 10mb)
4. `requestId` -- attaches UUID to each request
5. `requestLogger` -- logs method, path, duration
6. `rateLimit` -- global rate limit (1000/min per IP)

### Per-Route Middleware
| Middleware | File | Purpose | Used by |
|-----------|------|---------|---------|
| auth | `middleware/auth.ts` | Validates JWT, attaches user to req | all protected routes |
| roleCheck(roles) | `middleware/role.ts` | Checks user role against allowed list | admin routes |
| validate(schema) | `middleware/validate.ts` | Validates body/query with Yup schema | POST/PUT routes |
| upload(config) | `middleware/upload.ts` | Handles multipart file upload via Multer | product, brand routes |
| rateLimit(opts) | `middleware/rateLimit.ts` | Per-endpoint rate limiting | public endpoints |

### Middleware Details

#### auth
- Extracts token from `Authorization: Bearer {token}`
- Verifies JWT signature and expiry
- Attaches `req.user = { id, email, role }` to request
- Returns 401 if token missing/invalid, 403 if expired

#### validate(schema)
- Validates `req.body` against provided Yup schema
- Returns 400 with field-level errors: `{ message: "Validation failed", errors: { field: "msg" } }`

## Error Handling

### Custom Error Classes
| Class | HTTP Status | Usage |
|-------|------------|-------|
| AppError | varies | base class |
| NotFoundError | 404 | resource not found |
| ValidationError | 400 | input validation failure |
| UnauthorizedError | 401 | missing/invalid auth |
| ForbiddenError | 403 | insufficient permissions |
| ConflictError | 409 | duplicate resource |

### Error Response Format
```json
{
  "message": "Human-readable error description"
}
```

### Global Error Handler
- **File:** `src/middleware/errorHandler.ts`
- Catches all unhandled errors
- Maps known error classes to HTTP status codes
- Unknown errors -> 500 with generic message
- Logs full error stack in development, sanitized in production

## Server Validation Schemas
| Schema | File | Used by endpoint |
|--------|------|-----------------|
| createProductSchema | `schemas/product.ts` | POST /api/products |
| updateProductSchema | `schemas/product.ts` | PUT /api/products/:id |
| createOrderSchema | `schemas/order.ts` | POST /api/orders |
| loginSchema | `schemas/auth.ts` | POST /api/auth/login |

## Response Format Standards
- Success (single): `{ resourceName: T, message?: string }`
- Success (list): `{ resourceNames: T[], count: number }`
- Error: `{ message: string }`
- Pagination: `page` + `limit` query params, response includes `count`
```

---

## data-layer.md

Covers database schema, migrations, caching (Redis), transactions,
seeding, and connection pooling.

```markdown
# Data Layer
> Read this file BEFORE modifying database schema, caching, or writing queries.
> Check relations before adding fields. Check indexes before adding queries.
> Check cache keys before adding new caching logic.

## How to use
Find the model you need. Check its relations and indexes. Check the
cache strategy for read-heavy endpoints. Follow the migration strategy
for schema changes.

## Schema Overview

### Product
| Field | Type | Notes |
|-------|------|-------|
| id | String (cuid) | PK |
| name | String | required |
| slug | String | unique |
| price | Decimal | required |
| description | String? | nullable |
| categoryId | String | FK -> Category |
| isActive | Boolean | default true |
| createdAt | DateTime | auto |
| updatedAt | DateTime | auto-update |

**Relations:**
- belongsTo Category (categoryId)
- hasMany ProductReview
- hasMany OrderItem

**Indexes:**
- slug (unique)
- categoryId
- isActive, createdAt (composite -- for listing queries)

### Order
| Field | Type | Notes |
|-------|------|-------|
| id | String (cuid) | PK |
| orderNumber | String | unique, auto-generated |
| userId | String | FK -> User |
| status | OrderStatus | enum |
| total | Decimal | computed |
| createdAt | DateTime | auto |

**Relations:**
- belongsTo User (userId)
- hasMany OrderItem
- hasOne Payment

**Indexes:**
- orderNumber (unique)
- userId, status (composite)
- createdAt

### {TableName}
...

## Enums

### OrderStatus
Values: PENDING, CONFIRMED, PROCESSING, SHIPPED, DELIVERED, CANCELLED
Used by: Order.status

### UserRole
Values: USER, ADMIN
Used by: User.role

## Key Relations Diagram

```
User --hasMany--> Order --hasMany--> OrderItem --belongsTo--> Product
User --hasMany--> ProductReview --belongsTo--> Product
Product --belongsTo--> Category
Category --hasMany--> Category (self-referencing, parentId)
Order --hasOne--> Payment
```

## Migrations

### Strategy
- **Development:** `prisma db push` (rapid iteration, no migration files)
- **Production:** `prisma migrate deploy` (tracked migration files)
- **Breaking changes:** always add new column as nullable first, backfill, then make required
- **Migration naming:** `YYYYMMDDHHMMSS_description`

### Recent Migrations
| Migration | Date | Description |
|-----------|------|-------------|
| 20260315120000_add_product_slug | 2026-03-15 | Added unique slug field to Product |
| 20260320090000_order_status_enum | 2026-03-20 | Added OrderStatus enum, migrated from string |
| ... | ... | ... |

## Caching (Redis)

### Cache Keys
| Key Pattern | Data | TTL | Invalidated by |
|-------------|------|-----|----------------|
| product:list:{page}:{filters} | paginated products | 60s | product create/update/delete |
| product:{id} | single product | 300s | product update/delete |
| category:tree | category hierarchy | 600s | category create/update/delete |
| user:session:{token} | session data | 24h | logout, token refresh |

### Invalidation Strategy
- **Write-through:** update cache on write (session data)
- **Cache-aside:** invalidate on write, populate on read (product data)
- **Pattern deletion:** `product:*` when bulk operations occur

### Redis Connection
- **Client:** ioredis
- **Config:** `src/infrastructure/redis.ts`
- **Connection pooling:** single connection, auto-reconnect

## Transactions

### Patterns Used
| Operation | Type | Tables Involved |
|-----------|------|----------------|
| Create order | Prisma $transaction | Order, OrderItem, Product (stock) |
| Process payment | Prisma $transaction | Payment, Order (status update) |
| Delete category | Prisma $transaction | Category, Product (reassign) |

### Example Pattern
```typescript
await prisma.$transaction(async (tx) => {
  const order = await tx.order.create({ ... });
  await tx.product.updateMany({ ... }); // decrement stock
  return order;
});
```

## Seeding

- **Script:** `prisma/seed.ts`
- **Run:** `yarn prisma db seed`
- **Creates:** admin user, sample categories, sample products
- **Test factories:** `test/{feature}/{feature}.factory.ts`
```

---

## auth-security.md

Covers authentication, authorization, rate limiting, CORS,
secrets management, and security headers.

```markdown
# Auth & Security
> Read this file BEFORE modifying auth flows, permissions, or security config.
> Cross-reference with api-middleware.md for per-endpoint auth requirements.

## How to use
Check auth schemes to understand how users authenticate. Check the
role/permission model before adding new protected routes. Check rate
limits before exposing new public endpoints.

## Authentication

### Schemes
| Scheme | Used for | Library |
|--------|----------|---------|
| JWT (Bearer token) | API authentication | jsonwebtoken |
| NextAuth (credentials) | Admin/Customer web login | next-auth |
| API Key (header) | Service-to-service | custom middleware |

### JWT Token Format
- **Algorithm:** HS256
- **Payload:** `{ id: string, email: string, role: string, iat: number, exp: number }`
- **Access token expiry:** 15 minutes
- **Refresh token expiry:** 7 days
- **Storage:** httpOnly cookie (web), secure storage (mobile)

### Login Flow
1. POST /api/auth/login with credentials
2. Server validates, returns `{ accessToken, refreshToken }`
3. Client stores tokens, attaches accessToken to Authorization header
4. On 401, client calls POST /api/auth/refresh with refreshToken
5. On refresh failure, redirect to login

## Authorization

### Role Model
| Role | Description | Access |
|------|-------------|--------|
| USER | Regular customer | own profile, own orders, product browsing |
| ADMIN | Back-office staff | all admin routes, user management, product management |

### Permission Checks
- Role-based: `roleMiddleware(['ADMIN'])` on admin routes
- Resource ownership: service-level check `order.userId === req.user.id`
- No ABAC/fine-grained permissions currently

## Rate Limiting

### Rules
| Scope | Limit | Window | Applied to |
|-------|-------|--------|-----------|
| Global | 1000 req | 1 min | all endpoints |
| Auth endpoints | 10 req | 1 min | /api/auth/login, /api/auth/register |
| File upload | 20 req | 1 min | POST /api/upload |
| Public product API | 200 req | 1 min | GET /api/products |

### Implementation
- **Library:** express-rate-limit + rate-limit-redis
- **Store:** Redis (shared across instances)
- **Response on limit:** 429 Too Many Requests

## CORS

- **Config file:** `src/middleware/cors.ts`
- **Allowed origins:** configured via `CORS_ORIGINS` env var (comma-separated)
- **Credentials:** allowed
- **Exposed headers:** Content-Disposition (for file downloads)

## Security Headers

- **Library:** helmet
- CSP: script-src 'self', style-src 'self' 'unsafe-inline'
- HSTS: max-age=31536000, includeSubDomains
- X-Frame-Options: DENY
- X-Content-Type-Options: nosniff

## Secrets (expected env vars, no values)

| Variable | Purpose | Required |
|----------|---------|----------|
| JWT_SECRET | Token signing | yes |
| JWT_REFRESH_SECRET | Refresh token signing | yes |
| DATABASE_URL | PostgreSQL connection | yes |
| REDIS_URL | Redis connection | yes |
| S3_ACCESS_KEY | MinIO/S3 access | yes |
| S3_SECRET_KEY | MinIO/S3 secret | yes |
| NEXTAUTH_SECRET | NextAuth encryption | yes (web) |
| OMISE_SECRET_KEY | Payment processing | yes (prod) |
```

---

## background-events.md

Covers background jobs, event-driven architecture, scheduled tasks,
and service-to-service communication.

```markdown
# Background & Events
> Read this file BEFORE modifying background jobs, events, or service communication.
> Check job retry behavior before changing queue processing logic.

## How to use
Find the job/event you need. Check what triggers it, what it does,
and its retry behavior. For service communication, check contracts
before changing request/response shapes.

## Background Jobs

**Library:** BullMQ
**Redis connection:** shared with cache (separate DB index)
**Worker process:** separate process (`yarn worker:dev`)

### Job Index

| Job Name | Queue | Trigger | Retry |
|----------|-------|---------|-------|
| processPayment | paymentQueue | Order created | 3x, exponential backoff |
| sendOrderEmail | emailQueue | Order confirmed | 5x, 30s delay |
| syncProductInventory | syncQueue | Cron (every 15min) | 3x, linear |
| generateReport | reportQueue | Admin request | 1x, no retry |
| expireUnpaidOrders | orderQueue | Cron (every 5min) | 3x, exponential |

### Job Details

#### processPayment
- **File:** `src/infrastructure/queue/processors/payment.processor.ts`
- **Queue:** paymentQueue
- **Triggered by:** Order creation (service layer queues after $transaction)
- **Payload:** `{ orderId: string, amount: number, method: string }`
- **What it does:** Calls Omise API to charge, updates Order status, creates Payment record
- **Retry:** 3 attempts, exponential backoff (1min, 4min, 16min)
- **On failure:** Sets order status to PAYMENT_FAILED, sends admin notification
- **Dependencies:** Omise API, Order service, Email queue

#### sendOrderEmail
- **File:** `src/infrastructure/queue/processors/email.processor.ts`
- **Queue:** emailQueue
- **Triggered by:** Order status change to CONFIRMED
- **Payload:** `{ orderId: string, recipientEmail: string, template: string }`
- **What it does:** Renders email template, sends via SMTP
- **Retry:** 5 attempts, fixed 30s delay
- **On failure:** Logged, no user notification

#### syncProductInventory
- **File:** `src/infrastructure/queue/processors/sync.processor.ts`
- **Queue:** syncQueue
- **Triggered by:** Cron schedule (every 15 minutes)
- **Payload:** `{}`
- **What it does:** Pulls product data from SML integration, updates local DB
- **Retry:** 3 attempts, linear backoff (1min)
- **Dependencies:** sml-integration-app API

### Queue Infrastructure
- **Config:** `src/infrastructure/queue/`
- **Queue definitions:** `src/infrastructure/queue/queues.ts`
- **Processor registration:** `src/infrastructure/queue/worker.ts`

## Scheduled Tasks (Cron)

| Schedule | Job | Description |
|----------|-----|-------------|
| */15 * * * * | syncProductInventory | Pull latest product data from SML |
| */5 * * * * | expireUnpaidOrders | Cancel orders unpaid after 30min |
| 0 2 * * * | generateDailyReport | Nightly sales summary |

## Events (if applicable)

### Event Bus
- **Type:** in-process (Node.js EventEmitter) -- not distributed
- **Config:** `src/infrastructure/events/eventBus.ts`

| Event | Emitted by | Consumed by |
|-------|-----------|-------------|
| order.created | OrderService | PaymentQueue, AnalyticsService |
| order.statusChanged | OrderService | EmailQueue, WebhookService |
| product.updated | ProductService | CacheInvalidator |

## Service-to-Service Communication

### External Services (outbound)
| Service | Purpose | Protocol | Auth |
|---------|---------|----------|------|
| sml-integration-app | Product data sync | HTTP REST (:5052) | API key |
| Omise | Payment processing | HTTPS | Secret key |
| SMTP (MailPit/SES) | Email delivery | SMTP | credentials |

### Contracts
#### sml-integration-app
- GET /api/products -> `{ products: SmlProduct[] }`
- GET /api/products/:sku -> `{ product: SmlProduct }`
- Error handling: retry 3x on 5xx, skip on 4xx

### Webhook Handlers (inbound)
| Path | Source | Payload | Action |
|------|--------|---------|--------|
| POST /api/webhooks/omise | Omise | charge event | Update payment status |
```

---

## integrations-infra.md

Covers external APIs, file storage, logging, config management,
health checks, and graceful shutdown.

```markdown
# Integrations & Infrastructure
> Read this file BEFORE modifying external integrations, logging, or config.
> Check existing integrations before adding new 3rd-party services.

## How to use
Find the integration you need. Check its auth method and error handling.
For config changes, check the env var list to avoid conflicts.

## External Service Inventory

| Service | Purpose | Auth Method | SDK/Client |
|---------|---------|-------------|-----------|
| Omise | Payment processing | API secret key | omise-node |
| AWS S3 / MinIO | File storage | Access key + secret | @aws-sdk/client-s3 |
| Elasticsearch | Search + logging | none (internal) | @elastic/elasticsearch |
| SMTP (MailPit/SES) | Email delivery | SMTP credentials | nodemailer |
| SML Integration | Product data sync | API key header | axios |

## File Storage

### Config
- **Development:** MinIO (S3-compatible, Docker, port 9000)
- **Production:** AWS S3
- **Client:** `src/infrastructure/storage/s3.ts`
- **Bucket:** `thunonline` (auto-created in dev)

### Upload Flow
1. Client sends file via multipart form
2. Multer middleware processes upload
3. Service uploads to S3/MinIO
4. Returns signed URL or public URL

### Constraints
- Max file size: 10MB (images), 50MB (documents)
- Allowed types: jpg, png, webp, pdf
- Signed URL expiry: 1 hour

## Logging

### Config
- **Library:** Winston
- **Format:** JSON (structured logging)
- **Levels:** error, warn, info, debug
- **File:** `src/infrastructure/logger.ts`

### Log Destinations
| Environment | Destination |
|-------------|-------------|
| Development | console (pretty-print) |
| Production | stdout (JSON) -> collected by container runtime |

### Structured Log Fields
```json
{
  "level": "info",
  "message": "Order created",
  "orderId": "abc123",
  "userId": "usr456",
  "timestamp": "2026-03-31T10:00:00Z",
  "requestId": "req-789"
}
```

### Logging Conventions
- All state transitions: log fromStatus + toStatus
- All business events: log entity + action + actor
- All errors: log full error + context
- Reference: `.context/logging-convention.md`

## Config Management

### Environment Variables
- **Validation:** Zod schema at startup (`src/config/env.ts`)
- **Files:** `environments/{env}.env` copied to `.env` by Makefile
- **Environments:** development, ci, qa, uat, production

### Feature Flags
- Not currently implemented (potential future addition)

### Key Config Options
| Variable | Default | Description |
|----------|---------|-------------|
| PORT | 4000 | API server port |
| NODE_ENV | development | Environment name |
| DATABASE_URL | (required) | PostgreSQL connection string |
| REDIS_URL | (required) | Redis connection string |
| S3_ENDPOINT | http://localhost:9000 | S3/MinIO endpoint |
| S3_BUCKET | thunonline | Storage bucket name |
| CORS_ORIGINS | http://localhost:3000 | Allowed CORS origins |
| LOG_LEVEL | info | Minimum log level |

## Health Checks

### Endpoints
| Path | Method | What it checks |
|------|--------|---------------|
| /api/health | GET | API is running |
| /api/health/ready | GET | DB + Redis + Elasticsearch connected |

### Readiness Check Details
- PostgreSQL: `SELECT 1`
- Redis: `PING`
- Elasticsearch: cluster health
- Returns 200 if all pass, 503 if any fail

## Graceful Shutdown

### Sequence
1. Receive SIGTERM/SIGINT
2. Stop accepting new HTTP connections
3. Wait for in-flight requests to complete (timeout: 30s)
4. Close BullMQ workers (finish current job)
5. Close database connection pool
6. Close Redis connection
7. Exit process

### Implementation
- **File:** `src/infrastructure/shutdown.ts`
- **Timeout:** 30 seconds before force exit
```

---

## audit-meta.json

Per-category stats format. Tracks what was found and documented.

```json
{
  "last_audit": "2026-03-31T12:00:00Z",
  "git_commit": "abc1234",
  "categories": {
    "pages-routing": { "pages": 55, "screens": 0, "routes": 55, "layouts": 4 },
    "state-data": { "stores": 6, "queries": 12, "persisted": 2 },
    "design-system": { "components": 45, "locales": 2, "forms": 8, "themes": 1 },
    "api-middleware": { "endpoints": 172, "middleware": 8, "error_classes": 5, "validation_schemas": 12 },
    "data-layer": { "models": 29, "migrations": 15, "cache_keys": 10, "enums": 4, "transactions": 3 },
    "auth-security": { "auth_schemes": 2, "roles": 2, "rate_limits": 4, "secrets": 8 },
    "background-events": { "jobs": 5, "events": 3, "scheduled": 3, "webhooks": 1 },
    "integrations-infra": { "external_services": 5, "storage_backends": 1, "health_endpoints": 2, "log_level": "info" }
  },
  "orphans": {
    "endpoints": 1,
    "pages": 0,
    "screens": 0,
    "state": 1,
    "jobs": 0,
    "routes": 0,
    "tables": 0
  }
}
```

---

## Scanning Strategy

When scanning for pages, screens, endpoints, state, and other artifacts,
use framework-specific patterns. Auto-detect the platform and framework
first, then apply the appropriate scanning strategy.

### Frontend -- Web

#### Next.js App Router
- Pages: `src/app/**/page.tsx`
- Layouts: `src/app/**/layout.tsx`
- API routes: `src/app/api/**/route.ts`
- Method: exported function name = HTTP method (GET, POST, PUT, DELETE)
- Route groups: `(groupName)` directories (not part of URL)
- Parallel routes: `@slotName` directories

#### Next.js Pages Router
- Pages: `src/pages/**/*.tsx` (not `_app`, `_document`)
- API routes: `src/pages/api/**/*.ts`
- Method: check `req.method` in handler

#### React (CRA / Vite)
- Pages: search for route definitions in `createBrowserRouter` or `<Route>`
- Look for `react-router-dom` patterns

#### Vue / Nuxt
- Pages: `pages/**/*.vue` (Nuxt file-based routing)
- Or `router/index.ts` for manual routes

#### SvelteKit
- Pages: `src/routes/**/+page.svelte`
- API: `src/routes/api/**/+server.ts`

### Frontend -- Mobile

#### React Native / Expo
- Screens: `src/screens/**/*.tsx`
- Navigation: `createStackNavigator`, `createBottomTabNavigator`, `createDrawerNavigator`
- Expo Router: `app/**/*.tsx` (file-based, similar to Next.js)
- Deep links: `app.json` > `expo.scheme`, linking config

#### Flutter
- Screens: `lib/screens/**/*.dart`, `lib/pages/**/*.dart`
- Navigation: `GoRouter` config, `MaterialPageRoute`, named routes
- State: `Bloc`, `Cubit`, `StateNotifier`, `ChangeNotifier`
- Deep links: `AndroidManifest.xml` intent filters, `Info.plist` URL schemes

### Backend

#### Express / Koa / Fastify
- Endpoints: `app.get()`, `app.post()`, `router.get()`, `router.post()`
- Or scan route registration files (`routes/*.ts`)
- Middleware: `app.use()`, route-level middleware arrays
- Error handler: `app.use((err, req, res, next) => ...)`

#### NestJS
- Controllers: `@Controller()` + `@Get()`, `@Post()`, etc.
- Middleware: `@UseGuards()`, `@UseInterceptors()`, `@UsePipes()`
- Modules: `@Module()` for dependency graph

#### Django / FastAPI / Flask
- Django: `urlpatterns` in `urls.py`, `ViewSet` classes
- FastAPI: `@app.get()`, `@app.post()`, Pydantic models
- Flask: `@app.route()`, blueprints

#### Hono / Elysia (Bun)
- Routes: `app.get()`, `app.post()` or method chaining
- Middleware: `.use()` chain

### State Management

#### Redux Toolkit
- Slices: `createSlice()` calls
- RTK Query: `createApi()` with `endpoints` builder
- Store config: `configureStore()`

#### Zustand / Jotai / Valtio
- Stores: `create()` (Zustand), `atom()` (Jotai)

#### Riverpod / BLoC (Flutter)
- Riverpod: `StateNotifierProvider`, `FutureProvider`, `StreamProvider`
- BLoC: `Bloc<Event, State>`, `Cubit<State>`

#### Pinia (Vue)
- Stores: `defineStore()`

### Data Layer

#### Prisma
- Schema: `prisma/schema.prisma`
- Migrations: `prisma/migrations/`
- Client usage: `prisma.modelName.findMany()`, etc.

#### TypeORM / Drizzle / Sequelize
- Entities: `@Entity()` decorators, schema definitions
- Migrations: migration directories

#### Redis
- Client setup: `ioredis`, `redis` package
- Cache patterns: `get/set/del` with key patterns

### Background Jobs

#### BullMQ / Bull
- Queues: `new Queue('name')`, `queue.add()`
- Processors: `new Worker('name', processor)`

#### Celery / Sidekiq
- Tasks: `@app.task`, `perform_async`

### Generic Fallbacks
- Search for fetch/axios/http calls in page/component files
- Match URL patterns to known endpoints
- Search for `process.env` references for config discovery
- Search for `import` statements to find SDK usage

The goal is completeness, not perfection. It's better to document
90% of artifacts with gaps noted than to miss entire sections.
Note uncertainty: "This endpoint may have additional query params
not captured from static analysis."
