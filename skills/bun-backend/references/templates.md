# Bun Backend Production Templates & Reference Implementations

This document provides complete, production-ready reference implementations for foundational files and feature slices following the **bun-backend** skill standards.

---

## 1. Application Entry & Bootstrap

### `src/index.ts` (Bun Server Entrypoint with WebSockets & Graceful Shutdown)
```typescript
import { app, websocket } from "@/app";
import { env } from "@/config/env.config";
import { pool } from "@/db";
import { redis, redisSub } from "@/lib/redis";
import { emailWorker } from "@/queues";
import { logger } from "@/lib/logger";

const server = Bun.serve({
  port: env.PORT,
  fetch: app.fetch,
  websocket,
});

logger.info(`Server running on port ${env.PORT}`);

// Graceful Shutdown Handler
const shutdown = async (signal: string) => {
  logger.info(`Received ${signal}. Starting graceful shutdown...`);

  // Stop accepting new connections
  server.stop();

  try {
    // 1. Close BullMQ workers
    logger.info("Closing queue workers...");
    await emailWorker.close();

    // 2. Drain and close database pool
    logger.info("Draining database connections...");
    await pool.end();

    // 3. Disconnect Redis clients
    logger.info("Closing Redis connections...");
    await redisSub.quit();
    await redis.quit();

    logger.info("Graceful shutdown completed successfully.");
    process.exit(0);
  } catch (error) {
    logger.error("Error during graceful shutdown:", error);
    process.exit(1);
  }
};

process.on("SIGINT", () => shutdown("SIGINT"));
process.on("SIGTERM", () => shutdown("SIGTERM"));

export default server;
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

### `src/app/index.ts` (Hono Application & WebSocket Setup)
```typescript
import { Hono } from "hono";
import { createBunWebSocket } from "hono/bun";
import type { AppEnv } from "@/types/context";
import { requestIdMiddleware } from "@/middlewares/requestId.middleware";
import { corsMiddleware } from "@/middlewares/cors.middleware";
import { loggerMiddleware } from "@/middlewares/logger.middleware";
import { errorHandler } from "@/middlewares/errorHandler.middleware";
import { NotFoundError } from "@/lib/error";
import Router from "@/routes";
import { setupWebSocketRoutes } from "@/ws";

const { upgradeWebSocket, websocket } = createBunWebSocket();

export const app = new Hono<AppEnv>();

// Global Middlewares
app.use("*", requestIdMiddleware);
app.use("*", corsMiddleware);
app.use("*", loggerMiddleware);

// Error Handling & 404 Fallback
app.onError(errorHandler);
app.notFound((c) => {
  throw new NotFoundError(`${c.req.method} ${c.req.path} Route not found`);
});

// Setup WebSocket endpoint
setupWebSocketRoutes(app, upgradeWebSocket);

// Central Router
app.route("/", Router);

export { websocket };
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
  S3_BUCKET: z.string().min(1, "S3_BUCKET is required"),
  S3_REGION: z.string().default("auto"),
  S3_ENDPOINT: z.string().optional(),
  S3_ACCESS_KEY_ID: z.string().min(1, "S3_ACCESS_KEY_ID is required"),
  S3_SECRET_ACCESS_KEY: z.string().min(1, "S3_SECRET_ACCESS_KEY is required"),
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

export class NotFoundError extends AppError { constructor(message = "Not found") { super(message, 404, "NOT_FOUND"); } }
export class ValidationError extends AppError { constructor(message = "Unprocessable Entity") { super(message, 422, "VALIDATION_ERROR"); } }
export class AuthorizationError extends AppError { constructor(message = "Unauthorized") { super(message, 401, "UNAUTHORIZED"); } }
export class ForbiddenError extends AppError { constructor(message = "Forbidden") { super(message, 403, "FORBIDDEN"); } }
export class BadRequestError extends AppError { constructor(message = "Bad Request") { super(message, 400, "BAD_REQUEST"); } }
export class ConflictError extends AppError { constructor(message = "Conflict") { super(message, 409, "CONFLICT"); } }
export class RateLimitError extends AppError { constructor(message = "Too Many Requests") { super(message, 429, "RATE_LIMITED"); } }
export class ServerError extends AppError { constructor(message = "An unexpected error occurred") { super(message, 500, "SERVER_ERROR"); } }
```

### `src/lib/response.ts` (Unified Response Envelope)
```typescript
import type { Context } from "hono";
import type { ContentfulStatusCode, StatusCode } from "hono/utils/http-status";
import type { ErrorCode } from "./error";

export type SuccessResponse<T> = { ok: true; data?: T; message?: string; };
export type ErrorResponse = { ok: false; message: string; error: ErrorCode; };

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

### `src/lib/redis.ts` (Shared Redis Clients)
```typescript
import Redis from "ioredis";
import { env } from "@/config/env.config";
import { logger } from "@/lib/logger";

export const redis = new Redis(env.REDIS_URL, { maxRetriesPerRequest: null });
export const redisSub = new Redis(env.REDIS_URL, { maxRetriesPerRequest: null });

redis.on("error", (err) => logger.error("Redis client error:", err));
redisSub.on("error", (err) => logger.error("Redis subscriber error:", err));
```

### `src/lib/cache.ts` (Type-Safe Caching & Eviction)
```typescript
import { redis } from "./redis";
import { logger } from "./logger";

export const cache = {
  async getOrSet<T>(key: string, ttlSeconds: number, fetcher: () => Promise<T>): Promise<T> {
    try {
      const cached = await redis.get(key);
      if (cached) {
        return JSON.parse(cached) as T;
      }
    } catch (err) {
      logger.warn(`Cache get error for key '${key}':`, err);
    }

    const freshData = await fetcher();

    try {
      if (freshData !== null && freshData !== undefined) {
        await redis.set(key, JSON.stringify(freshData), "EX", ttlSeconds);
      }
    } catch (err) {
      logger.warn(`Cache set error for key '${key}':`, err);
    }

    return freshData;
  },

  async del(key: string): Promise<void> {
    try {
      await redis.del(key);
    } catch (err) {
      logger.warn(`Cache delete error for key '${key}':`, err);
    }
  },

  async delPrefix(prefix: string): Promise<void> {
    try {
      const keys = await redis.keys(`${prefix}:*`);
      if (keys.length > 0) {
        await redis.del(...keys);
      }
    } catch (err) {
      logger.warn(`Cache delete prefix error for '${prefix}':`, err);
    }
  },
};
```

### `src/lib/storage.ts` (Presigned URLs for S3 / Cloudflare R2)
```typescript
import { S3Client, PutObjectCommand, GetObjectCommand } from "@aws-sdk/client-s3";
import { getSignedUrl } from "@aws-sdk/s3-request-presigner";
import { env } from "@/config/env.config";

export const s3 = new S3Client({
  region: env.S3_REGION,
  endpoint: env.S3_ENDPOINT,
  credentials: {
    accessKeyId: env.S3_ACCESS_KEY_ID,
    secretAccessKey: env.S3_SECRET_ACCESS_KEY,
  },
});

export const storage = {
  async getPresignedUploadUrl(key: string, contentType: string, expiresIn = 300) {
    const command = new PutObjectCommand({
      Bucket: env.S3_BUCKET,
      Key: key,
      ContentType: contentType,
    });
    return getSignedUrl(s3, command, { expiresIn });
  },

  async getPresignedDownloadUrl(key: string, expiresIn = 3600) {
    const command = new GetObjectCommand({
      Bucket: env.S3_BUCKET,
      Key: key,
    });
    return getSignedUrl(s3, command, { expiresIn });
  },
};
```

### `src/lib/logger.ts` (ANSI Logger with Request ID)
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

---

## 4. Middlewares (`src/middlewares/`)

### `src/middlewares/requestId.middleware.ts`
```typescript
import type { Context, Next } from "hono";

export const requestIdMiddleware = async (c: Context, next: Next) => {
  const requestId = c.req.header("X-Request-Id") || crypto.randomUUID();
  c.set("requestId", requestId);
  c.header("X-Request-Id", requestId);
  await next();
};
```

### `src/middlewares/auth.middleware.ts`
```typescript
import type { Context, Next } from "hono";
import { AuthorizationError } from "@/lib/error";
import type { AuthUser } from "@/types/context";

export const authMiddleware = async (c: Context, next: Next) => {
  const authHeader = c.req.header("Authorization");
  if (!authHeader || !authHeader.startsWith("Bearer ")) {
    throw new AuthorizationError("Missing or invalid Authorization header");
  }

  const token = authHeader.substring(7);

  try {
    const [headerB64, payloadB64, signature] = token.split(".");
    if (!headerB64 || !payloadB64 || !signature) throw new Error("Malformed token");

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

### `src/middlewares/validate.middleware.ts`
```typescript
import { ValidationError } from "@/lib/error";
import { validator } from "hono/validator";
import { z } from "zod";

export const validateBody = <T extends z.ZodTypeAny>(schema: T) =>
  validator("json", (value) => {
    const result = schema.safeParse(value);
    if (!result.success) throw new ValidationError(result.error.issues[0].message);
    return result.data as z.infer<T>;
  });

export const validateParams = <T extends z.ZodTypeAny>(schema: T) =>
  validator("param", (value) => {
    const result = schema.safeParse(value);
    if (!result.success) throw new ValidationError(result.error.issues[0].message);
    return result.data as z.infer<T>;
  });

export const validateQuery = <T extends z.ZodTypeAny>(schema: T) =>
  validator("query", (value) => {
    const result = schema.safeParse(value);
    if (!result.success) throw new ValidationError(result.error.issues[0].message);
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
  const reqId = c.get("requestId") ? `[Req: ${c.get("requestId")}] ` : "";

  if (error instanceof AppError) {
    logger.error(`${reqId}${error.message}`);
    return Response.error(c, error.code, error.message, error.status);
  }

  if (error instanceof ZodError) {
    const message = error.issues[0]?.message || "Validation failed";
    logger.error(`${reqId}Validation Error: ${message}`);
    return Response.error(c, "VALIDATION_ERROR", message, 422);
  }

  if (error.message && error.message.includes("JSON")) {
    logger.error(`${reqId}${error.message}`);
    return Response.error(c, "BAD_REQUEST", "Invalid JSON", 400);
  }

  logger.error(`${reqId}[CRITICAL]`, error?.stack || error?.message || error);
  return Response.error(c, "SERVER_ERROR", "Internal Server Error", 500);
};
```

---

## 5. Database Layer (`src/db/`)

### `src/db/helpers/columns.ts` (Standard Schema Helpers)
```typescript
import { timestamp, uuid } from "drizzle-orm/pg-core";

export const primaryKeyUuidV7 = () => uuid("id").defaultRandom().primaryKey();

export const timestamps = {
  createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
  updatedAt: timestamp("updated_at", { withTimezone: true })
    .defaultNow()
    .$onUpdate(() => new Date())
    .notNull(),
};

export const softDelete = {
  deletedAt: timestamp("deleted_at", { withTimezone: true }),
};
```

### `src/db/index.ts` (Drizzle Client & Pool)
```typescript
import { env } from "@/config/env.config";
import { drizzle } from "drizzle-orm/node-postgres";
import { Pool } from "pg";
import * as schema from "./schemas";

export const pool = new Pool({
  connectionString: env.POSTGRESQL_URL,
  options: `-c timezone=${env.TZ}`,
});

export const db = drizzle(pool, { schema });

export type Database = typeof db;
export type Transaction = Parameters<Parameters<Database["transaction"]>[0]>[0];

export default db;
```

### `src/db/schemas/users.schema.ts` (With Partial Unique Index)
```typescript
import { pgTable, text, uniqueIndex, index } from "drizzle-orm/pg-core";
import { isNull } from "drizzle-orm";
import { primaryKeyUuidV7, timestamps, softDelete } from "../helpers/columns";

export const users = pgTable(
  "users",
  {
    id: primaryKeyUuidV7(),
    email: text("email").notNull(),
    passwordHash: text("password_hash").notNull(),
    fullName: text("full_name").notNull(),
    ...timestamps,
    ...softDelete,
  },
  (table) => [
    // Unique email only among active (non-soft-deleted) users
    uniqueIndex("users_email_active_unique").on(table.email).where(isNull(table.deletedAt)),
    // Index on deletedAt for fast query filters
    index("users_deleted_at_idx").on(table.deletedAt),
  ],
);

export type User = typeof users.$inferSelect;
export type NewUser = typeof users.$inferInsert;
```

---

## 6. Real-Time WebSockets (`src/ws/`)

### `src/ws/index.ts` (Hono + Bun WebSocket with Redis Pub/Sub)
```typescript
import type { Hono } from "hono";
import type { AppEnv } from "@/types/context";
import { redis, redisSub } from "@/lib/redis";
import { logger } from "@/lib/logger";

export function setupWebSocketRoutes(app: Hono<AppEnv>, upgradeWebSocket: any) {
  // Listen to cross-instance Redis Pub/Sub messages
  redisSub.subscribe("broadcast:notifications", (err) => {
    if (err) logger.error("Failed to subscribe to Redis channel:", err);
  });

  app.get(
    "/ws",
    upgradeWebSocket((c: any) => {
      return {
        onOpen(event: any, ws: any) {
          logger.info("WebSocket connection opened");
          ws.subscribe("global-notifications");
        },
        onMessage(event: any, ws: any) {
          const message = event.data.toString();
          // Publish message to Redis for multi-instance distribution
          redis.publish("broadcast:notifications", message);
        },
        onClose(event: any, ws: any) {
          logger.info("WebSocket connection closed");
        },
      };
    }),
  );
}
```

---

## 7. Complete Feature Reference Slice (`src/features/users/`)

### `src/features/users/schemas/index.ts`
```typescript
import { z } from "zod";
import { paginationQuerySchema } from "@/lib/pagination";

export const createUserSchema = z.object({
  email: z.string().trim().toLowerCase().email("Invalid email address"),
  password: z.string().min(8, "Password must be at least 8 characters"),
  fullName: z.string().trim().min(2, "Full name must be at least 2 characters"),
});

export const userIdParamSchema = z.object({
  id: z.string().uuid("Invalid user ID"),
});

export const listUsersQuerySchema = paginationQuerySchema.extend({
  search: z.string().trim().optional(),
});

export type CreateUserInput = z.infer<typeof createUserSchema>;
export type UserIdParam = z.infer<typeof userIdParamSchema>;
export type ListUsersQuery = z.infer<typeof listUsersQuerySchema>;
```

### `src/features/users/mappers/index.ts` (DTO Serialization)
```typescript
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

### `src/features/users/repositories/index.ts`
```typescript
import db, { type Transaction } from "@/db";
import { users, type User, type NewUser } from "@/db/schemas";
import { eq, isNull, and, sql } from "drizzle-orm";

export class UsersRepository {
  static async findById(id: string, tx?: Transaction): Promise<User | null> {
    const client = tx ?? db;
    const user = await client.query.users.findFirst({
      where: and(eq(users.id, id), isNull(users.deletedAt)),
    });
    return user ?? null;
  }

  static async findByEmail(email: string, tx?: Transaction): Promise<User | null> {
    const client = tx ?? db;
    const user = await client.query.users.findFirst({
      where: and(eq(users.email, email), isNull(users.deletedAt)),
    });
    return user ?? null;
  }

  static async create(data: NewUser, tx?: Transaction): Promise<User> {
    const client = tx ?? db;
    const [created] = await client.insert(users).values(data).returning();
    return created;
  }

  static async softDelete(id: string, tx?: Transaction): Promise<void> {
    const client = tx ?? db;
    await client
      .update(users)
      .set({ deletedAt: new Date() })
      .where(and(eq(users.id, id), isNull(users.deletedAt)));
  }

  static async list(page: number, limit: number, tx?: Transaction): Promise<{ items: User[]; total: number }> {
    const client = tx ?? db;
    const offset = (page - 1) * limit;

    const items = await client.query.users.findMany({
      where: isNull(users.deletedAt),
      limit,
      offset,
    });

    const [{ count }] = await client
      .select({ count: sql<number>`count(*)::int` })
      .from(users)
      .where(isNull(users.deletedAt));

    return { items, total: count };
  }
}
```

### `src/features/users/services/index.ts`
```typescript
import db from "@/db";
import { ConflictError, NotFoundError } from "@/lib/error";
import { cache } from "@/lib/cache";
import { emailQueue } from "@/queues";
import { UsersRepository } from "../repositories";
import type { CreateUserInput, ListUsersQuery } from "../schemas";

export class UsersService {
  static async getById(id: string) {
    return cache.getOrSet(`users:${id}`, 1800, async () => {
      const user = await UsersRepository.findById(id);
      if (!user) throw new NotFoundError(`User with id '${id}' not found`);
      return user;
    });
  }

  static async create(input: CreateUserInput) {
    const existing = await UsersRepository.findByEmail(input.email);
    if (existing) throw new ConflictError("Email already registered");

    const passwordHash = await Bun.password.hash(input.password, { algorithm: "argon2id" });

    const createdUser = await db.transaction(async (tx) => {
      return UsersRepository.create(
        {
          email: input.email,
          fullName: input.fullName,
          passwordHash,
        },
        tx,
      );
    });

    await emailQueue.add("welcomeEmail", { email: createdUser.email, name: createdUser.fullName });

    return createdUser;
  }

  static async delete(id: string) {
    const existing = await UsersRepository.findById(id);
    if (!existing) throw new NotFoundError("User not found");

    await UsersRepository.softDelete(id);
    await cache.del(`users:${id}`);
  }

  static async list(query: ListUsersQuery) {
    return UsersRepository.list(query.page, query.limit);
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
import { UserMapper } from "../mappers";
import { formatPaginatedResult } from "@/lib/pagination";
import Response from "@/lib/response";

const router = new Hono<AppEnv>();

// Public registration
router.post("/", validateBody(createUserSchema), async (c) => {
  const input = c.req.valid("json");
  const user = await UsersService.create(input);
  const response = UserMapper.toResponse(user);
  return Response.success(c, response, "User registered successfully", 201);
});

// Protected routes
router.use("*", authMiddleware);

router.get("/", validateQuery(listUsersQuerySchema), async (c) => {
  const query = c.req.valid("query");
  const { items, total } = await UsersService.list(query);
  const safeItems = UserMapper.toResponseList(items);
  const paginated = formatPaginatedResult(safeItems, total, query.page, query.limit);
  return Response.success(c, paginated);
});

router.get("/:id", validateParams(userIdParamSchema), async (c) => {
  const { id } = c.req.valid("param");
  const user = await UsersService.getById(id);
  const response = UserMapper.toResponse(user);
  return Response.success(c, response);
});

router.delete("/:id", validateParams(userIdParamSchema), async (c) => {
  const { id } = c.req.valid("param");
  await UsersService.delete(id);
  return Response.empty(c, 204);
});

export default router;
```

---

## 8. Testing Standards (`tests/`)

### `tests/setup.ts` (Test Runner Setup)
```typescript
import { beforeAll, afterAll } from "bun:test";
import { pool } from "@/db";
import { redis, redisSub } from "@/lib/redis";

beforeAll(async () => {
  // Setup test environment
});

afterAll(async () => {
  await pool.end();
  await redisSub.quit();
  await redis.quit();
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
