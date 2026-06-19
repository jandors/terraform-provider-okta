# Okta IaC — Backlog

Future work for the Okta-Terraform program (the multi-stage pipeline,
`okta-dev-local`, and `okta-org-export`). Current committed scope is tracked in
the specs/plans under `docs/superpowers/`; this file is the parking lot for
everything we intend to do **later**.

Resource-type names below are taken from the Okta provider's own registrations
(`okta/resources/resources.go`) — confirm the exact name and import support
against the provider version in use when a type is actually implemented.

## How this backlog works

- **Now** — in active/committed scope (has a spec + plan).
- **Next** — the immediately following increments.
- **Later** — desirable, not yet scheduled.

---

## Feature / epic backlog

### Now (committed)
- Multi-stage pipeline: `okta-base-config` + `okta-dev`/`test`/`prod` (built; awaiting cloud wiring).
- `okta-dev-local` developer sandbox (built; pending `BASE_CONFIG_READ_TOKEN`).
- `okta-org-export` brownfield adoption process (spec approved; plan next).
- `groups` module (built).

### Next
- **Cloud wiring for the stages** — Azure backend (Phase A), Okta service apps per org (Phase B), GitHub federated identities + environments/secrets (D2/D3, E2/E3), token substitution, **prod required-reviewers approval gate**. (See `2026-06-15-...-management.md`.)
- **`BASE_CONFIG_READ_TOKEN`** repo secret so `okta-dev-local` CI can read the private base-config sibling.
- **`okta-dev-local` OAuth2 docs** — document the service-app/`OKTA_API_*` auth path alongside the SSWS default (we already used OAuth2 against the trial org).
- **More `okta-base-config` modules** — build out by domain (apps, policies, auth servers, branding) following the `groups` pattern; each gets an example + CI validation.
- **`discover.py` discriminators** — apps `type_map` (one `/apps` list → many TF types) and per-`type` policy queries.
- **Live drift detection** — confirm the nightly `drift.yml` works end-to-end once stages are wired.
- **Promotion dry-run** — exercise the `v1.1.0` ref-bump flow DEV→TEST→PROD (plan Task F1).

### Later
- **Whole-org export coverage** — grow `okta-org-export`'s `resource-types.json` toward the full matrix below.
- **PII handling for exports** — redaction/exclusion policy for `okta_user` HCL + `manifest.json` before any per-org snapshot is committed.
- **Okta Governance** — separate epic; different API + scopes (campaigns, entitlements, reviews, access requests). Decide if it joins this program.
- **Quality gates** — pre-commit `terraform fmt`, `tflint`, CHANGELOG discipline in `okta-base-config`.
- **State migration** — if any adopted org graduates from local snapshot to a managed stage, move it onto the Azure backend + promotion pipeline.

---

## Resource-type coverage matrix

Types we intend to manage (modules in `okta-base-config`) and adopt
(`okta-org-export` registry). Legend: ✅ done · 🟡 starter registry (first
`okta-org-export` build) · ⬜ backlog.

### Groups & membership
- ✅ `okta_group`
- ⬜ `okta_group_rule`
- ⬜ `okta_group_role`
- ⬜ `okta_group_owner`
- ⬜ `okta_group_memberships`
- ⬜ `okta_group_schema_property`

### Users, schema & profile
- 🟡 `okta_user`
- ⬜ `okta_user_type`
- ⬜ `okta_user_schema_property`
- ⬜ `okta_user_base_schema_property`
- ⬜ `okta_user_admin_roles`
- ⬜ `okta_user_group_memberships`
- ⬜ `okta_user_factor_question`
- ⬜ `okta_user_security_questions`
- ⬜ `okta_profile_mapping`
- ⬜ `okta_link_definition`
- ⬜ `okta_link_value`
- ⬜ `okta_ui_schema`

### Applications
- ⬜ `okta_app_oauth`
- ⬜ `okta_app_saml`
- ⬜ `okta_app_swa`
- ⬜ `okta_app_auto_login`
- ⬜ `okta_app_basic_auth`
- ⬜ `okta_app_bookmark`
- ⬜ `okta_app_secure_password_store`
- ⬜ `okta_app_shared_credentials`
- ⬜ `okta_app_three_field`
- ⬜ `okta_app_group_assignment` / `okta_app_group_assignments`
- ⬜ `okta_app_user`
- ⬜ `okta_app_user_schema_property` / `okta_app_user_base_schema_property`
- ⬜ `okta_app_oauth_api_scope`
- ⬜ `okta_app_oauth_redirect_uri` / `okta_app_oauth_post_logout_redirect_uri`
- ⬜ `okta_app_oauth_role_assignment`
- ⬜ `okta_app_saml_app_settings`
- ⬜ `okta_app_signon_policy` / `okta_app_signon_policy_rule`
- ⬜ `okta_app_federated_claim`
- ⬜ `okta_app_features`

### Authentication & policies
- ⬜ `okta_policy_signon` / `okta_policy_rule_signon`
- ⬜ `okta_policy_password` / `okta_policy_password_default` / `okta_policy_rule_password`
- ⬜ `okta_policy_mfa` / `okta_policy_mfa_default` / `okta_policy_rule_mfa`
- ⬜ `okta_policy_profile_enrollment` / `okta_policy_rule_profile_enrollment` / `okta_policy_profile_enrollment_apps`
- ⬜ `okta_policy_rule_idp_discovery`
- ⬜ `okta_authenticator`
- ⬜ `okta_authenticator_method_webauthn` / `okta_authenticator_webauthn_custom_aaguid`
- ⬜ `okta_factor` / `okta_factor_totp`
- ⬜ `okta_template_sms`
- ⬜ `okta_security_notification_emails`
- ⬜ device assurance (`okta_policy_device_assurance_{android,ios,macos,windows,chromeos}` — framework resources; confirm names)

### Authorization servers (API Access Management)
- ⬜ `okta_auth_server` / `okta_auth_server_default`
- ⬜ `okta_auth_server_policy` / `okta_auth_server_policy_rule`
- ⬜ `okta_auth_server_scope`
- ⬜ `okta_auth_server_claim` / `okta_auth_server_claim_default`
- ⬜ `okta_trusted_origin`

### Identity providers & federation
- ⬜ `okta_idp_oidc` / `okta_idp_saml` / `okta_idp_social`
- ⬜ `okta_idp_saml_key`
- ⬜ `okta_security_events_provider`
- ⬜ `okta_identity_source_group_membership` (+ related identity-source sync)

### Network, security & risk
- 🟡 `okta_network_zone`
- ⬜ `okta_threat_insight_settings`
- ⬜ `okta_behavior`
- ⬜ `okta_captcha` / `okta_captcha_org_wide_settings`
- ⬜ `okta_rate_limiting` / `okta_rate_limit_admin_notification_settings` / `okta_rate_limit_warning_threshold_percentage` / `okta_principal_rate_limits`
- ⬜ `okta_entity_risk_policy` / `okta_entity_risk_policy_rule`
- ⬜ `okta_session_violation_policy` / `okta_session_violation_policy_rule`
- ⬜ `okta_post_auth_session_policy` / `okta_post_auth_session_policy_rule`

### Admin roles & access control
- ⬜ `okta_admin_role_custom` / `okta_admin_role_custom_assignments` / `okta_admin_role_targets`
- ⬜ `okta_resource_set` / `okta_iam_resource_set` / `okta_iam_assignees_user`
- ⬜ `okta_role_subscription`

### Branding & customization
- ⬜ `okta_brand`
- ⬜ `okta_theme`
- ⬜ `okta_customized_signin_page`
- ⬜ `okta_email_customization` / `okta_email_template`
- ⬜ `okta_email_sender` / `okta_email_smtp_server`
- ⬜ `okta_email_domain`
- ⬜ `okta_domain` / `okta_domain_certificate` / `okta_domain_verification`

### Org, hooks & integrations
- ⬜ `okta_org_configuration` / `okta_org_support`
- ⬜ `okta_event_hook` / `okta_inline_hook` / `okta_hook_key`
- ⬜ `okta_log_stream`
- ⬜ `okta_api_service_integration`
- ⬜ `okta_push_provider` / `okta_push_group`
- ⬜ `okta_agent_pool_update`
- ⬜ `okta_feature`
- ⬜ `okta_device`
- ⬜ `okta_realm` / `okta_realm_assignment`

### Okta Governance (separate API — undecided whether in program)
- ⬜ `okta_campaign` / `okta_review`
- ⬜ `okta_entitlement` / `okta_entitlement_bundle` / `okta_principal_entitlements`
- ⬜ `okta_request_v2` / `okta_request_condition` / `okta_request_sequence`
- ⬜ `okta_request_setting_organization` / `okta_request_setting_resource`
- ⬜ `okta_catalog_entry_default` / `okta_catalog_entry_user_access_request_fields`

---

## Explicitly out of scope (for now)

- Managing Okta **system/default** resources (`okta_everyone_group`,
  `okta_default_policy`, `okta_auth_server_default`, built-in apps) — these are
  curated *out* by `okta-org-export` and not modeled as managed config.
- Building a custom/forked Okta provider binary.
- A third-party exporter (terraformer) — we use native import + generate-config.
