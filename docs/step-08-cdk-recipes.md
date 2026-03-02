# Step 8 – CDK Recipes (`my-app-cdk-recipes`)

Before writing AWS infrastructure, set up a **shared construct library** that both APIs can install as a npm package. This avoids copy-pasting the same CDK patterns across repositories.

> **Repository:** create and work inside your `my-app-cdk-recipes` repo for this step.

> **📖 What is a CDK L3 Construct (recipe)?**
> AWS CDK ships with **L1** constructs (thin CloudFormation wrappers) and **L2** constructs (opinionated wrappers with safe defaults). An **L3** construct — informally called a *recipe* — is a custom class you write yourself that composes multiple L2 constructs into one reusable pattern. Instead of repeating the same Lambda configuration, log group, and IAM permissions for every API, you write it once as `LambdaApiService` and instantiate it as many times as needed.

> **💰 Free Tier**
> This recipe uses **AWS Lambda** — permanent free tier: 1 million invocations/month, 400,000 GB-seconds. There is no server to pay for outside of request time.

---

## 8.1 – Scaffold the library

```bash
mkdir my-app-cdk-recipes && cd my-app-cdk-recipes
npm init -y
npm install aws-cdk-lib constructs
npm install --save-dev typescript @types/node
npx tsc --init
```

> **🤖 Ask your AI assistant:**
> ```
> Update tsconfig.json for a CDK construct library:
> target ES2020, module commonjs, outDir dist/, rootDir src/,
> declaration true, strict true, esModuleInterop true, skipLibCheck true.
> ```

> **🤖 Ask your AI assistant:**
> ```
> Update package.json: set name to "@my-org/cdk-recipes".
> Add: "main": "dist/index.js", "types": "dist/index.d.ts",
> "files": ["dist/"], "scripts": { "build": "tsc", "prepare": "npm run build" }.
> ```

---

## 8.2 – Create the `LambdaApiService` construct

```bash
mkdir src
```

> **🤖 Ask your AI assistant:**
> ```
> Create src/lambda-api-service.ts: a CDK Construct called LambdaApiService.
>
> Props interface:
>   - functionName: string
>   - entry: string         (absolute path to the Lambda handler .ts file)
>   - environment: Record<string, string>
>
> The construct should:
>   1. Create a CloudWatch LogGroup named "/my-app/{functionName}" with
>      retention of 14 days and removalPolicy DESTROY.
>   2. Create a NodejsFunction (from aws-cdk-lib/aws-lambda-nodejs) with:
>      - functionName prop
>      - runtime NODEJS_22_X
>      - handler "handler" (the exported function name)
>      - entry prop pointing to the .ts handler file
>      - bundling: { minify: true, sourceMap: false }
>      - logGroup pointing to the log group above
>      - environment prop
>      - timeout Duration.seconds(30)
>      - memorySize 256
>   3. Expose public properties: lambdaFunction (NodejsFunction).
>
> Export a CfnOutput with the function ARN named "{functionName}Arn".
> ```

> **📖 What is `NodejsFunction`?**
> `NodejsFunction` is a CDK L2 construct that uses **esbuild** under the hood to bundle your TypeScript Lambda handler — including all its `node_modules` — into a single optimised JS file. You point it at your `.ts` source file; CDK handles compilation, bundling, and zipping automatically at deploy time. No manual `tsc` + `zip` step needed.

Create the barrel export:

> **🤖 Ask your AI assistant:**
> ```
> Create src/index.ts that re-exports everything from ./lambda-api-service.
> ```

Build:

```bash
npm run build
ls dist/
# index.js  index.d.ts  lambda-api-service.js  lambda-api-service.d.ts
```

---

## 8.3 – Phase 1: Install via git dependency

```bash
# From inside my-app-infra:
npm install github:<YOUR_ORG>/my-app-cdk-recipes#main
```

> **📖 How does a git dependency work?**
> npm clones the repo, runs the `prepare` script (which builds `dist/`), and links the result as a package. The `#main` suffix pins to the `main` branch.

---

## 8.4 – Use the recipe in `my-app-infra`

The recipe is used twice — once for the Express API, once for NestJS:

> **🤖 Ask your AI assistant:**
> ```
> In lib/compute-stack.ts in my-app-infra, import LambdaApiService
> from "my-app-cdk-recipes".
>
> Instantiate LambdaApiService twice:
>   1. functionName: "express-api"
>      entry: path.resolve(__dirname, "../../my-app-express-api/src/lambda.ts")
>      environment: { NODE_ENV: "production", DATABASE_URL: "<from props or SSM>" }
>
>   2. functionName: "nest-api"
>      entry: path.resolve(__dirname, "../../my-app-nest-api/src/lambda.ts")
>      environment: { NODE_ENV: "production", AWS_REGION: "us-east-1",
>                     EVENTS_TABLE: "<from DataStack output>" }
>
> Export both function ARNs as CfnOutputs.
> ```

Same pattern, two services, one construct definition.

---

## 8.5 – Phase 2: Publish to GitHub Packages

> **📖 What is GitHub Packages?**
> A private npm registry tied to your GitHub org, free within your org's storage quota. Packages are scoped to `@my-org/`.

Add to `package.json`:

```json
{
  "publishConfig": {
    "registry": "https://npm.pkg.github.com"
  }
}
```

Create `.npmrc`:

```
@my-org:registry=https://npm.pkg.github.com
//npm.pkg.github.com/:_authToken=${GITHUB_TOKEN}
```

> **🤖 Ask your AI assistant:**
> ```
> Create .github/workflows/publish.yml in my-app-cdk-recipes.
> Trigger: on push to main.
> Steps: checkout, setup Node 22 with registry "https://npm.pkg.github.com"
> and scope "@my-org", npm ci, npm run build, npm publish.
> Set NODE_AUTH_TOKEN to secrets.GITHUB_TOKEN.
> ```

```bash
npm version patch && git push && git push --tags
```

Switch `my-app-infra` to the published package:

```bash
npm uninstall my-app-cdk-recipes
npm install @my-org/cdk-recipes@^1.0.0
```

---

## 8.6 – Push to GitHub

```bash
git add .
git commit -m "feat: LambdaApiService L3 construct"
git remote add origin https://github.com/<YOUR_ORG>/my-app-cdk-recipes.git
git push -u origin main
```

---

## Checklist

- [ ] `my-app-cdk-recipes` pushed to GitHub
- [ ] `LambdaApiService` builds cleanly (`npm run build`)
- [ ] `my-app-infra` installs via git dependency and imports `LambdaApiService`
- [ ] `express-api` and `nest-api` Lambda functions instantiated in `ComputeStack`

---

## Next step

➡️ [Step 9 – AWS CDK Infrastructure](step-09-aws-cdk-infra.md)
