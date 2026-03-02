# Infrastructure – Docker & AWS CDK

Step-by-step instructions for containerising every service and provisioning cloud infrastructure with AWS CDK (TypeScript) and CloudFormation.

---

## Step 1 – Install Docker

Follow the [official Docker installation guide](https://docs.docker.com/engine/install/) for your OS.

Verify the installation:

```bash
docker --version        # e.g. Docker version 24.0.2
docker compose version  # e.g. Docker Compose version v2.20.2
```

---

## Step 2 – Run all services with Docker Compose

Create `docker-compose.yml` at the repository root:

```yaml
version: '3.9'

services:
  frontend:
    build: ./my-app
    ports:
      - '8080:80'
    depends_on:
      - express-api

  express-api:
    build: ./express-api
    ports:
      - '3000:3000'
    environment:
      - NODE_ENV=production
    depends_on:
      - postgres

  nest-api:
    build: ./nest-api
    ports:
      - '3001:3001'
    environment:
      - NODE_ENV=production
    depends_on:
      - postgres

  postgres:
    image: postgres:15-alpine
    restart: always
    environment:
      POSTGRES_USER: app
      POSTGRES_PASSWORD: secret
      POSTGRES_DB: appdb
    ports:
      - '5432:5432'
    volumes:
      - postgres_data:/var/lib/postgresql/data

  localstack:
    image: localstack/localstack:3
    ports:
      - '4566:4566'
    environment:
      - SERVICES=sqs,lambda,s3
      - DEFAULT_REGION=us-east-1

volumes:
  postgres_data:
```

Start all services:

```bash
docker compose up --build
```

Stop them:

```bash
docker compose down
```

---

## Step 3 – Install the AWS CDK

```bash
npm install -g aws-cdk
cdk --version   # e.g. 2.100.0 (build xxxxxxx)
```

---

## Step 4 – Understand CDK concepts

| Concept | Description |
|---------|-------------|
| **App** | Root of the CDK program; contains one or more stacks |
| **Stack** | Unit of deployment – maps 1-to-1 to a CloudFormation stack |
| **Construct** | Reusable building block representing one or more AWS resources |
| **Synthesize** | `cdk synth` converts CDK code to a CloudFormation template |
| **Bootstrap** | One-time setup that creates an S3 bucket and ECR repo CDK needs to deploy |

---

## Step 5 – Create a CDK app

```bash
mkdir infra && cd infra
cdk init app --language typescript
npm install
```

Generated structure:

```
infra/
├── bin/
│   └── infra.ts        # App entry point – instantiates stacks
├── lib/
│   └── infra-stack.ts  # Your stack definition
├── cdk.json            # CDK toolkit configuration
└── tsconfig.json
```

`bin/infra.ts`:

```ts
import * as cdk from 'aws-cdk-lib';
import { InfraStack } from '../lib/infra-stack';

const app = new cdk.App();

new InfraStack(app, 'InfraStack', {
  env: {
    account: process.env.CDK_DEFAULT_ACCOUNT,
    region: process.env.CDK_DEFAULT_REGION ?? 'us-east-1',
  },
});
```

`lib/infra-stack.ts`:

```ts
import * as cdk from 'aws-cdk-lib';
import * as s3 from 'aws-cdk-lib/aws-s3';
import { Construct } from 'constructs';

export class InfraStack extends cdk.Stack {
  constructor(scope: Construct, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    // S3 bucket for static frontend assets
    const frontendBucket = new s3.Bucket(this, 'FrontendBucket', {
      bucketName: `my-app-frontend-${this.stackName.toLowerCase()}`,
      websiteIndexDocument: 'index.html',
      websiteErrorDocument: 'index.html',
      publicReadAccess: true,
      blockPublicAccess: s3.BlockPublicAccess.BLOCK_ACLS,
      removalPolicy: cdk.RemovalPolicy.DESTROY,
      autoDeleteObjects: true,
    });

    new cdk.CfnOutput(this, 'FrontendBucketName', {
      value: frontendBucket.bucketName,
    });

    new cdk.CfnOutput(this, 'FrontendWebsiteUrl', {
      value: frontendBucket.bucketWebsiteUrl,
    });
  }
}
```

---

## Step 6 – Bootstrap, synthesize, and deploy

Bootstrap your AWS account/region (one-time step per account+region):

```bash
cdk bootstrap aws://<ACCOUNT_ID>/us-east-1
```

Preview the CloudFormation template CDK will generate:

```bash
cdk synth
```

Deploy:

```bash
cdk deploy
```

Destroy all resources when you no longer need them:

```bash
cdk destroy
```

---

## Step 7 – Multiple environments

Pass context values or use environment variables to differentiate stacks:

```ts
// bin/infra.ts
const env = app.node.tryGetContext('env') ?? 'dev';

new InfraStack(app, `InfraStack-${env}`, {
  env: { account: process.env.CDK_DEFAULT_ACCOUNT, region: 'us-east-1' },
  stackName: `my-app-${env}`,
});
```

Deploy to a specific environment:

```bash
cdk deploy -c env=staging
cdk deploy -c env=prod
```

---

## Next steps

- Set up AWS Lambda → [docs/backend/lambda](../backend/lambda/README.md)
- Add SQS messaging → [docs/backend/sqs](../backend/sqs/README.md)
