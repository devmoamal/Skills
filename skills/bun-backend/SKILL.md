---
name: bun-backend
description: Enforce high-performance, modular, and type-safe backend architecture for Bun using Hono, Drizzle ORM, Zod, and a strict 3-tier vertical slice (Route -> Service -> Repository) design. Use whenever designing, building, scaffolding, refactoring, or auditing Bun backend APIs, services, routes, database schemas, or middlewares.
---

# Bun Backend: Production Architecture & Standards

Build fast, bulletproof, type-safe backend services powered by Bun, Hono, Drizzle ORM, and Zod.

```
Request → Middlewares → Feature Route → Zod Validator → Domain Service → Drizzle Repository → Database Pool
                ↓                                               ↓
      Central Error Handler                             AppError Hierarchy
                ↓                                               ↓
     Response.error(c, ...)                          Response.success(c, ...)
```

Every endpoint, query, and service must adhere to strict separation of concerns, fail-fast schema validation, and unified response envelopes.

---

## Operating Modes

### 1. Build Mode (Default)
When scaffolding new features, endpoints, or services:
* Structure all domain features under `src/features/<feature>/` with explicit `routes/`, `services/`, `repositories/`, and `schemas/`.
* Strictly enforce the 3-tier boundary: **Route $\rightarrow$ Service $\rightarrow$ Repository**. Never bypass layers.
* Validate all request inputs (body, params, query) using `validateBody`, `validateParams`, `validateQuery` before reaching business logic.
* Standardize all outputs through `Response.success()` and `Response.error()`.
* Keep database table definitions modularized in `src/db/schemas/` and CLI scripts in `src/db/scripts/`.

### 2. Audit / Review Mode
When reviewing, refactoring, or debugging existing backend code:
* **Gate 1 (Layer Leakage):** Flag and eliminate any database calls in routes or services. All SQL / Drizzle queries must live in `repositories/`.
* **Gate 2 (Validation Coverage):** Check that every endpoint parses inputs through Zod schemas. Ban raw, untyped `c.req.json()` or `c.req.param()`.
* **Gate 3 (Error & Response Shape):** Verify that all thrown errors inherit from `AppError` and all responses use the unified JSON envelope `{ ok: true, data }` or `{ ok: false, error, message }`.
* **Gate 4 (Bun Idioms):** Replace legacy Node polyfills with native Bun primitives (`Bun.password`, `Bun.file`, `Bun.serve` export).

---

## Standard Directory Layout

Every Bun backend project must follow this standard folder structure:

```bash
src/
├── index.ts                    # Bun server entry point (Bun.serve export & startup log)
│
├── app/
│   └── index.ts                # Hono instantiation, global middleware registration, router mounting
│
├── config/
│   └── env.config.ts           # Fail-fast environment variable validation (Zod)
│
├── features/                   # Vertical domain slices
│   └── [feature-name]/         # e.g. auth, users, products, orders
│       ├── routes/
│       │   └── index.ts        # HTTP routes, validation bindings, response formatting
│       ├── services/
│       │   └── index.ts        # Pure business logic & orchestration (throws AppError)
│       ├── repositories/
│       │   └── index.ts        # Direct Drizzle ORM queries & database mutations
│       ├── schemas/
│       │   └── index.ts        # Zod DTOs for request body, params, query, and response types
│       └── types/
│           └── index.ts        # (Optional) Feature-specific domain interfaces & models
│
├── db/
│   ├── index.ts                # Connection pool & Drizzle client instance
│   ├── schemas/                # Modular database table definitions
│   │   ├── users.schema.ts     # Domain table definitions
│   │   ├── [entity].schema.ts
│   │   └── index.ts            # Central re-export of all database schemas & relations
│   └── scripts/                # Dedicated database CLI scripts
│       ├── migrate.ts          # Programmatic migration runner
│       └── seed.ts             # Database seeder
│
├── middlewares/                # Cross-cutting HTTP middlewares
│   ├── validate.middleware.ts  # Zod validation bindings (validateBody, validateParams, etc.)
│   ├── errorHandler.middleware.ts # Central error boundary catching AppError & ZodError
│   ├── logger.middleware.ts    # Request/Response logger with colored tags & status codes
│   └── cors.middleware.ts      # Environment-aware CORS configuration
│
├── lib/                        # Core shared utilities & foundational primitives
│   ├── error.ts                # AppError hierarchy & ErrorCode types
│   ├── response.ts             # Standardized Response envelope helpers
│   ├── logger.ts               # Colorized, dependency-isolated ANSI logger
│   └── tryCatch.ts             # Functional Result<T, E> wrapper
│
├── routes/
│   └── index.ts                # Central router aggregating all feature routers
│
└── scripts/                    # General Bun CLI tasks (e.g. createAdmin.ts, benchmark.ts)
```

---

## The 7 Core Engineering Pillars

### Pillar 1: Strict 3-Tier Vertical Slice
Features must be self-contained and respect a strict 3-tier boundary:

```
[ HTTP Route ]  ──(calls)──>  [ Domain Service ]  ──(calls)──>  [ Repository ]  ──(queries)──> [ Database ]
      │                              │                                │
 Validates Input               Business Rules                  Drizzle Queries
 Formats Response              Throws AppError                 Returns DB Models
 (Zero DB / Zero Rules)        (Zero Hono Context / Zero DB)   (Zero HTTP / Zero Rules)
```

1. **`routes/index.ts` (HTTP Boundary)**:
   * Only handles HTTP concerns: route registration, middleware attachment, parameter passing, and returning `Response.success()`.
   * **Rule:** NEVER write business logic, authorization checks, or database queries in routes.
2. **`services/index.ts` (Domain Logic Boundary)**:
   * Executes business workflows, domain validation, permission checks, and coordinates multiple repositories.
   * Throws typed domain errors (`NotFoundError`, `ConflictError`, `BadRequestError`).
   * **Rule:** NEVER pass Hono `Context` (`c`) to a service. Services are pure TypeScript functions accepting plain arguments and returning plain data or throwing `AppError`.
3. **`repositories/index.ts` (Data Access Boundary)**:
   * The ONLY place in the codebase where Drizzle ORM queries (`db.select()`, `db.insert()`, `db.update()`, `db.delete()`) are executed.
   * Encapsulates joins, filters, ordering, and transactions.
   * **Rule:** NEVER handle HTTP request/response or write business rules in repositories.

---

### Pillar 2: Schema-First Validation (Zero Unchecked Inputs)
Every incoming request must be validated using Zod schemas before hitting any service or repository:

* **Validate Request Body:** `validateBody(schema)`
* **Validate URL Params:** `validateParams(schema)`
* **Validate Query Strings:** `validateQuery(schema)`
* **Validate Form Data:** `validateFormData(schema)`

```typescript
// features/users/routes/index.ts
import { Hono } from "hono";
import { validateBody, validateParams } from "@/middlewares/validate.middleware";
import { createUserSchema, userIdParamSchema } from "../schemas";
import { UsersService } from "../services";
import Response from "@/lib/response";

const router = new Hono();

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
* **Single Source of Truth:** Derive TypeScript input types directly from schemas:
  ```typescript
  export type CreateUserInput = z.infer<typeof createUserSchema>;
  ```

---

### Pillar 3: Unified Response Envelopes
All API responses must follow a strict, predictable JSON envelope format:

#### Success Response Envelope:
```typescript
export type SuccessResponse<T> = {
  ok: true;
  data?: T;
  message?: string;
};
```
* Use `Response.success(c, data, message?, status?)` for all 2xx responses.
* Use `Response.empty(c, 204)` for empty 204 No Content responses.

#### Error Response Envelope:
```typescript
export type ErrorResponse = {
  ok: false;
  message: string;
  error: ErrorCode;
};
```
* Use `Response.error(c, code, message, status)` for all errors.
* **Rule:** Never return inconsistent ad-hoc JSON like `{ success: true, user }`, `{ error: "msg" }`, or raw arrays. The client must always rely on `ok: true` or `ok: false`.

---

### Pillar 4: Domain Error Hierarchy & Central Error Boundary
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

### Pillar 5: Scalable Drizzle Data Layer
Keep database schemas modular and migrations decoupled from application startup:

1. **Modular Schemas in `src/db/schemas/`**:
   * One file per domain (e.g. `users.schema.ts`, `orders.schema.ts`).
   * Explicit relations defined in schemas.
   * Central `src/db/schemas/index.ts` re-exports all tables so Drizzle Kit discovers them cleanly.
2. **Dedicated CLI Scripts in `src/db/scripts/`**:
   * `migrate.ts`: Runs pending migrations via `drizzle-orm/node-postgres/migrator`.
   * `seed.ts`: Inserts initial seed data.
   * Run with native Bun: `bun src/db/scripts/migrate.ts` and `bun src/db/scripts/seed.ts`.
3. **Transactions**:
   * Wrap multi-table or atomic state mutations inside `db.transaction(async (tx) => { ... })`.
   * Repositories should accept an optional `tx` instance to participate in caller-managed transactions.

---

### Pillar 6: Bun Runtime Mastery & Native Primitives
Take full advantage of Bun's native performance instead of dragging heavy Node dependencies:

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
  Use `Bun.password` with Argon2id instead of slow `bcrypt`/`bcryptjs`:
  ```typescript
  const hash = await Bun.password.hash(password, { algorithm: "argon2id" });
  const isValid = await Bun.password.verify(password, hash);
  ```
* **Native File I/O:**
  Use `Bun.file(path)` and `await Bun.write(path, data)` for zero-copy high-performance I/O instead of `node:fs`.
* **Native Hashing & Crypto:**
  Use `Bun.hash` for ultra-fast non-cryptographic checksums and `crypto.randomUUID()` for IDs.
* **Testing:**
  Run native `bun test`. No Jest, Vitest, or Babel config required.

---

### Pillar 7: Fail-Fast Configuration & Observability
Never let a server start with invalid or missing configuration:

1. **Validate Environment at Boot (`src/config/env.config.ts`)**:
   * Parse `process.env` through a strict Zod schema at startup.
   * If validation fails, immediately log the missing fields and exit: `process.exit(1)`.
   * Export typed constants `env`, `isDevelopment`, and `isProduction`.
2. **Circular-Dependency-Free Logger (`src/lib/logger.ts`)**:
   * The logger reads `process.env.NODE_ENV` directly rather than importing `env.config.ts`, preventing bootstrapping deadlocks.
   * Use structured ANSI colors: `\x1b[32m[INFO]\x1b[0m`, `\x1b[33m[WARN]\x1b[0m`, `\x1b[31m[ERROR]\x1b[0m`, `\x1b[34m[DEBUG]\x1b[0m`.
3. **HTTP Request Logging Middleware**:
   * Logs every incoming call: `[METHOD] /path [STATUS]`.

---

## The 3 Audit Gates (Verification)

Before considering any backend feature or refactor finished, evaluate it against all three gates:

### Gate 1: The Layer Separation Gate
> *Does any route contain direct SQL/Drizzle calls? Does any service accept Hono Context or touch `c.req`?*
* **If yes:** Refactor immediately. Route handles HTTP $\rightarrow$ Service handles logic $\rightarrow$ Repository handles database queries.

### Gate 2: The Validation Gate
> *Are all route parameters, query parameters, and request bodies strictly validated with Zod via `validateBody` / `validateParams` / `validateQuery`?*
* **If yes:** Input is safe. If no, apply the corresponding validation middleware before shipping.

### Gate 3: The Error & Response Gate
> *Does every endpoint return `Response.success()`? Does every thrown error inherit from `AppError`? Are uncaught errors handled by `errorHandler`?*
* **If yes:** Responses are predictable and type-safe. Never return ad-hoc JSON shapes.

---

## Pre-Flight Checklist

Before presenting or shipping backend code, verify each checkbox:

- [ ] **Folder Structure:** Feature lives in `src/features/<feature>/` with `routes/`, `services/`, `repositories/`, `schemas/`.
- [ ] **Route Isolation:** Route only handles validation bindings and calls service methods.
- [ ] **Service Isolation:** Service contains pure domain logic, accepts primitives/DTOs, and throws `AppError` subclasses.
- [ ] **Repository Isolation:** Drizzle queries are strictly encapsulated inside repository functions.
- [ ] **Input Validation:** Request body, params, and query parameters are validated with Zod.
- [ ] **Response Envelope:** All success responses return `Response.success(c, data, message, status)`.
- [ ] **Error Handling:** Central `errorHandler` registered in `app/index.ts`; errors thrown as `AppError`.
- [ ] **Database Schemas:** Tables defined in `src/db/schemas/<name>.schema.ts` and re-exported in `src/db/schemas/index.ts`.
- [ ] **Database Scripts:** Migrations and seeds live in `src/db/scripts/` and run via `bun src/db/scripts/<script>.ts`.
- [ ] **Bun Primitives:** Native `Bun.password`, `Bun.file`, and root `export default { port, fetch: app.fetch }` utilized.
- [ ] **Fail-Fast Env:** `env.config.ts` validates all required environment variables with `z.object()`.
