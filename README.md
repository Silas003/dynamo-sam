# dynamo-sam

A DynamoDB table provisioned with AWS SAM and deployed automatically to **dev** and **prod** environments via separate GitHub Actions pipelines using OIDC (no long-lived AWS keys).

---

## What this project does

It provisions a single DynamoDB table (`Users`) with:
- On-Demand billing
- Infrequent Access storage class
- Two Global Secondary Indexes (GSIs) for flexible querying

Everything is Infrastructure as Code — no manual AWS Console clicking to create resources. You push to a branch, GitHub Actions deploys it.

---

## Architecture

```
GitHub repo
├── push to dev  ──►  deploy-dev.yml  ──►  OIDC  ──►  github-actions-dynamo-sam-dev role
│                                                        └─► CloudFormation stack: dynamo-sam-dev
│                                                              └─► DynamoDB table: Users-dev
│
└── push to main ──►  deploy-prod.yml ──►  OIDC  ──►  github-actions-dynamo-sam-prod role
                                                        └─► CloudFormation stack: dynamo-sam-prod
                                                              └─► DynamoDB table: Users-prod
```

Each environment uses its own S3 bucket (created by the bootstrap stack) to store SAM deployment artifacts.

---

## File breakdown

### `template.yaml`
The SAM template — the heart of the project. Defines the DynamoDB table and its configuration.

| Property | Value | Why |
|---|---|---|
| `BillingMode` | `PAY_PER_REQUEST` | On-Demand; you pay per read/write, no capacity planning |
| `TableClass` | `STANDARD_INFREQUENT_ACCESS` | Cheaper storage for infrequently accessed data |
| `TableName` | `Users-dev` / `Users-prod` | Injected via the `Environment` parameter at deploy time |

**Schema defined in the template:**

| Attribute | Type | Role |
|---|---|---|
| `userId` | String | Primary key (partition key) |
| `email` | String | GSI partition key (EmailIndex) |
| `department` | String | GSI partition key (DepartmentIndex) |

> Only attributes used as keys need to be declared in `AttributeDefinitions`. All other attributes (e.g. `name`, `age`) are schema-less and added freely when inserting items.

**Global Secondary Indexes:**
- `EmailIndex` — lets you query the table by `email` without knowing the `userId`
- `DepartmentIndex` — lets you fetch all users in a department

**Outputs** expose the table name and ARN so other stacks can reference them via `Fn::ImportValue`.

---

### `samconfig.toml`
Stores default parameters for `sam deploy` so you don't repeat them on the command line.

```toml
[dev.deploy.parameters]
stack_name = "dynamo-sam-dev"
region     = "eu-central-1"
...
parameter_overrides = "Environment=dev"
```

The GitHub Actions workflows reference these with `--config-env dev` or `--config-env prod`. Sensitive values (bucket name, role ARN) are never stored here — they are injected at runtime from GitHub secrets.

---

### `bootstrap.yaml`
A plain CloudFormation template (not SAM) that sets up the deployment infrastructure itself. Run once manually before any GitHub Actions pipeline can work.

It creates:

| Resource | Purpose |
|---|---|
| `GitHubOIDCProvider` | Trusts GitHub Actions tokens so no AWS access keys are needed |
| `DevArtifactsBucket` | S3 bucket for dev SAM deployment artifacts |
| `ProdArtifactsBucket` | S3 bucket for prod SAM deployment artifacts |
| `DeployPolicy` | Least-privilege IAM policy (CloudFormation + DynamoDB + S3) |
| `DevDeployRole` | IAM role assumed by the dev pipeline; trusts the `dev` GitHub Environment |
| `ProdDeployRole` | IAM role assumed by the prod pipeline; trusts the `prod` GitHub Environment |

The `CreateOIDCProvider` parameter (default `true`) lets you skip re-creating the OIDC provider if it already exists in the account — set it to `false` on subsequent runs.

---

### `.github/workflows/deploy-dev.yml`
Triggered on every push to the `dev` branch. Runs in the `dev` GitHub Environment (which holds the dev-specific secrets).

Steps:
1. Checkout code
2. Install SAM CLI
3. Authenticate to AWS via OIDC (no stored keys — GitHub mints a short-lived token)
4. `sam build` — packages the template
5. `sam deploy --config-env dev` — deploys the `dynamo-sam-dev` CloudFormation stack using the dev S3 artifact bucket

---

### `.github/workflows/deploy-prod.yml`
Identical structure to the dev pipeline but triggered on push to `main` and uses the `prod` GitHub Environment secrets. Deploys the `dynamo-sam-prod` stack.

---

### `.github/workflows/deploy-bootstrap.yml`
Deploys the `bootstrap.yaml` stack via GitHub Actions. Triggered manually (`workflow_dispatch`) or automatically when `bootstrap.yaml` changes on `main`.

Uses a `BOOTSTRAP_ROLE_ARN` repository secret — a pre-existing IAM role you must create manually the very first time (or use local AWS credentials for the first bootstrap deploy).

---

## First-time setup

### 1. Deploy the bootstrap stack (run locally once)

```bash
aws cloudformation deploy \
  --template-file bootstrap.yaml \
  --stack-name dynamo-sam-bootstrap \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameter-overrides \
    GitHubOrg=<your-github-username> \
    GitHubRepo=dynamo-sam \
    CreateOIDCProvider=false \   # omit if OIDC provider doesn't exist yet
  --region eu-central-1
```

### 2. Grab the outputs

```bash
aws cloudformation describe-stacks \
  --stack-name dynamo-sam-bootstrap \
  --query 'Stacks[0].Outputs[*].{Key:OutputKey,Value:OutputValue}' \
  --output table \
  --region eu-central-1
```

### 3. Configure GitHub secrets

**Under Settings → Environments → `dev` (Secrets):**

| Secret | Value |
|---|---|
| `AWS_IAM_ROLE_ARN` | `DevDeployRoleArn` output |
| `SAM_ARTIFACTS_BUCKET` | `DevArtifactsBucketName` output |
| `AWS_REGION` | `eu-central-1` |

**Under Settings → Environments → `prod` (Secrets):**

| Secret | Value |
|---|---|
| `AWS_IAM_ROLE_ARN` | `ProdDeployRoleArn` output |
| `SAM_ARTIFACTS_BUCKET` | `ProdArtifactsBucketName` output |
| `AWS_REGION` | `eu-central-1` |

**Under Settings → Secrets and variables → Actions (Repository secrets):**

| Secret | Value |
|---|---|
| `BOOTSTRAP_ROLE_ARN` | ARN of the role/user used for the first bootstrap deploy |
| `AWS_REGION` | `eu-central-1` |
| `ORG_NAME` | Your GitHub username |
| `REPO_NAME` | `dynamo-sam` |

### 4. Trigger deployments

```bash
# Deploy to dev
git push origin dev

# Deploy to prod
git push origin main
```

---

## Testing via AWS Console

1. Go to **DynamoDB → Tables** in `eu-central-1`
2. Open `Users-dev` → **Explore table items** → **Create item**
3. Insert a sample item (JSON view):

```json
{
  "userId":     {"S": "user-001"},
  "email":      {"S": "alice@example.com"},
  "department": {"S": "Engineering"},
  "name":       {"S": "Alice Smith"}
}
```

4. **Query by primary key:** Query tab → partition key `user-001`
5. **Query by email GSI:** Switch index to `EmailIndex` → partition key `alice@example.com`
6. **Query by department GSI:** Switch index to `DepartmentIndex` → partition key `Engineering`
