# Bun Backend Architecture & Clean Code Rules

Apply these strict architectural standards and clean code guardrails when building, modifying, or refactoring this Bun backend service.

---

## 1. Directory & Feature Structure
- Group by domain features in `src/features/<feature>/`:
  - `routes/`: Hono inline route handlers, middleware bindings, parameter parsing, and response formatting.
  - `services/`: Pure business logic, caching, domain invariants, workflow coordination, and `AppError` throws. (Zero Hono `c` context).
  - `repositories/`: Dedicated Drizzle ORM queries (`db.query` for reads, Query Builder for writes, soft-delete filters). Accepts `tx?: Transaction`.
  - `schemas/`: Zod request/response validation schemas with automatic string trimming.
  - `mappers/`: Explicit transformation functions (`toResponse(entity)`) ensuring zero sensitive field leakage.
- Centralize database tables in `src/db/schemas/` re-exported via `src/db/schemas/index.ts`.
- Place common Drizzle column helpers in `src/db/helpers/columns.ts` (UUIDv7 primary keys, `timestamps`, `softDelete`).
- Place migration and seed runners in `src/db/scripts/` (`migrate.ts`, `seed.ts`).
- Dedicated test suites live in root `tests/` (`unit/`, `integration/`, `setup.ts`).

---

## 2. The 6 Banned Anti-Patterns (Clean Code Guardrails)
1. **Zero `any` or Loose Type Casting**: Everything must be strictly typed via Zod schemas, Drizzle models, or interfaces. Never use `as any` or `@ts-ignore`.
2. **Zero Raw `console.log` / `console.error`**: Always use the structured ANSI `logger` from `@/lib/logger` with request ID correlation.
3. **Zero Magic Strings / Numbers**: Use strongly typed constants or enums for statuses and roles.
4. **Early Return Guard Clauses Only**: Ban deeply nested `if/else` ladders (maximum nesting depth: 2). Invert conditions and throw/return early.
5. **Zero N+1 Query Loops**: Never execute DB queries inside loops or `.map()`. Use relational queries (`with: { ... }`) or batch in-array lookups.
6. **Mandatory Feature DTO Mappers**: Never return raw database records directly to the client or use ad-hoc object destructuring. Always route through `mappers/`.

---

## 3. Database Conventions, Soft Deletes & Indexing
- Primary keys: Standardize on `uuidv7` for entity tables to prevent B-tree fragmentation. Use `integer` strictly for small lookup or sequence tables.
- Soft Deletes: Primary entities use `softDelete` (`deletedAt`). Repositories filter `isNull(table.deletedAt)` by default. Hard deletes are reserved for ephemeral data (tokens, cache, sessions).
- Mandatory Indexes:
  1. All foreign keys must be indexed.
  2. Index `deletedAt` on all soft-delete tables.
  3. Enforce partial unique indexes for unique columns with soft deletes (`where: isNull(deletedAt)`).

---

## 4. Context Typing & Correlation Tracing
- ALWAYS import `AppEnv` from `@/types/context` and initialize routers with `new Hono<AppEnv>()`.
- ALWAYS attach `requestIdMiddleware`: sets `X-Request-Id` in headers and context (`c.get("requestId")`), and prefixes all log lines.

---

## 5. Request Validation & Sanitization
- NEVER access raw untyped inputs via `c.req.json()` or `c.req.param()`.
- ALWAYS validate inputs at the route layer using `@/middlewares/validate.middleware`:
  - Request body: `validateBody(schema)`
  - URL parameters: `validateParams(schema)`
  - Query parameters: `validateQuery(schema)`
  - Form data: `validateFormData(schema)`
- In Zod schemas: enforce `.trim()` and `.toLowerCase()` on email/string inputs, and normalize empty strings `""` to `undefined` or `null`.

---

## 6. Response Standard & Pagination
- ALWAYS return responses through `@/lib/response`:
  - Success: `Response.success(c, data, message?, status?)` $\rightarrow$ `{ ok: true, data, message }`
  - Empty: `Response.empty(c, 204)`
  - Error: `Response.error(c, code, message, status)` $\rightarrow$ `{ ok: false, error, message }`
- Standardize collection endpoints on Page & Limit offset pagination via `@/lib/pagination`:
  - `{ ok: true, data: { items, total, page, limit, totalPages } }`.

---

## 7. Caching & Presigned Cloud Storage
- High-read queries use type-safe `cache.getOrSet(key, ttl, fetcher)` from `@/lib/cache`.
- Services explicitly invalidate cache keys (`cache.del(key)`) upon writes.
- File uploads: NEVER buffer uploads on the Bun server. Use presigned URLs (`@/lib/storage.ts`) for direct S3 / Cloudflare R2 uploads.

---

## 8. Real-Time & Graceful Teardown
- Real-time updates: Use Hono's native Bun WebSocket adapter (`createBunWebSocket`) connected to Redis Pub/Sub.
- Heavy tasks (emails, webhooks, AI jobs): Offloaded to BullMQ queues (`@/queues`).
- Intercept `SIGINT` and `SIGTERM` in `src/index.ts` to drain in-flight requests, close BullMQ workers, drain DB pool, and disconnect Redis cleanly.
