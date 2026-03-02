# Step 6 – AWS CDK Infrastructure

Set up an AWS CDK project to define all cloud resources as TypeScript code. CDK synthesises your code into CloudFormation templates and deploys them.

---

## 6.1 – Understand CDK concepts

| Concept | Description |
|---------|-------------|
| **App** | Root of the CDK program; contains one or more stacks |
| **Stack** | Unit of deployment — maps 1-to-1 to a CloudFormation stack |
| **Construct** | Reusable building block representing one or more AWS resources |
| **Synthesize** | `cdk synth` converts CDK code into a CloudFormation template |
| **Bootstrap** | One-time setup that creates an S3 bucket and roles CDK needs to deploy |

---

## 6.2 – Initialise the CDK project

```bash
mkdir infra && cd infra
cdk init app --language typescript
npm install
```

Generated structure:

```
infra/
├── bin/
│   └── infra.ts          # App entry point — instantiates stacks
├── lib/
│   └── infra-stack.ts    # Your first stack
├── cdk.json              # CDK toolkit configuration
├── tsconfig.json
└── package.json
```

---

## 6.3 – Configure the entry point

Update `bin/infra.ts`:

```ts
#!/usr/bin/env node
import * as cdk from 'aws-cdk-lib';
import { InfraStack } from '../lib/infra-stack';

const app = new cdk.App();
const env = app.node.tryGetContext('env') ?? 'dev';

new InfraStack(app, `InfraStack-${env}`, {
  env: {
    account: process.env.CDK_DEFAULT_ACCOUNT,
    region: process.env.CDK_DEFAULT_REGION ?? 'us-east-1',
  },
  stackName: `my-app-${env}`,
});
```

---

## 6.4 – Create the base stack

Update `lib/infra-stack.ts`:

```ts
import * as cdk from 'aws-cdk-lib';
import * as s3 from 'aws-cdk-lib/aws-s3';
import { Construct } from 'constructs';

export class InfraStack extends cdk.Stack {
  constructor(scope: Construct, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    // S3 bucket for static frontend assets
    const frontendBucket = new s3.Bucket(this, 'FrontendBucket', {
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

In the next steps you will add DynamoDB tables, SQS queues, EventBridge rules, and Lambda functions to this stack.

---

## 6.5 – Bootstrap your AWS account

This is a one-time step per AWS account + region:

```bash
cdk bootstrap aws://<ACCOUNT_ID>/us-east-1
```

Replace `<ACCOUNT_ID>` with your 12-digit AWS account number. You can find it with:

```bash
aws sts get-caller-identity --query Account --output text
```

---

## 6.6 – Synthesize and deploy

Preview the CloudFormation template:

```bash
cdk synth
```

Deploy the stack:

```bash
cdk deploy
```

Deploy to a different environment:

```bash
cdk deploy -c env=staging
```

Destroy all resources when done:

```bash
cdk destroy
```

---

## 6.7 – View the CloudFormation stack in the console

After deploying, you can see the stack and its resources in the [AWS CloudFormation console](https://console.aws.amazon.com/cloudformation/).

---

## Next step

➡️ [Step 7 – DynamoDB](step-07-dynamodb.md)
