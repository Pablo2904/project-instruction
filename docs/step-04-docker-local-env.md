# Step 4 – Local Environment (Docker Compose)

Set up Docker Compose so you can run every service locally with a single command — frontend, Express API, NestJS API, PostgreSQL, DynamoDB Local, and LocalStack.

---

## 4.1 – Create `docker-compose.yml`

At the **repository root**, create `docker-compose.yml`:

```yaml
services:
  frontend:
    build: ./my-app
    ports:
      - '8080:80'
    depends_on:
      - express-api
      - nest-api

  express-api:
    build: ./express-api
    ports:
      - '3000:3000'
    environment:
      NODE_ENV: production
      DATABASE_URL: postgresql://app:secret@postgres:5432/appdb?schema=public
    depends_on:
      postgres:
        condition: service_healthy

  nest-api:
    build: ./nest-api
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

You can add a shell alias for convenience:

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

During development you will typically run **only** the databases and external services via Docker Compose, while running the frontend and APIs natively with hot-reload:

```bash
# Start only infrastructure
docker compose up postgres dynamodb-local localstack

# In separate terminals:
cd my-app && npm run dev          # frontend at :5173
cd express-api && npm run dev     # express at :3000
cd nest-api && npm run start:dev  # nest at :3001
```

The Vite dev proxy (configured in Step 2) forwards `/api/*` requests to the backend ports automatically.

---

## Next step

➡️ [Step 5 – NestJS API](step-05-nest-api.md)
