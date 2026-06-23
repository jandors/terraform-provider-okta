# okta-base-config Apps Modules (OIDC + SAML) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add `app_oauth` (OIDC) and `app_saml` (SAML SP-onboarding) modules to `okta-base-config`, each data-driven and managing optional group assignments, plus a developer-guide "Onboarding apps" section.

**Architecture:** Two modules under `modules/`, mirroring the existing `groups` module (a `var.apps` map keyed by app label, `for_each`). `app_oauth` uses a `flow` selector expanded via `locals` into the correct OIDC field combination, with raw overrides. Each module creates `okta_app_group_assignments` for apps that declare group IDs. Examples drive the existing offline CI (`fmt -check` + `init -backend=false` + `validate`).

**Tech Stack:** Terraform (`okta/okta >= 6.0.0`), HCL `optional()` object types, dynamic blocks. No new tooling.

**Spec:** `docs/superpowers/specs/2026-06-20-okta-apps-modules-design.md`

## Global Constraints

- Modules live in `/mnt/c/Git/okta-css/okta-base-config/modules/{app_oauth,app_saml}`; examples in `examples/{app_oauth,app_saml}`.
- Mirror the `groups` module conventions: `main.tf` + `variables.tf` + `outputs.tf` + `README.md`; provider pinned `okta/okta >= 6.0.0`; data-driven `var.apps` map keyed by app **label**; `default = {}`.
- **Verification per module = offline:** `terraform fmt -check -recursive`, then in the example dir `terraform init -backend=false` + `terraform validate` → `Success!`. No live org, no apply (real apply happens later in `okta-dev-local`).
- **Authoritative attribute names** (verified against the provider source — use exactly these):
  - `okta_app_oauth`: `label`, `type`, `grant_types`, `response_types`, `redirect_uris`, `post_logout_redirect_uris`, `token_endpoint_auth_method`, `pkce_required`, `status`; computed `client_id`, `client_secret`.
  - `okta_app_saml`: `label`, `sso_url`, `recipient`, `destination`, `audience`, `subject_name_id_template`, `subject_name_id_format`, `response_signed`, `assertion_signed`, `signature_algorithm`, `digest_algorithm`, `honor_force_authn`, `authn_context_class_ref`, `status`, `attribute_statements` block { `name`, `namespace`, `type`, `values`, `filter_type`, `filter_value` }.
  - `okta_app_group_assignments`: `app_id`, repeatable `group` block { `id`, `priority`, `profile` }.
- OIDC `flow` ∈ {`web_auth_code`, `spa_pkce`, `native_pkce`, `service_client_credentials`, `implicit`}; `status` ∈ {`ACTIVE`, `INACTIVE`}.
- Secrets never inputs: OIDC `client_secret` exposed only as a `sensitive` output.
- Commit author `Joseph Andor <joseph.andor@ic-consult.ch>` (already configured in the repo). Single foreground `git commit` per task (avoid background commits — they caused index contention).

---

### Task 1: `app_oauth` module + example

**Files:**
- Create: `/mnt/c/Git/okta-css/okta-base-config/modules/app_oauth/variables.tf`
- Create: `/mnt/c/Git/okta-css/okta-base-config/modules/app_oauth/main.tf`
- Create: `/mnt/c/Git/okta-css/okta-base-config/modules/app_oauth/outputs.tf`
- Create: `/mnt/c/Git/okta-css/okta-base-config/modules/app_oauth/README.md`
- Create: `/mnt/c/Git/okta-css/okta-base-config/examples/app_oauth/main.tf`

**Interfaces:**
- Consumes: the `okta` provider; group IDs supplied by the caller (e.g. `module.groups.group_ids[...]`).
- Produces: module input `var.apps` (map keyed by label; each value has `flow` + optional fields below); outputs `app_ids`, `client_ids`, `client_secrets` (sensitive).

- [ ] **Step 1: Write `variables.tf`**

```hcl
variable "apps" {
  description = "OIDC apps to manage, keyed by app label."
  type = map(object({
    flow                       = string
    redirect_uris              = optional(list(string), [])
    post_logout_redirect_uris  = optional(list(string), [])
    refresh_token              = optional(bool, false)
    status                     = optional(string, "ACTIVE")
    group_assignments          = optional(list(string), [])
    grant_types                = optional(list(string))
    response_types             = optional(list(string))
    token_endpoint_auth_method = optional(string)
    pkce_required              = optional(bool)
  }))
  default = {}

  validation {
    condition = alltrue([
      for a in values(var.apps) :
      contains(["web_auth_code", "spa_pkce", "native_pkce", "service_client_credentials", "implicit"], a.flow)
    ])
    error_message = "Each app's flow must be one of: web_auth_code, spa_pkce, native_pkce, service_client_credentials, implicit."
  }

  validation {
    condition     = alltrue([for a in values(var.apps) : contains(["ACTIVE", "INACTIVE"], a.status)])
    error_message = "Each app's status must be ACTIVE or INACTIVE."
  }
}
```

- [ ] **Step 2: Write `main.tf`** (flow expansion + resources)

```hcl
terraform {
  required_providers {
    okta = {
      source  = "okta/okta"
      version = ">= 6.0.0"
    }
  }
}

locals {
  flow_defaults = {
    web_auth_code              = { type = "web", grant_types = ["authorization_code"], response_types = ["code"], token_endpoint_auth_method = "client_secret_basic", pkce_required = false }
    spa_pkce                   = { type = "browser", grant_types = ["authorization_code"], response_types = ["code"], token_endpoint_auth_method = "none", pkce_required = true }
    native_pkce                = { type = "native", grant_types = ["authorization_code"], response_types = ["code"], token_endpoint_auth_method = "none", pkce_required = true }
    service_client_credentials = { type = "service", grant_types = ["client_credentials"], response_types = [], token_endpoint_auth_method = "client_secret_basic", pkce_required = false }
    implicit                   = { type = "browser", grant_types = ["implicit"], response_types = ["token", "id_token"], token_endpoint_auth_method = "none", pkce_required = false }
  }

  apps_resolved = {
    for k, v in var.apps : k => {
      type = local.flow_defaults[v.flow].type
      grant_types = distinct(concat(
        v.grant_types != null ? v.grant_types : local.flow_defaults[v.flow].grant_types,
        v.refresh_token ? ["refresh_token"] : []
      ))
      response_types             = v.response_types != null ? v.response_types : local.flow_defaults[v.flow].response_types
      token_endpoint_auth_method = v.token_endpoint_auth_method != null ? v.token_endpoint_auth_method : local.flow_defaults[v.flow].token_endpoint_auth_method
      pkce_required              = v.pkce_required != null ? v.pkce_required : local.flow_defaults[v.flow].pkce_required
      redirect_uris              = v.redirect_uris
      post_logout_redirect_uris  = v.post_logout_redirect_uris
      status                     = v.status
      group_assignments          = v.group_assignments
    }
  }
}

resource "okta_app_oauth" "this" {
  for_each = local.apps_resolved

  label                      = each.key
  type                       = each.value.type
  grant_types                = each.value.grant_types
  response_types             = each.value.response_types
  redirect_uris              = each.value.redirect_uris
  post_logout_redirect_uris  = each.value.post_logout_redirect_uris
  token_endpoint_auth_method = each.value.token_endpoint_auth_method
  pkce_required              = each.value.pkce_required
  status                     = each.value.status
}

resource "okta_app_group_assignments" "this" {
  for_each = { for k, v in var.apps : k => v if length(v.group_assignments) > 0 }

  app_id = okta_app_oauth.this[each.key].id

  dynamic "group" {
    for_each = each.value.group_assignments
    content {
      id = group.value
    }
  }
}
```

- [ ] **Step 3: Write `outputs.tf`**

```hcl
output "app_ids" {
  description = "Map of app label => Okta app id."
  value       = { for k, a in okta_app_oauth.this : k => a.id }
}

output "client_ids" {
  description = "Map of app label => OAuth client_id."
  value       = { for k, a in okta_app_oauth.this : k => a.client_id }
}

output "client_secrets" {
  description = "Map of app label => OAuth client_secret (confidential apps)."
  value       = { for k, a in okta_app_oauth.this : k => a.client_secret }
  sensitive   = true
}
```

- [ ] **Step 4: Write `README.md`**

````markdown
# app_oauth module

Manages OIDC application integrations (`okta_app_oauth`) from a data-driven map,
with optional group assignments. Okta acts as the IdP/OP; each app is an OIDC
client. (Onboarding an external entity *as an IdP* is out of scope — that is the
inbound-federation `idp_oidc` module.)

## The `flow` selector
Set `flow` per app; the module expands it to a coherent field combination:

| flow | type | grant_types | response_types | token_endpoint_auth_method | pkce_required |
|---|---|---|---|---|---|
| web_auth_code | web | authorization_code | code | client_secret_basic | false |
| spa_pkce | browser | authorization_code | code | none | true |
| native_pkce | native | authorization_code | code | none | true |
| service_client_credentials | service | client_credentials | — | client_secret_basic | false |
| implicit | browser | implicit | token, id_token | none | false |

`refresh_token = true` appends the `refresh_token` grant. Any of `grant_types`,
`response_types`, `token_endpoint_auth_method`, `pkce_required` can be set to
override the flow default.

## Inputs
- `apps` — map keyed by app label; each value: `flow` (required), `redirect_uris`,
  `post_logout_redirect_uris`, `refresh_token`, `status`, `group_assignments`
  (group IDs), plus the raw overrides above.

## Outputs
- `app_ids`, `client_ids`, `client_secrets` (sensitive).
````

- [ ] **Step 5: Write the example `examples/app_oauth/main.tf`**

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
module "app_oauth" {
  source = "../../modules/app_oauth"
  apps = {
    "Example Web App" = {
      flow              = "web_auth_code"
      redirect_uris     = ["https://example.com/authorization-code/callback"]
      refresh_token     = true
      group_assignments = ["00g00000000000000000"] # placeholder group id (validate-only)
    }
    "Example Service" = {
      flow = "service_client_credentials"
    }
  }
}
```

- [ ] **Step 6: Format, then validate the example offline**

```bash
cd /mnt/c/Git/okta-css/okta-base-config
terraform fmt -recursive
terraform fmt -check -recursive && echo "FMT CLEAN"
cd examples/app_oauth
terraform init -backend=false && terraform validate
```
Expected: `FMT CLEAN`, then `Success! The configuration is valid.`
If validate reports an unknown attribute, fix it against the names in Global Constraints (do not invent attributes).

- [ ] **Step 7: Clean transient init artifacts**

```bash
rm -rf /mnt/c/Git/okta-css/okta-base-config/examples/app_oauth/.terraform /mnt/c/Git/okta-css/okta-base-config/examples/app_oauth/.terraform.lock.hcl
```

- [ ] **Step 8: Commit**

```bash
cd /mnt/c/Git/okta-css/okta-base-config
git add modules/app_oauth examples/app_oauth
git commit -m "feat: add app_oauth module (OIDC, flow selector + group assignments)"
```

---

### Task 2: `app_saml` module + example

**Files:**
- Create: `/mnt/c/Git/okta-css/okta-base-config/modules/app_saml/variables.tf`
- Create: `/mnt/c/Git/okta-css/okta-base-config/modules/app_saml/main.tf`
- Create: `/mnt/c/Git/okta-css/okta-base-config/modules/app_saml/outputs.tf`
- Create: `/mnt/c/Git/okta-css/okta-base-config/modules/app_saml/README.md`
- Create: `/mnt/c/Git/okta-css/okta-base-config/examples/app_saml/main.tf`

**Interfaces:**
- Consumes: the `okta` provider; group IDs from the caller.
- Produces: module input `var.apps` (map keyed by label; `sso_url`+`audience` required); output `app_ids`.

- [ ] **Step 1: Write `variables.tf`**

```hcl
variable "apps" {
  description = "SAML apps (SP onboarding; Okta as IdP), keyed by app label."
  type = map(object({
    sso_url                  = string
    audience                 = string
    recipient                = optional(string)
    destination              = optional(string)
    subject_name_id_template = optional(string, "$${user.userName}")
    subject_name_id_format   = optional(string, "urn:oasis:names:tc:SAML:1.1:nameid-format:unspecified")
    response_signed          = optional(bool, true)
    assertion_signed         = optional(bool, true)
    signature_algorithm      = optional(string, "RSA_SHA256")
    digest_algorithm         = optional(string, "SHA256")
    honor_force_authn        = optional(bool, true)
    authn_context_class_ref  = optional(string, "urn:oasis:names:tc:SAML:2.0:ac:classes:PasswordProtectedTransport")
    status                   = optional(string, "ACTIVE")
    attribute_statements = optional(list(object({
      name      = string
      namespace = optional(string, "urn:oasis:names:tc:SAML:2.0:attrname-format:unspecified")
      values    = list(string)
    })), [])
    group_assignments = optional(list(string), [])
  }))
  default = {}

  validation {
    condition     = alltrue([for a in values(var.apps) : contains(["ACTIVE", "INACTIVE"], a.status)])
    error_message = "Each app's status must be ACTIVE or INACTIVE."
  }
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

resource "okta_app_saml" "this" {
  for_each = var.apps

  label                    = each.key
  sso_url                  = each.value.sso_url
  recipient                = coalesce(each.value.recipient, each.value.sso_url)
  destination              = coalesce(each.value.destination, each.value.sso_url)
  audience                 = each.value.audience
  subject_name_id_template = each.value.subject_name_id_template
  subject_name_id_format   = each.value.subject_name_id_format
  response_signed          = each.value.response_signed
  assertion_signed         = each.value.assertion_signed
  signature_algorithm      = each.value.signature_algorithm
  digest_algorithm         = each.value.digest_algorithm
  honor_force_authn        = each.value.honor_force_authn
  authn_context_class_ref  = each.value.authn_context_class_ref
  status                   = each.value.status

  dynamic "attribute_statements" {
    for_each = each.value.attribute_statements
    content {
      name      = attribute_statements.value.name
      namespace = attribute_statements.value.namespace
      type      = "EXPRESSION"
      values    = attribute_statements.value.values
    }
  }
}

resource "okta_app_group_assignments" "this" {
  for_each = { for k, v in var.apps : k => v if length(v.group_assignments) > 0 }

  app_id = okta_app_saml.this[each.key].id

  dynamic "group" {
    for_each = each.value.group_assignments
    content {
      id = group.value
    }
  }
}
```

- [ ] **Step 3: Write `outputs.tf`**

```hcl
output "app_ids" {
  description = "Map of app label => Okta app id."
  value       = { for k, a in okta_app_saml.this : k => a.id }
}
```

- [ ] **Step 4: Write `README.md`**

````markdown
# app_saml module

Manages SAML 2.0 application integrations (`okta_app_saml`) from a data-driven
map, with optional group assignments. **SP onboarding only:** Okta acts as the
**IdP**; each app is the **Service Provider**. Onboarding an external entity *as
an IdP* (inbound federation) is out of scope — that is the `idp_saml` module.

## Inputs
- `apps` — map keyed by app label. Required per app: `sso_url` (SP ACS URL),
  `audience` (SP entity ID). Optional: `recipient`/`destination` (default to
  `sso_url`), `subject_name_id_template`/`_format`, `response_signed`,
  `assertion_signed`, `signature_algorithm`, `digest_algorithm`,
  `honor_force_authn`, `authn_context_class_ref`, `status`,
  `attribute_statements` (list of `{ name, namespace?, values }`),
  `group_assignments` (group IDs).

## Outputs
- `app_ids` — map of app label => Okta app id.
````

- [ ] **Step 5: Write the example `examples/app_saml/main.tf`**

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
module "app_saml" {
  source = "../../modules/app_saml"
  apps = {
    "Example SAML App" = {
      sso_url  = "https://example.com/sso/saml"
      audience = "https://example.com/saml/metadata"
      attribute_statements = [
        { name = "email", values = ["user.email"] }
      ]
      group_assignments = ["00g00000000000000000"] # placeholder group id (validate-only)
    }
  }
}
```

- [ ] **Step 6: Format, then validate the example offline**

```bash
cd /mnt/c/Git/okta-css/okta-base-config
terraform fmt -recursive
terraform fmt -check -recursive && echo "FMT CLEAN"
cd examples/app_saml
terraform init -backend=false && terraform validate
```
Expected: `FMT CLEAN`, then `Success! The configuration is valid.`

- [ ] **Step 7: Clean transient init artifacts**

```bash
rm -rf /mnt/c/Git/okta-css/okta-base-config/examples/app_saml/.terraform /mnt/c/Git/okta-css/okta-base-config/examples/app_saml/.terraform.lock.hcl
```

- [ ] **Step 8: Commit**

```bash
cd /mnt/c/Git/okta-css/okta-base-config
git add modules/app_saml examples/app_saml
git commit -m "feat: add app_saml module (SAML SP onboarding + group assignments)"
```

---

### Task 3: Developer guide — "Onboarding apps" section

**Files:**
- Modify: `/mnt/c/Git/okta-css/okta-base-config/docs/DEVELOPER_GUIDE.md` (append a subsection at the end of §5 "Making configuration changes", before the `---` that precedes §6)

**Interfaces:**
- Consumes: the two modules from Tasks 1–2 and the existing `groups` module output `group_ids`.
- Produces: developer-facing docs. No code depends on it.

- [ ] **Step 1: Locate the insertion point**

Run: `grep -n "## 6. Landing a change" /mnt/c/Git/okta-css/okta-base-config/docs/DEVELOPER_GUIDE.md`
Insert the new subsection just **above** the `---` line that precedes that heading (i.e. at the end of §5).

- [ ] **Step 2: Insert this subsection**

````markdown
### Onboarding apps

Two modules manage application integrations (Okta as the IdP/OP):

- **`app_oauth`** — OIDC apps. Pick a `flow` and the module sets the right
  OAuth fields:

  | flow | use it for |
  |---|---|
  | `web_auth_code` | server-side web apps (confidential) |
  | `spa_pkce` | single-page apps (browser, public + PKCE) |
  | `native_pkce` | mobile/desktop apps (public + PKCE) |
  | `service_client_credentials` | machine-to-machine, no user |
  | `implicit` | legacy only — avoid for new apps |

- **`app_saml`** — SAML 2.0 apps (SP onboarding; you supply the SP's `sso_url`
  and `audience`).

> Onboarding an external entity *as an IdP* (inbound "sign in with…" federation)
> is a different concern handled by the federation modules, not these.

Both take a map keyed by the app label and an optional `group_assignments` list
of **group IDs** — wire the `groups` module's output straight in:

```hcl
module "groups" {
  source = ".../modules/groups"
  groups = { "engineering" = { description = "Engineering" } }
}

module "app_oauth" {
  source = ".../modules/app_oauth"
  apps = {
    "My Web App" = {
      flow              = "web_auth_code"
      redirect_uris     = ["https://app.example.com/cb"]
      refresh_token     = true
      group_assignments = [module.groups.group_ids["engineering"]]
    }
  }
}

module "app_saml" {
  source = ".../modules/app_saml"
  apps = {
    "My SAML App" = {
      sso_url              = "https://app.example.com/acs"
      audience             = "https://app.example.com/metadata"
      attribute_statements = [{ name = "email", values = ["user.email"] }]
      group_assignments    = [module.groups.group_ids["engineering"]]
    }
  }
}
```

As always: develop and test in `okta-dev-local` against your own org first
(§4), then land the change in `okta-base-config` (§6) and promote it (§7).
````

- [ ] **Step 3: Commit**

```bash
cd /mnt/c/Git/okta-css/okta-base-config
git add docs/DEVELOPER_GUIDE.md
git commit -m "docs: add Onboarding apps section to developer guide"
```

---

### Task 4: Release (outward-facing — operator runs)

**Files:** none (git remote + tag).

**Interfaces:**
- Consumes: the committed modules/examples/guide (Tasks 1–3).
- Produces: a pushed + tagged `okta-base-config` release that env repos can pin.

> Requires GitHub push access — runs from the operator's terminal, not the build sandbox.

- [ ] **Step 1: Push and confirm CI is green**

```bash
cd /mnt/c/Git/okta-css/okta-base-config
git push
gh run list --repo jandors/okta-base-config --limit 1   # or check the Actions tab; ci must pass (fmt + validate of both new examples)
```

- [ ] **Step 2: Tag the release**

```bash
git tag v1.1.0
git push origin v1.1.0
```
Add a `CHANGELOG.md` line noting the new `app_oauth` + `app_saml` modules.

- [ ] **Step 3: Promote (when the shared stages are wired)**

Bump the `?ref=` in the env repos to `v1.1.0` per the developer guide §7
(DEV → TEST → PROD). **Note:** promotion requires the stage pipeline to be wired
(Azure backend, service apps, environments) — if that isn't done yet, the
release is tagged and ready, and promotion waits on the cloud-wiring effort.

---

## Self-review notes

**Spec coverage:** two modules `app_oauth`+`app_saml` (Tasks 1–2) ✓; groups-pattern shape + `examples/` + provider pin (Tasks 1–2) ✓; cohesive app+group-assignments, caller passes IDs (both modules' `okta_app_group_assignments`) ✓; curated subset + defaults (both `variables.tf`) ✓; OIDC `flow` selector with expansion + raw overrides + `refresh_token`, `type` derived from flow (Task 1 locals) ✓; flow/status validations (Task 1 Step 1) ✓; SAML SP-direction note + `recipient`/`destination` coalesce + `attribute_statements` dynamic block (Task 2) ✓; outputs incl. sensitive `client_secrets`, SAML `app_ids` only (cert output omitted as the spec permitted) ✓; offline CI validate via examples (Tasks 1–2 Step 6) ✓; developer-guide "Onboarding apps" deliverable (Task 3) ✓; release/promote like any `okta-base-config` change (Task 4) ✓.

**Placeholder scan:** no TBD/TODO; all HCL is complete; `00g00000000000000000` is an explicit validate-only placeholder group id (validate does not check existence); attribute names are the verified provider names, not invented.

**Type consistency:** `var.apps` object fields are referenced identically in `locals`/resources/outputs within each module. `okta_app_group_assignments` uses `app_id` + `group { id }` in both modules, referencing `okta_app_<type>.this[each.key].id`. The `flow` enum in the validation matches the `flow_defaults` keys exactly (`web_auth_code`, `spa_pkce`, `native_pkce`, `service_client_credentials`, `implicit`). `group_ids` (consumed in the guide example) is the real output name of the existing `groups` module.

**Note for implementer:** `terraform validate` checks schema only — it will catch a wrong attribute name but not Okta business rules (e.g. a `web` app needing a redirect URI). Those surface at real apply time in `okta-dev-local`, which is the spec's verification path.
