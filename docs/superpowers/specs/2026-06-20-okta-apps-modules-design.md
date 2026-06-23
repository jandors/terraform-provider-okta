# okta-base-config Apps Modules (OIDC + SAML) — Design

**Date:** 2026-06-20
**Status:** Approved (design phase)
**Related:** [2026-06-15-okta-multi-stage-management-design.md](2026-06-15-okta-multi-stage-management-design.md), [2026-06-18-okta-dev-local-environment-design.md](2026-06-18-okta-dev-local-environment-design.md)

## Goal

Add two reusable modules to `okta-base-config` — **`app_oauth`** (OIDC) and
**`app_saml`** (SAML 2.0) — following the established `groups` module pattern, so
teams can manage Okta application integrations as data-driven Terraform config.
Each app can also declare the groups assigned to it. Update the developer guide
so developers can onboard apps.

## Decisions (locked)

| Area | Decision |
|---|---|
| App types | Two modules this effort: `app_oauth` (OIDC) + `app_saml` (SAML). Other types are future siblings. |
| Module shape | Mirror `groups`: a data-driven `var.apps` map keyed by app label, `for_each`, `variables.tf`/`main.tf`/`outputs.tf`/`README.md` + an `examples/` entry. |
| Boundary | App **and** its group assignments in one module (cohesive). Caller passes **group IDs**. |
| Attribute exposure | Curated subset + sensible defaults; add attributes later as needed (no escape hatch, no thin passthrough). |
| OIDC flow config | A high-level **`flow` selector** expands into the correct `type`/`grant_types`/`response_types`/`token_endpoint_auth_method`/`pkce_required` combination; raw fields remain optional overrides. `type` is derived from `flow`. |
| Docs | Update `okta-base-config/docs/DEVELOPER_GUIDE.md` with an "Onboarding apps" subsection (deliverable). |

## Module structure

```
okta-base-config/
├── modules/
│   ├── app_oauth/   # okta_app_oauth + okta_app_group_assignments
│   │   ├── main.tf  variables.tf  outputs.tf  README.md
│   └── app_saml/    # okta_app_saml  + okta_app_group_assignments
│       ├── main.tf  variables.tf  outputs.tf  README.md
└── examples/
    ├── app_oauth/main.tf
    └── app_saml/main.tf
```

Both modules consume the same `okta` provider already used elsewhere; no new
provider wiring. Each takes `var.apps` (a map keyed by the app **label**) and
exposes outputs other config can consume.

## `app_oauth` module

**Direction:** this module onboards OIDC **clients / relying parties** — Okta acts
as the **OP/IdP** issuing tokens to the app. Onboarding an external OIDC entity
*as an IdP* (inbound federation, Okta-as-RP) is a different resource
(`okta_idp_oidc`/`okta_idp_social`) and is out of scope — see Out of scope.

### Input — `var.apps`
```hcl
map(object({
  flow                       = string                       # REQUIRED selector (see table)
  redirect_uris              = optional(list(string), [])
  post_logout_redirect_uris  = optional(list(string), [])
  refresh_token              = optional(bool, false)        # appends "refresh_token" grant
  status                     = optional(string, "ACTIVE")   # ACTIVE | INACTIVE
  group_assignments          = optional(list(string), [])   # group IDs
  # raw overrides — null = use the flow's default:
  grant_types                = optional(list(string))
  response_types             = optional(list(string))
  token_endpoint_auth_method = optional(string)
  pkce_required              = optional(bool)
}))
```

### The `flow` selector
A `locals` map expands `flow` into a coherent field combination:

| `flow` | type | grant_types | response_types | token_endpoint_auth_method | pkce_required |
|---|---|---|---|---|---|
| `web_auth_code` | `web` | `["authorization_code"]` | `["code"]` | `client_secret_basic` | `false` |
| `spa_pkce` | `browser` | `["authorization_code"]` | `["code"]` | `none` | `true` |
| `native_pkce` | `native` | `["authorization_code"]` | `["code"]` | `none` | `true` |
| `service_client_credentials` | `service` | `["client_credentials"]` | `[]` | `client_secret_basic` | `false` |
| `implicit` | `browser` | `["implicit"]` | `["token","id_token"]` | `none` | `false` |

### Resolution logic (`locals`)
For each app: start from `flow_defaults[flow]`, replace any field whose raw
override is non-null, then append `refresh_token` to the grants when
`refresh_token = true`:
```hcl
grant_types = distinct(concat(
  coalesce(v.grant_types, local.flow_defaults[v.flow].grant_types),
  v.refresh_token ? ["refresh_token"] : []
))
# type, response_types, token_endpoint_auth_method, pkce_required resolved the same way
# (override via coalesce, else the flow default); type comes ONLY from the flow.
```
`okta_app_oauth.this` is created `for_each` over the resolved map.

### Validation
- `flow` ∈ {`web_auth_code`, `spa_pkce`, `native_pkce`, `service_client_credentials`, `implicit`}.
- `status` ∈ {`ACTIVE`, `INACTIVE`}.

### Notes
- `type` is **derived from `flow`**, never a direct input — structurally
  prevents type/flow mismatch.
- Client **secrets** are never inputs. `client_secret` is exposed only as a
  `sensitive` output (see Outputs); secrets do not belong in tfvars.

## `app_saml` module

**Direction:** this module covers **SP onboarding** — Okta acts as the **IdP** and
the external entity is the **Service Provider** (the SaaS app Okta logs users
into). Inputs describe the SP: its ACS URL (`sso_url`) and entity ID
(`audience`). Onboarding an external entity *as an IdP* (inbound federation,
Okta-as-SP) is a **different resource** (`okta_idp_saml`) and is explicitly out
of scope here — see Out of scope.

### Input — `var.apps`
```hcl
map(object({
  sso_url                  = string                                     # required (ACS URL)
  audience                 = string                                     # required (SP entity ID)
  recipient                = optional(string)                           # defaults to sso_url when null
  destination              = optional(string)                           # defaults to sso_url when null
  subject_name_id_template = optional(string, "$${user.userName}")
  subject_name_id_format   = optional(string, "urn:oasis:names:tc:SAML:1.1:nameid-format:unspecified")
  response_signed          = optional(bool, true)
  assertion_signed         = optional(bool, true)
  signature_algorithm      = optional(string, "RSA_SHA256")
  digest_algorithm         = optional(string, "SHA256")
  honor_force_authn        = optional(bool, true)
  authn_context_class_ref  = optional(string, "urn:oasis:names:tc:SAML:2.0:ac:classes:PasswordProtectedTransport")
  status                   = optional(string, "ACTIVE")
  attribute_statements     = optional(list(object({
    name      = string
    namespace = optional(string, "urn:oasis:names:tc:SAML:2.0:attrname-format:unspecified")
    values    = list(string)
  })), [])
  group_assignments        = optional(list(string), [])                 # group IDs
}))
```

- `recipient`/`destination` resolve to `sso_url` via `coalesce` when null.
- `attribute_statements` render as `dynamic "attribute_statements"` blocks.
- Validation: `status` ∈ {`ACTIVE`, `INACTIVE`}.

## Group assignments (both modules)

Each app with a non-empty `group_assignments` gets one
`okta_app_group_assignments` resource:
```hcl
resource "okta_app_group_assignments" "this" {
  for_each = { for k, v in var.apps : k => v if length(v.group_assignments) > 0 }
  app_id   = okta_app_<type>.this[each.key].id
  dynamic "group" {
    for_each = each.value.group_assignments
    content { id = group.value }
  }
}
```
The caller supplies group IDs, typically wiring the `groups` module output:
```hcl
group_assignments = [module.groups.group_ids["tf-managed-engineering"]]
```
This keeps the app modules decoupled from how groups are managed.

## Outputs

| Module | Output | Value |
|---|---|---|
| `app_oauth` | `app_ids` | map: label → app id |
| `app_oauth` | `client_ids` | map: label → `client_id` |
| `app_oauth` | `client_secrets` | map: label → `client_secret`, marked **`sensitive`** |
| `app_saml` | `app_ids` | map: label → app id |
| `app_saml` | `certificate` *(optional)* | map: label → signing cert, **only if** `okta_app_saml` exposes it directly; confirm against the provider schema at implementation, otherwise omit (SAML metadata is otherwise retrieved via the `okta_app_metadata_saml` data source, out of scope here) |

## Examples & testing

- **Examples** (`examples/app_oauth/`, `examples/app_saml/`) call each module
  with a representative entry — a `web_auth_code` app with a redirect URI and a
  group assignment; a SAML app with `sso_url`/`audience` and one attribute
  statement. These are the CI validation targets.
- **CI (offline):** existing `okta-base-config` `ci.yml` runs `terraform
  fmt -check` + `init -backend=false` + `validate` over `examples/*` — picks up
  the new examples automatically. Flow/status validations are exercised here.
- **Real verification:** developers create the apps in `okta-dev-local` against
  their own throwaway org (`plan`/`apply`/`destroy`). No live acceptance tests.

## Developer guide update (deliverable)

Add an **"Onboarding apps"** subsection under §5 *Making configuration changes*
of `okta-base-config/docs/DEVELOPER_GUIDE.md`, covering:
- the two modules and when to use each (OIDC vs SAML);
- the **`flow` selector table** with a one-line "pick your flow" guide;
- the data-driven `apps` map shape with a worked OIDC **and** SAML example;
- **assigning apps to groups** by wiring `module.groups.group_ids[...]` into
  `group_assignments`;
- the reminder to test in `okta-dev-local` first, then land in `okta-base-config`
  and promote (links to §6/§7).

## Release / promotion

These modules ship like any other `okta-base-config` change: PR → CI validate →
merge → tag a new release (e.g. `v1.1.0`) → promote the ref bump through
DEV → TEST → PROD per the existing pipeline. No pipeline changes required.

## Out of scope

- App types other than OIDC and SAML (bookmark, SWA, auto-login, etc.) — future
  sibling modules.
- **Inbound federation (SAML *and* OIDC)** — onboarding an external entity *as an
  IdP* that Okta delegates to: `okta_idp_saml` (+ `okta_idp_saml_key`) and
  `okta_idp_oidc`/`okta_idp_social`, with Okta as the SP/RP. A separate resource
  family with distinct concerns (issuer/cert or RP client creds, subject
  matching, JIT provisioning, account linking) — its own federation
  design/module. This spec covers only the outbound/client direction
  (`app_saml` = Okta-as-IdP, `app_oauth` = Okta-as-OP).
- App sign-on policies (`okta_app_signon_policy*`), OAuth API scopes/grants,
  per-app schema properties — separate modules later.
- Individual **user** assignments (`okta_app_user`) — structural config only;
  group assignments only.
- Rare provider attributes not in the curated sets — added as new optional
  attributes when a real app needs them.
