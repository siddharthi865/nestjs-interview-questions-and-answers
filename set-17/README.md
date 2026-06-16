# Set 17

| S.No. | Question                                                                                                                                                       |
| ----- | -------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1.    | [How do you implement **global caching** in NestJS?](#question-1-how-do-you-implement-global-caching-in-nestjs)                                                |
| 2.    | [How do you use **Redis as a cache store**?](#question-2-how-do-you-use-redis-as-a-cache-store)                                                                |
| 3.    | [How do you implement **request-level caching**?](#question-3-how-do-you-implement-request-level-caching)                                                      |
| 4.    | [How do you implement **response caching** with interceptors?](#question-4-how-do-you-implement-response-caching-with-interceptors)                            |
| 5.    | [How do you implement **conditional caching** based on user roles?](#question-5-how-do-you-implement-conditional-caching-based-on-user-roles)                  |
| 6.    | [How do you implement **TTL (time-to-live) caching**?](#question-6-how-do-you-implement-ttl-time-to-live-caching)                                              |
| 7.    | [How do you implement **cache invalidation** strategies?](#question-7-how-do-you-implement-cache-invalidation-strategies)                                      |
| 8.    | [How do you implement **per-route caching**?](#question-8-how-do-you-implement-per-route-caching)                                                              |
| 9.    | [How do you implement **query result caching** with TypeORM?](#question-9-how-do-you-implement-query-result-caching-with-typeorm)                              |
| 10.   | [How do you implement **in-memory caching for lightweight data**?](#question-10-how-do-you-implement-in-memory-caching-for-lightweight-data)                   |
| 11.   | [How do you implement **multi-level caching** (memory + Redis)?](#question-11-how-do-you-implement-multi-level-caching-memory--redis)                          |
| 12.   | [How do you **profile NestJS performance**?](#question-12-how-do-you-profile-nestjs-performance)                                                               |
| 13.   | [How do you monitor **database query performance**?](#question-13-how-do-you-monitor-database-query-performance)                                               |
| 14.   | [How do you implement **lazy loading for modules** to reduce startup time?](#question-14-how-do-you-implement-lazy-loading-for-modules-to-reduce-startup-time) |
| 15.   | [How do you optimize **JSON serialization/deserialization**?](#question-15-how-do-you-optimize-json-serializationdeserialization)                              |
| 16.   | [How do you implement **rate-limiting for APIs**?](#question-16-how-do-you-implement-rate-limiting-for-apis)                                                   |
| 17.   | [How do you implement **compression for responses**?](#question-17-how-do-you-implement-compression-for-responses)                                             |
| 18.   | [How do you implement **connection pooling**?](#question-18-how-do-you-implement-connection-pooling)                                                           |
| 19.   | [How do you implement **asynchronous job queues** for heavy tasks?](#question-19-how-do-you-implement-asynchronous-job-queues-for-heavy-tasks)                 |
| 20.   | [How do you optimize **NestJS startup time**?](#question-20-how-do-you-optimize-nestjs-startup-time)                                                           |

## Question 1. How do you implement **global caching** in NestJS?

## Short answer

Global caching in NestJS is implemented using the `@nestjs/cache-manager` package with a **CacheModule configured as global** (`isGlobal: true`) and optionally backed by Redis for distributed caching.

---

## Explanation

In NestJS, caching is typically built on top of the **Cache Manager abstraction**, which supports multiple stores (in-memory, Redis, Memcached, etc.). Global caching means:

- Cache is available across all modules without re-importing `CacheModule`
- Controllers/services can inject `CACHE_MANAGER`
- You can apply caching at:
  - Method level (manual caching via `cacheManager.get/set`)
  - Route level (via `CacheInterceptor`)
  - Global level (interceptor applied via `APP_INTERCEPTOR`)

### Architecture Overview

1. **CacheModule (global scope)**
   Registers cache provider in DI container

2. **Cache Interceptor (optional but common)**
   Automatically caches HTTP responses based on request key

3. **Cache Store (in-memory or Redis)**
   - In-memory: fast, single instance only
   - Redis: scalable across multiple instances (recommended for production)

---

### Scalability & Trade-offs

| Strategy        | Pros                  | Cons                                 |
| --------------- | --------------------- | ------------------------------------ |
| In-memory cache | Fast, simple          | Not shared across pods/instances     |
| Redis cache     | Distributed, scalable | Network latency, external dependency |
| Hybrid          | Flexible fallback     | More complexity                      |

---

## Example

### 1. Global CacheModule setup (Redis-backed)

```ts
// app.module.ts
import { Module } from "@nestjs/common";
import { CacheModule } from "@nestjs/cache-manager";
import { APP_INTERCEPTOR } from "@nestjs/core";
import { CacheInterceptor } from "@nestjs/cache-manager";
import * as redisStore from "cache-manager-ioredis";

@Module({
  imports: [
    CacheModule.register({
      isGlobal: true,
      store: redisStore,
      host: "localhost",
      port: 6379,
      ttl: 60, // seconds
    }),
  ],
  providers: [
    {
      provide: APP_INTERCEPTOR,
      useClass: CacheInterceptor,
    },
  ],
})
export class AppModule {}
```

---

### 2. Using cache manually in a service

```ts
import { Injectable, Inject } from "@nestjs/common";
import { CACHE_MANAGER } from "@nestjs/cache-manager";
import { Cache } from "cache-manager";

@Injectable()
export class UsersService {
  constructor(@Inject(CACHE_MANAGER) private cacheManager: Cache) {}

  async findUser(id: string) {
    const cacheKey = `user:${id}`;

    const cached = await this.cacheManager.get(cacheKey);
    if (cached) return cached;

    const user = await this.fetchFromDb(id);

    await this.cacheManager.set(cacheKey, user, 60); // TTL 60s
    return user;
  }

  private async fetchFromDb(id: string) {
    return { id, name: "John Doe" };
  }
}
```

---

### 3. Controller-level automatic caching

```ts
import { Controller, Get, UseInterceptors } from "@nestjs/common";
import { CacheInterceptor } from "@nestjs/cache-manager";

@Controller("users")
@UseInterceptors(CacheInterceptor)
export class UsersController {
  @Get()
  findAll() {
    return [{ id: 1, name: "Alice" }];
  }
}
```

---

## Pitfalls

- **Cache invalidation complexity** (most common production issue)
- **Stale data risks** if TTLs are too long or invalidation is missing
- **Memory leaks** with in-memory cache under high traffic
- **Non-deterministic caching keys** (e.g., query params ordering issues)
- **Multi-instance inconsistency** if not using Redis in Kubernetes/clustered deployments

## Question 2. How do you use **Redis as a cache store**?

## Question 3. How do you implement **request-level caching**?

## Question 4. How do you implement **response caching** with interceptors?

## Question 5. How do you implement **conditional caching** based on user roles?

## Question 6. How do you implement **TTL (time-to-live) caching**?

## Question 7. How do you implement **cache invalidation** strategies?

## Question 8. How do you implement **per-route caching**?

## Question 9. How do you implement **query result caching** with TypeORM?

## Question 10. How do you implement **in-memory caching for lightweight data**?

## Question 11. How do you implement **multi-level caching** (memory + Redis)?

## Question 12. How do you **profile NestJS performance**?

## Question 13. How do you monitor **database query performance**?

## Question 14. How do you implement **lazy loading for modules** to reduce startup time?

## Question 15. How do you optimize **JSON serialization/deserialization**?

## Question 16. How do you implement **rate-limiting for APIs**?

## Question 17. How do you implement **compression for responses**?

## Question 18. How do you implement **connection pooling**?

## Question 19. How do you implement **asynchronous job queues** for heavy tasks?

## Question 20. How do you optimize **NestJS startup time**?
