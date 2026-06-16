# Set 1

| S.No. | Question                                                                                                                                                   |
| ----- | ---------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1.    | [What is NestJS and why would you use it over plain Node.js or Express?](#question-1-what-is-nestjs-and-why-would-you-use-it-over-plain-nodejs-or-express) |
| 2.    | [What are the main features of NestJS?](#question-2-what-are-the-main-features-of-nestjs)                                                                  |
| 3.    | [Explain the concept of **modules** in NestJS](#question-3-explain-the-concept-of-modules-in-nestjs)                                                       |
| 4.    | [What is a **controller** in NestJS?](#question-4-what-is-a-controller-in-nestjs)                                                                          |
| 5.    | [What is a **service** in NestJS?](#question-5-what-is-a-service-in-nestjs)                                                                                |
| 6.    | [How do you create a new NestJS project?](#question-6-how-do-you-create-a-new-nestjs-project)                                                              |
| 7.    | [What is the role of **Decorators** in NestJS?](#question-7-what-is-the-role-of-decorators-in-nestjs)                                                      |
| 8.    | [Explain `@Module()` decorator and its properties](#question-8-explain-module-decorator-and-its-properties)                                                |
| 9.    | [Explain `@Controller()` decorator](#question-9-explain-controller-decorator)                                                                              |
| 10.   | [Explain `@Get()`, `@Post()`, `@Put()`, and `@Delete()` decorators](#question-10-explain-get-post-put-and-delete-decorators)                               |
| 11.   | [What is the purpose of `@Injectable()` decorator?](#question-11-what-is-the-purpose-of-injectable-decorator)                                              |
| 12.   | [How does dependency injection work in NestJS?](#question-12-how-does-dependency-injection-work-in-nestjs)                                                 |
| 13.   | [How do you use `@Param()` in controllers?](#question-13-how-do-you-use-param-in-controllers)                                                              |
| 14.   | [How do you use `@Body()` in controllers?](#question-14-how-do-you-use-body-in-controllers)                                                                |
| 15.   | [How do you use `@Query()` in controllers?](#question-15-how-do-you-use-query-in-controllers)                                                              |
| 16.   | [How do you use `@Req()` and `@Res()`?](#question-16-how-do-you-use-req-and-res)                                                                           |
| 17.   | [Explain the default **lifecycle of a NestJS request**](#question-17-explain-the-default-lifecycle-of-a-nestjs-request)                                    |
| 18.   | [How do you create and use a **DTO (Data Transfer Object)**?](#question-18-how-do-you-create-and-use-a-dto-data-transfer-object)                           |
| 19.   | [How do you validate incoming data in NestJS?](#question-19-how-do-you-validate-incoming-data-in-nestjs)                                                   |
| 20.   | [What is **class-validator** and how is it used with DTOs?](#question-20-what-is-class-validator-and-how-is-it-used-with-dtos)                             |

## Question 1. What is NestJS and why would you use it over plain Node.js or Express?

## Short answer

NestJS is a progressive Node.js framework for building scalable, maintainable server-side applications using TypeScript, heavily inspired by Angular’s architecture. You use it over plain Node.js or Express because it provides strong architectural structure, dependency injection, modularity, and enterprise-grade patterns out of the box.

---

## Explanation

NestJS (NestJS) is built on top of Express (or optionally Fastify) but adds a **structured application architecture layer**. While Express gives you minimal routing and middleware primitives, NestJS enforces a **clear architectural model**:

### 1. Architecture (key differentiator)

NestJS introduces:

- **Modules** → feature-based organization
- **Controllers** → request handling layer
- **Providers/Services** → business logic
- **Dependency Injection (DI)** → testable, decoupled components

This is similar to enterprise frameworks like Spring (Java) or Angular (frontend).

👉 In Express, you typically end up with:

- ad-hoc folder structure
- manual dependency wiring
- growing maintenance complexity at scale

---

### 2. Scalability & Maintainability

NestJS is designed for:

- large teams
- microservices architectures
- domain-driven design (DDD)
- monorepos (Nx integration is common)

It enforces boundaries between features, which prevents “spaghetti Express apps.”

---

### 3. Built-in enterprise features

NestJS provides out-of-the-box:

- Dependency Injection container
- Guards (auth/authorization)
- Interceptors (logging, transformation, caching)
- Pipes (validation/transformation)
- Exception filters (centralized error handling)
- Microservices support (Kafka, Redis, NATS, gRPC)
- Testing utilities (Jest integration built-in)

In Express, all of this must be assembled manually or via third-party middleware.

---

### 4. TypeScript-first design

NestJS is designed around TypeScript:

- decorators (`@Controller`, `@Injectable`)
- strong typing across layers
- metadata-driven reflection system

Express supports TypeScript but does not _enforce or leverage it structurally_.

---

### 5. Trade-offs vs Express / raw Node.js

| Aspect                      | NestJS          | Express / Node.js     |
| --------------------------- | --------------- | --------------------- |
| Learning curve              | Higher          | Lower                 |
| Flexibility                 | Opinionated     | Fully flexible        |
| Architecture enforcement    | Strong          | None                  |
| Boilerplate                 | More            | Minimal               |
| Scalability for large teams | Excellent       | Risk of inconsistency |
| Startup speed               | Slightly slower | Faster                |

---

### 6. When NOT to use NestJS

- Very small services / scripts
- Prototypes or quick APIs
- Ultra-low latency micro-optimizations (bare Node may be better)
- Teams not comfortable with DI/OO patterns

---

## Example

### NestJS structured architecture example

```ts
// user.module.ts
import { Module } from "@nestjs/common";
import { UserController } from "./user.controller";
import { UserService } from "./user.service";

@Module({
  controllers: [UserController],
  providers: [UserService],
})
export class UserModule {}
```

```ts
// user.service.ts
import { Injectable } from "@nestjs/common";

@Injectable()
export class UserService {
  private users = [{ id: 1, name: "Alice" }];

  findAll() {
    return this.users;
  }
}
```

```ts
// user.controller.ts
import { Controller, Get } from "@nestjs/common";
import { UserService } from "./user.service";

@Controller("users")
export class UserController {
  constructor(private readonly userService: UserService) {}

  @Get()
  getUsers() {
    return this.userService.findAll();
  }
}
```

---

## Pitfalls

- Overengineering small services with unnecessary abstraction layers
- Misusing DI scopes leading to memory leaks (e.g., request-scoped providers everywhere)
- Hidden complexity from decorators and metadata (harder debugging vs Express)
- Performance overhead vs bare Node/Express in ultra-light APIs
- Incorrect module boundaries leading to circular dependency issues

## Question 2. What are the main features of NestJS?

## Question 3. Explain the concept of **modules** in NestJS

## Question 4. What is a **controller** in NestJS?

## Question 5. What is a **service** in NestJS?

## Question 6. How do you create a new NestJS project?

## Question 7. What is the role of **Decorators** in NestJS?

## Question 8. Explain `@Module()` decorator and its properties

## Question 9. Explain `@Controller()` decorator

## Question 10. Explain `@Get()`, `@Post()`, `@Put()`, and `@Delete()` decorators

## Question 11. What is the purpose of `@Injectable()` decorator?

## Question 12. How does dependency injection work in NestJS?

## Question 13. How do you use `@Param()` in controllers?

## Question 14. How do you use `@Body()` in controllers?

## Question 15. How do you use `@Query()` in controllers?

## Question 16. How do you use `@Req()` and `@Res()`?

## Question 17. Explain the default **lifecycle of a NestJS request**

## Question 18. How do you create and use a **DTO (Data Transfer Object)**?

## Question 19. How do you validate incoming data in NestJS?

## Question 20. What is **class-validator** and how is it used with DTOs?
