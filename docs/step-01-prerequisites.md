# Step 1 – Prerequisites

Install every tool you need before writing a single line of code.

---

## 1.1 – Homebrew (macOS only)

If you are on macOS, install [Homebrew](https://brew.sh/) — the package manager that makes installing everything else easier:

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

Verify:

```bash
brew --version
```

> **Linux / Windows (WSL):** Skip this step — use your distribution's package manager (`apt`, `dnf`, etc.) or install tools directly.

---

## 1.2 – AWS account

You need an AWS account to deploy infrastructure in later steps.

1. Go to [https://aws.amazon.com/](https://aws.amazon.com/) and click **Create an AWS Account**.
2. You will need a valid credit or debit card. If you want a virtual card you can generate one with [Revolut](https://www.revolut.com/) or a similar service.
3. Complete the sign-up process and verify your email.
4. Sign in to the [AWS Management Console](https://console.aws.amazon.com/) at least once to confirm your account is active.

> **Free tier:** Most services used in this project fall within the [AWS Free Tier](https://aws.amazon.com/free/) for the first 12 months. Keep an eye on your usage to avoid unexpected charges.

---

## 1.3 – AWS CLI

Install the AWS CLI v2 so you can interact with AWS from your terminal.

**macOS (Homebrew):**

```bash
brew install awscli
```

**Other platforms:** Follow the [official installation guide](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html).

Verify:

```bash
aws --version   # e.g. aws-cli/2.15.0
```

Configure your credentials:

```bash
aws configure
```

You will be prompted for:

| Prompt | What to enter |
|--------|---------------|
| AWS Access Key ID | Your IAM user access key |
| AWS Secret Access Key | Your IAM user secret key |
| Default region name | `us-east-1` (or your preferred region) |
| Default output format | `json` |

> **Tip:** Create a dedicated IAM user with programmatic access instead of using root credentials. Attach the `AdministratorAccess` policy for learning purposes — tighten permissions in a real project.

---

## 1.4 – Node.js (via nvm)

Install [nvm](https://github.com/nvm-sh/nvm) (Node Version Manager) to manage Node.js versions:

```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.1/install.sh | bash
```

Close and reopen your terminal, then install Node.js 22:

```bash
nvm install 22
nvm use 22
nvm alias default 22
```

Verify:

```bash
node --version   # e.g. v22.x.x
npm --version    # e.g. 10.x.x
```

---

## 1.5 – Docker

Install [Docker Desktop](https://docs.docker.com/desktop/) (macOS / Windows) or [Docker Engine](https://docs.docker.com/engine/install/) (Linux).

**macOS (Homebrew):**

```bash
brew install --cask docker
```

Launch Docker Desktop once to finish setup, then verify:

```bash
docker --version          # e.g. Docker version 27.x.x
docker compose version    # e.g. Docker Compose version v2.x.x
```

---

## 1.6 – AWS CDK

Install the AWS CDK CLI globally:

```bash
npm install -g aws-cdk
```

Verify:

```bash
cdk --version   # e.g. 2.170.0
```

---

## Checklist

Before moving on, make sure you have:

- [ ] Homebrew installed (macOS) or equivalent package manager
- [ ] An active AWS account with a payment method
- [ ] AWS CLI v2 installed and configured (`aws sts get-caller-identity` returns your account)
- [ ] Node.js 22 installed via nvm
- [ ] Docker and Docker Compose installed and running
- [ ] AWS CDK CLI installed

---

## Next step

➡️ [Step 2 – Frontend](step-02-frontend.md)
