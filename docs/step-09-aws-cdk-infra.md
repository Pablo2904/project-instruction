# Step 9 – AWS CDK Infrastructure

Set up an AWS CDK project to define all cloud resources as code. CDK synthesises TypeScript into CloudFormation templates and deploys them.

> **Repository:** work inside your `my-app-infra` repo for this step.

> **💰 Free Tier**
> This entire stack uses only Lambda, DynamoDB, SQS, EventBridge, API Gateway, S3, CloudFront, and RDS db.t3.micro — all free tier eligible. No Fargate, no ALB, no NAT Gateway costs.

> **📖 What is CDK?**
> AWS resources are ultimately described by **CloudFormation** — AWS’s native infrastructure-as-JSON/YAML. Writing CloudFormation by hand is verbose and error-prone. **AWS CDK** (Cloud Development Kit) lets you describe the same infrastructure in TypeScript. You get autocompletion, type safety, and reusable classes.

---

## 7.1 – CDK concepts

| Concept | Description |
|---------|-------------|
| **App** | Root of the CDK program; contains one or more stacks |
| **Stack** | Unit of deployment — maps 1-to-1 to a CloudFormation stack |
| **Construct** | Reusable building block representing one or more AWS resources |
| **L1 Construct** | A thin wrapper around a single CloudFormation resource |
| **L2 Construct** | An opinionated, higher-level wrapper with sane defaults (e.g. `s3.Bucket`) |
| **L3 Construct** | A custom, domain-specific pattern built from multiple L2 constructs — also called a **recipe** |
| **Synthesize** | `cdk synth` converts CDK code into a CloudFormation template |
| **Bootstrap** | One-time setup creating an S3 bucket and IAM roles CDK needs to deploy |

---

## 7.2 – Initialise the CDK project

```bash
mkdir my-app-infra && cd my-app-infra
cdk init app --language typescript
npm install
```

Generated structure:

```
my-app-infra/
├── bin/
│   └── infra.ts          # App entry point — instantiates stacks
├── lib/
│   ├── constructs/       # ← you will create this (CDK recipes)
│   └── infra-stack.ts    # Your first stack
├── cdk.json
├── tsconfig.json
└── package.json
```

---

## 9.3 – Stack layout

Split infrastructure into focused stacks. Each stack owns one concern.

| Stack | Owns |
|---|---|
| `FrontendStack` | S3 bucket, CloudFront distribution |
| `DataStack` | DynamoDB `OrderEvents` table |
| `EventsStack` | SQS queues, EventBridge rules |
| `ComputeStack` | Express API Lambda, NestJS API Lambda, order-processor Lambda |
| `ApiStack` | API Gateway HTTP API + Lambda Authorizer |

> **🤖 Ask your AI assistant:**
> ```
> In bin/infra.ts, instantiate five CDK stacks: FrontendStack, DataStack,
> EventsStack, ComputeStack, ApiStack. Pass account from
> process.env.CDK_DEFAULT_ACCOUNT and region from process.env.CDK_DEFAULT_REGION.
> Suffix each stack name with an "env" context value (default "dev").
> ```

---

## 9.4 – ComputeStack: Lambda functions via CDK Recipe

Install the CDK recipe library (git dependency from Step 8):

```bash
npm install github:<YOUR_ORG>/my-app-cdk-recipes#main
```

> **🤖 Ask your AI assistant:**
> ```
> Create lib/compute-stack.ts.
> Import LambdaApiService from "my-app-cdk-recipes".
> Instantiate LambdaApiService twice:
>   1. functionName: "express-api"
>      entry: path.resolve(__dirname, "../../my-app-express-api/src/lambda.ts")
>      environment: { NODE_ENV: "production", DATABASE_URL: "<from props>" }
>   2. functionName: "nest-api"
>      entry: path.resolve(__dirname, "../../my-app-nest-api/src/lambda.ts")
>      environment: { NODE_ENV: "production", AWS_REGION: "us-east-1",
>                     EVENTS_TABLE: "<from DataStack output>" }
> Also create a NodejsFunction for the order-processor Lambda:
>   entry: path.resolve(__dirname, "../../my-app-lambda/src/handler.ts")
>   environment: { EVENTS_TABLE: "<from DataStack output>" }
> Export all three Lambda function ARNs via CfnOutput.
> ```

---

## 7.5 – Create the Frontend stack

> **🤖 Ask your AI assistant:**
> ```
> In lib/frontend-stack.ts, create an S3 bucket for static website hosting
> with websiteIndexDocument "index.html", publicReadAccess true,
> blockPublicAccess BLOCK_ACLS, removalPolicy DESTROY, autoDeleteObjects true.
> Export the bucket name and website URL using CfnOutput.
> ```

---

## 7.6 – Bootstrap your AWS account

One-time per account and region:

```bash
aws sts get-caller-identity --query Account --output text
cdk bootstrap aws://<ACCOUNT_ID>/us-east-1
```

---

## 7.7 – Synthesize and deploy

```bash
cdk synth              # preview CloudFormation output
cdk deploy --all       # deploy all stacks
cdk deploy DataStack-dev   # deploy a specific stack
cdk destroy --all      # tear down everything
```

---

## 7.8 – Push to GitHub

```bash
git add .
git commit -m "feat: cdk multi-stack infrastructure with NodeApiService recipe"
git remote add origin https://github.com/<YOUR_ORG>/my-app-infra.git
git push -u origin main
```

---

## Next step

➡️ [Step 10 – DynamoDB](step-10-dynamodb.md)
