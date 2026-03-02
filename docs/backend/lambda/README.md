# Backend – AWS Lambda

A step-by-step guide to creating, testing, and deploying serverless functions on AWS Lambda using Node.js and the Serverless Framework.

---

## Step 1 – Understand Lambda fundamentals

| Concept | Description |
|---------|-------------|
| **Function** | The unit of deployment – a single handler |
| **Event** | Trigger that invokes the function (API Gateway, SQS, S3, …) |
| **Handler** | Exported function `(event, context) => response` |
| **Cold start** | First invocation initialises the runtime (adds latency) |
| **Execution role** | IAM role that grants the function permissions |

Lambda bills per **invocation** and per **GB-second** of execution time – you pay nothing when the function is idle.

---

## Step 2 – Install the Serverless Framework

```bash
npm install -g serverless
serverless --version
```

---

## Step 3 – Create a new service

```bash
serverless create --template aws-nodejs-typescript --path my-lambda
cd my-lambda
npm install
```

Generated structure:

```
my-lambda/
├── serverless.yml      # service definition
├── tsconfig.json
├── src/
│   └── functions/
│       └── hello/
│           ├── handler.ts
│           └── index.ts
└── package.json
```

---

## Step 4 – Write your first handler

`src/functions/hello/handler.ts`:

```ts
import { APIGatewayProxyEvent, APIGatewayProxyResult } from 'aws-lambda';

export const hello = async (
  event: APIGatewayProxyEvent,
): Promise<APIGatewayProxyResult> => {
  return {
    statusCode: 200,
    body: JSON.stringify({
      message: 'Hello from Lambda!',
      input: event,
    }),
  };
};
```

`serverless.yml` (simplified):

```yaml
service: my-lambda

provider:
  name: aws
  runtime: nodejs18.x
  region: us-east-1

functions:
  hello:
    handler: src/functions/hello/handler.hello
    events:
      - httpApi:
          path: /hello
          method: GET
```

---

## Step 5 – Test locally with LocalStack

Start LocalStack (already configured in the Docker Compose file from [docs/infra](../../infra/README.md)):

```bash
docker compose up localstack
```

Install the LocalStack Serverless plugin:

```bash
npm install --save-dev serverless-localstack
```

Add to `serverless.yml`:

```yaml
plugins:
  - serverless-localstack

custom:
  localstack:
    stages:
      - local
    host: http://localhost
    edgePort: 4566
```

Deploy and invoke against LocalStack:

```bash
serverless deploy --stage local
serverless invoke --function hello --stage local
```

---

## Step 6 – Invoke locally without LocalStack

For fast iteration, invoke the handler directly with `serverless-offline`:

```bash
npm install --save-dev serverless-offline
```

`serverless.yml`:

```yaml
plugins:
  - serverless-offline

custom:
  serverless-offline:
    httpPort: 4000
```

```bash
serverless offline
curl http://localhost:4000/hello
```

---

## Step 7 – Environment variables and secrets

Store non-sensitive configuration in `serverless.yml`:

```yaml
provider:
  environment:
    STAGE: ${sls:stage}
    TABLE_NAME: ${self:custom.tableName}

custom:
  tableName: posts-${sls:stage}
```

For secrets, use AWS Systems Manager Parameter Store:

```yaml
provider:
  environment:
    DB_PASSWORD: ${ssm:/my-app/${sls:stage}/db-password}
```

Set the parameter:

```bash
aws ssm put-parameter \
  --name "/my-app/dev/db-password" \
  --value "supersecret" \
  --type SecureString
```

---

## Step 8 – Add IAM permissions

Grant the function access to SQS and DynamoDB:

```yaml
provider:
  iam:
    role:
      statements:
        - Effect: Allow
          Action:
            - sqs:SendMessage
            - sqs:ReceiveMessage
            - sqs:DeleteMessage
          Resource: !GetAtt MyQueue.Arn
        - Effect: Allow
          Action:
            - dynamodb:PutItem
            - dynamodb:GetItem
            - dynamodb:Query
          Resource: !GetAtt MyTable.Arn
```

---

## Step 9 – Connect Lambda to SQS

See the full SQS integration guide → [docs/backend/sqs](../sqs/README.md).

A Lambda can be triggered directly by SQS messages:

```yaml
functions:
  processMessage:
    handler: src/functions/processMessage/handler.main
    events:
      - sqs:
          arn: !GetAtt MyQueue.Arn
          batchSize: 10
```

---

## Step 10 – Deploy to AWS

```bash
serverless deploy --stage dev
```

The CLI will output the API Gateway endpoint URL. Test it:

```bash
curl https://<api-id>.execute-api.us-east-1.amazonaws.com/hello
```

Remove all resources when done:

```bash
serverless remove --stage dev
```

---

## Step 11 – Provision Lambda with Terraform

Alternatively, manage Lambda with Terraform for tighter IaC control.

`infra/lambda.tf`:

```hcl
resource "aws_lambda_function" "hello" {
  function_name    = "hello-${var.environment}"
  runtime          = "nodejs18.x"
  handler          = "dist/handler.hello"
  filename         = "${path.module}/../my-lambda/my-lambda.zip"
  source_code_hash = filebase64sha256("${path.module}/../my-lambda/my-lambda.zip")
  role             = aws_iam_role.lambda_exec.arn
}

resource "aws_iam_role" "lambda_exec" {
  name = "lambda-exec-${var.environment}"
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action    = "sts:AssumeRole"
      Effect    = "Allow"
      Principal = { Service = "lambda.amazonaws.com" }
    }]
  })
}

resource "aws_iam_role_policy_attachment" "basic" {
  role       = aws_iam_role.lambda_exec.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole"
}
```

---

## Next steps

- Async messaging with SQS → [docs/backend/sqs](../sqs/README.md)
- Manage all infrastructure together → [docs/infra](../../infra/README.md)
