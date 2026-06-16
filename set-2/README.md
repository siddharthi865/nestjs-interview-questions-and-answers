# Set 2

| S.No. | Question                                                                                                                                                         |
| ----- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1.    | [What is **middleware** in NestJS?](#question-1-what-is-middleware-in-nestjs)                                                                                    |
| 2.    | [How do you implement middleware in NestJS?](#question-2-how-do-you-implement-middleware-in-nestjs)                                                              |
| 3.    | [How do you apply middleware globally vs per route?](#question-3-how-do-you-apply-middleware-globally-vs-per-route)                                              |
| 4.    | [Explain **pipes** in NestJS](#question-4-explain-pipes-in-nestjs)                                                                                               |
| 5.    | [How do you use built-in pipes like `ValidationPipe` or `ParseIntPipe`?](#question-5-how-do-you-use-built-in-pipes-like-validationpipe-or-parseintpipe)          |
| 6.    | [How do you create a **custom pipe**?](#question-6-how-do-you-create-a-custom-pipe)                                                                              |
| 7.    | [Explain **guards** in NestJS](#question-7-explain-guards-in-nestjs)                                                                                             |
| 8.    | [How do guards differ from middleware?](#question-8-how-do-guards-differ-from-middleware)                                                                        |
| 9.    | [How do you implement **role-based access control** using guards?](#question-9-how-do-you-implement-role-based-access-control-using-guards)                      |
| 10.   | [What are **interceptors** in NestJS?](#question-10-what-are-interceptors-in-nestjs)                                                                             |
| 11.   | [How do interceptors differ from middleware?](#question-11-how-do-interceptors-differ-from-middleware)                                                           |
| 12.   | [How do you implement logging with interceptors?](#question-12-how-do-you-implement-logging-with-interceptors)                                                   |
| 13.   | [Explain **exception filters** in NestJS](#question-13-explain-exception-filters-in-nestjs)                                                                      |
| 14.   | [How do you handle custom exceptions?](#question-14-how-do-you-handle-custom-exceptions)                                                                         |
| 15.   | [How do you implement a **global exception filter**?](#question-15-how-do-you-implement-a-global-exception-filter)                                               |
| 16.   | [What is `ExecutionContext` and how is it used in guards and interceptors?](#question-16-what-is-executioncontext-and-how-is-it-used-in-guards-and-interceptors) |
| 17.   | [How do you handle CORS in NestJS?](#question-17-how-do-you-handle-cors-in-nestjs)                                                                               |
| 18.   | [How can you implement request throttling or rate limiting?](#question-18-how-can-you-implement-request-throttling-or-rate-limiting)                             |
| 19.   | [Explain **NestJS lifecycle hooks** (`OnModuleInit`, `OnApplicationBootstrap`)](#question-19-explain-nestjs-lifecycle-hooks-onmoduleinit-onapplicationbootstrap) |
| 20.   | [How do you implement **redirects** and **status codes** in controllers?](#question-20-how-do-you-implement-redirects-and-status-codes-in-controllers)           |

## Question 1. What is **middleware** in NestJS?

Understood — I’ll follow the senior-level NestJS interview format.

---

# Short answer

Middleware in NestJS is a function executed **before the route handler**, used to modify the request/response objects or apply logic like logging, authentication checks, or request transformation.

---

# Explanation

## What middleware is in NestJS architecture

In NestJS, middleware sits **between the raw incoming HTTP request and the route handler** (controller). It is conceptually similar to Express middleware because NestJS is built on top of Express (or optionally Fastify).

It has access to:

- `Request`
- `Response`
- `next()` function

### Execution flow (high level)

```
Request → Middleware → Guards → Interceptors → Pipes → Controller → Service
```

Middleware runs **first in the request lifecycle**, before Nest’s DI-based request pipeline kicks in.

---

## Key characteristics

### 1. Express-compatible layer

NestJS middleware is fundamentally Express-style:

- It does NOT have access to dependency injection by default (unless using `NestMiddleware` class form)
- It is stateless or request-scoped

### 2. Use cases

Middleware is best for **cross-cutting, low-level concerns**:

- Request logging (e.g., access logs)
- Request ID generation (correlation IDs)
- Body parsing / transformation (rare in NestJS because body-parser is already used)
- Simple authentication checks (not authorization — that’s Guards)
- Rate limiting (sometimes better in Guards or external proxy)
- Header enrichment (e.g., tenant ID injection)

---

## Middleware vs Guards vs Interceptors (important interview distinction)

| Concern                   | Middleware               | Guard                    | Interceptor                      |
| ------------------------- | ------------------------ | ------------------------ | -------------------------------- |
| Runs before route handler | ✅                       | ✅ (after middleware)    | wraps handler                    |
| Has DI support            | limited                  | full                     | full                             |
| Use case                  | raw request manipulation | authorization            | response transformation, caching |
| Knows execution context   | no                       | yes (`ExecutionContext`) | yes                              |

👉 Senior insight:

- Middleware = HTTP-layer plumbing
- Guards = security / authorization
- Interceptors = AOP (aspect-oriented programming)

---

## Scope in NestJS

Middleware can be applied:

### 1. Globally

Applies to all routes.

### 2. Module-level

Applies to routes inside a module.

### 3. Route-level

Applied selectively.

---

## Two forms of middleware

### 1. Functional middleware (simple)

```ts
import { Request, Response, NextFunction } from "express";

export function logger(req: Request, res: Response, next: NextFunction) {
  console.log(`[${req.method}] ${req.url}`);
  next();
}
```

### 2. Class-based middleware (recommended for DI)

```ts
import { Injectable, NestMiddleware } from "@nestjs/common";
import { Request, Response, NextFunction } from "express";

@Injectable()
export class LoggerMiddleware implements NestMiddleware {
  use(req: Request, res: Response, next: NextFunction) {
    console.log(`[Logger] ${req.method} ${req.originalUrl}`);
    next();
  }
}
```

---

## Applying middleware in a module

```ts
import { Module, MiddlewareConsumer, NestModule } from "@nestjs/common";
import { LoggerMiddleware } from "./logger.middleware";
import { UsersController } from "./users.controller";

@Module({
  controllers: [UsersController],
})
export class UsersModule implements NestModule {
  configure(consumer: MiddlewareConsumer) {
    consumer.apply(LoggerMiddleware).forRoutes(UsersController);
  }
}
```

---

## Example (real-world pattern: request ID middleware)

```ts
import { Injectable, NestMiddleware } from "@nestjs/common";
import { Request, Response, NextFunction } from "express";
import { randomUUID } from "crypto";

@Injectable()
export class RequestIdMiddleware implements NestMiddleware {
  use(req: Request, res: Response, next: NextFunction) {
    const requestId = req.headers["x-request-id"] ?? randomUUID();
    req.headers["x-request-id"] = requestId as string;

    res.setHeader("x-request-id", requestId);
    next();
  }
}
```

---

# Pitfalls

- Middleware **does not have full DI lifecycle control** like providers (can limit testability)
- Easy to misuse for **authorization logic** (should use Guards instead)
- Cannot access Nest execution context (no route metadata, no decorators)
- Overuse globally can hurt performance in high-throughput apps
- Order matters — middleware ordering bugs are common in large modules

## Question 2. How do you implement middleware in NestJS?

## Question 3. How do you apply middleware globally vs per route?

## Question 4. Explain **pipes** in NestJS

## Question 5. How do you use built-in pipes like `ValidationPipe` or `ParseIntPipe`?

## Question 6. How do you create a **custom pipe**?

## Question 7. Explain **guards** in NestJS

## Question 8. How do guards differ from middleware?

## Question 9. How do you implement **role-based access control** using guards?

## Question 10. What are **interceptors** in NestJS?

## Question 11. How do interceptors differ from middleware?

## Question 12. How do you implement logging with interceptors?

## Question 13. Explain **exception filters** in NestJS

## Question 14. How do you handle custom exceptions?

## Question 15. How do you implement a **global exception filter**?

## Question 16. What is `ExecutionContext` and how is it used in guards and interceptors?

## Question 17. How do you handle CORS in NestJS?

## Question 18. How can you implement request throttling or rate limiting?

## Question 19. Explain **NestJS lifecycle hooks** (`OnModuleInit`, `OnApplicationBootstrap`)

## Question 20. How do you implement **redirects** and **status codes** in controllers?
