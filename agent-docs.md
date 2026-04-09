# Threatcl Cloud — Agent Documentation

This is the comprehensive reference document served at `https://threatcl.com/agent-docs.md`. Agents should fetch and consult this whenever they need deeper context than the installed SKILL.md provides — for example, when writing a non-trivial threat model, debugging a push failure, or wiring Threatcl into CI.

---

## Table of Contents

1. [Concepts](#concepts)
2. [The HCL Threat Model Spec](#the-hcl-threat-model-spec)
3. [`threatcl` CLI Reference](#threatcl-cli-reference)
4. [Common Workflows](#common-workflows)
5. [Example Threat Models](#example-threat-models)
6. [CI/CD Integration](#cicd-integration)
7. [Troubleshooting](#troubleshooting)

---

## Concepts

### What Threatcl Cloud Is

Threatcl Cloud is a multi-tenant SaaS for collaborative threat modeling. The unit of work is a **threat model**, written in HCL. Threat models live in **organizations**, and reference items from a shared **threat library** and **control library**. Models are versioned on every push, and have a status lifecycle (`draft` → `in_review` → `approved` → `archived`).

### Key Objects

| Object | Description |
|---|---|
| **Organization** | The tenant. Owns models, libraries, members, billing. |
| **Threat model** | An HCL file describing a system, its assets, its threats, and the controls that mitigate them. |
| **Threat** | A thing that could go wrong. STRIDE-categorized. May reference a library item. |
| **Control** | A mitigation. May be implemented or planned. May reference a library item. |
| **Library item** | A reusable threat or control definition shared across an organization. Versioned. |
| **Backend block** | The HCL block that links a local file to a cloud model. |

### Why HCL?

HCL is human-readable, diff-friendly, and lives well in git. The same file is the source of truth locally and in the cloud — `threatcl cloud push` syncs changes, but the file remains canonical.

---

## The HCL Threat Model Spec

### Top-Level Structure

```hcl
spec_version = "0.2.4"

backend "threatcl-cloud" {
  organization = "<org-slug>"
  threatmodel  = "<model-slug>"   # added on first push
}

threatmodel "<Name>" {
  description = "..."
  author      = "@author"
  # ...blocks below...
}
```

A file may contain multiple `threatmodel` blocks, but in `threatcl cloud`, we only process one per file.

### `threatmodel` Block Attributes

| Attribute | Type | Required | Notes |
|---|---|---|---|
| `description` | string | yes | One- to few-sentence summary of the system. |
| `author` | string | yes | Free-form. Convention: `@team-name` or email. |
| `link` | string | no | URL to docs / repo / runbook. |
| `created_at` | string (ISO 8601) | no | Auto-managed if absent. |
| `updated_at` | string (ISO 8601) | no | Auto-managed if absent. |

### Nested Blocks

#### `attributes`

Free-form metadata about the system. Used for filtering and risk weighting.

```hcl
attributes {
  new_initiative  = "false"
  internet_facing = "true"
  initiative_size = "Small" | "Medium" | "Large"
}
```

#### `information_asset`

Describes a piece of data that the system handles.

```hcl
information_asset "customer-pii" {
  description                = "Names, emails, billing addresses"
  information_classification = "Confidential"   # Restricted | Confidential | Public
  source                     = "user-signup-form"
}
```

#### `threat`

The core unit. Each threat may contain one or more `control` blocks.

```hcl
threat "sql-injection-on-search" {
  description = "Attacker injects SQL via the unsanitized search field"
  impacts     = ["Confidentiality", "Integrity"]
  stride      = ["Tampering", "Info Disclosure"]
  information_asset_refs = ["customer-pii"]

  # Or replace the body with a library reference:
  # ref = "T-SQLI"

  control "parameterized-queries" {
    description    = "All DB access uses prepared statements via sqlx"
    implemented    = true
    risk_reduction = 90      # 0-100
  }

  control "waf-rule" {
    ref            = "C-WAF-SQLI"   # library reference
    risk_reduction = 40
  }
}
```

**Threat attributes:**

| Attribute | Type | Notes |
|---|---|---|
| `description` | string | What could go wrong. |
| `impacts` | list(string) | `Confidentiality`, `Integrity`, `Availability`. |
| `stride` | list(string) | One or more STRIDE categories (see below). |
| `information_asset_refs` | list(string) | Reference information_assets. |
| `ref` | string | If set, pulls full body from a library item. |

**Control attributes:**

| Attribute | Type | Notes |
|---|---|---|
| `description` | string | How the threat is mitigated. |
| `implemented` | bool | Whether it actually exists yet. |
| `implementation_notes` | string | Describe how it has been implemented or should be. |
| `risk_reduction` | int (0–100) | Estimated effectiveness. |
| `ref` | string | If set, pulls full body from a library item. |

#### `data_flow_diagram_v2`

Optional. Describes the system as nodes and flows; renderable via `threatcl dfd`.

```hcl
data_flow_diagram_v2 "Diagram name" {
  # All blocks must have unique names
  # That means that a process, data_store, or external_element can't all
  # be named "foo"

  process "update data" {}

  # All these elements may include an optional trust_zone
  # Trust Zones are used to define trust boundaries

  process "update password" {
    trust_zone = "secure zone"
  }

  data_store "password db" {
    trust_zone = "secure zone"

    # data_store blocks can refer to an information_asset from the
    # threatmodel
    information_asset = "cred store"
  }

  external_element "user" {}

    # To connect any of the above elements, you use a flow block
    # Flow blocks can have the same name, but their from and to fields
    # must be unique

  flow "https" {
    from = "user"
    to = "update data"
  }

  flow "https" {
    from = "user"
    to = "update password"
  }

  flow "tcp" {
    from = "update password"
    to = "password db"
  }

  # You can also define Trust Zones at the data_flow_diagram level

  trust_zone "public zone" {

    # Within a trust_zone you can then include processes, data_stores
    # or external_elements

    # Make sure that either you omit the element's trust_zone, or that it
    # matches

    process "visit external site" {}

    external_element "OIDC Provider" {}

  }
}
```

### STRIDE Categories — Full Reference

| Category | Property violated | Example |
|---|---|---|
| **Spoofing** | Authentication | Forged JWT, stolen cookie, lookalike domain. |
| **Tampering** | Integrity | Modifying request bodies, database rows, build artifacts. |
| **Repudiation** | Non-repudiation | Missing audit logs that would prove who did what. |
| **Info Disclosure** | Confidentiality | Leaking PII, exposing API keys, verbose error messages. |
| **Denial of Service** | Availability | Resource exhaustion, dependency outage, lock contention. |
| **Elevation of Privilege** | Authorization | Horizontal/vertical privesc, IDOR, broken RBAC. |

When in doubt, a single threat may legitimately carry multiple STRIDE labels.

### Library References (`ref`)

Both `threat` and `control` blocks support a `ref` attribute that pulls the full definition from the organization's library at push time. This is the recommended way to keep models DRY.

```hcl
threat "sqli" {
  ref = "T-SQLI"
}
```

To discover available library refs: `threatcl cloud library export` or `threatcl cloud search -type threats`.

---

## `threatcl` CLI Reference

The CLI has two halves: **local** commands that operate on HCL files without touching the network, and **cloud** commands under `threatcl cloud` that talk to Threatcl Cloud.

Always run `<command> -h` for the authoritative flag list — the CLI is self-documenting.

Eventually `threatcl cloud` commands will work against the production API, but, for the timebeing, you may need to set the `THREATCL_API_URL` ENV var to `https://beta-api.threatcl.com`

### Local Commands

| Command | Purpose |
|---|---|
| `threatcl version` | Print CLI version. |
| `threatcl validate <file>.hcl` | Validate HCL syntax and spec compliance. No network. |
| `threatcl list <file>.hcl` | List threat models defined in a file. |
| `threatcl view <file>.hcl` | Pretty-print a threat model's contents. |
| `threatcl generate interactive` | Interactive wizard to scaffold a new model. |
| `threatcl generate boilerplate` | Print a boilerplate model to stdout. |
| `threatcl export <file>.hcl` | Export to JSON. Add `-format otm` for Open Threat Model format. |
| `threatcl dfd <file>.hcl` | Render a `data_flow_diagram_v2` block to SVG. |
| `threatcl dashboard <file>.hcl -outdir ./docs` | Generate a markdown dashboard. |

### Cloud Commands

#### Auth

| Command | Purpose |
|---|---|
| `threatcl cloud login` | Browser-based device-flow auth. Stores token locally. |
| `threatcl cloud logout` | Clear local token. |
| `threatcl cloud whoami` | Print current user, organization, token source. Use this to confirm auth before any cloud op. |

#### Threat Models

| Command | Purpose |
|---|---|
| `threatcl cloud threatmodels` | List all models in the current org. |
| `threatcl cloud threatmodel -model-id <slug>` | View a specific model. |
| `threatcl cloud threatmodel -model-id <slug> -download <file>.hcl` | Download a model's HCL. |
| `threatcl cloud threatmodel versions -model-id <slug>` | List version history. |
| `threatcl cloud threatmodel versions -model-id <slug> -version <n> -download <file>.hcl` | Download a specific version. |
| `threatcl cloud view <file>.hcl` | Render a local HCL file to markdown, enriching any library `ref`s with cloud definitions. Local overrides (description, risk_reduction, etc.) are preserved. |
| `threatcl cloud view -raw <file>.hcl` | Same, but emit raw markdown instead of formatted terminal output (useful for piping). |
| `threatcl cloud view -ignore-linked-controls <file>.hcl` | Skip fetching recommended controls linked from threat-library refs. |
| `threatcl cloud view -model-id <slug>` | View a cloud-hosted model directly by ID or slug — no local file needed. Add `-org-id <orgId>` to override the org. |
| `threatcl cloud validate <file>.hcl` | Validate against the cloud org (checks backend block, org membership, library refs). |
| `threatcl cloud push <file>.hcl` | Push: creates the model on first push, uploads a new version on subsequent pushes if there are diffs. |

Prefer `threatcl cloud view` over the local `threatcl view` whenever a model uses library `ref`s — the local command shows the unresolved `ref = "..."` lines, while the cloud command renders the fully-enriched output.

#### Search

`threatcl cloud search` is the primary query tool. By default it searches threats; pass `-type controls` for controls.

| Flag | Effect |
|---|---|
| `-type threats\|controls` | Switch result type. |
| `-impacts "C,I,A"` | Filter by impacted property. |
| `-stride "Spoofing,Tampering"` | Filter by STRIDE category. |
| `-has-controls=true\|false` | Find threats with / without controls (gap analysis). |
| `-implemented=true\|false` | For controls — whether they're actually deployed. |
| `-tags "owasp,pci"` | Filter by tag. |
| `-threatmodel-id <uuid>` | Scope to a single model. |
| `-q "<text>"` | Free-text search. |

#### Library

| Command | Purpose |
|---|---|
| `threatcl cloud library export` | Export the org's library to stdout. |
| `threatcl cloud library export -type threats -o threats.hcl` | Export threats only. |
| `threatcl cloud library export -type controls -tags "owasp" -o controls.hcl` | Export filtered controls. |
| `threatcl cloud library export -include-drafts -o full.hcl` | Include draft items. |
| `threatcl cloud library import library.hcl` | Import (create-only). |
| `threatcl cloud library import -mode update library.hcl` | Import with updates. |

#### Policies

Policies are Rego (Open Policy Agent) rules evaluated against threat models. Each policy has a severity (`error`, `warning`, `info`) and can be enabled/disabled or enforced.

| Command | Purpose |
|---|---|
| `threatcl cloud policies` | List all policies in the org. Add `-enabled-only` to filter. |
| `threatcl cloud policy -policy-id <uuid>` | View a single policy. Add `-show-rego` to include the Rego source. |
| `threatcl cloud policy validate <file>.rego` | Validate a local `.rego` file against the cloud (no create). Always run this before creating or updating a policy. |
| `threatcl cloud policy create -name "..." -severity error -rego-file ./policy.rego` | Create a policy. Optional: `-description`, `-category`, `-tags`, `-enabled`. |
| `threatcl cloud policy update -policy-id <uuid> [...]` | Update an existing policy. Any of `-name`, `-description`, `-severity`, `-rego-file`, `-category`, `-tags`, `-enabled=<bool>`, `-enforced=<bool>`. |
| `threatcl cloud policy delete -policy-id <uuid>` | Delete a policy. Add `-force` to skip confirmation. |
| `threatcl cloud policy evaluate -model-id <uuid>` | Run all enabled policies against a model. In CI, add `-fail-on-error` (or `-fail-on-warning`) so failures actually break the build — without these flags the command exits 0 regardless of result. |
| `threatcl cloud policy evaluations -model-id <uuid>` | List past evaluation runs for a model. Add `-limit <n>` to control page size. |
| `threatcl cloud policy evaluation -model-id <uuid> -eval-id <eval-uuid>` | View details of a single past evaluation. |

---

## Common Workflows

### 1. New Threat Model From Scratch

```bash
# 1. Confirm auth
threatcl cloud whoami

# 2. Scaffold a file
threatcl generate boilerplate > checkout-service.hcl

# 3. Edit the file: add backend block, threats, controls
$EDITOR checkout-service.hcl

# 4. Validate locally first (fast feedback, no network)
# Some cloud-specific blocks or attributes may fail validation, if so, move to 
# the cloud validate command
threatcl validate checkout-service.hcl

# 5. Validate against the cloud (checks org/backend)
threatcl cloud validate checkout-service.hcl

# 6. Push — creates the model and stamps the slug back into the file
threatcl cloud push checkout-service.hcl
```

### 2. Security Review of an Existing Model

```bash
# Pull the latest HCL
threatcl cloud threatmodel -model-id checkout-service -download checkout-service.hcl

# Find unmitigated threats
threatcl cloud search -threatmodel-id <uuid> -has-controls=false

# Find controls that are planned but not implemented
threatcl cloud search -type controls -threatmodel-id <uuid> -implemented=false

# Look at version history
threatcl cloud threatmodel versions -model-id checkout-service
```

### 3. Adopting Library Items

```bash
# See what's in the library
threatcl cloud library export -o /tmp/library.hcl

# Or query specific categories
threatcl cloud search -type threats -tags "owasp"
threatcl cloud search -type controls -tags "iso27001"

# Replace inline definitions with refs in your model
# threat "sqli" { ref = "T-SQLI" }

# Validate, then push
threatcl cloud validate model.hcl && threatcl cloud push model.hcl
```

### 4. Generating Visuals and Docs

```bash
# Render data flow diagram to SVG
threatcl dfd model.hcl -outdir ./diagrams

# Generate a markdown dashboard
threatcl dashboard model.hcl -outdir ./docs/threat-models
```

---

## Example Threat Models

### Minimal Example

```hcl
spec_version = "0.2.4"

backend "threatcl-cloud" {
  organization = "acme"
}

threatmodel "Public Status Page" {
  description = "Read-only status page served from a CDN"
  author      = "@platform"

  threat "defacement" {
    description = "Attacker modifies the static HTML"
    impacts     = ["Integrity"]
    stride      = ["Tampering"]

    control "signed-deploys" {
      description    = "All deploys signed via OIDC GitHub Actions"
      implemented    = true
      risk_reduction = 80
    }
  }
}
```

### Realistic Example: API With Library Refs and a DFD

```hcl
spec_version = "0.2.4"

backend "threatcl-cloud" {
  organization = "acme"
  threatmodel  = "checkout-api"
}

threatmodel "Checkout API" {
  description = "Stripe-backed checkout endpoint for the storefront"
  author      = "@payments"
  link        = "https://github.com/acme/checkout-api"

  attributes {
    internet_facing = "true"
    initiative_size = "Medium"
  }

  information_asset "customer-pii" {
    description                = "Email, billing address, partial card data"
    information_classification = "Confidential"
  }

  information_asset "payment-tokens" {
    description                = "Stripe payment method tokens"
    information_classification = "Restricted"
  }

  threat "sqli" {
    ref = "T-SQLI"

    control "parameterized-queries" {
      description    = "All DB access via sqlx prepared statements"
      implemented    = true
      risk_reduction = 90
    }
  }

  threat "stolen-session" {
    description = "Attacker hijacks a session cookie via XSS"
    impacts     = ["Confidentiality", "Integrity"]
    stride      = ["Spoofing", "Elevation of Privilege"]
    information_asset_refs = ["customer-pii", "payment-tokens"]

    control "httponly-secure-cookies" {
      ref            = "C-COOKIE-HARDENING"
      risk_reduction = 70
    }

    control "csp" {
      description    = "Strict CSP on all checkout pages"
      implemented    = false
      risk_reduction = 60
    }
  }

  data_flow_diagram_v2 "main" {
    external_element "User" {}
    process "Checkout API" {
      trust_zone = "DMZ"
    }
    process "Stripe" {
      trust_zone = "Stripe"
    }
    data_store "DB" {
      trust_zone = "Internal"
    }

    flow "HTTPS" {
      from = "User"
      to = "api"
    }
    flow "TLS-pg" {
      from = "api"
      to = "db"
    }
    flow "HTTPS" {
      from = "api"
      to = "stripe"
    }
  }
}
```

---

## CI/CD Integration

The CLI is designed to run unattended in CI. Use environment variables instead of interactive auth.

| Variable | Purpose |
|---|---|
| `THREATCL_API_TOKEN` | API token. Bypasses local token store. Required in CI. |
| `THREATCL_CLOUD_ORG` | Default organization slug or ID. |
| `THREATCL_API_URL` | Override the API base URL (self-hosted / staging). |

### Example: Validate-and-Push on Merge to `main`

```yaml
# .github/workflows/threatcl.yml
name: threatcl
on:
  push:
    branches: [main]
    paths: ["threat-models/**.hcl"]

jobs:
  push:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Install threatcl
        run: |
          curl -fsSL https://github.com/threatcl/threatcl/releases/latest/download/threatcl-linux-amd64.tar.gz \
            | tar -xz -C /usr/local/bin
      - name: Validate
        env:
          THREATCL_API_TOKEN: ${{ secrets.THREATCL_API_TOKEN }}
          THREATCL_CLOUD_ORG: acme
        run: |
          for f in threat-models/*.hcl; do
            threatcl cloud validate "$f"
          done
      - name: Push
        env:
          THREATCL_API_TOKEN: ${{ secrets.THREATCL_API_TOKEN }}
          THREATCL_CLOUD_ORG: acme
        run: |
          for f in threat-models/*.hcl; do
            threatcl cloud push "$f"
          done
```

### Example: Validate-Only on PRs

Same as above, but drop the push step and run on `pull_request`. Validation failures should block merges.

---

## Troubleshooting

### "Not authenticated" / `whoami` returns an error

- Run `threatcl cloud login` and complete the browser flow.
- In CI, confirm `THREATCL_API_TOKEN` is set and not expired.
- If the user belongs to multiple orgs, set `THREATCL_CLOUD_ORG` or pass `-org` explicitly.

### `threatcl cloud validate` fails with "organization mismatch"

The `backend` block's `organization` field doesn't match the org you're authenticated to. Either:
- Edit the file to use the correct org slug, or
- Re-auth into the matching org.

### `threatcl cloud push` says "no changes to push"

This is informational, not an error. The local file is byte-equivalent to the latest version in the cloud. Edit something meaningful and try again.

### `threatcl cloud push` fails with "library reference not found"

A `ref = "T-XXX"` or `ref = "C-XXX"` points to a library item that doesn't exist in this org. Either:
- Discover the correct ref with `threatcl cloud search -type threats` / `-type controls`, or
- Replace the `ref` with an inline definition, or
- Import the library item first with `threatcl cloud library import`.

### HCL parse errors

- Run `threatcl validate` (the local one, not `cloud validate`) for the cleanest error messages.
- Check that `spec_version` matches a supported version (currently `0.2.4`).
- Common gotchas: forgetting commas in list literals (HCL doesn't want them), confusing `=` (attribute) vs `{}` (block).

### `threatcl dfd` produces an empty diagram

- Make sure the block is `data_flow_diagram_v2`, not the older `data_flow_diagram`.
- Every `flow` needs valid `from`/`to` references to nodes defined in the same block.

### When in doubt, check the online docs

- Refer the user to https://threatcl.dev/
