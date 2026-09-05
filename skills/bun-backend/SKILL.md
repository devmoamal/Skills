---
name: bun-backend
description: Enforce high-performance, modular, and type-safe backend architecture for Bun using Hono, Drizzle ORM, Zod, and a strict 3-tier vertical slice (Route -> Service -> Repository) design. Use whenever designing, building, scaffolding, refactoring, or auditing Bun backend APIs, services, routes, database schemas, or middlewares.
---

# Bun Backend: Production Architecture & Standards

Build fast, bulletproof, type-safe backend services powered by Bun, Hono, Drizzle ORM, Zod, and Redis.

```
Request → Middlewares (CORS, Logger, RateLimit, Auth) → Feature Route → Zod Validator
                ↓                                                               ↓
      Central Error Handler                                             Domain Service (AppError / db.transaction)
                ↓                                                               ↓
     Response.error(c, ...)                                            Drizzle Repository (Relational Reads / QB Writes)
                                                                                ↓
                                                                       Database Pool (Postgres)
```

Every endpoint, query, service, and queue must adhere to strict separation of concerns, fail-fast schema validation, and unified response envelopes.

---

## Operating Modes

### 1. Build Mode (Default)
When scaffolding new features, endpoints, or services:
* Structure all domain features under `src/features/<feature>/` with explicit `routes/`, `services/`, `repositories/`, and `schemas/`.
* Strictly enforce the 3-tier boundary: **Route $\rightarrow$ Service $\rightarrow$ Repository**. Never bypass layers.
* Strong Context Typing: Always initialize routers with `new Hono<AppEnv>()` importing `AppEnv` from `@/types/context`.
* Validate all request inputs (body, params, query) using `validateBody`, `validateParams`, `validateQuery` before reaching business logic.
* Use Drizzle **Relational Queries** (`db.query.*`) for reads, and **Query Builder** (`db.insert()`, `db.update()`, `db.delete()`) for writes.
* Atomic multi-table operations: Allow the Service to initiate `db.transaction(async (tx) => ...)` and pass `tx` down as an optional argument to repositories.
* Standardize pagination: Use Page & Limit offset pagination returning `{ ok: true, data: { items, total, page, limit, totalPages } }`.
* Standardize all outputs through `Response.success()` and `Response.error()`.
* Keep database table definitions modularized in `src/db/schemas/` and CLI scripts in `src/db/scripts/`.

### 2. Audit / Review Mode
When reviewing, refactoring, or debugging existing backend code:
* **Gate 1 (Layer Leakage):** Flag and eliminate direct database queries in routes or services. Repositories exclusively execute Drizzle queries.
* **Gate 2 (Context & Validation Integrity):** Ensure routers use `new Hono<AppEnv>()` and all inputs parse through Zod schemas. Ban untyped `c.req.json()` or `c.req.param()`.
* **Gate 3 (Transaction & Error Safety):** Verify multi-table mutations run inside `db.transaction`, all thrown errors extend `AppError`, and responses follow the unified envelope.
* **Gate 4 (Bun & Tooling Idioms):** Replace legacy Node dependencies with native Bun primitives (`Bun.password`, `Bun.file`, native `export default { port, fetch }`, and `bun test`).

---

## Standard Directory Layout

```bash
my-bun-backend/
├── bunfig.toml                     # Bun runtime config (watch flags, test timeout)
├── drizzle.config.ts               # Drizzle Kit CLI configuration
├── tsconfig.json                   # Strict TypeScript compiler options
├── package.json
├── .env.example
│
├── src/
│   ├── index.ts                    # Bun server entry point (export default { port, fetch })
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
│   │       │   └── index.ts        # Hono<AppEnv> routes, validation bindings, response formatting
│   │       ├── services/
│   │       │   └── index.ts        # Pure business logic & transaction orchestration (throws AppError)
│   │       ├── repositories/
│   │       │   └── index.ts        # Drizzle ORM queries (relational reads, query builder writes)
│   │       ├── schemas/
│   │       │   └── index.ts        # Zod DTOs for request body, params, query, and pagination
│   │       └── types/
│   │           └── index.ts        # (Optional) Feature domain types
│   │
│   ├── db/
│   │   ├── index.ts                # Connection pool & Drizzle client instance (with schema)
│   │   ├── schemas/                # Modular database table definitions
│   │   │   ├── users.schema.ts
│   │   │   ├── [entity].schema.ts
│   │   │   └── index.ts            # Central re-export of all database schemas & relations
│   │   └── scripts/                # Database CLI scripts
│   │       ├── migrate.ts          # Programmatic migration runner
│   │       └── seed.ts             # Database seeder
│   │
│   ├── middlewares/                # Cross-cutting HTTP middlewares
│   │   ├── auth.middleware.ts      # JWT Bearer token authentication & user context injection
│   │   ├── validate.middleware.ts  # Zod validation bindings (validateBody, validateParams, etc.)
│   │   ├── errorHandler.middleware.ts # Central error boundary catching AppError & ZodError
│   │   ├── logger.middleware.ts    # Request/Response logger with colored tags & status codes
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
    ├── unit/                       # Unit tests (Services, validation schemas, lib utilities)
    └── integration/                # Integration tests (HTTP endpoints via app.request, repositories)
```

---

## The 8 Core Engineering Pillars

### Pillar 1: Strict 3-Tier Vertical Slice & Transaction Boundaries
Features must be self-contained and respect a strict 3-tier boundary:

```
[ HTTP Route ]  ──(calls)──>  [ Domain Service ]  ──(calls)──>  [ Repository ]  ──(queries)──> [ Database ]
      │                              │                                │
 Validates Input               Business Rules                  Drizzle Queries
 Formats Response              Throws AppError                 (db.query for reads)
 (Zero DB / Zero Rules)        (Orchestrates db.transaction)   (Query Builder writes)
                               (Zero Hono Context)             (Accepts tx?: Transaction)
```

1. **`routes/index.ts` (HTTP Boundary)**:
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

### Pillar 2: Strongly Typed Hono Context (`AppEnv`)
Never leave Hono context untyped or cast `c.get("user") as any`. Define the global application environment in `src/types/context.ts`:

```typescript
// src/types/context.ts
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

All feature routers and middleware use this contract:
```typescript
// In all feature routes
import { Hono } from "hono";
import type { AppEnv } from "@/types/context";

const router = new Hono<AppEnv>();

router.get("/profile", async (c) => {
  const user = c.get("user"); // Fully typed as AuthUser!
  return Response.success(c, user);
});
```

---

### Pillar 3: Schema-First Validation (Zero Unchecked Inputs)
Every incoming request must be validated using Zod schemas before hitting any service or repository:

* **Validate Request Body:** `validateBody(schema)`
* **Validate URL Params:** `validateParams(schema)`
* **Validate Query Strings:** `validateQuery(schema)`
* **Validate Form Data:** `validateFormData(schema)`

```typescript
// features/users/routes/index.ts
import { Hono } from "hono";
import type { AppEnv } from "@/types/context";
import { validateBody, validateParams } from "@/middlewares/validate.middleware";
import { createUserSchema, userIdParamSchema } from "../schemas";
import { UsersService } from "../services";
import Response from "@/lib/response";

const router = new Hono<AppEnv>();

router.post("/", validateBody(createUserSchema), async (c) => {
  const data = c.req.valid("json");
  const user = await UsersService.create(data);
  return Response.success(c, user, "User created successfully", 201);
});

router.get("/:id", validateParams(userIdParamSchema), async (c) => {
  const { id } = c.req.valid("param");
  const user = await UsersService.getById(id);
  return Response.success(c, user);
});

export default router;
```

* **Rule:** Never use raw `await c.req.json()` or `c.req.param("id")` without a validator.
* **Single Source of Truth:** Derive TypeScript input types directly from schemas (`z.infer<typeof schema>`).

---

### Pillar 4: Unified Response & Pagination Envelopes
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

### Pillar 5: Domain Error Hierarchy & Central Error Boundary
Never write repetitive `try/catch` blocks across routes. Throw structured domain errors and let the central error handler format the response:

```typescript
// Custom error hierarchy extending AppError
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
```

* **Central Error Handler:**
  `app.onError(errorHandler)` intercepts:
  1. `AppError` subclasses $\rightarrow$ returns configured status and error code.
  2. `ZodError` $\rightarrow$ wraps into `ValidationError` (422).
  3. Malformed JSON $\rightarrow$ wraps into `BadRequestError("Invalid JSON")` (400).
  4. Unhandled runtime crashes $\rightarrow$ logs `[CRITICAL]` and returns safe 500 without leaking stack traces.

---

### Pillar 6: Drizzle ORM Querying Standards
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

### Pillar 7: Redis-Backed Queues & Rate Limiting
Decouple heavy tasks and protect against abusive traffic using Redis:

1. **Shared Redis Connection (`src/lib/redis.ts`)**:
   A single configured `ioredis` client instance shared between rate limiting and BullMQ queues.
2. **Background Jobs (`src/queues/`)**:
   * Heavy tasks (sending emails, webhook delivery, AI generation) are offloaded to BullMQ queues.
   * HTTP routes enqueue jobs without blocking the response:
     ```typescript
     await emailQueue.add("sendWelcomeEmail", { userId: user.id, email: user.email });
     ```
3. **Rate Limiting (`src/middlewares/rateLimit.middleware.ts`)**:
   Enforce sliding-window request throttling on sensitive endpoints (e.g. login, registration, public APIs) backed by Redis.

---

### Pillar 8: Bun Runtime Mastery & Testing Architecture
Take full advantage of Bun's native performance and test runner:

* **Bun Server Entrypoint:** Use native object export in `src/index.ts`:
  ```typescript
  import { app } from "@/app";
  import { env } from "@/config/env.config";
  import { logger } from "@/lib/logger";

  export default {
    port: env.PORT,
    fetch: app.fetch,
  };

  logger.info(`Server running on port ${env.PORT}`);
  ```
* **Native Password Hashing:**
  Use `Bun.password` with Argon2id:
  ```typescript
  const hash = await Bun.password.hash(password, { algorithm: "argon2id" });
  const isValid = await Bun.password.verify(password, hash);
  ```
* **Dedicated Test Suites (`tests/`)**:
  * Run native `bun test`.
  * `tests/unit/`: Test business logic in services and schema validations in isolation.
  * `tests/integration/`: Test HTTP endpoints end-to-end via Hono's `app.request("/api/...")` and verify database mutations against a test database.

---

## The 4 Audit Gates (Verification)

Before considering any backend feature or refactor finished, evaluate it against all four gates:

### Gate 1: The Layer Separation Gate
> *Does any route contain direct SQL/Drizzle calls? Does any service accept Hono Context?*
* **If yes:** Refactor immediately. Route handles HTTP $\rightarrow$ Service handles logic & transactions $\rightarrow$ Repository handles database queries.

### Gate 2: The Context & Validation Gate
> *Does the route use `new Hono<AppEnv>()`? Are all route parameters, queries, and bodies validated with Zod via `validate*`?*
* **If yes:** Context and inputs are type-safe.

### Gate 3: The Transaction & Error Gate
> *Are multi-table mutations wrapped in `db.transaction`? Does every thrown error extend `AppError`? Are responses returned via `Response.success()`?*
* **If yes:** Atomicity and error boundaries are guaranteed.

### Gate 4: The Performance & Queues Gate
> *Are blocking operations (emails, heavy processing) delegated to Redis/BullMQ background queues rather than run inside HTTP request cycles?*
* **If yes:** Latency remains low and resilient.

---

## Pre-Flight Checklist

Before presenting or shipping backend code, verify each checkbox:

- [ ] **Folder Structure:** Feature lives in `src/features/<feature>/` with `routes/`, `services/`, `repositories/`, `schemas/`.
- [ ] **Context Typing:** Router initialized with `new Hono<AppEnv>()`.
- [ ] **Route Isolation:** Route only handles validation bindings, service calls, and `Response.success()`.
- [ ] **Service Isolation:** Service contains pure domain logic, accepts typed DTOs, orchestrates `db.transaction()` if multi-table, and throws `AppError` subclasses.
- [ ] **Repository Isolation:** Drizzle queries (relational reads & query builder writes) strictly encapsulated in repository methods; accepts optional `tx?: Transaction`.
- [ ] **Input Validation:** Request body, params, and query parameters validated with `validateBody`, `validateParams`, `validateQuery`.
- [ ] **Response Envelope:** All success responses return `Response.success(c, data, message, status)`. List endpoints return `{ items, total, page, limit, totalPages }`.
- [ ] **Error Handling:** Central `errorHandler` registered in `app/index.ts`; errors thrown as `AppError`.
- [ ] **Database Schemas:** Tables defined in `src/db/schemas/<name>.schema.ts` and re-exported in `src/db/schemas/index.ts`.
- [ ] **Database Scripts:** Migrations and seeds live in `src/db/scripts/` and run via `bun src/db/scripts/<script>.ts`.
- [ ] **Background Processing:** Asynchronous heavy tasks offloaded to BullMQ queues (`src/queues/`).
- [ ] **Testing:** Unit and integration tests placed in `tests/unit/` and `tests/integration/` running cleanly on `bun test`.
- [ ] **Bun Primitives:** Native `Bun.password`, `Bun.file`, and root `export default { port, fetch: app.fetch }` utilized.
- [ ] **Fail-Fast Env:** `env.config.ts` validates all required environment variables with `z.object()`.
