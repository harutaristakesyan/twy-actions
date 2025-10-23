# GitHub Actions Workflows

This directory contains reusable GitHub Actions workflows for deploying CDK applications and publishing npm libraries.

## CDK Deployment Workflows

### Core Workflows

#### `deploy-cdk.yml` (Reusable)
The main reusable workflow for CDK deployments with separate install, build, and deploy jobs.

**Features:**
- Separate jobs for install, build, and deploy
- Deploy to multiple environments in parallel using matrix strategy
- Supports AWS CodeArtifact for private npm packages
- Uses GitHub OIDC for AWS authentication
- Caches dependencies and build artifacts

**Jobs:**
1. **install**: Installs dependencies and uploads `node_modules`
2. **build**: Builds the CDK project and uploads artifacts
3. **deploy**: Deploys to environments in parallel (matrix strategy)

### Usage Examples

#### `deploy-example.yml`
Deploy to both dev and prod environments automatically on push to main.

```yaml
on:
  push:
    branches: [main]
```

#### `deploy-single-env.yml`
Manually deploy to a single environment with full control.

**How to use:**
1. Go to Actions tab → "Deploy to Single Environment"
2. Click "Run workflow"
3. Select environment, AWS account ID, and region
4. Choose whether to run migrations

#### `deploy-with-secrets.yml`
Deploy to a single environment using GitHub Environment secrets.

**Setup:**
1. Create GitHub Environments: Settings → Environments
2. Add secrets per environment:
   - `AWS_ACCOUNT_ID`: Your AWS account ID
   - `AWS_REGION`: AWS region (optional)

**How to use:**
1. Go to Actions tab → "Deploy to Single Environment (with GitHub Secrets)"
2. Click "Run workflow"
3. Select environment name

---

## Library Publishing Workflows

### Core Workflows

#### `publish-library.yml` (Reusable)
The main reusable workflow for publishing npm packages.

**Features:**
- Separate jobs for install/build and publish
- Publish to AWS CodeArtifact or npm registry
- Supports scoped packages
- Caches dependencies and build artifacts

**Jobs:**
1. **install**: Installs dependencies, builds, and uploads artifacts
2. **publish**: Publishes to the selected registry

### Usage Examples

#### `publish-on-tag.yml`
Automatically publish when you create a version tag.

**How to use:**
```bash
git tag v1.0.0
git push origin v1.0.0
```

#### `publish-manual.yml`
Manually publish with choice of registry.

**How to use:**
1. Go to Actions tab → "Publish Library (Manual)"
2. Click "Run workflow"
3. Choose registry (CodeArtifact or npm)
4. For npm, choose access level (public/restricted)

**Setup for npm registry:**
- Add `NPM_TOKEN` secret: Settings → Secrets → Actions
- Get token from: https://www.npmjs.com/settings/[username]/tokens

#### `publish-dual.yml`
Publish to both CodeArtifact and npm registry in sequence.

**Use case:** Libraries you want in both private and public registries.

---

## Configuration

### AWS OIDC Setup

Required for all workflows that access AWS:

1. **Create OIDC Provider** in AWS IAM:
   - Provider URL: `https://token.actions.githubusercontent.com`
   - Audience: `sts.amazonaws.com`

2. **Create IAM Role** `github-deploy-role`:
   ```json
   {
     "Version": "2012-10-17",
     "Statement": [
       {
         "Effect": "Allow",
         "Principal": {
           "Federated": "arn:aws:iam::ACCOUNT_ID:oidc-provider/token.actions.githubusercontent.com"
         },
         "Action": "sts:AssumeRoleWithWebIdentity",
         "Condition": {
           "StringEquals": {
             "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
           },
           "StringLike": {
             "token.actions.githubusercontent.com:sub": "repo:YOUR_ORG/YOUR_REPO:*"
           }
         }
       }
     ]
   }
   ```

3. **Attach necessary policies** to the role:
   - For CDK deployment: CDK execution permissions
   - For CodeArtifact: `AWSCodeArtifactReadOnlyAccess` + publish permissions

### CodeArtifact Configuration

If using AWS CodeArtifact, update these values in your workflows:
- `ca_domain`: Your CodeArtifact domain name
- `ca_domain_owner`: AWS account ID that owns the domain
- `ca_repository`: Repository name
- `ca_region`: AWS region
- `ca_scope`: npm scope (e.g., `@finrisk`)

---

## Workflow Matrix

| Workflow | Type | Trigger | Purpose |
|----------|------|---------|---------|
| **CDK Deployment** |
| `deploy-cdk.yml` | Reusable | Called by others | Core CDK deployment with install/build/deploy |
| `deploy-example.yml` | Caller | Push to main | Deploy to dev + prod |
| `deploy-single-env.yml` | Caller | Manual | Deploy to one env with inputs |
| `deploy-with-secrets.yml` | Caller | Manual | Deploy to one env using secrets |
| **Library Publishing** |
| `publish-library.yml` | Reusable | Called by others | Core npm publish with install/publish |
| `publish-on-tag.yml` | Caller | Version tags | Auto-publish on tag |
| `publish-manual.yml` | Caller | Manual | Manual publish with registry choice |
| `publish-dual.yml` | Caller | Tags/Manual | Publish to both registries |

---

## Customization

### Adding New Environments

Edit the environment choices in workflows:

```yaml
type: choice
options:
  - dev
  - dev-1
  - dev-2
  - staging
  - prod
  - your-new-env  # Add here
```

### Changing Build Commands

Override defaults when calling reusable workflows:

```yaml
with:
  install_cmd: 'pnpm install'
  build_cmd: 'pnpm run build'
  migrate_cmd: 'pnpm run db:migrate'
```

### Multiple Packages

For monorepos, set `workdir` to package directory:

```yaml
with:
  workdir: './packages/my-package'
```