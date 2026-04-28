# agent-skills — Specification

**Version**: 0.1.0 (draft)
**Status**: Open for comment. Nothing here is frozen.

## 0. Notational conventions

The key words **MUST**, **MUST NOT**, **SHOULD**, **SHOULD NOT**, and **MAY** are to be interpreted as in [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119).

A "skill bank" is the runtime that subscribes to skill providers, indexes their skills, and exposes them to an agent. The reference skill bank uses [`just-bash-data`](https://www.npmjs.com/package/just-bash-data) but the spec is implementation-agnostic — anything that can store metadata and do nearest-neighbor vector search qualifies.

A "skill provider" is whoever publishes one or more skills.

A "skill consumer" is the agent runtime that executes a skill on behalf of a user.

## 1. Skill identity

Every skill has a globally unique identity composed of three parts:

```
<source>@<version-or-sha>/<path>
```

Where:

- `<source>` is one of:
  - `<host>/<owner>/<repo>` for a git-hosted skill (e.g., `github.com/stripe/agent-skills`)
  - A bare URL host for a server-hosted skill (e.g., `img.automators.work`)
- `<version-or-sha>` is:
  - A 40-character lowercase hex git commit SHA (preferred, immutable), OR
  - A semver tag (`v1.2.0`) (mutable apuntador to a SHA), OR
  - The literal string `latest` (NOT recommended — see §6 Versioning)
- `<path>` is the path within the source to the `SKILL.md` file's containing directory, slash-separated.

Examples of well-formed identity strings:

```
github.com/stripe/agent-skills@a1b2c3d4e5f67890abcdef1234567890abcdef12/charge-customer
github.com/openai/agent-skills@v3.4.5/embeddings
img.automators.work@latest/placeholder
```

Skill banks **MUST** store the full identity as the document's primary key when indexing. Skill banks **SHOULD** prefer SHA-pinned identities over tag-pinned ones for production use.

## 2. The `SKILL.md` file

A `SKILL.md` file is a UTF-8-encoded plain text file with **YAML frontmatter** followed by **markdown body**. The frontmatter is machine-readable; the body is for humans (or for LLM consumption when more context is needed).

### 2.1 File structure

```yaml
---
# (required fields, see §2.2)
schema_version: "1.0"
id: "<short-id>"
version: "<semver>"
title: "<human-readable name>"
description: "<one-paragraph what-it-does>"
use_when: "<one-sentence when-an-agent-should-pick-this>"
command_template: "<bash command, with {placeholder} args>"

# (recommended fields, see §2.3)
license: "..."
author: { name: "...", url: "..." }
homepage: "..."
category: "..."
tags: ["...", "..."]
args: { ... }
examples: [ ... ]

# (optional fields, see §2.4)
required_commands: ["..."]
required_env: ["..."]
network: ["..."]
applicable_when: { ... }
deprecates: ["..."]
related: ["..."]
chains: [ ... ]

# (provenance — auto-populated by tooling, see §2.5)
provenance: { ... }
---

# Title

[Optional human-readable prose. Markdown.]

## When to use this

...

## When NOT to use this

...

## Examples

...
```

### 2.2 Required fields

Every conformant `SKILL.md` **MUST** declare these frontmatter fields:

| Field | Type | Description |
|---|---|---|
| `schema_version` | string | The spec version this skill conforms to. Currently `"1.0"`. |
| `id` | string | Stable identifier within the publisher's namespace. **MUST** match `^[a-z][a-z0-9_-]{0,63}$`. |
| `version` | string | Semver (`MAJOR.MINOR.PATCH`). Bumped on any change. |
| `title` | string | Human-readable name, ≤ 80 chars. |
| `description` | string | What the skill does. Plain prose, ≤ 500 chars. Indexed for embedding (see §4.2). |
| `use_when` | string | One sentence describing when an agent should pick this skill. **This is the primary signal for retrieval.** Indexed for embedding. |
| `command_template` | string | The bash command(s) the skill executes. May contain `{placeholder}` substitutions filled at runtime. |

### 2.3 Recommended fields

Every well-formed `SKILL.md` **SHOULD** declare:

| Field | Type | Description |
|---|---|---|
| `license` | SPDX string | E.g., `"MIT"`, `"Apache-2.0"`. Required for legal redistribution. |
| `author` | object | `{ name: string, url?: string, email?: string }`. |
| `homepage` | URL | Where to learn more. |
| `category` | string | Coarse-grained category (e.g., `"data-management"`, `"image-generation"`). |
| `tags` | array of strings | Fine-grained labels for filtering. |
| `args` | object | Schema for `{placeholder}` substitutions. See §2.6. |
| `examples` | array of objects | Each `{ intent: string, command: string, expected_output?: string }`. **Indexed for embedding** — including diverse formulations of the same intent significantly improves retrieval. |

### 2.4 Optional fields

| Field | Type | Description |
|---|---|---|
| `required_commands` | array of strings | Commands the `command_template` invokes (e.g., `["jq", "curl"]`). Skill banks **MAY** filter skills out if a required command is unavailable. |
| `required_env` | array of strings | Environment variable **names** the skill expects. **Values are never published.** |
| `network` | array of URL patterns | Allowlist of hosts the skill might contact. Used by sandboxed runtimes to enforce a permission boundary. |
| `applicable_when` | object | Conditions under which the skill is applicable. See §2.7. |
| `deprecates` | array of identity strings | Skill identities this version replaces. Skill banks **SHOULD** mark superseded skills as `deprecated: true` upon syncing. |
| `migration_notes` | string | Markdown prose explaining migration from the deprecated skill. |
| `related` | array of identity strings | Cross-references to related skills. Used for "did you mean" / "see also" UX. |
| `chains` | array of objects | Multi-step composition. See §2.8. |

### 2.5 Provenance fields (auto-populated)

These fields are **NOT written by the skill author**. They are filled by the publishing tooling (CI/CD) and verified by skill banks at ingest time:

```yaml
provenance:
  source: "git"                # or "url"
  repo: "github.com/stripe/agent-skills"     # for git source
  commit: "a1b2c3d4..."        # SHA at publish time (40 hex chars)
  tag: "v1.2.0"                # optional; if the publish was tagged
  signed_by: "stripe-llm.gpg"  # optional; fingerprint of the signing key
  published_at: "2026-04-28T12:34:56Z"
  publisher_verified: true     # set by skill bank at ingest if signature verified
```

Skill banks **MUST** record `provenance` on every indexed skill so the agent's `find` queries can filter by it (e.g., "only skills signed by trusted-keys").

### 2.6 The `args` schema

When `command_template` contains placeholders, `args` documents them. Format:

```yaml
args:
  amount:
    type: integer        # one of: integer, number, string, boolean
    description: "amount in cents"
    range: [1, 100000]   # for numeric types; closed interval
    default: null        # if provided, placeholder is optional
  currency:
    type: string
    enum: ["usd", "eur", "gbp"]
    default: "usd"
  customer_id:
    type: string
    pattern: "^cus_[a-zA-Z0-9]+$"
    description: "Stripe customer ID"
```

A skill bank **SHOULD** validate the agent's substitutions against `args` before executing `command_template`. Validation failure **MUST** result in a non-zero exit and a clear stderr message.

### 2.7 The `applicable_when` schema

```yaml
applicable_when:
  os: ["linux", "macos"]                      # optional; if absent, all OSes
  shell_commands_present: ["jq", "curl"]      # all must be on PATH
  env_present: ["AUTH_TOKEN"]                 # all must be set
  exclude_if_env_absent: ["DEBUG_MODE"]       # skill not applicable if these unset
```

Skill banks **MAY** evaluate `applicable_when` at retrieval time and exclude inapplicable skills from `vec search` results.

### 2.8 The `chains` schema (advanced)

A skill can compose other skills:

```yaml
chains:
  - skill: "github.com/openai/agent-skills@v3.4.5/embeddings"
    args: { input: "$user_query" }
    output_var: "QEMB"
  - skill: "github.com/example/agent-skills@v1.0.0/vec-search"
    args: { collection: "docs", embedding: "$QEMB", k: 5 }
    output_var: "TOP_IDS"
  - skill: "github.com/example/agent-skills@v1.0.0/fetch-by-ids"
    args: { ids: "$TOP_IDS" }
```

Output variables from earlier steps are referenced as `$NAME` in later steps. The skill bank's executor resolves the chain by recursively fetching and executing referenced skills. Cycles **MUST** be detected and rejected.

## 3. The `/llms.txt` extension

A skill provider's web root **SHOULD** publish two discovery files:

### 3.1 `/llms.txt` — human + LLM friendly index

Format follows the [llmstxt.org](https://llmstxt.org/) convention with an `agent-skills` extension:

```
# example.com

> One-paragraph description of what example.com does.

## Skills

- [Charge a customer](https://cdn.jsdelivr.net/gh/example/agent-skills@v1.2.0/charge-customer/SKILL.md): create a Stripe charge
- [Refund a charge](https://cdn.jsdelivr.net/gh/example/agent-skills@v1.2.0/refund/SKILL.md): refund a previously-created charge

## Skills index (machine-readable)

- [skills-index.json](https://cdn.jsdelivr.net/gh/example/agent-skills@v1.2.0/skills-index.json)
```

This file is for humans browsing the site **and** for LLMs that want a quick survey of the provider's offerings.

### 3.2 `/skills-index.json` — machine-readable manifest

```json
{
  "schema_version": "1.0",
  "publisher": {
    "name": "Example Inc.",
    "domain": "example.com",
    "github_org": "example",
    "verified_at": "2026-04-28"
  },
  "default_source": {
    "type": "git",
    "repo": "github.com/example/agent-skills",
    "default_branch": "main",
    "latest_release": "v1.2.0",
    "latest_commit": "a1b2c3d4e5f67890abcdef1234567890abcdef12"
  },
  "skills": [
    {
      "id": "charge-customer",
      "version": "1.2.0",
      "url": "https://cdn.jsdelivr.net/gh/example/agent-skills@v1.2.0/charge-customer/SKILL.md",
      "summary": "Create a Stripe charge against a customer."
    },
    {
      "id": "refund",
      "version": "1.0.5",
      "url": "https://cdn.jsdelivr.net/gh/example/agent-skills@v1.2.0/refund/SKILL.md",
      "summary": "Refund a previously-created charge."
    }
  ]
}
```

Skill banks **SHOULD** prefer `skills-index.json` for programmatic ingest (one HTTP request gives the full catalog) and fall back to crawling `/llms.txt` if absent.

## 4. Skill bank behavior

### 4.1 Subscription model

A skill bank **MUST** track its subscriptions in a persistent collection. Each subscription record **MUST** capture:

```yaml
_id: "<arbitrary local id>"
source: "git" | "url"

# For source=git:
repo: "github.com/owner/repo"
version_pin: "v1.2.0"             # tag or "main" or specific SHA
sha_pin: "a1b2c3d4..."            # immutable SHA at last sync
cdn_base: "https://cdn.jsdelivr.net/gh/owner/repo@<sha>"

# For source=url:
url: "https://example.com/skills-index.json"

# Common:
auto_update: false                 # if true, sync follows the tag/branch; if false, only re-pinned SHA
last_synced: "<ISO-8601>"
verify_signature: true             # if true, refuse to ingest unsigned tags
trusted_keys: ["fingerprint1", "..."]
```

### 4.2 Embedding text composition

The skill bank **MUST** produce one embedding vector per skill. The text fed to the embedding model **SHOULD** be the concatenation of:

1. `title`
2. `use_when`
3. `description`
4. Each `examples[].intent`, joined by newline
5. `tags`, joined by space

Separators between sections **SHOULD** be a single `. ` (period + space). Skill banks **MUST** use the **same** embedding model for the indexed skills and the agent's query embeddings — otherwise vector-space mismatch destroys retrieval quality.

The choice of embedding model is a per-skill-bank decision. **Skills do NOT publish vectors.** The skill bank generates them at ingest time using its locally configured model.

### 4.3 Retrieval contract

When the agent submits a query, the skill bank **MUST**:

1. Embed the query using the same model used for skills.
2. Run nearest-neighbor search against `vec skills`, returning top-K (default K=10).
3. Optionally re-rank by `usage_count`, `avg_rating`, `provenance.publisher_verified`, or other secondary signals.
4. Optionally filter by `applicable_when` against the current shell environment.
5. Return the top-N (default N=3) skill identities to the agent.

The agent then **MAY** fetch full metadata via `db skills find` for the chosen skill.

### 4.4 Execution contract

When the agent decides to execute a skill:

1. The agent **MUST** provide values for every required `args` placeholder.
2. The skill bank **MUST** validate values against the `args` schema.
3. The skill bank **MUST** verify `applicable_when` (if declared).
4. The skill bank **MUST** substitute placeholders into `command_template` and execute.
5. The skill bank **SHOULD** capture stdout, stderr, exit code, and elapsed time for the audit log.

Substitution **MUST** properly escape values to prevent command injection. The reference rule: substitutions are inserted as **shell-quoted strings** (single-quoted, with embedded single quotes encoded as `'\''`). A skill that needs an unquoted substitution (rare) **MUST** declare `args.<name>.unquoted: true` and the skill bank **MUST** reject any substitution containing shell metacharacters before inserting raw.

### 4.5 Feedback contract

A conformant skill bank **SHOULD** support:

```yaml
db skill_audit insert:
  skill_id: "<full identity>"
  intent: "<the original query>"
  args: { ... }                  # substituted values (with sensitive ones redacted)
  exit_code: 0
  elapsed_ms: 234
  rating: 5                      # optional, agent-supplied or user-supplied
  notes: "..."                   # optional
  timestamp: "<ISO-8601>"
```

Aggregating `skill_audit` over time produces signals that bias retrieval (`avg_rating`, `success_rate`) without requiring central infrastructure.

## 5. Identity, signing, and trust

### 5.1 Provenance verification

Skill banks **MAY** require provenance verification before ingesting a skill:

- **Level 0 (no verification)**: any URL is accepted. Suitable for personal/dev use.
- **Level 1 (TLS only)**: HTTPS chain validates the source domain. Default for most consumers.
- **Level 2 (commit pinning)**: skill identity **MUST** include a SHA. Tag-only references are rejected.
- **Level 3 (signed tags)**: the git tag at `version_pin` **MUST** be signed by a key in `trusted_keys`. Verification follows the standard `git verify-tag` semantics.
- **Level 4 (Sigstore / Rekor)**: signature **MUST** be present in the public Sigstore transparency log, and not revoked.

The skill bank operator chooses the level. Per-subscription overrides **SHOULD** be supported.

### 5.2 Trusted-key management

Skill banks **SHOULD** maintain a `db trusted_keys` collection:

```yaml
_id: "stripe-llm.gpg"
fingerprint: "B5A4 9C28 D9F1 ..."
public_key: "<armored>"
imported_at: "<ISO-8601>"
imported_from: "https://stripe.com/.well-known/agent-skills-key.asc"
```

Skill banks **SHOULD NOT** auto-import keys; addition is a deliberate user action.

## 6. Versioning

### 6.1 Skill version semantics

Skill version follows [semver](https://semver.org/):

- **PATCH** (`1.0.0 → 1.0.1`): bug fix in `command_template` (e.g., escape an arg correctly), description clarification, examples added. **No** changes to `command_template` semantics or `args` schema.
- **MINOR** (`1.0.0 → 1.1.0`): additive features. New optional `args`. New `examples`. New `tags`. Unchanged behavior for existing invocations.
- **MAJOR** (`1.0.0 → 2.0.0`): breaking change. `command_template` now takes different args, semantics changed, removed args, etc. **MUST** declare `deprecates` pointing at the predecessor major version.

### 6.2 Version pinning policies

| Pin style | Use | Risk |
|---|---|---|
| `latest` | Personal exploration | High — silent breakage |
| `main` (branch) | Following development | Medium — may break |
| `vX.Y.Z` (tag) | Production | Low — only if provider re-tags |
| `<40-char SHA>` | Strict production | Zero — bit-immutable |

Skill banks running in `strict_mode` **MUST** reject `latest`. They **SHOULD** warn on `main` / branch pins.

## 7. Sync protocol

### 7.1 Initial subscription

```
1. User specifies repo + version pin.
2. Skill bank resolves version pin to a SHA via:
     GET https://api.github.com/repos/<owner>/<repo>/git/refs/tags/<pin>
3. Skill bank verifies signature (if required) at the SHA.
4. Skill bank fetches skills-index.json:
     GET https://cdn.jsdelivr.net/gh/<owner>/<repo>@<sha>/skills-index.json
5. For each skill in the index:
     a. Fetch <cdn_base>/<skill_id>/SKILL.md
     b. Parse YAML frontmatter; validate against schema_version
     c. Compute embedding text (§4.2); call embedding model
     d. Upsert into db skills (keyed by full identity); upsert vec embedding
6. Persist subscription record with sha_pin = resolved SHA.
```

### 7.2 Re-sync

Periodic re-sync (cron, manual, or watch-driven):

```
1. For each subscription with auto_update=true:
     a. Resolve version pin → new SHA.
     b. If new SHA == sha_pin, skip.
     c. Otherwise, present diff to operator (changed skills, removed skills, added skills).
     d. On approval, repeat ingest from §7.1 step 4.
2. For subscriptions with auto_update=false:
     a. Notify operator of newer version, but DO NOT auto-apply.
```

A skill that no longer exists at the new SHA **SHOULD** be marked `removed: true` in `db skills` rather than deleted, preserving audit history.

### 7.3 Network requirements

- TLS for all transport.
- Skill banks **SHOULD** respect the CDN's `Cache-Control` headers.
- Skill banks **MUST NOT** transmit user query text or feedback to skill providers (privacy invariant).

## 8. Privacy invariants

The spec is designed to provide several privacy properties **by construction**:

1. **Credentials never enter the LLM context.** `command_template` references `$ENV_VAR`; the shell substitutes at exec time.
2. **Query text never leaves the local machine** except to the user's chosen embedding model API.
3. **Skill providers receive zero telemetry** about who uses their skills, when, or how. Audit data lives only in the user's `db skill_audit`.
4. **No central registry sees usage.** Discovery is web-native; ingestion is HTTP fetches with no identifying headers beyond user-agent.

Skill banks **MUST NOT** add telemetry, beacons, or upstream reporting that violate these invariants.

## 9. Reserved field names

Future spec versions may add fields. To avoid collisions, the following names are **reserved** and **MUST NOT** be used by skill authors for ad-hoc data:

- Any field starting with `provenance.`
- `_id`, `_rev`, `_meta`
- `inserted_at`, `updated_at`, `last_synced`, `usage_count`, `avg_rating`, `success_rate`, `removed`, `deprecated`

Custom data **MUST** use a `metadata` sub-object:

```yaml
metadata:
  vendor_specific_field: "..."
```

## 10. Conformance levels

A skill bank or skill provider **MAY** declare its conformance level:

- **A1 (Author-conformant)**: produces `SKILL.md` files matching §2's required + recommended fields. Does not necessarily publish `/llms.txt`.
- **A2 (Publisher-conformant)**: A1 + `/llms.txt` + `/skills-index.json` + git-sourced provenance.
- **A3 (Verified publisher)**: A2 + signed tags + a published key.
- **B1 (Consumer-conformant)**: skill bank that ingests A1+ skills, performs embedding, supports retrieval and execution per §4.
- **B2 (Strict consumer)**: B1 + provenance level ≥ 2 + audit log + applicable_when filtering.

## Open questions

- How do skills handle binary artifacts (e.g., a Python helper script)? Probably out of scope for v1.0; skills should rely on commands already in the user's shell.
- Should there be an "uninstall" semantic for skills, separate from "deprecation"? Currently no — deprecation + skill bank pruning is sufficient.
- Internationalization: are skills in non-English use_when fields a separate skill or a translation? Likely separate skills with `tags: ["lang:es"]`.

See [`ROADMAP.md`](./ROADMAP.md) for the active discussion.
