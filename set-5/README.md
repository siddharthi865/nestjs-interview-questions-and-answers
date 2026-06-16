# Set 5

| S.No. | Question                                                                                                                                                           |
| ----- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| 1.    | [How do you implement **event sourcing**?](#question-1-how-do-you-implement-event-sourcing)                                                                        |
| 2.    | [Explain **request-scoped providers** in microservices](#question-2-explain-request-scoped-providers-in-microservices)                                             |
| 3.    | [How do you handle **async/await with pipes and guards**?](#question-3-how-do-you-handle-asyncawait-with-pipes-and-guards)                                         |
| 4.    | [How do you implement **global modules**?](#question-4-how-do-you-implement-global-modules)                                                                        |
| 5.    | [How do you implement **inter-service communication** in microservices?](#question-5-how-do-you-implement-inter-service-communication-in-microservices)            |
| 6.    | [How do you write **unit tests** in NestJS?](#question-6-how-do-you-write-unit-tests-in-nestjs)                                                                    |
| 7.    | [How do you write **integration tests** in NestJS?](#question-7-how-do-you-write-integration-tests-in-nestjs)                                                      |
| 8.    | [What is the difference between **unit, integration, and end-to-end tests**?](#question-8-what-is-the-difference-between-unit-integration-and-end-to-end-tests)    |
| 9.    | [How do you mock providers in tests?](#question-9-how-do-you-mock-providers-in-tests)                                                                              |
| 10.   | [How do you test **controllers** and **services**?](#question-10-how-do-you-test-controllers-and-services)                                                         |
| 11.   | [How do you test **guards, pipes, and interceptors**?](#question-11-how-do-you-test-guards-pipes-and-interceptors)                                                 |
| 12.   | [How do you run **e2e tests** in NestJS?](#question-12-how-do-you-run-e2e-tests-in-nestjs)                                                                         |
| 13.   | [How do you use **Supertest** for testing controllers?](#question-13-how-do-you-use-supertest-for-testing-controllers)                                             |
| 14.   | [How do you test **asynchronous services**?](#question-14-how-do-you-test-asynchronous-services)                                                                   |
| 15.   | [How do you test **GraphQL endpoints**?](#question-15-how-do-you-test-graphql-endpoints)                                                                           |
| 16.   | [How do you implement **caching** in NestJS?](#question-16-how-do-you-implement-caching-in-nestjs)                                                                 |
| 17.   | [How do you use **Redis** with NestJS for caching?](#question-17-how-do-you-use-redis-with-nestjs-for-caching)                                                     |
| 18.   | [How do you optimize **database queries** for performance?](#question-18-how-do-you-optimize-database-queries-for-performance)                                     |
| 19.   | [How do you handle **logging** in production?](#question-19-how-do-you-handle-logging-in-production)                                                               |
| 20.   | [What are **best practices for folder structure** in a large NestJS project?](#question-20-what-are-best-practices-for-folder-structure-in-a-large-nestjs-project) |

## Question 1. How do you implement **event sourcing**?

## Short answer

Event sourcing in NestJS is implemented by storing all state changes as immutable events in an event store, rebuilding current state by replaying events, and optionally projecting those events into read models using CQRS patterns (`@nestjs/cqrs`).

---

## Explanation

### 1. Core concept (architecture)

Event sourcing replaces traditional CRUD persistence with **append-only events**:

- Instead of storing `User { name: "John" }`
- You store events like:
  - `UserCreated`
  - `UserNameChanged`
  - `UserEmailUpdated`

The **current state is derived**, not stored directly.

In NestJS, this is typically implemented using:

- **Event Store** (database or log system: PostgreSQL, MongoDB, EventStoreDB, Kafka)
- **Command layer** (writes intent)
- **Event layer** (facts that happened)
- **Projection layer** (read models / views)
- **CQRS module** (`@nestjs/cqrs`) for separation of concerns

---

### 2. Flow in a NestJS system

1. Controller receives command (e.g., `CreateUser`)
2. Command handler validates business rules
3. Command handler emits domain events
4. Events are persisted in event store
5. Event handlers update projections (read models)
6. Queries read from projections (not event store)

---

### 3. Trade-offs

#### Advantages

- Full audit trail (every state change)
- Time travel debugging (rebuild state at any point in time)
- High scalability for reads (via projections)
- Strong domain modeling (DDD-friendly)

#### Disadvantages

- Higher complexity
- Event schema evolution is hard
- Requires CQRS discipline
- Debugging can be harder without tooling

---

### 4. When to use

- Financial systems (banking, billing)
- Audit-heavy systems (compliance)
- Collaborative systems (docs, workflows)
- Systems requiring replayability

---

## Example

Using `@nestjs/cqrs` with a simplified in-memory event store.

### 1. Event definitions

```ts
// user.events.ts
export class UserCreatedEvent {
  constructor(
    public readonly userId: string,
    public readonly email: string,
  ) {}
}
```

---

### 2. Command

```ts
// create-user.command.ts
export class CreateUserCommand {
  constructor(
    public readonly userId: string,
    public readonly email: string,
  ) {}
}
```

---

### 3. Event Store (simplified)

```ts
import { Injectable } from "@nestjs/common";

@Injectable()
export class EventStore {
  private readonly events: any[] = [];

  append(event: any) {
    this.events.push(event);
  }

  getAll() {
    return this.events;
  }
}
```

---

### 4. Command Handler

```ts
import { CommandHandler, ICommandHandler, EventBus } from "@nestjs/cqrs";

@CommandHandler(CreateUserCommand)
export class CreateUserHandler implements ICommandHandler<CreateUserCommand> {
  constructor(
    private readonly eventBus: EventBus,
    private readonly eventStore: EventStore,
  ) {}

  async execute(command: CreateUserCommand) {
    const event = new UserCreatedEvent(command.userId, command.email);

    this.eventStore.append(event);
    this.eventBus.publish(event);

    return { success: true };
  }
}
```

---

### 5. Event Handler (Projection update)

```ts
import { EventsHandler, IEventHandler } from "@nestjs/cqrs";

interface UserReadModel {
  id: string;
  email: string;
}

const users: UserReadModel[] = [];

@EventsHandler(UserCreatedEvent)
export class UserCreatedHandler implements IEventHandler<UserCreatedEvent> {
  handle(event: UserCreatedEvent) {
    users.push({
      id: event.userId,
      email: event.email,
    });
  }
}
```

---

### 6. Module setup

```ts
import { Module } from "@nestjs/common";
import { CqrsModule } from "@nestjs/cqrs";

@Module({
  imports: [CqrsModule],
  providers: [EventStore, CreateUserHandler, UserCreatedHandler],
})
export class UserModule {}
```

---

## Pitfalls

- Event schema evolution without versioning breaks replays
- No idempotency → duplicate event processing issues
- Projections can become inconsistent if not carefully managed
- Event store becomes a single critical dependency (needs scaling strategy)
- Debugging requires tooling (logs alone are insufficient)

## Question 2. Explain **request-scoped providers** in microservices

## Question 3. How do you handle **async/await with pipes and guards**?

## Question 4. How do you implement **global modules**?

## Question 5. How do you implement **inter-service communication** in microservices?

## Question 6. How do you write **unit tests** in NestJS?

## Question 7. How do you write **integration tests** in NestJS?

## Question 8. What is the difference between **unit, integration, and end-to-end tests**?

## Question 9. How do you mock providers in tests?

## Question 10. How do you test **controllers** and **services**?

## Question 11. How do you test **guards, pipes, and interceptors**?

## Question 12. How do you run **e2e tests** in NestJS?

## Question 13. How do you use **Supertest** for testing controllers?

## Question 14. How do you test **asynchronous services**?

## Question 15. How do you test **GraphQL endpoints**?

## Question 16. How do you implement **caching** in NestJS?

## Question 17. How do you use **Redis** with NestJS for caching?

## Question 18. How do you optimize **database queries** for performance?

## Question 19. How do you handle **logging** in production?

## Question 20. What are **best practices for folder structure** in a large NestJS project?
