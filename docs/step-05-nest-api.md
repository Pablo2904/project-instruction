# Step 5 – NestJS API (Hexagonal Architecture)

Build a NestJS API that reads processed events from DynamoDB. The API is structured using **Hexagonal Architecture** (Ports & Adapters) — a pattern that keeps business logic completely independent of frameworks and infrastructure.

> **Repository:** work inside your `my-app-nest-api` repo for this step.

> **📖 Why NestJS instead of plain Express?**
> NestJS is a structured framework built on top of Express. It enforces a module system, uses TypeScript decorators to declare routes and services, and provides **Dependency Injection (DI)** out of the box. DI means you declare what dependencies a class needs in its constructor and the framework wires them together — you never call `new SomeService()` yourself. This makes code easy to test (swap real services for mocks) and easy to reason about (each class is responsible for one thing).

---

## 5.1 – Scaffold the project

```bash
npm install -g @nestjs/cli
nest new . --skip-git
npm run start:dev
```

Visit [http://localhost:3000](http://localhost:3000) — you should see `Hello World!`.

Change the port to `3001`:

> **🤖 Ask your AI assistant:**
> ```
> In src/main.ts, change the app to listen on port 3001.
> Also enable CORS and add ValidationPipe with whitelist: true globally.
> ```

---

## 5.2 – Hexagonal Architecture concepts

> **📖 Why hexagonal architecture?**
> In a traditional layered app the database details leak into every layer — if you swap PostgreSQL for DynamoDB you touch a dozen files. Hexagonal architecture inverts this: the domain defines an **interface** (Port) describing what it needs (e.g. “fetch all events”), and the infrastructure provides a concrete **Adapter** that fulfils that interface. Swapping DynamoDB for an in-memory store means writing one new class, nothing else changes. It also makes unit testing trivial — inject a stub adapter instead of a real database.

Hexagonal Architecture (also called Ports & Adapters) organises code into three layers:

```
┌─────────────────────────────────────────────┐
│              Infrastructure                 │
│  (HTTP controllers, DynamoDB adapter, etc.) │
│           │              ▲                  │
│     calls │              │ implements       │
│           ▼              │                  │
│         ┌────────────────────────┐          │
│         │      Application       │          │
│         │  (use cases / services)│          │
│         │  depends on → Ports    │          │
│         └────────────────────────┘          │
│                    │                        │
│              depends on                     │
│                    ▼                        │
│         ┌────────────────────────┐          │
│         │        Domain          │          │
│         │ (entities, port defs)  │          │
│         └────────────────────────┘          │
└─────────────────────────────────────────────┘
```

| Concept | Role | Example |
|---------|------|---------|
| **Domain** | Core business entities and port interfaces — zero framework dependencies | `ProcessedEvent` entity, `EventRepository` interface |
| **Port** | An interface that the application layer depends on | `EventRepository` with `findAll()` method |
| **Adapter** | A concrete implementation of a port | `DynamoDBEventRepository` implementing `EventRepository` |
| **Use Case** | Application logic orchestrating domain objects through ports | `GetEventsUseCase` |
| **Controller** | Infrastructure adapter that receives HTTP requests and calls use cases | `EventsController` |

The key benefit: **swap DynamoDB for any other data store by writing a new adapter** — no changes to the domain or application layers.

---

## 5.3 – Folder structure

Create the following directory layout:

```bash
mkdir -p src/domain
mkdir -p src/application
mkdir -p src/infrastructure/adapters
mkdir -p src/infrastructure/http
```

```
src/
├── domain/
│   ├── processed-event.entity.ts    # pure TypeScript class, no imports
│   └── event-repository.port.ts     # interface (port)
├── application/
│   └── get-events.use-case.ts       # orchestrates via the port
├── infrastructure/
│   ├── adapters/
│   │   └── dynamodb-event.repository.ts  # DynamoDB adapter (implements port)
│   └── http/
│       └── events.controller.ts     # HTTP adapter
├── events.module.ts
└── main.ts
```

---

## 5.4 – Domain layer

The domain layer has **no NestJS imports** — pure TypeScript only.

> **🤖 Ask your AI assistant:**
> ```
> Create src/domain/processed-event.entity.ts:
> A plain TypeScript class ProcessedEvent with four public fields:
> id (string), type (string), orderId (number), processedAt (string).
> ```

> **🤖 Ask your AI assistant:**
> ```
> Create src/domain/event-repository.port.ts:
> An exported TypeScript interface EventRepository with one method:
> findAll(): Promise<ProcessedEvent[]>
> Also export a Symbol injection token: EVENT_REPOSITORY = Symbol('EventRepository').
> ```

---

## 5.5 – Application layer (Use Case)

> **🤖 Ask your AI assistant:**
> ```
> Create src/application/get-events.use-case.ts:
> A NestJS injectable class GetEventsUseCase.
> Constructor injects EventRepository using the EVENT_REPOSITORY token (@Inject).
> It has one method: execute(): Promise<ProcessedEvent[]> that calls findAll() on the repository.
> ```

---

## 5.6 – Infrastructure adapter (DynamoDB)

```bash
npm install @aws-sdk/client-dynamodb @aws-sdk/lib-dynamodb
```

> **🤖 Ask your AI assistant:**
> ```
> Create src/infrastructure/adapters/dynamodb-event.repository.ts:
> A NestJS injectable class DynamoDBEventRepository that implements EventRepository.
> In the constructor, create a DynamoDBDocumentClient using DynamoDBClient with region
> from process.env.AWS_REGION and optional endpoint from process.env.DYNAMODB_ENDPOINT.
>
> Implement two methods:
>   findByOrderId(orderId: number): Promise<OrderEvent[]>
>     — sends a QueryCommand with KeyConditionExpression "PK = :pk"
>       and ExpressionAttributeValues { ":pk": "ORDER#" + orderId }
>       to the table process.env.EVENTS_TABLE.
>   findByEventType(eventType: string): Promise<OrderEvent[]>
>     — sends a QueryCommand against the GSI "eventType-processedAt-index"
>       with KeyConditionExpression "eventType = :et".
> Map each DynamoDB item to an OrderEvent instance.
> ```

---

## 5.7 – HTTP Controller

```bash
nest generate module events
```

> **🤖 Ask your AI assistant:**
> ```
> Create src/infrastructure/http/events.controller.ts:
> A NestJS controller with @Controller('api/events').
> Inject GetEventsUseCase in the constructor.
> Add a GET handler that calls useCase.execute() and returns the result.
> ```

---

## 5.8 – Wire the module

> **🤖 Ask your AI assistant:**
> ```
> Create src/events.module.ts:
> A NestJS @Module that:
> - imports nothing
> - provides: GetEventsUseCase and { provide: EVENT_REPOSITORY, useClass: DynamoDBEventRepository }
> - exports: GetEventsUseCase
> - declares EventsController
> Import EventsModule in AppModule.
> ```

---

## 5.9 – Swagger (three lines vs. Step 3's boilerplate)

```bash
npm install @nestjs/swagger
```

> **🤖 Ask your AI assistant:**
> ```
> In src/main.ts, after creating the app, set up SwaggerModule:
> Use DocumentBuilder to set title "NestJS API" and version "1.0".
> Mount the docs at /api/docs.
> ```

Visit [http://localhost:3001/api/docs](http://localhost:3001/api/docs).

> **Compare with Step 3:** In Express you wrote JSDoc `@swagger` comments on every route by hand. In NestJS, `@nestjs/swagger` reads your TypeScript decorators automatically — the documentation is derived from source code with **zero extra annotations** for basic endpoints, and minimal ones for advanced cases.

---

## 5.10 – Run tests

```bash
npm run test          # unit tests
npm run test:e2e      # end-to-end tests
npm run test:cov      # coverage report
```

> **🤖 Ask your AI assistant:**
> ```
> Write a unit test for GetEventsUseCase.
> Mock EventRepository to return two ProcessedEvent instances.
> Assert that execute() returns exactly those two items.
> ```

---

## 5.11 – Dockerise

> **🤖 Ask your AI assistant:**
> ```
> Create a multi-stage Dockerfile for the NestJS app.
> Stage 1 (build): node:22-alpine, npm ci, npm run build.
> Stage 2 (run): node:22-alpine, npm ci --omit=dev, copy dist/, expose 3001,
> CMD node dist/main.js.
> ```

---

## 5.12 – Lambda handler with `serverless-http`

Same approach as in Step 3 (Express API) — `serverless-http` bridges API Gateway's JSON events to a Node.js HTTP interface. NestJS needs one extra step: the framework has a startup phase (DI container wiring, decorator scanning) that takes ~300ms and must be handled carefully.

```bash
npm install serverless-http
npm install --save-dev @types/aws-lambda
```

> **📖 Lambda container reuse**
> AWS Lambda reuses the same Node.js process for multiple consecutive invocations (**warm start**). Code outside the `handler` function runs only once when the container starts, not on every request. This is exactly where we put the NestJS bootstrap.

> **🤖 Ask your AI assistant:**
> ```
> Create src/lambda.ts in my-app-nest-api.
> Import NestFactory from "@nestjs/core", serverlessHttp from "serverless-http",
> and AppModule from "./app.module".
>
> Declare a module-level variable: let cachedApp: any;
>
> Define an async function bootstrap() that:
>   1. Returns cachedApp immediately if it is already set (warm start — skips init).
>   2. Creates a NestJS app: NestFactory.create(AppModule, { logger: false })
>   3. Calls await nestApp.init()
>   4. Assigns cachedApp = serverlessHttp(nestApp.getHttpAdapter().getInstance())
>   5. Returns cachedApp.
>
> Export:
>   export const handler = async (event: any, context: any) => {
>     const app = await bootstrap();
>     return app(event, context);
>   };
> ```

> **📖 Why cache `cachedApp`?**
> ```
> Cold start (first request after idle):
>   handler() → bootstrap() → NestFactory.create() [~300ms] → serverless-http()
>                                                    ↓
>                                              cachedApp = handler
>
> Warm start (subsequent requests):
>   handler() → bootstrap() → cachedApp already set → return immediately [~0ms]
> ```
> Without caching, NestJS re-wires its entire DI container on every single Lambda invocation — that's 300ms added to every response. Caching makes warm invocations as fast as Express.

---

## 5.13 – Push to GitHub

```bash
git add .
git commit -m "feat: nest api with hexagonal architecture, swagger, and lambda handler"
git remote add origin https://github.com/<YOUR_ORG>/my-app-nest-api.git
git push -u origin main
```

---

## Checklist

- [ ] `GET /api/events` returns a list (empty list is fine — DynamoDB will be populated in Step 10)
- [ ] `GET /api/docs` shows the Swagger UI with `EventsController` listed
- [ ] Domain layer (`src/domain/`) has zero NestJS imports
- [ ] `src/lambda.ts` exports a `handler` with app caching
- [ ] Swapping the DynamoDB adapter for an in-memory stub works without touching use case or domain code

---

## Next step

➡️ [Step 6 – CloudWatch Observability](step-14-cloudwatch.md)
