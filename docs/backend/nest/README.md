# Backend – NestJS

A step-by-step guide to building a production-grade REST API with NestJS, covering modules, dependency injection, guards, pipes, and database integration.

---

## Step 1 – Scaffold the project

```bash
npm install -g @nestjs/cli
nest new nest-api
cd nest-api
npm run start:dev
```

Visit [http://localhost:3000](http://localhost:3000) – you should see `Hello World!`.

---

## Step 2 – Understand the project structure

```
nest-api/
├── src/
│   ├── app.module.ts       # root module – imports all feature modules
│   ├── app.controller.ts   # default controller with GET /
│   ├── app.service.ts      # default service
│   └── main.ts             # bootstrap entry point
├── test/
│   └── app.e2e-spec.ts
└── nest-cli.json
```

**Key concepts**

| Concept | Description |
|---------|-------------|
| **Module** | Groups related controllers and providers |
| **Controller** | Handles incoming HTTP requests |
| **Service** | Business logic; injected via constructor |
| **Provider** | Any class registered in the DI container |
| **Guard** | Runs before a route handler; used for authentication |
| **Pipe** | Transforms or validates request data |
| **Interceptor** | Wraps request/response lifecycle |

---

## Step 3 – Generate a Posts resource

NestJS CLI generates the full CRUD scaffold in one command:

```bash
nest generate resource posts
# Select REST API, generate CRUD entry points: Yes
```

This creates:

```
src/posts/
├── posts.module.ts
├── posts.controller.ts
├── posts.controller.spec.ts
├── posts.service.ts
├── posts.service.spec.ts
├── dto/
│   ├── create-post.dto.ts
│   └── update-post.dto.ts
└── entities/
    └── post.entity.ts
```

Import `PostsModule` in `app.module.ts` (the CLI does this automatically).

---

## Step 4 – Validate DTOs with class-validator

```bash
npm install class-validator class-transformer
```

`src/posts/dto/create-post.dto.ts`:

```ts
import { IsString, MinLength } from 'class-validator';

export class CreatePostDto {
  @IsString()
  @MinLength(1)
  title: string;
}
```

Enable the global validation pipe in `src/main.ts`:

```ts
import { ValidationPipe } from '@nestjs/common';
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  app.useGlobalPipes(new ValidationPipe({ whitelist: true }));
  await app.listen(3001);
}
bootstrap();
```

Now sending a `POST /posts` with a missing `title` field returns a structured 400 error automatically.

---

## Step 5 – Connect to PostgreSQL with TypeORM

```bash
npm install @nestjs/typeorm typeorm pg
```

Update `app.module.ts`:

```ts
import { TypeOrmModule } from '@nestjs/typeorm';
import { Post } from './posts/entities/post.entity';

@Module({
  imports: [
    TypeOrmModule.forRoot({
      type: 'postgres',
      host: process.env.DB_HOST ?? 'localhost',
      port: Number(process.env.DB_PORT) || 5432,
      username: process.env.DB_USER ?? 'app',
      password: process.env.DB_PASS ?? 'secret',
      database: process.env.DB_NAME ?? 'appdb',
      entities: [Post],
      synchronize: true, // use migrations in production
    }),
    PostsModule,
  ],
})
export class AppModule {}
```

`src/posts/entities/post.entity.ts`:

```ts
import { Column, CreateDateColumn, Entity, PrimaryGeneratedColumn } from 'typeorm';

@Entity()
export class Post {
  @PrimaryGeneratedColumn()
  id: number;

  @Column()
  title: string;

  @CreateDateColumn()
  createdAt: Date;
}
```

Import the entity repository in `posts.module.ts`:

```ts
import { TypeOrmModule } from '@nestjs/typeorm';
import { Post } from './entities/post.entity';

@Module({
  imports: [TypeOrmModule.forFeature([Post])],
  controllers: [PostsController],
  providers: [PostsService],
})
export class PostsModule {}
```

Inject and use the repository in `posts.service.ts`:

```ts
import { Injectable } from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository } from 'typeorm';
import { CreatePostDto } from './dto/create-post.dto';
import { Post } from './entities/post.entity';

@Injectable()
export class PostsService {
  constructor(
    @InjectRepository(Post)
    private readonly postsRepository: Repository<Post>,
  ) {}

  findAll(): Promise<Post[]> {
    return this.postsRepository.find();
  }

  create(dto: CreatePostDto): Promise<Post> {
    const post = this.postsRepository.create(dto);
    return this.postsRepository.save(post);
  }
}
```

---

## Step 6 – Add JWT authentication

```bash
npm install @nestjs/jwt @nestjs/passport passport passport-jwt
npm install --save-dev @types/passport-jwt
```

Generate an Auth module:

```bash
nest generate module auth
nest generate service auth
nest generate guard auth/jwt
```

`src/auth/auth.service.ts` (abbreviated):

```ts
import { Injectable } from '@nestjs/common';
import { JwtService } from '@nestjs/jwt';

@Injectable()
export class AuthService {
  constructor(private readonly jwtService: JwtService) {}

  login(userId: number): { accessToken: string } {
    return { accessToken: this.jwtService.sign({ sub: userId }) };
  }
}
```

Apply `JwtAuthGuard` to a controller route:

```ts
import { UseGuards } from '@nestjs/common';
import { JwtAuthGuard } from '../auth/jwt.guard';

@UseGuards(JwtAuthGuard)
@Get()
findAll() {
  return this.postsService.findAll();
}
```

---

## Step 7 – API documentation with Swagger

```bash
npm install @nestjs/swagger
```

`src/main.ts`:

```ts
import { DocumentBuilder, SwaggerModule } from '@nestjs/swagger';

const config = new DocumentBuilder()
  .setTitle('NestJS API')
  .setVersion('1.0')
  .addBearerAuth()
  .build();

const document = SwaggerModule.createDocument(app, config);
SwaggerModule.setup('api', app, document);
```

Visit [http://localhost:3001/api](http://localhost:3001/api) to see the interactive Swagger UI.

---

## Step 8 – Run tests

```bash
npm run test          # unit tests
npm run test:e2e      # end-to-end tests
npm run test:cov      # coverage report
```

---

## Step 9 – Dockerise

`Dockerfile`:

```dockerfile
FROM node:18-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM node:18-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --omit=dev
COPY --from=build /app/dist ./dist
ENV NODE_ENV=production
EXPOSE 3001
CMD ["node", "dist/main.js"]
```

---

## Next steps

- Deploy as a Lambda → [docs/backend/lambda](../lambda/README.md)
- Add async processing with SQS → [docs/backend/sqs](../sqs/README.md)
