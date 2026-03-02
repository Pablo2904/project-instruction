# Step 3 – Express API

Build a REST API with Express and TypeScript that manages orders in PostgreSQL.

---

## 3.1 – Scaffold the project

```bash
mkdir express-api && cd express-api
npm init -y
npm install express cors
npm install --save-dev typescript ts-node-dev @types/node @types/express @types/cors
npx tsc --init
```

Update `tsconfig.json`:

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "commonjs",
    "outDir": "dist",
    "rootDir": "src",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true
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

## 3.2 – Create the entry point

Create `src/index.ts`:

```ts
import express from 'express';
import cors from 'cors';

const app = express();
const PORT = process.env.PORT ?? 3000;

app.use(cors());
app.use(express.json());

app.get('/health', (_req, res) => {
  res.json({ status: 'ok' });
});

app.listen(PORT, () => {
  console.log(`Express API listening on port ${PORT}`);
});

export { app };
```

Run and test:

```bash
npm run dev
curl http://localhost:3000/health
```

---

## 3.3 – Add request validation with Zod

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

---

## 3.4 – Connect to PostgreSQL with Prisma

```bash
npm install @prisma/client
npm install --save-dev prisma
npx prisma init
```

Update `prisma/schema.prisma`:

```prisma
datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

generator client {
  provider = "prisma-client-js"
}

model Order {
  id           Int      @id @default(autoincrement())
  customerName String
  product      String
  quantity     Int
  status       String   @default("pending")
  createdAt    DateTime @default(now())
}
```

Create `.env`:

```
DATABASE_URL="postgresql://app:secret@localhost:5432/appdb?schema=public"
```

Run the migration:

```bash
npx prisma migrate dev --name init
```

---

## 3.5 – Create the Orders router

Create `src/routes/orders.ts`:

```ts
import { Router } from 'express';
import { PrismaClient } from '@prisma/client';
import { z } from 'zod';
import { validate } from '../middleware/validate';

const router = Router();
const prisma = new PrismaClient();

const createOrderSchema = z.object({
  customerName: z.string().min(1),
  product: z.string().min(1),
  quantity: z.number().int().positive(),
});

router.get('/', async (_req, res) => {
  const orders = await prisma.order.findMany({ orderBy: { createdAt: 'desc' } });
  res.json(orders);
});

router.get('/:id', async (req, res) => {
  const order = await prisma.order.findUnique({
    where: { id: Number(req.params.id) },
  });
  if (!order) {
    res.status(404).json({ error: 'Order not found' });
    return;
  }
  res.json(order);
});

router.post('/', validate(createOrderSchema), async (req, res) => {
  const order = await prisma.order.create({ data: req.body });

  // TODO (Step 9): publish order.created event to EventBridge here

  res.status(201).json(order);
});

router.delete('/:id', async (req, res) => {
  const id = Number(req.params.id);
  const order = await prisma.order.findUnique({ where: { id } });
  if (!order) {
    res.status(404).json({ error: 'Order not found' });
    return;
  }
  await prisma.order.delete({ where: { id } });
  res.status(204).send();
});

export default router;
```

Register the router in `src/index.ts`:

```ts
import ordersRouter from './routes/orders';

// after app.use(express.json()):
app.use('/api/orders', ordersRouter);
```

---

## 3.6 – Add error handling

Create `src/middleware/errorHandler.ts`:

```ts
import { ErrorRequestHandler } from 'express';

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

## 3.7 – Write tests with Jest + Supertest

```bash
npm install --save-dev jest ts-jest supertest @types/supertest @types/jest
```

Create `jest.config.js`:

```js
module.exports = {
  preset: 'ts-jest',
  testEnvironment: 'node',
};
```

Create `src/__tests__/orders.test.ts`:

```ts
import request from 'supertest';
import { app } from '../index';

describe('POST /api/orders', () => {
  it('creates an order', async () => {
    const res = await request(app)
      .post('/api/orders')
      .send({ customerName: 'Alice', product: 'Widget', quantity: 3 });
    expect(res.status).toBe(201);
    expect(res.body).toMatchObject({ customerName: 'Alice', product: 'Widget' });
  });

  it('returns 400 when customerName is missing', async () => {
    const res = await request(app)
      .post('/api/orders')
      .send({ product: 'Widget', quantity: 1 });
    expect(res.status).toBe(400);
  });
});
```

Run tests:

```bash
npx jest
```

---

## 3.8 – Dockerise

Create `express-api/Dockerfile`:

```dockerfile
FROM node:22-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npx prisma generate
RUN npm run build

FROM node:22-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --omit=dev
COPY --from=build /app/dist ./dist
COPY --from=build /app/node_modules/.prisma ./node_modules/.prisma
ENV NODE_ENV=production
EXPOSE 3000
CMD ["node", "dist/index.js"]
```

---

## Next step

➡️ [Step 4 – Local Environment (Docker Compose)](step-04-docker-local-env.md)
