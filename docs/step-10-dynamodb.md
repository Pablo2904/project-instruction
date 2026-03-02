# Step 10 – DynamoDB (Single-Table Design)

Design and provision the DynamoDB table that stores all order lifecycle events, using a **Single-Table Design** with composite keys.

---

## 10.1 – DynamoDB concepts

| Concept | Description |
|---------|-------------|
| **Partition key (PK)** | Determines which partition (physical node) stores the item. Items with the same PK are co-located. |
| **Sort key (SK)** | Secondary key within a partition. Enables range queries and ordering within a partition. |
| **Composite key** | PK + SK together. The combination must be unique, but PK alone does not have to be. |
| **Query** | Efficient lookup: fetches all items matching a PK (and optionally an SK condition) — reads only the relevant partition. **Always prefer Query over Scan.** |
| **Scan** | Reads every item in the table — expensive, slow at scale. **Avoid Scan in production.** |
| **GSI (Global Secondary Index)** | An alternate key structure on different attributes, enabling additional Query patterns. |

---

## 10.2 – Access patterns first

> **📖 Why design access patterns before the schema?**
> In relational databases you normalise your tables and queries adapt. In DynamoDB it is the opposite: **define your access patterns first, then design your key schema to serve them efficiently**. Each Query must resolve to a single PK — the table schema is the answer to the question "what will I search by?".

This project's access patterns:

| # | Pattern | How |
|---|---------|-----|
| 1 | Get all events for a specific order | Query `PK = "ORDER#<id>"` |
| 2 | Get the latest event for a specific order | Query `PK = "ORDER#<id>"`, sort by SK descending, limit 1 |
| 3 | Get all events of a specific type across all orders | Query GSI `eventType-index` |

---

## 10.3 – Key schema

Single table `OrderEvents`:

| Attribute | Type | Role | Example |
|---|---|---|---|
| `PK` | String | Partition key | `ORDER#1` |
| `SK` | String | Sort key | `EVENT#order.created#2024-01-15T10:30:00Z` |
| `orderId` | Number | Duplicate of the id in PK (for readability/GSI) | `1` |
| `eventType` | String | Event name | `order.created` |
| `processedAt` | String | ISO 8601 timestamp | `2024-01-15T10:30:00Z` |

> **📖 Why `ORDER#` prefix on the PK?**
> This is a common single-table design convention. Prefixing keys with the entity type (e.g. `ORDER#`, `USER#`, `PRODUCT#`) prevents collisions when multiple entity types share one table and makes key patterns self-documenting. It also enables GSI patterns: set the GSI PK to the prefix and query all entities of that type efficiently.

Items for a single order that goes through the full lifecycle:

```
PK="ORDER#1"  SK="EVENT#order.created#2024-01-15T10:30:00Z"  eventType="order.created"
PK="ORDER#1"  SK="EVENT#order.paid#2024-01-15T10:35:00Z"     eventType="order.paid"
PK="ORDER#1"  SK="EVENT#order.shipped#2024-01-15T10:40:00Z"  eventType="order.shipped"
```

Query `PK = "ORDER#1"` returns **all three**, sorted chronologically by SK — no scan.

---

## 10.4 – GSI for querying by event type

To support pattern #3 ("all `order.paid` events across all orders"), add a GSI:

| GSI | PK | SK |
|---|---|---|
| `eventType-processedAt-index` | `eventType` | `processedAt` |

Query: `eventType = "order.paid"` → returns all paid events, sorted by time.

> **📖 What is a GSI?**
> A GSI is a separate index maintained automatically by DynamoDB. You define a different PK (and optionally SK) on any attribute. Writes to the main table propagate to the GSI automatically. GSI reads are eventually consistent and cost the same as table reads. Each table supports up to 20 GSIs.

---

## 10.5 – Create the table locally with DynamoDB Local

```bash
docker compose up dynamodb-local
```

```bash
aws --endpoint-url=http://localhost:8000 dynamodb create-table \
  --table-name OrderEvents \
  --attribute-definitions \
    AttributeName=PK,AttributeType=S \
    AttributeName=SK,AttributeType=S \
    AttributeName=eventType,AttributeType=S \
    AttributeName=processedAt,AttributeType=S \
  --key-schema \
    AttributeName=PK,KeyType=HASH \
    AttributeName=SK,KeyType=RANGE \
  --global-secondary-indexes '[
    {
      "IndexName": "eventType-processedAt-index",
      "KeySchema": [
        {"AttributeName":"eventType","KeyType":"HASH"},
        {"AttributeName":"processedAt","KeyType":"RANGE"}
      ],
      "Projection": {"ProjectionType":"ALL"}
    }
  ]' \
  --billing-mode PAY_PER_REQUEST \
  --region us-east-1
```

Test — insert a sample item and query it:

```bash
# Put an item
aws --endpoint-url=http://localhost:8000 dynamodb put-item \
  --table-name OrderEvents \
  --item '{
    "PK":          {"S":"ORDER#1"},
    "SK":          {"S":"EVENT#order.created#2024-01-15T10:30:00Z"},
    "orderId":     {"N":"1"},
    "eventType":   {"S":"order.created"},
    "processedAt": {"S":"2024-01-15T10:30:00Z"}
  }' \
  --region us-east-1

# Query all events for ORDER#1 — no scan!
aws --endpoint-url=http://localhost:8000 dynamodb query \
  --table-name OrderEvents \
  --key-condition-expression "PK = :pk" \
  --expression-attribute-values '{":pk":{"S":"ORDER#1"}}' \
  --region us-east-1
```

> **📈 Paste into `setup-local.sh` (Step 10 block)**
> Add this after the readiness checks from Step 4.

```bash

TABLE_NAME="OrderEvents"

echo "▶ Creating DynamoDB table: $TABLE_NAME ..."
aws --endpoint-url="$DYNAMODB_URL" dynamodb create-table \
  --table-name "$TABLE_NAME" \
  --attribute-definitions \
    AttributeName=PK,AttributeType=S \
    AttributeName=SK,AttributeType=S \
    AttributeName=eventType,AttributeType=S \
    AttributeName=processedAt,AttributeType=S \
  --key-schema AttributeName=PK,KeyType=HASH AttributeName=SK,KeyType=RANGE \
  --global-secondary-indexes '[{"IndexName":"eventType-processedAt-index","KeySchema":[{"AttributeName":"eventType","KeyType":"HASH"},{"AttributeName":"processedAt","KeyType":"RANGE"}],"Projection":{"ProjectionType":"ALL"}}]' \
  --billing-mode PAY_PER_REQUEST \
  --region us-east-1 2>/dev/null && echo "  ✔ Table created" || echo "  ⚠ Already exists"
```

---

## 10.6 – Provision with CDK

> **🤖 Ask your AI assistant:**
> ```
> In lib/data-stack.ts, create a DynamoDB table named "OrderEvents" with:
> - Partition key PK (STRING) and sort key SK (STRING)
> - A GSI named "eventType-processedAt-index" with PK eventType (STRING)
>   and SK processedAt (STRING), projection ALL
> - billingMode PAY_PER_REQUEST, removalPolicy DESTROY
> Export the table name and ARN via CfnOutput.
> ```

```bash
cd my-app-infra && cdk deploy DataStack-dev
```

---

## Checklist

- [ ] `OrderEvents` table with composite PK+SK created locally
- [ ] GSI `eventType-processedAt-index` created
- [ ] Query by `PK = "ORDER#1"` returns events without a scan
- [ ] CDK DataStack deployed to AWS

---

## Next step

➡️ [Step 11 – SQS](step-11-sqs.md)
