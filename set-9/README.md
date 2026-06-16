# Set 9

| S.No. | Question                                                                                                                                    |
| ----- | ------------------------------------------------------------------------------------------------------------------------------------------- |
| 1.    | [How do you integrate **Sequelize** with NestJS?](#question-1-how-do-you-integrate-sequelize-with-nestjs)                                   |
| 2.    | [How do you define **associations in TypeORM**?](#question-2-how-do-you-define-associations-in-typeorm)                                     |
| 3.    | [How do you implement **one-to-many and many-to-many relations**?](#question-3-how-do-you-implement-one-to-many-and-many-to-many-relations) |
| 4.    | [How do you use **query builders** in TypeORM?](#question-4-how-do-you-use-query-builders-in-typeorm)                                       |
| 5.    | [How do you implement **soft deletion** in Prisma?](#question-5-how-do-you-implement-soft-deletion-in-prisma)                               |
| 6.    | [How do you implement **transactions with TypeORM**?](#question-6-how-do-you-implement-transactions-with-typeorm)                           |
| 7.    | [How do you implement **lazy loading vs eager loading**?](#question-7-how-do-you-implement-lazy-loading-vs-eager-loading)                   |
| 8.    | [How do you handle **database migrations** in NestJS?](#question-8-how-do-you-handle-database-migrations-in-nestjs)                         |
| 9.    | [How do you implement **caching queries** in database services?](#question-9-how-do-you-implement-caching-queries-in-database-services)     |
| 10.   | [How do you validate **entities before saving**?](#question-10-how-do-you-validate-entities-before-saving)                                  |
| 11.   | [How do you use **repository patterns** with custom methods?](#question-11-how-do-you-use-repository-patterns-with-custom-methods)          |
| 12.   | [How do you implement **composite primary keys**?](#question-12-how-do-you-implement-composite-primary-keys)                                |
| 13.   | [How do you implement **auditing fields** (createdAt, updatedAt)?](#question-13-how-do-you-implement-auditing-fields-createdat-updatedat)   |
| 14.   | [How do you implement **soft vs hard deletes**?](#question-14-how-do-you-implement-soft-vs-hard-deletes)                                    |
| 15.   | [How do you optimize **bulk inserts and updates**?](#question-15-how-do-you-optimize-bulk-inserts-and-updates)                              |
| 16.   | [How do you implement **database connection pooling**?](#question-16-how-do-you-implement-database-connection-pooling)                      |
| 17.   | [How do you handle **multi-database connections**?](#question-17-how-do-you-handle-multi-database-connections)                              |
| 18.   | [How do you implement **database transactions across services**?](#question-18-how-do-you-implement-database-transactions-across-services)  |
| 19.   | [How do you handle **query performance bottlenecks**?](#question-19-how-do-you-handle-query-performance-bottlenecks)                        |
| 20.   | [How do you implement **data seeding** for testing?](#question-20-how-do-you-implement-data-seeding-for-testing)                            |

## Question 1. How do you integrate **Sequelize** with NestJS?

## Short answer

Sequelize is integrated in NestJS using the `@nestjs/sequelize` package, which wraps Sequelize as a Nest provider system. You register the Sequelize module in your root module using `SequelizeModule.forRoot()` and define models using `@nestjs/sequelize` decorators. Models are then injected via `@InjectModel()` in services.

---

## Explanation

### 1. Architecture overview

NestJS integrates Sequelize through a **module-based abstraction layer**:

- `SequelizeModule` → bootstraps DB connection (singleton at app level)
- Models → registered per feature module
- Dependency Injection → models injected into services via tokens
- Lifecycle → connection initialized during `onModuleInit`

This aligns with Nest’s **modular + DI-first architecture**, keeping ORM concerns isolated from business logic.

---

### 2. Key integration steps

#### Step 1: Install dependencies

```bash
npm install @nestjs/sequelize sequelize sequelize-typescript mysql2
```

(Use `pg` instead of `mysql2` if using PostgreSQL)

---

#### Step 2: Configure Sequelize in root module

- Use `forRoot` for global DB config
- Enable auto-load models or explicitly register them

---

#### Step 3: Define models with decorators

Use `sequelize-typescript` decorators like `@Table`, `@Column`.

---

#### Step 4: Inject models in services

Use `@InjectModel(User)` for repository-like access.

---

### 3. Scalability & trade-offs

#### Pros

- Tight NestJS DI integration
- Strong TypeScript support via `sequelize-typescript`
- Good for monolithic or modular monorepo apps
- Mature SQL ORM with migrations support

#### Cons / trade-offs

- Heavier runtime overhead than raw SQL or Prisma (in some cases)
- Metadata decorator system can become complex in large domains
- Less compile-time safety than Prisma
- Query performance tuning requires Sequelize-specific knowledge

---

### 4. Testing strategy

- Use `getModelToken()` for mocking models
- Replace Sequelize connection with in-memory SQLite for integration tests
- Prefer repository abstraction over direct model usage in services for easier mocking

---

### 5. Security considerations

- Always use parameterized queries (Sequelize does this by default)
- Validate DTOs before persistence (Nest `ValidationPipe`)
- Avoid exposing Sequelize models directly in controllers
- Enable logging carefully in production (SQL leak risk)

---

## Example

### `user.model.ts`

```ts
import { Table, Column, Model, DataType } from "sequelize-typescript";

@Table({ tableName: "users" })
export class User extends Model<User> {
  @Column({
    type: DataType.INTEGER,
    autoIncrement: true,
    primaryKey: true,
  })
  id!: number;

  @Column({
    type: DataType.STRING,
    allowNull: false,
  })
  email!: string;

  @Column({
    type: DataType.STRING,
  })
  password!: string;
}
```

---

### `app.module.ts`

```ts
import { Module } from "@nestjs/common";
import { SequelizeModule } from "@nestjs/sequelize";
import { User } from "./user.model";
import { UserModule } from "./user.module";

@Module({
  imports: [
    SequelizeModule.forRoot({
      dialect: "mysql",
      host: "localhost",
      port: 3306,
      username: "root",
      password: "password",
      database: "test_db",
      models: [User],
      autoLoadModels: true,
      synchronize: true, // ❗ disable in production
      logging: false,
    }),
    UserModule,
  ],
})
export class AppModule {}
```

---

### `user.service.ts`

```ts
import { Injectable } from "@nestjs/common";
import { InjectModel } from "@nestjs/sequelize";
import { User } from "./user.model";

@Injectable()
export class UserService {
  constructor(
    @InjectModel(User)
    private readonly userModel: typeof User,
  ) {}

  async createUser(email: string, password: string) {
    return this.userModel.create({ email, password });
  }

  async findAll() {
    return this.userModel.findAll();
  }
}
```

---

### `user.module.ts`

```ts
import { Module } from "@nestjs/common";
import { SequelizeModule } from "@nestjs/sequelize";
import { UserService } from "./user.service";
import { User } from "./user.model";

@Module({
  imports: [SequelizeModule.forFeature([User])],
  providers: [UserService],
})
export class UserModule {}
```

---

## Pitfalls

- ❌ Using `synchronize: true` in production (can drop/alter schema unintentionally)
- ❌ Injecting models directly into controllers instead of services (breaks layering)
- ❌ Not separating domain logic from ORM models (tight coupling)
- ❌ Missing transaction handling for multi-step operations
- ❌ Overusing eager loading (`include`) leading to N+1 or heavy joins

## Question 2. How do you define **associations in TypeORM**?

## Question 3. How do you implement **one-to-many and many-to-many relations**?

## Question 4. How do you use **query builders** in TypeORM?

## Question 5. How do you implement **soft deletion** in Prisma?

## Question 6. How do you implement **transactions with TypeORM**?

## Question 7. How do you implement **lazy loading vs eager loading**?

## Question 8. How do you handle **database migrations** in NestJS?

## Question 9. How do you implement **caching queries** in database services?

## Question 10. How do you validate **entities before saving**?

## Question 11. How do you use **repository patterns** with custom methods?

## Question 12. How do you implement **composite primary keys**?

## Question 13. How do you implement **auditing fields** (createdAt, updatedAt)?

## Question 14. How do you implement **soft vs hard deletes**?

## Question 15. How do you optimize **bulk inserts and updates**?

## Question 16. How do you implement **database connection pooling**?

## Question 17. How do you handle **multi-database connections**?

## Question 18. How do you implement **database transactions across services**?

## Question 19. How do you handle **query performance bottlenecks**?

## Question 20. How do you implement **data seeding** for testing?
