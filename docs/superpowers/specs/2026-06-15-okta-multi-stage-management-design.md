# Managing Okta DEV / TEST / PROD with Terraform — Design

**Date:** 2026-06-15
**Status:** Approved (design phase)

## Goal

Stand up a Terraform-based workflow to manage three separate Okta orgs —
**DEV**, **TEST**, **PROD** — by *consuming* the published `okta/okta` provider
(not building a custom provider). The design covers repository topology, local
development, Okta and Azure requirements, and a GitHub Actions CI/CD promotion
pipeline.

## Decisions (locked)

| Area | Decision |
|---|---|
| Provider | Consume published `okta/okta` from the Terraform Registry (`~> 6.x`). No custom build. |
| Okta topology | 3 separate Okta orgs (DEV/TEST preview orgs + PROD). |
| Config sharing | One central **`okta-base-config`** repo (reusable modules + base elements), released via semver git tags. |
| Env topology | One git repo per stage: **`okta-dev`**, **`okta-test`**, **`okta-prod`**, each with its own plan/apply pipeline. |
| State backend | Azure Blob Storage, **one storage account, three containers** (`tfstate-dev/test/prod`), native blob-lease locking, AzureAD auth (account key disabled). |
| Okta auth | OAuth 2.0 **service app per org** using private-key JWT (client-credentials) flow. Least-privilege scopes. |
| Azure CI auth | GitHub **OIDC federation** to Entra ID apps — no stored Azure secrets. |
| CI/CD | GitHub Actions; PR runs plan, merge-to-main applies; PROD gated by GitHub Environment manual approval. |
| Promotion | Bump the `?ref=` module version pin per env repo via PR (direct ref bump). |
| Scope of config | **Structural config only** — groups, group rules, apps, assignments, policies, auth servers, branding. **No `okta_user` CRUD.** |
| Local dev | Each developer uses a **personal throwaway dev org + local state**. Devs never touch shared dev/test/prod state or CI credentials. |

## Architecture

```
                 ┌─────────────────────────────┐
                 │   okta-base-config (repo)    │
                 │  reusable TF modules + base  │
                 │  config; released via tags   │
                 └─────────────┬───────────────┘
            pinned @v1.2  pinned @v1.1  pinned @v1.0
                 │             │             │
       ┌─────────▼───┐  ┌──────▼──────┐  ┌───▼─────────┐
       │  okta-dev   │  │  okta-test  │  │  okta-prod  │
       │  root TF +  │  │  root TF +  │  │  root TF +  │
       │  GHA pipe   │  │  GHA pipe   │  │  GHA pipe   │
       └──────┬──────┘  └──────┬──────┘  └──────┬──────┘
              │                │                │
   Azure Blob │     Azure Blob │     Azure Blob │  (1 account, 3 containers,
   tfstate-dev│     tfstate-test│    tfstate-prod│   OIDC + AzureAD auth)
              │                │                │
       ┌──────▼──────┐  ┌──────▼──────┐  ┌──────▼──────┐
       │  Okta DEV   │  │  Okta TEST  │  │  Okta PROD  │
       │  OAuth2 svc │  │  OAuth2 svc │  │  OAuth2 svc │
       │  app        │  │  app        │  │  app        │
       └─────────────┘  └─────────────┘  └─────────────┘
```

`okta-base-config` is the single source of truth for *what* the config looks
like. Each env repo decides *which version* it runs and *against which org*,
supplying only environment-specific values (org name, base URL, credentials,
backend, tfvars). Changes flow DEV → TEST → PROD by bumping the version pin.
Each env repo runs an independent plan/apply, so a DEV apply can never touch
PROD.

**Variant chosen:** `okta-base-config` holds *modules* (definitions) only; env
repos instantiate them. Resources are not defined once-and-identical across
orgs — each env instantiates with its own inputs, preserving per-env flexibility.

## Repository layouts

### `okta-base-config`

```
okta-base-config/
├── modules/
│   ├── groups/            # okta_group, okta_group_rule
│   ├── apps/              # okta_app_oauth, okta_app_saml, assignments
│   ├── auth_policies/     # okta_policy_*, okta_app_signon_policy*
│   ├── authservers/       # okta_auth_server + scopes/claims/policies
│   └── branding/          # brand, email templates, customization
│       ├── main.tf  variables.tf  outputs.tf  README.md
├── examples/              # how to call each module; doubles as CI target + docs
├── CHANGELOG.md           # per-tag change log — drives promotion decisions
├── .github/workflows/ci.yml   # fmt -check + validate + tflint on PR (NO apply)
└── README.md
```

Rules:
- **No backend, no credentials, never applied directly.** CI runs only
  `terraform fmt -check`, `validate`, and `tflint` against `examples/`.
- Every promotable merge to `main` gets a **semver git tag** (`v1.3.0`). The tag
  is the unit of promotion.
- Modules expose inputs for everything that legitimately differs by environment
  (names, counts, feature toggles); nothing org-specific is hardcoded.
- The `users_lifecycle` module is **out of scope** (structural config only).

### Each env repo (`okta-dev` / `okta-test` / `okta-prod` — identical shape)

```
okta-dev/
├── main.tf            # module blocks; every source pinned to the SAME ?ref=vX.Y.Z
├── providers.tf       # okta provider config (values injected, never committed)
├── backend.tf         # azurerm backend → container tfstate-dev, key okta.tfstate
├── variables.tf       # env input declarations
├── dev.auto.tfvars    # org_name, base_url, feature toggles
├── versions.tf        # required_version + required_providers (okta ~> 6.x)
├── .github/workflows/
│   ├── plan.yml       # on PR: fmt/validate/plan, post plan as PR comment
│   └── apply.yml      # on merge to main: apply (PROD gated by Environment approval)
└── README.md
```

Module source example:
```hcl
module "groups" {
  source = "git::https://github.com/<org>/okta-base-config.git//modules/groups?ref=v1.3.0"
  # ...env-specific inputs...
}
```

What differs between the three env repos:
- `backend.tf` → container `tfstate-dev` / `-test` / `-prod`
- `*.auto.tfvars` → `org_name`, `base_url` (`oktapreview.com` for dev/test,
  `okta.com` for prod), per-stage toggles
- the `?ref=` version pin in every module block (the promotion lever)
- GitHub Environment + its secrets/vars (that org's OAuth2 creds, that stage's
  Azure scope)
- `apply.yml` → PROD adds a manual-approval GitHub Environment

**Single root per env repo** (one state file per org). Splitting into multiple
states for blast-radius isolation is deferred (YAGNI) until an org's config
grows large enough to warrant it.

## Okta authentication & requirements

### OAuth2 service app (one per org, bootstrapped once)

The provider authenticates as an **OAuth 2.0 service app using the private-key
JWT (client-credentials) flow**. Per org:

1. Create an **API Services / OAuth2 service app** in the org.
2. Generate a **public/private key pair**. The app holds the public JWK;
   Terraform holds the private key as a secret. No password, no long-lived token.
3. **Grant least-privilege API scopes** matching what the modules manage, e.g.
   `okta.groups.manage`, `okta.apps.manage`, `okta.policies.manage`,
   `okta.authorizationServers.manage`, `okta.idps.manage`, `okta.brands.manage`,
   plus matching `.read` scopes. (No `okta.users.manage` — structural only;
   `okta.groups.manage` covers group rules.)
4. Bind an **admin role** to the app — prefer a scoped custom admin role over
   Super Admin where the managed resources allow it. Document the role each
   stage requires.

Provider block (each env repo's `providers.tf`; values injected, never committed):
```hcl
provider "okta" {
  org_name       = var.org_name          # e.g. "dev-12345"
  base_url       = var.base_url           # oktapreview.com | okta.com
  client_id      = var.okta_client_id
  scopes         = var.okta_scopes
  private_key    = var.okta_private_key   # PEM, from secret
  private_key_id = var.okta_private_key_id
}
```

### Bootstrap chicken-and-egg

The service app **cannot create itself**. For each org the first setup is
manual (or a one-off admin-run script):

1. An Okta admin creates the service app + key pair + scope grants + admin role.
2. Records `client_id`, scope list, private key.
3. Stores them in the env repo's GitHub Environment secrets.

The service app itself is **kept out of Terraform state** so that a bad apply or
`destroy` can never revoke the credentials Terraform is currently using.

### Identity chain ("local development to user accounts")

| Level | Who/what | Credentials | Access |
|---|---|---|---|
| Local dev | Developer running `plan` locally | Their **own** personal **throwaway dev org**, **local state** | Personal dev org only. Never shared state or CI creds. |
| CI service identity | GitHub Actions runner | Per-org **service app** private key (GitHub Environment secret) | That one stage's org only. |
| Okta admin (human) | Person bootstrapping each org | Super Admin console login | One-time service-app creation + break-glass. |

Local-dev requirements:
- A personal Okta Integrator/developer org per developer (free), for
  experimentation with **local state**.
- Installed locally: Terraform/OpenTofu, the `okta` provider (auto-installed by
  `terraform init`), git.
- A **gitignored** `*.tfvars`/`.env` for personal-org credentials — never
  committed.
- Read access to `okta-base-config` to resolve pinned module sources.

## Azure Blob state backend

One storage account, three containers:
```
Storage account: stoktatfstate (RG: rg-okta-tfstate)
├── container: tfstate-dev    → key: okta.tfstate
├── container: tfstate-test   → key: okta.tfstate
└── container: tfstate-prod   → key: okta.tfstate
```

`backend.tf` (dev shown):
```hcl
terraform {
  backend "azurerm" {
    resource_group_name  = "rg-okta-tfstate"
    storage_account_name = "stoktatfstate"
    container_name       = "tfstate-dev"   # differs per env repo
    key                  = "okta.tfstate"
    use_oidc             = true
    use_azuread_auth     = true             # account key disabled
  }
}
```

- **Locking:** native Azure blob leasing — no external lock table needed.
- **Access:** account key disabled; access via Entra ID RBAC. Each stage's
  GitHub federated identity gets `Storage Blob Data Contributor` on **only its
  own container**. Developers need no backend access (local state). Break-glass
  humans get PIM-eligible roles.
- **Bootstrap:** the storage account + containers + RBAC must exist before any
  env repo can `init`. Provisioned out of band via a small dedicated
  `tf-bootstrap` config (local state), documented and reproducible — not part of
  the okta repos.

## GitHub Actions CI/CD

### Promotion mechanism

Terraform requires `source`/`?ref=` to be string literals (cannot be driven by
a variable). Promotion is therefore a reviewable **direct ref bump**: a PR in the
next env repo changing every module `?ref=vX.Y.Z` to the new tag. The PR diff is
the promotion record, and `plan.yml` shows the effect before approval. A
`make promote VERSION=vX.Y.Z` helper does the local find-replace to keep it a
single action.

(Alternative considered and rejected as default: git submodule of
`okta-base-config` referenced by local path — avoids network fetch but adds
submodule friction.)

### Pipelines (per env repo)

**`plan.yml`** — on `pull_request`:
```
permissions: { id-token: write, contents: read, pull-requests: write }
jobs.plan:
  environment: dev           # binds env secrets/vars
  steps:
    - checkout
    - azure/login@v2          # OIDC, no stored secret
    - setup terraform
    - terraform fmt -check
    - terraform init          # reads tfstate-dev via OIDC + AzureAD
    - terraform validate
    - terraform plan -out=tf.plan   # Okta creds from env
    - post plan summary as PR comment
```

**`apply.yml`** — on `push` to `main`:
```
permissions: { id-token: write, contents: read }
jobs.apply:
  environment: dev           # PROD repo uses environment: prod w/ required reviewers
  steps: [ checkout, azure/login, setup terraform, init, apply -auto-approve ]
```

The three repos are identical except for which **GitHub Environment** the jobs
bind and the values behind it.

### GitHub Environments (isolation + gates)

| | `okta-dev` | `okta-test` | `okta-prod` |
|---|---|---|---|
| Required reviewers | none (auto-apply) | optional | **yes — manual approval** |
| Azure federated identity | dev app → `tfstate-dev` | test app | prod app |
| Okta secrets | DEV service app | TEST | PROD |
| `OKTA_BASE_URL` var | `oktapreview.com` | `oktapreview.com` | `okta.com` |

Per-environment secrets: `OKTA_CLIENT_ID`, `OKTA_PRIVATE_KEY`,
`OKTA_PRIVATE_KEY_ID`, `OKTA_SCOPES`, plus Azure `ARM_CLIENT_ID` /
`ARM_TENANT_ID` / `ARM_SUBSCRIPTION_ID` (no client secret — OIDC). The Okta
private key is an Environment secret, so PROD's key is unreachable from dev/test
jobs.

### Azure OIDC federation (one-time per env repo)

Create an Entra ID app registration (or user-assigned managed identity) with a
**federated credential** scoped to the repo + environment, e.g. subject
`repo:<org>/okta-prod:environment:prod`. Grant `Storage Blob Data Contributor`
on **only its own container**. No Azure secrets stored in GitHub — the runner
exchanges its OIDC token for a short-lived Azure token at job time.

### Drift detection (recommended)

A scheduled `plan` (cron `workflow_dispatch`) per env repo that alerts if the
org has drifted from state due to out-of-band console changes.

## End-to-end promotion flow

```
1. PR into okta-base-config → CI validates → merge → tag v1.4.0
2. PR in okta-dev bumping refs → v1.4.0 → plan.yml posts plan
   → merge → apply.yml applies to DEV (auto)
3. Validate DEV. PR in okta-test → v1.4.0 → plan → merge → applies TEST
4. Validate TEST. PR in okta-prod → v1.4.0 → plan → merge
   → apply.yml waits on manual approval → approver reviews → applies PROD
```

## Step-by-step bootstrapping runbook

**Phase 0 — Azure backend (out of band, once)**
1. Create RG `rg-okta-tfstate` and storage account `stoktatfstate` with the
   account key disabled and AzureAD auth enabled.
2. Create containers `tfstate-dev`, `tfstate-test`, `tfstate-prod`.
3. Defer RBAC grants until the federated identities exist (Phase 3).

**Phase 1 — Okta orgs (out of band, once per org)**
4. For each org (DEV, TEST, PROD): create an OAuth2 service app, generate a key
   pair, grant least-privilege scopes, bind an admin role.
5. Record `client_id`, scopes, private key, `private_key_id`, `org_name`,
   `base_url` for each org. Keep the service app out of Terraform state.

**Phase 2 — `okta-base-config` repo**
6. Create the repo, author the modules (groups, apps, auth_policies,
   authservers, branding) + `examples/`.
7. Add `ci.yml` (fmt/validate/tflint, no apply). Merge, then tag `v1.0.0`.

**Phase 3 — Per env repo (`okta-dev` first, then test, then prod)**
8. Create the repo with `main.tf` (modules pinned to `v1.0.0`), `providers.tf`,
   `backend.tf` (its container), `variables.tf`, `*.auto.tfvars`, `versions.tf`.
9. Create an Entra ID app + federated credential scoped to `repo:<org>/<repo>:environment:<env>`;
   grant it `Storage Blob Data Contributor` on that repo's container only.
10. Create the GitHub Environment (`dev`/`test`/`prod`) with its secrets/vars;
    add required reviewers on `prod`.
11. Add `plan.yml` + `apply.yml`. Open a PR → confirm plan runs → merge →
    confirm apply against the org.

**Phase 4 — Operate**
12. Develop changes in personal throwaway dev orgs with local state.
13. Land changes in `okta-base-config`; tag a release.
14. Promote per the end-to-end flow above (DEV → TEST → PROD).
15. (Optional) enable scheduled drift-detection plans.

## Out of scope

- Building or distributing a custom/forked provider binary.
- `okta_user` lifecycle / real user account CRUD.
- Splitting an env into multiple state files (deferred until needed).
- A private module registry (using version-tagged git sources instead).
```
