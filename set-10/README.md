# Set 10

| S.No. | Question                                                                                                                                                     |
| ----- | ------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| 1.    | [How do you implement **JWT refresh token rotation**?](#question-1-how-do-you-implement-jwt-refresh-token-rotation)                                          |
| 2.    | [How do you implement **OAuth2 login with Google/Facebook**?](#question-2-how-do-you-implement-oauth2-login-with-googlefacebook)                             |
| 3.    | [How do you implement **role hierarchies**?](#question-3-how-do-you-implement-role-hierarchies)                                                              |
| 4.    | [How do you implement **JWT blacklisting**?](#question-4-how-do-you-implement-jwt-blacklisting)                                                              |
| 5.    | [How do you implement **multi-tenant authentication**?](#question-5-how-do-you-implement-multi-tenant-authentication)                                        |
| 6.    | [How do you store **JWT securely in client apps**?](#question-6-how-do-you-store-jwt-securely-in-client-apps)                                                |
| 7.    | [How do you validate **JWT tokens manually**?](#question-7-how-do-you-validate-jwt-tokens-manually)                                                          |
| 8.    | [How do you handle **token expiration**?](#question-8-how-do-you-handle-token-expiration)                                                                    |
| 9.    | [How do you protect **GraphQL endpoints**?](#question-9-how-do-you-protect-graphql-endpoints)                                                                |
| 10.   | [How do you implement **two-factor authentication**?](#question-10-how-do-you-implement-two-factor-authentication)                                           |
| 11.   | [How do you implement **Redis transport** in NestJS microservices?](#question-11-how-do-you-implement-redis-transport-in-nestjs-microservices)               |
| 12.   | [How do you implement **NATS transport**?](#question-12-how-do-you-implement-nats-transport)                                                                 |
| 13.   | [How do you handle **message acknowledgment** in microservices?](#question-13-how-do-you-handle-message-acknowledgment-in-microservices)                     |
| 14.   | [How do you implement **event-driven communication**?](#question-14-how-do-you-implement-event-driven-communication)                                         |
| 15.   | [How do you implement **Kafka transport** in NestJS?](#question-15-how-do-you-implement-kafka-transport-in-nestjs)                                           |
| 16.   | [How do you implement **custom serialization** for microservice messages?](#question-16-how-do-you-implement-custom-serialization-for-microservice-messages) |
| 17.   | [How do you implement **gRPC microservices**?](#question-17-how-do-you-implement-grpc-microservices)                                                         |
| 18.   | [How do you implement **WebSocket gateways** with NestJS?](#question-18-how-do-you-implement-websocket-gateways-with-nestjs)                                 |
| 19.   | [How do you implement **GraphQL subscriptions**?](#question-19-how-do-you-implement-graphql-subscriptions)                                                   |
| 20.   | [How do you implement **CQRS with Event Sourcing** in NestJS?](#question-20-how-do-you-implement-cqrs-with-event-sourcing-in-nestjs)                         |

## Question 1. How do you implement **JWT refresh token rotation**?

## Short answer

JWT refresh token rotation is implemented by issuing a new refresh token every time one is used, invalidating the old one (server-side tracking), and detecting reuse to prevent token replay attacks.

---

## Explanation

Refresh token rotation is a **security pattern for long-lived authentication sessions** where:

- Access token: short-lived (e.g., 10–15 min)
- Refresh token: long-lived (days/weeks)
- Every refresh request:
  1. Validates refresh token
  2. Issues **new access token + new refresh token**
  3. Invalidates the old refresh token

### Why rotation matters (security model)

Without rotation:

- If a refresh token is stolen → attacker can reuse it indefinitely

With rotation:

- Each refresh token is **single-use**
- If a stolen token is reused → system detects reuse → can revoke entire session family

### Core architecture in NestJS

You typically combine:

- **JWT strategy (access token)**
- **Persistent store for refresh tokens** (Redis or DB)
- **Token family tracking**
- **Hashing refresh tokens (never store raw tokens)**

#### Key design choices

| Concern          | Approach                                          |
| ---------------- | ------------------------------------------------- |
| Storage          | Redis (fast TTL) or DB (relational auditability)  |
| Token revocation | Store token hash + session id                     |
| Rotation safety  | Detect reuse via token version / family id        |
| Scalability      | Stateless access tokens + stateful refresh tokens |

---

## Example (NestJS implementation)

### Auth module with refresh rotation

```ts
// auth.module.ts
import { Module } from "@nestjs/common";
import { JwtModule } from "@nestjs/jwt";
import { AuthService } from "./auth.service";
import { AuthController } from "./auth.controller";
import { RefreshTokenStore } from "./refresh-token.store";

@Module({
  imports: [
    JwtModule.register({
      secret: process.env.JWT_SECRET!,
      signOptions: { expiresIn: "15m" },
    }),
  ],
  providers: [AuthService, RefreshTokenStore],
  controllers: [AuthController],
})
export class AuthModule {}
```

---

### Refresh token store (Redis-like abstraction)

```ts
import { Injectable } from "@nestjs/common";
import * as crypto from "crypto";

@Injectable()
export class RefreshTokenStore {
  // simulate Redis: sessionId -> tokenHash
  private store = new Map<string, string>();

  hash(token: string) {
    return crypto.createHash("sha256").update(token).digest("hex");
  }

  save(sessionId: string, token: string) {
    this.store.set(sessionId, this.hash(token));
  }

  validate(sessionId: string, token: string): boolean {
    const stored = this.store.get(sessionId);
    return stored === this.hash(token);
  }

  revoke(sessionId: string) {
    this.store.delete(sessionId);
  }
}
```

---

### Auth service with rotation logic

```ts
import { Injectable, UnauthorizedException } from "@nestjs/common";
import { JwtService } from "@nestjs/jwt";
import { RefreshTokenStore } from "./refresh-token.store";
import { randomUUID } from "crypto";

@Injectable()
export class AuthService {
  constructor(
    private jwt: JwtService,
    private store: RefreshTokenStore,
  ) {}

  async login(userId: string) {
    const sessionId = randomUUID();

    const accessToken = this.jwt.sign({ sub: userId });
    const refreshToken = this.jwt.sign(
      { sub: userId, sid: sessionId },
      { expiresIn: "7d" },
    );

    this.store.save(sessionId, refreshToken);

    return { accessToken, refreshToken };
  }

  async refresh(refreshToken: string) {
    let payload: any;

    try {
      payload = this.jwt.verify(refreshToken);
    } catch {
      throw new UnauthorizedException("Invalid refresh token");
    }

    const { sub, sid } = payload;

    // detect reuse / tampering
    const valid = this.store.validate(sid, refreshToken);
    if (!valid) {
      // token reuse detected → revoke session
      this.store.revoke(sid);
      throw new UnauthorizedException("Refresh token reuse detected");
    }

    // rotate: invalidate old token first
    this.store.revoke(sid);

    const newSessionId = randomUUID();

    const newAccessToken = this.jwt.sign({ sub });
    const newRefreshToken = this.jwt.sign(
      { sub, sid: newSessionId },
      { expiresIn: "7d" },
    );

    this.store.save(newSessionId, newRefreshToken);

    return {
      accessToken: newAccessToken,
      refreshToken: newRefreshToken,
    };
  }
}
```

---

### Controller

```ts
import { Controller, Post, Body } from "@nestjs/common";
import { AuthService } from "./auth.service";

@Controller("auth")
export class AuthController {
  constructor(private auth: AuthService) {}

  @Post("login")
  login(@Body() body: { userId: string }) {
    return this.auth.login(body.userId);
  }

  @Post("refresh")
  refresh(@Body() body: { refreshToken: string }) {
    return this.auth.refresh(body.refreshToken);
  }
}
```

---

## Pitfalls

- Storing refresh tokens in JWT alone (no server-side state) → rotation becomes ineffective
- Not hashing refresh tokens → DB leakage = session takeover
- Missing reuse detection → attacker can replay stolen token silently
- Not handling multi-device sessions (session ID per device required)
- Race conditions in distributed systems without atomic Redis operations

## Question 2. How do you implement **OAuth2 login with Google/Facebook**?

## Question 3. How do you implement **role hierarchies**?

## Question 4. How do you implement **JWT blacklisting**?

## Question 5. How do you implement **multi-tenant authentication**?

## Question 6. How do you store **JWT securely in client apps**?

## Question 7. How do you validate **JWT tokens manually**?

## Question 8. How do you handle **token expiration**?

## Question 9. How do you protect **GraphQL endpoints**?

## Question 10. How do you implement **two-factor authentication**?

## Question 11. How do you implement **Redis transport** in NestJS microservices?

## Question 12. How do you implement **NATS transport**?

## Question 13. How do you handle **message acknowledgment** in microservices?

## Question 14. How do you implement **event-driven communication**?

## Question 15. How do you implement **Kafka transport** in NestJS?

## Question 16. How do you implement **custom serialization** for microservice messages?

## Question 17. How do you implement **gRPC microservices**?

## Question 18. How do you implement **WebSocket gateways** with NestJS?

## Question 19. How do you implement **GraphQL subscriptions**?

## Question 20. How do you implement **CQRS with Event Sourcing** in NestJS?
