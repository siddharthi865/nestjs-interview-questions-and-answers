# Set 24

| S.No. | Question                                                                                                                                                   |
| ----- | ---------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1.    | [How do you implement **JWT token revocation lists**?](#question-1-how-do-you-implement-jwt-token-revocation-lists)                                        |
| 2.    | [How do you implement **role-based dynamic permissions**?](#question-2-how-do-you-implement-role-based-dynamic-permissions)                                |
| 3.    | [How do you implement **time-limited access tokens**?](#question-3-how-do-you-implement-time-limited-access-tokens)                                        |
| 4.    | [How do you implement **token refresh strategies in clustered apps**?](#question-4-how-do-you-implement-token-refresh-strategies-in-clustered-apps)        |
| 5.    | [How do you implement **secure cookie-based authentication**?](#question-5-how-do-you-implement-secure-cookie-based-authentication)                        |
| 6.    | [How do you implement **GraphQL field-level security**?](#question-6-how-do-you-implement-graphql-field-level-security)                                    |
| 7.    | [How do you implement **OAuth2 with multiple providers simultaneously**?](#question-7-how-do-you-implement-oauth2-with-multiple-providers-simultaneously)  |
| 8.    | [How do you implement **multi-factor authentication**?](#question-8-how-do-you-implement-multi-factor-authentication)                                      |
| 9.    | [How do you implement **API key rotation**?](#question-9-how-do-you-implement-api-key-rotation)                                                            |
| 10.   | [How do you implement **rate-limited login endpoints**?](#question-10-how-do-you-implement-rate-limited-login-endpoints)                                   |
| 11.   | [How do you implement **IP-based access restrictions**?](#question-11-how-do-you-implement-ip-based-access-restrictions)                                   |
| 12.   | [How do you implement **dynamic permission-based data masking**?](#question-12-how-do-you-implement-dynamic-permission-based-data-masking)                 |
| 13.   | [How do you implement **passwordless login with email links**?](#question-13-how-do-you-implement-passwordless-login-with-email-links)                     |
| 14.   | [How do you implement **JWT encryption**?](#question-14-how-do-you-implement-jwt-encryption)                                                               |
| 15.   | [How do you implement **secure WebSocket connections**?](#question-15-how-do-you-implement-secure-websocket-connections)                                   |
| 16.   | [How do you implement **token expiration warnings**?](#question-16-how-do-you-implement-token-expiration-warnings)                                         |
| 17.   | [How do you implement **secure refresh token storage**?](#question-17-how-do-you-implement-secure-refresh-token-storage)                                   |
| 18.   | [How do you implement **attribute-based access control**?](#question-18-how-do-you-implement-attribute-based-access-control)                               |
| 19.   | [How do you implement **abnormal behavior detection** in authentication?](#question-19-how-do-you-implement-abnormal-behavior-detection-in-authentication) |
| 20.   | [How do you implement **authorization logging**?](#question-20-how-do-you-implement-authorization-logging)                                                 |

## Question 1. How do you implement **JWT token revocation lists**?

## Short answer

JWT token revocation is implemented by maintaining a **server-side blacklist (revocation list)** of token identifiers (usually `jti`) or by tracking token validity using a **stateful store (Redis/database)** and checking it on every request in a guard.

---

## Explanation

### 1. Why JWT revocation is non-trivial

JWTs are **stateless by design**, meaning once issued, they are valid until expiration. This creates a challenge when:

- A user logs out
- A token is compromised
- Permissions are revoked immediately

To solve this, we introduce **state back into an otherwise stateless system**.

---

### 2. Common strategies for revocation

#### A. Blacklist (token revocation list)

- Each JWT includes a unique identifier (`jti`)
- On logout or revoke event, store `jti` in Redis (or DB)
- On every request, check if `jti` is revoked

**Trade-offs:**

- ✔ Immediate revocation
- ❌ Extra network call per request (mitigated with Redis)
- ❌ Storage grows with active revoked tokens

---

#### B. Token versioning (scalable alternative)

- Store `tokenVersion` in user table
- Include it in JWT payload
- If version mismatch → reject token

**Trade-offs:**

- ✔ No per-token storage
- ✔ Easy global logout
- ❌ Cannot revoke single token (only all user tokens)

---

#### C. Short-lived access + refresh token rotation (best practice)

- Access token: 5–15 min expiry
- Refresh token: stored server-side and rotated
- Revocation happens at refresh layer

**Trade-offs:**

- ✔ Highly secure
- ✔ Minimal blacklist usage
- ❌ More complex system

---

### 3. Recommended enterprise approach

In production NestJS systems:

- Use **Redis-backed blacklist for access tokens**
- Combine with **refresh token rotation**
- Use `jti` for precise invalidation

---

## Example

### JWT payload with `jti`

```ts
// auth/jwt-payload.ts
export interface JwtPayload {
  sub: string;
  email: string;
  jti: string; // unique token id
}
```

---

### Redis-based revocation service

```ts
import { Injectable } from "@nestjs/common";
import { InjectRedis } from "@nestjs-modules/ioredis";
import { Redis } from "ioredis";

@Injectable()
export class TokenBlacklistService {
  constructor(@InjectRedis() private readonly redis: Redis) {}

  async revoke(jti: string, exp: number) {
    const ttl = exp - Math.floor(Date.now() / 1000);
    if (ttl > 0) {
      await this.redis.set(`blacklist:${jti}`, "1", "EX", ttl);
    }
  }

  async isRevoked(jti: string): Promise<boolean> {
    const result = await this.redis.get(`blacklist:${jti}`);
    return result === "1";
  }
}
```

---

### JWT Guard enforcing revocation

```ts
import {
  CanActivate,
  ExecutionContext,
  Injectable,
  UnauthorizedException,
} from "@nestjs/common";
import { JwtService } from "@nestjs/jwt";
import { Request } from "express";
import { TokenBlacklistService } from "./token-blacklist.service";

@Injectable()
export class JwtAuthGuard implements CanActivate {
  constructor(
    private readonly jwtService: JwtService,
    private readonly blacklist: TokenBlacklistService,
  ) {}

  async canActivate(context: ExecutionContext): Promise<boolean> {
    const req = context.switchToHttp().getRequest<Request>();
    const token = req.headers.authorization?.split(" ")[1];

    if (!token) throw new UnauthorizedException();

    const payload = await this.jwtService.verifyAsync(token);

    if (await this.blacklist.isRevoked(payload.jti)) {
      throw new UnauthorizedException("Token revoked");
    }

    req.user = payload;
    return true;
  }
}
```

---

### Revocation on logout

```ts
import { Controller, Post, Req } from "@nestjs/common";
import { Request } from "express";
import { TokenBlacklistService } from "./token-blacklist.service";

@Controller("auth")
export class AuthController {
  constructor(private readonly blacklist: TokenBlacklistService) {}

  @Post("logout")
  async logout(@Req() req: Request) {
    const user = req.user as any;

    await this.blacklist.revoke(user.jti, user.exp);
    return { message: "Logged out successfully" };
  }
}
```

---

## Pitfalls

- ⚠ **Performance overhead**: DB lookup per request if not using Redis
- ⚠ **Missing TTL handling** → blacklist grows indefinitely
- ⚠ **Clock skew issues** when calculating expiration time
- ⚠ **Not including `jti` in tokens** makes per-token revocation impossible
- ⚠ **Multi-region deployments** require distributed cache consistency

## Question 2. How do you implement **role-based dynamic permissions**?

## Question 3. How do you implement **time-limited access tokens**?

## Question 4. How do you implement **token refresh strategies in clustered apps**?

## Question 5. How do you implement **secure cookie-based authentication**?

## Question 6. How do you implement **GraphQL field-level security**?

## Question 7. How do you implement **OAuth2 with multiple providers simultaneously**?

## Question 8. How do you implement **multi-factor authentication**?

## Question 9. How do you implement **API key rotation**?

## Question 10. How do you implement **rate-limited login endpoints**?

## Question 11. How do you implement **IP-based access restrictions**?

## Question 12. How do you implement **dynamic permission-based data masking**?

## Question 13. How do you implement **passwordless login with email links**?

## Question 14. How do you implement **JWT encryption**?

## Question 15. How do you implement **secure WebSocket connections**?

## Question 16. How do you implement **token expiration warnings**?

## Question 17. How do you implement **secure refresh token storage**?

## Question 18. How do you implement **attribute-based access control**?

## Question 19. How do you implement **abnormal behavior detection** in authentication?

## Question 20. How do you implement **authorization logging**?
