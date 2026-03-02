# Step 13 – Lambda (Order Processor)

Create a Lambda function that is triggered by the SQS queue, handles all three event types, and writes records to DynamoDB using the single-table access pattern from Step 10.

> **Repository:** work inside your `my-app-lambda` repo for this step.

> **📖 What is AWS Lambda?**
> Lambda is a **serverless compute** service: you upload code, AWS runs it when triggered, and you pay only for the milliseconds it executes. Lambda functions are stateless — each invocation is independent. They are ideal for event-driven tasks like processing a queue message.

> **📖 What is a cold start?**
> The first invocation (or after a period of inactivity) requires AWS to download your code and start a runtime process. This takes 100ms–1s. Subsequent invocations reuse the same process (**warm start**) and are much faster.

---

## 13.1 – Role of this function

```
EventBridge → SQS → Lambda → DynamoDB (OrderEvents)
```

For every SQS record, the handler:
1. Parses the EventBridge-wrapped payload from the SQS message body
2. Determines the event type (`order.created`, `order.paid`, or `order.shipped`)
3. Writes a record to `OrderEvents` using the composite key `PK = ORDER#<orderId>` / `SK = EVENT#<eventType>#<timestamp>`

> **📖 What is an Event Source Mapping?**
> AWS manages a long-polling loop: it continuously reads from SQS, batches messages, and invokes your Lambda with an `SQSEvent`. If the handler throws, the messages are **not deleted** — they are retried (up to `maxReceiveCount`) then moved to the DLQ.

---

## 13.2 – Scaffold the project

```bash
npm init -y
npm install @aws-sdk/client-dynamodb @aws-sdk/lib-dynamodb
npm install --save-dev typescript @types/node @types/aws-lambda
npx tsc --init
```

---

## 13.3 – Write the handler

> **🤖 Ask your AI assistant:**
> ```
> Create src/handler.ts: a Lambda handler for SQSEvent.
> For each SQS record:
>   1. Parse the record body as JSON.
>   2. Extract the EventBridge fields: detail-type (into eventType) and detail (into { orderId, timestamp }).
>   3. Compose the DynamoDB item:
>      PK = "ORDER#" + orderId
>      SK = "EVENT#" + eventType + "#" + timestamp
>      orderId (Number)
>      eventType (String)
>      processedAt = timestamp
>   4. Write the item using PutCommand to the table in process.env.EVENTS_TABLE.
> Log each processed record with orderId and eventType.
> If any record fails, throw so SQS retries it.
> ```

---

## 13.4 – Build and package

> **🤖 Ask your AI assistant:**
> ```
> Add two scripts to package.json:
> "build": "tsc"
> "package": "npm run build && cd dist && zip -r ../function.zip ."
> ```

```bash
npm run build
```

---

## 13.5 – Test locally with LocalStack

```bash
npm run package

# Deploy function to LocalStack
aws --endpoint-url=http://localhost:4566 lambda create-function \
  --function-name order-processor \
  --runtime nodejs22.x \
  --handler handler.handler \
  --zip-file fileb://function.zip \
  --role arn:aws:iam::000000000000:role/lambda-role \
  --environment "Variables={EVENTS_TABLE=OrderEvents,AWS_REGION=us-east-1}" \
  --region us-east-1

# Attach SQS trigger
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

> **📈 Paste into `setup-local.sh` (Step 13 block — final)**
> Add this after the EventBridge block from Step 12. Run `npm run package` in `my-app-lambda` first.

```bash

# Requires: cd my-app-lambda && npm run package
LAMBDA_NAME="order-processor"
LAMBDA_ZIP="$(pwd)/../my-app-lambda/function.zip"

echo "▶ Deploying Lambda: $LAMBDA_NAME ..."
aws --endpoint-url="$LOCALSTACK_URL" lambda create-function \
  --function-name "$LAMBDA_NAME" \
  --runtime nodejs22.x \
  --handler handler.handler \
  --zip-file "fileb://$LAMBDA_ZIP" \
  --role "arn:aws:iam::000000000000:role/lambda-role" \
  --environment "Variables={EVENTS_TABLE=$TABLE_NAME,AWS_REGION=us-east-1,DYNAMODB_ENDPOINT=$DYNAMODB_URL}" \
  --region us-east-1 2>/dev/null && echo "  ✔ Lambda created" || \
  (aws --endpoint-url="$LOCALSTACK_URL" lambda update-function-code \
    --function-name "$LAMBDA_NAME" --zip-file "fileb://$LAMBDA_ZIP" \
    --region us-east-1 &>/dev/null && echo "  ✔ Lambda updated")

aws --endpoint-url="$LOCALSTACK_URL" lambda create-event-source-mapping \
  --function-name "$LAMBDA_NAME" \
  --event-source-arn "$QUEUE_ARN" \
  --batch-size 10 \
  --region us-east-1 2>/dev/null && echo "  ✔ SQS trigger attached" || echo "  ⚠ Trigger exists"

echo ""; echo "✅ Local environment ready."
```

Test all three event types:

```bash
# Simulate order.created
aws --endpoint-url=http://localhost:4566 sqs send-message \
  --queue-url http://sqs.us-east-1.localhost.localstack.cloud:4566/000000000000/order-events \
  --message-body '{
    "source":"express-api",
    "detail-type":"order.created",
    "detail":{"orderId":1,"timestamp":"2024-01-15T10:30:00Z"}
  }'

# Simulate order.paid
aws --endpoint-url=http://localhost:4566 sqs send-message \
  --queue-url http://sqs.us-east-1.localhost.localstack.cloud:4566/000000000000/order-events \
  --message-body '{
    "source":"express-api",
    "detail-type":"order.paid",
    "detail":{"orderId":1,"timestamp":"2024-01-15T10:35:00Z"}
  }'

# Simulate order.shipped
aws --endpoint-url=http://localhost:4566 sqs send-message \
  --queue-url http://sqs.us-east-1.localhost.localstack.cloud:4566/000000000000/order-events \
  --message-body '{
    "source":"express-api",
    "detail-type":"order.shipped",
    "detail":{"orderId":1,"timestamp":"2024-01-15T10:40:00Z"}
  }'
```

Query DynamoDB — should return 3 items for ORDER#1, no scan:

```bash
aws --endpoint-url=http://localhost:8000 dynamodb query \
  --table-name OrderEvents \
  --key-condition-expression "PK = :pk" \
  --expression-attribute-values '{":pk":{"S":"ORDER#1"}}' \
  --region us-east-1
```

Expected output: three items sorted by SK (chronological):
```
EVENT#order.created#2024-01-15T10:30:00Z
EVENT#order.paid#2024-01-15T10:35:00Z
EVENT#order.shipped#2024-01-15T10:40:00Z
```

---

## 13.6 – Provision Lambda with CDK

> **🤖 Ask your AI assistant:**
> ```
> In lib/compute-stack.ts, create a Lambda function named "order-processor"
> with runtime NODEJS_22_X, handler "handler.handler",
> code from asset "../my-app-lambda/dist",
> environment variable EVENTS_TABLE = DynamoDB table name (imported from DataStack output).
> Grant the function write access to the table.
> Add SqsEventSource from the EventsStack queue as an event source with batchSize 10.
> Export the function ARN via CfnOutput.
> ```

```bash
cd my-app-infra && cdk deploy ComputeStack-dev
```

---

## 13.7 – Push to GitHub

```bash
git add .
git commit -m "feat: order-processor lambda — handles 3 event types with PK/SK pattern"
git remote add origin https://github.com/<YOUR_ORG>/my-app-lambda.git
git push -u origin main
```

---

## Next step

➡️ [Step 14 – CloudWatch Observability](step-14-cloudwatch.md)
