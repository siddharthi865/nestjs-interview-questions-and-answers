# Set 16

| S.No. | Question                                                                                                                        |
| ----- | ------------------------------------------------------------------------------------------------------------------------------- |
| 1.    | [How do you write **unit tests for controllers** in NestJS?](#question-1-how-do-you-write-unit-tests-for-controllers-in-nestjs) |
| 2.    | [How do you write **unit tests for services**?](#question-2-how-do-you-write-unit-tests-for-services)                           |
| 3.    | [How do you **mock repositories** in unit tests?](#question-3-how-do-you-mock-repositories-in-unit-tests)                       |
| 4.    | [How do you write **integration tests** in NestJS?](#question-4-how-do-you-write-integration-tests-in-nestjs)                   |
| 5.    | [How do you test **asynchronous methods**?](#question-5-how-do-you-test-asynchronous-methods)                                   |
| 6.    | [How do you test **guards** in NestJS?](#question-6-how-do-you-test-guards-in-nestjs)                                           |
| 7.    | [How do you test **pipes**?](#question-7-how-do-you-test-pipes)                                                                 |
| 8.    | [How do you test **interceptors**?](#question-8-how-do-you-test-interceptors)                                                   |
| 9.    | [How do you test **exception filters**?](#question-9-how-do-you-test-exception-filters)                                         |
| 10.   | [How do you test **microservice message handlers**?](#question-10-how-do-you-test-microservice-message-handlers)                |
| 11.   | [How do you use **Supertest for e2e tests**?](#question-11-how-do-you-use-supertest-for-e2e-tests)                              |
| 12.   | [How do you handle **database state in tests**?](#question-12-how-do-you-handle-database-state-in-tests)                        |
| 13.   | [How do you implement **mocking for HTTP requests**?](#question-13-how-do-you-implement-mocking-for-http-requests)              |
| 14.   | [How do you test **GraphQL resolvers**?](#question-14-how-do-you-test-graphql-resolvers)                                        |
| 15.   | [How do you test **WebSocket gateways**?](#question-15-how-do-you-test-websocket-gateways)                                      |
| 16.   | [How do you implement **dependency injection in tests**?](#question-16-how-do-you-implement-dependency-injection-in-tests)      |
| 17.   | [How do you test **request-scoped providers**?](#question-17-how-do-you-test-request-scoped-providers)                          |
| 18.   | [How do you test **dynamic modules**?](#question-18-how-do-you-test-dynamic-modules)                                            |
| 19.   | [How do you test **caching and Redis integration**?](#question-19-how-do-you-test-caching-and-redis-integration)                |
| 20.   | [How do you **debug asynchronous errors** in NestJS?](#question-20-how-do-you-debug-asynchronous-errors-in-nestjs)              |

## Question 1. How do you write **unit tests for controllers** in NestJS?

## Short answer

In NestJS, you unit test controllers by isolating them from the rest of the app using `@nestjs/testing`, mocking their dependent services, and invoking controller methods directly (or via `supertest` only for e2e tests). The goal is to test controller logic, not framework wiring or database behavior.

---

## Explanation

In a well-designed NestJS architecture, controllers should be **thin layers** that:

- Accept HTTP input
- Delegate business logic to services
- Return DTOs/responses

So unit testing controllers focuses on:

- Request → controller mapping correctness
- Validation/pipes behavior (if applied manually)
- Interaction with injected services (via mocks)
- Error propagation handling

### Key principles

#### 1. Isolation via Dependency Injection

NestJS controllers depend on providers (services). In unit tests, you replace these with **mock providers** using `useValue` or `useFactory`.

#### 2. Testing module boundary

You use `Test.createTestingModule()` to instantiate a minimal DI container, not the full app.

#### 3. Avoid HTTP server startup

Unit tests should NOT use `INestApplication.listen()` or `supertest`. That’s e2e testing.

#### 4. Focus on behavior, not framework

You test:

- method output
- service calls
- exception behavior

---

## Example

```ts
// user.controller.ts
import { Controller, Get, Param } from "@nestjs/common";
import { UserService } from "./user.service";

@Controller("users")
export class UserController {
  constructor(private readonly userService: UserService) {}

  @Get(":id")
  async getUser(@Param("id") id: string) {
    return this.userService.findById(id);
  }
}
```

### Unit test (Jest)

```ts
import { Test, TestingModule } from "@nestjs/testing";
import { UserController } from "./user.controller";
import { UserService } from "./user.service";

describe("UserController", () => {
  let controller: UserController;
  let service: jest.Mocked<UserService>;

  const mockUserService = {
    findById: jest.fn(),
  };

  beforeEach(async () => {
    const module: TestingModule = await Test.createTestingModule({
      controllers: [UserController],
      providers: [
        {
          provide: UserService,
          useValue: mockUserService,
        },
      ],
    }).compile();

    controller = module.get<UserController>(UserController);
    service = module.get(UserService);
  });

  afterEach(() => {
    jest.clearAllMocks();
  });

  it("should return a user by id", async () => {
    const result = { id: "1", name: "John" };
    service.findById.mockResolvedValue(result);

    expect(await controller.getUser("1")).toEqual(result);
    expect(service.findById).toHaveBeenCalledWith("1");
  });

  it("should propagate service errors", async () => {
    service.findById.mockRejectedValue(new Error("Not found"));

    await expect(controller.getUser("1")).rejects.toThrow("Not found");
  });
});
```

---

## Pitfalls

- ❌ Over-testing controllers (business logic should live in services)
- ❌ Using real databases in unit tests (turns them into integration tests)
- ❌ Not resetting mocks between tests → flaky test behavior
- ❌ Testing NestJS framework behavior instead of your code logic
- ❌ Forgetting to mock all provider dependencies → DI resolution errors

## Question 2. How do you write **unit tests for services**?

## Question 3. How do you **mock repositories** in unit tests?

## Question 4. How do you write **integration tests** in NestJS?

## Question 5. How do you test **asynchronous methods**?

## Question 6. How do you test **guards** in NestJS?

## Question 7. How do you test **pipes**?

## Question 8. How do you test **interceptors**?

## Question 9. How do you test **exception filters**?

## Question 10. How do you test **microservice message handlers**?

## Question 11. How do you use **Supertest for e2e tests**?

## Question 12. How do you handle **database state in tests**?

## Question 13. How do you implement **mocking for HTTP requests**?

## Question 14. How do you test **GraphQL resolvers**?

## Question 15. How do you test **WebSocket gateways**?

## Question 16. How do you implement **dependency injection in tests**?

## Question 17. How do you test **request-scoped providers**?

## Question 18. How do you test **dynamic modules**?

## Question 19. How do you test **caching and Redis integration**?

## Question 20. How do you **debug asynchronous errors** in NestJS?
