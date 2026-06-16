# Set 20

| S.No. | Question                                                                                                                                       |
| ----- | ---------------------------------------------------------------------------------------------------------------------------------------------- |
| 1.    | [How do you implement **modular architecture** for large apps?](#question-1-how-do-you-implement-modular-architecture-for-large-apps)          |
| 2.    | [How do you implement **feature-based folder structure**?](#question-2-how-do-you-implement-feature-based-folder-structure)                    |
| 3.    | [How do you implement **shared modules**?](#question-3-how-do-you-implement-shared-modules)                                                    |
| 4.    | [How do you implement **core modules for services and utilities**?](#question-4-how-do-you-implement-core-modules-for-services-and-utilities)  |
| 5.    | [How do you implement **environment-based configuration modules**?](#question-5-how-do-you-implement-environment-based-configuration-modules)  |
| 6.    | [How do you handle **versioning of APIs**?](#question-6-how-do-you-handle-versioning-of-apis)                                                  |
| 7.    | [How do you handle **backward compatibility in APIs**?](#question-7-how-do-you-handle-backward-compatibility-in-apis)                          |
| 8.    | [How do you implement **API documentation with Swagger**?](#question-8-how-do-you-implement-api-documentation-with-swagger)                    |
| 9.    | [How do you generate **GraphQL schema automatically**?](#question-9-how-do-you-generate-graphql-schema-automatically)                          |
| 10.   | [How do you implement **error handling best practices**?](#question-10-how-do-you-implement-error-handling-best-practices)                     |
| 11.   | [How do you implement **logging best practices**?](#question-11-how-do-you-implement-logging-best-practices)                                   |
| 12.   | [How do you implement **caching best practices**?](#question-12-how-do-you-implement-caching-best-practices)                                   |
| 13.   | [How do you implement **transaction best practices**?](#question-13-how-do-you-implement-transaction-best-practices)                           |
| 14.   | [How do you implement **security best practices**?](#question-14-how-do-you-implement-security-best-practices)                                 |
| 15.   | [How do you handle **sensitive data encryption**?](#question-15-how-do-you-handle-sensitive-data-encryption)                                   |
| 16.   | [How do you implement **environment-specific validation**?](#question-16-how-do-you-implement-environment-specific-validation)                 |
| 17.   | [How do you implement **multi-module testing strategy**?](#question-17-how-do-you-implement-multi-module-testing-strategy)                     |
| 18.   | [How do you implement **async job scheduling**?](#question-18-how-do-you-implement-async-job-scheduling)                                       |
| 19.   | [How do you implement **event-driven microservices architecture**?](#question-19-how-do-you-implement-event-driven-microservices-architecture) |
| 20.   | [How do you implement **CQRS with real-world examples**?](#question-20-how-do-you-implement-cqrs-with-real-world-examples)                     |

## Question 1. How do you implement **modular architecture** for large apps?

## Short answer

Modular architecture in NestJS is implemented by splitting the application into **feature-based modules**, each encapsulating its own controllers, services, providers, and optionally database entities, and then composing them in a root `AppModule` with clear dependency boundaries and shared/global modules.

---

## Explanation

In large-scale NestJS applications, modular architecture is the backbone of **maintainability, scalability, and team autonomy**.

### 1. Core idea: feature-based encapsulation

Each business domain becomes a module:

- `UsersModule`
- `AuthModule`
- `PaymentsModule`
- `OrdersModule`

Each module owns:

- Controllers (API layer)
- Services (business logic)
- Providers (infrastructure adapters)
- Entities/Repositories (data access)

This enforces **high cohesion and low coupling**.

---

### 2. Module boundaries and dependency control

NestJS modules explicitly declare dependencies:

- `imports`: other modules required
- `exports`: providers exposed to other modules
- `providers`: internal services
- `controllers`: route handlers

This creates a **dependency graph controlled at compile time**, preventing hidden coupling.

---

### 3. Shared and global modules

Large systems usually introduce:

#### SharedModule

- Common utilities (logger, config, guards, interceptors)

#### Global modules (`@Global()`)

- Configuration
- Logging
- Database connection

But overusing global modules leads to:

- hidden dependencies
- harder testing
- unclear ownership

---

### 4. Scalability patterns

For enterprise systems:

#### a. Domain-driven modules (DDD style)

- `User`
- `Billing`
- `Notification`
  Each module aligns with business domains.

#### b. Layered inside module

- `controller/`
- `service/`
- `repository/`
- `dto/`

#### c. Feature isolation

Avoid cross-module direct service injection unless necessary. Prefer:

- events (event-driven architecture)
- interfaces via tokens
- shared contracts

---

### 5. Testing strategy

- Unit test per module (services isolated)
- Mock inter-module dependencies
- E2E tests at app boundary
- Use `Test.createTestingModule()` per module

---

### 6. Deployment implications

- Modules map well to **microservices later**
- Can extract a module into a separate NestJS service
- Supports monorepos (Nx / Turborepo)
- Enables independent scaling if split into microservices

---

### 7. Trade-offs

- More modules = more boilerplate
- Over-modularization can hurt readability
- Requires discipline in dependency management

---

## Example

### `users.module.ts`

```ts
import { Module } from "@nestjs/common";
import { UsersController } from "./users.controller";
import { UsersService } from "./users.service";

@Module({
  controllers: [UsersController],
  providers: [UsersService],
  exports: [UsersService], // exposed for other modules (e.g., AuthModule)
})
export class UsersModule {}
```

### `auth.module.ts` consuming UsersModule

```ts
import { Module } from "@nestjs/common";
import { AuthService } from "./auth.service";
import { UsersModule } from "../users/users.module";

@Module({
  imports: [UsersModule],
  providers: [AuthService],
})
export class AuthModule {}
```

### `auth.service.ts`

```ts
import { Injectable } from "@nestjs/common";
import { UsersService } from "../users/users.service";

@Injectable()
export class AuthService {
  constructor(private readonly usersService: UsersService) {}

  async validateUser(email: string, password: string) {
    const user = await this.usersService.findByEmail(email);
    return user && user.password === password ? user : null;
  }
}
```

---

## Pitfalls

- ❌ Overusing `@Global()` modules → hidden dependencies and tight coupling
- ❌ Circular dependencies between modules (often from poor boundary design)
- ❌ Sharing ORM entities directly across modules without abstraction
- ❌ Putting too much logic into a single “god module” (e.g., AppModule)
- ❌ Direct service-to-service coupling instead of event-driven decoupling in large systems

## Question 2. How do you implement **feature-based folder structure**?

## Question 3. How do you implement **shared modules**?

## Question 4. How do you implement **core modules for services and utilities**?

## Question 5. How do you implement **environment-based configuration modules**?

## Question 6. How do you handle **versioning of APIs**?

## Question 7. How do you handle **backward compatibility in APIs**?

## Question 8. How do you implement **API documentation with Swagger**?

## Question 9. How do you generate **GraphQL schema automatically**?

## Question 10. How do you implement **error handling best practices**?

## Question 11. How do you implement **logging best practices**?

## Question 12. How do you implement **caching best practices**?

## Question 13. How do you implement **transaction best practices**?

## Question 14. How do you implement **security best practices**?

## Question 15. How do you handle **sensitive data encryption**?

## Question 16. How do you implement **environment-specific validation**?

## Question 17. How do you implement **multi-module testing strategy**?

## Question 18. How do you implement **async job scheduling**?

## Question 19. How do you implement **event-driven microservices architecture**?

## Question 20. How do you implement **CQRS with real-world examples**?
