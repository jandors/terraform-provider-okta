# okta-dev-local Environment Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Create the `okta-dev-local` repo — a shared starter that lets developers run the Okta modules against their own throwaway org with local Terraform state.

**Architecture:** A single root Terraform config that consumes `okta-base-config` modules via a local sibling path, uses the `backend "local"` state backend, authenticates with an SSWS API token from the `OKTA_API_TOKEN` env var, and runs only by hand (no apply automation). A minimal PR-only CI validates formatting and config.

**Tech Stack:** Terraform (`okta/okta ~> 6.12`, local backend), GitHub Actions (`fmt -check` + `validate`).

**Spec:** `docs/superpowers/specs/2026-06-18-okta-dev-local-environment-design.md`

## Global Constraints

- Repo location (local): `/mnt/c/Git/okta-css/okta-dev-local` (sibling of `okta-base-config`).
- GitHub owner: `jandors`. Remote: `https://github.com/jandors/okta-dev-local.git`.
- okta provider version: `~> 6.12`; Terraform `>= 1.6`. CI pins Terraform `1.9.8`.
- State is **local only** (`backend "local"`); never Azure. No `apply`/promotion automation, ever.
- Auth is **API token (SSWS)** via the `OKTA_API_TOKEN` env var — no credentials in any file.
- Module source is the **local sibling path** `../okta-base-config/modules/<name>` — not a git ref.
- `var.groups` type (matches `okta-base-config` groups module and the env repos): `map(object({ description = string }))`.
- The real `dev-local.auto.tfvars` is gitignored; `dev-local.auto.tfvars.example` is committed.
- This environment lives next to existing siblings: `okta-base-config/`, `okta-dev/`, `okta-test/`, `okta-prod/`.

## Verification note

`okta-base-config` is a **private** repo and this sandbox has no GitHub
credentials, so a real `terraform init` over the git module source can't run
here. Local validation uses the **sibling checkout** at
`/mnt/c/Git/okta-css/okta-base-config` (already present), which the local-path
module source resolves directly with no network — so `init -backend=false` +
`validate` work fully offline.

---

### Task 1: Scaffold the repo with the core Terraform config

**Files:**
- Create: `/mnt/c/Git/okta-css/okta-dev-local/.gitignore`
- Create: `/mnt/c/Git/okta-css/okta-dev-local/versions.tf`
- Create: `/mnt/c/Git/okta-css/okta-dev-local/providers.tf`
- Create: `/mnt/c/Git/okta-css/okta-dev-local/variables.tf`
- Create: `/mnt/c/Git/okta-css/okta-dev-local/main.tf`

**Interfaces:**
- Consumes: `okta-base-config` `modules/groups` (input `groups = map(object({ description = string }))`, output `group_ids`) via local path `../okta-base-config/modules/groups`.
- Produces: a root module with variables `org_name` (string), `base_url` (string), `groups` (map(object({description=string}))), default `{}`.

- [ ] **Step 1: Create the directory**

```bash
mkdir -p /mnt/c/Git/okta-css/okta-dev-local
cd /mnt/c/Git/okta-css/okta-dev-local
```

- [ ] **Step 2: Write `.gitignore`**

```gitignore
.terraform/
*.tfstate
*.tfstate.*
*.auto.tfvars
!*.auto.tfvars.example
*.pem
.env
crash.log
```

- [ ] **Step 3: Write `versions.tf`** (explicit local backend so Azure can never be wired here)

```hcl
terraform {
  required_version = ">= 1.6"

  backend "local" {}

  required_providers {
    okta = {
      source  = "okta/okta"
      version = "~> 6.12"
    }
  }
}
```

- [ ] **Step 4: Write `providers.tf`** (SSWS token read from `OKTA_API_TOKEN` env var)

```hcl
provider "okta" {
  org_name = var.org_name
  base_url = var.base_url
}
```

- [ ] **Step 5: Write `variables.tf`**

```hcl
variable "org_name" {
  description = "Your personal throwaway Okta org subdomain (e.g. dev-12345)."
  type        = string
}

variable "base_url" {
  description = "Okta base URL for your org (usually oktapreview.com)."
  type        = string
}

variable "groups" {
  description = "Groups to manage in your personal org."
  type = map(object({
    description = string
  }))
  default = {}
}
```

- [ ] **Step 6: Write `main.tf`** (local sibling-path module source)

```hcl
module "groups" {
  source = "../okta-base-config/modules/groups"
  groups = var.groups
}
```

- [ ] **Step 7: Verify formatting is clean**

Run: `terraform fmt -check -recursive`
Expected: no output, exit code 0. (If it reports files, run `terraform fmt -recursive` and re-check.)

- [ ] **Step 8: Verify the config validates against the sibling module (offline)**

Run:
```bash
terraform init -backend=false && terraform validate
```
Expected: `Success! The configuration is valid.`
(Resolves `../okta-base-config/modules/groups` from the local sibling checkout; no network.)

- [ ] **Step 9: Clean transient init artifacts**

Run: `rm -rf .terraform .terraform.lock.hcl`
(Reason: `.terraform/` is gitignored; removing keeps the tree tidy before git init.)

- [ ] **Step 10: Initialize git and commit**

```bash
git init -b main
git config user.name "Joseph Andor"
git config user.email "joseph.andor@ic-consult.ch"
git add .gitignore versions.tf providers.tf variables.tf main.tf
git commit -m "feat: scaffold okta-dev-local root config"
```
Expected: a commit containing exactly those 5 files (`git show --stat HEAD`).

---

### Task 2: Add the example tfvars template

**Files:**
- Create: `/mnt/c/Git/okta-css/okta-dev-local/dev-local.auto.tfvars.example`

**Interfaces:**
- Consumes: the `org_name`, `base_url`, `groups` variables from Task 1.
- Produces: a committed template a developer copies to `dev-local.auto.tfvars` (gitignored).

- [ ] **Step 1: Write `dev-local.auto.tfvars.example`**

```hcl
# Copy this file to dev-local.auto.tfvars (gitignored) and edit the values.
# dev-local.auto.tfvars is auto-loaded by terraform; the .example is not.

org_name = "dev-XXXXXXX"    # your personal Okta org subdomain
base_url = "oktapreview.com"

groups = {
  "tf-sandbox-engineering" = { description = "Local sandbox group" }
}
```

- [ ] **Step 2: Verify the example is NOT gitignored but a real tfvars WOULD be**

Run:
```bash
git check-ignore dev-local.auto.tfvars.example; echo "example ignored? exit=$?"
git check-ignore dev-local.auto.tfvars; echo "real ignored? exit=$?"
```
Expected: first prints nothing with `exit=1` (NOT ignored → will be committed); second prints `dev-local.auto.tfvars` with `exit=0` (ignored → never committed).

- [ ] **Step 3: Commit**

```bash
git add dev-local.auto.tfvars.example
git commit -m "docs: add example tfvars template"
```

---

### Task 3: Add the developer README

**Files:**
- Create: `/mnt/c/Git/okta-css/okta-dev-local/README.md`

**Interfaces:**
- Consumes: the workflow established by Tasks 1–2 (local path layout, `OKTA_API_TOKEN`, example tfvars).
- Produces: developer-facing documentation. No code depends on this.

- [ ] **Step 1: Write `README.md`**

````markdown
# okta-dev-local

A local-state Terraform sandbox for developing Okta config against your **own
throwaway Okta org**. Consumes the shared modules in `okta-base-config` via a
local path, so you can edit a module and test it immediately — no push or tag.

> This repo never touches shared dev/test/prod orgs or remote state. It uses
> **local state** and an **API token** scoped to your personal org.

## One-time setup

1. Create a personal Okta org (free Integrator/developer org) — your sandbox.
2. In its Admin Console: **Security → API → Tokens → Create token**. Copy it.
3. Clone both repos side by side:
   ```bash
   git clone https://github.com/jandors/okta-base-config.git
   git clone https://github.com/jandors/okta-dev-local.git
   ```
   Your layout must be:
   ```
   <workspace>/
   ├── okta-base-config/
   └── okta-dev-local/
   ```
4. Create your tfvars from the template and edit it:
   ```bash
   cd okta-dev-local
   cp dev-local.auto.tfvars.example dev-local.auto.tfvars
   # edit org_name and base_url
   ```

## Each session

```bash
cd okta-dev-local
export OKTA_API_TOKEN=00your_personal_token
terraform init      # local backend; resolves ../okta-base-config modules
terraform plan
terraform apply     # creates resources in YOUR org only
terraform destroy   # tear down when done
```

## Notes

- State lives in `terraform.tfstate` on your machine and is gitignored.
- To test an unreleased module change, just edit it in `../okta-base-config`
  and re-run `terraform plan` — the local path picks it up.
- This repo has no `apply` automation; everything runs by hand.
````

- [ ] **Step 2: Commit**

```bash
git add README.md
git commit -m "docs: add developer workflow README"
```

---

### Task 4: Add minimal CI (fmt + validate)

**Files:**
- Create: `/mnt/c/Git/okta-css/okta-dev-local/.github/workflows/ci.yml`

**Interfaces:**
- Consumes: the root config (Task 1) and the `okta-base-config` repo (checked out as a sibling at runtime).
- Produces: a PR gate. Requires repo secret `BASE_CONFIG_READ_TOKEN` (see Task 5) to read the private base-config repo.

- [ ] **Step 1: Write `.github/workflows/ci.yml`**

```yaml
name: ci
on:
  pull_request:
jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout okta-dev-local
        uses: actions/checkout@v4
        with:
          path: okta-dev-local
      - name: Checkout okta-base-config (sibling)
        uses: actions/checkout@v4
        with:
          repository: jandors/okta-base-config
          path: okta-base-config
          token: ${{ secrets.BASE_CONFIG_READ_TOKEN }}
      - uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: "1.9.8"
      - name: fmt + validate
        working-directory: okta-dev-local
        run: |
          terraform fmt -check -recursive
          terraform init -backend=false
          terraform validate
```

- [ ] **Step 2: Verify the workflow reproduces the sibling layout locally (smoke test)**

The CI relies on `okta-dev-local` and `okta-base-config` being siblings — which
is exactly the local layout. Confirm validate still passes from the repo root:
```bash
cd /mnt/c/Git/okta-css/okta-dev-local
terraform init -backend=false && terraform validate && rm -rf .terraform .terraform.lock.hcl
```
Expected: `Success! The configuration is valid.`

- [ ] **Step 3: Commit**

```bash
git add .github/workflows/ci.yml
git commit -m "ci: add fmt + validate workflow"
```

---

### Task 5: Publish the repo and wire CI access (outward-facing — operator runs)

**Files:** none (GitHub provisioning).

**Interfaces:**
- Consumes: the local repo from Tasks 1–4 and the existing `jandors/okta-base-config` repo.
- Produces: the published `jandors/okta-dev-local` remote with passing CI.

> This task requires GitHub access and cannot run from the build sandbox (no
> `gh`, no git credentials). The operator performs it.

- [ ] **Step 1: Create the empty remote**

On https://github.com/new create `jandors/okta-dev-local`, **Private**, with **no**
README/.gitignore/license.

- [ ] **Step 2: Add the remote and push**

```bash
cd /mnt/c/Git/okta-css/okta-dev-local
git remote add origin https://github.com/jandors/okta-dev-local.git
git push -u origin main
```

- [ ] **Step 3: Create the `BASE_CONFIG_READ_TOKEN` secret for CI**

Create a fine-grained PAT (or classic PAT) with **read access to
`jandors/okta-base-config`**, then add it as a repository secret:
- GitHub UI: `okta-dev-local` → Settings → Secrets and variables → Actions → New repository secret → name `BASE_CONFIG_READ_TOKEN`, value = the PAT.

(Alternative: add `okta-base-config` as a deploy-key-readable repo, or a GitHub App install. The PAT is the simplest.)

- [ ] **Step 4: Verify CI passes**

Open a trivial PR (e.g. tweak a comment in `README.md`) and confirm the `ci`
workflow runs `fmt -check` + `validate` and goes green. If the base-config
checkout fails with a 403/auth error, the `BASE_CONFIG_READ_TOKEN` secret is
missing or lacks read scope on `okta-base-config`.

---

## Self-review notes

**Spec coverage:** form = shared repo (Task 1, 5) ✓; local state `backend "local"` (Task 1 Step 3) ✓; API-token/`OKTA_API_TOKEN` auth (Task 1 Step 4, README Task 3) ✓; local sibling-path module source (Task 1 Step 6) ✓; no apply automation (only `ci.yml` exists; Task 4) ✓; minimal fmt+validate CI with sibling checkout + private-repo token dependency (Task 4, Task 5 Step 3) ✓; personal gitignored tfvars + committed `.example` (Task 1 Step 2 gitignore, Task 2) ✓; differences-from-env-repos preserved (no backend.tf, no OAuth, no plan/apply/drift workflows — nothing in the plan adds them) ✓; out-of-scope items (shared sandbox org, OAuth, Azure, promotion) — none introduced ✓.

**Placeholder scan:** no TBD/TODO; every file step has full content; `dev-XXXXXXX` and `00your_personal_token` are explicitly user-supplied values, not plan placeholders; `BASE_CONFIG_READ_TOKEN` is a named secret defined in Task 5.

**Type consistency:** `var.groups` is `map(object({ description = string }))` in `variables.tf` (Task 1 Step 5), the module call passes it straight through (Task 1 Step 6), the example tfvars matches the shape (Task 2 Step 1), and it agrees with the `okta-base-config` groups module input. `org_name`/`base_url` names match between `variables.tf`, `providers.tf`, and the example tfvars.

**Gitignore correctness:** `*.auto.tfvars` + `!*.auto.tfvars.example` — verified by the Task 2 Step 2 `git check-ignore` assertions (example tracked, real ignored).
