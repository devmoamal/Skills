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
              Domain Service (Pure Business Logic + db.transaction + AppError)
                    ↓
              Drizzle Repository (Relational Reads + Query Builder Writes)
                    ↓
              Feature Mapper (Strict DTO Transformation — Zero Sensitive Leaks)
                    ↓
              Unified Response Envelope: Response.success(c, safeData)
```

Every endpoint, query, service, and data contract must adhere to uncompromising type safety, strict separation of concerns, and clean code hygiene.

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
* Use Drizzle **Relational Queries** (`db.query.*`) for reads, and **Query Builder** (`db.insert()`, `db.update()`, `db.delete()`) for writes.
* Primary Keys: Standardize on **`uuidv7`** for entity tables to prevent B-tree fragmentation; use `integer` strictly for small lookup or sequence tables.
* Always transform database models into safe client responses using dedicated feature mappers in `mappers/`. Never return raw database records directly.
* Multi-table operations: Allow the Service to initiate `db.transaction(async (tx) => ...)` and pass `tx` down as an optional argument to repositories.
* Standardize pagination: Use Page & Limit offset pagination returning `{ ok: true, data: { items, total, page, limit, totalPages } }`.
* Standardize all outputs through `Response.success()` and `Response.error()`.
* Ensure graceful shutdown in `src/index.ts` to cleanly close DB pools, Redis, and BullMQ workers on termination signals (`SIGINT`, `SIGTERM`).

### 2. Audit / Review Mode
When reviewing, refactoring, or debugging existing backend code:
* **Gate 1 (Layer Leakage):** Flag and eliminate direct database queries in routes or services. Repositories exclusively execute Drizzle queries.
* **Gate 2 (Context & Validation Integrity):** Ensure routers use `new Hono<AppEnv>()` and all inputs parse through Zod schemas with proper string trimming.
* **Gate 3 (DTO Sanitization & Data Leaks):** Verify all returned data passes through dedicated mappers. Ban ad-hoc destructuring (`const { passwordHash, ...user } = ...`) that can accidentally leak sensitive internal columns.
* **Gate 4 (Anti-Slop Audit):** Inspect against **The 6 Banned Backend Anti-Patterns** (no `any`, no raw `console.log`, no magic strings, no N+1 query loops, early return guard clauses only).
* **Gate 5 (Bun & Tooling Idioms):** Replace legacy Node dependencies with native Bun primitives (`Bun.password`, `Bun.file`, native `export default { port, fetch }`, and `bun test`).

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
│   ├── index.ts                    # Server entry point, Bun.serve export & graceful shutdown
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
│   │       │   └── index.ts        # Pure business logic & transaction orchestration (throws AppError)
│   │       ├── repositories/
│   │       │   └── index.ts        # Drizzle ORM queries (relational reads, query builder writes)
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
│   │   │   └── columns.ts          # Common Drizzle helpers (uuidv7 primary keys, timestamps)
│   │   ├── schemas/                # Modular database table definitions
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
│   │   ├── redis.ts                # Shared Redis client (ioredis) for queues & rate limiting
│   │   └── tryCatch.ts             # Functional Result<T, E> wrapper
│   │
│   ├── queues/                     # Distributed background tasks (BullMQ + Redis)
│   │   ├── index.ts                # Queue definitions & worker registrations
│   │   └── workers/                # Dedicated task workers (email, processing, webhooks)
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

AI coding assistants frequently produce messy, boilerplate-heavy backend code. Reject these patterns outright:

### 1. Zero `any` or Loose Type Casting
* **Rule:** Never use `any`, `as any`, or ignore compiler warnings with `@ts-ignore`.
* Everything must be strictly typed via Zod schemas, Drizzle `$inferSelect`/`$inferInsert`, or explicit interfaces.

### 2. Zero Raw `console.log` / `console.error`
* **Rule:** Never use raw `console.log` or `console.error` in application code.
* Always use the structured ANSI `logger` from `@/lib/logger` with appropriate log levels (`info`, `warn`, `error`, `debug`).

### 3. Zero Magic Numbers & Strings
* **Rule:** Never hardcode magic numbers or status strings in logic (e.g. `status === 3`, `user.role === "admin"`).
* Define strongly typed enums or `as const` objects in `types/` or `schemas/`.

### 4. Early Return Guard Clauses Only
* **Rule:** Ban deeply nested `if/else` ladders (maximum nesting depth: 2 levels).
* Invert conditions and return or throw immediately:
  ```typescript
  // BAD: Deeply nested pyramid
  if (user) {
    if (user.isActive) {
      // do something
    } else {
      throw new ForbiddenError();
    }
  } else {
    throw new NotFoundError();
  }

  // GOOD: Clean guard clauses
  if (!user) throw new NotFoundError("User not found");
  if (!user.isActive) throw new ForbiddenError("Account is inactive");
  // Proceed with linear happy path
  ```

### 5. Zero N+1 Query Loops
* **Rule:** Never execute database queries inside an `array.map()` or loop.
* Use Drizzle Relational Queries (`with: { ... }`) or batch queries (`where(inArray(table.id, ids))`) to fetch related data in a single round trip.

### 6. Mandatory Feature DTO Mappers
* **Rule:** Never return raw database model objects directly to API callers or rely on error-prone object destructuring.
* Every feature must provide a dedicated mapper in `mappers/` that explicitly picks safe fields.

---

## The 9 Core Engineering Pillars

### Pillar 1: Strict 3-Tier Vertical Slice & Transaction Boundaries
Features must be self-contained and respect a strict 3-tier boundary:

```
[ HTTP Route ]  ──(calls)──>  [ Domain Service ]  ──(calls)──>  [ Repository ]  ──(queries)──> [ Database ]
      │                              │                                │
 Validates Input               Business Rules                  Drizzle Queries
 Inline Handlers               Throws AppError                 (db.query for reads)
 Formats Response              (Orchestrates db.transaction)   (Query Builder writes)
 (Zero DB / Zero Rules)        (Zero Hono Context)             (Accepts tx?: Transaction)
```

1. **`routes/index.ts` (HTTP Boundary)**:
   * Keep handlers inline for maximum readability.
   * Only handles HTTP concerns: route registration, middleware attachment, parameter parsing, and returning `Response.success()`.
   * **Rule:** Initialize with `new Hono<AppEnv>()`. NEVER write business logic or database queries in routes.
2. **`services/index.ts` (Domain Logic & Transaction Boundary)**:
   * Executes business workflows, domain validation, permission checks, and coordinates multiple repositories.
   * Throws typed domain errors (`NotFoundError`, `ConflictError`, `BadRequestError`).
   * **Transactions:** When an operation mutates multiple tables atomically, the Service opens `db.transaction(async (tx) => ...)` and passes `tx` into each repository call.
   * **Rule:** NEVER pass Hono `Context` (`c`) to a service. Services accept plain typed arguments and return data or throw `AppError`.
3. **`repositories/index.ts` (Data Access Boundary)**:
   * The ONLY place where Drizzle ORM queries are executed.
   * Every repository method accepts an optional `tx?: Transaction` parameter:
     ```typescript
     const client = tx ?? db;
     ```
   * **Rule:** NEVER handle HTTP request/response or domain authorization rules in repositories.

---

### Pillar 2: Feature Mappers (Zero Data Leakage)
Every domain feature must contain a `mappers/index.ts` file that explicitly transforms database records into safe response DTOs:

```typescript
// features/users/mappers/index.ts
import type { User } from "@/db/schemas";

export type UserResponseDTO = {
  id: string;
  email: string;
  fullName: string;
  createdAt: string;
};

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

* **Rule:** Never rely on ad-hoc object destructuring (`const { passwordHash: _, ...user } = ...`) in services. Destructuring is fragile and easily leaks newly added private database columns.

---

### Pillar 3: Database ID Strategy & Common Column Helpers
1. **Primary Keys**:
   * Standardize on **`uuidv7`** for entity tables (users, orders, products, sessions). UUIDv7 values are time-ordered, preventing B-tree index fragmentation while remaining globally unique and non-enumerable.
   * Use **`integer` / `serial`** strictly when explicitly required (e.g. small lookup/reference tables, numeric status codes).
2. **Shared Column Helpers (`src/db/helpers/columns.ts`)**:
   * Standardize `uuidv7` and `timestamps()` across all tables for consistent column naming:
   ```typescript
   export const timestamps = {
     createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
     updatedAt: timestamp("updated_at", { withTimezone: true }).defaultNow().$onUpdate(() => new Date()).notNull(),
   };
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
   * Reads existing `X-Request-Id` header (from reverse proxy / Cloudflare) or generates a new `uuidv7`.
   * Sets `c.set("requestId", id)` and attaches `X-Request-Id` to response headers.
   * Prefixes all log lines with the correlation ID for distributed tracing.

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
* **Rule:** Never use raw `await c.req.json()` or `c.req.param("id")` without a validator.

---

### Pillar 6: Unified Response & Pagination Envelopes
All API responses must follow a strict, predictable JSON envelope format:

#### Standard Response Envelopes:
```typescript
// Success
export type SuccessResponse<T> = {
  ok: true;
  data?: T;
  message?: string;
};

// Error
export type ErrorResponse = {
  ok: false;
  message: string;
  error: ErrorCode;
};
```

#### Standard List Pagination:
Collections use Page & Limit offset pagination with metadata:
```typescript
export type PaginatedData<T> = {
  items: T[];
  total: number;
  page: number;
  limit: number;
  totalPages: number;
};
```
* Use `Response.success(c, { items, total, page, limit, totalPages })` for list endpoints.
* **Rule:** Never return inconsistent ad-hoc JSON like `{ success: true, user }`, `{ error: "msg" }`, or raw arrays.

---

### Pillar 7: Domain Error Hierarchy & Central Error Boundary
Never write repetitive `try/catch` blocks across routes. Throw structured domain errors and let the central error handler format the response:

```typescript
export class AppError extends Error {
  constructor(
    message: string,
    public readonly status: ContentfulStatusCode = 500,
    public readonly code: ErrorCode = "UNKNOWN",
  ) {
    super(message);
    this.name = this.constructor.name;
  }
}

export class NotFoundError extends AppError {
  constructor(message = "Resource not found") {
    super(message, 404, "NOT_FOUND");
  }
}

export class ValidationError extends AppError {
  constructor(message = "Unprocessable Entity") {
    super(message, 422, "VALIDATION_ERROR");
  }
}

export class BadRequestError extends AppError {
  constructor(message = "Bad Request") {
    super(message, 400, "BAD_REQUEST");
  }
}

export class AuthorizationError extends AppError {
  constructor(message = "Unauthorized") {
    super(message, 401, "UNAUTHORIZED");
  }
}

export class ForbiddenError extends AppError {
  constructor(message = "Forbidden") {
    super(message, 403, "FORBIDDEN");
  }
}

export class ConflictError extends AppError {
  constructor(message = "Conflict") {
    super(message, 409, "CONFLICT");
  }
}

export class RateLimitError extends AppError {
  constructor(message = "Too Many Requests") {
    super(message, 429, "RATE_LIMITED");
  }
}
```

* **Central Error Handler:**
  `app.onError(errorHandler)` catches:
  1. `AppError` subclasses $\rightarrow$ returns configured status and error code.
  2. `ZodError` $\rightarrow$ wraps into `ValidationError` (422).
  3. Malformed JSON $\rightarrow$ wraps into `BadRequestError("Invalid JSON")` (400).
  4. Unhandled runtime crashes $\rightarrow$ logs `[CRITICAL]` with request ID and returns safe 500 without leaking stack traces.

---

### Pillar 8: Drizzle ORM Querying Standards
Repositories must follow explicit querying conventions:

1. **Reads $\rightarrow$ Drizzle Relational Queries API**:
   Use `db.query.<table_name>.findFirst` or `findMany` with declarative relations (`with: { ... }`) for clean, nested reads:
   ```typescript
   const user = await client.query.users.findFirst({
     where: eq(users.id, id),
     with: { profile: true },
   });
   ```
2. **Writes $\rightarrow$ Query Builder**:
   Use `db.insert()`, `db.update()`, and `db.delete()` with `.returning()`:
   ```typescript
   const [created] = await client.insert(users).values(data).returning();
   ```
3. **Modular Schemas in `src/db/schemas/`**:
   Tables and relations defined per domain and re-exported in `src/db/schemas/index.ts`.
4. **Dedicated CLI Scripts in `src/db/scripts/`**:
   `migrate.ts` and `seed.ts` executed via native Bun: `bun src/db/scripts/migrate.ts`.

---

### Pillar 9: Graceful Lifecycle, Queues & Runtime Mastery
1. **Graceful Teardown (`src/index.ts`)**:
   Intercept `SIGINT` and `SIGTERM` signals:
   - Drain active HTTP requests.
   - Close BullMQ workers (`await worker.close()`).
   - Drain PostgreSQL connection pool (`await pool.end()`).
   - Disconnect Redis client (`await redis.quit()`).
   - Cleanly exit with code 0.
2. **Redis-Backed Queues (`src/queues/`)**:
   Offload heavy asynchronous operations (emails, webhooks, processing) to BullMQ queues.
3. **Bun Native Primitives**:
   - Native password hashing: `await Bun.password.hash(pwd, { algorithm: "argon2id" })`.
   - Native file I/O: `Bun.file(path)` and `await Bun.write(path, data)`.
   - Native testing: `bun test` across root `tests/unit/` and `tests/integration/`.

---

## The 5 Audit Gates (Verification)

Before considering any backend feature or refactor finished, evaluate it against all five gates:

### Gate 1: The Layer Separation Gate
> *Does any route contain direct SQL/Drizzle calls? Does any service accept Hono Context?*
* **If yes:** Refactor immediately. Route handles HTTP $\rightarrow$ Service handles logic & transactions $\rightarrow$ Repository handles database queries.

### Gate 2: The Context & Validation Gate
> *Does the route use `new Hono<AppEnv>()`? Are all route parameters, queries, and bodies validated with Zod via `validate*` with string trimming?*
* **If yes:** Context and inputs are type-safe.

### Gate 3: The DTO & Sanitization Gate
> *Does every endpoint return data mapped through a dedicated `mappers/` transformation function? Are all private database columns stripped?*
* **If yes:** Zero chance of sensitive data leakage.

### Gate 4: The Clean Code & Anti-Pattern Gate
> *Are there any `any` types, raw `console.log` calls, magic numbers, nested `if/else` ladders, or N+1 query loops?*
* **If yes:** Clean up the code to satisfy all 6 Clean Code Guardrails.

### Gate 5: The Teardown & Queues Gate
> *Are heavy tasks offloaded to BullMQ? Are connections properly registered for graceful teardown on termination signals?*
* **If yes:** Production deployment is resilient and crash-safe.

---

## Pre-Flight Checklist

Before presenting or shipping backend code, verify each checkbox:

- [ ] **Folder Structure:** Feature lives in `src/features/<feature>/` with `routes/`, `services/`, `repositories/`, `schemas/`, `mappers/`.
- [ ] **Context Typing:** Router initialized with `new Hono<AppEnv>()`. Handlers remain inline.
- [ ] **Route Isolation:** Route only handles validation bindings, service calls, mapper calls, and `Response.success()`.
- [ ] **Service Isolation:** Service contains pure domain logic, accepts typed DTOs, orchestrates `db.transaction()` if multi-table, and throws `AppError` subclasses.
- [ ] **Repository Isolation:** Drizzle queries (relational reads & query builder writes) strictly encapsulated in repository methods; accepts optional `tx?: Transaction`.
- [ ] **DTO Mapping:** All database records mapped through dedicated feature mapper before returning to client.
- [ ] **Input Sanitization:** Strings trimmed and normalized via Zod schemas.
- [ ] **No Anti-Patterns:** Zero `any`, zero `console.log`, zero magic strings, early return guard clauses only, zero N+1 queries.
- [ ] **Response Envelope:** All success responses return `Response.success(c, data, message, status)`. List endpoints return `{ items, total, page, limit, totalPages }`.
- [ ] **Error Handling:** Central `errorHandler` registered in `app/index.ts`; errors thrown as `AppError`.
- [ ] **Database Schemas:** Tables use `uuidv7` primary keys and shared `timestamps()` helper from `src/db/helpers/columns.ts`.
- [ ] **Database Scripts:** Migrations and seeds live in `src/db/scripts/` and run via `bun src/db/scripts/<script>.ts`.
- [ ] **Request ID Tracing:** `requestId` middleware attached; logs and responses include correlation ID.
- [ ] **Background Processing:** Asynchronous heavy tasks offloaded to BullMQ queues (`src/queues/`).
- [ ] **Graceful Shutdown:** Process termination signals close DB pool, Redis, and workers cleanly.
- [ ] **Testing:** Unit and integration tests placed in `tests/unit/` and `tests/integration/` running cleanly on `bun test`.
