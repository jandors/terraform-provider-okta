# okta-dev-local — Developer-Only Local-State Environment — Design

**Date:** 2026-06-18
**Status:** Approved (design phase)
**Related:** [2026-06-15-okta-multi-stage-management-design.md](2026-06-15-okta-multi-stage-management-design.md)

## Goal

Add a shared starter repo, **`okta-dev-local`**, that standardizes the
developer local-development workflow defined (but left informal) in the
multi-stage design. Developers clone it, point it at their **own throwaway Okta
org**, and run Terraform **by hand with local state** — no Azure backend, no
CI apply, no shared state.

## Why this exists

The multi-stage design states: *"local dev = each developer uses a personal
throwaway dev org + local state; developers never touch shared dev/test/prod
state."* That was a principle with no artifact. This repo turns it into a
concrete, version-controlled scaffold so every developer starts from the same
known-good setup instead of hand-rolling one.

## Decisions (locked)

| Area | Decision |
|---|---|
| Form | New shared repo `okta-dev-local` (cloned by developers; not a per-dev template). |
| Target org | Each developer's **own** personal throwaway Okta org. Never a shared org. |
| State | **Local** (`backend "local"`), `terraform.tfstate` on the dev's machine, gitignored. No Azure. |
| Auth | **API token (SSWS)** via the `OKTA_API_TOKEN` env var. Each dev generates their own token. |
| Module source | **Local sibling path** `../okta-base-config/modules/...` (dev clones both repos side by side). |
| Apply automation | **None.** Developers run `terraform plan/apply/destroy` manually. No apply workflow, ever. |
| CI | **Minimal**: PR-only `fmt -check` + `validate` (checks out `okta-base-config` as a sibling). |
| tfvars | Personal + gitignored; a committed `.example` is the template. |

## How it differs from the env repos (`okta-dev/test/prod`)

| Aspect | env repos | `okta-dev-local` |
|---|---|---|
| State backend | Azure Blob (per-stage container) | Local file |
| Auth | OAuth2 service app (private-key JWT) | API token (SSWS) env var |
| Module source | Pinned git `?ref=vX.Y.Z` | Local sibling path |
| CI | plan on PR, apply on merge, drift | fmt + validate only |
| tfvars | Committed (shared config) | Personal, gitignored (`.example` committed) |
| Target org | One shared org per stage | Each dev's own throwaway org |

Everything else is identical — crucially, it consumes the **same
`okta-base-config` modules**, so what a developer validates locally is the same
module code that flows through the promotion pipeline.

## Repository layout

```
okta-dev-local/
├── README.md                      # the developer workflow (below)
├── .gitignore                     # .terraform/, *.tfstate*, *.auto.tfvars (keep .example), *.pem, .env
├── versions.tf                    # terraform { backend "local" {}; required_providers { okta ~> 6.12 } }
├── providers.tf                   # provider "okta" { org_name, base_url }  (token via OKTA_API_TOKEN)
├── variables.tf                   # org_name, base_url, groups (same shape as env repos)
├── main.tf                        # module "groups" { source = "../okta-base-config/modules/groups" }
├── dev-local.auto.tfvars.example  # committed template; dev copies to dev-local.auto.tfvars (gitignored)
└── .github/workflows/ci.yml       # PR: fmt -check + validate, base-config checked out as sibling
```

Expected developer machine layout (siblings):
```
<workspace>/
├── okta-base-config/     # cloned alongside; reached via ../okta-base-config
└── okta-dev-local/
```

### File specifics

- **`versions.tf`** — explicit `backend "local" {}` (state in `terraform.tfstate`
  in the repo dir, gitignored). Explicit, not omitted, so Azure can never be
  wired here by accident.
- **`providers.tf`** — minimal block; the SSWS token is read from the
  `OKTA_API_TOKEN` env var by the provider, so no secret lands in any file.
- **`main.tf`** — `source = "../okta-base-config/modules/groups"`, passing
  `var.groups`. A dev can edit a module and immediately `plan` it with no
  push/tag.
- **`dev-local.auto.tfvars.example`** — sample `org_name`, `base_url`
  (`oktapreview.com`), and a sample `groups` entry. The real
  `dev-local.auto.tfvars` is gitignored.

## CI design

The module source is a local sibling path, so CI reconstructs that layout by
checking out both repos into named directories, then validates with the backend
disabled (no creds, no state, no apply):

```yaml
on: pull_request
jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with: { path: okta-dev-local }
      - uses: actions/checkout@v4
        with:
          repository: jandors/okta-base-config
          path: okta-base-config
          token: ${{ secrets.BASE_CONFIG_READ_TOKEN }}
      - uses: hashicorp/setup-terraform@v3
        with: { terraform_version: "1.9.8" }
      - working-directory: okta-dev-local
        run: |
          terraform fmt -check -recursive
          terraform init -backend=false
          terraform validate
```

**Dependency to satisfy:** `okta-base-config` is private, and a workflow's
default `GITHUB_TOKEN` can only read its own repo. The second checkout therefore
needs a credential with read access — a PAT stored as the repo secret
`BASE_CONFIG_READ_TOKEN` (or a deploy key / GitHub App). This is the only
infra setup CI requires. Fallback if undesirable: drop to `fmt -check` only and
remove the base-config checkout (loses `validate`).

## Developer workflow (README content)

One-time setup:
1. Create a personal Okta org (free Integrator/developer org) — the throwaway sandbox.
2. In its Admin Console create a personal **API token** (Security → API → Tokens).
3. Clone both repos side by side:
   ```bash
   git clone https://github.com/jandors/okta-base-config.git
   git clone https://github.com/jandors/okta-dev-local.git
   ```
4. `cp dev-local.auto.tfvars.example dev-local.auto.tfvars` and set `org_name` + `base_url`.

Each session:
```bash
cd okta-dev-local
export OKTA_API_TOKEN=00your_personal_token
terraform init      # local backend; resolves ../okta-base-config modules
terraform plan
terraform apply     # creates resources in YOUR org only
terraform destroy   # tear down when done
```

## Guardrails

- **Local state only** — `terraform.tfstate` stays on the dev's machine,
  gitignored; no shared state to corrupt, no locking needed (single user).
- **No apply workflow** — nothing in this repo touches a shared org or runs with
  creds in CI. Blast radius = exactly one personal org.
- **Same modules as prod** — consumes `okta-base-config`, so local validation
  reflects the real module code; only version resolution (local path vs pinned
  tag) and backend/auth differ.

## Out of scope

- Any shared/sandbox org applied to by multiple developers (explicitly rejected
  — shared state + local backend is unsafe).
- OAuth2 auth for this repo (env repos keep that; local dev uses API tokens).
- Azure backend, remote state, or state locking.
- Apply/promotion automation.
