# Step 0 – Project Overview

Before writing any code, read this page. It describes exactly what you will build, how every part connects, and what the data looks like end to end.

> **💰 This architecture is free tier eligible.**
> All AWS services used are either always-free or free for 12 months. The only service with a time limit: RDS db.t3.micro (PostgreSQL) — free for 12 months, ~$15/month after.

---

## What you are building

A full-stack, **event-driven order management system** with a three-stage order lifecycle (`created → paid → shipped`). The system spans six independent GitHub repositories:

```
my-app-frontend          React + Vite UI
my-app-express-api       Orders REST API  (PostgreSQL, status machine)
my-app-nest-api          Events read API  (DynamoDB, hexagonal architecture)
my-app-cdk-recipes       Shared CDK L3 Constructs  (npm library)
my-app-infra             AWS infrastructure  (CDK multi-stack)
my-app-lambda            Order-processor Lambda  (writes to DynamoDB)
```

---

## Frontend — two pages

### `/` — Orders
A sortable table of all orders fetched from the Express API. A **Create Order** form at the top. Each row has **action buttons** that advance the order through its lifecycle:

| id | customerName | product | qty | status | createdAt | Actions |
|---|---|---|---|---|---|---|
| 1 | Alice | Widget | 3 | pending | 2024-01-15 10:30 | **Pay** |
| 2 | Bob | Gadget | 1 | paid | 2024-01-15 10:31 | **Ship** |
| 3 | Carol | Doohickey | 5 | shipped | 2024-01-15 10:32 | — |

- **Pay** button visible only when `status === "pending"`
- **Ship** button visible only when `status === "paid"`
- Clicking **Pay** or **Ship** calls the Express API, which **publishes an event** to the async pipeline

### `/events` — Order Event Timeline
A table showing the full event history for all orders, fetched from the NestJS API via DynamoDB:

| orderId | eventType | processedAt |
|---|---|---|
| 1 | order.created | 2024-01-15 10:30:00 |
| 1 | order.paid | 2024-01-15 10:35:00 |
| 1 | order.shipped | 2024-01-15 10:40:00 |

This is a read-only audit log populated entirely by the async pipeline — the frontend never writes to DynamoDB directly.

---

## API endpoints

### Express API — `my-app-express-api` (port 3000)

| Method | Path | What it does |
|---|---|---|
| `GET` | `/health` | Health check — `{ status: "ok" }` |
| `GET` | `/api/orders` | List all orders, sorted `createdAt DESC` |
| `GET` | `/api/orders/:id` | Single order or 404 |
| `POST` | `/api/orders` | Validate → save to PostgreSQL → publish `order.created` → return 201 |
| `PATCH` | `/api/orders/:id/pay` | Validate `status=pending` → update to `paid` → publish `order.paid` → 200, or 409 if wrong status |
| `PATCH` | `/api/orders/:id/ship` | Validate `status=paid` → update to `shipped` → publish `order.shipped` → 200, or 409 if wrong status |
| `DELETE` | `/api/orders/:id` | Delete order or 404 |
| `GET` | `/api/docs` | Swagger UI |

### NestJS API — `my-app-nest-api` (port 3001)

| Method | Path | What it does |
|---|---|---|
| `GET` | `/api/events?orderId=1` | Query DynamoDB `PK = "ORDER#1"` — all events for this order |
| `GET` | `/api/events?eventType=order.paid` | Query DynamoDB GSI `eventType-processedAt-index` — all paid events |
| `GET` | `/api/docs` | Swagger UI |

> **No Scan.** The NestJS API always queries by a known key (orderId or eventType). Table scans are not used.

---

## Order lifecycle (status machine)

```
          POST /api/orders
                │
                ▼
           ┌─────────┐
           │ pending │ ── PATCH /pay ──► ┌──────┐
           └─────────┘                   │ paid │ ── PATCH /ship ──► ┌──────────┐
                                         └──────┘                    │ shipped  │
                                                                      └──────────┘
```

Each transition publishes an EventBridge event → SQS → Lambda → DynamoDB write.

---

## End-to-end data flow

```
[1] User clicks "Pay" on order #1
        │
        ▼
[2] Frontend → PATCH /api/orders/1/pay
              (API Gateway → Lambda Authorizer checks X-API-Key header)
        │
        ▼
[3] Express API → Prisma.order.update({ status: "paid" }) → PostgreSQL
        │
        ▼
[4] Express API → EventBridge PutEvents
                  source:      "express-api"
                  detail-type: "order.paid"
                  detail:      { orderId: 1, timestamp: "2024-01-15T10:35:00Z" }
        │
        ▼
[5] EventBridge rule "order-paid-rule" matches → forwards to SQS "order-events"
        │
        ▼
[6] Lambda Event Source Mapping polls SQS → invoked with SQSEvent
        │
        ▼
[7] Lambda handler → composes DynamoDB item:
                     PK = "ORDER#1"
                     SK = "EVENT#order.paid#2024-01-15T10:35:00Z"
        │
        ▼
[8] Lambda → DynamoDB PutCommand → table "OrderEvents"
        │
        ▼
[9] Frontend /events page → NestJS GET /api/events?orderId=1
                           → DynamoDB Query PK="ORDER#1"
                           → returns all 3 events (created, paid, shipped)
```

---

## Database records

### PostgreSQL — `Order` table

| Column | Type | Notes |
|---|---|---|
| `id` | integer | Auto-increment primary key |
| `customerName` | text | |
| `product` | text | |
| `quantity` | integer | |
| `status` | text | `"pending"` → `"paid"` → `"shipped"` |
| `createdAt` | timestamp | Set by the database on insert |

Example:
```
id=1 | customerName="Alice" | product="Widget" | quantity=3 | status="paid" | createdAt="2024-01-15T10:30:00Z"
```

### DynamoDB — `OrderEvents` table (single-table design)

| Attribute | Type | Notes |
|---|---|---|
| `PK` | String | Partition key — `ORDER#<orderId>` |
| `SK` | String | Sort key — `EVENT#<eventType>#<timestamp>` |
| `orderId` | Number | |
| `eventType` | String | `order.created` / `order.paid` / `order.shipped` |
| `processedAt` | String | ISO 8601 |

Three items for order #1 (all returned by `Query PK = "ORDER#1"`):
```
PK="ORDER#1"  SK="EVENT#order.created#2024-01-15T10:30:00Z"
PK="ORDER#1"  SK="EVENT#order.paid#2024-01-15T10:35:00Z"
PK="ORDER#1"  SK="EVENT#order.shipped#2024-01-15T10:40:00Z"
```

GSI `eventType-processedAt-index` enables querying all orders of a given type across the entire table.

---

## Technology stack

| Layer | Technology | Why |
|---|---|---|
| Frontend | React + Vite, React Query, TanStack Table | Fast dev server, declarative data fetching, headless table |
| Orders API | Express + Prisma + Zod | Lightweight, explicit, easy to test |
| Events API | NestJS (hexagonal) + AWS SDK | Structured, testable, Swagger for free |
| Async pipeline | EventBridge → SQS → Lambda | Fully managed, decoupled, DLQ for retries |
| Databases | PostgreSQL + DynamoDB | Relational CRUD + schema-less event log with proper access patterns |
| Infrastructure | AWS CDK + CDK Recipes (L3 Constructs) | Type-safe infra, reusable patterns |
| Observability | CloudWatch Logs + Alarms | Centralised logs, alerting |
| Auth | API Gateway + Lambda Authorizer | Single entry point, enforced before backend sees any request |

---

## Next step

➡️ [Step 1 – Prerequisites](step-01-prerequisites.md)
