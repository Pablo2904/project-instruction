# Step 5 – NestJS API

Build a NestJS API that reads processed events from DynamoDB and serves them to the frontend.

---

## 5.1 – Scaffold the project

```bash
npm install -g @nestjs/cli
nest new nest-api
cd nest-api
npm run start:dev
```

Visit [http://localhost:3001](http://localhost:3001) — you should see `Hello World!` (change the port in `src/main.ts` to `3001`).

---

## 5.2 – Understand the project structure

```
nest-api/
├── src/
│   ├── app.module.ts       # root module
│   ├── app.controller.ts   # default controller
│   ├── app.service.ts      # default service
│   └── main.ts             # bootstrap entry point
├── test/
│   └── app.e2e-spec.ts
└── nest-cli.json
```

| Concept | Description |
|---------|-------------|
| **Module** | Groups related controllers and providers |
| **Controller** | Handles incoming HTTP requests |
| **Service** | Business logic; injected via constructor |
| **Provider** | Any class registered in the DI container |
| **Guard** | Runs before a route handler; used for auth |
| **Pipe** | Transforms or validates request data |

---

## 5.3 – Change the port to 3001

Update `src/main.ts`:

```ts
import { ValidationPipe } from '@nestjs/common';
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  app.enableCors();
  app.useGlobalPipes(new ValidationPipe({ whitelist: true }));
  await app.listen(3001);
}
bootstrap();
```

---

## 5.4 – Install DynamoDB dependencies

```bash
npm install @aws-sdk/client-dynamodb @aws-sdk/lib-dynamodb class-validator class-transformer
```

---

## 5.5 – Create a DynamoDB service

Create `src/dynamo/dynamo.module.ts`:

```ts
import { Module, Global } from '@nestjs/common';
import { DynamoService } from './dynamo.service';

@Global()
@Module({
  providers: [DynamoService],
  exports: [DynamoService],
})
export class DynamoModule {}
```

Create `src/dynamo/dynamo.service.ts`:

```ts
import { Injectable } from '@nestjs/common';
import { DynamoDBClient } from '@aws-sdk/client-dynamodb';
import { DynamoDBDocumentClient } from '@aws-sdk/lib-dynamodb';

@Injectable()
export class DynamoService {
  public readonly client: DynamoDBDocumentClient;

  constructor() {
    const baseClient = new DynamoDBClient({
      region: process.env.AWS_REGION ?? 'us-east-1',
      ...(process.env.DYNAMODB_ENDPOINT
        ? { endpoint: process.env.DYNAMODB_ENDPOINT }
        : {}),
    });
    this.client = DynamoDBDocumentClient.from(baseClient);
  }
}
```

Import `DynamoModule` in `app.module.ts`:

```ts
import { DynamoModule } from './dynamo/dynamo.module';

@Module({
  imports: [DynamoModule, EventsModule],
})
export class AppModule {}
```

---

## 5.6 – Generate an Events resource

```bash
nest generate module events
nest generate controller events
nest generate service events
```

Create `src/events/dto/processed-event.dto.ts`:

```ts
export class ProcessedEventDto {
  id: string;
  type: string;
  orderId: number;
  processedAt: string;
}
```

Create `src/events/events.service.ts`:

```ts
import { Injectable } from '@nestjs/common';
import { ScanCommand } from '@aws-sdk/lib-dynamodb';
import { DynamoService } from '../dynamo/dynamo.service';

const TABLE_NAME = process.env.EVENTS_TABLE ?? 'ProcessedEvents';

@Injectable()
export class EventsService {
  constructor(private readonly dynamo: DynamoService) {}

  async findAll() {
    const result = await this.dynamo.client.send(
      new ScanCommand({ TableName: TABLE_NAME }),
    );
    return result.Items ?? [];
  }
}
```

Create `src/events/events.controller.ts`:

```ts
import { Controller, Get } from '@nestjs/common';
import { EventsService } from './events.service';

@Controller('api/events')
export class EventsController {
  constructor(private readonly eventsService: EventsService) {}

  @Get()
  findAll() {
    return this.eventsService.findAll();
  }
}
```

---

## 5.7 – Add JWT authentication (optional)

```bash
npm install @nestjs/jwt @nestjs/passport passport passport-jwt
npm install --save-dev @types/passport-jwt
```

Generate an Auth module:

```bash
nest generate module auth
nest generate service auth
nest generate guard auth/jwt
```

`src/auth/auth.service.ts` (abbreviated):

```ts
import { Injectable } from '@nestjs/common';
import { JwtService } from '@nestjs/jwt';

@Injectable()
export class AuthService {
  constructor(private readonly jwtService: JwtService) {}

  login(userId: number): { accessToken: string } {
    return { accessToken: this.jwtService.sign({ sub: userId }) };
  }
}
```

Apply `JwtAuthGuard` to routes you want to protect:

```ts
import { UseGuards } from '@nestjs/common';
import { JwtAuthGuard } from '../auth/jwt.guard';

@UseGuards(JwtAuthGuard)
@Get()
findAll() { ... }
```

---

## 5.8 – Add Swagger UI

```bash
npm install @nestjs/swagger
```

Update `src/main.ts`:

```ts
import { DocumentBuilder, SwaggerModule } from '@nestjs/swagger';

// inside bootstrap(), before app.listen():
const config = new DocumentBuilder()
  .setTitle('NestJS API')
  .setVersion('1.0')
  .addBearerAuth()
  .build();

const document = SwaggerModule.createDocument(app, config);
SwaggerModule.setup('api/docs', app, document);
```

Visit [http://localhost:3001/api/docs](http://localhost:3001/api/docs) to see the interactive API docs.

---

## 5.9 – Run tests

```bash
npm run test          # unit tests
npm run test:e2e      # end-to-end tests
npm run test:cov      # coverage report
```

---

## 5.10 – Dockerise

Create `nest-api/Dockerfile`:

```dockerfile
FROM node:22-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM node:22-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --omit=dev
COPY --from=build /app/dist ./dist
ENV NODE_ENV=production
EXPOSE 3001
CMD ["node", "dist/main.js"]
```

---

## Next step

➡️ [Step 6 – AWS CDK Infrastructure](step-06-aws-cdk-infra.md)
