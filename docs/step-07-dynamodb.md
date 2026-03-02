# Step 7 – DynamoDB

Create a DynamoDB table for storing processed events, provision it with CDK, and develop locally against DynamoDB Local.

---

## 7.1 – Understand DynamoDB concepts

| Concept | Description |
|---------|-------------|
| **Table** | Top-level resource that stores items |
| **Item** | A single record (like a row) |
| **Partition key** | Primary key attribute used to distribute data |
| **Sort key** | Optional second part of the primary key for range queries |
| **Attribute** | A name-value pair on an item (schema-less except keys) |
| **On-demand** | Pay per request — good for unpredictable workloads |
| **Provisioned** | Fixed read/write capacity — good for steady workloads |

---

## 7.2 – Design the ProcessedEvents table

| Attribute | Type | Key |
|-----------|------|-----|
| `id` | String | Partition key |
| `orderId` | Number | Sort key |
| `type` | String | — |
| `processedAt` | String (ISO 8601) | — |

---

## 7.3 – Create the table locally with DynamoDB Local

Make sure DynamoDB Local is running (see [Step 4](step-04-docker-local-env.md)):

```bash
docker compose up dynamodb-local
```

Create the table:

```bash
aws --endpoint-url=http://localhost:8000 dynamodb create-table \
  --table-name ProcessedEvents \
  --attribute-definitions \
    AttributeName=id,AttributeType=S \
    AttributeName=orderId,AttributeType=N \
  --key-schema \
    AttributeName=id,KeyType=HASH \
    AttributeName=orderId,KeyType=RANGE \
  --billing-mode PAY_PER_REQUEST \
  --region us-east-1
```

Verify:

```bash
aws --endpoint-url=http://localhost:8000 dynamodb list-tables --region us-east-1
```

Insert a test item:

```bash
aws --endpoint-url=http://localhost:8000 dynamodb put-item \
  --table-name ProcessedEvents \
  --item '{
    "id": {"S": "evt-001"},
    "orderId": {"N": "1"},
    "type": {"S": "order.created"},
    "processedAt": {"S": "2025-01-01T00:00:00Z"}
  }' \
  --region us-east-1
```

Scan the table:

```bash
aws --endpoint-url=http://localhost:8000 dynamodb scan \
  --table-name ProcessedEvents \
  --region us-east-1
```

---

## 7.4 – Provision the table with CDK

Add to `lib/infra-stack.ts` in your CDK project (from [Step 6](step-06-aws-cdk-infra.md)):

```ts
import * as dynamodb from 'aws-cdk-lib/aws-dynamodb';

// Inside the InfraStack constructor:
const eventsTable = new dynamodb.Table(this, 'ProcessedEventsTable', {
  tableName: 'ProcessedEvents',
  partitionKey: { name: 'id', type: dynamodb.AttributeType.STRING },
  sortKey: { name: 'orderId', type: dynamodb.AttributeType.NUMBER },
  billingMode: dynamodb.BillingMode.PAY_PER_REQUEST,
  removalPolicy: cdk.RemovalPolicy.DESTROY,
});

new cdk.CfnOutput(this, 'EventsTableName', {
  value: eventsTable.tableName,
});
```

Deploy:

```bash
cd infra
cdk deploy
```

---

## 7.5 – Verify the NestJS integration

Start the NestJS API (from [Step 5](step-05-nest-api.md)) pointing at DynamoDB Local:

```bash
DYNAMODB_ENDPOINT=http://localhost:8000 npm run start:dev
```

Test the events endpoint:

```bash
curl http://localhost:3001/api/events
```

You should see the test item you inserted in section 7.3.

---

## Next step

➡️ [Step 8 – SQS](step-08-sqs.md)
