# Set 7

| S.No. | Question                                                                                                                                   |
| ----- | ------------------------------------------------------------------------------------------------------------------------------------------ |
| 1.    | [How do you create route prefixes in NestJS?](#question-1-how-do-you-create-route-prefixes-in-nestjs)                                      |
| 2.    | [How do you implement **versioned routes**?](#question-2-how-do-you-implement-versioned-routes)                                            |
| 3.    | [How do you implement **route parameters**?](#question-3-how-do-you-implement-route-parameters)                                            |
| 4.    | [How do you use `@Header()` decorator in NestJS?](#question-4-how-do-you-use-header-decorator-in-nestjs)                                   |
| 5.    | [How do you send custom **HTTP status codes** from a controller?](#question-5-how-do-you-send-custom-http-status-codes-from-a-controller)  |
| 6.    | [How do you implement **query string parsing**?](#question-6-how-do-you-implement-query-string-parsing)                                    |
| 7.    | [How do you handle **multipart file uploads** in NestJS?](#question-7-how-do-you-handle-multipart-file-uploads-in-nestjs)                  |
| 8.    | [How do you implement **global middleware** in NestJS?](#question-8-how-do-you-implement-global-middleware-in-nestjs)                      |
| 9.    | [How do you execute middleware conditionally?](#question-9-how-do-you-execute-middleware-conditionally)                                    |
| 10.   | [How do you implement **request logging middleware**?](#question-10-how-do-you-implement-request-logging-middleware)                       |
| 11.   | [What is the difference between **middleware and interceptors**?](#question-11-what-is-the-difference-between-middleware-and-interceptors) |
| 12.   | [How do you implement **compression middleware**?](#question-12-how-do-you-implement-compression-middleware)                               |
| 13.   | [How do you implement **CORS with different options per route**?](#question-13-how-do-you-implement-cors-with-different-options-per-route) |
| 14.   | [How do you validate **query parameters** using pipes?](#question-14-how-do-you-validate-query-parameters-using-pipes)                     |
| 15.   | [How do you transform **request data** using pipes?](#question-15-how-do-you-transform-request-data-using-pipes)                           |
| 16.   | [How do you implement **custom exceptions** for HTTP errors?](#question-16-how-do-you-implement-custom-exceptions-for-http-errors)         |
| 17.   | [How do you handle **not found (404) errors** in NestJS?](#question-17-how-do-you-handle-not-found-404-errors-in-nestjs)                   |
| 18.   | [How do you implement **redirects** in NestJS controllers?](#question-18-how-do-you-implement-redirects-in-nestjs-controllers)             |
| 19.   | [How do you implement **route guards** for authentication?](#question-19-how-do-you-implement-route-guards-for-authentication)             |
| 20.   | [How do you handle **global exception handling**?](#question-20-how-do-you-handle-global-exception-handling)                               |

## Question 1. How do you create route prefixes in NestJS?

# Short answer

In NestJS, route prefixes can be created at multiple levels:

1. **Global prefix** – applied to all routes using `app.setGlobalPrefix()`.
2. **Controller prefix** – defined in `@Controller('prefix')`.
3. **Versioned/API grouping prefixes** – often combined with global prefixes (e.g., `/api/v1/users`).

Example:

```ts
app.setGlobalPrefix("api");
```

and

```ts
@Controller('users')
```

creates routes like:

```
/api/users
```

---

# Explanation

Route prefixes help organize APIs, support versioning, improve maintainability, and simplify large-scale applications.

## 1. Global Route Prefix

The most common production pattern is configuring a global prefix in `main.ts`.

```ts
app.setGlobalPrefix("api");
```

All controllers automatically inherit the prefix:

```text
/api/users
/api/orders
/api/products
```

### Why use it?

- Clean API namespace separation.
- Easier reverse proxy and gateway configuration.
- Simplifies API versioning (`api/v1`).
- Prevents collisions with health checks, static assets, or frontend routes.

Example:

```ts
app.setGlobalPrefix("api/v1");
```

Produces:

```text
/api/v1/users
/api/v1/orders
```

---

## 2. Controller-Level Prefix

NestJS controllers define route segments using the `@Controller()` decorator.

```ts
@Controller("users")
export class UsersController {}
```

Every handler inside the controller is prefixed with `users`.

```ts
@Get()
findAll()
```

becomes:

```text
GET /users
```

and

```ts
@Get(':id')
findOne()
```

becomes:

```text
GET /users/:id
```

---

## 3. Combining Global and Controller Prefixes

NestJS automatically concatenates prefixes.

```ts
app.setGlobalPrefix('api/v1');

@Controller('users')
```

Results:

```text
GET /api/v1/users
GET /api/v1/users/:id
```

This is the recommended enterprise pattern.

---

## 4. Prefixes with API Versioning

NestJS provides built-in versioning support.

```ts
app.enableVersioning({
  type: VersioningType.URI,
});
```

```ts
@Controller({
  path: "users",
  version: "1",
})
export class UsersV1Controller {}
```

Produces:

```text
/v1/users
```

Combined with a global prefix:

```text
/api/v1/users
```

Useful when maintaining multiple API versions simultaneously.

---

## Example

```ts
import { Controller, Get, Module } from "@nestjs/common";
import { NestFactory } from "@nestjs/core";

@Controller("users")
class UsersController {
  @Get()
  findAll(): string {
    return "All users";
  }
}

@Module({
  controllers: [UsersController],
})
class AppModule {}

async function bootstrap(): Promise<void> {
  const app = await NestFactory.create(AppModule);

  app.setGlobalPrefix("api/v1");

  await app.listen(3000);
}

bootstrap();
```

Request:

```http
GET http://localhost:3000/api/v1/users
```

Response:

```text
All users
```

---

# Pitfalls

- **Changing global prefixes later** can break existing clients unless versioning is used.
- **Hardcoding URLs in frontend applications** makes migrations difficult; use API configuration variables.
- **Multiple nested prefixes** (`api/v1/admin/users`) can become difficult to maintain if route structure isn't documented.
- When using **Swagger/OpenAPI**, ensure the global prefix is reflected in Swagger configuration, otherwise generated URLs may be incorrect.

## Question 2. How do you implement **versioned routes**?

## Question 3. How do you implement **route parameters**?

## Question 4. How do you use `@Header()` decorator in NestJS?

## Question 5. How do you send custom **HTTP status codes** from a controller?

## Question 6. How do you implement **query string parsing**?

## Question 7. How do you handle **multipart file uploads** in NestJS?

## Question 8. How do you implement **global middleware** in NestJS?

## Question 9. How do you execute middleware conditionally?

## Question 10. How do you implement **request logging middleware**?

## Question 11. What is the difference between **middleware and interceptors**?

## Question 12. How do you implement **compression middleware**?

## Question 13. How do you implement **CORS with different options per route**?

## Question 14. How do you validate **query parameters** using pipes?

## Question 15. How do you transform **request data** using pipes?

## Question 16. How do you implement **custom exceptions** for HTTP errors?

## Question 17. How do you handle **not found (404) errors** in NestJS?

## Question 18. How do you implement **redirects** in NestJS controllers?

## Question 19. How do you implement **route guards** for authentication?

## Question 20. How do you handle **global exception handling**?
