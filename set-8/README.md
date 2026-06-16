# Set 8

| S.No. | Question                                                                                                                                                        |
| ----- | --------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1.    | [How do you provide **third-party libraries** via NestJS providers?](#question-1-how-do-you-provide-third-party-libraries-via-nestjs-providers)                 |
| 2.    | [How do you use **factory providers**?](#question-2-how-do-you-use-factory-providers)                                                                           |
| 3.    | [How do you implement **async providers** with `useFactory`?](#question-3-how-do-you-implement-async-providers-with-usefactory)                                 |
| 4.    | [How do you inject **custom tokens** in NestJS?](#question-4-how-do-you-inject-custom-tokens-in-nestjs)                                                         |
| 5.    | [What is the difference between **singleton and request-scoped providers**?](#question-5-what-is-the-difference-between-singleton-and-request-scoped-providers) |
| 6.    | [How do you implement **request-specific services**?](#question-6-how-do-you-implement-request-specific-services)                                               |
| 7.    | [How do you use `@Optional()` decorator in NestJS?](#question-7-how-do-you-use-optional-decorator-in-nestjs)                                                    |
| 8.    | [How do you handle **circular dependencies** between modules?](#question-8-how-do-you-handle-circular-dependencies-between-modules)                             |
| 9.    | [What is the purpose of `forwardRef()`?](#question-9-what-is-the-purpose-of-forwardref)                                                                         |
| 10.   | [How do you implement **dynamic configuration providers**?](#question-10-how-do-you-implement-dynamic-configuration-providers)                                  |
| 11.   | [How do you share **services between microservices**?](#question-11-how-do-you-share-services-between-microservices)                                            |
| 12.   | [How do you implement **global interceptors** using providers?](#question-12-how-do-you-implement-global-interceptors-using-providers)                          |
| 13.   | [How do you inject **environment variables** into services?](#question-13-how-do-you-inject-environment-variables-into-services)                                |
| 14.   | [How do you create **custom providers for testing**?](#question-14-how-do-you-create-custom-providers-for-testing)                                              |
| 15.   | [How do you implement **custom lifecycle hooks** in providers?](#question-15-how-do-you-implement-custom-lifecycle-hooks-in-providers)                          |
| 16.   | [How do you implement **request logging at service level**?](#question-16-how-do-you-implement-request-logging-at-service-level)                                |
| 17.   | [How do you use **module-level providers**?](#question-17-how-do-you-use-module-level-providers)                                                                |
| 18.   | [How do you inject **constants across modules**?](#question-18-how-do-you-inject-constants-across-modules)                                                      |
| 19.   | [How do you create **multi-instance providers**?](#question-19-how-do-you-create-multi-instance-providers)                                                      |
| 20.   | [How do you implement **decorators for dependency injection**?](#question-20-how-do-you-implement-decorators-for-dependency-injection)                          |

## Question 1. How do you provide **third-party libraries** via NestJS providers?

# Short answer

Third-party libraries are typically provided through **custom providers** in NestJS. You create a provider that instantiates or configures the library (e.g., Axios, Redis, Stripe, AWS SDK), register it in a module, and inject it wherever needed using dependency injection. This centralizes configuration, improves testability, and allows easy replacement or mocking.

---

# Explanation

NestJS's Dependency Injection (DI) container is designed to manage not only your own services but also external libraries.

Instead of creating library instances directly inside services:

```ts
const redis = new Redis();
```

you should register them as providers:

```ts
{
  provide: REDIS_CLIENT,
  useFactory: () => new Redis(...)
}
```

and inject them:

```ts
constructor(
  @Inject(REDIS_CLIENT) private readonly redis: Redis,
) {}
```

## Common provider patterns

### 1. Value Provider (`useValue`)

Useful when the library instance is already created.

```ts
{
  provide: REDIS_CLIENT,
  useValue: redisInstance,
}
```

Good for:

- Configuration objects
- SDK clients created during bootstrap
- Constants

---

### 2. Factory Provider (`useFactory`) — Most Common

Creates and configures the library dynamically.

```ts
{
  provide: REDIS_CLIENT,
  useFactory: (config: ConfigService) => {
    return new Redis(config.getOrThrow('REDIS_URL'));
  },
  inject: [ConfigService],
}
```

Good for:

- Database clients
- Redis
- AWS SDK
- Stripe
- Elasticsearch
- Kafka

This is usually the preferred enterprise pattern.

---

### 3. Class Provider (`useClass`)

Wrap a third-party library inside an adapter.

```ts
{
  provide: PaymentGateway,
  useClass: StripePaymentGateway,
}
```

Benefits:

- Decouples application from vendor SDK
- Easier migrations
- Easier testing

This follows the Dependency Inversion Principle.

---

### Dynamic Modules for Reusable Libraries

When building shared modules, expose a configurable module:

```ts
PaymentModule.forRoot({
  apiKey: "xxx",
});
```

This pattern is used extensively by NestJS ecosystem packages such as:

- [NestJS Config Module](https://docs.nestjs.com/techniques/configuration?utm_source=chatgpt.com)
- [NestJS TypeORM Module](https://docs.nestjs.com/techniques/database?utm_source=chatgpt.com)
- [NestJS Microservices Module](https://docs.nestjs.com/microservices/basics?utm_source=chatgpt.com)

For enterprise applications, reusable integrations are usually implemented as Dynamic Modules with `forRoot()` and `forRootAsync()` methods.

---

## Testing implications

A major advantage of provider-based integration is easy mocking.

Instead of connecting to a real Redis server:

```ts
{
  provide: REDIS_CLIENT,
  useValue: mockRedis,
}
```

Unit tests remain isolated and fast.

Example:

```ts
{
  provide: REDIS_CLIENT,
  useValue: {
    get: jest.fn(),
    set: jest.fn(),
  },
}
```

---

## Scalability and deployment considerations

### Connection lifecycle

Long-lived clients should be initialized once.

Examples:

- Redis
- Kafka
- PostgreSQL pools
- AWS SDK clients

Prefer singleton providers (default scope).

### Health checks

External dependencies should expose readiness/liveness information.

For example:

- Redis ping
- Kafka broker connectivity
- Database connection status

Often implemented with:

- [NestJS Terminus](https://docs.nestjs.com/recipes/terminus?utm_source=chatgpt.com)

### Configuration management

Avoid hardcoded credentials.

Inject configuration via:

```ts
ConfigService;
```

and environment variables.

### Resource cleanup

Use lifecycle hooks:

```ts
OnModuleDestroy;
```

or

```ts
OnApplicationShutdown;
```

to close connections gracefully.

---

# Example

A Redis client provided through a custom provider.

```ts
import { Module, Injectable, Inject } from "@nestjs/common";
import { ConfigModule, ConfigService } from "@nestjs/config";
import Redis from "ioredis";

export const REDIS_CLIENT = Symbol("REDIS_CLIENT");

@Injectable()
export class CacheService {
  constructor(
    @Inject(REDIS_CLIENT)
    private readonly redis: Redis,
  ) {}

  async getValue(key: string): Promise<string | null> {
    return this.redis.get(key);
  }
}

@Module({
  imports: [ConfigModule.forRoot()],
  providers: [
    {
      provide: REDIS_CLIENT,
      useFactory: (config: ConfigService): Redis => {
        return new Redis(config.getOrThrow<string>("REDIS_URL"));
      },
      inject: [ConfigService],
    },
    CacheService,
  ],
  exports: [REDIS_CLIENT, CacheService],
})
export class CacheModule {}
```

---

# Pitfalls

- Creating SDK/database clients inside services instead of providers leads to multiple connections and harder testing.
- Using string tokens (`'REDIS_CLIENT'`) across large monorepos can cause collisions; prefer `Symbol()` tokens.
- Forgetting connection cleanup can cause hanging processes during shutdown and failed Kubernetes pod termination.
- Directly coupling business logic to vendor SDKs makes migrations (e.g., Stripe → Adyen, Redis → KeyDB) expensive; consider adapter abstractions.

## Question 2. How do you use **factory providers**?

## Question 3. How do you implement **async providers** with `useFactory`?

## Question 4. How do you inject **custom tokens** in NestJS?

## Question 5. What is the difference between **singleton and request-scoped providers**?

## Question 6. How do you implement **request-specific services**?

## Question 7. How do you use `@Optional()` decorator in NestJS?

## Question 8. How do you handle **circular dependencies** between modules?

## Question 9. What is the purpose of `forwardRef()`?

## Question 10. How do you implement **dynamic configuration providers**?

## Question 11. How do you share **services between microservices**?

## Question 12. How do you implement **global interceptors** using providers?

## Question 13. How do you inject **environment variables** into services?

## Question 14. How do you create **custom providers for testing**?

## Question 15. How do you implement **custom lifecycle hooks** in providers?

## Question 16. How do you implement **request logging at service level**?

## Question 17. How do you use **module-level providers**?

## Question 18. How do you inject **constants across modules**?

## Question 19. How do you create **multi-instance providers**?

## Question 20. How do you implement **decorators for dependency injection**?
