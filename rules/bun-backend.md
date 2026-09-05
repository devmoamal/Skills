# Bun Backend Architecture & Engineering Rules

Apply these strict architectural standards when building, modifying, or refactoring this Bun backend service.

---

## 1. Directory & Feature Structure
- Group by domain features in `src/features/<feature>/`.
- Inside every feature, maintain strict separation of concerns across subdirectories:
  - `routes/`: Hono HTTP routing, middleware attachments, parameter passing, and response envelopes.
  - `services/`: Pure business logic, domain invariants, workflow coordination, and `AppError` throws. (Zero Hono `c` context, Zero Drizzle queries).
  - `repositories/`: Dedicated Drizzle ORM queries (`db.select()`, `db.insert()`, etc.).
  - `schemas/`: Zod request/response validation schemas and inferred DTO types.
- Centralize database tables in `src/db/schemas/` re-exported via `src/db/schemas/index.ts`.
- Place migration and seed runners in `src/db/scripts/` (`migrate.ts`, `seed.ts`).

---

## 2. Request Validation
- NEVER access raw untyped inputs via `c.req.json()` or `c.req.param()`.
- ALWAYS validate inputs at the route layer using `@/middlewares/validate.middleware`:
  - Request body: `validateBody(schema)`
  - URL parameters: `validateParams(schema)`
  - Query parameters: `validateQuery(schema)`
  - Form data: `validateFormData(schema)`
- Retrieve validated data using `c.req.valid("json")`, `c.req.valid("param")`, or `c.req.valid("query")`.

---

## 3. Response Standard & Envelope
- ALWAYS return responses through `@/lib/response`:
  - Success: `Response.success(c, data, message?, status?)` $\rightarrow$ `{ ok: true, data, message }`
  - Empty: `Response.empty(c, 204)`
  - Error: `Response.error(c, code, message, status)` $\rightarrow$ `{ ok: false, error, message }`
- NEVER return ad-hoc, unstructured JSON like `{ success: true }` or `{ error: "msg" }`.

---

## 4. Error Handling
- NEVER write empty `catch` blocks or let unhandled errors crash the server.
- Throw custom domain errors extending `AppError` from `@/lib/error`:
  - `NotFoundError` (404)
  - `ValidationError` (422)
  - `BadRequestError` (400)
  - `AuthorizationError` (401)
  - `ForbiddenError` (403)
  - `ConflictError` (409)
- All unhandled exceptions are caught by `errorHandler.middleware.ts` and returned as safe 500 responses without leaking internal stack traces.

---

## 5. Bun Runtime Idioms
- Export native Bun HTTP server in root `src/index.ts`:
  ```typescript
  export default { port: env.PORT, fetch: app.fetch };
  ```
- Use `Bun.password.hash()` / `Bun.password.verify()` with Argon2id for password hashing.
- Use `Bun.file()` / `Bun.write()` for file streaming.
- Use native `bun test` for test suites.
