# Step 6 – GitHub Actions CI/CD

Before sending anything to AWS, every change must build and tests must pass automatically on every `git push`. That is **Continuous Integration (CI)**.

**Why CI before AWS?** Cloud infrastructure costs money. Shipping broken code to AWS wastes time and budget. A green CI pipeline is your entry ticket to the cloud.

> **Repository:** each of your five repos (`my-app-frontend`, `my-app-express-api`, `my-app-nest-api`, `my-app-infra`, `my-app-lambda`) gets its own `ci.yml`. This step shows the pattern — repeat it for each repo.

---

## 6.1 – Push your code to GitHub

From each service's directory run:

```bash
git add .
git commit -m "chore: initial commit"
git branch -M main
git remote add origin https://github.com/<YOUR_ORG>/<REPO_NAME>.git
git push -u origin main
```

> Each repository is independent. CI for `my-app-express-api` only runs when that repo changes — it does not affect the frontend pipeline.

---

## 6.2 – Add a `.gitignore`

> **🤖 Ask your AI assistant:**
> ```
> Create a .gitignore for a Node.js / TypeScript project.
> Must ignore: node_modules, dist, build, .env, *.log, .DS_Store, coverage/.
> ```

```bash
git add .gitignore
git commit -m "chore: add .gitignore"
git push
```

---

## 6.3 – How GitHub Actions works

GitHub Actions reads YAML files from `.github/workflows/`. When a trigger event fires (e.g. `push`), GitHub spins up a fresh virtual machine (runner) and executes each step in order.

```
my-app-express-api/
└── .github/
    └── workflows/
        └── ci.yml    ← pipeline definition
```

Create the directory:

```bash
mkdir -p .github/workflows
```

---

## 6.4 – Create the CI pipeline

> **🤖 Ask your AI assistant:**
> ```
> Create .github/workflows/ci.yml for a Node.js service.
> It should trigger on push and pull_request to main.
> Runner: ubuntu-latest.
> Steps:
> 1. Checkout code (actions/checkout@v4)
> 2. Set up Node 22 (actions/setup-node@v4)
> 3. Run: npm ci
> 4. Run: npm test
> 5. Build the Docker image: docker build -t service:ci .
> ```

---

## 6.5 – YAML explained

Once generated, every key section of the file follows the same pattern:

| YAML key | What it does |
|---|---|
| `name` | Display name shown in the GitHub Actions tab |
| `on.push.branches` | Trigger on push to these branches |
| `on.pull_request.branches` | Trigger when a PR targets these branches |
| `jobs.<name>.runs-on` | The OS of the runner VM (`ubuntu-latest` = Ubuntu LTS) |
| `steps[].uses` | A pre-built action from the GitHub Marketplace |
| `steps[].run` | A raw shell command |
| `working-directory` | Sets the working folder for that step |
| `npm ci` | Installs dependencies exactly as in `package-lock.json` — deterministic, faster than `npm install` |
| `docker build` | Validates the Dockerfile builds cleanly in CI |

---

## 6.6 – Configure AWS secrets in GitHub

When the pipeline later needs to deploy to AWS, it will read credentials from **encrypted GitHub Secrets** — they are never visible in logs.

Get your account ID:

```bash
aws sts get-caller-identity
```

Create an IAM access key at [AWS IAM Console](https://console.aws.amazon.com/iam/) → Users → Security credentials → Create access key → Application running outside AWS.

Add secrets to **each repo** that deploys to AWS:

1. GitHub repo → **Settings** → **Secrets and variables** → **Actions**
2. Add **New repository secret** for each:

| Secret name | Value |
|---|---|
| `AWS_ACCESS_KEY_ID` | Your IAM access key ID |
| `AWS_SECRET_ACCESS_KEY` | Your IAM secret key |
| `AWS_REGION` | `us-east-1` |

---

## 6.7 – Push the workflow and verify

```bash
git add .github/
git commit -m "ci: add GitHub Actions pipeline"
git push
```

1. Open the repo on GitHub
2. Click the **Actions** tab
3. Watch the `CI` workflow run — ✅ green means all good, ❌ red means check the logs

---

## Checklist

- [ ] `.github/workflows/ci.yml` exists in each service repo
- [ ] CI is green on the `main` branch for all active repos
- [ ] AWS secrets configured in each repo that will deploy

---

## Next step

➡️ [Step 8 – CDK Recipes](step-08-cdk-recipes.md)
