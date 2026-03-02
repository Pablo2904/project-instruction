# project-instruction

A step-by-step learning guide that walks you through building a full-stack system from scratch – covering frontend, infrastructure, and multiple backend patterns.

## Table of Contents

1. [Frontend (React + Vite)](docs/frontend/README.md)
2. [Infrastructure (Docker + Terraform)](docs/infra/README.md)
3. Backend apps
   - [Express.js](docs/backend/express/README.md)
   - [NestJS](docs/backend/nest/README.md)
   - [AWS Lambda](docs/backend/lambda/README.md)
   - [Amazon SQS](docs/backend/sqs/README.md)

## Recommended Learning Order

```
1. docs/frontend    – build a React UI first so you have something to connect to
2. docs/backend/express  – simplest Node.js server; understand HTTP fundamentals
3. docs/backend/nest     – production-grade Node.js with DI, modules and decorators
4. docs/infra            – containerise and deploy everything with Docker & Terraform
5. docs/backend/lambda   – go serverless; deploy functions to AWS
6. docs/backend/sqs      – add async messaging between services with Amazon SQS
```

## Prerequisites

| Tool | Minimum version |
|------|----------------|
| Node.js | 18 LTS |
| npm | 9 |
| Docker | 24 |
| AWS CLI | 2 |
| Terraform | 1.6 |

Install Node.js via [nvm](https://github.com/nvm-sh/nvm):

```bash
nvm install 18
nvm use 18
```

Install the AWS CLI by following the [official guide](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html), then configure it:

```bash
aws configure
# Enter: AWS Access Key ID, Secret Access Key, default region, output format
```
