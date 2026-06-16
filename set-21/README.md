# Set 21

| S.No. | Question                                                                                                                                                         |
| ----- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1.    | [How do you implement **conditional routing** based on request headers?](#question-1-how-do-you-implement-conditional-routing-based-on-request-headers)          |
| 2.    | [How do you implement **dynamic route handlers** at runtime?](#question-2-how-do-you-implement-dynamic-route-handlers-at-runtime)                                |
| 3.    | [How do you handle **optional route parameters**?](#question-3-how-do-you-handle-optional-route-parameters)                                                      |
| 4.    | [How do you implement **query parameter transformations**?](#question-4-how-do-you-implement-query-parameter-transformations)                                    |
| 5.    | [How do you implement **custom HTTP response formatting**?](#question-5-how-do-you-implement-custom-http-response-formatting)                                    |
| 6.    | [How do you implement **controller-level interceptors**?](#question-6-how-do-you-implement-controller-level-interceptors)                                        |
| 7.    | [How do you implement **controller-level exception filters**?](#question-7-how-do-you-implement-controller-level-exception-filters)                              |
| 8.    | [How do you handle **content negotiation** in controllers?](#question-8-how-do-you-handle-content-negotiation-in-controllers)                                    |
| 9.    | [How do you implement **redirects conditionally**?](#question-9-how-do-you-implement-redirects-conditionally)                                                    |
| 10.   | [How do you implement **versioned controllers dynamically**?](#question-10-how-do-you-implement-versioned-controllers-dynamically)                               |
| 11.   | [How do you handle **wildcard routes** in NestJS?](#question-11-how-do-you-handle-wildcard-routes-in-nestjs)                                                     |
| 12.   | [How do you implement **controller inheritance**?](#question-12-how-do-you-implement-controller-inheritance)                                                     |
| 13.   | [How do you implement **method decorators that apply multiple HTTP verbs**?](#question-13-how-do-you-implement-method-decorators-that-apply-multiple-http-verbs) |
| 14.   | [How do you implement **controller-specific middleware**?](#question-14-how-do-you-implement-controller-specific-middleware)                                     |
| 15.   | [How do you implement **controller-level caching**?](#question-15-how-do-you-implement-controller-level-caching)                                                 |
| 16.   | [How do you implement **custom response headers** per controller method?](#question-16-how-do-you-implement-custom-response-headers-per-controller-method)       |
| 17.   | [How do you implement **controller-level guards**?](#question-17-how-do-you-implement-controller-level-guards)                                                   |
| 18.   | [How do you handle **file streaming responses**?](#question-18-how-do-you-handle-file-streaming-responses)                                                       |
| 19.   | [How do you implement **controller-level WebSocket gateways**?](#question-19-how-do-you-implement-controller-level-websocket-gateways)                           |
| 20.   | [How do you implement **custom content-type handling**?](#question-20-how-do-you-implement-custom-content-type-handling)                                         |

## Question 1. How do you implement **conditional routing** based on request headers?

## Short answer

In NestJS, conditional routing based on request headers is typically implemented using **Guards, Middleware, or Interceptors** rather than defining multiple routes. A **Guard** is the most idiomatic approach because it integrates with the request lifecycle and can allow/deny access or redirect logic based on headers.

---

## Explanation

### 1. Design options (senior-level view)

#### A. Guards (recommended for routing decisions)

- Best for **access control or request shaping**
- Executes **before route handler execution**
- Can dynamically allow/deny or redirect logic based on headers

Use cases:

- A/B testing routes
- Versioned APIs via headers (`x-api-version`)
- Feature flag-based routing
- Multi-tenant routing

#### B. Middleware (best for pre-processing)

- Runs **before Nest routing**
- Good for:
  - attaching metadata to request
  - rewriting request properties

- Less powerful than guards for decision-making

#### C. Interceptors (response-level logic)

- Not ideal for routing decisions
- Useful if behavior changes response format instead of routing

---

### 2. Architecture considerations

#### Scalability

- Header-based routing should remain **stateless**
- Avoid database calls inside guards unless cached
- Prefer feature flags via external config service (Redis/LaunchDarkly/etc.)

#### Security

- Always validate header values (avoid injection-like misuse)
- Never trust headers without authentication context if sensitive routing is involved

#### Observability

- Log routing decisions via interceptor or guard metadata
- Useful for debugging API version drift or A/B routing issues

#### Deployment

- Works seamlessly in containerized or serverless environments
- Can be combined with API Gateway routing for hybrid strategy

---

## Example

### Header-based conditional routing using Guard

```ts
import {
  CanActivate,
  ExecutionContext,
  Injectable,
  NotFoundException,
} from "@nestjs/common";
import { Request } from "express";

@Injectable()
export class ApiVersionGuard implements CanActivate {
  canActivate(context: ExecutionContext): boolean {
    const req = context.switchToHttp().getRequest<Request>();

    const version = req.headers["x-api-version"];

    if (version === "v1") {
      req.url = "/v1" + req.url;
      return true;
    }

    if (version === "v2") {
      req.url = "/v2" + req.url;
      return true;
    }

    throw new NotFoundException("Unsupported API version");
  }
}
```

### Controller setup

```ts
import { Controller, Get, UseGuards } from "@nestjs/common";
import { ApiVersionGuard } from "./api-version.guard";

@UseGuards(ApiVersionGuard)
@Controller()
export class AppController {
  @Get("v1/hello")
  getV1() {
    return { version: "v1", message: "Hello from v1" };
  }

  @Get("v2/hello")
  getV2() {
    return { version: "v2", message: "Hello from v2" };
  }
}
```

### Module wiring

```ts
import { Module } from "@nestjs/common";
import { AppController } from "./app.controller";
import { ApiVersionGuard } from "./api-version.guard";

@Module({
  controllers: [AppController],
  providers: [ApiVersionGuard],
})
export class AppModule {}
```

---

## Pitfalls

- ❌ Mutating `req.url` can break logging, metrics, or proxy layers if not carefully handled
- ❌ Overusing guards for complex routing logic can lead to hard-to-debug flows
- ❌ Header-based routing without validation opens inconsistencies and spoofing risks
- ❌ Middleware-based routing may execute too early, bypassing Nest route resolution assumptions
- ❌ Coupling API versioning logic inside application layer instead of gateway layer can reduce maintainability

## Question 2. How do you implement **dynamic route handlers** at runtime?

## Question 3. How do you handle **optional route parameters**?

## Question 4. How do you implement **query parameter transformations**?

## Question 5. How do you implement **custom HTTP response formatting**?

## Question 6. How do you implement **controller-level interceptors**?

## Question 7. How do you implement **controller-level exception filters**?

## Question 8. How do you handle **content negotiation** in controllers?

## Question 9. How do you implement **redirects conditionally**?

## Question 10. How do you implement **versioned controllers dynamically**?

## Question 11. How do you handle **wildcard routes** in NestJS?

## Question 12. How do you implement **controller inheritance**?

## Question 13. How do you implement **method decorators that apply multiple HTTP verbs**?

## Question 14. How do you implement **controller-specific middleware**?

## Question 15. How do you implement **controller-level caching**?

## Question 16. How do you implement **custom response headers** per controller method?

## Question 17. How do you implement **controller-level guards**?

## Question 18. How do you handle **file streaming responses**?

## Question 19. How do you implement **controller-level WebSocket gateways**?

## Question 20. How do you implement **custom content-type handling**?
