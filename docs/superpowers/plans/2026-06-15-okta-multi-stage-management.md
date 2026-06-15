# Okta DEV/TEST/PROD Terraform Management — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Stand up a working Terraform pipeline that manages three Okta orgs (DEV/TEST/PROD) by consuming the published `okta/okta` provider, with Azure Blob state, OAuth2 service-app auth, and GitHub Actions promotion.

**Architecture:** One `okta-base-config` repo holds version-tagged reusable modules; three env repos (`okta-dev/test/prod`) pin a module version and apply it to their org via GitHub Actions. State lives in one Azure storage account (three containers); CI authenticates to Azure via GitHub OIDC and to Okta via per-org service apps.

**Tech Stack:** Terraform (azurerm backend + okta provider `~> 6.12`), Azure Blob Storage + Entra ID, GitHub Actions, `az` CLI, `gh` CLI.

**Spec:** `docs/superpowers/specs/2026-06-15-okta-multi-stage-management-design.md`

---

## Strategy: vertical slice first

This plan drives **one real resource — an `okta_group` — end to end through the whole pipeline** before adding breadth. After Phase D you have a green path: PR → plan → apply → a group created in the DEV org. Phases E–F replicate to TEST/PROD and prove promotion. Adding more Okta resources later is purely additive (new modules in `okta-base-config`, new entries in each env's tfvars) and does not change the machinery.

## Prerequisites

Install locally and authenticate before starting:

```bash
terraform -version      # >= 1.6
az version              # Azure CLI
gh auth status          # GitHub CLI, logged in
az login                # operator must have Owner/Contributor on the subscription
```

The operator running Phase A also needs **Storage Blob Data Owner** on the storage account to create containers with AAD auth.

## Variables you must set (used verbatim in commands below)

These are environment-specific and intentionally parameterized. Set them once in your shell; every command in this plan references them.

```bash
export GH_OWNER="jandors"                       # GitHub org/user that will own the repos
export AZ_LOCATION="switzerlandnorth"            # Azure region
export AZ_SUB="$(az account show --query id -o tsv)"
export AZ_RG="rg-okta-tfstate"
export AZ_SA="stoktatfstate$RANDOM"              # storage account name must be globally unique; record the final value
export REPO_ROOT="/mnt/c/Git/okta-css"           # where the new repos are cloned (sibling of this repo)
```

> After picking `AZ_SA`, **record the final value** — it is hardcoded in each repo's `backend.tf`.

Okta org coordinates (fill in after you have the orgs; preview orgs end in `oktapreview.com`, prod in `okta.com`):

```bash
export OKTA_ORG_DEV="dev-XXXX";  export OKTA_URL_DEV="oktapreview.com"
export OKTA_ORG_TEST="test-XXXX"; export OKTA_URL_TEST="oktapreview.com"
export OKTA_ORG_PROD="acme";      export OKTA_URL_PROD="okta.com"
```

---

## Phase A — Azure backend bootstrap (out of band, once)

### Task A1: Create storage account + three containers

**Files:** none (cloud resources via CLI).

- [ ] **Step 1: Create the resource group and storage account (account key disabled)**

```bash
az group create -n "$AZ_RG" -l "$AZ_LOCATION"
az storage account create \
  -n "$AZ_SA" -g "$AZ_RG" -l "$AZ_LOCATION" \
  --sku Standard_LRS --kind StorageV2 \
  --allow-shared-key-access false --min-tls-version TLS1_2
```

- [ ] **Step 2: Grant yourself blob data access (needed because shared key is off)**

```bash
az role assignment create \
  --assignee "$(az ad signed-in-user show --query id -o tsv)" \
  --role "Storage Blob Data Owner" \
  --scope "/subscriptions/$AZ_SUB/resourceGroups/$AZ_RG/providers/Microsoft.Storage/storageAccounts/$AZ_SA"
```

- [ ] **Step 3: Create one container per stage**

```bash
for env in dev test prod; do
  az storage container create \
    --account-name "$AZ_SA" -n "tfstate-$env" --auth-mode login
done
```

- [ ] **Step 4: Verify all three containers exist**

```bash
az storage container list --account-name "$AZ_SA" --auth-mode login \
  --query "[].name" -o tsv
```
Expected output (order may vary):
```
tfstate-dev
tfstate-prod
tfstate-test
```

The per-stage Entra federated identities and their container-scoped role assignments are created later (Phase D Task D2 for dev; Phase E for test/prod), because they reference repos that don't exist yet.

---

## Phase B — Okta DEV service app (out of band, console)

### Task B1: Create the DEV org OAuth2 service app

**Files:** none (Okta Admin Console). This is the chicken-and-egg bootstrap; the app is **not** managed by Terraform.

- [ ] **Step 1: Create the service app**

In the DEV org Admin Console: **Applications → Applications → Create App Integration → API Services → Next**. Name it `terraform-automation`. Save. Record the **Client ID**.

- [ ] **Step 2: Switch the app to private-key JWT auth and add a key**

On the app's **General** tab → **Client Credentials** → edit → **Client authentication = Public key / Private key**. Under **Public Keys → Add key → Generate new key**. Copy the **private key in PEM** form when shown (you cannot retrieve it again). Record the **Key ID (kid)**.

- [ ] **Step 3: Grant least-privilege API scopes**

On the app's **Okta API Scopes** tab, grant (for the structural config in scope):
`okta.groups.manage`, `okta.groups.read`, `okta.apps.manage`, `okta.apps.read`, `okta.policies.manage`, `okta.policies.read`, `okta.authorizationServers.manage`, `okta.authorizationServers.read`, `okta.idps.manage`, `okta.idps.read`, `okta.brands.manage`, `okta.brands.read`.

- [ ] **Step 4: Assign an admin role to the app**

On the app's **Admin roles** tab → **Edit assignments** → add a role. Use a scoped custom admin role if your managed resources allow; otherwise **Super Administrator** for the initial slice. Save.

- [ ] **Step 5: Record the credentials for later (do NOT commit)**

You should now have, for DEV: Client ID, Key ID, private-key PEM, plus `$OKTA_ORG_DEV` / `$OKTA_URL_DEV`. Keep the PEM in a local file you will reference in Task D3, e.g.:

```bash
mkdir -p ~/.okta-tf-secrets && chmod 700 ~/.okta-tf-secrets
# paste the PEM into this file:
$EDITOR ~/.okta-tf-secrets/dev.pem
chmod 600 ~/.okta-tf-secrets/dev.pem
```

---

## Phase C — `okta-base-config` repo (modules + CI)

### Task C1: Scaffold the repo

**Files:**
- Create: `$REPO_ROOT/okta-base-config/.gitignore`
- Create: `$REPO_ROOT/okta-base-config/README.md`

- [ ] **Step 1: Create the repo locally and on GitHub**

```bash
cd "$REPO_ROOT"
gh repo create "$GH_OWNER/okta-base-config" --private --clone
cd okta-base-config
```

- [ ] **Step 2: Add `.gitignore`**

```gitignore
.terraform/
*.tfstate
*.tfstate.*
*.tfvars
!*.example.tfvars
.terraform.lock.hcl
crash.log
```

- [ ] **Step 3: Add `README.md`**

```markdown
# okta-base-config

Reusable Terraform modules for managing Okta structural configuration
(groups, apps, policies, auth servers, branding).

**This repo is never applied directly.** It is consumed by the per-environment
repos (`okta-dev`, `okta-test`, `okta-prod`), which pin a released git tag.

## Release process
1. Merge changes to `main`.
2. Tag a semver release: `git tag v1.2.0 && git push --tags`.
3. Bump the `?ref=` pin in the next environment repo to promote.
```

- [ ] **Step 4: Commit**

```bash
git add .gitignore README.md
git commit -m "chore: scaffold okta-base-config repo"
```

### Task C2: Write the `groups` module (the worked example)

**Files:**
- Create: `$REPO_ROOT/okta-base-config/modules/groups/variables.tf`
- Create: `$REPO_ROOT/okta-base-config/modules/groups/main.tf`
- Create: `$REPO_ROOT/okta-base-config/modules/groups/outputs.tf`
- Create: `$REPO_ROOT/okta-base-config/modules/groups/README.md`

- [ ] **Step 1: Write `variables.tf`**

```hcl
variable "groups" {
  description = "Map of Okta groups to manage, keyed by group name."
  type = map(object({
    description = string
  }))
  default = {}
}
```

- [ ] **Step 2: Write `main.tf`**

```hcl
terraform {
  required_providers {
    okta = {
      source  = "okta/okta"
      version = ">= 6.0.0"
    }
  }
}

resource "okta_group" "this" {
  for_each    = var.groups
  name        = each.key
  description = each.value.description
}
```

- [ ] **Step 3: Write `outputs.tf`**

```hcl
output "group_ids" {
  description = "Map of group name => Okta group id."
  value       = { for name, g in okta_group.this : name => g.id }
}
```

- [ ] **Step 4: Write `README.md`**

```markdown
# groups module

Manages Okta groups from a data-driven map.

## Inputs
- `groups` — map keyed by group name; each value has `description`.

## Outputs
- `group_ids` — map of group name => group id.
```

- [ ] **Step 5: Verify the module is valid HCL**

```bash
cd "$REPO_ROOT/okta-base-config/modules/groups"
terraform init -backend=false && terraform validate
```
Expected: `Success! The configuration is valid.`

- [ ] **Step 6: Commit**

```bash
cd "$REPO_ROOT/okta-base-config"
git add modules/groups
git commit -m "feat: add groups module"
```

### Task C3: Add an example that exercises the module

**Files:**
- Create: `$REPO_ROOT/okta-base-config/examples/groups/main.tf`

- [ ] **Step 1: Write the example**

```hcl
terraform {
  required_providers {
    okta = {
      source  = "okta/okta"
      version = ">= 6.0.0"
    }
  }
}

# Example only — CI validates this without applying.
module "groups" {
  source = "../../modules/groups"
  groups = {
    "tf-managed-engineering" = { description = "Managed by Terraform" }
  }
}
```

- [ ] **Step 2: Validate the example**

```bash
cd "$REPO_ROOT/okta-base-config/examples/groups"
terraform init -backend=false && terraform validate
```
Expected: `Success! The configuration is valid.`

- [ ] **Step 3: Commit**

```bash
cd "$REPO_ROOT/okta-base-config"
git add examples/groups
git commit -m "docs: add groups example"
```

### Task C4: Add CI (validate only, never apply)

**Files:**
- Create: `$REPO_ROOT/okta-base-config/.github/workflows/ci.yml`

- [ ] **Step 1: Write `ci.yml`**

```yaml
name: ci
on:
  pull_request:
  push:
    branches: [main]
jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: "1.9.8"
      - name: terraform fmt
        run: terraform fmt -check -recursive
      - name: validate examples
        run: |
          set -e
          for dir in examples/*/; do
            echo "== validating $dir =="
            terraform -chdir="$dir" init -backend=false
            terraform -chdir="$dir" validate
          done
```

- [ ] **Step 2: Format the repo so `fmt -check` will pass**

```bash
cd "$REPO_ROOT/okta-base-config"
terraform fmt -recursive
```

- [ ] **Step 3: Commit and push**

```bash
git add -A
git commit -m "ci: validate modules and examples"
git push -u origin main
```

- [ ] **Step 4: Verify CI passed on GitHub**

```bash
gh run list --repo "$GH_OWNER/okta-base-config" --limit 1
```
Expected: the latest `ci` run shows `completed  success`.

### Task C5: Cut the first release tag

- [ ] **Step 1: Tag and push**

```bash
cd "$REPO_ROOT/okta-base-config"
git tag v1.0.0
git push origin v1.0.0
```

- [ ] **Step 2: Verify the tag is on the remote**

```bash
git ls-remote --tags origin | grep v1.0.0
```
Expected: a line ending in `refs/tags/v1.0.0`.

---

## Phase D — `okta-dev` vertical slice (end-to-end apply)

### Task D1: Scaffold the env repo

**Files:**
- Create: `$REPO_ROOT/okta-dev/versions.tf`
- Create: `$REPO_ROOT/okta-dev/providers.tf`
- Create: `$REPO_ROOT/okta-dev/backend.tf`
- Create: `$REPO_ROOT/okta-dev/variables.tf`
- Create: `$REPO_ROOT/okta-dev/main.tf`
- Create: `$REPO_ROOT/okta-dev/dev.auto.tfvars`
- Create: `$REPO_ROOT/okta-dev/.gitignore`

- [ ] **Step 1: Create repo locally + on GitHub**

```bash
cd "$REPO_ROOT"
gh repo create "$GH_OWNER/okta-dev" --private --clone
cd okta-dev
```

- [ ] **Step 2: Write `.gitignore`**

```gitignore
.terraform/
*.tfstate
*.tfstate.*
crash.log
*.pem
.env
```

- [ ] **Step 3: Write `versions.tf`**

```hcl
terraform {
  required_version = ">= 1.6"
  required_providers {
    okta = {
      source  = "okta/okta"
      version = "~> 6.12"
    }
  }
}
```

- [ ] **Step 4: Write `providers.tf`** (credentials come from `OKTA_API_*` env vars in CI)

```hcl
provider "okta" {
  org_name = var.org_name
  base_url = var.base_url
}
```

- [ ] **Step 5: Write `backend.tf`** (replace `STORAGE_ACCOUNT` with your recorded `$AZ_SA`)

```hcl
terraform {
  backend "azurerm" {
    resource_group_name  = "rg-okta-tfstate"
    storage_account_name = "STORAGE_ACCOUNT"
    container_name       = "tfstate-dev"
    key                  = "okta.tfstate"
    use_oidc             = true
    use_azuread_auth     = true
  }
}
```

- [ ] **Step 6: Substitute the storage account name**

```bash
sed -i "s/STORAGE_ACCOUNT/$AZ_SA/" backend.tf
grep storage_account_name backend.tf   # confirm it shows your real account
```

- [ ] **Step 7: Write `variables.tf`**

```hcl
variable "org_name" {
  description = "Okta org subdomain (e.g. dev-12345)."
  type        = string
}

variable "base_url" {
  description = "Okta base URL (oktapreview.com or okta.com)."
  type        = string
}

variable "groups" {
  description = "Groups to manage in this org."
  type = map(object({
    description = string
  }))
  default = {}
}
```

- [ ] **Step 8: Write `main.tf`** (module pinned to v1.0.0)

```hcl
module "groups" {
  source = "git::https://github.com/jandors/okta-base-config.git//modules/groups?ref=v1.0.0"
  groups = var.groups
}
```

> If `$GH_OWNER` is not `jandors`, edit the `source` host path accordingly.

- [ ] **Step 9: Write `dev.auto.tfvars`** (the real resource for the slice)

```hcl
org_name = "dev-XXXX"      # set to your DEV org subdomain ($OKTA_ORG_DEV)
base_url = "oktapreview.com"

groups = {
  "tf-managed-engineering" = { description = "Managed by Terraform (DEV)" }
}
```

- [ ] **Step 10: Set the real org name and commit**

```bash
sed -i "s/dev-XXXX/$OKTA_ORG_DEV/" dev.auto.tfvars
git add -A
git commit -m "feat: okta-dev root config with groups slice"
```

### Task D2: Create the DEV Azure federated identity

**Files:** none (cloud resources via CLI).

- [ ] **Step 1: Create an Entra app + service principal**

```bash
az ad app create --display-name "okta-dev-tf"
export DEV_APP_ID="$(az ad app list --display-name 'okta-dev-tf' --query '[0].appId' -o tsv)"
az ad sp create --id "$DEV_APP_ID"
```

- [ ] **Step 2: Add a federated credential bound to the repo's `dev` environment**

```bash
az ad app federated-credential create --id "$DEV_APP_ID" --parameters "{
  \"name\": \"okta-dev-env\",
  \"issuer\": \"https://token.actions.githubusercontent.com\",
  \"subject\": \"repo:$GH_OWNER/okta-dev:environment:dev\",
  \"audiences\": [\"api://AzureADTokenExchange\"]
}"
```

- [ ] **Step 3: Grant blob access scoped to ONLY the dev container**

```bash
az role assignment create \
  --assignee "$DEV_APP_ID" \
  --role "Storage Blob Data Contributor" \
  --scope "/subscriptions/$AZ_SUB/resourceGroups/$AZ_RG/providers/Microsoft.Storage/storageAccounts/$AZ_SA/blobServices/default/containers/tfstate-dev"
```

- [ ] **Step 4: Verify the federated credential exists**

```bash
az ad app federated-credential list --id "$DEV_APP_ID" --query "[].subject" -o tsv
```
Expected: `repo:<owner>/okta-dev:environment:dev`

### Task D3: Create the GitHub `dev` environment + secrets

**Files:** none (GitHub config via CLI).

- [ ] **Step 1: Create the `dev` environment**

```bash
gh api --method PUT "repos/$GH_OWNER/okta-dev/environments/dev"
```

- [ ] **Step 2: Set Azure (OIDC) secrets on the environment**

```bash
export AZ_TENANT="$(az account show --query tenantId -o tsv)"
gh secret set ARM_CLIENT_ID       --env dev --repo "$GH_OWNER/okta-dev" --body "$DEV_APP_ID"
gh secret set ARM_TENANT_ID       --env dev --repo "$GH_OWNER/okta-dev" --body "$AZ_TENANT"
gh secret set ARM_SUBSCRIPTION_ID --env dev --repo "$GH_OWNER/okta-dev" --body "$AZ_SUB"
```

- [ ] **Step 3: Set Okta service-app secrets on the environment**

```bash
gh secret set OKTA_API_CLIENT_ID      --env dev --repo "$GH_OWNER/okta-dev" --body "PASTE_DEV_CLIENT_ID"
gh secret set OKTA_API_PRIVATE_KEY_ID --env dev --repo "$GH_OWNER/okta-dev" --body "PASTE_DEV_KEY_ID"
gh secret set OKTA_API_PRIVATE_KEY    --env dev --repo "$GH_OWNER/okta-dev" < ~/.okta-tf-secrets/dev.pem
gh secret set OKTA_API_SCOPES         --env dev --repo "$GH_OWNER/okta-dev" \
  --body "okta.groups.manage,okta.groups.read,okta.apps.manage,okta.apps.read,okta.policies.manage,okta.policies.read,okta.authorizationServers.manage,okta.authorizationServers.read,okta.idps.manage,okta.idps.read,okta.brands.manage,okta.brands.read"
```

- [ ] **Step 4: Verify secrets are present**

```bash
gh secret list --env dev --repo "$GH_OWNER/okta-dev"
```
Expected: all six `ARM_*` / `OKTA_API_*` names listed.

### Task D4: Add the `plan.yml` workflow

**Files:**
- Create: `$REPO_ROOT/okta-dev/.github/workflows/plan.yml`

- [ ] **Step 1: Write `plan.yml`**

```yaml
name: plan
on:
  pull_request:
permissions:
  id-token: write
  contents: read
  pull-requests: write
jobs:
  plan:
    runs-on: ubuntu-latest
    environment: dev
    env:
      ARM_USE_OIDC: "true"
      ARM_CLIENT_ID: ${{ secrets.ARM_CLIENT_ID }}
      ARM_TENANT_ID: ${{ secrets.ARM_TENANT_ID }}
      ARM_SUBSCRIPTION_ID: ${{ secrets.ARM_SUBSCRIPTION_ID }}
      OKTA_API_CLIENT_ID: ${{ secrets.OKTA_API_CLIENT_ID }}
      OKTA_API_PRIVATE_KEY: ${{ secrets.OKTA_API_PRIVATE_KEY }}
      OKTA_API_PRIVATE_KEY_ID: ${{ secrets.OKTA_API_PRIVATE_KEY_ID }}
      OKTA_API_SCOPES: ${{ secrets.OKTA_API_SCOPES }}
    steps:
      - uses: actions/checkout@v4
      - uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: "1.9.8"
      - run: terraform fmt -check -recursive
      - run: terraform init
      - run: terraform validate
      - run: terraform plan -no-color -input=false -out=tf.plan | tee plan.txt
      - name: Post plan to PR
        uses: actions/github-script@v7
        with:
          script: |
            const fs = require('fs');
            const body = '```\n' + fs.readFileSync('plan.txt','utf8').slice(0, 60000) + '\n```';
            await github.rest.issues.createComment({
              owner: context.repo.owner, repo: context.repo.repo,
              issue_number: context.issue.number, body
            });
```

- [ ] **Step 2: Commit**

```bash
cd "$REPO_ROOT/okta-dev"
terraform fmt -recursive
git add .github/workflows/plan.yml
git commit -m "ci: add plan workflow"
```

### Task D5: Add the `apply.yml` workflow

**Files:**
- Create: `$REPO_ROOT/okta-dev/.github/workflows/apply.yml`

- [ ] **Step 1: Write `apply.yml`**

```yaml
name: apply
on:
  push:
    branches: [main]
permissions:
  id-token: write
  contents: read
jobs:
  apply:
    runs-on: ubuntu-latest
    environment: dev
    env:
      ARM_USE_OIDC: "true"
      ARM_CLIENT_ID: ${{ secrets.ARM_CLIENT_ID }}
      ARM_TENANT_ID: ${{ secrets.ARM_TENANT_ID }}
      ARM_SUBSCRIPTION_ID: ${{ secrets.ARM_SUBSCRIPTION_ID }}
      OKTA_API_CLIENT_ID: ${{ secrets.OKTA_API_CLIENT_ID }}
      OKTA_API_PRIVATE_KEY: ${{ secrets.OKTA_API_PRIVATE_KEY }}
      OKTA_API_PRIVATE_KEY_ID: ${{ secrets.OKTA_API_PRIVATE_KEY_ID }}
      OKTA_API_SCOPES: ${{ secrets.OKTA_API_SCOPES }}
    steps:
      - uses: actions/checkout@v4
      - uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: "1.9.8"
      - run: terraform init
      - run: terraform apply -auto-approve -input=false
```

- [ ] **Step 2: Commit and push to main**

```bash
git add .github/workflows/apply.yml
git commit -m "ci: add apply workflow"
git push -u origin main
```

### Task D6: Prove the slice via a PR (plan) then merge (apply)

- [ ] **Step 1: Open a PR that changes a group description**

```bash
cd "$REPO_ROOT/okta-dev"
git checkout -b slice-test
sed -i 's/Managed by Terraform (DEV)/Managed by Terraform (DEV) - slice test/' dev.auto.tfvars
git commit -am "test: tweak group description to exercise plan"
git push -u origin slice-test
gh pr create --fill --base main
```

- [ ] **Step 2: Verify `plan.yml` succeeds and posts a plan comment**

```bash
gh pr checks --watch
gh pr view --comments | grep -i "okta_group" | head
```
Expected: the `plan` check is green and a comment shows `module.groups.okta_group.this["tf-managed-engineering"]` will be created/updated.

- [ ] **Step 3: Merge to trigger apply**

```bash
gh pr merge --squash --delete-branch
gh run list --repo "$GH_OWNER/okta-dev" --workflow apply.yml --limit 1
gh run watch
```
Expected: `apply` run completes `success`.

- [ ] **Step 4: Verify the group exists in the DEV org**

In the DEV Okta Admin Console: **Directory → Groups** → confirm `tf-managed-engineering` is present. (Or query the API with the service-app token.) This is the end-to-end proof.

### Task D7: Confirm state landed in Azure

- [ ] **Step 1: Verify the dev state blob exists**

```bash
az storage blob list --account-name "$AZ_SA" --container-name tfstate-dev \
  --auth-mode login --query "[].name" -o tsv
```
Expected: `okta.tfstate`

---

## Phase E — Replicate to TEST and PROD

> TEST and PROD repos are **structurally identical** to `okta-dev`. The steps below copy the dev repo and substitute the per-stage values. Repeat the full block once for TEST, once for PROD, using the substitution table.

| Token | TEST value | PROD value |
|---|---|---|
| repo name | `okta-test` | `okta-prod` |
| container | `tfstate-test` | `tfstate-prod` |
| Entra app | `okta-test-tf` | `okta-prod-tf` |
| environment | `test` | `prod` |
| org var | `$OKTA_ORG_TEST` | `$OKTA_ORG_PROD` |
| base_url | `oktapreview.com` | `okta.com` |
| Okta service app | TEST org (repeat Task B1 in TEST org) | PROD org (repeat Task B1 in PROD org) |
| approval gate | optional | **required reviewers** |

### Task E1: Bootstrap the TEST org service app and repo

- [ ] **Step 1: Create the TEST org service app** — repeat all of Task B1 in the **TEST** Okta org. Save the PEM to `~/.okta-tf-secrets/test.pem` (chmod 600). Record Client ID + Key ID.

- [ ] **Step 2: Copy the dev repo as the test repo and fix per-stage values**

```bash
cd "$REPO_ROOT"
gh repo create "$GH_OWNER/okta-test" --private
cp -r okta-dev okta-test
cd okta-test
rm -rf .git
git init -b main
sed -i "s/tfstate-dev/tfstate-test/" backend.tf
sed -i "s/Managed by Terraform (DEV).*/Managed by Terraform (TEST)\"/" dev.auto.tfvars
mv dev.auto.tfvars test.auto.tfvars
sed -i "s/$OKTA_ORG_DEV/$OKTA_ORG_TEST/" test.auto.tfvars
# main.tf module ref stays v1.0.0 for now; promotion bumps it later
git add -A
git commit -m "feat: okta-test root config (copied from okta-dev)"
git remote add origin "https://github.com/$GH_OWNER/okta-test.git"
git push -u origin main
```

- [ ] **Step 3: Verify the environment line in both workflows points at `test`**

```bash
sed -i "s/environment: dev/environment: test/" .github/workflows/plan.yml .github/workflows/apply.yml
git commit -am "ci: bind workflows to test environment"
git push
grep -n "environment:" .github/workflows/*.yml
```
Expected: both files show `environment: test`.

### Task E2: TEST Azure identity + GitHub environment

- [ ] **Step 1: Create the TEST federated identity** — repeat Task D2 substituting `okta-test-tf`, `okta-test`, `tfstate-test`, environment `test`. Capture `TEST_APP_ID`.

```bash
az ad app create --display-name "okta-test-tf"
export TEST_APP_ID="$(az ad app list --display-name 'okta-test-tf' --query '[0].appId' -o tsv)"
az ad sp create --id "$TEST_APP_ID"
az ad app federated-credential create --id "$TEST_APP_ID" --parameters "{
  \"name\": \"okta-test-env\",
  \"issuer\": \"https://token.actions.githubusercontent.com\",
  \"subject\": \"repo:$GH_OWNER/okta-test:environment:test\",
  \"audiences\": [\"api://AzureADTokenExchange\"]
}"
az role assignment create --assignee "$TEST_APP_ID" \
  --role "Storage Blob Data Contributor" \
  --scope "/subscriptions/$AZ_SUB/resourceGroups/$AZ_RG/providers/Microsoft.Storage/storageAccounts/$AZ_SA/blobServices/default/containers/tfstate-test"
```

- [ ] **Step 2: Create the `test` environment + secrets** — repeat Task D3 with `--env test`, `--repo "$GH_OWNER/okta-test"`, `$TEST_APP_ID`, and `~/.okta-tf-secrets/test.pem` plus the TEST Client ID / Key ID.

```bash
gh api --method PUT "repos/$GH_OWNER/okta-test/environments/test"
gh secret set ARM_CLIENT_ID       --env test --repo "$GH_OWNER/okta-test" --body "$TEST_APP_ID"
gh secret set ARM_TENANT_ID       --env test --repo "$GH_OWNER/okta-test" --body "$AZ_TENANT"
gh secret set ARM_SUBSCRIPTION_ID --env test --repo "$GH_OWNER/okta-test" --body "$AZ_SUB"
gh secret set OKTA_API_CLIENT_ID      --env test --repo "$GH_OWNER/okta-test" --body "PASTE_TEST_CLIENT_ID"
gh secret set OKTA_API_PRIVATE_KEY_ID --env test --repo "$GH_OWNER/okta-test" --body "PASTE_TEST_KEY_ID"
gh secret set OKTA_API_PRIVATE_KEY    --env test --repo "$GH_OWNER/okta-test" < ~/.okta-tf-secrets/test.pem
gh secret set OKTA_API_SCOPES         --env test --repo "$GH_OWNER/okta-test" \
  --body "okta.groups.manage,okta.groups.read,okta.apps.manage,okta.apps.read,okta.policies.manage,okta.policies.read,okta.authorizationServers.manage,okta.authorizationServers.read,okta.idps.manage,okta.idps.read,okta.brands.manage,okta.brands.read"
```

- [ ] **Step 3: Verify TEST apply succeeds** — the push in E1 already triggered `apply.yml`; re-run if it ran before secrets existed.

```bash
gh run list --repo "$GH_OWNER/okta-test" --workflow apply.yml --limit 1
# if it failed due to missing secrets, re-run now:
gh run rerun "$(gh run list --repo "$GH_OWNER/okta-test" --workflow apply.yml --limit 1 --json databaseId -q '.[0].databaseId')"
gh run watch --repo "$GH_OWNER/okta-test"
```
Expected: `apply` completes `success`; group appears in the TEST org.

### Task E3: PROD repo, identity, environment — with approval gate

- [ ] **Step 1: Create the PROD org service app** — repeat Task B1 in the **PROD** org. Save PEM to `~/.okta-tf-secrets/prod.pem`.

- [ ] **Step 2: Copy and fix the prod repo**

```bash
cd "$REPO_ROOT"
gh repo create "$GH_OWNER/okta-prod" --private
cp -r okta-dev okta-prod
cd okta-prod
rm -rf .git
git init -b main
sed -i "s/tfstate-dev/tfstate-prod/" backend.tf
mv dev.auto.tfvars prod.auto.tfvars
sed -i "s/oktapreview.com/okta.com/" prod.auto.tfvars
sed -i "s/$OKTA_ORG_DEV/$OKTA_ORG_PROD/" prod.auto.tfvars
sed -i "s/Managed by Terraform (DEV).*/Managed by Terraform (PROD)\"/" prod.auto.tfvars
sed -i "s/environment: dev/environment: prod/" .github/workflows/plan.yml .github/workflows/apply.yml
git add -A
git commit -m "feat: okta-prod root config (copied from okta-dev)"
git remote add origin "https://github.com/$GH_OWNER/okta-prod.git"
```

- [ ] **Step 3: Create PROD federated identity** — repeat Task E2 Step 1 with `okta-prod-tf`, `okta-prod`, `tfstate-prod`, environment `prod`. Capture `PROD_APP_ID`.

- [ ] **Step 4: Create the `prod` environment WITH required reviewers**

```bash
# Replace REVIEWER_USER_ID with the GitHub numeric user id of an approver
# (get it: gh api users/<login> --jq .id)
gh api --method PUT "repos/$GH_OWNER/okta-prod/environments/prod" \
  -f "reviewers[][type]=User" -F "reviewers[][id]=REVIEWER_USER_ID"
```

- [ ] **Step 5: Set PROD secrets** — repeat Task D3 with `--env prod`, `--repo "$GH_OWNER/okta-prod"`, `$PROD_APP_ID`, `~/.okta-tf-secrets/prod.pem`, PROD Client ID / Key ID. (Same six `gh secret set` calls as Task E2 Step 2 with prod substitutions.)

- [ ] **Step 6: Push and verify apply waits for approval**

```bash
cd "$REPO_ROOT/okta-prod"
git push -u origin main
gh run list --repo "$GH_OWNER/okta-prod" --workflow apply.yml --limit 1
```
Expected: the `apply` run shows status `waiting` (blocked on the environment approval), NOT auto-applied.

- [ ] **Step 7: Approve and confirm**

Approve in the GitHub UI (Actions → the waiting run → Review deployments → Approve), then:
```bash
gh run watch --repo "$GH_OWNER/okta-prod"
```
Expected: `apply` completes `success`; group appears in the PROD org.

---

## Phase F — Promotion + drift detection

### Task F1: Prove the promotion lever (a v1.1.0 change flows DEV→TEST→PROD)

- [ ] **Step 1: Make a change in `okta-base-config` and release v1.1.0**

Add a second group to the example and tag a release:
```bash
cd "$REPO_ROOT/okta-base-config"
# (the module is already data-driven; a "change" here is any module improvement)
git tag v1.1.0 && git push origin v1.1.0
```

- [ ] **Step 2: Promote to DEV via a ref bump PR**

```bash
cd "$REPO_ROOT/okta-dev"
git checkout -b promote-v1.1.0
sed -i 's|?ref=v1.0.0|?ref=v1.1.0|' main.tf
# also add the new group to dev's tfvars if the release expects it
git commit -am "chore: promote base-config to v1.1.0"
git push -u origin promote-v1.1.0
gh pr create --fill --base main
gh pr checks --watch
```
Expected: plan succeeds showing the new version's effect.

- [ ] **Step 3: Merge to apply in DEV, then repeat the ref bump in okta-test, then okta-prod**

```bash
gh pr merge --squash --delete-branch
```
Then repeat Step 2–3 in `okta-test`, validate, then `okta-prod` (which will pause for approval). This is the promotion flow working end to end.

- [ ] **Step 4: Verify all three repos pin v1.1.0**

```bash
for r in okta-dev okta-test okta-prod; do
  echo "$r:"; grep ref= "$REPO_ROOT/$r/main.tf"
done
```
Expected: each shows `?ref=v1.1.0`.

### Task F2: Add nightly drift detection (optional but recommended)

**Files:**
- Create: `$REPO_ROOT/okta-dev/.github/workflows/drift.yml` (and copy to test/prod with env substitution)

- [ ] **Step 1: Write `drift.yml`**

```yaml
name: drift
on:
  schedule:
    - cron: "0 6 * * *"
  workflow_dispatch:
permissions:
  id-token: write
  contents: read
jobs:
  drift:
    runs-on: ubuntu-latest
    environment: dev
    env:
      ARM_USE_OIDC: "true"
      ARM_CLIENT_ID: ${{ secrets.ARM_CLIENT_ID }}
      ARM_TENANT_ID: ${{ secrets.ARM_TENANT_ID }}
      ARM_SUBSCRIPTION_ID: ${{ secrets.ARM_SUBSCRIPTION_ID }}
      OKTA_API_CLIENT_ID: ${{ secrets.OKTA_API_CLIENT_ID }}
      OKTA_API_PRIVATE_KEY: ${{ secrets.OKTA_API_PRIVATE_KEY }}
      OKTA_API_PRIVATE_KEY_ID: ${{ secrets.OKTA_API_PRIVATE_KEY_ID }}
      OKTA_API_SCOPES: ${{ secrets.OKTA_API_SCOPES }}
    steps:
      - uses: actions/checkout@v4
      - uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: "1.9.8"
      - run: terraform init
      - name: Detect drift
        run: terraform plan -detailed-exitcode -input=false
        # exit code 2 = drift detected → job fails → notification
```

- [ ] **Step 2: Commit to okta-dev, then copy to test/prod with env substitution**

```bash
cd "$REPO_ROOT/okta-dev"
git add .github/workflows/drift.yml
git commit -m "ci: nightly drift detection"
git push
for r in okta-test okta-prod; do
  env="${r#okta-}"
  sed "s/environment: dev/environment: $env/" .github/workflows/drift.yml > "$REPO_ROOT/$r/.github/workflows/drift.yml"
  (cd "$REPO_ROOT/$r" && git add .github/workflows/drift.yml && git commit -m "ci: nightly drift detection" && git push)
done
```

- [ ] **Step 3: Verify a manual drift run is green**

```bash
gh workflow run drift.yml --repo "$GH_OWNER/okta-dev"
gh run watch --repo "$GH_OWNER/okta-dev"
```
Expected: `drift` completes `success` (no drift) immediately after an apply.

---

## Self-review notes

**Spec coverage:** Architecture (Phases C–E) ✓; base-config repo + modules (C) ✓; env repos shared shape (D, E) ✓; Azure one-account/three-containers + locking + AAD auth + container-scoped RBAC (A, D2, E2, E3) ✓; OAuth2 service apps + bootstrap-out-of-state + identity chain (B, E1, E3) ✓; OIDC federation (D2/E2/E3) ✓; plan-on-PR / apply-on-merge / PROD approval gate (D4, D5, E3) ✓; direct ref-bump promotion (F1) ✓; structural-config-only / no `okta_user` (groups module only; users module omitted) ✓; local dev = personal org + local state (Prerequisites note; no backend access granted to devs) ✓; drift detection (F2) ✓.

**Scope note:** This plan delivers the *machinery* plus one real resource (a group) flowing end to end. Authoring the full set of Okta modules (apps, policies, auth servers, branding) and the actual org content is additive follow-on work: each is a new module in `okta-base-config` (same pattern as `groups`) plus entries in each env's tfvars — it does not change any pipeline built here.

**Type consistency:** `var.groups` is `map(object({ description = string }))` in the module (C2), in the env `variables.tf` (D1 Step 7), and in every tfvars (D1 Step 9, E, F) — consistent throughout. The module output `group_ids` is defined once (C2) and not relied on by later tasks.

**Manual steps flagged:** Okta service-app creation (B1, E1, E3) and the PROD approval click (E3 Step 7) are necessarily human/console actions and are marked as such.
