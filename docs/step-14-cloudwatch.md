# Step 13 – CloudWatch Observability

Add structured logging and monitoring to your services so you can observe what is happening in production. CloudWatch is the AWS-native solution for logs, metrics, and alarms.

> **📖 What is observability?**
> Observability is the ability to understand the internal state of a system by examining its **outputs**. The three pillars are:
> - **Logs** — timestamped records of events (errors, requests, state changes)
> - **Metrics** — numeric measurements over time (CPU usage, request count, error rate)
> - **Traces** — end-to-end records of a single request as it moves through multiple services
>
> Without observability you are flying blind: you can tell *that* something is broken but not *why* or *where*.

> **📖 What is Amazon CloudWatch?**
> CloudWatch is AWS's managed observability service. Every AWS service pushes data to CloudWatch automatically: Lambda sends its `console.log` output, ECS Fargate sends container stdout, API Gateway sends request counts and latency. You then query, visualise, and alert on this data in one place — the **CloudWatch console**.

---

## 13.1 – Where to find your logs in the AWS Console

Every resource you deploy in this project generates logs in CloudWatch. Here is where to look:

| Service | CloudWatch location |
|---|---|
| **Express API Lambda** | CloudWatch → **Log groups** → `/my-app/express-api` (created by `LambdaApiService` recipe) |
| **NestJS API Lambda** | CloudWatch → **Log groups** → `/my-app/nest-api` |
| **order-processor Lambda** | CloudWatch → **Log groups** → `/aws/lambda/order-processor` |
| **API Gateway** | CloudWatch → **Log groups** → `API-Gateway-Execution-Logs_<id>/dev` |

Navigation path in the console:

1. Open [https://console.aws.amazon.com/cloudwatch/](https://console.aws.amazon.com/cloudwatch/)
2. Left sidebar → **Logs** → **Log groups**
3. Use the search box to filter by service name (e.g. `order-processor`)
4. Click a log group → click the most recent **log stream** → see individual log events

> **💡 Tip:** Use the **Live tail** feature (top of the Log groups page) to stream logs in real time while you test your deployment — like `tail -f` but for the cloud.

---

## 13.2 – Lambda logs (automatic)

Lambda automatically sends `console.log`, `console.error`, and uncaught exceptions to CloudWatch. No configuration needed.

After deploying and invoking the Lambda (Step 12), find its logs:

```bash
# List Lambda log streams from the CLI
aws logs describe-log-streams \
  --log-group-name /aws/lambda/order-processor \
  --order-by LastEventTime \
  --descending \
  --region us-east-1

# Tail the latest log stream
aws logs get-log-events \
  --log-group-name /aws/lambda/order-processor \
  --log-stream-name <STREAM_NAME_FROM_ABOVE> \
  --region us-east-1
```

---

## 13.3 – Add structured logging to the Express API

Plain `console.log` strings are hard to query. **Structured logging** means emitting JSON objects — CloudWatch can then filter by field.

```bash
cd my-app-express-api
npm install pino pino-http
npm install --save-dev @types/pino-http
```

> **📖 What is Pino?**
> Pino is a fast Node.js logger that emits **newline-delimited JSON**. Each log line is a JSON object with standard fields (`level`, `time`, `msg`) and any custom fields you add. CloudWatch Logs Insights can query JSON fields directly — e.g. find all requests where `statusCode >= 500` across thousands of log lines in seconds.

> **🤖 Ask your AI assistant:**
> ```
> In src/index.ts of the Express API, set up pino-http as middleware.
> It should log every incoming request and response including method, url,
> statusCode, and responseTime.
> Also replace any existing console.log/error calls with the pino logger instance.
> ```

Verify locally — each request should produce a JSON log line:

```bash
npm run dev
curl http://localhost:3000/api/orders
# Console output should be a JSON object, not a plain string
```

---

## 13.4 – Add structured logging to the NestJS API

NestJS has a built-in logger, but replace it with Pino for consistency:

```bash
cd my-app-nest-api
npm install nestjs-pino pino-http
```

> **🤖 Ask your AI assistant:**
> ```
> In src/app.module.ts of the NestJS API, import LoggerModule from nestjs-pino
> and configure it with pino-http options: set level to "info" in production
> and "debug" otherwise. Replace NestJS's default Logger in main.ts with
> app.useLogger(app.get(Logger)) from nestjs-pino.
> ```

---

## 13.5 – CloudWatch Log Groups via CDK

The `LambdaApiService` recipe (Step 8) already creates log groups with 14-day retention for the Express and NestJS API Lambdas. Ensure the order-processor Lambda also has a managed log group:

> **🤖 Ask your AI assistant:**
> ```
> In lib/compute-stack.ts in my-app-infra, create a CloudWatch LogGroup
> named "/aws/lambda/order-processor" with retentionDays set to 14
> and removalPolicy DESTROY.
> Pass the log group name to the Lambda function's environment as LOG_GROUP_NAME.
> ```

> **📖 Why set a retention policy?**
> By default CloudWatch never deletes log data — it accumulates forever and you pay for storage. Setting a retention period (e.g. 14 days for dev, 90 days for prod) keeps costs under control and complies with data-handling policies.

---

## 13.6 – Create a CloudWatch Alarm via CDK

An alarm watches a metric and notifies you (via SNS email) when a threshold is crossed.

> **🤖 Ask your AI assistant:**
> ```
> In lib/compute-stack.ts, create a CloudWatch Alarm that monitors the Lambda
> function's Errors metric. Set the threshold to 1, evaluation period to 1 minute,
> treat missing data as "not breaching". Create an SNS Topic and add your email
> address as a subscription. Connect the alarm to the topic using SnsAction.
> ```

After deploying, confirm the SNS subscription in your inbox (AWS sends a confirmation email).

Test the alarm:

```bash
# Invoke the Lambda with a deliberately malformed payload to trigger an error
aws lambda invoke \
  --function-name order-processor \
  --payload '{"Records":[{"body":"not-valid-json"}]}' \
  --region us-east-1 \
  /tmp/response.json

cat /tmp/response.json
```

Within ~1 minute, check the alarm state:

```bash
aws cloudwatch describe-alarms \
  --alarm-names "order-processor-errors" \
  --region us-east-1 \
  --query "MetricAlarms[0].StateValue"
```

You should receive an email notification.

---

## 13.7 – CloudWatch Logs Insights query

Once your services are producing structured JSON logs, you can query them:

1. Console → CloudWatch → **Logs Insights**
2. Select log group `/my-app/express-api`
3. Run this query to find all 5xx errors in the last hour:

```
fields @timestamp, req.method, req.url, res.statusCode, responseTime
| filter res.statusCode >= 500
| sort @timestamp desc
| limit 20
```

> **💡 Tip:** Save frequently used queries using the **Save** button — they appear in the **Saved queries** panel for quick access in future.

---

## 13.8 – Checklist

- [ ] Lambda log group visible in CloudWatch console under `/aws/lambda/order-processor`
- [ ] Express API emitting JSON log lines (Pino)
- [ ] NestJS API emitting JSON log lines (nestjs-pino)
- [ ] CDK creates log groups with 14-day retention for all Lambda functions
- [ ] CloudWatch Alarm created for Lambda errors, connected to SNS email
- [ ] SNS subscription confirmed in inbox
- [ ] Logs Insights query runs successfully on Express API log group

---

## Next step

➡️ [Step 15 – API Gateway](step-15-api-gateway.md)
