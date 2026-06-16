# Set 11

| S.No. | Question                                                                                                                                                                    |
| ----- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1.    | [How does NestJS implement **Inversion of Control (IoC)** internally?](#question-1-how-does-nestjs-implement-inversion-of-control-ioc-internally)                           |
| 2.    | [How does **Reflector** work in NestJS?](#question-2-how-does-reflector-work-in-nestjs)                                                                                     |
| 3.    | [How do **custom decorators** interact with metadata?](#question-3-how-do-custom-decorators-interact-with-metadata)                                                         |
| 4.    | [What is the difference between **synchronous and asynchronous lifecycle hooks**?](#question-4-what-is-the-difference-between-synchronous-and-asynchronous-lifecycle-hooks) |
| 5.    | [How does **NestJS handle dependency resolution** under the hood?](#question-5-how-does-nestjs-handle-dependency-resolution-under-the-hood)                                 |
| 6.    | [Explain **ModuleRef** and how it is used](#question-6-explain-moduleref-and-how-it-is-used)                                                                                |
| 7.    | [How do you dynamically resolve dependencies at runtime?](#question-7-how-do-you-dynamically-resolve-dependencies-at-runtime)                                               |
| 8.    | [How do you create **dynamic controllers**?](#question-8-how-do-you-create-dynamic-controllers)                                                                             |
| 9.    | [How does NestJS implement **lazy module loading**?](#question-9-how-does-nestjs-implement-lazy-module-loading)                                                             |
| 10.   | [How does NestJS handle **circular module references** internally?](#question-10-how-does-nestjs-handle-circular-module-references-internally)                              |
| 11.   | [How do you use `OnApplicationShutdown` hook?](#question-11-how-do-you-use-onapplicationshutdown-hook)                                                                      |
| 12.   | [How do you implement **cleanup tasks on application exit**?](#question-12-how-do-you-implement-cleanup-tasks-on-application-exit)                                          |
| 13.   | [How does **NestJS integrate with Express vs Fastify**?](#question-13-how-does-nestjs-integrate-with-express-vs-fastify)                                                    |
| 14.   | [How do you switch **HTTP adapters** in NestJS?](#question-14-how-do-you-switch-http-adapters-in-nestjs)                                                                    |
| 15.   | [What are **application context hierarchies** in NestJS?](#question-15-what-are-application-context-hierarchies-in-nestjs)                                                  |
| 16.   | [How does **module encapsulation** work?](#question-16-how-does-module-encapsulation-work)                                                                                  |
| 17.   | [How does **request-scoped provider caching** work internally?](#question-17-how-does-request-scoped-provider-caching-work-internally)                                      |
| 18.   | [How do you implement **context-aware logging**?](#question-18-how-do-you-implement-context-aware-logging)                                                                  |
| 19.   | [What are the limitations of **request-scoped providers**?](#question-19-what-are-the-limitations-of-request-scoped-providers)                                              |
| 20.   | [How do you implement **multi-tenant service instances**?](#question-20-how-do-you-implement-multi-tenant-service-instances)                                                |

## Question 1. How does NestJS implement **Inversion of Control (IoC)** internally?

## Short answer

NestJS implements Inversion of Control (IoC) using a **dependency injection (DI) container** that automatically resolves and injects providers based on metadata (primarily TypeScript decorators + reflection). Instead of manually instantiating dependencies, NestJS creates and manages them in a **module-scoped IoC container**.

---

## Explanation

At its core, NestJS is built around a **metadata-driven IoC container** inspired by Angular. The key idea is:

> You declare dependencies; NestJS resolves and injects them at runtime.

### 1. Metadata + Reflection (design-time → runtime bridge)

Nest uses `reflect-metadata` to store dependency information:

- `@Injectable()` marks a class as a provider
- Constructor parameter types are emitted by TypeScript (`design:paramtypes`)
- Custom tokens can override type-based resolution

Example metadata:

```ts
Reflect.getMetadata("design:paramtypes", SomeService);
```

This allows Nest to know:

> “What does this class depend on?”

---

### 2. Module-based DI container

Each `@Module()` creates a **scoped container context**:

- Providers registered in `providers: []`
- Exported providers shared across modules
- Each module has its own injector hierarchy

Nest builds a **module graph**, not just a flat container.

---

### 3. Provider resolution algorithm

When a class is requested:

1. Nest checks if it already exists in the module scope (singleton by default)
2. If not, it:
   - Reads constructor metadata
   - Resolves each dependency recursively
   - Instantiates dependencies in correct order

3. Stores instance in container (unless `Scope.REQUEST` or `TRANSIENT`)

---

### 4. IoC types supported

NestJS supports multiple lifecycle scopes:

- **Singleton (default)** → shared across app
- **Request scope** → new instance per request
- **Transient** → new instance every injection

This is crucial for performance vs isolation trade-offs.

---

### 5. Provider tokens and abstraction

IoC is not limited to classes:

- Class-based providers
- Value providers
- Factory providers
- Async providers

This allows abstraction like:

- swapping implementations (e.g., mock DB, Redis vs memory cache)
- environment-based configuration

---

### 6. Circular dependency handling

Nest resolves circular dependencies using:

- forward references (`forwardRef`)
- lazy resolution inside DI container

But this increases complexity and can indicate design issues.

---

### Example

```ts
import { Module, Injectable } from "@nestjs/common";

@Injectable()
class UserRepository {
  findAll() {
    return ["user1", "user2"];
  }
}

@Injectable()
class UserService {
  constructor(private readonly repo: UserRepository) {}

  getUsers() {
    return this.repo.findAll();
  }
}

@Module({
  providers: [UserService, UserRepository],
  exports: [UserService],
})
class UserModule {}
```

Nest internally:

- detects `UserService` depends on `UserRepository`
- creates `UserRepository` first
- injects it into `UserService`
- caches both in module container

---

## Pitfalls

- Overusing request-scoped providers → **severe performance degradation**
- Hidden circular dependencies → runtime injection errors or `undefined` providers
- Misconfigured module exports → providers not visible across modules
- Over-reliance on global modules → breaks modular boundaries and testability
- Reflection-based DI can break in edge cases (e.g., pure JS without metadata)

## Question 2. How does **Reflector** work in NestJS?

## Question 3. How do **custom decorators** interact with metadata?

## Question 4. What is the difference between **synchronous and asynchronous lifecycle hooks**?

## Question 5. How does **NestJS handle dependency resolution** under the hood?

## Question 6. Explain **ModuleRef** and how it is used

## Question 7. How do you dynamically resolve dependencies at runtime?

## Question 8. How do you create **dynamic controllers**?

## Question 9. How does NestJS implement **lazy module loading**?

## Question 10. How does NestJS handle **circular module references** internally?

## Question 11. How do you use `OnApplicationShutdown` hook?

## Question 12. How do you implement **cleanup tasks on application exit**?

## Question 13. How does **NestJS integrate with Express vs Fastify**?

## Question 14. How do you switch **HTTP adapters** in NestJS?

## Question 15. What are **application context hierarchies** in NestJS?

## Question 16. How does **module encapsulation** work?

## Question 17. How does **request-scoped provider caching** work internally?

## Question 18. How do you implement **context-aware logging**?

## Question 19. What are the limitations of **request-scoped providers**?

## Question 20. How do you implement **multi-tenant service instances**?
