# Set 14

| S.No. | Question                                                                                                                                 |
| ----- | ---------------------------------------------------------------------------------------------------------------------------------------- |
| 1.    | [How do you implement **multi-role JWT access tokens**?](#question-1-how-do-you-implement-multi-role-jwt-access-tokens)                  |
| 2.    | [How do you implement **dynamic permissions per user**?](#question-2-how-do-you-implement-dynamic-permissions-per-user)                  |
| 3.    | [How do you implement **JWT token revocation**?](#question-3-how-do-you-implement-jwt-token-revocation)                                  |
| 4.    | [How do you implement **OAuth2 with multiple providers**?](#question-4-how-do-you-implement-oauth2-with-multiple-providers)              |
| 5.    | [How do you implement **single sign-on (SSO)**?](#question-5-how-do-you-implement-single-sign-on-sso)                                    |
| 6.    | [How do you implement **encrypted JWT payloads**?](#question-6-how-do-you-implement-encrypted-jwt-payloads)                              |
| 7.    | [How do you implement **secure cookie-based authentication**?](#question-7-how-do-you-implement-secure-cookie-based-authentication)      |
| 8.    | [How do you implement **JWT token rotation**?](#question-8-how-do-you-implement-jwt-token-rotation)                                      |
| 9.    | [How do you implement **GraphQL field-level authorization**?](#question-9-how-do-you-implement-graphql-field-level-authorization)        |
| 10.   | [How do you implement **attribute-based access control (ABAC)**?](#question-10-how-do-you-implement-attribute-based-access-control-abac) |
| 11.   | [How do you implement **rate-limited login attempts**?](#question-11-how-do-you-implement-rate-limited-login-attempts)                   |
| 12.   | [How do you implement **2FA with TOTP**?](#question-12-how-do-you-implement-2fa-with-totp)                                               |
| 13.   | [How do you implement **refresh token invalidation**?](#question-13-how-do-you-implement-refresh-token-invalidation)                     |
| 14.   | [How do you implement **token blacklist storage in Redis**?](#question-14-how-do-you-implement-token-blacklist-storage-in-redis)         |
| 15.   | [How do you implement **passwordless login**?](#question-15-how-do-you-implement-passwordless-login)                                     |
| 16.   | [How do you implement **session management in microservices**?](#question-16-how-do-you-implement-session-management-in-microservices)   |
| 17.   | [How do you implement **IP-based access restrictions**?](#question-17-how-do-you-implement-ip-based-access-restrictions)                 |
| 18.   | [How do you implement **time-limited access** for resources?](#question-18-how-do-you-implement-time-limited-access-for-resources)       |
| 19.   | [How do you implement **user-specific data masking**?](#question-19-how-do-you-implement-user-specific-data-masking)                     |
| 20.   | [How do you implement **secure GraphQL subscriptions**?](#question-20-how-do-you-implement-secure-graphql-subscriptions)                 |

## Question 1. How do you implement **multi-role JWT access tokens**?

## Short answer

Multi-role JWT access tokens are implemented by embedding an array of roles (e.g., `roles: string[]`) inside the JWT payload, then enforcing authorization using a custom NestJS `RolesGuard` that checks whether the user’s roles intersect with the required roles declared via a custom `@Roles()` decorator.

---

## Explanation

In a NestJS system, multi-role JWT access control typically combines:

1. **Authentication layer (JWT Strategy)**
   - Validates token signature and extracts payload.
   - Attaches `user` object (including roles) to `request`.

2. **Authorization layer (Roles-based Guard)**
   - Reads metadata from route handlers (`@Roles()` decorator).
   - Compares required roles vs. user roles from JWT payload.

3. **Token design**
   - JWT payload contains:

     ```ts
     {
       sub: userId,
       email: string,
       roles: string[]   // key part for multi-role support
     }
     ```

### Architectural considerations

- **Scalability**: Stateless JWT allows horizontal scaling (no DB lookup required for each request unless role validation is dynamic).
- **Security trade-off**:
  - Pros: fast, no DB hit
  - Cons: role changes don’t reflect until token refresh/expiry

- **Enterprise pattern**:
  - Combine JWT roles with **policy-based access control (RBAC/ABAC)** for complex systems.

- **Testing**:
  - Unit test guards in isolation
  - E2E test protected routes with valid/invalid tokens

- **Deployment**:
  - Ensure consistent JWT secret across services (or use public/private key pairs for RS256 in microservices)

---

## Example

### 1. Roles Decorator

```ts
import { SetMetadata } from "@nestjs/common";

export const ROLES_KEY = "roles";
export const Roles = (...roles: string[]) => SetMetadata(ROLES_KEY, roles);
```

---

### 2. Roles Guard

```ts
import { CanActivate, ExecutionContext, Injectable } from "@nestjs/common";
import { Reflector } from "@nestjs/core";
import { ROLES_KEY } from "./roles.decorator";

@Injectable()
export class RolesGuard implements CanActivate {
  constructor(private reflector: Reflector) {}

  canActivate(context: ExecutionContext): boolean {
    const requiredRoles = this.reflector.getAllAndOverride<string[]>(
      ROLES_KEY,
      [context.getHandler(), context.getClass()],
    );

    if (!requiredRoles) return true;

    const request = context.switchToHttp().getRequest();
    const user = request.user;

    return requiredRoles.some((role) => user.roles?.includes(role));
  }
}
```

---

### 3. JWT Strategy (multi-role payload)

```ts
import { Injectable } from "@nestjs/common";
import { PassportStrategy } from "@nestjs/passport";
import { ExtractJwt, Strategy } from "passport-jwt";

@Injectable()
export class JwtStrategy extends PassportStrategy(Strategy) {
  constructor() {
    super({
      jwtFromRequest: ExtractJwt.fromAuthHeaderAsBearerToken(),
      secretOrKey: process.env.JWT_SECRET,
    });
  }

  async validate(payload: any) {
    return {
      userId: payload.sub,
      email: payload.email,
      roles: payload.roles, // critical for RBAC
    };
  }
}
```

---

### 4. Controller Usage

```ts
import { Controller, Get, UseGuards } from "@nestjs/common";
import { JwtAuthGuard } from "./jwt-auth.guard";
import { RolesGuard } from "./roles.guard";
import { Roles } from "./roles.decorator";

@Controller("admin")
@UseGuards(JwtAuthGuard, RolesGuard)
export class AdminController {
  @Get()
  @Roles("admin", "superadmin")
  getAdminData() {
    return { message: "Admin access granted" };
  }
}
```

---

## Pitfalls

- **Stale roles in JWT**: role updates won’t apply until token refresh/expiry
- **Overloading JWT payload**: adding too many roles/claims increases token size
- **Weak RBAC model**: only roles-based checks become insufficient for complex permissions (ABAC needed)
- **Guard ordering issues**: `JwtAuthGuard` must run before `RolesGuard`
- **Microservices inconsistency**: different services validating roles differently if not standardized

## Question 2. How do you implement **dynamic permissions per user**?

## Question 3. How do you implement **JWT token revocation**?

## Question 4. How do you implement **OAuth2 with multiple providers**?

## Question 5. How do you implement **single sign-on (SSO)**?

## Question 6. How do you implement **encrypted JWT payloads**?

## Question 7. How do you implement **secure cookie-based authentication**?

## Question 8. How do you implement **JWT token rotation**?

## Question 9. How do you implement **GraphQL field-level authorization**?

## Question 10. How do you implement **attribute-based access control (ABAC)**?

## Question 11. How do you implement **rate-limited login attempts**?

## Question 12. How do you implement **2FA with TOTP**?

## Question 13. How do you implement **refresh token invalidation**?

## Question 14. How do you implement **token blacklist storage in Redis**?

## Question 15. How do you implement **passwordless login**?

## Question 16. How do you implement **session management in microservices**?

## Question 17. How do you implement **IP-based access restrictions**?

## Question 18. How do you implement **time-limited access** for resources?

## Question 19. How do you implement **user-specific data masking**?

## Question 20. How do you implement **secure GraphQL subscriptions**?
