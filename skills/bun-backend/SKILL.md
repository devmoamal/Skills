---
name: bun-backend
description: Enforce high-performance, modular, and type-safe backend architecture for Bun using Hono, Drizzle ORM, Zod, and a strict 3-tier vertical slice (Route -> Service -> Repository) design. Use whenever designing, building, scaffolding, refactoring, or auditing Bun backend APIs, services, routes, database schemas, or middlewares.
---

# Bun Backend: Production Architecture & Clean Code Standards

Build high-performance, modular, bulletproof, and exceptionally clean backend services powered by Bun, Hono, Drizzle ORM, Zod, and Redis.

```
Request ──> Middlewares (CORS, RequestId, Logger, RateLimit, Auth)
                    ↓
              Feature Route (Inline Handlers + Zod Validation)
                    ↓
              Domain Service (Pure Business Logic + Cache + db.transaction + AppError)
                    ↓
              Drizzle Repository (Relational Reads + Query Builder Writes + Soft Deletes)
                    ↓
              Feature Mapper (Strict DTO Transformation — Zero Sensitive Leaks)
                    ↓
              Unified Response Envelope: Response.success(c, safeData)
```

Every endpoint, query, service, background job, and WebSocket must adhere to uncompromising type safety, strict separation of concerns, and clean production hygiene.

---

## Operating Modes

### 1. Build Mode (Default)
When scaffolding new features, endpoints, or services:
* Structure all domain features under `src/features/<feature>/` with explicit `routes/`, `services/`, `repositories/`, `schemas/`, and `mappers/`.
* Strictly enforce the 3-tier boundary: **Route $\rightarrow$ Service $\rightarrow$ Repository**. Never bypass layers.
* Always initialize routers with `new Hono<AppEnv>()` importing `AppEnv` from `@/types/context`.
* Keep route handlers inline in `routes/index.ts` for maximum clarity and direct middleware association.
* Validate all request inputs (body, params, query) using `validateBody`, `validateParams`, `validateQuery` before reaching business logic.
* Sanitize all inputs in Zod schemas: enforce `.trim()` and `.toLowerCase()` on email/string inputs, and convert empty strings `""` to `undefined` or `null`.
* Primary Keys: Standardize on **`uuidv7`** for entity tables to prevent B-tree fragmentation; use `integer` strictly for small lookup or sequence tables.
* Soft Deletes: Apply `softDelete` helper (`deletedAt`) to all primary business entities. Repositories must filter `isNull(table.deletedAt)` by default.
* Mandatory Database Indexes: Index all foreign keys, `deletedAt`, and use partial unique indexes for soft-deleted entities (e.g. unique email where `deleted_at IS NULL`).
* Dedicated Feature Mappers: Transform all database models into safe client responses using `mappers/`. Never return raw database records directly.
* Caching: Use type-safe `cache.getOrSet(key, ttl, fetcher)` from `@/lib/cache`, with explicit invalidation in services upon writes.
* File Uploads: Use direct-to-storage presigned URLs (S3/Cloudflare R2) via `@/lib/storage`, keeping the Bun server fast and memory-light.
* Real-Time: Use Hono's native Bun WebSocket adapter (`createBunWebSocket`) backed by Redis Pub/Sub for horizontal scaling.
* Multi-table operations: Allow the Service to initiate `db.transaction(async (tx) => ...)` and pass `tx` down as an optional argument to repositories.
* Standardize pagination: Use Page & Limit offset pagination returning `{ ok: true, data: { items, total, page, limit, totalPages } }`.
* Standardize all outputs through `Response.success()` and `Response.error()`.
* Ensure graceful shutdown in `src/index.ts` to cleanly close DB pools, Redis, and BullMQ workers on termination signals (`SIGINT`, `SIGTERM`).

### 2. Audit / Review Mode
When reviewing, refactoring, or debugging existing backend code:
* **Gate 1 (Layer Leakage):** Flag and eliminate direct database queries in routes or services. Repositories exclusively execute Drizzle queries.
* **Gate 2 (Context & Validation Integrity):** Ensure routers use `new Hono<AppEnv>()` and all inputs parse through Zod schemas with proper string trimming.
* **Gate 3 (DTO Sanitization & Soft Deletes):** Verify all returned data passes through dedicated mappers and queries exclude soft-deleted records.
* **Gate 4 (Database Performance & Indexing):** Verify all foreign keys and soft-delete columns are indexed in Drizzle schemas. Ban unindexed table scans.
* **Gate 5 (Anti-Slop Audit):** Inspect against **The 6 Banned Backend Anti-Patterns** (no `any`, no raw `console.log`, no magic strings, no N+1 query loops, early return guard clauses only).
* **Gate 6 (Bun & Cloud Idioms):** Verify file uploads use presigned URLs instead of buffering on the server; ensure native Bun primitives are leveraged.

---

## Standard Directory Layout

```bash
my-bun-backend/
├── bunfig.toml                     # Bun runtime config (test timeout, flags)
├── drizzle.config.ts               # Drizzle Kit CLI configuration
├── tsconfig.json                   # Strict TypeScript compiler options
├── package.json
├── .env.example
│
├── src/
│   ├── index.ts                    # Server entry point, Bun.serve export, WebSockets & graceful shutdown
│   │
│   ├── app/
│   │   └── index.ts                # Hono bootstrap, global middlewares, router mounting
│   │
│   ├── config/
│   │   └── env.config.ts           # Fail-fast environment variable validation (Zod)
│   │
│   ├── types/
│   │   └── context.ts              # Strongly typed Hono AppEnv & AppVariables (user, requestId)
│   │
│   ├── features/                   # Vertical domain slices
│   │   └── [feature-name]/         # e.g. auth, users, products, orders
│   │       ├── routes/
│   │       │   └── index.ts        # Hono<AppEnv> inline route handlers & validation bindings
│   │       ├── services/
│   │       │   └── index.ts        # Pure business logic, caching & transactions (throws AppError)
│   │       ├── repositories/
│   │       │   └── index.ts        # Drizzle ORM queries (relational reads, QB writes, soft deletes)
│   │       ├── schemas/
│   │       │   └── index.ts        # Zod DTOs with auto-trim & normalization
│   │       ├── mappers/
│   │       │   └── index.ts        # Dedicated DTO transformation functions (zero sensitive leaks)
│   │       └── types/
│   │           └── index.ts        # (Optional) Feature domain types
│   │
│   ├── db/
│   │   ├── index.ts                # Connection pool & Drizzle client instance (with schema)
│   │   ├── helpers/
│   │   │   └── columns.ts          # Common Drizzle helpers (uuidv7 primary keys, timestamps, softDelete)
│   │   ├── schemas/                # Modular database table definitions with mandatory indexes
│   │   │   ├── users.schema.ts
│   │   │   ├── [entity].schema.ts
│   │   │   └── index.ts            # Central re-export of all database schemas & relations
│   │   └── scripts/                # Database CLI scripts
│   │       ├── migrate.ts          # Programmatic migration runner
│   │       └── seed.ts             # Database seeder
│   │
│   ├── middlewares/                # Cross-cutting HTTP middlewares
│   │   ├── requestId.middleware.ts # UUIDv7 request correlation ID injection
│   │   ├── auth.middleware.ts      # JWT Bearer token authentication & user context injection
│   │   ├── validate.middleware.ts  # Zod validation bindings (validateBody, validateParams, etc.)
│   │   ├── errorHandler.middleware.ts # Central error boundary catching AppError & ZodError
│   │   ├── logger.middleware.ts    # Request/Response logger with colored tags & request ID
│   │   ├── cors.middleware.ts      # Environment-aware CORS configuration
│   │   └── rateLimit.middleware.ts # Redis-backed rate limiting
│   │
│   ├── lib/                        # Core shared primitives & utilities
│   │   ├── error.ts                # AppError hierarchy & ErrorCode types
│   │   ├── response.ts             # Standardized Response envelope helpers
│   │   ├── logger.ts               # Colorized, dependency-isolated ANSI logger
│   │   ├── pagination.ts           # Offset pagination helper & standard response envelope
│   │   ├── redis.ts                # Shared Redis client (ioredis) for queues, rate limiting, pub/sub
│   │   ├── cache.ts                # Type-safe Redis getOrSet and eviction utilities
│   │   ├── storage.ts              # Presigned URL generator for S3 / Cloudflare R2
│   │   └── tryCatch.ts             # Functional Result<T, E> wrapper
│   │
│   ├── queues/                     # Distributed background tasks (BullMQ + Redis)
│   │   ├── index.ts                # Queue definitions & worker registrations
│   │   └── workers/                # Dedicated task workers (email, processing, webhooks)
│   │
│   ├── ws/                         # Real-time WebSockets (Hono Bun WebSocket + Redis Pub/Sub)
│   │   └── index.ts                # WebSocket upgrade route, authentication, and event distribution
│   │
│   └── routes/
│       └── index.ts                # Central router aggregating all feature routers
│
└── tests/                          # Automated test suites (bun test)
    ├── setup.ts                    # Test environment configuration & DB reset hooks
    ├── unit/                       # Unit tests (Services, mappers, validation schemas)
    └── integration/                # Integration tests (HTTP endpoints via app.request, repositories)
```

---

## The 6 Banned Backend Anti-Patterns (Clean Code Guardrails)

Reject these patterns unconditionally in any codebase:

### 1. Zero `any` or Loose Type Casting
* Never use `any`, `as any`, or ignore compiler warnings with `@ts-ignore`.
* Everything must be strictly typed via Zod schemas, Drizzle `$inferSelect`/`$inferInsert`, or explicit interfaces.

### 2. Zero Raw `console.log` / `console.error`
* Never use raw `console.log` or `console.error` in application code.
* Always use the structured ANSI `logger` from `@/lib/logger` with request ID correlation.

### 3. Zero Magic Numbers & Strings
* Never hardcode magic numbers or status strings in logic (e.g. `status === 3`, `user.role === "admin"`).
* Define strongly typed enums or `as const` objects in `types/` or `schemas/`.

### 4. Early Return Guard Clauses Only
* Ban deeply nested `if/else` ladders (maximum nesting depth: 2 levels).
* Invert conditions and return or throw immediately.

### 5. Zero N+1 Query Loops
* Never execute database queries inside an `array.map()` or loop.
* Use Drizzle Relational Queries (`with: { ... }`) or batch in-array lookups.

### 6. Mandatory Feature DTO Mappers
* Never return raw database model objects directly to API callers or rely on ad-hoc destructuring.
* Every feature must provide a dedicated mapper in `mappers/` that explicitly picks safe fields.

---

## The 10 Core Engineering Pillars

### Pillar 1: Strict 3-Tier Vertical Slice & Transaction Boundaries
Features must be self-contained and respect a strict 3-tier boundary:

```
[ HTTP Route ]  ──(calls)──>  [ Domain Service ]  ──(calls)──>  [ Repository ]  ──(queries)──> [ Database ]
      │                              │                                │
 Validates Input               Business Rules                  Drizzle Queries
 Inline Handlers               Throws AppError                 (db.query for reads)
 Formats Response              (Orchestrates db.transaction)   (Query Builder writes)
 (Zero DB / Zero Rules)        (Zero Hono Context)             (Soft-delete filters)
```

1. **`routes/index.ts` (HTTP Boundary)**:
   * Keep handlers inline for maximum readability.
   * Only handles HTTP concerns: route registration, middleware attachment, parameter parsing, and returning `Response.success()`.
   * **Rule:** Initialize with `new Hono<AppEnv>()`. NEVER write business logic or database queries in routes.
2. **`services/index.ts` (Domain Logic & Transaction Boundary)**:
   * Executes business workflows, domain validation, permission checks, caching, and coordinates repositories.
   * Throws typed domain errors (`NotFoundError`, `ConflictError`, `BadRequestError`).
   * **Transactions:** Multi-table atomic mutations open `db.transaction(async (tx) => ...)` and pass `tx` into repositories.
   * **Rule:** NEVER pass Hono `Context` (`c`) to a service. Services accept plain typed arguments and return data or throw `AppError`.
3. **`repositories/index.ts` (Data Access Boundary)**:
   * The ONLY place where Drizzle ORM queries are executed.
   * Every repository method accepts an optional `tx?: Transaction` parameter (`const client = tx ?? db`).
   * Enforces soft-delete filtering (`where: isNull(table.deletedAt)`).

---

### Pillar 2: Feature Mappers (Zero Data Leakage)
Every domain feature must contain a `mappers/index.ts` file that explicitly transforms database records into safe response DTOs:

```typescript
export class UserMapper {
  static toResponse(user: User): UserResponseDTO {
    return {
      id: user.id,
      email: user.email,
      fullName: user.fullName,
      createdAt: user.createdAt.toISOString(),
    };
  }

  static toResponseList(users: User[]): UserResponseDTO[] {
    return users.map(this.toResponse);
  }
}
```

* **Rule:** Never rely on ad-hoc object destructuring (`const { passwordHash: _, ...user } = ...`) in services.

---

### Pillar 3: Database ID Strategy, Schema Helpers & Mandatory Indexing
1. **Primary Keys**:
   * Standardize on **`uuidv7`** for entity tables (users, orders, products, sessions) to prevent B-tree fragmentation.
   * Use **`integer` / `serial`** strictly when explicitly required (lookup tables, status sequences).
2. **Shared Column Helpers (`src/db/helpers/columns.ts`)**:
   * `primaryKeyUuidV7()`
   * `timestamps`: `createdAt` and auto-updating `updatedAt`.
   * `softDelete`: `deletedAt: timestamp("deleted_at", { withTimezone: true })`.
3. **Mandatory Indexing Rules**:
   * Index all foreign keys (`userId`, `orderId`, etc.).
   * Index `deletedAt` on all soft-delete tables.
   * Enforce partial unique indexes for unique fields on soft-delete tables:
     ```typescript
     uniqueIndex("users_email_unique").on(users.email).where(isNull(users.deletedAt))
     ```

---

### Pillar 4: Strongly Typed Context & Correlation IDs
1. **Typed Hono Environment (`src/types/context.ts`)**:
   ```typescript
   export type AuthUser = {
     id: string;
     email: string;
     role: "admin" | "user";
   };

   export type AppVariables = {
     user: AuthUser;
     requestId: string;
   };

   export type AppEnv = {
     Variables: AppVariables;
   };
   ```
2. **Request ID Middleware (`src/middlewares/requestId.middleware.ts`)**:
   * Reads existing `X-Request-Id` or generates a new `uuidv7`.
   * Sets `c.set("requestId", id)` and attaches `X-Request-Id` to response headers.
   * Prefixes all log lines with the correlation ID.

---

### Pillar 5: Schema-First Validation & Automatic Sanitization
Every incoming request must be validated using Zod schemas with automatic string sanitization:

```typescript
export const createUserSchema = z.object({
  email: z.string().trim().toLowerCase().email("Invalid email address"),
  password: z.string().min(8, "Password must be at least 8 characters"),
  fullName: z.string().trim().min(2, "Full name must be at least 2 characters"),
  bio: z.string().trim().transform((val) => val === "" ? null : val).optional(),
});
```

* **Validate Request Body:** `validateBody(schema)`
* **Validate URL Params:** `validateParams(schema)`
* **Validate Query Strings:** `validateQuery(schema)`
* **Validate Form Data:** `validateFormData(schema)`

---

### Pillar 6: Unified Response & Pagination Envelopes
All API responses must follow a strict, predictable JSON envelope format:

```typescript
// Success
export type SuccessResponse<T> = { ok: true; data?: T; message?: string };

// Error
export type ErrorResponse = { ok: false; message: string; error: ErrorCode };

// List Pagination
export type PaginatedData<T> = {
  items: T[];
  total: number;
  page: number;
  limit: number;
  totalPages: number;
};
```
* Use `Response.success(c, { items, total, page, limit, totalPages })` for list endpoints.

---

### Pillar 7: Domain Error Hierarchy & Central Error Boundary
Throw structured domain errors extending `AppError`:

```typescript
export class NotFoundError extends AppError { constructor(message = "Resource not found") { super(message, 404, "NOT_FOUND"); } }
export class ValidationError extends AppError { constructor(message = "Unprocessable Entity") { super(message, 422, "VALIDATION_ERROR"); } }
export class BadRequestError extends AppError { constructor(message = "Bad Request") { super(message, 400, "BAD_REQUEST"); } }
export class AuthorizationError extends AppError { constructor(message = "Unauthorized") { super(message, 401, "UNAUTHORIZED"); } }
export class ForbiddenError extends AppError { constructor(message = "Forbidden") { super(message, 403, "FORBIDDEN"); } }
export class ConflictError extends AppError { constructor(message = "Conflict") { super(message, 409, "CONFLICT"); } }
export class RateLimitError extends AppError { constructor(message = "Too Many Requests") { super(message, 429, "RATE_LIMITED"); } }
```

* `app.onError(errorHandler)` catches all `AppError`, `ZodError`, malformed JSON, and critical 500 fallbacks.

---

### Pillar 8: High-Performance Caching & Eviction (`@/lib/cache.ts`)
* High-read queries use type-safe `cache.getOrSet(key, ttlSeconds, fetcherFn)`:
  ```typescript
  return cache.getOrSet(`user:${id}`, 3600, () => UsersRepository.findById(id));
  ```
* Mutating operations in Services explicitly invalidate cached keys (`cache.del(`user:${id}`)`).

---

### Pillar 9: Direct Presigned Storage Architecture (`@/lib/storage.ts`)
* **Never buffer file uploads in the Bun process.**
* The backend generates signed PUT URLs via S3/Cloudflare R2 SDK.
* Clients upload directly to object storage with expiration windows, minimizing server memory and bandwidth consumption.

---

### Pillar 10: Real-Time WebSockets & Redis Pub/Sub (`@/ws/`)
* Standardize on Hono's native Bun WebSocket adapter (`createBunWebSocket`).
* Authenticate during upgrade via query token or ticket.
* Use Redis Pub/Sub (`redisSub` and `redisPub`) for horizontal message broadcasting across server instances.

---

## The 6 Audit Gates (Verification)

Before shipping backend code, evaluate it against all six gates:

### Gate 1: The Layer Separation Gate
> *Does any route contain direct SQL/Drizzle calls? Does any service accept Hono Context?*
* **If yes:** Refactor immediately. Route handles HTTP $\rightarrow$ Service handles logic & transactions $\rightarrow$ Repository handles database queries.

### Gate 2: The Context & Validation Gate
> *Does the route use `new Hono<AppEnv>()`? Are all inputs validated with Zod via `validate*` with automatic trimming?*
* **If yes:** Context and inputs are type-safe.

### Gate 3: The DTO & Soft-Delete Gate
> *Does every endpoint return data mapped through a dedicated `mappers/` transformation function? Are soft-deleted rows filtered out?*
* **If yes:** Zero chance of sensitive data leakage or zombie records.

### Gate 4: The Database Performance & Index Gate
> *Are all foreign keys, `deletedAt` columns, and soft-delete unique fields properly indexed in Drizzle?*
* **If yes:** Database access is optimized for scale.

### Gate 5: The Anti-Slop & Clean Code Gate
> *Are there any `any` types, raw `console.log` calls, magic numbers, nested `if/else` ladders, or N+1 query loops?*
* **If yes:** Clean up the code to satisfy all 6 Clean Code Guardrails.

### Gate 6: The Storage, Queues & Teardown Gate
> *Are file uploads presigned? Are heavy tasks offloaded to BullMQ? Is graceful shutdown configured?*
* **If yes:** Production deployment is resilient, fast, and crash-safe.

---

## Pre-Flight Checklist

Before presenting or shipping backend code, verify each checkbox:

- [ ] **Folder Structure:** Feature lives in `src/features/<feature>/` with `routes/`, `services/`, `repositories/`, `schemas/`, `mappers/`.
- [ ] **Context Typing:** Router initialized with `new Hono<AppEnv>()`. Handlers remain inline.
- [ ] **Route Isolation:** Route only handles validation bindings, service calls, mapper calls, and `Response.success()`.
- [ ] **Service Isolation:** Service contains pure domain logic, orchestrates `db.transaction()`, manages cache, and throws `AppError` subclasses.
- [ ] **Repository Isolation:** Drizzle queries strictly encapsulated in repository methods; accepts optional `tx?: Transaction`; filters `isNull(deletedAt)`.
- [ ] **DTO Mapping:** All database records mapped through dedicated feature mapper before returning to client.
- [ ] **Input Sanitization:** Strings trimmed and normalized via Zod schemas.
- [ ] **No Anti-Patterns:** Zero `any`, zero `console.log`, zero magic strings, early return guard clauses only, zero N+1 queries.
- [ ] **Response Envelope:** All success responses return `Response.success(c, data, message, status)`. List endpoints return `{ items, total, page, limit, totalPages }`.
- [ ] **Error Handling:** Central `errorHandler` registered in `app/index.ts`; errors thrown as `AppError`.
- [ ] **Database Schemas:** Tables use `uuidv7` primary keys, `timestamps`, `softDelete`, and explicit indexes on foreign keys.
- [ ] **Database Scripts:** Migrations and seeds live in `src/db/scripts/` and run via `bun src/db/scripts/<script>.ts`.
- [ ] **Caching:** High-read queries use `cache.getOrSet`; mutations evict cache keys.
- [ ] **Object Storage:** File uploads use presigned URLs (`@/lib/storage.ts`).
- [ ] **WebSockets:** Real-time updates use `createBunWebSocket` with Redis Pub/Sub.
- [ ] **Request ID Tracing:** `requestId` middleware attached; logs and responses include correlation ID.
- [ ] **Graceful Shutdown:** Process termination signals close DB pool, Redis, and workers cleanly.
- [ ] **Testing:** Unit and integration tests placed in `tests/unit/` and `tests/integration/` running cleanly on `bun test`.
