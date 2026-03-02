# Infrastructure – Docker & Terraform

Step-by-step instructions for containerising every service and provisioning cloud infrastructure with Terraform.

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

## Step 3 – Install Terraform

Follow the [official Terraform installation guide](https://developer.hashicorp.com/terraform/tutorials/aws-get-started/install-cli).

Verify:

```bash
terraform version   # e.g. Terraform v1.6.0
```

---

## Step 4 – Understand Terraform concepts

| Concept | Description |
|---------|-------------|
| **Provider** | Plugin that talks to a cloud API (e.g. `hashicorp/aws`) |
| **Resource** | A single infrastructure object (e.g. `aws_sqs_queue`) |
| **Variable** | Input value – keeps configs reusable |
| **Output** | Exported value you can read after `apply` |
| **State** | File (`terraform.tfstate`) tracking what Terraform has created |

---

## Step 5 – Create a Terraform project

Create `infra/` at the repo root:

```
infra/
├── main.tf
├── variables.tf
├── outputs.tf
└── terraform.tfvars
```

`infra/variables.tf`:

```hcl
variable "aws_region" {
  description = "AWS region to deploy to"
  type        = string
  default     = "us-east-1"
}

variable "environment" {
  description = "Deployment environment (dev / staging / prod)"
  type        = string
  default     = "dev"
}
```

`infra/main.tf`:

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = var.aws_region
}

# S3 bucket for static frontend assets
resource "aws_s3_bucket" "frontend" {
  bucket = "my-app-frontend-${var.environment}"
}

resource "aws_s3_bucket_website_configuration" "frontend" {
  bucket = aws_s3_bucket.frontend.id

  index_document { suffix = "index.html" }
  error_document { key    = "index.html" }
}
```

`infra/outputs.tf`:

```hcl
output "frontend_bucket_name" {
  value = aws_s3_bucket.frontend.bucket
}

output "frontend_website_endpoint" {
  value = aws_s3_bucket_website_configuration.frontend.website_endpoint
}
```

---

## Step 6 – Initialise and apply

```bash
cd infra
terraform init      # download providers
terraform plan      # preview changes
terraform apply     # create resources (type 'yes' when prompted)
```

Destroy when you no longer need the resources:

```bash
terraform destroy
```

---

## Step 7 – Store Terraform state remotely (S3 backend)

Using a local `terraform.tfstate` file is fine for learning, but in a team environment you need a remote backend. Add to `main.tf`:

```hcl
terraform {
  backend "s3" {
    bucket         = "my-terraform-state-bucket"
    key            = "project-instruction/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-state-lock"
    encrypt        = true
  }
  # ... required_providers stays the same
}
```

Create the S3 bucket and DynamoDB table once manually (or bootstrap them with a separate Terraform config).

---

## Next steps

- Set up AWS Lambda → [docs/backend/lambda](../backend/lambda/README.md)
- Add SQS messaging → [docs/backend/sqs](../backend/sqs/README.md)
