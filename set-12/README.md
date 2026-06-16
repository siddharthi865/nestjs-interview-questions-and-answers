# Set 12

| S.No. | Question                                                                                                                                               |
| ----- | ------------------------------------------------------------------------------------------------------------------------------------------------------ |
| 1.    | [How do you implement **global interceptors conditionally**?](#question-1-how-do-you-implement-global-interceptors-conditionally)                      |
| 2.    | [How do you implement **logging with interceptors per module**?](#question-2-how-do-you-implement-logging-with-interceptors-per-module)                |
| 3.    | [How do you measure **request execution time using interceptors**?](#question-3-how-do-you-measure-request-execution-time-using-interceptors)          |
| 4.    | [How do you implement **caching using interceptors**?](#question-4-how-do-you-implement-caching-using-interceptors)                                    |
| 5.    | [How do you combine **multiple guards on a single route**?](#question-5-how-do-you-combine-multiple-guards-on-a-single-route)                          |
| 6.    | [How do you implement **hierarchical roles with guards**?](#question-6-how-do-you-implement-hierarchical-roles-with-guards)                            |
| 7.    | [How do you implement **rate-limiting using guards**?](#question-7-how-do-you-implement-rate-limiting-using-guards)                                    |
| 8.    | [How do you implement **custom pipes for complex validation**?](#question-8-how-do-you-implement-custom-pipes-for-complex-validation)                  |
| 9.    | [How do you implement **transformations with async pipes**?](#question-9-how-do-you-implement-transformations-with-async-pipes)                        |
| 10.   | [How do you implement **conditional middleware execution**?](#question-10-how-do-you-implement-conditional-middleware-execution)                       |
| 11.   | [How do you implement **middleware for multipart/form-data**?](#question-11-how-do-you-implement-middleware-for-multipartform-data)                    |
| 12.   | [How do you implement **exception handling for WebSockets**?](#question-12-how-do-you-implement-exception-handling-for-websockets)                     |
| 13.   | [How do you implement **global exception filters with DI**?](#question-13-how-do-you-implement-global-exception-filters-with-di)                       |
| 14.   | [How do you implement **guards for GraphQL resolvers**?](#question-14-how-do-you-implement-guards-for-graphql-resolvers)                               |
| 15.   | [How do you implement **interceptors for GraphQL responses**?](#question-15-how-do-you-implement-interceptors-for-graphql-responses)                   |
| 16.   | [How do you implement **custom decorators with parameters**?](#question-16-how-do-you-implement-custom-decorators-with-parameters)                     |
| 17.   | [How do you implement **role-based data filtering with interceptors**?](#question-17-how-do-you-implement-role-based-data-filtering-with-interceptors) |
| 18.   | [How do you implement **schema validation in pipes**?](#question-18-how-do-you-implement-schema-validation-in-pipes)                                   |
| 19.   | [How do you implement **request throttling using interceptors**?](#question-19-how-do-you-implement-request-throttling-using-interceptors)             |
| 20.   | [How do you implement **dynamic exception messages**?](#question-20-how-do-you-implement-dynamic-exception-messages)                                   |

## Question 1. How do you implement **global interceptors conditionally**?

## Short answer

You implement conditional global interceptors in NestJS by using `APP_INTERCEPTOR` with a provider factory that decides whether to apply logic based on request context, configuration, or environment—often combined with a “no-op” interceptor or internal conditional branching inside the interceptor itself.

---

## Explanation

In NestJS, interceptors are part of the request lifecycle pipeline and are typically applied globally using the `APP_INTERCEPTOR` token.

However, **global interceptors are always registered at app bootstrap time**, so “conditional global application” is not about dynamically attaching/removing them per request at registration time. Instead, you have three production-grade strategies:

---

### 1. Conditional logic inside a global interceptor (most common, recommended)

You always register the interceptor globally, but decide per request whether to execute logic.

Typical conditions:

- route metadata (`Reflector`)
- headers (feature flags, internal requests)
- environment-based toggles
- user roles / auth context

**Trade-off:** simple, but interceptor still runs for every request (light overhead).

---

### 2. Dynamic provider factory (conditional registration at bootstrap)

Use `APP_INTERCEPTOR` with `useFactory` to decide whether to register it at all.

Useful when:

- disabling logging in test environments
- enabling tracing only in production

**Trade-off:** cannot change at runtime without restart.

---

### 3. Scope-based interception using `ContextIdFactory` / request-scoped providers (advanced)

You can combine request-scoped providers or context-based logic for fine-grained control, but this is usually heavier and impacts performance.

---

### Architectural perspective

In large-scale NestJS systems:

- **Interceptors are cross-cutting concerns** (logging, caching, serialization, tracing)
- Conditional logic should be:
  - centralized (via metadata or config service)
  - predictable (avoid hidden side effects)

- Prefer **decorator-driven enable/disable** over scattered `if` conditions

---

### Scalability considerations

- Global interceptors execute for every request → even empty logic adds latency at high RPS
- Prefer early exits (`if (!shouldApply) return next.handle()`)
- Avoid heavy DI or async calls inside interceptor condition checks
- Use caching for config flags (e.g., feature flags)

---

### Security implications

- Be careful not to rely solely on headers for disabling security-related interceptors
- Ensure bypass conditions cannot be spoofed (e.g., internal headers should be validated by gateway/API proxy)

---

## Example

### Conditional global interceptor using `APP_INTERCEPTOR` + metadata + config

```ts
// logging.interceptor.ts
import {
  CallHandler,
  ExecutionContext,
  Injectable,
  NestInterceptor,
} from "@nestjs/common";
import { Reflector } from "@nestjs/core";
import { Observable, tap } from "rxjs";

export const SKIP_LOGGING = "skip_logging";

@Injectable()
export class LoggingInterceptor implements NestInterceptor {
  constructor(private readonly reflector: Reflector) {}

  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    const skip = this.reflector.getAllAndOverride<boolean>(SKIP_LOGGING, [
      context.getHandler(),
      context.getClass(),
    ]);

    if (skip) {
      return next.handle();
    }

    const start = Date.now();

    return next.handle().pipe(
      tap(() => {
        const duration = Date.now() - start;
        const req = context.switchToHttp().getRequest();
        console.log(`[${req.method}] ${req.url} - ${duration}ms`);
      }),
    );
  }
}
```

```ts
// logging.decorator.ts
import { SetMetadata } from "@nestjs/common";
import { SKIP_LOGGING } from "./logging.interceptor";

export const SkipLogging = () => SetMetadata(SKIP_LOGGING, true);
```

```ts
// app.module.ts
import { Module } from "@nestjs/common";
import { APP_INTERCEPTOR } from "@nestjs/core";
import { LoggingInterceptor } from "./logging.interceptor";

@Module({
  providers: [
    {
      provide: APP_INTERCEPTOR,
      useClass: LoggingInterceptor,
    },
  ],
})
export class AppModule {}
```

---

## Pitfalls

- ❌ Overusing global interceptors for everything → hard to debug request lifecycle
- ❌ Heavy synchronous logic in interceptors → increases latency across all endpoints
- ❌ Relying on headers for conditional bypass → security risk if not validated upstream
- ❌ Using request-scoped interceptors excessively → performance degradation under load

## Question 2. How do you implement **logging with interceptors per module**?

## Question 3. How do you measure **request execution time using interceptors**?

## Question 4. How do you implement **caching using interceptors**?

## Question 5. How do you combine **multiple guards on a single route**?

## Question 6. How do you implement **hierarchical roles with guards**?

## Question 7. How do you implement **rate-limiting using guards**?

## Question 8. How do you implement **custom pipes for complex validation**?

## Question 9. How do you implement **transformations with async pipes**?

## Question 10. How do you implement **conditional middleware execution**?

## Question 11. How do you implement **middleware for multipart/form-data**?

## Question 12. How do you implement **exception handling for WebSockets**?

## Question 13. How do you implement **global exception filters with DI**?

## Question 14. How do you implement **guards for GraphQL resolvers**?

## Question 15. How do you implement **interceptors for GraphQL responses**?

## Question 16. How do you implement **custom decorators with parameters**?

## Question 17. How do you implement **role-based data filtering with interceptors**?

## Question 18. How do you implement **schema validation in pipes**?

## Question 19. How do you implement **request throttling using interceptors**?

## Question 20. How do you implement **dynamic exception messages**?
