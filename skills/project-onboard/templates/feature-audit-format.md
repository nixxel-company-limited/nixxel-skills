# Feature Audit Output Format

Templates for Phase 3 (Feature Audit) output files.
Read this file BEFORE running the feature audit.

## _feature-map.md (read this first)

The feature map is the cross-reference. It connects pages, endpoints,
and database tables into coherent features.

```markdown
# Feature Map
> Read this file WHEN you need to understand how a feature works end-to-end.
> Read specific feature sections — you rarely need the whole file.

## How to use
Find the feature you're working on. Follow the connections:
Page → API → DB. This tells you what's affected by your change.

## Features

### {Feature Name}
**Pages:** /path → `ComponentFile.tsx`
**API:** GET /api/resource, POST /api/resource
**DB:** TableA, TableB (via relation)
**Logic:** {1-2 sentence description of what it does}

### {Feature Name}
...

## Unmapped

### Orphan Endpoints (API exists, no page calls it)
- {method} {path} — {possible reason}

### Dead Pages (page exists, calls no API)
- {path} — {possible reason: static page? WIP?}

### Unused Tables (not referenced by any endpoint)
- {table} — {possible reason}
```

---

## api-endpoints.md

```markdown
# API Endpoints
> Read this file BEFORE modifying or creating any API endpoint.

## How to use
Check existing endpoints before creating new ones.
Follow the same patterns for auth, validation, and response shape.

## Endpoint Index

| Method | Path | Auth | Summary |
|--------|------|------|---------|
| GET | /api/users | JWT | List users |
| POST | /api/users | JWT+Admin | Create user |
| ... | ... | ... | ... |

## Endpoint Details

### GET /api/users
- **File:** `src/app/api/users/route.ts`
- **Auth:** JWT required
- **Query params:** `page` (number), `limit` (number), `search` (string)
- **Response:** `{ data: User[], total: number, page: number }`
- **Logic:** Fetches paginated users with optional text search
- **Side effects:** None

### POST /api/users
- **File:** `src/app/api/users/route.ts`
- **Auth:** JWT + admin role
- **Body:** `{ name: string, email: string, role: "admin" | "user" }`
- **Response:** `{ data: User }`
- **Logic:** Creates user, hashes password, sends welcome email
- **Side effects:** Sends email via {service}

## Common Patterns
- Auth: {how auth works across endpoints}
- Validation: {how validation is handled}
- Error format: {standard error response shape}
- Pagination: {standard pagination pattern}
```

---

## db-schema.md

```markdown
# Database Schema
> Read this file BEFORE modifying database schema or writing queries.

## How to use
Check relations before adding fields. Check indexes before adding queries.

## Schema Overview

### {TableName}
| Field | Type | Notes |
|-------|------|-------|
| id | String (cuid) | PK |
| name | String | required |
| email | String | unique |
| roleId | String | FK → Role |
| createdAt | DateTime | auto |

**Relations:**
- belongsTo Role (roleId)
- hasMany Post

**Indexes:**
- email (unique)
- roleId

### {TableName}
...

## Enums

### {EnumName}
Values: VALUE_A, VALUE_B, VALUE_C
Used by: TableName.fieldName

## Key Relations Diagram
{text-based diagram showing main entity relationships}

User ──hasMany──▸ Post
User ──hasMany──▸ Comment
Post ──hasMany──▸ Comment
Post ──belongsTo──▸ Category
```

---

## pages-views.md

```markdown
# Pages & Views
> Read this file BEFORE modifying or creating any page or component.

## How to use
Check which APIs a page uses before changing them.
Check which pages use an API before changing the API.

## Page Index

| Route | Component | API Calls | Key Components |
|-------|-----------|-----------|----------------|
| / | `HomePage` | GET /api/stats | Dashboard, Chart |
| /users | `UsersPage` | GET /api/users | DataTable, Filter |
| /users/[id] | `UserDetail` | GET /api/users/:id | Form, Avatar |
| ... | ... | ... | ... |

## Page Details

### / → HomePage
- **File:** `src/app/page.tsx`
- **API calls:** GET /api/stats, GET /api/recent-activity
- **Key components:** Dashboard, StatCard, ActivityFeed
- **Features:** Shows overview stats and recent activity
- **State management:** {React Query / SWR / useState}

### /users → UsersPage
...

## Shared Components
{components used across multiple pages}
- `DataTable` — used by: /users, /posts, /reports
- `SearchFilter` — used by: /users, /posts
```

---

## audit-meta.json

```json
{
  "last_audit": "2026-02-27T12:00:00Z",
  "git_commit": "abc1234",
  "coverage": {
    "endpoints_found": 12,
    "endpoints_documented": 12,
    "pages_found": 8,
    "pages_documented": 8,
    "db_tables": 6,
    "features_mapped": 5,
    "orphan_endpoints": 1,
    "dead_pages": 0,
    "unused_tables": 1
  }
}
```

---

## Scanning Strategy

When scanning for endpoints, pages, and DB references, use
framework-specific patterns:

### Next.js App Router
- Routes: `src/app/**/route.ts` (API), `src/app/**/page.tsx` (pages)
- Method: exported function name = HTTP method (GET, POST, PUT, DELETE)

### Next.js Pages Router
- Routes: `src/pages/api/**/*.ts` (API), `src/pages/**/*.tsx` (pages)
- Method: check `req.method` in handler

### Express / Fastify / Hono
- Scan for: `app.get()`, `app.post()`, `router.get()`, etc.
- Or scan route registration files

### Flutter
- Scan for: screen files, route definitions, API service files

### Generic
- Search for fetch/axios/http calls in page/component files
- Match URL patterns to known endpoints

The goal is completeness, not perfection. It's better to document
90% of endpoints with some gaps noted than to miss entire sections.
Note uncertainty: "This endpoint may have additional query params
not captured from static analysis."
