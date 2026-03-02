# Step 9 – SQS

Set up an Amazon SQS queue for asynchronous messaging between services, with a dead-letter queue for failed messages.

---

## 9.1 – SQS concepts

| Concept | Description |
|---------|-------------|
| **Queue** | Managed message buffer |
| **Standard queue** | At-least-once delivery, best-effort ordering |
| **FIFO queue** | Exactly-once processing, strict ordering |
| **Producer** | Any service that sends a message |
| **Consumer** | Service that reads and processes messages |
| **Visibility timeout** | How long a message is hidden after being received (prevents duplicate processing) |
| **Dead-letter queue (DLQ)** | Destination for messages that repeatedly fail processing |

> **📖 Why a Dead-Letter Queue?**
> If a consumer fails to process a message (e.g. the database is down, or the payload is malformed), SQS makes the message visible again after the visibility timeout expires, and another consumer retries it. After `maxReceiveCount` retries the message moves to the **DLQ** instead of disappearing silently. This gives you a safe place to inspect failed messages, fix the bug, and re-process them manually.

> **📖 What is visibility timeout?**
> When a consumer receives a message, SQS hides it from other consumers for the duration of the visibility timeout. If the consumer processes and deletes it in time — great. If not (e.g. the consumer crashes), the message becomes visible again and another consumer can pick it up. Set the timeout slightly longer than your Lambda or service needs to process one message.

> **📖 What is long polling?**
> Instead of constantly asking SQS “are there any messages?” (short poll), long polling keeps the connection open for up to 20 seconds waiting for a message to arrive. This reduces empty responses, lowers cost, and decreases latency. `--wait-time-seconds 5` in the CLI commands below enables long polling.

---

## 9.2 – Create the queue locally with LocalStack

Start LocalStack:

```bash
docker compose up localstack
```

```bash
export AWS_ACCESS_KEY_ID=test
export AWS_SECRET_ACCESS_KEY=test
export AWS_DEFAULT_REGION=us-east-1

# Create the DLQ first
aws --endpoint-url=http://localhost:4566 sqs create-queue \
  --queue-name order-events-dlq

# Get the DLQ ARN
DLQ_ARN=$(aws --endpoint-url=http://localhost:4566 sqs get-queue-attributes \
  --queue-url http://sqs.us-east-1.localhost.localstack.cloud:4566/000000000000/order-events-dlq \
  --attribute-names QueueArn \
  --query 'Attributes.QueueArn' --output text)

# Create the main queue with a redrive policy (max 3 retries before DLQ)
aws --endpoint-url=http://localhost:4566 sqs create-queue \
  --queue-name order-events \
  --attributes "{\"RedrivePolicy\":\"{\\\"deadLetterTargetArn\\\":\\\"${DLQ_ARN}\\\",\\\"maxReceiveCount\\\":\\\"3\\\"}\"}"
```

Verify:

```bash
aws --endpoint-url=http://localhost:4566 sqs list-queues
```

> **📈 Paste into `setup-local.sh` (Step 11 block)**
> Add this after the DynamoDB block from Step 10.

```bash

QUEUE_NAME="order-events"
DLQ_NAME="order-events-dlq"

echo "▶ Creating SQS queues..."
aws --endpoint-url="$LOCALSTACK_URL" sqs create-queue \
  --queue-name "$DLQ_NAME" --region us-east-1 2>/dev/null && echo "  ✔ DLQ created" || echo "  ⚠ DLQ exists"

DLQ_ARN=$(aws --endpoint-url="$LOCALSTACK_URL" sqs get-queue-attributes \
  --queue-url "$LOCALSTACK_URL/000000000000/$DLQ_NAME" \
  --attribute-names QueueArn --query 'Attributes.QueueArn' --output text)

aws --endpoint-url="$LOCALSTACK_URL" sqs create-queue \
  --queue-name "$QUEUE_NAME" \
  --attributes "{\"RedrivePolicy\":\"{\\\"deadLetterTargetArn\\\":\\\"${DLQ_ARN}\\\",\\\"maxReceiveCount\\\":\\\"3\\\"}\"}"\
  --region us-east-1 2>/dev/null && echo "  ✔ Main queue created" || echo "  ⚠ Queue exists"

QUEUE_URL=$(aws --endpoint-url="$LOCALSTACK_URL" sqs get-queue-url \
  --queue-name "$QUEUE_NAME" --query QueueUrl --output text)
QUEUE_ARN=$(aws --endpoint-url="$LOCALSTACK_URL" sqs get-queue-attributes \
  --queue-url "$QUEUE_URL" --attribute-names QueueArn \
  --query 'Attributes.QueueArn' --output text)
echo "  ✔ Queue URL: $QUEUE_URL"
```

---

## 9.3 – Test send and receive with the AWS CLI

```bash
QUEUE_URL=$(aws --endpoint-url=http://localhost:4566 sqs get-queue-url \
  --queue-name order-events --query QueueUrl --output text)

# Send a message
aws --endpoint-url=http://localhost:4566 sqs send-message \
  --queue-url "$QUEUE_URL" \
  --message-body '{"type":"order.created","orderId":42}'

# Receive messages (long-poll up to 5 s)
aws --endpoint-url=http://localhost:4566 sqs receive-message \
  --queue-url "$QUEUE_URL" \
  --wait-time-seconds 5
```

---

## 9.4 – Implement the SQS producer in Express API

```bash
cd my-app-express-api
npm install @aws-sdk/client-sqs
```

> **🤖 Ask your AI assistant:**
> ```
> Create src/sqs/producer.ts in the Express API.
> It should export a publishOrderEvent(type: string, orderId: number) function
> that sends a JSON message to the SQS queue URL from process.env.SQS_QUEUE_URL
> using @aws-sdk/client-sqs. Support an optional SQS_ENDPOINT env var for LocalStack.
> ```

Call `publishOrderEvent` from the `POST /api/orders` handler after creating the order.

---

## 9.5 – Provision SQS with CDK

> **🤖 Ask your AI assistant:**
> ```
> In lib/events-stack.ts, create two SQS queues:
> 1. A DLQ named "order-events-dlq" with retentionPeriod of 14 days.
> 2. A main queue named "order-events" with visibilityTimeout 60 seconds
>    and a dead-letter queue pointing to the DLQ with maxReceiveCount 3.
> Export the main queue URL and ARN using CfnOutput.
> ```

Deploy:

```bash
cd my-app-infra && cdk deploy EventsStack-dev
```

---

## Next step

➡️ [Step 12 – EventBridge](step-12-eventbridge.md)
