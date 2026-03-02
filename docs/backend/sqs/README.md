# Backend – Amazon SQS

A step-by-step guide to sending and receiving messages with Amazon SQS, integrating it with Lambda and NestJS, and testing locally with LocalStack.

---

## Step 1 – Understand SQS concepts

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

## Step 2 – Create a queue with the AWS CLI

```bash
# Standard queue
aws sqs create-queue --queue-name my-queue

# FIFO queue (name must end with .fifo)
aws sqs create-queue \
  --queue-name my-queue.fifo \
  --attributes FifoQueue=true,ContentBasedDeduplication=true
```

Get the queue URL (needed for all operations):

```bash
aws sqs get-queue-url --queue-name my-queue
```

---

## Step 3 – Create the queue locally with LocalStack

Start LocalStack as described in [docs/infra](../../infra/README.md), then use the same AWS CLI commands with a custom endpoint:

```bash
export AWS_DEFAULT_REGION=us-east-1
export AWS_ACCESS_KEY_ID=test
export AWS_SECRET_ACCESS_KEY=test

aws --endpoint-url=http://localhost:4566 sqs create-queue --queue-name my-queue
```

You can add an alias to simplify repeated use:

```bash
alias awslocal='aws --endpoint-url=http://localhost:4566'
awslocal sqs list-queues
```

---

## Step 4 – Send and receive messages with the AWS CLI

```bash
# Send
aws sqs send-message \
  --queue-url https://sqs.us-east-1.amazonaws.com/123456789/my-queue \
  --message-body '{"event":"order.created","orderId":42}'

# Receive (long-poll for up to 20 s)
aws sqs receive-message \
  --queue-url https://sqs.us-east-1.amazonaws.com/123456789/my-queue \
  --wait-time-seconds 20

# Delete after successful processing
aws sqs delete-message \
  --queue-url https://sqs.us-east-1.amazonaws.com/123456789/my-queue \
  --receipt-handle <ReceiptHandle from receive-message>
```

---

## Step 5 – Produce messages from Node.js

```bash
npm install @aws-sdk/client-sqs
```

`src/sqs/producer.ts`:

```ts
import { SendMessageCommand, SQSClient } from '@aws-sdk/client-sqs';

const client = new SQSClient({
  region: process.env.AWS_REGION ?? 'us-east-1',
  // Point to LocalStack in development
  ...(process.env.SQS_ENDPOINT
    ? { endpoint: process.env.SQS_ENDPOINT }
    : {}),
});

const QUEUE_URL = process.env.SQS_QUEUE_URL!;

export async function sendMessage(payload: Record<string, unknown>): Promise<void> {
  await client.send(
    new SendMessageCommand({
      QueueUrl: QUEUE_URL,
      MessageBody: JSON.stringify(payload),
    }),
  );
}
```

Usage:

```ts
import { sendMessage } from './sqs/producer';

await sendMessage({ event: 'order.created', orderId: 42 });
```

---

## Step 6 – Consume messages from Node.js (polling loop)

`src/sqs/consumer.ts`:

```ts
import {
  DeleteMessageCommand,
  ReceiveMessageCommand,
  SQSClient,
} from '@aws-sdk/client-sqs';

const client = new SQSClient({
  region: process.env.AWS_REGION ?? 'us-east-1',
  ...(process.env.SQS_ENDPOINT
    ? { endpoint: process.env.SQS_ENDPOINT }
    : {}),
});

const QUEUE_URL = process.env.SQS_QUEUE_URL!;

async function processMessage(body: string): Promise<void> {
  const payload = JSON.parse(body) as Record<string, unknown>;
  console.log('Processing message:', payload);
  // Add your business logic here
}

export async function startPolling(): Promise<void> {
  console.log('SQS consumer started');
  for (;;) {
    const { Messages = [] } = await client.send(
      new ReceiveMessageCommand({
        QueueUrl: QUEUE_URL,
        MaxNumberOfMessages: 10,
        WaitTimeSeconds: 20, // long polling
      }),
    );

    for (const message of Messages) {
      try {
        await processMessage(message.Body!);
        await client.send(
          new DeleteMessageCommand({
            QueueUrl: QUEUE_URL,
            ReceiptHandle: message.ReceiptHandle,
          }),
        );
      } catch (err) {
        console.error('Failed to process message, leaving in queue:', err);
        // The visibility timeout will expire and the message will be redelivered
      }
    }
  }
}
```

---

## Step 7 – Integrate SQS with NestJS

```bash
cd nest-api
npm install @aws-sdk/client-sqs
```

Generate a module:

```bash
nest generate module sqs
nest generate service sqs
```

`src/sqs/sqs.service.ts`:

```ts
import { Injectable, OnModuleInit, OnModuleDestroy } from '@nestjs/common';
import {
  DeleteMessageCommand,
  ReceiveMessageCommand,
  SendMessageCommand,
  SQSClient,
} from '@aws-sdk/client-sqs';

@Injectable()
export class SqsService implements OnModuleInit, OnModuleDestroy {
  private readonly client: SQSClient;
  private polling = true;

  constructor() {
    this.client = new SQSClient({
      region: process.env.AWS_REGION ?? 'us-east-1',
      ...(process.env.SQS_ENDPOINT
        ? { endpoint: process.env.SQS_ENDPOINT }
        : {}),
    });
  }

  async send(payload: Record<string, unknown>): Promise<void> {
    await this.client.send(
      new SendMessageCommand({
        QueueUrl: process.env.SQS_QUEUE_URL!,
        MessageBody: JSON.stringify(payload),
      }),
    );
  }

  onModuleInit(): void {
    this.poll().catch(console.error);
  }

  onModuleDestroy(): void {
    this.polling = false;
  }

  private async poll(): Promise<void> {
    while (this.polling) {
      const { Messages = [] } = await this.client.send(
        new ReceiveMessageCommand({
          QueueUrl: process.env.SQS_QUEUE_URL!,
          MaxNumberOfMessages: 10,
          WaitTimeSeconds: 20,
        }),
      );

      for (const msg of Messages) {
        await this.handleMessage(msg.Body!);
        await this.client.send(
          new DeleteMessageCommand({
            QueueUrl: process.env.SQS_QUEUE_URL!,
            ReceiptHandle: msg.ReceiptHandle,
          }),
        );
      }
    }
  }

  private async handleMessage(body: string): Promise<void> {
    const payload = JSON.parse(body) as Record<string, unknown>;
    console.log('Received SQS message:', payload);
    // Dispatch to the appropriate handler
  }
}
```

---

## Step 8 – Trigger Lambda from SQS

Configure the Lambda event source in `serverless.yml` (see [docs/backend/lambda](../lambda/README.md)):

```yaml
functions:
  processOrder:
    handler: src/functions/processOrder/handler.main
    events:
      - sqs:
          arn: !GetAtt OrderQueue.Arn
          batchSize: 10
          maximumBatchingWindow: 5   # max seconds Lambda waits to fill a batch before invoking; higher = more efficient, higher latency

resources:
  Resources:
    OrderQueue:
      Type: AWS::SQS::Queue
      Properties:
        QueueName: order-queue-${sls:stage}
        VisibilityTimeout: 60
        RedrivePolicy:
          deadLetterTargetArn: !GetAtt OrderDLQ.Arn
          maxReceiveCount: 3

    OrderDLQ:
      Type: AWS::SQS::Queue
      Properties:
        QueueName: order-dlq-${sls:stage}
```

Lambda handler for SQS events:

```ts
import { SQSEvent } from 'aws-lambda';

export const main = async (event: SQSEvent): Promise<void> => {
  for (const record of event.Records) {
    const payload = JSON.parse(record.body) as Record<string, unknown>;
    console.log('Processing order:', payload);
    // Business logic here
  }
};
```

---

## Step 9 – Add a Dead-Letter Queue (DLQ)

A DLQ captures messages that failed processing after `maxReceiveCount` attempts. Monitor it with a CloudWatch alarm or an additional Lambda consumer.

```bash
aws sqs create-queue --queue-name my-dlq

aws sqs set-queue-attributes \
  --queue-url <MAIN_QUEUE_URL> \
  --attributes '{
    "RedrivePolicy": "{\"deadLetterTargetArn\":\"<DLQ_ARN>\",\"maxReceiveCount\":\"3\"}"
  }'
```

---

## Step 10 – Provision SQS with Terraform

`infra/sqs.tf`:

```hcl
resource "aws_sqs_queue" "orders_dlq" {
  name                      = "orders-dlq-${var.environment}"
  message_retention_seconds = 1209600 # 14 days
}

resource "aws_sqs_queue" "orders" {
  name                       = "orders-${var.environment}"
  visibility_timeout_seconds = 60
  redrive_policy = jsonencode({
    deadLetterTargetArn = aws_sqs_queue.orders_dlq.arn
    maxReceiveCount     = 3
  })
}

output "orders_queue_url" {
  value = aws_sqs_queue.orders.url
}
```

Apply:

```bash
cd infra
terraform apply
```

---

## Next steps

- Review the full infrastructure setup → [docs/infra](../../infra/README.md)
- Go back to the main guide → [README](../../../README.md)
