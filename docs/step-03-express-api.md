# Step 3 – Express API

Build a REST API with Express and TypeScript that manages orders in PostgreSQL and publishes events to EventBridge.

> **Repository:** work inside your `my-app-express-api` repo for this step.

> **📖 What is a REST API?**
> REST (Representational State Transfer) is a convention for designing HTTP endpoints. Each endpoint represents a _resource_ (e.g. `orders`) and uses HTTP verbs to express intent: `GET` to read, `POST` to create, `PUT`/`PATCH` to update, `DELETE` to remove. Clients and servers communicate only through these standard HTTP methods — no special protocol needed.

---

## 3.1 – Scaffold the project

```bash
mkdir src
npm init -y
npm install express cors
npm install --save-dev typescript ts-node-dev @types/node @types/express @types/cors
npx tsc --init
```

Add scripts to `package.json`:

> **🤖 Ask your AI assistant:**
> ```
> Add these scripts to package.json:
> "dev": "ts-node-dev --respawn src/index.ts"
> "build": "tsc"
> "start": "node dist/index.js"
> ```

---

## 3.2 – Create the entry point

> **🤖 Ask your AI assistant:**
> ```
> Create src/index.ts: an Express app that enables CORS and JSON body parsing,
> exposes a GET /health endpoint returning { status: "ok" }, and listens on
> process.env.PORT or 3000. Export the app instance for testing.
> ```

Run and verify:

```bash
npm run dev
curl http://localhost:3000/health
```

---

## 3.3 – Add request validation with Zod

> **📖 What is schema validation?**
> When a client sends a request body, you cannot trust that it contains the right fields and types. **Zod** lets you define the exact shape of valid data as a TypeScript schema, then call `.safeParse()` on incoming data. If the data does not match, you get a structured list of what is wrong — without writing a single `if` statement.

```bash
npm install zod
```

> **🤖 Ask your AI assistant:**
> ```
> Create src/middleware/validate.ts: an Express middleware factory that takes a
> ZodSchema, calls safeParse on req.body, returns 400 with field errors on
> failure, or calls next() with the parsed data.
> ```

---

## 3.4 – Connect to PostgreSQL with Prisma

> **📖 What is an ORM?**
> An ORM (Object-Relational Mapper) is a library that lets you work with a relational database using TypeScript/JavaScript objects instead of raw SQL. **Prisma** is a type-safe ORM: you define your data models in a `schema.prisma` file and Prisma generates a fully-typed client. This means your editor autocompletes field names and types, and typos in database queries become compile-time errors rather than runtime surprises.

> **📖 What is a database migration?**
> A **migration** is a versioned, incremental change to your database schema. Instead of manually altering tables, you describe the change in a migration file and Prisma applies it. This matters for two reasons:
> 1. **Safety** — the database is updated in a controlled, repeatable way. If something goes wrong you can roll back.
> 2. **Team collaboration** — every developer, every CI pipeline, and every environment runs the exact same sequence of schema changes. `npx prisma migrate dev` generates the SQL for you and applies it locally.

```bash
npm install @prisma/client
npm install --save-dev prisma
npx prisma init
```

Create `.env`:

```
DATABASE_URL="postgresql://app:secret@localhost:5432/appdb?schema=public"
```

> **🤖 Ask your AI assistant:**
> ```
> Update prisma/schema.prisma to define an Order model with fields:
> id (autoincrement), customerName (String), product (String),
> quantity (Int), status (String, default "pending"), createdAt (DateTime, default now()).
> ```

Run the migration:

```bash
npx prisma migrate dev --name init
```

---

## 3.5 – Create the Orders router

> **🤖 Ask your AI assistant:**
> ```
> Create src/routes/orders.ts: an Express Router with four endpoints:
> GET /    → return all orders ordered by createdAt desc
> GET /:id → return one order or 404
> POST /   → validate body with Zod (customerName, product, quantity), create order, return 201
> DELETE /:id → delete order or 404
> Use PrismaClient and the validate middleware.
> ```

Register the router in `src/index.ts`:

```bash
# Add this import and line after app.use(express.json()):
# import ordersRouter from './routes/orders';
# app.use('/api/orders', ordersRouter);
```

> **🤖 Ask your AI assistant:**
> ```
> Register ordersRouter at /api/orders in src/index.ts.
> Also add a global error handler middleware at the very end that catches
> unhandled errors and returns 500 with { error: "Internal server error" }.
> ```

---

## 3.6 – Add Swagger / OpenAPI

```bash
npm install swagger-ui-express swagger-jsdoc @types/swagger-ui-express @types/swagger-jsdoc
```

> **🤖 Ask your AI assistant:**
> ```
> In src/index.ts, set up swagger-jsdoc with OpenAPI 3.0 info block
> (title: "Express API", version: "1.0").
> Mount SwaggerUI at /api/docs using swagger-ui-express.
> Add JSDoc @swagger comments to each route in orders.ts documenting
> the request body and possible responses.
> ```

Visit [http://localhost:3000/api/docs](http://localhost:3000/api/docs) to see the interactive docs.

> **Note:** Notice how much manual JSDoc boilerplate was required. In Step 5 you will set up the equivalent in NestJS with just three lines of code — no JSDoc comments needed.

---

## 3.7 – Write tests with Jest + Supertest

```bash
npm install --save-dev jest ts-jest supertest @types/supertest @types/jest
```

> **🤖 Ask your AI assistant:**
> ```
> Create jest.config.js using ts-jest preset with testEnvironment: "node".
> Then create src/__tests__/orders.test.ts with two tests:
> POST /api/orders with valid body returns 201.
> POST /api/orders without customerName returns 400.
> Use supertest and import app from src/index.
> ```

Run tests:

```bash
npx jest
```

---

## 3.8 – Dockerise

> **🤖 Ask your AI assistant:**
> ```
> Create a multi-stage Dockerfile for the Express API.
> Stage 1 (build): node:22-alpine, npm ci, npx prisma generate, npm run build.
> Stage 2 (run): node:22-alpine, npm ci --omit=dev, copy dist/ and
> the generated Prisma client, expose port 3000, CMD node dist/index.js.
> ```

## 3.9 – Lambda handler with `serverless-http`

In AWS, this Express app runs as a **Lambda function** — there is no long-running server. Lambda receives HTTP requests from API Gateway as JSON, not as real HTTP sockets. `serverless-http` acts as the bridge between the two.

> **📖 How `serverless-http` works**
>
> API Gateway calls Lambda with an event like this:
> ```json
> {
>   "httpMethod": "POST",
>   "path": "/api/orders",
>   "headers": { "Content-Type": "application/json" },
>   "body": "{\"customerName\":\"Alice\",\"product\":\"Widget\"}"
> }
> ```
>
> Express expects a Node.js `IncomingMessage` stream — not JSON. `serverless-http` translates:
>
> ```
> API Gateway JSON event
>         │
>         ▼
>   serverless-http
>   ├── creates a fake IncomingMessage (method, url, headers, body)
>   ├── creates a fake ServerResponse to capture output
>   └── runs your normal Express app (all middleware, routes, validation)
>         │
>         ▼
>   serverless-http
>   ├── reads the captured response (status, headers, body)
>   └── formats it as a Lambda response object
>         │
>         ▼
>   { statusCode: 201, headers: {...}, body: "..." }
> ```
>
> **Your Express code changes nothing.** Every route, middleware, and Zod validator is identical — they still receive a normal `req`/`res` pair.

```bash
npm install serverless-http
npm install --save-dev @types/aws-lambda
```

First, extract app creation into a factory function so it can be shared between the local dev server and the Lambda handler:

> **🤖 Ask your AI assistant:**
> ```
> Refactor src/index.ts: extract Express app creation into a factory function
> createApp() exported from src/app.ts. It sets up all middleware (cors, json)
> and mounts all routers. src/index.ts imports createApp() and calls
> app.listen() as before — local development is completely unchanged.
> ```

Then add the Lambda entry point — this is the file that CDK will bundle and deploy:

> **🤖 Ask your AI assistant:**
> ```
> Create src/lambda.ts in my-app-express-api.
> Import serverlessHttp from "serverless-http" and createApp from "./app".
> Export: export const handler = serverlessHttp(createApp());
> ```

> **💰 Cost:** Lambda is billed only for the milliseconds your handler runs. With 1M free invocations per month, a learning project costs nothing.

---

## 3.10 – Push to GitHub

```bash
git add .
git commit -m "feat: express api with orders, swagger, tests, and lambda handler"
git remote add origin https://github.com/<YOUR_ORG>/my-app-express-api.git
git push -u origin main
```

---

## Next step

➡️ [Step 4 – Local Environment](step-04-docker-local-env.md)
