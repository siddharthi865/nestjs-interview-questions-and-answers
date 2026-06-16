# Set 3

| S.No. | Question                                                                                                                                            |
| ----- | --------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1.    | [What is the difference between a provider and a service in NestJS?](#question-1-what-is-the-difference-between-a-provider-and-a-service-in-nestjs) |
| 2.    | [How does NestJS manage dependency injection?](#question-2-how-does-nestjs-manage-dependency-injection)                                             |
| 3.    | [Explain **custom providers** and when to use them](#question-3-explain-custom-providers-and-when-to-use-them)                                      |
| 4.    | [What are **asynchronous providers**?](#question-4-what-are-asynchronous-providers)                                                                 |
| 5.    | [How do you inject **values or constants** into services?](#question-5-how-do-you-inject-values-or-constants-into-services)                         |
| 6.    | [What is `forwardRef()` in NestJS?](#question-6-what-is-forwardref-in-nestjs)                                                                       |
| 7.    | [How do you inject one module into another module?](#question-7-how-do-you-inject-one-module-into-another-module)                                   |
| 8.    | [Explain the **request vs singleton scope** of providers](#question-8-explain-the-request-vs-singleton-scope-of-providers)                          |
| 9.    | [How do you implement **dynamic modules**?](#question-9-how-do-you-implement-dynamic-modules)                                                       |
| 10.   | [How do you provide **configuration values** across the app?](#question-10-how-do-you-provide-configuration-values-across-the-app)                  |
| 11.   | [How do you integrate **TypeORM** with NestJS?](#question-11-how-do-you-integrate-typeorm-with-nestjs)                                              |
| 12.   | [How do you integrate **Prisma** with NestJS?](#question-12-how-do-you-integrate-prisma-with-nestjs)                                                |
| 13.   | [How do you create **entities** in TypeORM?](#question-13-how-do-you-create-entities-in-typeorm)                                                    |
| 14.   | [How do you implement **repositories**?](#question-14-how-do-you-implement-repositories)                                                            |
| 15.   | [Explain the difference between `Repository` and `EntityManager`](#question-15-explain-the-difference-between-repository-and-entitymanager)         |
| 16.   | [How do you implement **migrations** in NestJS with TypeORM?](#question-16-how-do-you-implement-migrations-in-nestjs-with-typeorm)                  |
| 17.   | [How do you use **DTOs with entities**?](#question-17-how-do-you-use-dtos-with-entities)                                                            |
| 18.   | [How do you handle **transactions** in NestJS?](#question-18-how-do-you-handle-transactions-in-nestjs)                                              |
| 19.   | [How do you implement **soft deletes**?](#question-19-how-do-you-implement-soft-deletes)                                                            |
| 20.   | [How do you optimize queries with **relations** in TypeORM?](#question-20-how-do-you-optimize-queries-with-relations-in-typeorm)                    |

## Question 1. What is the difference between a provider and a service in NestJS?

## Short answer

A **provider** is a generic NestJS concept for any injectable class/value/factory managed by the DI container, while a **service** is a **specific type of provider** typically used to encapsulate business logic.

---

## Explanation

In NestJS, **Dependency Injection (DI)** is core to the architecture. Everything registered in the DI container is a **provider**.

### 1. Provider (Generic concept)

A **provider** can be:

- A class (`class-based provider`)
- A value (`useValue`)
- A factory (`useFactory`)
- An alias (`useExisting`)

Providers are registered in a module via `providers: []`.

They are used for:

- Business logic (services)
- Repositories / database access layers
- External API clients
- Configuration objects
- Mock implementations in testing

👉 So “provider” is an umbrella term.

---

### 2. Service (Specialized provider)

A **service** is:

- A **class-based provider**
- Usually annotated with `@Injectable()`
- Used to encapsulate **business logic and domain rules**

Example responsibilities:

- User creation logic
- Authentication rules
- Data transformations
- Orchestration between repositories and external APIs

👉 In practice:

> All services are providers, but not all providers are services.

---

### Architectural perspective

#### Why Nest separates the concepts

- **Providers** = DI mechanism abstraction
- **Services** = domain/business layer pattern

This separation allows:

- Swapping implementations without changing consumers
- Easier testing via mocking providers
- Clean architecture layering (Controller → Service → Repository)

---

### Scalability & maintainability implications

- Services keep business logic isolated → easier horizontal scaling and reuse
- Providers allow flexible architecture patterns (multi-tenant DBs, dynamic configs, microservice clients)
- Factory providers enable runtime decision-making (e.g., region-based DB selection)

---

### Security considerations

- Services often enforce **authorization logic (but not authentication itself)**
- Providers like guards/interceptors should handle cross-cutting concerns instead
- Misplacing logic in generic providers can blur security boundaries

---

### Testing implications

- Services are easiest to unit test via DI mocking
- Providers with `useFactory` may require more complex mocking strategies
- You can override providers in tests using `overrideProvider()`

---

## Example

```ts
import { Module, Injectable } from "@nestjs/common";

// Service (class-based provider)
@Injectable()
class UsersService {
  getUsers() {
    return [{ id: 1, name: "Alice" }];
  }
}

// Generic provider (factory example)
const ConfigProvider = {
  provide: "CONFIG",
  useValue: { env: "dev", debug: true },
};

import { Controller, Get, Inject } from "@nestjs/common";

@Controller("users")
class UsersController {
  constructor(
    private readonly usersService: UsersService,
    @Inject("CONFIG") private readonly config: any,
  ) {}

  @Get()
  findAll() {
    return {
      data: this.usersService.getUsers(),
      config: this.config,
    };
  }
}

@Module({
  controllers: [UsersController],
  providers: [UsersService, ConfigProvider],
})
export class UsersModule {}
```

---

## Pitfalls

- Confusing “service” as a framework construct → it’s just a convention, not a special NestJS type
- Overusing services for everything → leads to “god services”
- Misusing providers for business logic without structure → harder maintainability
- Circular dependencies when services tightly couple each other
- Overusing `useFactory` without caching → performance overhead

## Question 2. How does NestJS manage dependency injection?

## Question 3. Explain **custom providers** and when to use them

## Question 4. What are **asynchronous providers**?

## Question 5. How do you inject **values or constants** into services?

## Question 6. What is `forwardRef()` in NestJS?

## Question 7. How do you inject one module into another module?

## Question 8. Explain the **request vs singleton scope** of providers

## Question 9. How do you implement **dynamic modules**?

## Question 10. How do you provide **configuration values** across the app?

## Question 11. How do you integrate **TypeORM** with NestJS?

## Question 12. How do you integrate **Prisma** with NestJS?

## Question 13. How do you create **entities** in TypeORM?

## Question 14. How do you implement **repositories**?

## Question 15. Explain the difference between `Repository` and `EntityManager`

## Question 16. How do you implement **migrations** in NestJS with TypeORM?

## Question 17. How do you use **DTOs with entities**?

## Question 18. How do you handle **transactions** in NestJS?

## Question 19. How do you implement **soft deletes**?

## Question 20. How do you optimize queries with **relations** in TypeORM?
