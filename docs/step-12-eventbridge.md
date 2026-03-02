# Step 12 – EventBridge (3 Event Rules)

Set up Amazon EventBridge to route three different order lifecycle events from the Express API to the SQS queue. EventBridge decouples the producer from the consumer — Express does not need to know that SQS or Lambda exist.

> **📖 What is EventBridge?**
> EventBridge is a serverless event bus. Services publish structured JSON events; you define **rules** that match specific events by pattern and route them to one or more **targets**. One source can fan out to many targets simultaneously. Adding a new consumer (e.g. a notification service) requires zero changes to the producer.

---

## 12.1 – The three events

| Event type | Triggered when | Source |
|---|---|---|
| `order.created` | User submits the create order form | `POST /api/orders` |
| `order.paid` | User clicks **Pay** on the Orders table | `PATCH /api/orders/:id/pay` |
| `order.shipped` | User clicks **Ship** on the Orders table | `PATCH /api/orders/:id/ship` |

All three use the same source (`"express-api"`) and the same SQS target. EventBridge matches each event by `detail-type` and routes it to the queue. Lambda then handles all three.

---

## 12.2 – Express API: publish all three events

```bash
cd my-app-express-api
npm install @aws-sdk/client-eventbridge
```

> **🤖 Ask your AI assistant:**
> ```
> Create src/events/eventbridge.ts in the Express API.
> Export a publishOrderEvent(eventType: string, orderId: number) function
> that sends a PutEventsCommand with:
>   source: "express-api"
>   detail-type: eventType  (e.g. "order.created", "order.paid", "order.shipped")
>   detail: { orderId, timestamp: new Date().toISOString() }
> Support process.env.EVENTBRIDGE_ENDPOINT for LocalStack.
> ```

> **🤖 Ask your AI assistant:**
> ```
> In src/routes/orders.ts, add three handlers:
> 1. POST /api/orders — after Prisma create, call publishOrderEvent("order.created", order.id)
> 2. PATCH /api/orders/:id/pay — validate current status is "pending",
>    update status to "paid" with Prisma, call publishOrderEvent("order.paid", id)
> 3. PATCH /api/orders/:id/ship — validate current status is "paid",
>    update status to "shipped" with Prisma, call publishOrderEvent("order.shipped", id)
> Return 409 if the status transition is not valid.
> ```

---

## 12.3 – Create three rules locally with LocalStack

```bash
docker compose up localstack
export AWS_ACCESS_KEY_ID=test
export AWS_SECRET_ACCESS_KEY=test
export AWS_DEFAULT_REGION=us-east-1

QUEUE_ARN=$(aws --endpoint-url=http://localhost:4566 sqs get-queue-attributes \
  --queue-url http://sqs.us-east-1.localhost.localstack.cloud:4566/000000000000/order-events \
  --attribute-names QueueArn \
  --query 'Attributes.QueueArn' --output text)

# Rule 1: order.created
aws --endpoint-url=http://localhost:4566 events put-rule \
  --name order-created-rule \
  --event-pattern '{"source":["express-api"],"detail-type":["order.created"]}'
aws --endpoint-url=http://localhost:4566 events put-targets \
  --rule order-created-rule \
  --targets "[{\"Id\":\"queue\",\"Arn\":\"${QUEUE_ARN}\"}]"

# Rule 2: order.paid
aws --endpoint-url=http://localhost:4566 events put-rule \
  --name order-paid-rule \
  --event-pattern '{"source":["express-api"],"detail-type":["order.paid"]}'
aws --endpoint-url=http://localhost:4566 events put-targets \
  --rule order-paid-rule \
  --targets "[{\"Id\":\"queue\",\"Arn\":\"${QUEUE_ARN}\"}]"

# Rule 3: order.shipped
aws --endpoint-url=http://localhost:4566 events put-rule \
  --name order-shipped-rule \
  --event-pattern '{"source":["express-api"],"detail-type":["order.shipped"]}'
aws --endpoint-url=http://localhost:4566 events put-targets \
  --rule order-shipped-rule \
  --targets "[{\"Id\":\"queue\",\"Arn\":\"${QUEUE_ARN}\"}]"
```

> **📈 Paste into `setup-local.sh` (Step 12 block)**
> Add this after the SQS block from Step 11. The loop handles all three event types.

```bash

echo "▶ Creating EventBridge rules..."
for EVENT_TYPE in "order.created" "order.paid" "order.shipped"; do
  RULE_NAME="${EVENT_TYPE//./-}-rule"
  aws --endpoint-url="$LOCALSTACK_URL" events put-rule \
    --name "$RULE_NAME" \
    --event-pattern "{\"source\":[\"express-api\"],\"detail-type\":[\"$EVENT_TYPE\"]}" \
    --region us-east-1 &>/dev/null
  aws --endpoint-url="$LOCALSTACK_URL" events put-targets \
    --rule "$RULE_NAME" \
    --targets "[{\"Id\":\"queue\",\"Arn\":\"${QUEUE_ARN}\"}]" \
    --region us-east-1 &>/dev/null
  echo "  ✔ Rule: $RULE_NAME → $QUEUE_NAME"
done
```

---

## 12.4 – Test the full local flow

```bash
# Create an order → triggers order.created
curl -X POST http://localhost:3000/api/orders \
  -H 'Content-Type: application/json' \
  -d '{"customerName":"Alice","product":"Widget","quantity":3}'

# Pay the order (use the id returned above, e.g. 1) → triggers order.paid
curl -X PATCH http://localhost:3000/api/orders/1/pay

# Ship the order → triggers order.shipped
curl -X PATCH http://localhost:3000/api/orders/1/ship

# Check SQS — should have received 3 messages
aws --endpoint-url=http://localhost:4566 sqs get-queue-attributes \
  --queue-url http://sqs.us-east-1.localhost.localstack.cloud:4566/000000000000/order-events \
  --attribute-names ApproximateNumberOfMessages
```

---

## 12.5 – Provision three rules with CDK

> **🤖 Ask your AI assistant:**
> ```
> In lib/events-stack.ts, create three EventBridge Rules targeting the same SQS queue:
> 1. "order-created-rule"  — source: "express-api", detail-type: "order.created"
> 2. "order-paid-rule"     — source: "express-api", detail-type: "order.paid"
> 3. "order-shipped-rule"  — source: "express-api", detail-type: "order.shipped"
> Use SqsQueue as the target for each rule (from @aws-cdk/aws-events-targets).
> ```

```bash
cd my-app-infra && cdk deploy EventsStack-dev
```

---

## Next step

➡️ [Step 13 – Lambda](step-13-lambda.md)
