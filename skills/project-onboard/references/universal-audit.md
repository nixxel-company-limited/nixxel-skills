# Universal Audit Reference (Phase 3)

> Read this file BEFORE running Phase 3 (Feature Audit).
> It contains detection logic, scanning strategies, and output templates
> for all 8 audit categories across web, mobile, and backend platforms.

---

## Table of Contents

1. [Category Detection](#category-detection)
2. [Category 1: Pages & Routing](#1-pages--routing)
3. [Category 2: State & Data Fetching](#2-state--data-fetching)
4. [Category 3: Design System & UI](#3-design-system--ui)
5. [Category 4: API & Middleware](#4-api--middleware)
6. [Category 5: Data Layer](#5-data-layer)
7. [Category 6: Auth & Security](#6-auth--security)
8. [Category 7: Background & Events](#7-background--events)
9. [Category 8: Integrations & Infrastructure](#8-integrations--infrastructure)
10. [Extended Feature Map Format](#extended-feature-map-format)
11. [Extended Unmapped Section](#extended-unmapped-section)
12. [audit-meta.json Schema](#audit-metajson-schema)

---

## Category Detection

Before scanning, determine which categories apply. Check the detection
table below. If no signals are found for a category, skip it entirely.
Record skipped categories in `audit-meta.json` with `"skipped": true`.

### Detection Table

| Category | Detection Signals | Glob Patterns | Grep Patterns |
|----------|------------------|---------------|---------------|
| Pages & Routing | Page/screen files, router config | `**/page.tsx`, `**/screen*.tsx`, `**/screens/**`, `lib/screens/**/*.dart` | `createStackNavigator`, `GoRoute`, `createBrowserRouter` |
| State & Data Fetching | State lib in deps | `**/store*`, `**/slice*`, `**/*Provider*` | `createSlice`, `create()`, `useQuery`, `defineStore`, `Bloc<`, `StateNotifierProvider` |
| Design System & UI | Theme/component files, UI lib in deps | `**/theme*`, `**/tokens/**`, `**/components/**`, `**/locales/**` | `ThemeProvider`, `createTheme`, `tailwind.config`, `useForm`, `useTranslation` |
| API & Middleware | Route/controller files, middleware | `**/routes/**`, `**/controllers/**`, `**/middleware/**` | `app.get(`, `app.post(`, `router.get(`, `@Controller`, `@app.route`, `def view` |
| Data Layer | ORM schema, migration dirs | `**/schema.prisma`, `**/migrations/**`, `**/models/**` | `model `, `createTable`, `class Meta:`, `Schema.define` |
| Auth & Security | Auth middleware, JWT config | `**/auth*`, `**/guard*`, `**/passport*` | `jwt.verify`, `NextAuth`, `passport.use`, `@UseGuards`, `authenticate` |
| Background & Events | Queue lib in deps, job files | `**/queue*`, `**/jobs/**`, `**/workers/**`, `**/tasks/**` | `queue.add`, `@app.task`, `perform_async`, `Bull`, `defineJob` |
| Integrations & Infra | 3rd-party SDKs, logger setup | `**/health*`, `**/logger*`, `**/config*` | `S3Client`, `createClient`, `Winston`, `Pino`, `structlog`, `Stripe`, `Omise` |

### Dependency Detection (package.json / pubspec.yaml / requirements.txt)

| Dependency | Triggers Category |
|------------|------------------|
| `react-native`, `expo` | Pages & Routing (mobile screens) |
| `flutter` | Pages & Routing (mobile screens) |
| `@reduxjs/toolkit`, `zustand`, `jotai`, `recoil` | State & Data Fetching |
| `@tanstack/react-query`, `swr`, `rtk-query` | State & Data Fetching |
| `riverpod`, `bloc`, `provider`, `pinia` | State & Data Fetching |
| `@mui/material`, `@chakra-ui/react`, `tailwindcss` | Design System & UI |
| `react-hook-form`, `formik`, `yup`, `zod` | Design System & UI |
| `next-intl`, `react-i18next`, `i18next`, `flutter_localizations` | Design System & UI |
| `express`, `fastify`, `hono`, `@nestjs/core` | API & Middleware |
| `fastapi`, `django`, `flask`, `rails` | API & Middleware |
| `prisma`, `@prisma/client`, `typeorm`, `drizzle-orm`, `knex` | Data Layer |
| `sequelize`, `sqlalchemy`, `activerecord` | Data Layer |
| `ioredis`, `redis` | Data Layer (caching) |
| `passport`, `next-auth`, `@auth/core`, `jsonwebtoken` | Auth & Security |
| `express-rate-limit`, `helmet`, `cors` | Auth & Security |
| `bullmq`, `bull`, `celery`, `sidekiq`, `agenda` | Background & Events |
| `@aws-sdk/client-s3`, `minio`, `stripe`, `@sendgrid/mail` | Integrations & Infra |
| `winston`, `pino`, `morgan` | Integrations & Infra |

---

## 1. Pages & Routing

**Output file:** `pages-routing.md`
**Covers:** Pages, screens, routing, navigation, deep links, layouts, route guards

### Scanning Strategy

#### Web: Next.js (App Router)

```
Glob: app/**/page.tsx, app/**/page.jsx, app/**/page.ts
```
- Each `page.tsx` = one route. Path derived from directory structure.
- Layouts: `app/**/layout.tsx` — document nesting hierarchy.
- Route groups: `(groupName)/` directories — note grouping purpose.
- Parallel routes: `@slotName/` directories.
- Intercepting routes: `(.)`, `(..)`, `(...)` prefixes.

```
Grep: "use client" — marks client components (affects data fetching strategy)
Grep: generateMetadata|generateStaticParams — marks SSG/SSR behavior
Grep: redirect\(|notFound\( — route guard behavior
```

#### Web: Next.js (Pages Router)

```
Glob: pages/**/*.tsx, pages/**/*.jsx (exclude pages/api/**)
```
- `getServerSideProps` / `getStaticProps` / `getStaticPaths` for data strategy.
- `_app.tsx` for global layout, `_document.tsx` for HTML wrapper.

#### Web: React SPA (React Router)

```
Grep: createBrowserRouter|createHashRouter|BrowserRouter
Grep: <Route |<Routes|useRoutes
Glob: src/pages/**/*.tsx, src/views/**/*.tsx
```
- Scan route config object or JSX `<Route>` tree for path mapping.
- Look for `loader` and `action` functions (React Router v6.4+).

#### Web: Vue/Nuxt

```
Glob: pages/**/*.vue (Nuxt — file-based routing)
Glob: src/views/**/*.vue, src/pages/**/*.vue (Vue SPA)
Grep: createRouter|definePageMeta|useRoute
```
- Nuxt: middleware in `middleware/` directory.
- Vue Router: scan `router/index.ts` for route definitions.

#### Web: SvelteKit

```
Glob: src/routes/**/+page.svelte, src/routes/**/+page.ts
Glob: src/routes/**/+layout.svelte
```
- `+page.server.ts` for server-side load functions.
- `+layout.ts` / `+layout.server.ts` for layout data.

#### Mobile: React Native (Expo Router)

```
Glob: app/**/*.tsx, app/**/*.ts (Expo Router — file-based)
Grep: <Stack|<Tabs|<Drawer — navigator types
Grep: useLocalSearchParams|useGlobalSearchParams — route params
Grep: <Link href|router.push|router.replace — navigation calls
```
- `app/_layout.tsx` defines root navigator.
- `app/(tabs)/` for tab-based navigation.

#### Mobile: React Native (React Navigation)

```
Grep: createStackNavigator|createBottomTabNavigator|createDrawerNavigator
Grep: createNativeStackNavigator|NavigationContainer
Glob: src/screens/**/*.tsx, src/navigation/**/*.tsx
```
- Scan navigator factory calls for screen registration.
- Deep links: look for `linking` config in `NavigationContainer`.

#### Mobile: Flutter

```
Glob: lib/screens/**/*.dart, lib/pages/**/*.dart
Grep: GoRoute|GoRouter|MaterialPageRoute|Navigator.push
Grep: @RoutePage|AutoRoute — auto_route package
```
- Check `MaterialApp.router` or `GoRouter` config for route definitions.
- Deep links: `GoRoute(path:` patterns.

### What to Document Per Page/Screen

For each page or screen found:

| Field | Description |
|-------|-------------|
| Route path | URL path (web) or screen name (mobile) |
| File path | Relative file path from project root |
| API calls | Endpoints called (fetch/axios/http patterns) |
| Key components | Main UI components or widgets used |
| Route params | Dynamic segments (`:id`, `[slug]`, path params) |
| Auth required | Public or private (inferred from guards/middleware) |
| Layout | Parent layout or navigator |
| Data strategy | SSR / SSG / CSR / SWR (web) or provider type (mobile) |

### Output Template

```markdown
# Pages & Routing
> Read this file BEFORE modifying or creating any page, screen, or route.

## How to use
Check existing routes before creating new ones. Follow the same layout
and auth patterns. Check API calls before changing any endpoint.

## Route Index

| Route | File | Auth | API Calls | Layout |
|-------|------|------|-----------|--------|
| / | `app/page.tsx` | public | GET /api/stats | RootLayout |
| /products | `app/products/page.tsx` | public | GET /api/products | RootLayout |
| /products/[id] | `app/products/[id]/page.tsx` | public | GET /api/products/:id | RootLayout |
| /admin/orders | `app/(admin)/orders/page.tsx` | admin | GET /api/orders | AdminLayout |
| /cart | `app/cart/page.tsx` | user | GET /api/cart | RootLayout |

## Route Details

### / -> HomePage
- **File:** `app/page.tsx`
- **Auth:** public
- **Data strategy:** SSR (server component with direct DB query)
- **API calls:** GET /api/stats, GET /api/featured-products
- **Key components:** HeroBanner, FeaturedProducts, CategoryGrid
- **Layout:** RootLayout (`app/layout.tsx`)

### /products -> ProductListPage
- **File:** `app/products/page.tsx`
- **Auth:** public
- **Data strategy:** CSR (React Query)
- **API calls:** GET /api/products?page=&limit=&category=
- **Key components:** ProductCard, FilterSidebar, Pagination
- **Route params:** none (uses query string)
- **Layout:** RootLayout

### /admin/orders -> AdminOrdersPage
- **File:** `app/(admin)/orders/page.tsx`
- **Auth:** JWT + admin role (redirects to /login if unauthorized)
- **Data strategy:** CSR (SWR with revalidation)
- **API calls:** GET /api/orders, PATCH /api/orders/:id/status
- **Key components:** OrderTable, StatusBadge, OrderDetail
- **Layout:** AdminLayout (`app/(admin)/layout.tsx`)

## Layout Hierarchy

RootLayout (app/layout.tsx)
  ├── (public pages) — Header, Footer
  └── AdminLayout (app/(admin)/layout.tsx) — Sidebar, AdminHeader

## Navigation (mobile only — include if mobile detected)

### Navigator Hierarchy
MainNavigator (Stack)
  ├── AuthStack
  │   ├── LoginScreen
  │   └── RegisterScreen
  ├── MainTabs (BottomTab)
  │   ├── HomeStack
  │   │   ├── HomeScreen
  │   │   └── ProductDetailScreen
  │   ├── CartStack
  │   │   └── CartScreen
  │   └── ProfileStack
  │       ├── ProfileScreen
  │       └── OrderHistoryScreen
  └── CheckoutStack
      ├── ShippingScreen
      └── PaymentScreen

### Deep Links
| Deep Link | Screen | Params |
|-----------|--------|--------|
| /product/:id | ProductDetailScreen | productId |
| /order/:id | OrderDetailScreen | orderId |

## Route Guards
- Admin routes: middleware checks JWT + role === 'admin', redirects to /login
- User routes: middleware checks JWT exists, redirects to /login
- Public routes: no guard
```

---

## 2. State & Data Fetching

**Output file:** `state-data.md`
**Covers:** State management, client caching, API client patterns, data fetching hooks

### Scanning Strategy

#### Redux Toolkit

```
Grep: createSlice\(|createAsyncThunk\(|createApi\(|configureStore\(
Glob: src/store/**/*.ts, src/slices/**/*.ts, src/features/**/slice.ts
```
- `createSlice` — document name, initial state shape, reducers, extraReducers.
- `createApi` (RTK Query) — document endpoints, tags, transformResponse.
- `configureStore` — document middleware chain, devtools config.
- `redux-persist` — document which slices are persisted, storage backend.

#### Zustand

```
Grep: create\(\(set|create\(\(set, get
Glob: src/store/**/*.ts, src/stores/**/*.ts
```
- Each `create()` call = one store. Document state shape and actions.
- Check for middleware: `persist`, `devtools`, `immer`.

#### React Query / TanStack Query

```
Grep: useQuery\(|useMutation\(|useInfiniteQuery\(|queryClient
Grep: QueryClientProvider|queryOptions\(
Glob: src/hooks/**/*.ts, src/api/**/*.ts, src/queries/**/*.ts
```
- Document query keys, fetcher functions, stale time, cache time.
- Document mutations and their `onSuccess` / `onError` / invalidation logic.

#### SWR

```
Grep: useSWR\(|useSWRMutation\(|SWRConfig
```

#### Pinia (Vue)

```
Grep: defineStore\(
Glob: src/stores/**/*.ts
```

#### Riverpod (Flutter)

```
Grep: StateNotifierProvider|FutureProvider|StreamProvider|ChangeNotifierProvider
Grep: @riverpod|ref\.watch|ref\.read
Glob: lib/providers/**/*.dart, lib/notifiers/**/*.dart
```

#### BLoC (Flutter)

```
Grep: class \w+ extends Bloc<|class \w+ extends Cubit<
Grep: BlocProvider|BlocBuilder|BlocListener
Glob: lib/bloc/**/*.dart, lib/blocs/**/*.dart, lib/cubits/**/*.dart
```

#### Data Persistence

```
Grep: redux-persist|AsyncStorage|MMKV|Hive|SharedPreferences|SecureStorage
```

### What to Document Per Store/Slice

| Field | Description |
|-------|-------------|
| Name | Store or slice name |
| File path | Relative path |
| Type | Redux slice / Zustand store / React Query / BLoC / etc. |
| State shape | Key fields with types |
| Actions/methods | List of actions or mutations |
| Async operations | Which API endpoints they call |
| Persistence | What is persisted and where |
| Consumers | Which pages/screens read or write this state |

### Output Template

```markdown
# State & Data Fetching
> Read this file BEFORE modifying stores, slices, providers, or data fetching logic.

## How to use
Check existing state before adding new stores. Understand which pages consume
which state to avoid breaking changes. Follow the same patterns for new state.

## State Management Overview
- **Library:** Redux Toolkit (slices) + React Query (server state)
- **Persistence:** redux-persist with AsyncStorage (auth slice only)
- **DevTools:** enabled in development

## Stores & Slices

### authSlice
- **File:** `src/store/authSlice.ts`
- **Type:** Redux Toolkit slice
- **State shape:**
  ```typescript
  { user: User | null, token: string | null, isLoading: boolean }
  ```
- **Actions:** `setCredentials`, `logout`, `setLoading`
- **Async:** `loginThunk` -> POST /api/auth/login, `refreshThunk` -> POST /api/auth/refresh
- **Persistence:** yes (AsyncStorage, whitelist: ['token'])
- **Consumers:** LoginPage, Header (user info), all auth-guarded routes

### cartSlice
- **File:** `src/store/cartSlice.ts`
- **Type:** Redux Toolkit slice
- **State shape:**
  ```typescript
  { items: CartItem[], coupon: string | null }
  ```
- **Actions:** `addItem`, `removeItem`, `updateQuantity`, `applyCoupon`, `clearCart`
- **Async:** none (local state only, synced on checkout)
- **Persistence:** yes (localStorage)
- **Consumers:** CartPage, ProductDetail (add to cart), Header (badge count)

## Data Fetching (React Query)

### Product Queries
- **File:** `src/api/products.ts`
- **Queries:**
  | Key | Endpoint | Stale Time | Used By |
  |-----|----------|------------|---------|
  | `['products', filters]` | GET /api/products | 30s | ProductListPage |
  | `['product', id]` | GET /api/products/:id | 60s | ProductDetailPage |
  | `['products', 'featured']` | GET /api/products/featured | 5m | HomePage |
- **Mutations:**
  | Name | Endpoint | Invalidates |
  |------|----------|-------------|
  | `useCreateProduct` | POST /api/products | `['products']` |
  | `useUpdateProduct` | PATCH /api/products/:id | `['products']`, `['product', id]` |

### Order Queries
- **File:** `src/api/orders.ts`
- **Queries:**
  | Key | Endpoint | Stale Time | Used By |
  |-----|----------|------------|---------|
  | `['orders', page]` | GET /api/orders | 10s | AdminOrdersPage |
  | `['order', id]` | GET /api/orders/:id | 30s | OrderDetailPage |
```

---

## 3. Design System & UI

**Output file:** `design-system.md`
**Covers:** Component library, theming, forms/validation (client-side), i18n, accessibility

### Scanning Strategy

#### Theming

```
Glob: **/theme.ts, **/theme.tsx, **/theme.js, **/tokens/**, **/styles/theme*
Glob: tailwind.config.*, **/tailwind.config.*
Grep: createTheme\(|extendTheme\(|ThemeProvider
```
- MUI: scan `createTheme()` for palette, typography, spacing overrides.
- Tailwind: scan `tailwind.config.*` for custom colors, fonts, spacing.
- Chakra: scan `extendTheme()` for custom tokens.
- Flutter: scan `ThemeData(` for material theme customizations.

#### Shared Components

```
Glob: src/components/**/*.tsx, src/components/**/*.vue, src/components/**/*.svelte
Glob: lib/widgets/**/*.dart, lib/components/**/*.dart
```
- Inventory all shared components. Document name, purpose, key props.
- Identify "base" components (Button, Input, Modal) vs. "composite" (ProductCard, OrderTable).

#### Forms & Validation (Client)

```
Grep: useForm\(|useFormik\(|Formik|useFormContext
Grep: yup\.object|z\.object|zod\.object — client-side schemas
Grep: FormProvider|Controller|Field
Glob: src/**/schema*.ts, src/**/validation*.ts
```
- Document which form library is used and the common pattern.
- List validation schemas and which forms use them.

#### i18n

```
Glob: **/locales/**/*.json, **/messages/**/*.json, **/translations/**/*.json
Glob: **/en.json, **/th.json, **/ja.json
Grep: useTranslation\(|useTranslations\(|t\(|$t\(|AppLocalizations
Grep: next-intl|react-i18next|i18next|flutter_localizations
```
- List supported locales.
- Document how to add new translation keys.
- Note if there is a default/fallback locale.

#### Accessibility

```
Grep: aria-|role="|tabIndex|focus-visible|Semantics\(
Grep: accessibilityLabel|accessibilityRole|accessibilityHint
```

### What to Document

| Area | Details |
|------|---------|
| Theme tokens | Colors (primary, secondary, etc.), spacing scale, typography scale, breakpoints |
| Component inventory | Name, purpose, key props, where used |
| Form patterns | Library, validation approach, common patterns |
| Locales | Supported languages, how to add text, fallback behavior |
| a11y | Patterns in use, known gaps |

### Output Template

```markdown
# Design System & UI
> Read this file BEFORE modifying UI components, theming, forms, or i18n text.

## How to use
Check existing components before creating new ones. Follow theming tokens
for all new styles. Add translations to ALL locale files when adding text.

## Theming

### Library: Tailwind CSS + custom tokens
- **Config:** `tailwind.config.ts`
- **Tokens:**
  | Token | Values |
  |-------|--------|
  | Colors (primary) | `blue-600` (#2563EB) |
  | Colors (secondary) | `gray-700` (#374151) |
  | Colors (accent) | `amber-500` (#F59E0B) |
  | Colors (error) | `red-500` (#EF4444) |
  | Spacing | 4px base (Tailwind default) |
  | Typography | Inter (body), Poppins (headings) |
  | Breakpoints | sm:640, md:768, lg:1024, xl:1280 |
- **Dark mode:** class-based (`dark:` prefix), toggle in Header

## Component Inventory

### Base Components
| Component | File | Purpose | Key Props |
|-----------|------|---------|-----------|
| Button | `src/components/ui/Button.tsx` | Primary action trigger | variant, size, loading, disabled |
| Input | `src/components/ui/Input.tsx` | Text input with label | label, error, helperText |
| Modal | `src/components/ui/Modal.tsx` | Dialog overlay | open, onClose, title, size |
| DataTable | `src/components/ui/DataTable.tsx` | Sortable data grid | columns, data, onSort, pagination |
| Badge | `src/components/ui/Badge.tsx` | Status indicator | variant (success/warning/error), label |

### Composite Components
| Component | File | Used By |
|-----------|------|---------|
| ProductCard | `src/components/ProductCard.tsx` | ProductList, FeaturedProducts, SearchResults |
| OrderTable | `src/components/OrderTable.tsx` | AdminOrders, UserOrders |
| NavigationMenu | `src/components/NavigationMenu.tsx` | Header (all pages) |

## Forms & Validation
- **Library:** React Hook Form + Yup
- **Pattern:** schema-first validation, `useForm({ resolver: yupResolver(schema) })`
- **Schemas:**
  | Schema | File | Used By |
  |--------|------|---------|
  | loginSchema | `src/schemas/auth.ts` | LoginForm |
  | productSchema | `src/schemas/product.ts` | CreateProduct, EditProduct |
  | orderSchema | `src/schemas/order.ts` | CheckoutForm |

## i18n
- **Library:** next-intl
- **Supported locales:** en, th
- **Files:** `messages/en.json`, `messages/th.json`
- **Fallback:** en
- **How to add text:**
  1. Add key to `messages/en.json`
  2. Add translation to `messages/th.json`
  3. Use `const t = useTranslations('namespace')` then `t('key')`
- **IMPORTANT:** Never hardcode user-facing text. Always use translation keys.

## Accessibility
- Focus management: modal trap focus, skip-to-content link
- ARIA: labels on all interactive elements, live regions for notifications
- Keyboard: all actions reachable via keyboard
- Known gaps: DataTable sort not announced to screen readers
```

---

## 4. API & Middleware

**Output file:** `api-middleware.md`
**Covers:** API endpoints, middleware pipeline, server-side validation, error handling

### Scanning Strategy

#### Express / Fastify / Hono

```
Grep: (app|router)\.(get|post|put|patch|delete)\(
Glob: src/routes/**/*.ts, src/controllers/**/*.ts, routes/**/*.ts
Grep: app\.use\( — middleware registration
```
- Scan route files for HTTP method + path.
- Follow controller imports to find handler logic.
- Check middleware chains: `router.get('/path', authMiddleware, validate(schema), controller)`.

#### NestJS

```
Grep: @(Get|Post|Put|Patch|Delete|Controller)\(
Glob: src/**/*.controller.ts, src/**/*.module.ts
Grep: @UseGuards|@UseInterceptors|@UsePipes
```
- Each `@Controller('path')` = route prefix.
- Each `@Get()`, `@Post()` etc. = endpoint.
- Guards and interceptors = middleware equivalent.

#### FastAPI (Python)

```
Grep: @app\.(get|post|put|patch|delete)\(|@router\.(get|post|put|patch|delete)\(
Glob: app/routers/**/*.py, app/api/**/*.py
Grep: Depends\( — dependency injection (middleware equivalent)
```

#### Django (Python)

```
Grep: path\(|re_path\(|urlpatterns
Glob: **/urls.py, **/views.py, **/viewsets.py
Grep: class \w+View|class \w+ViewSet|@api_view
```

#### Rails (Ruby)

```
Grep: (get|post|put|patch|delete|resources|resource)\s
Glob: config/routes.rb, app/controllers/**/*.rb
Grep: before_action|after_action|around_action
```

#### Go (net/http, Gin, Echo, Chi)

```
Grep: (r|router|e)\.(GET|POST|PUT|PATCH|DELETE|Handle|HandleFunc)\(
Glob: **/handler*.go, **/routes*.go, **/server*.go
Grep: func.*http\.Handler|gin\.Context|echo\.Context
```

#### Next.js API Routes

```
Glob: app/api/**/route.ts (App Router)
Glob: pages/api/**/*.ts (Pages Router)
Grep: export.*(GET|POST|PUT|PATCH|DELETE) — App Router named exports
```

#### Validation (Server-side)

```
Grep: z\.object\(|yup\.object\(|Joi\.object\( — schema definitions
Grep: class-validator|IsString|IsNotEmpty|IsEmail — NestJS decorators
Grep: BaseModel|Field\( — Pydantic models
Glob: src/**/schema*.ts, src/**/dto*.ts, src/**/validator*.ts
```

#### Error Handling

```
Grep: class \w+Error extends|class \w+Exception extends — custom error classes
Grep: app\.use\(.*(err|error).*req.*res — Express error middleware
Grep: @Catch\(|ExceptionFilter — NestJS exception filters
Grep: @app\.exception_handler — FastAPI exception handlers
```

### What to Document

**Per endpoint:**

| Field | Description |
|-------|-------------|
| HTTP method + path | e.g., `POST /api/v1/products` |
| File | Controller/route file path |
| Auth | Required auth scheme, role restrictions |
| Request schema | Params, body, query with types |
| Response shape | Success and error response format |
| Side effects | Emails sent, jobs queued, events emitted |
| Middleware chain | Ordered list of middleware applied |

**For middleware pipeline:**

| Field | Description |
|-------|-------------|
| Execution order | Global middleware order as registered |
| Per-middleware | What it does, which routes use it |
| Route-specific | Which middleware chains are applied per route group |

**For error handling:**

| Field | Description |
|-------|-------------|
| Error class hierarchy | Custom error classes and their HTTP status codes |
| Error response format | Standard error response shape |
| Retry/fallback | Any retry or circuit-breaker logic |

### Output Template

```markdown
# API & Middleware
> Read this file BEFORE modifying or creating any API endpoint.

## How to use
Check existing endpoints before creating new ones. Follow the same auth,
validation, and response patterns. Check middleware chains for new routes.

## Endpoint Index

| Method | Path | Auth | Controller | Summary |
|--------|------|------|------------|---------|
| POST | /api/v1/auth/login | public | auth.controller | User login |
| POST | /api/v1/auth/register | public | auth.controller | User registration |
| GET | /api/v1/products | public | product.controller | List products (paginated) |
| GET | /api/v1/products/:id | public | product.controller | Get single product |
| POST | /api/v1/products | admin | product.controller | Create product |
| PATCH | /api/v1/products/:id | admin | product.controller | Update product |
| POST | /api/v1/orders | user | order.controller | Create order |
| GET | /api/v1/orders | user | order.controller | List user orders |
| PATCH | /api/v1/orders/:id/status | admin | order.controller | Update order status |

## Endpoint Details

### POST /api/v1/auth/login
- **File:** `src/controllers/auth.controller.ts`
- **Auth:** public
- **Body:** `{ email: string, password: string }`
- **Response:** `{ user: User, token: string, refreshToken: string }`
- **Logic:** validates credentials, issues JWT + refresh token
- **Side effects:** logs login event
- **Middleware:** [rateLimiter(5/min), validateBody(loginSchema)]

### POST /api/v1/orders
- **File:** `src/controllers/order.controller.ts`
- **Auth:** JWT (user role)
- **Body:** `{ items: [{ productId: string, qty: number }], coupon?: string }`
- **Response:** `{ order: Order, message: "Order created" }`
- **Logic:** validates stock, calculates total, applies coupon, creates order
- **Side effects:** queues `orderConfirmationEmail`, decrements stock
- **Middleware:** [auth, validateBody(orderSchema)]

## Middleware Pipeline

### Global middleware (execution order)
1. `cors()` — CORS headers
2. `helmet()` — security headers
3. `express.json()` — body parsing
4. `requestLogger` — structured request logging
5. `rateLimiter` — global rate limit (100 req/min)

### Route-specific middleware
- `/api/v1/admin/**` — [auth, roleGuard('admin')]
- `/api/v1/user/**` — [auth]
- `/api/v1/public/**` — no auth middleware

## Error Handling

### Custom error classes
| Class | HTTP Status | Usage |
|-------|-------------|-------|
| ValidationError | 400 | Request body/param validation failure |
| UnauthorizedError | 401 | Missing or invalid token |
| ForbiddenError | 403 | Insufficient role/permissions |
| NotFoundError | 404 | Resource not found |
| ConflictError | 409 | Duplicate resource |
| AppError | 500 | Unexpected server error |

### Error response format
```json
{ "message": "Human-readable error description" }
```

### Validation error format
```json
{ "message": "Validation failed", "errors": [{ "field": "email", "message": "Required" }] }
```
```

---

## 5. Data Layer

**Output file:** `data-layer.md`
**Covers:** DB schema, migrations, transactions, caching (Redis), seeding, connection pooling

### Scanning Strategy

#### Prisma

```
Glob: prisma/schema.prisma, **/schema.prisma
Glob: prisma/migrations/**/migration.sql
Grep: model \w+ \{  — model definitions
Grep: enum \w+ \{  — enum definitions
Grep: @@index|@@unique|@@map — indexes and constraints
Grep: \$transaction\( — transaction usage
Glob: prisma/seed.ts, prisma/seed.js
```

#### TypeORM

```
Glob: src/entities/**/*.ts, src/entity/**/*.ts
Grep: @Entity\(|@Column\(|@ManyToOne|@OneToMany|@ManyToMany
Grep: getRepository|createQueryBuilder
Glob: src/migrations/**/*.ts
```

#### Drizzle

```
Glob: drizzle.config.ts, src/db/schema*.ts
Grep: pgTable\(|mysqlTable\(|sqliteTable\(
Grep: drizzle\(|db\.select|db\.insert|db\.update
```

#### Django ORM

```
Glob: **/models.py
Grep: class \w+\(models\.Model\)|CharField|ForeignKey|ManyToManyField
Glob: **/migrations/**/*.py
```

#### ActiveRecord (Rails)

```
Glob: app/models/**/*.rb, db/migrate/**/*.rb
Grep: has_many|belongs_to|has_one|has_and_belongs_to_many
Grep: create_table|add_column|add_index
```

#### SQLAlchemy (Python)

```
Glob: **/models.py, **/models/**/*.py
Grep: class \w+\(Base\)|Column\(|relationship\(|ForeignKey\(
Glob: alembic/versions/**/*.py
```

#### Redis / Caching

```
Grep: Redis\(|createClient|ioredis|new Redis
Grep: \.set\(|\.get\(|\.del\(|\.expire\(|\.hset\(
Grep: cache\.get|cache\.set|@Cacheable|@cache
```
- Document cache key patterns, TTL values, invalidation triggers.

#### Elasticsearch

```
Grep: Client\(.*elastic|@elastic/elasticsearch|elasticsearch-py
Grep: \.index\(|\.search\(|\.bulk\(
```

### What to Document

| Area | Details |
|------|---------|
| Models/tables | Name, key fields (with types), relations, important constraints |
| Enums | Name, values, which model uses them |
| Indexes | Non-default indexes and why they exist |
| Migrations | Strategy (auto-generated vs manual), naming convention |
| Cache strategy | What is cached, key pattern, TTL, invalidation triggers |
| Transactions | Where transactions are used and what they protect |
| Seeds/fixtures | What seed data exists, how to run seeds |

### Output Template

```markdown
# Data Layer
> Read this file BEFORE modifying database schema, caching, or writing queries.

## How to use
Check relations before adding fields. Check indexes before adding queries.
Check cache keys before adding caching logic.

## ORM: Prisma 7.3.0
- **Schema:** `prisma/schema.prisma`
- **Migrations:** `prisma/migrations/` (YYYYMMDDHHMMSS_description)
- **Client:** auto-generated via `prisma generate`
- **DB push (dev):** `make db-push` (no migration file)
- **Migrate (prod):** `make migrate` (creates migration file)

## Schema Overview

### User
| Field | Type | Constraints |
|-------|------|-------------|
| id | String (cuid) | PK |
| email | String | unique, required |
| name | String | required |
| password | String | hashed (bcrypt) |
| role | Role (enum) | default: USER |
| createdAt | DateTime | auto |
| updatedAt | DateTime | auto-update |

**Relations:** hasMany Order, hasMany Review
**Indexes:** email (unique), role

### Product
| Field | Type | Constraints |
|-------|------|-------------|
| id | String (cuid) | PK |
| name | String | required |
| slug | String | unique |
| price | Decimal | required |
| stock | Int | default: 0 |
| categoryId | String | FK -> Category |
| isActive | Boolean | default: true |

**Relations:** belongsTo Category, hasMany OrderItem, hasMany Review
**Indexes:** slug (unique), categoryId, [name, categoryId] (composite)

### Order
| Field | Type | Constraints |
|-------|------|-------------|
| id | String (cuid) | PK |
| userId | String | FK -> User |
| status | OrderStatus (enum) | default: PENDING |
| total | Decimal | required |
| paidAt | DateTime? | nullable |

**Relations:** belongsTo User, hasMany OrderItem
**Indexes:** userId, status, [userId, status] (composite)

## Enums

| Enum | Values | Used By |
|------|--------|---------|
| Role | USER, ADMIN | User.role |
| OrderStatus | PENDING, PAID, SHIPPED, DELIVERED, CANCELLED | Order.status |
| PaymentMethod | CREDIT_CARD, BANK_TRANSFER, COD | Payment.method |

## Cache Strategy (Redis)
| Key Pattern | TTL | Invalidation | Purpose |
|-------------|-----|-------------|---------|
| `product:{id}` | 60s | on product update/delete | Single product cache |
| `products:list:{hash}` | 30s | on any product mutation | Paginated product list |
| `user:session:{token}` | 24h | on logout | Session data |
| `rate:ip:{ip}` | 60s | auto-expire | Rate limit counter |

## Transaction Patterns
- **Order creation:** wraps stock decrement + order insert + payment record in `$transaction`
- **User deletion:** wraps user delete + related data cleanup in `$transaction`

## Seeds
- **File:** `prisma/seed.ts`
- **Run:** `make db-seed`
- **Creates:** admin user, sample categories, sample products
```

---

## 6. Auth & Security

**Output file:** `auth-security.md`
**Covers:** Authentication, authorization, rate limiting, CORS, secrets, security headers

### Scanning Strategy

#### Authentication

```
Grep: jwt\.sign\(|jwt\.verify\(|jsonwebtoken — JWT-based auth
Grep: NextAuth|getServerSession|getSession — NextAuth.js
Grep: passport\.use\(|passport\.authenticate — Passport.js
Grep: @auth/core|Auth\( — Auth.js v5
Grep: OAuth2|GoogleProvider|GithubProvider — OAuth providers
Grep: signIn\(|signOut\( — auth actions (various frameworks)
Glob: src/**/auth*.ts, src/**/auth*.tsx, src/middleware/auth*
```

#### Authorization

```
Grep: role|permission|rbac|abac|can\(|authorize
Grep: @Roles\(|@Permissions\(|@UseGuards\(RoleGuard — NestJS
Grep: roleMiddleware|requireRole|isAdmin|checkPermission
Glob: src/**/guard*.ts, src/**/permission*.ts, src/**/role*.ts
```

#### Rate Limiting

```
Grep: rateLimit|RateLimit|rate-limit|rateLimiter|throttle|Throttle
Grep: express-rate-limit|@nestjs/throttler|slowapi
```

#### CORS

```
Grep: cors\(|CORS|Access-Control-Allow|allowedOrigins
```

#### Security Headers

```
Grep: helmet\(|CSP|Content-Security-Policy|X-Frame-Options|HSTS
Grep: Strict-Transport-Security|X-Content-Type-Options
```

#### Secrets / Environment

```
Glob: .env.example, .env.sample, environments/**
Grep: process\.env\.|os\.environ|ENV\[|Deno\.env
```
- List expected secrets WITHOUT values.
- Note how secrets are loaded (dotenv, vault, AWS SSM, etc.).

### What to Document

| Area | Details |
|------|---------|
| Auth schemes | JWT / OAuth2 / session / API keys — how each works |
| Token structure | Payload fields, expiry, refresh strategy |
| Role/permission model | All roles, what each can do, how checked |
| Per-endpoint auth | Cross-reference with api-middleware.md |
| Rate limits | Global and per-endpoint rules |
| CORS policy | Allowed origins, methods, headers |
| Security headers | What is configured and why |
| Secrets inventory | Expected env vars (without values) |

### Output Template

```markdown
# Auth & Security
> Read this file BEFORE modifying auth, permissions, rate limits, or security config.

## How to use
Check auth requirements before adding endpoints. Check role model before
adding authorization logic. Never hardcode secrets.

## Authentication

### Primary: JWT (access + refresh)
- **Library:** jsonwebtoken
- **Access token:** 15min expiry, payload: `{ userId, role, email }`
- **Refresh token:** 7d expiry, stored in httpOnly cookie
- **Flow:**
  1. POST /api/auth/login -> returns { accessToken } + sets refreshToken cookie
  2. Client sends `Authorization: Bearer {accessToken}` header
  3. Expired -> POST /api/auth/refresh (auto via axios interceptor)
  4. Refresh expired -> redirect to login

### OAuth Providers (if any)
- Google: client ID in `GOOGLE_CLIENT_ID` env var
- Login flow: redirect -> callback -> issue local JWT

## Authorization

### Role Model
| Role | Can Do |
|------|--------|
| USER | browse products, create orders, view own orders, write reviews |
| ADMIN | all USER permissions + manage products, manage orders, manage users, view analytics |

### How roles are checked
- **Middleware:** `roleMiddleware(['admin'])` on route definition
- **File:** `src/middleware/role.middleware.ts`
- **Pattern:** decode JWT -> check `role` field -> 403 if insufficient

## Rate Limiting
| Scope | Limit | Window | Config |
|-------|-------|--------|--------|
| Global | 100 requests | 1 min | `src/middleware/rateLimiter.ts` |
| Login | 5 attempts | 1 min | per IP, stricter |
| Register | 3 attempts | 5 min | per IP |
| File upload | 10 requests | 1 min | per user |

## CORS
- **Allowed origins:** `http://localhost:3000`, `https://app.example.com`
- **Allowed methods:** GET, POST, PUT, PATCH, DELETE, OPTIONS
- **Credentials:** true (cookies)
- **Config:** `src/config/cors.ts`

## Security Headers (Helmet)
- X-Content-Type-Options: nosniff
- X-Frame-Options: DENY
- Strict-Transport-Security: max-age=31536000; includeSubDomains
- Content-Security-Policy: default-src 'self'
- Referrer-Policy: strict-origin-when-cross-origin

## Secrets Inventory
| Variable | Purpose | Required |
|----------|---------|----------|
| DATABASE_URL | PostgreSQL connection string | yes |
| JWT_SECRET | Token signing key | yes |
| JWT_REFRESH_SECRET | Refresh token signing key | yes |
| REDIS_URL | Redis connection | yes |
| GOOGLE_CLIENT_ID | OAuth login | if OAuth enabled |
| GOOGLE_CLIENT_SECRET | OAuth login | if OAuth enabled |
| AWS_ACCESS_KEY_ID | S3 file storage | yes |
| AWS_SECRET_ACCESS_KEY | S3 file storage | yes |
| S3_BUCKET | Storage bucket name | yes |
| SMTP_HOST | Email delivery | yes |
```

---

## 7. Background & Events

**Output file:** `background-events.md`
**Covers:** Background jobs, event-driven architecture, scheduled tasks, service-to-service communication

### Scanning Strategy

#### BullMQ / Bull (Node.js)

```
Grep: Queue\(|new Queue|new Worker|queue\.add\(
Grep: process\(|defineJob\(|createBullBoard
Glob: src/queue*/**/*.ts, src/jobs/**/*.ts, src/workers/**/*.ts
Glob: src/infrastructure/queue/**/*.ts
```
- Document queue names, processor files, job options (retries, backoff, delay).

#### Celery (Python)

```
Grep: @app\.task|@shared_task|celery_app
Grep: \.delay\(|\.apply_async\(
Glob: **/tasks.py, **/tasks/**/*.py
```

#### Sidekiq (Ruby)

```
Grep: include Sidekiq::Worker|include Sidekiq::Job|perform_async
Glob: app/workers/**/*.rb, app/jobs/**/*.rb
```

#### Event-Driven

```
Grep: EventEmitter|eventBus|emit\(|on\(|subscribe\(|publish\(
Grep: @OnEvent|@EventPattern — NestJS
Grep: pubsub|PubSub|nats|rabbitmq|kafka
```

#### Scheduled Tasks

```
Grep: cron|@Cron|schedule\.|CronJob|node-cron|agenda
Grep: vercel\.json.*cron — Vercel Cron
Glob: **/cron*.ts, **/scheduler*.ts, **/schedule*.ts
```

#### Service-to-Service

```
Grep: axios\.create\(|fetch\(.*http|httpClient|HttpService
Grep: grpc|gRPC|proto|\.proto
Glob: src/clients/**/*.ts, src/services/external/**/*.ts
```

### What to Document

**Per job/task:**

| Field | Description |
|-------|-------------|
| Name | Job or task name |
| File | Processor/handler file path |
| Queue/topic | Which queue or topic |
| Trigger | What causes this job to be enqueued |
| Payload | Expected data shape |
| Logic | What the job does |
| Retries | Max retries, backoff strategy |
| Dependencies | Other services or jobs it calls |

**Per scheduled task:**

| Field | Description |
|-------|-------------|
| Name | Task name |
| Schedule | Cron expression + human-readable |
| Logic | What it does |

**Per service communication:**

| Field | Description |
|-------|-------------|
| Service name | External service |
| Purpose | Why it is called |
| Protocol | HTTP / gRPC / event |
| Endpoints called | Specific endpoints |
| Error handling | Retry, fallback, circuit breaker |

### Output Template

```markdown
# Background & Events
> Read this file BEFORE modifying background jobs, events, or service communication.

## How to use
Check existing jobs before creating new ones. Follow the same retry and
error handling patterns. Check queue names to avoid conflicts.

## Job Infrastructure
- **Library:** BullMQ
- **Backend:** Redis (`REDIS_URL`)
- **Dashboard:** BullBoard at /admin/queues (admin only)
- **Worker process:** separate process (`yarn worker:dev`)

## Job Definitions

### orderConfirmationEmail
- **Queue:** `emailQueue`
- **File:** `src/infrastructure/queue/processors/email.processor.ts`
- **Trigger:** order creation (from `order.service.ts`)
- **Payload:** `{ orderId: string, userEmail: string, orderTotal: number }`
- **Logic:** fetches order details, renders email template, sends via SMTP
- **Retries:** 3, exponential backoff (1s, 4s, 16s)
- **Dependencies:** SMTP service

### orderExpire
- **Queue:** `orderQueue`
- **File:** `src/infrastructure/queue/processors/order.processor.ts`
- **Trigger:** delayed job added on order creation (30 min delay)
- **Payload:** `{ orderId: string }`
- **Logic:** if order still PENDING after 30 min, cancel and restore stock
- **Retries:** 2, fixed 30s backoff
- **Dependencies:** none

### syncProductIndex
- **Queue:** `searchQueue`
- **File:** `src/infrastructure/queue/processors/search.processor.ts`
- **Trigger:** product create/update/delete
- **Payload:** `{ productId: string, action: 'upsert' | 'delete' }`
- **Logic:** sync product data to Elasticsearch index
- **Retries:** 5, exponential backoff
- **Dependencies:** Elasticsearch

## Scheduled Tasks

| Task | Schedule | File | Logic |
|------|----------|------|-------|
| cleanExpiredTokens | 0 2 * * * (daily 2am) | `src/cron/tokenCleanup.ts` | Delete refresh tokens older than 30 days |
| generateDailyReport | 0 6 * * * (daily 6am) | `src/cron/dailyReport.ts` | Aggregate yesterday's orders, email to admins |

## Service Communication

### SML Integration App
- **Purpose:** pull product data from SML REST API
- **Protocol:** HTTP (internal service)
- **Base URL:** `SML_INTEGRATION_URL` env var
- **Endpoints called:**
  - GET /api/products — bulk product sync
  - GET /api/products/:sku — single product lookup
- **Error handling:** retry 3x with 5s backoff, log failure, continue with next product
- **Runs:** as independent cron job (separate service)

## Event Bus (if applicable)

| Event | Emitted By | Listeners | Payload |
|-------|------------|-----------|---------|
| order.created | OrderService | EmailService, InventoryService | `{ orderId, items }` |
| user.registered | AuthService | EmailService | `{ userId, email }` |
| product.updated | ProductService | SearchIndexer, CacheInvalidator | `{ productId }` |
```

---

## 8. Integrations & Infrastructure

**Output file:** `integrations-infra.md`
**Covers:** External APIs, file storage, logging, config management, health checks, graceful shutdown

### Scanning Strategy

#### External APIs / 3rd-party SDKs

```
Grep: Stripe|stripe|Omise|omise — payment
Grep: Twilio|twilio|SendGrid|sendgrid|@sendgrid — messaging
Grep: aws-sdk|@aws-sdk|S3Client|SESClient — AWS services
Grep: firebase|Firebase|initializeApp — Firebase
Grep: google-auth|googleapis — Google APIs
Grep: line-bot-sdk|LINE — LINE integration
Glob: src/services/external/**/*.ts, src/integrations/**/*.ts, src/lib/clients/**/*.ts
```

#### File Storage

```
Grep: S3Client|PutObjectCommand|GetObjectCommand|getSignedUrl
Grep: multer|formidable|busboy — upload middleware
Grep: MinIO|minio|Minio
Grep: GCS|Storage\(\)|@google-cloud/storage
Glob: src/**/upload*.ts, src/**/storage*.ts, src/**/file*.ts
```

#### Logging

```
Grep: winston|createLogger|Logger — Winston
Grep: pino|pinoHttp — Pino
Grep: morgan — Morgan (HTTP logging)
Grep: structlog|logging\.getLogger — Python logging
Grep: log\.Printf|slog\. — Go logging
Glob: src/**/logger*.ts, src/config/logger*.ts
```
- Document: library, log levels, format (structured/text), destinations (console, file, service).

#### Config / Environment

```
Glob: .env.example, environments/**, src/config/**/*.ts
Grep: z\.object.*process\.env|envSchema|validateEnv — Zod env validation
Grep: process\.env\.\w+ — all env var access points
```

#### Health Checks

```
Grep: /health|/ready|/live|healthCheck|readinessProbe|livenessProbe
Glob: src/**/health*.ts
```

#### Graceful Shutdown

```
Grep: SIGTERM|SIGINT|process\.on\(|gracefulShutdown|beforeExit
```

### What to Document

| Area | Details |
|------|---------|
| External services | Name, purpose, auth method (API key, OAuth, etc.), SDK used |
| File storage | Backend, upload limits, file types allowed, signed URL config |
| Logging | Library, format, levels, destinations, structured fields |
| Config | All env vars with purpose, defaults, which are required |
| Health checks | Endpoint, what it verifies, expected response |
| Shutdown | Signal handlers, connection cleanup order |

### Output Template

```markdown
# Integrations & Infrastructure
> Read this file BEFORE modifying external integrations, logging, or environment config.

## How to use
Check existing integrations before adding new external services. Follow the
same auth patterns. Check env vars before adding new config.

## External Services

| Service | Purpose | Auth | SDK/Client |
|---------|---------|------|------------|
| Omise | Payment processing | API key (secret key) | `omise-node` |
| SendGrid | Transactional email | API key | `@sendgrid/mail` |
| AWS S3 | File storage (production) | IAM credentials | `@aws-sdk/client-s3` |
| MinIO | File storage (development) | access/secret key | `@aws-sdk/client-s3` (S3-compatible) |
| LINE Notify | Order notifications | Bearer token | HTTP client |
| Elasticsearch | Product search | no auth (internal) | `@elastic/elasticsearch` |

### Omise (Payment)
- **Config:** `src/services/payment/omise.service.ts`
- **Auth:** `OMISE_SECRET_KEY` env var
- **Used by:** checkout flow (order creation)
- **Webhook:** POST /api/webhooks/omise — handles charge.complete, charge.fail
- **Test mode:** uses test keys from env (prefixed `skey_test_`)

### SendGrid (Email)
- **Config:** `src/services/email/sendgrid.service.ts`
- **Auth:** `SENDGRID_API_KEY` env var
- **Templates:** order confirmation, welcome email, password reset
- **Used by:** background jobs (emailQueue)

## File Storage
- **Development:** MinIO (Docker, port 9000, console 9001)
- **Production:** AWS S3
- **Client:** `@aws-sdk/client-s3` (works with both MinIO and S3)
- **Buckets:** `uploads` (user files), `assets` (static assets)
- **Upload limits:** 10MB per file, allowed types: jpg, png, webp, pdf
- **Signed URLs:** 1h expiry for private files
- **Config:** `src/config/storage.ts`

## Logging
- **Library:** Winston
- **Format:** JSON (structured) in production, colorized text in development
- **Levels:** error, warn, info, http, debug
- **Destinations:**
  - Console (all environments)
  - File: `logs/error.log` (error only), `logs/combined.log` (all)
- **Structured fields:** `{ timestamp, level, message, service, requestId, userId }`
- **Config:** `src/config/logger.ts`
- **HTTP logging:** Morgan middleware -> Winston transport

## Config / Environment

### Required variables
| Variable | Purpose | Default |
|----------|---------|---------|
| NODE_ENV | Environment mode | development |
| PORT | API server port | 4000 |
| DATABASE_URL | PostgreSQL connection | - |
| REDIS_URL | Redis connection | redis://localhost:6379 |
| JWT_SECRET | Token signing | - |
| JWT_REFRESH_SECRET | Refresh token signing | - |
| OMISE_SECRET_KEY | Payment API key | - |
| SENDGRID_API_KEY | Email API key | - |
| S3_ENDPOINT | Storage endpoint | http://localhost:9000 |
| S3_BUCKET | Storage bucket | uploads |
| S3_ACCESS_KEY | Storage auth | - |
| S3_SECRET_KEY | Storage auth | - |

### Environment files
- `environments/development` — local dev
- `environments/ci` — GitHub Actions
- `environments/uat` — UAT deployment
- `environments/production` — production
- Makefile commands auto-copy to `.env`

## Health Checks

### GET /health
- **Response:** `{ status: "ok", uptime: 12345, timestamp: "ISO-8601" }`
- **Checks:** server is running (no dependency checks)

### GET /health/ready
- **Response:** `{ status: "ok", db: "connected", redis: "connected", elasticsearch: "connected" }`
- **Checks:** PostgreSQL connection, Redis ping, Elasticsearch cluster health
- **Used by:** Kubernetes readiness probe

## Graceful Shutdown
1. Receive SIGTERM / SIGINT
2. Stop accepting new HTTP connections
3. Wait for in-flight requests (30s timeout)
4. Close BullMQ workers (wait for active jobs)
5. Close Redis connection
6. Close database pool
7. Exit process
- **Config:** `src/server.ts`
```

---

## Extended Feature Map Format

The feature map (`_feature-map.md`) cross-references all 8 categories per feature.
Group by business feature, not by technical layer.

### How to Build the Feature Map

1. Start from pages/screens (Category 1) — each page is usually one feature or part of one.
2. For each page, trace its API calls (Category 4).
3. For each API endpoint, trace its DB models (Category 5).
4. Link state stores (Category 2) that the page uses.
5. Link UI components (Category 3) that are shared across features.
6. Link auth requirements (Category 6).
7. Link any background jobs triggered (Category 7).
8. Link any external integrations involved (Category 8).

### Template

```markdown
# Feature Map
> Read this file WHEN you need to understand how a feature works end-to-end.
> Read the specific feature section — you rarely need the whole file.

## How to use
Find the feature you're working on. Follow the connections across all layers.
This tells you everything affected by your change.

## Features

### Product Catalog

**Frontend:**
- Pages/Screens: /products -> `ProductListPage`, /products/[id] -> `ProductDetailPage`
- State: `productApi` (React Query), `filterStore` (Zustand)
- Components: ProductCard, FilterSidebar, Pagination, ProductGallery

**Backend:**
- API: GET /api/v1/products, GET /api/v1/products/:id, POST /api/v1/products (admin)
- Middleware: public (list/detail), auth + admin (create/update/delete)
- Jobs: `syncProductIndex` (Elasticsearch sync on product change)
- Integrations: Elasticsearch (search), S3/MinIO (product images)

**Data:**
- DB: Product, Category, ProductImage, Review
- Cache: `product:{id}` (60s), `products:list:{hash}` (30s)

**Auth:** browsing is public, management requires admin role

---

### Order Management

**Frontend:**
- Pages/Screens: /cart -> `CartPage`, /checkout -> `CheckoutPage`, /orders -> `OrdersPage`
- State: `cartSlice` (Redux), `orderApi` (React Query)
- Components: CartItem, OrderTable, PaymentForm, StatusBadge

**Backend:**
- API: POST /api/v1/orders, GET /api/v1/orders, PATCH /api/v1/orders/:id/status
- Middleware: auth (all), admin (status update)
- Jobs: `orderConfirmationEmail`, `orderExpire` (30 min timeout)
- Integrations: Omise (payment), SendGrid (email), LINE Notify (admin alert)

**Data:**
- DB: Order, OrderItem, Payment
- Cache: none (always fresh)

**Auth:** requires login (role: user), admin for status management

---

### User Authentication

**Frontend:**
- Pages/Screens: /login -> `LoginPage`, /register -> `RegisterPage`
- State: `authSlice` (Redux, persisted)
- Components: LoginForm, RegisterForm

**Backend:**
- API: POST /api/v1/auth/login, POST /api/v1/auth/register, POST /api/v1/auth/refresh
- Middleware: rate limiter (login: 5/min, register: 3/5min)
- Jobs: none
- Integrations: none (or OAuth provider if applicable)

**Data:**
- DB: User
- Cache: `user:session:{token}` (24h)

**Auth:** login/register are public, refresh requires valid refresh token
```

---

## Extended Unmapped Section

After building the feature map, identify orphans in each category.
These indicate dead code, WIP features, or missing connections.

### Template

```markdown
## Unmapped

### Orphan Endpoints (API exists, no page/screen calls it)
- GET /api/v1/analytics/export — possibly admin-only CSV export, no UI page found
- POST /api/v1/webhooks/stripe — webhook endpoint, called by Stripe not by frontend

### Orphan Pages/Screens (exists, calls no API)
- /about — static content page, no API calls
- /maintenance — placeholder page, appears to be WIP

### Orphan State (store/slice defined, no consumer reads it)
- `notificationSlice` — defined in `src/store/notificationSlice.ts`, no component imports it

### Orphan Jobs (job processor defined, never queued)
- `reportGeneration` — processor exists at `src/jobs/report.processor.ts`, no `queue.add` call found

### Dead Routes (defined in router but component is empty/placeholder)
- /settings — route exists, component renders "Coming soon"

### Unused DB Tables (not referenced by any endpoint or service)
- `AuditLog` — table exists in schema, no service writes to it
```

### Orphan Classification Rules

| Orphan Type | Detection Method |
|-------------|-----------------|
| Orphan Endpoints | Endpoint found in Category 4 but no fetch/axios call matches it in Category 1 |
| Orphan Pages | Page found in Category 1 but makes no API calls (may be legitimate: static pages) |
| Orphan State | Store/slice found in Category 2 but no page/screen imports it |
| Orphan Jobs | Job processor found in Category 7 but no `queue.add` call references the queue |
| Dead Routes | Route defined but component renders placeholder or empty content |
| Unused DB Tables | Model in Category 5 but no service/controller references the table |

Mark legitimate orphans (webhooks, static pages) with a note explaining why they are unmapped.
Do not flag them as problems.

---

## audit-meta.json Schema

Track audit coverage and orphans per category.
Write this file after completing all category scans.

```json
{
  "last_audit": "2026-03-31T14:00:00Z",
  "git_commit": "abc1234def5678",
  "categories": {
    "pages-routing": {
      "detected": true,
      "pages": 55,
      "screens": 0,
      "routes": 55,
      "layouts": 4,
      "deep_links": 0
    },
    "state-data": {
      "detected": true,
      "stores": 6,
      "queries": 12,
      "mutations": 8,
      "persisted": 2
    },
    "design-system": {
      "detected": true,
      "components": 45,
      "locales": 2,
      "forms": 8,
      "theme_library": "tailwindcss"
    },
    "api-middleware": {
      "detected": true,
      "endpoints": 172,
      "middleware": 8,
      "error_classes": 3,
      "validation_schemas": 15
    },
    "data-layer": {
      "detected": true,
      "models": 29,
      "enums": 7,
      "migrations": 15,
      "cache_keys": 10,
      "orm": "prisma"
    },
    "auth-security": {
      "detected": true,
      "auth_schemes": 1,
      "roles": 2,
      "rate_limits": 3,
      "cors_origins": 2,
      "secrets_count": 12
    },
    "background-events": {
      "detected": true,
      "jobs": 5,
      "events": 3,
      "scheduled": 2,
      "queues": 3,
      "job_library": "bullmq"
    },
    "integrations-infra": {
      "detected": true,
      "external_services": 4,
      "storage_backends": 1,
      "log_library": "winston",
      "log_level": "info",
      "health_endpoints": 2,
      "env_vars_required": 14
    }
  },
  "orphans": {
    "endpoints": 2,
    "pages": 1,
    "state": 1,
    "jobs": 1,
    "dead_routes": 1,
    "unused_tables": 1
  },
  "features_mapped": 8,
  "skipped_categories": []
}
```

### Field Descriptions

| Field | Type | Description |
|-------|------|-------------|
| `last_audit` | ISO-8601 | When the audit was last run |
| `git_commit` | string | Commit hash at time of audit |
| `categories.{name}.detected` | boolean | Whether this category was found in the project |
| `categories.{name}.*` | number/string | Category-specific stats (counts, library names) |
| `orphans.*` | number | Count of unmapped items per orphan type |
| `features_mapped` | number | Total features in the feature map |
| `skipped_categories` | string[] | Categories that were not detected and were skipped |

### When a Category Is Skipped

If a category is not detected, record it like this:

```json
{
  "categories": {
    "background-events": {
      "detected": false,
      "skipped": true,
      "reason": "No queue library in dependencies, no job/worker files found"
    }
  },
  "skipped_categories": ["background-events"]
}
```

This allows `/onboard-update` to re-check skipped categories if new
dependencies are added.
