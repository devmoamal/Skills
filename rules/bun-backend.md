# Bun Backend Architecture & Clean Code Rules

Apply these strict architectural standards and clean code guardrails when building, modifying, or refactoring this Bun backend service.

---

## 1. Directory & Feature Structure
- Group by domain features in `src/features/<feature>/`:
  - `routes/`: Hono inline route handlers, middleware bindings, parameter parsing, and response formatting.
  - `services/`: Pure business logic, domain invariants, workflow coordination, and `AppError` throws. (Zero Hono `c` context).
  - `repositories/`: Dedicated Drizzle ORM queries (`db.query` for reads, Query Builder for writes). Accepts `tx?: Transaction`.
  - `schemas/`: Zod request/response validation schemas with automatic string trimming.
  - `mappers/`: Explicit transformation functions (`toResponse(entity)`) ensuring zero sensitive field leakage.
- Centralize database tables in `src/db/schemas/` re-exported via `src/db/schemas/index.ts`.
- Place common Drizzle column helpers in `src/db/helpers/columns.ts` (UUIDv7 primary keys, `timestamps`).
- Place migration and seed runners in `src/db/scripts/` (`migrate.ts`, `seed.ts`).
- Dedicated test suites live in root `tests/` (`unit/`, `integration/`, `setup.ts`).

---

## 2. The 6 Banned Anti-Patterns (Clean Code Guardrails)
1. **Zero `any` or Loose Type Casting**: Everything must be strictly typed via Zod schemas, Drizzle models, or interfaces. Never use `as any` or `@ts-ignore`.
2. **Zero Raw `console.log` / `console.error`**: Always use the structured ANSI `logger` from `@/lib/logger`.
3. **Zero Magic Strings / Numbers**: Use strongly typed constants or enums for statuses and roles.
4. **Early Return Guard Clauses Only**: Ban deeply nested `if/else` ladders (maximum nesting depth: 2). Invert conditions and throw/return early.
5. **Zero N+1 Query Loops**: Never execute DB queries inside loops or `.map()`. Use relational queries (`with: { ... }`) or batch in-array lookups.
6. **Mandatory Feature DTO Mappers**: Never return raw database records directly to the client or use ad-hoc object destructuring. Always route through `mappers/`.

---

## 3. Context Typing & Correlation Tracing
- ALWAYS import `AppEnv` from `@/types/context` and initialize routers with `new Hono<AppEnv>()`.
- ALWAYS attach `requestIdMiddleware`: sets `X-Request-Id` in headers and context (`c.get("requestId")`), and prefixes all log lines.

---

## 4. Request Validation & Sanitization
- NEVER access raw untyped inputs via `c.req.json()` or `c.req.param()`.
- ALWAYS validate inputs at the route layer using `@/middlewares/validate.middleware`:
  - Request body: `validateBody(schema)`
  - URL parameters: `validateParams(schema)`
  - Query parameters: `validateQuery(schema)`
  - Form data: `validateFormData(schema)`
- In Zod schemas: enforce `.trim()` and `.toLowerCase()` on email/string inputs, and normalize empty strings `""` to `undefined` or `null`.

---

## 5. Response Standard & Pagination
- ALWAYS return responses through `@/lib/response`:
  - Success: `Response.success(c, data, message?, status?)` $\rightarrow$ `{ ok: true, data, message }`
  - Empty: `Response.empty(c, 204)`
  - Error: `Response.error(c, code, message, status)` $\rightarrow$ `{ ok: false, error, message }`
- Standardize collection endpoints on Page & Limit offset pagination via `@/lib/pagination`:
  - `{ ok: true, data: { items, total, page, limit, totalPages } }`.
- NEVER return ad-hoc, unstructured JSON like `{ success: true }` or `{ error: "msg" }`.

---

## 6. Transactions & Database Conventions
- Primary keys: Standardize on `uuidv7` for entity tables to prevent B-tree fragmentation. Use `integer` strictly for small lookup or sequence tables.
- Use shared `timestamps` helper for `createdAt` and `updatedAt`.
- Drizzle reads MUST use Relational Queries API (`db.query.*.findFirst`, `db.query.*.findMany`).
- Drizzle writes MUST use Query Builder (`db.insert()`, `db.update()`, `db.delete()`).
- Multi-table atomic mutations are orchestrated in the Service via `db.transaction(async (tx) => ...)`.
- Every repository method MUST accept `tx?: Transaction` and resolve client with `const client = tx ?? db`.

---

## 7. Error Handling
- NEVER write empty `catch` blocks or let unhandled errors crash the server.
- Throw custom domain errors extending `AppError` from `@/lib/error`:
  - `NotFoundError` (404), `ValidationError` (422), `BadRequestError` (400), `AuthorizationError` (401), `ForbiddenError` (403), `ConflictError` (409), `RateLimitError` (429).
- Central `errorHandler.middleware.ts` catches `AppError`, `ZodError`, malformed JSON, and critical 500 fallbacks.

---

## 8. Background Tasks & Graceful Shutdown
- Heavy tasks (emails, webhooks, processing) MUST be offloaded to BullMQ queues (`@/queues`).
- Intercept `SIGINT` and `SIGTERM` in `src/index.ts` to drain in-flight requests, close BullMQ workers, drain DB pool, and disconnect Redis cleanly.
