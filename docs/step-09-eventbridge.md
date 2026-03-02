# Step 9 – EventBridge

Set up Amazon EventBridge to route `order.created` events from the Express API to the SQS queue. EventBridge decouples the producer from the consumer — the Express API does not need to know about SQS.

---

## 9.1 – Understand EventBridge concepts

| Concept | Description |
|---------|-------------|
| **Event bus** | A channel that receives events (default bus is provided by AWS) |
| **Event** | A JSON object describing something that happened |
| **Rule** | Pattern-matching logic that routes matching events to one or more targets |
| **Target** | Destination for matched events (SQS, Lambda, SNS, Step Functions, etc.) |

---

## 9.2 – Create a rule locally with LocalStack

Start LocalStack (see [Step 4](step-04-docker-local-env.md)):

```bash
docker compose up localstack
```

Make sure the SQS queue exists (see [Step 8](step-08-sqs.md)):

```bash
export AWS_ACCESS_KEY_ID=test
export AWS_SECRET_ACCESS_KEY=test
export AWS_DEFAULT_REGION=us-east-1

# Get the SQS queue ARN
QUEUE_ARN=$(aws --endpoint-url=http://localhost:4566 sqs get-queue-attributes \
  --queue-url http://sqs.us-east-1.localhost.localstack.cloud:4566/000000000000/order-events \
  --attribute-names QueueArn \
  --query 'Attributes.QueueArn' --output text)
```

Create an EventBridge rule that matches `order.created` events:

```bash
aws --endpoint-url=http://localhost:4566 events put-rule \
  --name order-created-rule \
  --event-pattern '{
    "source": ["express-api"],
    "detail-type": ["order.created"]
  }'
```

Add the SQS queue as a target:

```bash
aws --endpoint-url=http://localhost:4566 events put-targets \
  --rule order-created-rule \
  --targets "[{\"Id\":\"order-events-queue\",\"Arn\":\"${QUEUE_ARN}\"}]"
```

---

## 9.3 – Publish events from the Express API

Install the EventBridge SDK in the Express API:

```bash
cd express-api
npm install @aws-sdk/client-eventbridge
```

Create `src/events/eventbridge.ts`:

```ts
import {
  EventBridgeClient,
  PutEventsCommand,
} from '@aws-sdk/client-eventbridge';

const client = new EventBridgeClient({
  region: process.env.AWS_REGION ?? 'us-east-1',
  ...(process.env.EVENTBRIDGE_ENDPOINT
    ? { endpoint: process.env.EVENTBRIDGE_ENDPOINT }
    : {}),
});

export async function publishOrderCreated(orderId: number): Promise<void> {
  await client.send(
    new PutEventsCommand({
      Entries: [
        {
          Source: 'express-api',
          DetailType: 'order.created',
          Detail: JSON.stringify({
            orderId,
            timestamp: new Date().toISOString(),
          }),
        },
      ],
    }),
  );
}
```

---

## 9.4 – Wire it into the Orders router

Update the `POST /api/orders` handler in `src/routes/orders.ts` (from [Step 3](step-03-express-api.md)):

```ts
import { publishOrderCreated } from '../events/eventbridge';

router.post('/', validate(createOrderSchema), async (req, res) => {
  const order = await prisma.order.create({ data: req.body });

  // Publish event to EventBridge
  await publishOrderCreated(order.id);

  res.status(201).json(order);
});
```

---

## 9.5 – Test the full flow locally

1. Start infrastructure:

   ```bash
   docker compose up postgres localstack
   ```

2. Create the SQS queue and EventBridge rule (sections 9.2 and 8.2).

3. Start the Express API:

   ```bash
   cd express-api
   EVENTBRIDGE_ENDPOINT=http://localhost:4566 \
   SQS_ENDPOINT=http://localhost:4566 \
   npm run dev
   ```

4. Create an order:

   ```bash
   curl -X POST http://localhost:3000/api/orders \
     -H 'Content-Type: application/json' \
     -d '{"customerName":"Alice","product":"Widget","quantity":3}'
   ```

5. Check the SQS queue for the event:

   ```bash
   aws --endpoint-url=http://localhost:4566 sqs receive-message \
     --queue-url http://sqs.us-east-1.localhost.localstack.cloud:4566/000000000000/order-events \
     --wait-time-seconds 5
   ```

   You should see the `order.created` event in the message body.

---

## 9.6 – Provision EventBridge with CDK

Add to `lib/infra-stack.ts`:

```ts
import * as events from 'aws-cdk-lib/aws-events';
import * as targets from 'aws-cdk-lib/aws-events-targets';

// Inside the InfraStack constructor (after creating orderEventsQueue):
const orderCreatedRule = new events.Rule(this, 'OrderCreatedRule', {
  ruleName: 'order-created-rule',
  eventPattern: {
    source: ['express-api'],
    detailType: ['order.created'],
  },
});

orderCreatedRule.addTarget(new targets.SqsQueue(orderEventsQueue));
```

Deploy:

```bash
cd infra && cdk deploy
```

---

## Next step

➡️ [Step 10 – Lambda](step-10-lambda.md)
