# Step 10 – Lambda

Create a Lambda function that is triggered by the SQS queue and writes processed events to DynamoDB.

---

## 10.1 – Understand the role of this Lambda

```
EventBridge → SQS → Lambda → DynamoDB
```

When an `order.created` event lands in the SQS queue (from [Step 9](step-09-eventbridge.md)), this Lambda function:

1. Reads the message from the SQS batch.
2. Extracts the order details.
3. Writes a processed-event record to the `ProcessedEvents` DynamoDB table.

---

## 10.2 – Scaffold the Lambda project

```bash
mkdir order-processor-lambda && cd order-processor-lambda
npm init -y
npm install @aws-sdk/client-dynamodb @aws-sdk/lib-dynamodb uuid
npm install --save-dev typescript @types/node @types/aws-lambda @types/uuid
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

---

## 10.3 – Write the handler

Create `src/handler.ts`:

```ts
import { SQSEvent } from 'aws-lambda';
import { DynamoDBClient } from '@aws-sdk/client-dynamodb';
import { DynamoDBDocumentClient, PutCommand } from '@aws-sdk/lib-dynamodb';
import { v4 as uuidv4 } from 'uuid';

const client = DynamoDBDocumentClient.from(new DynamoDBClient({}));
const TABLE_NAME = process.env.EVENTS_TABLE ?? 'ProcessedEvents';

export const handler = async (event: SQSEvent): Promise<void> => {
  for (const record of event.Records) {
    const body = JSON.parse(record.body);

    // EventBridge wraps the payload in a 'detail' field
    const detail = body.detail ?? body;

    const item = {
      id: uuidv4(),
      orderId: detail.orderId,
      type: detail.type ?? 'order.created',
      processedAt: new Date().toISOString(),
    };

    await client.send(
      new PutCommand({
        TableName: TABLE_NAME,
        Item: item,
      }),
    );

    console.log('Processed event:', item);
  }
};
```

---

## 10.4 – Build the Lambda

Add to `package.json`:

```json
{
  "scripts": {
    "build": "tsc",
    "package": "npm run build && cd dist && zip -r ../function.zip ."
  }
}
```

Build:

```bash
npm run build
```

---

## 10.5 – Test locally with LocalStack

Make sure LocalStack is running with the SQS queue and DynamoDB Local table created (Steps 4, 7, 8).

Deploy the Lambda to LocalStack:

```bash
# Package the function
cd order-processor-lambda
npm run build
cd dist && zip -r ../function.zip . && cd ..

# Create the function
aws --endpoint-url=http://localhost:4566 lambda create-function \
  --function-name order-processor \
  --runtime nodejs22.x \
  --handler handler.handler \
  --zip-file fileb://function.zip \
  --role arn:aws:iam::000000000000:role/lambda-role \
  --environment "Variables={EVENTS_TABLE=ProcessedEvents,AWS_REGION=us-east-1}" \
  --region us-east-1

# Create event source mapping (SQS → Lambda)
QUEUE_ARN=$(aws --endpoint-url=http://localhost:4566 sqs get-queue-attributes \
  --queue-url http://sqs.us-east-1.localhost.localstack.cloud:4566/000000000000/order-events \
  --attribute-names QueueArn \
  --query 'Attributes.QueueArn' --output text)

aws --endpoint-url=http://localhost:4566 lambda create-event-source-mapping \
  --function-name order-processor \
  --event-source-arn "$QUEUE_ARN" \
  --batch-size 10 \
  --region us-east-1
```

Test the full flow:

```bash
# Send a message to SQS
aws --endpoint-url=http://localhost:4566 sqs send-message \
  --queue-url http://sqs.us-east-1.localhost.localstack.cloud:4566/000000000000/order-events \
  --message-body '{"detail":{"orderId":1,"type":"order.created"}}'

# Wait a few seconds, then check DynamoDB
aws --endpoint-url=http://localhost:8000 dynamodb scan \
  --table-name ProcessedEvents \
  --region us-east-1
```

---

## 10.6 – Provision Lambda with CDK

Add to `lib/infra-stack.ts`:

```ts
import * as lambda from 'aws-cdk-lib/aws-lambda';
import * as lambdaEventSources from 'aws-cdk-lib/aws-lambda-event-sources';

// Inside the InfraStack constructor (after creating eventsTable and orderEventsQueue):
const orderProcessor = new lambda.Function(this, 'OrderProcessorFn', {
  functionName: 'order-processor',
  runtime: lambda.Runtime.NODEJS_22_X,
  handler: 'handler.handler',
  code: lambda.Code.fromAsset('../order-processor-lambda/dist'),
  environment: {
    EVENTS_TABLE: eventsTable.tableName,
  },
});

// Grant the Lambda write access to the DynamoDB table
eventsTable.grantWriteData(orderProcessor);

// Trigger Lambda from SQS
orderProcessor.addEventSource(
  new lambdaEventSources.SqsEventSource(orderEventsQueue, {
    batchSize: 10,
  }),
);

new cdk.CfnOutput(this, 'OrderProcessorFnName', {
  value: orderProcessor.functionName,
});
```

Deploy:

```bash
cd infra && cdk deploy
```

---

## 10.7 – Test the full end-to-end flow

1. Create an order via the frontend or curl:

   ```bash
   curl -X POST http://localhost:3000/api/orders \
     -H 'Content-Type: application/json' \
     -d '{"customerName":"Bob","product":"Gadget","quantity":5}'
   ```

2. The Express API saves the order to PostgreSQL and publishes an `order.created` event to EventBridge.

3. EventBridge routes the event to the SQS queue.

4. The Lambda function consumes the SQS message and writes a processed-event record to DynamoDB.

5. The NestJS API reads from DynamoDB and serves the processed event via `GET /api/events`.

6. The frontend Events page displays the processed event.

---

## Next step

🎉 Congratulations — you have built a complete event-driven full-stack system! Go back to the [main guide](../README.md) for the architecture overview and links to all steps.
