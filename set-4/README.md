# Set 4

| S.No. | Question                                                                                                                                                                         |
| ----- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1.    | [How do you implement JWT authentication in NestJS?](#question-1-how-do-you-implement-jwt-authentication-in-nestjs)                                                              |
| 2.    | [How do you implement Passport.js strategies in NestJS?](#question-2-how-do-you-implement-passportjs-strategies-in-nestjs)                                                       |
| 3.    | [What is the difference between **local, JWT, and OAuth strategies**?](#question-3-what-is-the-difference-between-local-jwt-and-oauth-strategies)                                |
| 4.    | [How do you secure routes with guards?](#question-4-how-do-you-secure-routes-with-guards)                                                                                        |
| 5.    | [How do you protect endpoints with role-based access?](#question-5-how-do-you-protect-endpoints-with-role-based-access)                                                          |
| 6.    | [How do you implement refresh tokens in NestJS?](#question-6-how-do-you-implement-refresh-tokens-in-nestjs)                                                                      |
| 7.    | [How do you hash passwords securely?](#question-7-how-do-you-hash-passwords-securely)                                                                                            |
| 8.    | [How do you handle user sessions?](#question-8-how-do-you-handle-user-sessions)                                                                                                  |
| 9.    | [How do you implement API key authentication?](#question-9-how-do-you-implement-api-key-authentication)                                                                          |
| 10.   | [How do you secure sensitive data in DTOs?](#question-10-how-do-you-secure-sensitive-data-in-dtos)                                                                               |
| 11.   | [How do you implement **microservices** in NestJS?](#question-11-how-do-you-implement-microservices-in-nestjs)                                                                   |
| 12.   | [What transport layers are supported by NestJS microservices?](#question-12-what-transport-layers-are-supported-by-nestjs-microservices)                                         |
| 13.   | [How do you implement **event-based architecture**?](#question-13-how-do-you-implement-event-based-architecture)                                                                 |
| 14.   | [Explain **message patterns** in microservices](#question-14-explain-message-patterns-in-microservices)                                                                          |
| 15.   | [How do you use **WebSockets** with NestJS?](#question-15-how-do-you-use-websockets-with-nestjs)                                                                                 |
| 16.   | [How do you implement **GraphQL** in NestJS?](#question-16-how-do-you-implement-graphql-in-nestjs)                                                                               |
| 17.   | [Explain **code-first** vs **schema-first** approaches in GraphQL](#question-17-explain-code-first-vs-schema-first-approaches-in-graphql)                                        |
| 18.   | [How do you implement **custom decorators**?](#question-18-how-do-you-implement-custom-decorators)                                                                               |
| 19.   | [How do you implement **dynamic modules**?](#question-19-how-do-you-implement-dynamic-modules)                                                                                   |
| 20.   | [How do you implement **CQRS (Command Query Responsibility Segregation)** in NestJS?](#question-20-how-do-you-implement-cqrs-command-query-responsibility-segregation-in-nestjs) |

## Question 1. How do you implement JWT authentication in NestJS?

## Short answer

JWT authentication in NestJS is implemented using the `@nestjs/passport` integration with a JWT strategy (`passport-jwt`), where a login endpoint issues a signed token and a guard (`AuthGuard('jwt')`) validates it on protected routes.

---

## Explanation

### 1. High-level architecture

JWT auth in NestJS typically follows this flow:

1. **User logs in** → credentials validated via `AuthService`
2. **JWT is signed** using `JwtService` (`@nestjs/jwt`)
3. Client stores token (usually in memory or secure storage)
4. Client sends token in `Authorization: Bearer <token>`
5. **JWT Strategy validates token** on each request
6. `AuthGuard('jwt')` protects routes

### Key NestJS components involved:

- **AuthModule**: orchestrates everything
- **AuthService**: validates user + issues tokens
- **JwtStrategy**: validates and decodes JWT
- **JwtAuthGuard**: protects endpoints
- **Passport integration**: standard auth abstraction layer

---

### 2. Security, scalability, and design considerations

#### Security

- Always sign tokens with a strong secret or asymmetric keys (RS256 preferred in enterprise systems)
- Use short-lived access tokens + refresh token rotation
- Never store sensitive data in JWT payload (only claims like `sub`, `roles`)
- Consider revocation strategy (token blacklist or versioning)

#### Scalability

- JWT is stateless → works well in horizontally scaled microservices
- Avoid DB lookup per request unless you enforce revocation or role sync
- Combine with Redis if you need session invalidation

#### Testing

- Unit test `AuthService.validateUser`
- E2E test guarded routes using `supertest`
- Mock `JwtService` for isolation

#### Deployment

- Store secrets in environment variables or secret managers (AWS Secrets Manager, Vault)
- Rotate keys periodically
- Use HTTPS always (JWT is not encrypted by default)

---

## Example

### Install dependencies

```bash
npm install @nestjs/jwt @nestjs/passport passport passport-jwt bcrypt
npm install -D @types/passport-jwt
```

---

### auth.module.ts

```ts
import { Module } from "@nestjs/common";
import { JwtModule } from "@nestjs/jwt";
import { PassportModule } from "@nestjs/passport";
import { AuthService } from "./auth.service";
import { JwtStrategy } from "./jwt.strategy";

@Module({
  imports: [
    PassportModule,
    JwtModule.register({
      secret: process.env.JWT_SECRET,
      signOptions: { expiresIn: "15m" },
    }),
  ],
  providers: [AuthService, JwtStrategy],
  exports: [AuthService],
})
export class AuthModule {}
```

---

### auth.service.ts

```ts
import { Injectable, UnauthorizedException } from "@nestjs/common";
import { JwtService } from "@nestjs/jwt";
import * as bcrypt from "bcrypt";

@Injectable()
export class AuthService {
  constructor(private readonly jwtService: JwtService) {}

  async validateUser(username: string, password: string) {
    const user = await this.findUserByUsername(username); // pretend DB call

    if (!user) throw new UnauthorizedException();

    const passwordValid = await bcrypt.compare(password, user.passwordHash);

    if (!passwordValid) throw new UnauthorizedException();

    return user;
  }

  async login(user: any) {
    const payload = {
      sub: user.id,
      username: user.username,
      roles: user.roles,
    };

    return {
      access_token: this.jwtService.sign(payload),
    };
  }

  private async findUserByUsername(username: string) {
    return {
      id: 1,
      username,
      passwordHash: await bcrypt.hash("pass", 10),
      roles: ["user"],
    };
  }
}
```

---

### jwt.strategy.ts

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
    // attaches to request.user
    return {
      userId: payload.sub,
      username: payload.username,
      roles: payload.roles,
    };
  }
}
```

---

### auth.controller.ts

```ts
import {
  Controller,
  Post,
  Body,
  UseGuards,
  Get,
  Request,
} from "@nestjs/common";
import { AuthService } from "./auth.service";
import { AuthGuard } from "@nestjs/passport";

@Controller("auth")
export class AuthController {
  constructor(private readonly authService: AuthService) {}

  @Post("login")
  async login(@Body() body: { username: string; password: string }) {
    const user = await this.authService.validateUser(
      body.username,
      body.password,
    );
    return this.authService.login(user);
  }

  @UseGuards(AuthGuard("jwt"))
  @Get("profile")
  getProfile(@Request() req) {
    return req.user;
  }
}
```

---

## Pitfalls

- Storing too much data in JWT → large headers, security risks
- No token expiration or refresh strategy → security vulnerability
- Using symmetric secrets across multiple services without rotation strategy
- Forgetting to validate `audience` and `issuer` in enterprise setups
- Not handling logout/revocation in stateless systems

## Question 2. How do you implement Passport.js strategies in NestJS?

## Question 3. What is the difference between **local, JWT, and OAuth strategies**?

## Question 4. How do you secure routes with guards?

## Question 5. How do you protect endpoints with role-based access?

## Question 6. How do you implement refresh tokens in NestJS?

## Question 7. How do you hash passwords securely?

## Question 8. How do you handle user sessions?

## Question 9. How do you implement API key authentication?

## Question 10. How do you secure sensitive data in DTOs?

## Question 11. How do you implement **microservices** in NestJS?

## Question 12. What transport layers are supported by NestJS microservices?

## Question 13. How do you implement **event-based architecture**?

## Question 14. Explain **message patterns** in microservices

## Question 15. How do you use **WebSockets** with NestJS?

## Question 16. How do you implement **GraphQL** in NestJS?

## Question 17. Explain **code-first** vs **schema-first** approaches in GraphQL

## Question 18. How do you implement **custom decorators**?

## Question 19. How do you implement **dynamic modules**?

## Question 20. How do you implement **CQRS (Command Query Responsibility Segregation)** in NestJS?
