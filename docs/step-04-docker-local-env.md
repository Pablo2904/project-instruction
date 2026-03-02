# Step 4 – Local Environment (Docker Compose)

Set up Docker Compose so you can run every service locally with a single command — frontend, Express API, NestJS API, PostgreSQL, DynamoDB Local, and LocalStack.

> **📖 What is Docker Compose?**
> Docker Compose is a tool for running **multiple containers** as one coordinated system. You define every service in a single `docker-compose.yml` file and start all of them with `docker compose up`. Each container is isolated (its own filesystem, network, process space) but they can communicate with each other using service names as hostnames.

> **📈 Your local bootstrap script — start it here**
> In the next several steps you will run AWS CLI commands to create a DynamoDB table, SQS queues, EventBridge rules, and deploy a Lambda to LocalStack. Rather than running these commands one by one every time you reset your environment, collect them in a single **`setup-local.sh`** file that you keep on your machine (not committed to any repo).
>
> Each step will show you the exact block to paste in. By Step 13 the file covers the complete local setup and you can re-run it any time with `bash setup-local.sh`.

Create the file now:

```bash
touch setup-local.sh && chmod +x setup-local.sh
```

**Paste this as the first block — common variables and readiness checks:**

```bash
#!/usr/bin/env bash
# setup-local.sh — paste each section as you complete the corresponding step
set -euo pipefail

export AWS_ACCESS_KEY_ID=test
export AWS_SECRET_ACCESS_KEY=test
export AWS_DEFAULT_REGION=us-east-1

LOCALSTACK_URL="http://localhost:4566"
DYNAMODB_URL="http://localhost:8000"

echo "▶ Waiting for LocalStack..."
until curl -s "$LOCALSTACK_URL/_localstack/health" | grep -q '"sqs": "running"'; do sleep 1; done
echo "  ✔ LocalStack ready"

echo "▶ Waiting for DynamoDB Local..."
until aws --endpoint-url="$DYNAMODB_URL" dynamodb list-tables --region us-east-1 &>/dev/null; do sleep 1; done
echo "  ✔ DynamoDB Local ready"
```

---

## 4.1 – Create `docker-compose.yml`

At the **repository root** (or a dedicated local dev folder), create `docker-compose.yml`:

```yaml
services:
  frontend:
    build: ./my-app-frontend
    ports:
      - '8080:80'
    depends_on:
      - express-api
      - nest-api

  express-api:
    build: ./my-app-express-api
    ports:
      - '3000:3000'
    environment:
      NODE_ENV: production
      DATABASE_URL: postgresql://app:secret@postgres:5432/appdb?schema=public
    depends_on:
      postgres:
        condition: service_healthy

  nest-api:
    build: ./my-app-nest-api
    ports:
      - '3001:3001'
    environment:
      NODE_ENV: production
      DYNAMODB_ENDPOINT: http://dynamodb-local:8000
      AWS_REGION: us-east-1
      AWS_ACCESS_KEY_ID: local
      AWS_SECRET_ACCESS_KEY: local
    depends_on:
      - dynamodb-local

  postgres:
    image: postgres:16-alpine
    restart: always
    environment:
      POSTGRES_USER: app
      POSTGRES_PASSWORD: secret
      POSTGRES_DB: appdb
    ports:
      - '5432:5432'
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ['CMD-SHELL', 'pg_isready -U app -d appdb']
      interval: 5s
      timeout: 5s
      retries: 5

> **📖 What is a healthcheck?**
> A healthcheck is a command Docker runs periodically inside the container to verify the service is actually ready to accept traffic. Here, `pg_isready` checks whether PostgreSQL is accepting connections. The `depends_on: condition: service_healthy` in the `express-api` service means Docker will wait for PostgreSQL to pass its healthcheck before starting the API — preventing connection errors on startup.

  dynamodb-local:
    image: amazon/dynamodb-local:latest
    ports:
      - '8000:8000'
    command: -jar DynamoDBLocal.jar -sharedDb

  localstack:
    image: localstack/localstack:3
    ports:
      - '4566:4566'
    environment:
      SERVICES: sqs,events,lambda,s3
      DEFAULT_REGION: us-east-1

volumes:
  postgres_data:
```

---

## 4.2 – Start everything

```bash
docker compose up --build
```

| Service | URL |
|---------|-----|
| Frontend | [http://localhost:8080](http://localhost:8080) |
| Express API | [http://localhost:3000/health](http://localhost:3000/health) |
| NestJS API | [http://localhost:3001/api](http://localhost:3001/api) |
| PostgreSQL | `localhost:5432` |
| DynamoDB Local | `localhost:8000` |
| LocalStack | `localhost:4566` |

Stop everything:

```bash
docker compose down
```

Remove volumes (reset databases):

```bash
docker compose down -v
```

---

## 4.3 – Verify LocalStack services

```bash
export AWS_ACCESS_KEY_ID=test
export AWS_SECRET_ACCESS_KEY=test
export AWS_DEFAULT_REGION=us-east-1

# List SQS queues (should be empty initially)
aws --endpoint-url=http://localhost:4566 sqs list-queues

# List EventBridge rules
aws --endpoint-url=http://localhost:4566 events list-rules
```

Add a shell alias for convenience:

```bash
alias awslocal='aws --endpoint-url=http://localhost:4566'
```

---

## 4.4 – Verify DynamoDB Local

```bash
aws --endpoint-url=http://localhost:8000 dynamodb list-tables \
  --region us-east-1
```

---

## 4.5 – Development workflow

During development, run **only** the infrastructure services via Docker Compose and start each application natively for a faster hot-reload experience:

```bash
# Start only infrastructure
docker compose up postgres dynamodb-local localstack

# In separate terminals:
cd my-app-frontend     && npm run dev         # frontend on :5173
cd my-app-express-api  && npm run dev         # express on :3000
cd my-app-nest-api     && npm run start:dev   # nest on :3001
```

The Vite dev proxy (configured in Step 2) forwards `/api/*` requests to the correct backend ports automatically.

---

## Next step

➡️ [Step 5 – NestJS API](step-05-nest-api.md)
