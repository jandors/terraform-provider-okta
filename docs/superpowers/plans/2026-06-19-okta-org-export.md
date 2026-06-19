# okta-org-export Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the `okta-org-export` repo — a reusable process that reverse-engineers a live Okta org into Terraform-managed config (HCL + state) without deleting anything.

**Architecture:** A stdlib-only Python script (`discover.py`) walks the Okta API per a JSON resource-type registry, curates out system defaults via a JSON skip-list, and emits native Terraform `import` blocks. `terraform plan -generate-config-out` then generates the HCL; import brings resources into local state; a "0 to destroy" plan gates any apply. A runbook ties the stages together.

**Tech Stack:** Python 3 (stdlib only — `urllib`, `json`, `re`, `unittest`), Terraform (`okta/okta ~> 6.12`, local backend, `-generate-config-out`).

**Spec:** `docs/superpowers/specs/2026-06-19-okta-org-export-adoption-design.md`

## Global Constraints

- Repo location (local): `/mnt/c/Git/okta-css/okta-org-export` (sibling of `okta-base-config`). GitHub: `jandors/okta-org-export`.
- `discover.py` is **stdlib only** — no `pip install`, no PyYAML. Registry/skip files are **JSON**.
- Tests use stdlib **`unittest`**, run via `python3 -m unittest -v`.
- Discovery auth: read-only **SSWS** token in `OKTA_EXPORT_TOKEN`; target via `OKTA_ORG_NAME` + `OKTA_BASE_URL`.
- Terraform: `>= 1.6` (needs `-generate-config-out`, TF ≥ 1.5); local backend only; provider `okta/okta ~> 6.12`.
- Per-org output lives under `workspaces/<org>/` and is **gitignored** (PII in `okta_user` HCL).
- Safety: generate-before-import, add-only config, **Stage-5 `plan` must show `0 to destroy`** before any apply.
- Resource-name identifiers must be valid HCL: lowercase, non-alphanumerics → `_`, no leading digit, unique within a type.

## Scope note (starter registry)

Per the [backlog](../BACKLOG.md), the first build ships the **generic** registry
driver plus three concrete types — `okta_group`, `okta_user`,
`okta_network_zone`. The apps/policies **discriminators** (one `/apps` list → many
TF types; per-`type` policy queries) need extra `type_map` logic and are a
**Next** backlog item, not part of this plan. Adding more plain types later is
just new entries in `resource-types.json`.

---

### Task 1: Scaffold the repo (structure, registry, skip-list, templates)

**Files:**
- Create: `/mnt/c/Git/okta-css/okta-org-export/.gitignore`
- Create: `/mnt/c/Git/okta-css/okta-org-export/README.md`
- Create: `/mnt/c/Git/okta-css/okta-org-export/resource-types.json`
- Create: `/mnt/c/Git/okta-css/okta-org-export/skip-list.json`
- Create: `/mnt/c/Git/okta-css/okta-org-export/templates/provider.tf`
- Create: `/mnt/c/Git/okta-css/okta-org-export/templates/versions.tf`
- Create: `/mnt/c/Git/okta-css/okta-org-export/workspaces/.gitkeep`

**Interfaces:**
- Produces: `resource-types.json` (a JSON array of `{tf_type, api, id_field, name_field}`) and `skip-list.json` (`{auto_skip_when: [...], <tf_type>: {names: [...]}}`) — both consumed by `discover.py` in Task 3.

- [ ] **Step 1: Create directories**

```bash
mkdir -p /mnt/c/Git/okta-css/okta-org-export/templates /mnt/c/Git/okta-css/okta-org-export/workspaces
cd /mnt/c/Git/okta-css/okta-org-export
```

- [ ] **Step 2: Write `.gitignore`**

```gitignore
# per-org output (generated HCL/state, may contain PII)
workspaces/*/
!workspaces/.gitkeep
.terraform/
*.tfstate
*.tfstate.*
*.pem
.env
__pycache__/
```

- [ ] **Step 3: Write `workspaces/.gitkeep`** (so the dir is tracked despite the ignore)

```
# keeps workspaces/ in git; per-org subdirs are gitignored
```

- [ ] **Step 4: Write `resource-types.json`**

```json
[
  { "tf_type": "okta_group",        "api": "/api/v1/groups", "id_field": "id", "name_field": "profile.name" },
  { "tf_type": "okta_user",         "api": "/api/v1/users",  "id_field": "id", "name_field": "profile.login" },
  { "tf_type": "okta_network_zone", "api": "/api/v1/zones",  "id_field": "id", "name_field": "name" }
]
```

- [ ] **Step 5: Write `skip-list.json`**

```json
{
  "auto_skip_when": [ { "system": true }, { "default": true } ],
  "okta_group":       { "names": ["Everyone"] },
  "okta_app":         { "names": ["Okta Admin Console", "Okta Dashboard", "Okta Browser Plugin"] },
  "okta_auth_server": { "names": ["default"] }
}
```

- [ ] **Step 6: Write `templates/provider.tf`** (tokens replaced per workspace by the runbook)

```hcl
provider "okta" {
  org_name = "ORG_NAME"
  base_url = "BASE_URL"
}
```

- [ ] **Step 7: Write `templates/versions.tf`**

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

- [ ] **Step 8: Write `README.md`**

````markdown
# okta-org-export

A reusable process to reverse-engineer an existing Okta org into
Terraform-managed config (HCL + state) **without deleting anything**, then layer
test config on top.

## How it works (7 stages)

1. **Discover** — `discover.py` lists every resource ID per type in
   `resource-types.json`, skipping anything in `skip-list.json`, → `imports.tf`.
2. **Generate** — `terraform plan -generate-config-out=generated.tf`.
3. **Curate** — fix generated HCL; confirm system defaults were skipped.
4. **Import** — `terraform apply` (import blocks pull resources into state).
5. **Verify** — `terraform plan` must show `0 to add, 0 to change, 0 to destroy`.
6. **Add test config** — `test-config.tf`; plan shows only `+1`.
7. **Apply** — existing config untouched, test config created.

See `runbook.md` for the full step-by-step.

## Safety

Config is generated from what exists before import; the process only ever *adds*
config; Stage 5's `0 to destroy` is a hard gate. System/default resources are
curated out via `skip-list.json`.

> Per-org output under `workspaces/<org>/` is gitignored — generated `okta_user`
> HCL contains PII.
````

- [ ] **Step 9: Validate the JSON files parse**

```bash
cd /mnt/c/Git/okta-css/okta-org-export
python3 -m json.tool resource-types.json >/dev/null && echo "resource-types.json OK"
python3 -m json.tool skip-list.json >/dev/null && echo "skip-list.json OK"
```
Expected: both print `... OK`.

- [ ] **Step 10: Initialize git and commit**

```bash
git init -b main
git config user.name "Joseph Andor"
git config user.email "joseph.andor@ic-consult.ch"
git add .gitignore README.md resource-types.json skip-list.json templates workspaces/.gitkeep
git commit -m "chore: scaffold okta-org-export repo"
```

---

### Task 2: `discover.py` pure helpers (TDD)

**Files:**
- Create: `/mnt/c/Git/okta-css/okta-org-export/discover.py`
- Create: `/mnt/c/Git/okta-css/okta-org-export/test_discover.py`

**Interfaces:**
- Produces (consumed by Task 3 and the tests):
  - `get_field(obj: dict, dotted: str) -> object | None` — dotted-path lookup.
  - `sanitize_identifier(raw: str, used: set) -> str` — unique valid HCL id; mutates `used`.
  - `import_block(tf_type: str, name: str, okta_id: str) -> str` — one `import {}` block.
  - `parse_next_link(link_header: str | None) -> str | None` — RFC-5988 `rel="next"` URL.
  - `matches_skip(item: dict, tf_type: str, name, skip: dict) -> tuple[bool, str | None]`.

- [ ] **Step 1: Write the failing tests**

Create `test_discover.py`:
```python
import unittest
import discover as d


class TestHelpers(unittest.TestCase):
    def test_get_field_nested(self):
        self.assertEqual(d.get_field({"profile": {"name": "Eng"}}, "profile.name"), "Eng")

    def test_get_field_missing(self):
        self.assertIsNone(d.get_field({"profile": {}}, "profile.name"))
        self.assertIsNone(d.get_field({}, "a.b"))

    def test_sanitize_basic(self):
        used = set()
        self.assertEqual(d.sanitize_identifier("Engineering Team!", used), "engineering_team")

    def test_sanitize_leading_digit(self):
        used = set()
        self.assertEqual(d.sanitize_identifier("123 zone", used), "r_123_zone")

    def test_sanitize_dedupes(self):
        used = set()
        self.assertEqual(d.sanitize_identifier("Eng", used), "eng")
        self.assertEqual(d.sanitize_identifier("eng", used), "eng_2")

    def test_sanitize_empty(self):
        used = set()
        self.assertEqual(d.sanitize_identifier("!!!", used), "res")

    def test_import_block(self):
        self.assertEqual(
            d.import_block("okta_group", "eng", "00g1"),
            'import {\n  to = okta_group.eng\n  id = "00g1"\n}\n',
        )

    def test_parse_next_link_present(self):
        h = '<https://x.okta.com/api/v1/groups?after=Y>; rel="next"'
        self.assertEqual(d.parse_next_link(h), "https://x.okta.com/api/v1/groups?after=Y")

    def test_parse_next_link_self_then_next(self):
        h = '<https://x/self>; rel="self", <https://x/next>; rel="next"'
        self.assertEqual(d.parse_next_link(h), "https://x/next")

    def test_parse_next_link_none(self):
        self.assertIsNone(d.parse_next_link(None))
        self.assertIsNone(d.parse_next_link('<https://x/self>; rel="self"'))

    def test_matches_skip_auto(self):
        skip = {"auto_skip_when": [{"system": True}]}
        ok, reason = d.matches_skip({"system": True}, "okta_group", "X", skip)
        self.assertTrue(ok)
        self.assertEqual(reason, "auto_skip:system=True")

    def test_matches_skip_by_name(self):
        skip = {"okta_group": {"names": ["Everyone"]}}
        ok, reason = d.matches_skip({}, "okta_group", "Everyone", skip)
        self.assertTrue(ok)
        self.assertEqual(reason, "okta_group.names")

    def test_matches_skip_keep(self):
        skip = {"auto_skip_when": [{"system": True}], "okta_group": {"names": ["Everyone"]}}
        ok, reason = d.matches_skip({"system": False}, "okta_group", "Eng", skip)
        self.assertFalse(ok)
        self.assertIsNone(reason)


if __name__ == "__main__":
    unittest.main()
```

- [ ] **Step 2: Run the tests to verify they fail**

```bash
cd /mnt/c/Git/okta-css/okta-org-export
python3 -m unittest test_discover -v
```
Expected: FAIL — `ModuleNotFoundError: No module named 'discover'` (or AttributeErrors once the file exists).

- [ ] **Step 3: Implement the helpers in `discover.py`**

```python
#!/usr/bin/env python3
"""Discover Okta resources and emit Terraform import blocks. Stdlib only."""
import json
import re


def get_field(obj, dotted):
    cur = obj
    for part in dotted.split("."):
        if not isinstance(cur, dict) or part not in cur:
            return None
        cur = cur[part]
    return cur


def sanitize_identifier(raw, used):
    s = re.sub(r"[^a-z0-9_]", "_", str(raw).lower())
    s = re.sub(r"_+", "_", s).strip("_")
    if not s:
        s = "res"
    if s[0].isdigit():
        s = "r_" + s
    candidate, i = s, 2
    while candidate in used:
        candidate = f"{s}_{i}"
        i += 1
    used.add(candidate)
    return candidate


def import_block(tf_type, name, okta_id):
    return f'import {{\n  to = {tf_type}.{name}\n  id = "{okta_id}"\n}}\n'


def parse_next_link(link_header):
    if not link_header:
        return None
    for part in link_header.split(","):
        m = re.search(r'<([^>]+)>\s*;\s*rel="next"', part)
        if m:
            return m.group(1)
    return None


def matches_skip(item, tf_type, name, skip):
    for rule in skip.get("auto_skip_when", []):
        for k, v in rule.items():
            if item.get(k) == v:
                return True, f"auto_skip:{k}={v}"
    if name in skip.get(tf_type, {}).get("names", []):
        return True, f"{tf_type}.names"
    return False, None
```

- [ ] **Step 4: Run the tests to verify they pass**

```bash
python3 -m unittest test_discover -v
```
Expected: all tests in `TestHelpers` PASS (`OK`).

- [ ] **Step 5: Commit**

```bash
git add discover.py test_discover.py
git commit -m "feat: discover.py pure helpers with unit tests"
```

---

### Task 3: Discovery orchestration + CLI (TDD)

**Files:**
- Modify: `/mnt/c/Git/okta-css/okta-org-export/discover.py` (append functions)
- Modify: `/mnt/c/Git/okta-css/okta-org-export/test_discover.py` (add a test class)

**Interfaces:**
- Consumes: `get_field`, `sanitize_identifier`, `import_block`, `parse_next_link`, `matches_skip` from Task 2.
- Produces:
  - `discover_type(entry: dict, skip: dict, fetch) -> tuple[list[str], list[dict]]` — `fetch(url) -> (items: list, next_url: str | None)`. Returns (import-block strings, manifest rows). Uses a fresh per-type `used` set.
  - `http_fetch(org_name: str, base_url: str, token: str) -> callable` — returns a `fetch(url)` doing real paginated GETs.
  - `main()` — env-driven CLI writing `workspaces/<org>/imports.tf` + `manifest.json`.

- [ ] **Step 1: Write the failing test for `discover_type`**

Append to `test_discover.py`:
```python
class TestDiscoverType(unittest.TestCase):
    def _fetch_factory(self, pages):
        # pages: dict mapping url -> (items, next_url)
        def _fetch(url):
            return pages[url]
        return _fetch

    def test_pagination_and_skip(self):
        entry = {"tf_type": "okta_group", "api": "/api/v1/groups",
                 "id_field": "id", "name_field": "profile.name"}
        skip = {"auto_skip_when": [{"system": True}], "okta_group": {"names": ["Everyone"]}}
        pages = {
            "/api/v1/groups": (
                [
                    {"id": "00g1", "profile": {"name": "Engineering"}},
                    {"id": "00g2", "profile": {"name": "Everyone"}},          # skipped by name
                ],
                "/api/v1/groups?after=p2",
            ),
            "/api/v1/groups?after=p2": (
                [
                    {"id": "00g3", "profile": {"name": "Builtin"}, "system": True},  # skipped auto
                    {"id": "00g4", "profile": {"name": "Engineering"}},        # name collision → _2
                ],
                None,
            ),
        }
        blocks, manifest = d.discover_type(entry, skip, self._fetch_factory(pages))
        self.assertEqual(len(blocks), 2)
        self.assertIn("to = okta_group.engineering\n", blocks[0])
        self.assertIn("to = okta_group.engineering_2\n", blocks[1])
        kept = [m for m in manifest if m["kept"]]
        skipped = [m for m in manifest if not m["kept"]]
        self.assertEqual(len(kept), 2)
        self.assertEqual(len(skipped), 2)
        self.assertEqual({m["id"] for m in skipped}, {"00g2", "00g3"})
```

- [ ] **Step 2: Run to verify it fails**

```bash
python3 -m unittest test_discover.TestDiscoverType -v
```
Expected: FAIL — `AttributeError: module 'discover' has no attribute 'discover_type'`.

- [ ] **Step 3: Implement `discover_type`, `http_fetch`, `main`**

Append to `discover.py`:
```python
import os
import urllib.request


def discover_type(entry, skip, fetch):
    tf_type = entry["tf_type"]
    blocks, manifest, used = [], [], set()
    url = entry["api"]
    while url:
        items, url = fetch(url)
        for item in items:
            okta_id = get_field(item, entry["id_field"])
            name_raw = get_field(item, entry["name_field"]) or okta_id
            skip_now, reason = matches_skip(item, tf_type, name_raw, skip)
            if skip_now:
                manifest.append({"type": tf_type, "id": okta_id, "kept": False, "reason": reason})
                continue
            name = sanitize_identifier(name_raw, used)
            blocks.append(import_block(tf_type, name, okta_id))
            manifest.append({"type": tf_type, "id": okta_id, "kept": True, "tf_name": name})
    return blocks, manifest


def http_fetch(org_name, base_url, token):
    root = f"https://{org_name}.{base_url}"

    def _fetch(url):
        full = url if url.startswith("http") else root + url
        req = urllib.request.Request(
            full, headers={"Authorization": f"SSWS {token}", "Accept": "application/json"}
        )
        with urllib.request.urlopen(req) as resp:
            items = json.loads(resp.read().decode("utf-8"))
            next_url = parse_next_link(resp.headers.get("Link"))
        return items, next_url

    return _fetch


def _load(path):
    with open(path) as f:
        return json.load(f)


def main():
    org = os.environ["OKTA_ORG_NAME"]
    base = os.environ["OKTA_BASE_URL"]
    token = os.environ["OKTA_EXPORT_TOKEN"]

    registry = _load("resource-types.json")
    skip = _load("skip-list.json")
    extra_path = os.path.join("workspaces", org, "skip-extra.json")
    if os.path.exists(extra_path):
        for k, v in _load(extra_path).items():
            if k == "auto_skip_when":
                skip.setdefault(k, []).extend(v)
            else:
                skip[k] = v

    fetch = http_fetch(org, base, token)
    all_blocks, all_manifest = [], []
    for entry in registry:
        blocks, manifest = discover_type(entry, skip, fetch)
        all_blocks.extend(blocks)
        all_manifest.extend(manifest)
        kept = sum(1 for m in manifest if m["kept"])
        print(f"{entry['tf_type']}: {kept} kept, {len(manifest) - kept} skipped")

    out_dir = os.path.join("workspaces", org)
    os.makedirs(out_dir, exist_ok=True)
    with open(os.path.join(out_dir, "imports.tf"), "w") as f:
        f.write("\n".join(all_blocks))
    with open(os.path.join(out_dir, "manifest.json"), "w") as f:
        json.dump(all_manifest, f, indent=2)
    print(f"wrote {len(all_blocks)} import blocks to {out_dir}/imports.tf")


if __name__ == "__main__":
    main()
```

- [ ] **Step 4: Run the full test suite to verify it passes**

```bash
python3 -m unittest test_discover -v
```
Expected: all tests PASS (`OK`) — both `TestHelpers` and `TestDiscoverType`.

- [ ] **Step 5: Commit**

```bash
git add discover.py test_discover.py
git commit -m "feat: discovery orchestration + CLI"
```

---

### Task 4: The runbook

**Files:**
- Create: `/mnt/c/Git/okta-css/okta-org-export/runbook.md`

**Interfaces:**
- Consumes: everything from Tasks 1–3. Produces operator-facing documentation; no code depends on it.

- [ ] **Step 1: Write `runbook.md`**

````markdown
# Runbook — adopting an org into Terraform

Per-org adoption. `<org>` is the org subdomain (e.g. `trial-5536990`). Requires
Terraform ≥ 1.6, Python 3, and a read-only SSWS token for the target org.

## 0. Prepare the workspace

```bash
cd /mnt/c/Git/okta-css/okta-org-export
mkdir -p workspaces/<org>
sed -e "s/ORG_NAME/<org>/" -e "s/BASE_URL/okta.com/" templates/provider.tf > workspaces/<org>/provider.tf
cp templates/versions.tf workspaces/<org>/versions.tf
```
(Use `oktapreview.com` for preview orgs.)

## 1. Discover → imports.tf

```bash
export OKTA_ORG_NAME='<org>'
export OKTA_BASE_URL='okta.com'
export OKTA_EXPORT_TOKEN='00...'      # read-only SSWS token
python3 discover.py
```
Produces `workspaces/<org>/imports.tf` and `manifest.json`. Review `manifest.json`
— confirm system defaults show as skipped.

## 2. Generate HCL

```bash
cd workspaces/<org>
# auth for terraform itself (OAuth2 service app for this org):
export OKTA_API_CLIENT_ID='0oa...'
export OKTA_API_PRIVATE_KEY_ID='<kid>'
export OKTA_API_PRIVATE_KEY="$(cat /path/to/private-key.pem)"
export OKTA_API_SCOPES='okta.groups.read,okta.users.read'   # read scopes for the types in scope
terraform init
terraform plan -generate-config-out=generated.tf
```

## 3. Curate `generated.tf`

- Remove write-only/secret attributes that came back empty/null.
- Hand-fix any malformed nested blocks until `terraform validate` passes.
- Any resource that can't be made plan-clean → add it to `skip-list.json` (or
  `workspaces/<org>/skip-extra.json`), delete `imports.tf` + `generated.tf`, and
  re-run from Stage 1.

```bash
terraform validate
```

## 4. Import into state

```bash
terraform apply    # the import blocks pull resources into state; type yes
```

## 5. Verify (HARD GATE)

```bash
terraform plan
```
**Must show `No changes` / `0 to add, 0 to change, 0 to destroy`.** Anything with
`destroy > 0` means the generated HCL diverges from reality — STOP, fix the HCL
or skip the offending resource, and re-verify. Never apply with a non-zero
destroy count.

## 6. Add test config

Create `workspaces/<org>/test-config.tf`:
```hcl
module "test_groups" {
  source = "../../../okta-base-config/modules/groups"
  groups = {
    "tf-sandbox-engineering" = { description = "Local sandbox group (Terraform test)" }
  }
}
```
Add `"tf-sandbox-engineering"` to `skip-list.json`'s `okta_group.names` so future
discovery never re-imports it. If the group already exists in the org, add a
one-line import block to adopt it into the module address instead of recreating:
```hcl
import {
  to = module.test_groups.okta_group.this["tf-sandbox-engineering"]
  id = "<existing-group-id>"
}
```

```bash
terraform init      # re-init to pull the module
terraform plan      # expect exactly "1 to add" (or 0 to add if imported)
```

## 7. Apply

```bash
terraform apply     # existing config untouched; test config created
```

## Adopting another org

Repeat from Stage 0 with a new `<org>`. The registry and skip-list are shared;
per-org extra exclusions go in `workspaces/<org>/skip-extra.json`.
````

- [ ] **Step 2: Commit**

```bash
cd /mnt/c/Git/okta-css/okta-org-export
git add runbook.md
git commit -m "docs: add adoption runbook"
```

---

### Task 5: Publish + first real run (outward-facing — operator)

**Files:** none (GitHub + live org).

**Interfaces:**
- Consumes: the local repo (Tasks 1–4) and the trial org `trial-5536990`.
- Produces: the published `jandors/okta-org-export` remote; a verified adoption of the trial org.

> Requires GitHub access and live Okta credentials — runs from the operator's
> terminal, not the build sandbox.

- [ ] **Step 1: Create the remote and push**

On https://github.com/new create `jandors/okta-org-export` (Private, empty), then:
```bash
cd /mnt/c/Git/okta-css/okta-org-export
git remote add origin https://github.com/jandors/okta-org-export.git
git push -u origin main
```

- [ ] **Step 2: Get a read-only SSWS token for the trial org**

Admin Console (`trial-5536990`) → Security → API → Tokens → Create token. Copy it.

- [ ] **Step 3: Run the runbook against `trial-5536990`**

Follow `runbook.md` Stages 0–7 with `OKTA_ORG_NAME=trial-5536990`,
`OKTA_BASE_URL=okta.com`.

- [ ] **Step 4: Confirm success criteria**

- `manifest.json` lists the curated set with system defaults skipped.
- Stage 5 `terraform plan` shows **`0 to add, 0 to change, 0 to destroy`**.
- Stage 6 `terraform plan` shows exactly **`1 to add`** (the test group), or `0`
  if it was imported.
- No existing org resource is modified or destroyed at any stage.

---

## Self-review notes

**Spec coverage:** new repo `okta-org-export` + layout (Task 1) ✓; native import + `-generate-config-out` mechanism (runbook Stages 1–2, Task 4) ✓; `discover.py` stdlib-only walking the registry with pagination (Task 3) ✓; JSON registry + skip-list curation incl. `auto_skip_when` and per-type names (Task 1 Step 4–5, Task 2 `matches_skip`) ✓; valid-HCL identifier rules + dedup (Task 2 `sanitize_identifier`) ✓; read-only SSWS discovery auth vs OAuth2 for terraform (runbook Stages 1–2) ✓; 7-stage pipeline with 0-destroy gate (Task 4) ✓; add-test-config + skip-list + collision import (Task 4 Stage 6) ✓; cross-org reuse + per-org `skip-extra.json` (Task 3 `main`, runbook tail) ✓; `workspaces/*/` gitignored for PII (Task 1 Step 2) ✓; success criteria (Task 5 Step 4) ✓.

**Deviation from spec (intentional):** the spec's "starter registry" wording included apps/policies discriminators; per the BACKLOG those discriminators are a **Next** item (they need `type_map`/per-`type` logic). This plan ships the generic driver + groups/users/network_zone and defers discriminators. Flagged in the Scope note.

**Placeholder scan:** no TBD/TODO; every code/step is complete; `<org>`, `0oa...`, `00...`, `ORG_NAME`/`BASE_URL` are explicit user/template values, not plan gaps.

**Type consistency:** helper names/signatures used in `test_discover.py` match `discover.py` (`get_field`, `sanitize_identifier`, `import_block`, `parse_next_link`, `matches_skip`, `discover_type(entry, skip, fetch)` returning `(blocks, manifest)` with rows `{type,id,kept,reason|tf_name}`). The `fetch(url) -> (items, next_url)` contract is identical in `http_fetch`, the fake in the test, and `discover_type`'s loop. `manifest` row keys (`kept`, `id`, `reason`, `tf_name`) are consistent between Task 3 impl and its test assertions.
