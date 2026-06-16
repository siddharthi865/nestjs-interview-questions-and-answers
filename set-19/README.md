# Set 19

| S.No. | Question                                                                                                                                             |
| ----- | ---------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1.    | [How do you deploy NestJS on **Docker**?](#question-1-how-do-you-deploy-nestjs-on-docker)                                                            |
| 2.    | [How do you deploy NestJS on **Kubernetes**?](#question-2-how-do-you-deploy-nestjs-on-kubernetes)                                                    |
| 3.    | [How do you configure **environment variables** for production?](#question-3-how-do-you-configure-environment-variables-for-production)              |
| 4.    | [How do you implement **graceful shutdown**?](#question-4-how-do-you-implement-graceful-shutdown)                                                    |
| 5.    | [How do you implement **horizontal scaling**?](#question-5-how-do-you-implement-horizontal-scaling)                                                  |
| 6.    | [How do you implement **clustering with Node.js**?](#question-6-how-do-you-implement-clustering-with-nodejs)                                         |
| 7.    | [How do you configure **NestJS logging for production**?](#question-7-how-do-you-configure-nestjs-logging-for-production)                            |
| 8.    | [How do you implement **monitoring and alerting**?](#question-8-how-do-you-implement-monitoring-and-alerting)                                        |
| 9.    | [How do you handle **rate-limiting in production**?](#question-9-how-do-you-handle-rate-limiting-in-production)                                      |
| 10.   | [How do you implement **load balancing with NestJS apps**?](#question-10-how-do-you-implement-load-balancing-with-nestjs-apps)                       |
| 11.   | [How do you configure **NestJS with Nginx**?](#question-11-how-do-you-configure-nestjs-with-nginx)                                                   |
| 12.   | [How do you handle **session management in a clustered environment**?](#question-12-how-do-you-handle-session-management-in-a-clustered-environment) |
| 13.   | [How do you configure **caching with Redis in production**?](#question-13-how-do-you-configure-caching-with-redis-in-production)                     |
| 14.   | [How do you handle **JWT token management in production**?](#question-14-how-do-you-handle-jwt-token-management-in-production)                       |
| 15.   | [How do you configure **database connection pools in production**?](#question-15-how-do-you-configure-database-connection-pools-in-production)       |
| 16.   | [How do you implement **hot reload in production deployments**?](#question-16-how-do-you-implement-hot-reload-in-production-deployments)             |
| 17.   | [How do you implement **zero-downtime deployments**?](#question-17-how-do-you-implement-zero-downtime-deployments)                                   |
| 18.   | [How do you scale **WebSocket connections horizontally**?](#question-18-how-do-you-scale-websocket-connections-horizontally)                         |
| 19.   | [How do you implement **distributed cron jobs**?](#question-19-how-do-you-implement-distributed-cron-jobs)                                           |
| 20.   | [How do you implement **graceful restart for long-running tasks**?](#question-20-how-do-you-implement-graceful-restart-for-long-running-tasks)       |

## Question 1. How do you deploy NestJS on **Docker**?

## Short answer

NestJS is deployed on Docker by containerizing the compiled Node.js application using a multi-stage Docker build (build → production runtime image), exposing the app port (usually 3000), and running it via `node dist/main.js` in a lightweight base image like `node:alpine`.

---

## Explanation

Deploying a NestJS application in Docker is a standard enterprise approach for portability, scalability, and environment consistency.

### 1. Architecture overview

A production-grade Docker setup typically uses:

- **Multi-stage build**
  - Stage 1: Install dependencies + compile TypeScript (`nest build`)
  - Stage 2: Copy only `dist/` + production `node_modules`

- **Slim runtime image**
  - Reduces attack surface and image size

- **Environment injection**
  - Config via `ConfigModule` + `.env` or Kubernetes secrets

- **Health checks (optional)**
  - Used in orchestration systems (Kubernetes / ECS)

---

### 2. Trade-offs

- ✔ Pros:
  - Reproducible builds across environments
  - Easy horizontal scaling
  - Works well with Kubernetes / ECS / Swarm

- ❌ Cons:
  - Build-time overhead (TypeScript compilation)
  - Image layering complexity
  - Requires proper caching strategy for CI/CD efficiency

---

### 3. Scaling considerations

- Stateless NestJS app → horizontally scalable containers
- Use:
  - Redis for caching/session state
  - External DB connections with pooling
  - Load balancer (NGINX / ALB / Kubernetes service)

---

### 4. Security implications

- Run as **non-root user**
- Use `.dockerignore` to avoid leaking secrets
- Keep dependencies minimal (`npm ci --only=production`)
- Use pinned Node versions (avoid `latest`)

---

### 5. Deployment flow

1. Build Docker image
2. Push to registry (Docker Hub / ECR / GCR)
3. Deploy via:
   - Docker Compose (simple environments)
   - Kubernetes (production-scale)
   - ECS / Cloud Run (managed)

---

## Example

### `Dockerfile` (production-ready multi-stage)

```dockerfile
# ---- Build stage ----
FROM node:20-alpine AS builder

WORKDIR /app

COPY package*.json ./
RUN npm ci

COPY . .

RUN npm run build


# ---- Production stage ----
FROM node:20-alpine

WORKDIR /app

COPY package*.json ./
RUN npm ci --only=production

COPY --from=builder /app/dist ./dist

# Optional: copy env template
COPY .env.example ./.env

EXPOSE 3000

CMD ["node", "dist/main.js"]
```

---

### `docker-compose.yml` (with DB example)

```yaml
version: "3.9"

services:
  api:
    build: .
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=production
      - DATABASE_URL=postgres://user:pass@db:5432/app
    depends_on:
      - db

  db:
    image: postgres:16
    environment:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: pass
      POSTGRES_DB: app
    ports:
      - "5432:5432"
```

---

### Minimal NestJS app (for context)

```ts
import { Controller, Get, Module } from "@nestjs/common";
import { NestFactory } from "@nestjs/core";

@Controller()
class AppController {
  @Get()
  getHello(): string {
    return "NestJS running in Docker";
  }
}

@Module({
  controllers: [AppController],
})
class AppModule {}

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  await app.listen(3000);
}

bootstrap();
```

---

## Pitfalls

- Not using multi-stage builds → large, insecure images
- Copying `node_modules` from host → platform incompatibility (Linux vs Mac/Windows)
- Forgetting `npm ci` → inconsistent builds in CI/CD
- Running as root inside container → security risk
- Missing `dist/` build step → runtime failures
- Hardcoding environment variables instead of using config system

## Question 2. How do you deploy NestJS on **Kubernetes**?

## Question 3. How do you configure **environment variables** for production?

## Question 4. How do you implement **graceful shutdown**?

## Question 5. How do you implement **horizontal scaling**?

## Question 6. How do you implement **clustering with Node.js**?

## Question 7. How do you configure **NestJS logging for production**?

## Question 8. How do you implement **monitoring and alerting**?

## Question 9. How do you handle **rate-limiting in production**?

## Question 10. How do you implement **load balancing with NestJS apps**?

## Question 11. How do you configure **NestJS with Nginx**?

## Question 12. How do you handle **session management in a clustered environment**?

## Question 13. How do you configure **caching with Redis in production**?

## Question 14. How do you handle **JWT token management in production**?

## Question 15. How do you configure **database connection pools in production**?

## Question 16. How do you implement **hot reload in production deployments**?

## Question 17. How do you implement **zero-downtime deployments**?

## Question 18. How do you scale **WebSocket connections horizontally**?

## Question 19. How do you implement **distributed cron jobs**?

## Question 20. How do you implement **graceful restart for long-running tasks**?
