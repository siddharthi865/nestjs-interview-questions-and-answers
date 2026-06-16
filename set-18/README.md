# Set 18

| S.No. | Question                                                                                                                                       |
| ----- | ---------------------------------------------------------------------------------------------------------------------------------------------- |
| 1.    | [How do you integrate **AWS S3 for file uploads**?](#question-1-how-do-you-integrate-aws-s3-for-file-uploads)                                  |
| 2.    | [How do you integrate **Stripe or payment gateways**?](#question-2-how-do-you-integrate-stripe-or-payment-gateways)                            |
| 3.    | [How do you integrate **Firebase services**?](#question-3-how-do-you-integrate-firebase-services)                                              |
| 4.    | [How do you integrate **external REST APIs** with NestJS?](#question-4-how-do-you-integrate-external-rest-apis-with-nestjs)                    |
| 5.    | [How do you integrate **GraphQL clients**?](#question-5-how-do-you-integrate-graphql-clients)                                                  |
| 6.    | [How do you implement **email sending services**?](#question-6-how-do-you-implement-email-sending-services)                                    |
| 7.    | [How do you implement **SMS gateways**?](#question-7-how-do-you-implement-sms-gateways)                                                        |
| 8.    | [How do you implement **push notifications**?](#question-8-how-do-you-implement-push-notifications)                                            |
| 9.    | [How do you implement **OAuth login with multiple providers**?](#question-9-how-do-you-implement-oauth-login-with-multiple-providers)          |
| 10.   | [How do you implement **LDAP authentication**?](#question-10-how-do-you-implement-ldap-authentication)                                         |
| 11.   | [How do you integrate **ElasticSearch**?](#question-11-how-do-you-integrate-elasticsearch)                                                     |
| 12.   | [How do you integrate **full-text search in TypeORM**?](#question-12-how-do-you-integrate-full-text-search-in-typeorm)                         |
| 13.   | [How do you integrate **WebSocket clients for external services**?](#question-13-how-do-you-integrate-websocket-clients-for-external-services) |
| 14.   | [How do you integrate **Kafka or RabbitMQ producers/consumers**?](#question-14-how-do-you-integrate-kafka-or-rabbitmq-producersconsumers)      |
| 15.   | [How do you integrate **prometheus metrics**?](#question-15-how-do-you-integrate-prometheus-metrics)                                           |
| 16.   | [How do you implement **distributed logging**?](#question-16-how-do-you-implement-distributed-logging)                                         |
| 17.   | [How do you integrate **OpenTelemetry for tracing**?](#question-17-how-do-you-integrate-opentelemetry-for-tracing)                             |
| 18.   | [How do you integrate **third-party analytics**?](#question-18-how-do-you-integrate-third-party-analytics)                                     |
| 19.   | [How do you implement **service-to-service authentication**?](#question-19-how-do-you-implement-service-to-service-authentication)             |
| 20.   | [How do you integrate **GraphQL federation**?](#question-20-how-do-you-integrate-graphql-federation)                                           |

## Question 1. How do you integrate **AWS S3 for file uploads**?

## Short answer

You integrate AWS S3 in NestJS by using the AWS SDK (`@aws-sdk/client-s3`) inside a dedicated provider/service, handling file uploads via `multer` (or stream-based upload), and exposing an API endpoint that uploads files directly or via a service abstraction.

---

## Explanation

### 1. Architecture (senior-level view)

In production NestJS systems, S3 integration is typically designed as:

- **Controller layer** → receives multipart/form-data upload
- **Interceptor (Multer)** → handles file parsing (memory or disk storage)
- **Service layer (S3Service)** → encapsulates AWS SDK logic
- **Provider (ConfigModule)** → injects AWS credentials securely
- **Optional presigned URLs** → for direct client-to-S3 uploads (best scalability)

### 2. Upload strategies

#### A. Server-mediated upload (simple)

Client → NestJS → S3

Pros:

- Easy validation, security checks, transformations
  Cons:
- Higher server bandwidth cost
- Bottleneck at API layer

#### B. Presigned URL upload (recommended for scale)

Client → S3 directly using signed URL

Pros:

- Highly scalable
- Minimal backend load
  Cons:
- Requires careful auth + permission design

---

### 3. Security considerations

- Store credentials in **AWS Secrets Manager / environment variables**
- Use **least privilege IAM roles** (S3 bucket scoped)
- Validate file types + size in NestJS
- Use **signed URLs expiration (short TTL)**
- Consider antivirus scanning pipeline (event-driven via S3 triggers)

---

### 4. Scalability considerations

- Avoid memory storage for large files (`multer memoryStorage` only for small files)
- Prefer **streaming uploads**
- Use **multipart upload for large files (>5MB–100MB+)**
- Offload static file serving to S3 + CloudFront CDN

---

### 5. Testing strategy

- Mock AWS SDK using dependency injection
- Use localstack for integration tests
- Validate controller + service separately
- E2E tests should verify upload response metadata (not actual AWS upload in CI)

---

## Example

### Install dependencies

```bash
npm install @aws-sdk/client-s3 @nestjs/platform-express multer
```

---

### S3 Module

```ts
import { Module } from "@nestjs/common";
import { ConfigModule, ConfigService } from "@nestjs/config";
import { S3Client } from "@aws-sdk/client-s3";
import { S3Service } from "./s3.service";

@Module({
  imports: [ConfigModule],
  providers: [
    {
      provide: "S3_CLIENT",
      inject: [ConfigService],
      useFactory: (config: ConfigService) => {
        return new S3Client({
          region: config.get<string>("AWS_REGION"),
          credentials: {
            accessKeyId: config.get<string>("AWS_ACCESS_KEY_ID")!,
            secretAccessKey: config.get<string>("AWS_SECRET_ACCESS_KEY")!,
          },
        });
      },
    },
    S3Service,
  ],
  exports: [S3Service],
})
export class S3Module {}
```

---

### S3 Service

```ts
import { Inject, Injectable } from "@nestjs/common";
import { S3Client, PutObjectCommand } from "@aws-sdk/client-s3";

@Injectable()
export class S3Service {
  constructor(@Inject("S3_CLIENT") private readonly s3: S3Client) {}

  async uploadFile(params: {
    bucket: string;
    key: string;
    buffer: Buffer;
    mimetype: string;
  }): Promise<string> {
    await this.s3.send(
      new PutObjectCommand({
        Bucket: params.bucket,
        Key: params.key,
        Body: params.buffer,
        ContentType: params.mimetype,
      }),
    );

    return `https://${params.bucket}.s3.amazonaws.com/${params.key}`;
  }
}
```

---

### Controller with Multer

```ts
import {
  Controller,
  Post,
  UploadedFile,
  UseInterceptors,
} from "@nestjs/common";
import { FileInterceptor } from "@nestjs/platform-express";
import { S3Service } from "./s3.service";

@Controller("files")
export class FileController {
  constructor(private readonly s3Service: S3Service) {}

  @Post("upload")
  @UseInterceptors(FileInterceptor("file"))
  async upload(@UploadedFile() file: Express.Multer.File) {
    const key = `uploads/${Date.now()}-${file.originalname}`;

    const url = await this.s3Service.uploadFile({
      bucket: process.env.AWS_BUCKET_NAME!,
      key,
      buffer: file.buffer,
      mimetype: file.mimetype,
    });

    return { url };
  }
}
```

---

## Pitfalls

- Using **memory storage for large files** → can crash Node.js due to heap pressure
- Missing **IAM least-privilege policies** → security risk (public bucket exposure)
- Not handling **S3 retries / network failures**
- Hardcoding bucket names or credentials instead of using `ConfigModule`
- Returning S3 URL without CDN → performance degradation globally
- Ignoring **multipart upload for large files**

## Question 2. How do you integrate **Stripe or payment gateways**?

## Question 3. How do you integrate **Firebase services**?

## Question 4. How do you integrate **external REST APIs** with NestJS?

## Question 5. How do you integrate **GraphQL clients**?

## Question 6. How do you implement **email sending services**?

## Question 7. How do you implement **SMS gateways**?

## Question 8. How do you implement **push notifications**?

## Question 9. How do you implement **OAuth login with multiple providers**?

## Question 10. How do you implement **LDAP authentication**?

## Question 11. How do you integrate **ElasticSearch**?

## Question 12. How do you integrate **full-text search in TypeORM**?

## Question 13. How do you integrate **WebSocket clients for external services**?

## Question 14. How do you integrate **Kafka or RabbitMQ producers/consumers**?

## Question 15. How do you integrate **prometheus metrics**?

## Question 16. How do you implement **distributed logging**?

## Question 17. How do you integrate **OpenTelemetry for tracing**?

## Question 18. How do you integrate **third-party analytics**?

## Question 19. How do you implement **service-to-service authentication**?

## Question 20. How do you integrate **GraphQL federation**?
