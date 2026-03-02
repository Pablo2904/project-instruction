# Step 15 – API Gateway

Add Amazon API Gateway as the single public HTTPS entry point. It receives every request from the frontend and **forwards** it directly to the correct Lambda function. No business logic lives here.

> **📖 What is API Gateway?**
> API Gateway is a fully managed AWS service that sits in front of your backend services and acts as a **reverse proxy**. Clients send requests to one stable HTTPS URL. API Gateway matches the path and invokes the right Lambda function with the full request details. Benefits: one domain for the whole API, centralised logging and metrics, built-in throttling, and a place to enforce authentication before a request ever reaches your code.

> **💰 Free Tier:** HTTP API Gateway — 1 million calls/month free for 12 months. Lambda invocations — 1 million/month **always free**.

---

## 15.1 – Architecture

```
Browser
  └─ HTTPS + X-API-Key ──► API Gateway (HTTP API)
                              │
                              ├─ /api/orders/*  ──► Express API Lambda
                              └─ /api/events/*  ──► NestJS API Lambda
```

Each backend is now a Lambda function (wrapped with `serverless-http`, from Steps 3 and 5). API Gateway invokes them directly — no ALB, no load balancer, no EC2.

---

## 15.2 – API Gateway concepts

| Concept | Description |
|---------|-------------|
| **HTTP API** | Lightweight, low-latency proxy — right choice for Lambda integrations |
| **Stage** | A named deployment snapshot (`dev`, `prod`) |
| **Route** | Method + path pattern matched against incoming requests |
| **Lambda integration** | API Gateway invokes a Lambda and passes the full request as a `Proxy` event |
| **Authorizer** | A function that runs before the integration — blocks unauthorised requests |

> We use an **HTTP API** (not REST API) — lower latency, simpler, cheaper.

---

## 15.3 – Provision HTTP API with CDK

> **🤖 Ask your AI assistant:**
> ```
> In lib/api-stack.ts, import HttpApi and CorsHttpMethod from
> "@aws-cdk/aws-apigatewayv2-alpha" and HttpLambdaIntegration from
> "@aws-cdk/aws-apigatewayv2-integrations-alpha".
>
> Import the express-api Lambda function and nest-api Lambda function
> from ComputeStack as IFunction using Fn.importValue() or via props.
>
> Create an HttpApi with a CORS configuration that allows all origins,
> methods GET/POST/PATCH/DELETE/OPTIONS, and header Content-Type and X-API-Key.
>
> Add two routes:
>   ANY /api/orders/{proxy+} → HttpLambdaIntegration("ExpressIntegration", expressLambda)
>   ANY /api/events/{proxy+} → HttpLambdaIntegration("NestIntegration", nestLambda)
>
> Export the API endpoint URL via CfnOutput named "ApiGatewayUrl".
> ```

```bash
cd my-app-infra
npm install @aws-cdk/aws-apigatewayv2-alpha \
            @aws-cdk/aws-apigatewayv2-integrations-alpha
cdk deploy ApiStack-dev
```

---

## 15.4 – Test the endpoint

```bash
API_URL=$(aws cloudformation describe-stacks \
  --stack-name ApiStack-dev \
  --query "Stacks[0].Outputs[?OutputKey=='ApiGatewayUrl'].OutputValue" \
  --output text)

curl "$API_URL/api/orders"
curl "$API_URL/api/events?orderId=1"
```

---

## 15.5 – Add a Lambda Authorizer

Right now anyone with the API URL can call your API. An **authorizer** blocks unauthenticated requests before they reach your Lambda functions.

> **📖 What is a Lambda Authorizer?**
> A Lambda Authorizer is a Lambda function that API Gateway invokes before any integration. It receives the request headers, runs auth logic, and returns `{ isAuthorized: true/false }`. If `false`, API Gateway returns 403 immediately — the backend Lambdas never execute and are never billed for that request.

We implement a simple **API key header check**:
- The client must include `X-API-Key: <secret>` in every request.
- The authorizer reads the header and compares it to a value in AWS SSM Parameter Store.

> **🤖 Ask your AI assistant:**
> ```
> Create src/authorizer.ts in my-app-lambda.
> This is an API Gateway HTTP API authorizer (payload format 2.0).
> It reads event.headers["x-api-key"] and compares it to process.env.API_KEY.
> Return { isAuthorized: true } if they match, { isAuthorized: false } otherwise.
> ```

Store the secret in SSM:

```bash
aws ssm put-parameter \
  --name "/my-app/dev/api-key" \
  --value "super-secret-key-change-me" \
  --type SecureString \
  --region us-east-1
```

> **📖 What is SSM Parameter Store?**
> A managed key–value store for configuration and secrets. `SecureString` parameters are encrypted at rest. Storing secrets here instead of in source code means you can rotate the key without redeploying — just update the parameter.

> **🤖 Ask your AI assistant:**
> ```
> In lib/api-stack.ts:
> 1. Create a NodejsFunction for the authorizer using entry
>    "../../my-app-lambda/src/authorizer.ts", runtime NODEJS_22_X.
> 2. Read the SSM SecureString parameter "/my-app/dev/api-key" using
>    ssm.StringParameter.valueForSecureStringParameter and pass it as
>    API_KEY environment variable to the authorizer Lambda.
> 3. Create an HttpLambdaAuthorizer named "ApiKeyAuthorizer" with
>    authorizerResultsTtlInSeconds: 0 (disables caching for development).
> 4. Set it as the defaultAuthorizer on the HttpApi.
> ```

Test the authorizer:

```bash
# Without key — should return 403
curl -i "$API_URL/api/orders"

# With correct key — should return orders
curl -i -H "X-API-Key: super-secret-key-change-me" "$API_URL/api/orders"
```

---

## 15.6 – Update the frontend

> **🤖 Ask your AI assistant:**
> ```
> Update all fetch hooks in the frontend so that when VITE_API_BASE_URL is set,
> every request includes the header X-API-Key with the value from VITE_API_KEY.
> ```

Set both variables in your CI pipeline for the production build:

```bash
VITE_API_BASE_URL=https://<API_ID>.execute-api.us-east-1.amazonaws.com
VITE_API_KEY=super-secret-key-change-me
```

---

## 15.7 – Checklist

- [ ] `ApiStack` deployed — `cdk deploy ApiStack-dev` succeeds
- [ ] `GET /api/orders` returns orders through the gateway
- [ ] `GET /api/events?orderId=1` returns events through the gateway
- [ ] Request without `X-API-Key` returns 403
- [ ] Request with correct `X-API-Key` returns data
- [ ] API key stored in SSM, not in source code

---

## You're done 🎉

```
Browser
  └─ X-API-Key ──► API Gateway ──► Lambda Authorizer (checks SSM)
                       │
          ┌────────────┴─────────────┐
          ▼                          ▼
  Express API Lambda          NestJS API Lambda
  (serverless-http)           (serverless-http + cache)
       │                            │
  PostgreSQL (RDS)            DynamoDB (OrderEvents)
       │
  EventBridge → SQS → order-processor Lambda → DynamoDB
```

Go back to the [main guide](../README.md) for the full overview.
