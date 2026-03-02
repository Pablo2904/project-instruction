# Step 8 – SQS

Set up an Amazon SQS queue for asynchronous messaging between services, with a dead-letter queue for failed messages.

---

## 8.1 – Understand SQS concepts

| Concept | Description |
|---------|-------------|
| **Queue** | Managed buffer of messages |
| **Standard queue** | At-least-once delivery, best-effort ordering |
| **FIFO queue** | Exactly-once processing, strict ordering |
| **Producer** | Any service that sends a message to the queue |
| **Consumer** | Service that reads and processes messages |
| **Visibility timeout** | Time a message is hidden after being received (prevents duplicate processing) |
| **Dead-letter queue (DLQ)** | Destination for messages that repeatedly fail processing |

---

## 8.2 – Create a queue locally with LocalStack

Start LocalStack (see [Step 4](step-04-docker-local-env.md)):

```bash
docker compose up localstack
```

Create the queue and DLQ:

```bash
export AWS_ACCESS_KEY_ID=test
export AWS_SECRET_ACCESS_KEY=test
export AWS_DEFAULT_REGION=us-east-1

# Create the DLQ first
aws --endpoint-url=http://localhost:4566 sqs create-queue \
  --queue-name order-events-dlq

# Get DLQ ARN
DLQ_ARN=$(aws --endpoint-url=http://localhost:4566 sqs get-queue-attributes \
  --queue-url http://sqs.us-east-1.localhost.localstack.cloud:4566/000000000000/order-events-dlq \
  --attribute-names QueueArn \
  --query 'Attributes.QueueArn' --output text)

# Create the main queue with redrive policy
aws --endpoint-url=http://localhost:4566 sqs create-queue \
  --queue-name order-events \
  --attributes "{\"RedrivePolicy\":\"{\\\"deadLetterTargetArn\\\":\\\"${DLQ_ARN}\\\",\\\"maxReceiveCount\\\":\\\"3\\\"}\"}"
```

Verify:

```bash
aws --endpoint-url=http://localhost:4566 sqs list-queues
```

---

## 8.3 – Send and receive messages with the AWS CLI

```bash
# Get the queue URL
QUEUE_URL=$(aws --endpoint-url=http://localhost:4566 sqs get-queue-url \
  --queue-name order-events --query QueueUrl --output text)

# Send a message
aws --endpoint-url=http://localhost:4566 sqs send-message \
  --queue-url "$QUEUE_URL" \
  --message-body '{"type":"order.created","orderId":42}'

# Receive messages (long-poll for up to 5 s)
aws --endpoint-url=http://localhost:4566 sqs receive-message \
  --queue-url "$QUEUE_URL" \
  --wait-time-seconds 5
```

---

## 8.4 – Produce messages from Node.js

In the Express API (`express-api/`):

```bash
npm install @aws-sdk/client-sqs
```

Create `src/sqs/producer.ts`:

```ts
import { SendMessageCommand, SQSClient } from '@aws-sdk/client-sqs';

const client = new SQSClient({
  region: process.env.AWS_REGION ?? 'us-east-1',
  ...(process.env.SQS_ENDPOINT ? { endpoint: process.env.SQS_ENDPOINT } : {}),
});

const QUEUE_URL = process.env.SQS_QUEUE_URL!;

export async function publishOrderEvent(
  type: string,
  orderId: number,
): Promise<void> {
  await client.send(
    new SendMessageCommand({
      QueueUrl: QUEUE_URL,
      MessageBody: JSON.stringify({ type, orderId, timestamp: new Date().toISOString() }),
    }),
  );
}
```

---

## 8.5 – Consume messages from Node.js (polling loop)

This pattern is useful for long-running services:

```ts
import {
  DeleteMessageCommand,
  ReceiveMessageCommand,
  SQSClient,
} from '@aws-sdk/client-sqs';

const client = new SQSClient({
  region: process.env.AWS_REGION ?? 'us-east-1',
  ...(process.env.SQS_ENDPOINT ? { endpoint: process.env.SQS_ENDPOINT } : {}),
});

const QUEUE_URL = process.env.SQS_QUEUE_URL!;

export async function startPolling(): Promise<void> {
  console.log('SQS consumer started');
  for (;;) {
    const { Messages = [] } = await client.send(
      new ReceiveMessageCommand({
        QueueUrl: QUEUE_URL,
        MaxNumberOfMessages: 10,
        WaitTimeSeconds: 20,
      }),
    );

    for (const message of Messages) {
      try {
        const payload = JSON.parse(message.Body!);
        console.log('Processing:', payload);
        // Business logic here

        await client.send(
          new DeleteMessageCommand({
            QueueUrl: QUEUE_URL,
            ReceiptHandle: message.ReceiptHandle,
          }),
        );
      } catch (err) {
        console.error('Failed to process message:', err);
      }
    }
  }
}
```

---

## 8.6 – Provision SQS with CDK

Add to `lib/infra-stack.ts`:

```ts
import * as sqs from 'aws-cdk-lib/aws-sqs';

// Inside the InfraStack constructor:
const orderEventsDlq = new sqs.Queue(this, 'OrderEventsDLQ', {
  queueName: 'order-events-dlq',
  retentionPeriod: cdk.Duration.days(14),
});

const orderEventsQueue = new sqs.Queue(this, 'OrderEventsQueue', {
  queueName: 'order-events',
  visibilityTimeout: cdk.Duration.seconds(60),
  deadLetterQueue: {
    queue: orderEventsDlq,
    maxReceiveCount: 3,
  },
});

new cdk.CfnOutput(this, 'OrderEventsQueueUrl', {
  value: orderEventsQueue.queueUrl,
});

new cdk.CfnOutput(this, 'OrderEventsQueueArn', {
  value: orderEventsQueue.queueArn,
});
```

Deploy:

```bash
cd infra && cdk deploy
```

---

## Next step

➡️ [Step 9 – EventBridge](step-09-eventbridge.md)
