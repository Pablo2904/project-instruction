# Backend – Express.js

A step-by-step guide to building a REST API with Express.js, including routing, middleware, database access, and testing.

---

## Step 1 – Scaffold the project

```bash
mkdir express-api && cd express-api
npm init -y
npm install express
npm install --save-dev typescript ts-node-dev @types/node @types/express
npx tsc --init
```

Update `tsconfig.json` to output to `dist/`:

```json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "commonjs",
    "outDir": "dist",
    "rootDir": "src",
    "strict": true,
    "esModuleInterop": true
  }
}
```

Add scripts to `package.json`:

```json
{
  "scripts": {
    "dev": "ts-node-dev --respawn src/index.ts",
    "build": "tsc",
    "start": "node dist/index.js"
  }
}
```

---

## Step 2 – Create the entry point

`src/index.ts`:

```ts
import express from 'express';

const app = express();
const PORT = process.env.PORT ?? 3000;

app.use(express.json());

app.get('/health', (_req, res) => {
  res.json({ status: 'ok' });
});

app.listen(PORT, () => {
  console.log(`Express API listening on port ${PORT}`);
});

export { app };
```

Run the server:

```bash
npm run dev
```

Test it:

```bash
curl http://localhost:3000/health
# {"status":"ok"}
```

---

## Step 3 – Add a resource router

Create `src/routes/posts.ts`:

```ts
import { Router } from 'express';

const router = Router();

// In-memory store for this example
const posts: { id: number; title: string }[] = [];
let nextId = 1;

router.get('/', (_req, res) => {
  res.json(posts);
});

router.post('/', (req, res) => {
  const { title } = req.body as { title: string };
  if (!title) {
    res.status(400).json({ error: 'title is required' });
    return;
  }
  const post = { id: nextId++, title };
  posts.push(post);
  res.status(201).json(post);
});

router.get('/:id', (req, res) => {
  const post = posts.find((p) => p.id === Number(req.params.id));
  if (!post) {
    res.status(404).json({ error: 'Post not found' });
    return;
  }
  res.json(post);
});

router.delete('/:id', (req, res) => {
  const index = posts.findIndex((p) => p.id === Number(req.params.id));
  if (index === -1) {
    res.status(404).json({ error: 'Post not found' });
    return;
  }
  posts.splice(index, 1);
  res.status(204).send();
});

export default router;
```

Register the router in `src/index.ts`:

```ts
import postsRouter from './routes/posts';
// ...
app.use('/posts', postsRouter);
```

---

## Step 4 – Add request validation middleware

```bash
npm install zod
```

Create `src/middleware/validate.ts`:

```ts
import { NextFunction, Request, Response } from 'express';
import { ZodSchema } from 'zod';

export function validate(schema: ZodSchema) {
  return (req: Request, res: Response, next: NextFunction) => {
    const result = schema.safeParse(req.body);
    if (!result.success) {
      res.status(400).json({ errors: result.error.flatten().fieldErrors });
      return;
    }
    req.body = result.data;
    next();
  };
}
```

Use it in `posts.ts`:

```ts
import { z } from 'zod';
import { validate } from '../middleware/validate';

const createPostSchema = z.object({ title: z.string().min(1) });

router.post('/', validate(createPostSchema), (req, res) => {
  // req.body is now validated
});
```

---

## Step 5 – Connect to PostgreSQL with Prisma

```bash
npm install @prisma/client
npm install --save-dev prisma
npx prisma init
```

`prisma/schema.prisma`:

```prisma
datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

generator client {
  provider = "prisma-client-js"
}

model Post {
  id        Int      @id @default(autoincrement())
  title     String
  createdAt DateTime @default(now())
}
```

Create and run the migration:

```bash
npx prisma migrate dev --name init
```

Use Prisma in the route:

```ts
import { PrismaClient } from '@prisma/client';
const prisma = new PrismaClient();

router.get('/', async (_req, res) => {
  const posts = await prisma.post.findMany();
  res.json(posts);
});
```

---

## Step 6 – Error handling

Create `src/middleware/errorHandler.ts`:

```ts
import { ErrorRequestHandler } from 'express';

// eslint-disable-next-line @typescript-eslint/no-unused-vars
export const errorHandler: ErrorRequestHandler = (err, _req, res, _next) => {
  console.error(err);
  res.status(500).json({ error: 'Internal server error' });
};
```

Register it **last** in `src/index.ts`:

```ts
import { errorHandler } from './middleware/errorHandler';
// after all routes:
app.use(errorHandler);
```

---

## Step 7 – Write tests with Jest + Supertest

```bash
npm install --save-dev jest ts-jest supertest @types/supertest @types/jest
```

`jest.config.js`:

```js
module.exports = {
  preset: 'ts-jest',
  testEnvironment: 'node',
};
```

`src/__tests__/posts.test.ts`:

```ts
import request from 'supertest';
import { app } from '../index';

describe('POST /posts', () => {
  it('creates a post', async () => {
    const res = await request(app)
      .post('/posts')
      .send({ title: 'Hello world' });
    expect(res.status).toBe(201);
    expect(res.body).toMatchObject({ title: 'Hello world' });
  });

  it('returns 400 when title is missing', async () => {
    const res = await request(app).post('/posts').send({});
    expect(res.status).toBe(400);
  });
});
```

Run tests:

```bash
npx jest
```

---

## Step 8 – Dockerise

`Dockerfile`:

```dockerfile
FROM node:18-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM node:18-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --omit=dev
COPY --from=build /app/dist ./dist
ENV NODE_ENV=production
EXPOSE 3000
CMD ["node", "dist/index.js"]
```

---

## Next steps

- Upgrade to NestJS → [docs/backend/nest](../nest/README.md)
- Deploy as a Lambda → [docs/backend/lambda](../lambda/README.md)
