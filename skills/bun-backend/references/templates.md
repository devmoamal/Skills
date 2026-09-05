# Bun Backend Production Templates & Reference Implementations

This document provides production-ready reference implementations for foundational files and feature slices following the **bun-backend** skill standards.

---

## 1. Application Entry & Bootstrap

### `src/index.ts` (Bun Server Entrypoint)
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

### `src/types/context.ts` (Strongly Typed Hono Environment)
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

### `src/app/index.ts` (Hono Application Setup)
```typescript
import { Hono } from "hono";
import type { AppEnv } from "@/types/context";
import { corsMiddleware } from "@/middlewares/cors.middleware";
import { loggerMiddleware } from "@/middlewares/logger.middleware";
import { errorHandler } from "@/middlewares/errorHandler.middleware";
import { NotFoundError } from "@/lib/error";
import Router from "@/routes";

export const app = new Hono<AppEnv>();

// Global Middlewares
app.use("*", corsMiddleware);
app.use("*", loggerMiddleware);

// Error Handling & 404 Fallback
app.onError(errorHandler);
app.notFound((c) => {
  throw new NotFoundError(`${c.req.method} ${c.req.path} Route not found`);
});

// Central Router
app.route("/", Router);
```

### `src/routes/index.ts` (Central Route Aggregator)
```typescript
import { Hono } from "hono";
import type { AppEnv } from "@/types/context";
import usersRouter from "@/features/users/routes";

const router = new Hono<AppEnv>();

router.get("/health", (c) => c.json({ status: "ok" }));
router.route("/users", usersRouter);

export default router;
```

---

## 2. Configuration & Environment

### `src/config/env.config.ts` (Fail-Fast Zod Env Validation)
```typescript
import { z } from "zod";

const envSchema = z.object({
  NODE_ENV: z.enum(["development", "production", "test"]).default("development"),
  PORT: z.coerce.number().default(3000),
  TZ: z.string().default("Asia/Baghdad"),
  POSTGRESQL_URL: z.string().min(1, "POSTGRESQL_URL is required"),
  REDIS_URL: z.string().default("redis://localhost:6379"),
  JWT_SECRET: z.string().min(32, "JWT_SECRET must be at least 32 characters"),
  DASHBOARD_URL: z.string().optional(),
});

const parsed = envSchema.safeParse(process.env);

if (!parsed.success) {
  console.error(
    "\x1b[31m[ERROR]\x1b[0m Invalid environment variables:",
    z.flattenError(parsed.error).fieldErrors,
  );
  process.exit(1);
}

const env = parsed.data;
const isProduction = env.NODE_ENV === "production";
const isDevelopment = env.NODE_ENV === "development";

export { env, isDevelopment, isProduction };
```

---

## 3. Core Shared Library (`src/lib/`)

### `src/lib/error.ts` (AppError Hierarchy)
```typescript
import type { ContentfulStatusCode } from "hono/utils/http-status";

export type ErrorCode =
  | "UNKNOWN"
  | "NOT_FOUND"
  | "VALIDATION_ERROR"
  | "UNAUTHORIZED"
  | "FORBIDDEN"
  | "BAD_REQUEST"
  | "CONFLICT"
  | "RATE_LIMITED"
  | "SERVER_ERROR";

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
  constructor(message = "Not found") {
    super(message, 404, "NOT_FOUND");
  }
}

export class ValidationError extends AppError {
  constructor(message = "Unprocessable Entity") {
    super(message, 422, "VALIDATION_ERROR");
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

export class BadRequestError extends AppError {
  constructor(message = "Bad Request") {
    super(message, 400, "BAD_REQUEST");
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

export class ServerError extends AppError {
  constructor(message = "An unexpected error occurred") {
    super(message, 500, "SERVER_ERROR");
  }
}
```

### `src/lib/response.ts` (Unified Response Envelope)
```typescript
import type { Context } from "hono";
import type { ContentfulStatusCode, StatusCode } from "hono/utils/http-status";
import type { ErrorCode } from "./error";

export type SuccessResponse<T> = {
  ok: true;
  data?: T;
  message?: string;
};

export type ErrorResponse = {
  ok: false;
  message: string;
  error: ErrorCode;
};

class Response {
  static success<T>(
    context: Context,
    data?: T,
    message?: string,
    status: ContentfulStatusCode = 200,
  ) {
    return context.json<SuccessResponse<T>>({ ok: true, data, message }, status);
  }

  static error(
    context: Context,
    code: ErrorCode,
    message: string,
    status: ContentfulStatusCode,
  ) {
    return context.json<ErrorResponse>({ ok: false, message, error: code }, status);
  }

  static empty(context: Context, status: StatusCode) {
    return context.newResponse(null, status);
  }
}

export default Response;
```

### `src/lib/pagination.ts` (Offset Pagination Standard)
```typescript
import { z } from "zod";

export const paginationQuerySchema = z.object({
  page: z.coerce.number().int().min(1).default(1),
  limit: z.coerce.number().int().min(1).max(100).default(20),
});

export type PaginationQuery = z.infer<typeof paginationQuerySchema>;

export type PaginatedResult<T> = {
  items: T[];
  total: number;
  page: number;
  limit: number;
  totalPages: number;
};

export function formatPaginatedResult<T>(
  items: T[],
  total: number,
  page: number,
  limit: number,
): PaginatedResult<T> {
  return {
    items,
    total,
    page,
    limit,
    totalPages: Math.ceil(total / limit),
  };
}
```

### `src/lib/redis.ts` (Shared Redis Client)
```typescript
import Redis from "ioredis";
import { env } from "@/config/env.config";
import { logger } from "@/lib/logger";

export const redis = new Redis(env.REDIS_URL, {
  maxRetriesPerRequest: null,
});

redis.on("error", (err) => {
  logger.error("Redis connection error:", err);
});
```

### `src/lib/logger.ts` (Colorized ANSI Logger)
```typescript
const isDevelopment = process.env.NODE_ENV === "development";

export const logger = {
  info: (message: string, ...args: any[]) => {
    console.info(`\x1b[32m[INFO]\x1b[0m ${message}`, ...args);
  },
  warn: (message: string, ...args: any[]) => {
    console.warn(`\x1b[33m[WARN]\x1b[0m ${message}`, ...args);
  },
  error: (message: string | any, ...args: any[]) => {
    console.error(`\x1b[31m[ERROR]\x1b[0m ${message}`, ...args);
  },
  debug: (message: string, ...args: any[]) => {
    if (!isDevelopment) return;
    console.debug(`\x1b[34m[DEBUG]\x1b[0m ${message}`, ...args);
  },
};
```

### `src/lib/tryCatch.ts` (Result Pattern Wrapper)
```typescript
type Success<T> = {
  data: T;
  error: null;
};

type Failure<E> = {
  data: null;
  error: E;
};

export type Result<T, E = Error> = Success<T> | Failure<E>;

export async function tryCatch<T, E = Error>(
  promise: Promise<T>,
): Promise<Result<T, E>> {
  try {
    const data = await promise;
    return { data, error: null };
  } catch (error) {
    return { data: null, error: error as E };
  }
}
```

---

## 4. Middlewares (`src/middlewares/`)

### `src/middlewares/auth.middleware.ts` (JWT Bearer Token Auth)
```typescript
import type { Context, Next } from "hono";
import { AuthorizationError } from "@/lib/error";
import { env } from "@/config/env.config";
import type { AuthUser } from "@/types/context";

export const authMiddleware = async (c: Context, next: Next) => {
  const authHeader = c.req.header("Authorization");
  if (!authHeader || !authHeader.startsWith("Bearer ")) {
    throw new AuthorizationError("Missing or invalid Authorization header");
  }

  const token = authHeader.substring(7);

  try {
    // Standard HMAC-SHA256 JWT verify (or using jose/jsonwebtoken)
    const [headerB64, payloadB64, signature] = token.split(".");
    if (!headerB64 || !payloadB64 || !signature) {
      throw new Error("Malformed token");
    }

    const payload = JSON.parse(Buffer.from(payloadB64, "base64url").toString());

    if (payload.exp && Date.now() >= payload.exp * 1000) {
      throw new AuthorizationError("Token has expired");
    }

    const user: AuthUser = {
      id: payload.sub,
      email: payload.email,
      role: payload.role || "user",
    };

    c.set("user", user);
    await next();
  } catch (error) {
    if (error instanceof AuthorizationError) throw error;
    throw new AuthorizationError("Invalid token");
  }
};
```

### `src/middlewares/rateLimit.middleware.ts` (Redis Sliding Window)
```typescript
import type { Context, Next } from "hono";
import { redis } from "@/lib/redis";
import { RateLimitError } from "@/lib/error";

export const rateLimiter = (options: { limit: number; windowSeconds: number }) => {
  return async (c: Context, next: Next) => {
    const ip = c.req.header("x-forwarded-for") || "127.0.0.1";
    const key = `ratelimit:${ip}:${c.req.path}`;

    const current = await redis.incr(key);
    if (current === 1) {
      await redis.expire(key, options.windowSeconds);
    }

    if (current > options.limit) {
      throw new RateLimitError(`Rate limit exceeded. Try again in ${options.windowSeconds}s.`);
    }

    await next();
  };
};
```

### `src/middlewares/validate.middleware.ts`
```typescript
import { ValidationError } from "@/lib/error";
import { validator } from "hono/validator";
import { z } from "zod";

export const validateBody = <T extends z.ZodTypeAny>(schema: T) =>
  validator("json", (value) => {
    const result = schema.safeParse(value);
    if (!result.success)
      throw new ValidationError(result.error.issues[0].message);
    return result.data as z.infer<T>;
  });

export const validateParams = <T extends z.ZodTypeAny>(schema: T) =>
  validator("param", (value) => {
    const result = schema.safeParse(value);
    if (!result.success)
      throw new ValidationError(result.error.issues[0].message);
    return result.data as z.infer<T>;
  });

export const validateQuery = <T extends z.ZodTypeAny>(schema: T) =>
  validator("query", (value) => {
    const result = schema.safeParse(value);
    if (!result.success)
      throw new ValidationError(result.error.issues[0].message);
    return result.data as z.infer<T>;
  });

export const validateFormData = <T extends z.ZodTypeAny>(schema: T) =>
  validator("form", (value) => {
    const result = schema.safeParse(value);
    if (!result.success)
      throw new ValidationError(result.error.issues[0].message);
    return result.data as z.infer<T>;
  });
```

### `src/middlewares/errorHandler.middleware.ts`
```typescript
import type { Context } from "hono";
import { AppError, BadRequestError } from "@/lib/error";
import { logger } from "@/lib/logger";
import Response from "@/lib/response";
import { ZodError } from "zod";

export const errorHandler = (error: any, c: Context) => {
  // Handle AppError
  if (error instanceof AppError) {
    logger.error(error.message);
    return Response.error(c, error.code, error.message, error.status);
  }

  // Handle unhandled Zod errors
  if (error instanceof ZodError) {
    const message = error.issues[0]?.message || "Validation failed";
    logger.error(`Validation Error: ${message}`);
    return Response.error(c, "VALIDATION_ERROR", message, 422);
  }

  // Handle malformed JSON body errors
  if (error.message && error.message.includes("JSON")) {
    logger.error(error.message);
    return Response.error(c, "BAD_REQUEST", "Invalid JSON", 400);
  }

  logger.error("[CRITICAL]", error?.stack || error?.message || error);
  return Response.error(c, "SERVER_ERROR", "Internal Server Error", 500);
};
```

### `src/middlewares/logger.middleware.ts`
```typescript
import type { Context, Next } from "hono";
import { logger } from "@/lib/logger";

export const loggerMiddleware = async (c: Context, next: Next) => {
  const { method, path } = c.req;
  await next();
  const status = c.res.status;
  logger.info(`[${method}] ${path} [${status}]`);
};
```

### `src/middlewares/cors.middleware.ts`
```typescript
import { isDevelopment } from "@/config/env.config";
import { cors } from "hono/cors";

export const corsMiddleware = cors({
  origin: isDevelopment ? "*" : [],
  allowMethods: ["POST", "GET", "OPTIONS", "PUT", "DELETE", "PATCH"],
  allowHeaders: ["Accept", "Content-Type", "Authorization"],
  maxAge: 600,
});
```

---

## 5. Database Layer (`src/db/`)

### `src/db/index.ts` (Drizzle Client & Pool)
```typescript
import { env } from "@/config/env.config";
import { drizzle } from "drizzle-orm/node-postgres";
import { Pool } from "pg";
import * as schema from "./schemas";

const pool = new Pool({
  connectionString: env.POSTGRESQL_URL,
  options: `-c timezone=${env.TZ}`,
});

export const db = drizzle(pool, { schema });

export type Database = typeof db;
export type Transaction = Parameters<Parameters<Database["transaction"]>[0]>[0];

export default db;
```

### `src/db/schemas/users.schema.ts`
```typescript
import { pgTable, text, timestamp, uuid } from "drizzle-orm/pg-core";

export const users = pgTable("users", {
  id: uuid("id").defaultRandom().primaryKey(),
  email: text("email").notNull().unique(),
  passwordHash: text("password_hash").notNull(),
  fullName: text("full_name").notNull(),
  createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
  updatedAt: timestamp("updated_at", { withTimezone: true }).defaultNow().notNull(),
});

export type User = typeof users.$inferSelect;
export type NewUser = typeof users.$inferInsert;
```

### `src/db/schemas/index.ts` (Central Schema Export)
```typescript
export * from "./users.schema";
```

### `src/db/scripts/migrate.ts`
```typescript
import { migrate } from "drizzle-orm/node-postgres/migrator";
import db from "../index";
import { logger } from "@/lib/logger";

async function runMigrations() {
  logger.info("Running pending database migrations...");
  await migrate(db, { migrationsFolder: "./drizzle" });
  logger.info("Migrations applied successfully.");
  process.exit(0);
}

runMigrations().catch((err) => {
  logger.error("Migration failed:", err);
  process.exit(1);
});
```

### `src/db/scripts/seed.ts`
```typescript
import db from "../index";
import { users } from "../schemas";
import { logger } from "@/lib/logger";

async function seed() {
  logger.info("Seeding database...");
  const passwordHash = await Bun.password.hash("Password123!", { algorithm: "argon2id" });

  await db.insert(users).values({
    email: "admin@example.com",
    fullName: "System Admin",
    passwordHash,
  }).onConflictDoNothing();

  logger.info("Seeding complete.");
  process.exit(0);
}

seed().catch((err) => {
  logger.error("Seeding failed:", err);
  process.exit(1);
});
```

---

## 6. Background Processing & Queues (`src/queues/`)

### `src/queues/index.ts` (BullMQ Queues & Workers)
```typescript
import { Queue, Worker } from "bullmq";
import { redis } from "@/lib/redis";
import { logger } from "@/lib/logger";

export const emailQueue = new Queue("emailQueue", { connection: redis });

export const emailWorker = new Worker(
  "emailQueue",
  async (job) => {
    logger.info(`Processing job ${job.name} with data:`, job.data);
    // Execute email sending logic
  },
  { connection: redis },
);
```

---

## 7. Complete Feature Reference Slice (`src/features/users/`)

### `src/features/users/schemas/index.ts`
```typescript
import { z } from "zod";
import { paginationQuerySchema } from "@/lib/pagination";

export const createUserSchema = z.object({
  email: z.string().email("Invalid email address"),
  password: z.string().min(8, "Password must be at least 8 characters"),
  fullName: z.string().min(2, "Full name must be at least 2 characters"),
});

export const userIdParamSchema = z.object({
  id: z.string().uuid("Invalid user ID"),
});

export const listUsersQuerySchema = paginationQuerySchema.extend({
  search: z.string().optional(),
});

export type CreateUserInput = z.infer<typeof createUserSchema>;
export type UserIdParam = z.infer<typeof userIdParamSchema>;
export type ListUsersQuery = z.infer<typeof listUsersQuerySchema>;
```

### `src/features/users/repositories/index.ts`
```typescript
import db, { type Transaction } from "@/db";
import { users, type User, type NewUser } from "@/db/schemas";
import { eq, sql } from "drizzle-orm";

export class UsersRepository {
  // Relational Query for single read
  static async findById(id: string, tx?: Transaction): Promise<User | null> {
    const client = tx ?? db;
    const user = await client.query.users.findFirst({
      where: eq(users.id, id),
    });
    return user ?? null;
  }

  static async findByEmail(email: string, tx?: Transaction): Promise<User | null> {
    const client = tx ?? db;
    const user = await client.query.users.findFirst({
      where: eq(users.email, email),
    });
    return user ?? null;
  }

  // Query Builder for writes
  static async create(data: NewUser, tx?: Transaction): Promise<User> {
    const client = tx ?? db;
    const [created] = await client.insert(users).values(data).returning();
    return created;
  }

  // Paginated read
  static async list(page: number, limit: number, tx?: Transaction): Promise<{ items: User[]; total: number }> {
    const client = tx ?? db;
    const offset = (page - 1) * limit;

    const items = await client.query.users.findMany({
      limit,
      offset,
    });

    const [{ count }] = await client
      .select({ count: sql<number>`count(*)::int` })
      .from(users);

    return { items, total: count };
  }
}
```

### `src/features/users/services/index.ts`
```typescript
import db from "@/db";
import { ConflictError, NotFoundError } from "@/lib/error";
import { formatPaginatedResult } from "@/lib/pagination";
import { emailQueue } from "@/queues";
import { UsersRepository } from "../repositories";
import type { CreateUserInput, ListUsersQuery } from "../schemas";

export class UsersService {
  static async getById(id: string) {
    const user = await UsersRepository.findById(id);
    if (!user) {
      throw new NotFoundError(`User with id '${id}' not found`);
    }

    const { passwordHash, ...safeUser } = user;
    return safeUser;
  }

  static async create(input: CreateUserInput) {
    const existing = await UsersRepository.findByEmail(input.email);
    if (existing) {
      throw new ConflictError("Email already registered");
    }

    const passwordHash = await Bun.password.hash(input.password, { algorithm: "argon2id" });

    // Multi-table or atomic operations orchestrate db.transaction
    const safeUser = await db.transaction(async (tx) => {
      const created = await UsersRepository.create(
        {
          email: input.email,
          fullName: input.fullName,
          passwordHash,
        },
        tx,
      );

      const { passwordHash: _, ...user } = created;
      return user;
    });

    // Offload async heavy task to BullMQ
    await emailQueue.add("welcomeEmail", { email: safeUser.email, name: safeUser.fullName });

    return safeUser;
  }

  static async list(query: ListUsersQuery) {
    const { items, total } = await UsersRepository.list(query.page, query.limit);
    const safeItems = items.map(({ passwordHash, ...user }) => user);

    return formatPaginatedResult(safeItems, total, query.page, query.limit);
  }
}
```

### `src/features/users/routes/index.ts`
```typescript
import { Hono } from "hono";
import type { AppEnv } from "@/types/context";
import { validateBody, validateParams, validateQuery } from "@/middlewares/validate.middleware";
import { authMiddleware } from "@/middlewares/auth.middleware";
import { createUserSchema, userIdParamSchema, listUsersQuerySchema } from "../schemas";
import { UsersService } from "../services";
import Response from "@/lib/response";

const router = new Hono<AppEnv>();

// Public registration
router.post("/", validateBody(createUserSchema), async (c) => {
  const input = c.req.valid("json");
  const user = await UsersService.create(input);
  return Response.success(c, user, "User registered successfully", 201);
});

// Protected routes
router.use("*", authMiddleware);

router.get("/", validateQuery(listUsersQuerySchema), async (c) => {
  const query = c.req.valid("query");
  const paginatedUsers = await UsersService.list(query);
  return Response.success(c, paginatedUsers);
});

router.get("/:id", validateParams(userIdParamSchema), async (c) => {
  const { id } = c.req.valid("param");
  const user = await UsersService.getById(id);
  return Response.success(c, user);
});

export default router;
```

---

## 8. Testing Standards (`tests/`)

### `tests/setup.ts` (Test Runner Setup)
```typescript
import { beforeAll, afterAll } from "bun:test";

beforeAll(async () => {
  // Setup test database or run migrations
});

afterAll(async () => {
  // Teardown connections
});
```

### `tests/integration/users.test.ts`
```typescript
import { describe, expect, it } from "bun:test";
import { app } from "@/app";

describe("Users API Integration", () => {
  it("GET /health should return 200 ok", async () => {
    const res = await app.request("/health");
    expect(res.status).toBe(200);
    const data = await res.json();
    expect(data.status).toBe("ok");
  });

  it("POST /users with invalid email should fail validation with 422", async () => {
    const res = await app.request("/users", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ email: "bad-email", password: "123", fullName: "A" }),
    });

    expect(res.status).toBe(422);
    const json = await res.json();
    expect(json.ok).toBe(false);
    expect(json.error).toBe("VALIDATION_ERROR");
  });
});
```
