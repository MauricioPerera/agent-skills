# agent-skills — Specification

**Version**: 0.1.1 (draft)
**Status**: Open for comment. Schema and protocol are subject to change before v1.0.0.

This document defines a **format and a protocol**. It does not define a runtime, a storage backend, or a UI. Conformant implementations MAY be built atop any sufficient infrastructure (filesystem + vector index + shell). One reference runtime is described in [`IMPLEMENTATION.md`](./IMPLEMENTATION.md), but the spec itself is implementation-agnostic.

## 0. Notational conventions

The key words **MUST**, **MUST NOT**, **SHOULD**, **SHOULD NOT**, and **MAY** are to be interpreted as in [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119).

A **skill bank** is an implementation that subscribes to skill providers, ingests their skills (parses, embeds, indexes), and exposes them to an agent.

A **skill provider** is whoever publishes one or more skills.

A **skill consumer** is the runtime that executes a skill on behalf of a user.

A **skill author** is whoever writes a `SKILL.md` file. The author and provider may be the same entity or different (e.g., a community member submits a PR to a provider's repository).

The shell semantics targeted throughout this document are **POSIX shell, specifically bash 4.x or higher** (see §2.9). Implementations on non-POSIX shells (Windows `cmd.exe`, PowerShell, fish) require translation outside the scope of this spec.

## 1. Skill identity

Every skill has a globally unique identity:

```
<source>@<ref>/<path>
```

where:

- `<source>` is one of:
  - `<host>/<owner>/<repo>` — git-hosted skill. `<host>` MUST be a DNS name (`github.com`, `gitlab.com`, `git.example.com`, etc.).
  - `<host>` (alone) — server-hosted skill at the host's `/skills/` namespace (see §3.3).
- `<ref>` is one of:
  - A git commit hash, ≥ 40 hex characters lowercase. (40 chars for SHA-1, 64 chars for SHA-256.) **Recommended for production — immutable.**
  - A git tag, format `^[a-zA-Z0-9_.+-]+$` (matches typical semver tags like `v1.2.0`, `1.0.0-beta.1`). Mutable apuntador to a hash.
  - The literal `latest` (NOT recommended — see §6.2).
- `<path>` is the slash-separated path within the source to the directory containing `SKILL.md`. Path components MUST match `^[a-zA-Z0-9_-]+$`.

The first slash after `@<ref>` separates source from path; subsequent slashes are path separators.

**Examples** (all well-formed):

```
github.com/stripe/agent-skills@a1b2c3d4e5f67890abcdef1234567890abcdef12/charge-customer
gitlab.com/some-org/skills@v1.2.0/send-email
git.example.com/team/internal-skills@a1b2c3d4e5f67890abcdef1234567890abcdef12abcdef1234567890abcdef12abc/deploy
img.automators.work@latest/placeholder
```

### 1.1 Identity → URL derivation

The skill bank MUST be able to derive a fetchable URL from an identity. Two methods are supported:

**Method A — declared by the provider via `skills-index.json`** (preferred, see §3.2). The index file maps each skill `id` to a full URL.

**Method B — built-in templates for known hosts**, applied when no index is available:

| Host | URL template (filled with `{owner}`, `{repo}`, `{ref}`, `{path}`) |
|---|---|
| `github.com` | `https://cdn.jsdelivr.net/gh/{owner}/{repo}@{ref}/{path}/SKILL.md` |
| `github.com` (alt) | `https://raw.githubusercontent.com/{owner}/{repo}/{ref}/{path}/SKILL.md` |
| `gitlab.com` | `https://gitlab.com/{owner}/{repo}/-/raw/{ref}/{path}/SKILL.md` |
| `bitbucket.org` | `https://bitbucket.org/{owner}/{repo}/raw/{ref}/{path}/SKILL.md` |
| Self-hosted git | (no built-in template; provider MUST supply via `skills-index.json`) |
| Server-hosted (host alone) | `https://{host}/skills/{path}/SKILL.md` |

When multiple URL templates are available for a host, banks SHOULD attempt them in the order listed and accept the first 200 OK response.

A bank that does not recognize a host's URL template MUST refuse to ingest from that source unless the provider's `skills-index.json` declares an explicit URL.

## 2. The `SKILL.md` file

A `SKILL.md` file is a UTF-8 plain text file with **YAML frontmatter** and **markdown body**. The frontmatter is machine-readable; the body is human-readable (and OPTIONAL).

YAML version: **1.2** (per [yaml.org](https://yaml.org/spec/1.2.2/)). Implementations that only support YAML 1.1 MUST refuse to ingest skills containing YAML 1.2-specific syntax.

Markdown variant: **CommonMark** ([commonmark.org](https://commonmark.org/)). GitHub Flavored Markdown extensions (tables, task lists) MAY be used in the body but MUST NOT be relied on by the bank for parsing.

### 2.1 File structure

```yaml
---
# Required (§2.2)
schema_version: "0.1"
id: "<short-id>"
version: "<semver>"
title: "<human-readable name>"
description: "<one-paragraph what-it-does>"
use_when: "<one-sentence when-an-agent-should-pick-this>"
command_template: "<bash command, with {placeholder} args>"

# Recommended (§2.3)
license: "MIT"
author: { name: "...", url: "..." }
homepage: "..."
category: "..."
tags: ["...", "..."]
args: { ... }
examples: [ ... ]

# Optional (§2.4)
shell: "bash"
idempotent: true
required_commands: ["..."]
required_env: ["..."]
network: ["..."]
applicable_when: { ... }
deprecates: ["..."]
related: ["..."]
chains: [ ... ]

# Reserved — see §9 (NOT to be filled by skill author)
# provenance is computed at ingest time from git metadata; NOT placed in the file.
---

# Title

[Optional human-readable prose. Markdown body.]
```

### 2.2 Required fields

| Field | Type | Constraints |
|---|---|---|
| `schema_version` | string | Spec version this skill targets. Currently `"0.1"`. |
| `id` | string | Stable identifier in publisher namespace. MUST match `^[a-z][a-z0-9_-]{0,63}$`. |
| `version` | string | [Semver](https://semver.org/) MAJOR.MINOR.PATCH. |
| `title` | string | Human name, ≤ 80 UTF-8 bytes. |
| `description` | string | What the skill does. ≤ 1000 UTF-8 bytes. Indexed (§4.2). |
| `use_when` | string | One-sentence when-to-pick. ≤ 500 UTF-8 bytes. **Primary signal for retrieval.** Indexed (§4.2). |
| `command_template` | string | Shell command with `{placeholder}` substitutions (§2.6). |

### 2.3 Recommended fields

| Field | Type | Notes |
|---|---|---|
| `license` | SPDX string | `"MIT"`, `"Apache-2.0"`, etc. |
| `author` | object | `{ name: string, url?: string, email?: string }` |
| `homepage` | URL | Where to learn more |
| `category` | string | Coarse-grained category for filtering |
| `tags` | array of strings | Fine-grained labels |
| `args` | object | Schema for `{placeholder}` substitutions (§2.6) |
| `examples` | array of objects | Each `{ intent: string, command: string, expected_output?: string }`. Indexed (§4.2). |

### 2.4 Optional fields

| Field | Type | Description |
|---|---|---|
| `shell` | string | `"bash"` (default) or other POSIX-compliant. Banks running on non-matching shells MAY refuse the skill. |
| `idempotent` | boolean | If `true`, re-running the skill with same args is safe. Default: `false`. Used by chain executor for retry decisions (§2.8). |
| `required_commands` | string[] | Commands the template invokes (`["jq", "curl"]`). Banks MAY filter out skills whose required commands are unavailable. |
| `required_env` | string[] | Env var **names** the skill expects. **Values are never published.** |
| `network` | string[] | URL-prefix allowlist (§2.10). |
| `applicable_when` | object | Conditions for the skill to be applicable (§2.7). |
| `deprecates` | string[] | Identities of skills this version replaces. Banks SHOULD mark superseded skills as `deprecated: true` upon syncing. |
| `migration_notes` | string | Markdown prose explaining migration. |
| `related` | string[] | Cross-references to related skills. |
| `chains` | array of objects | Multi-step composition (§2.8). |

### 2.5 Provenance fields (computed at ingest, NOT in the file)

Provenance is **derived by the skill bank at ingest time** from the source's metadata (git commit info, HTTP headers, signature data). It is **NOT** declared by the skill author in the `SKILL.md` file — doing so would create a chicken-and-egg problem (the SHA depends on the file content; the file would have to know its own SHA pre-commit).

The skill bank populates these fields in its local index:

```yaml
provenance:
  source_type: "git" | "url"
  source: "github.com/stripe/agent-skills"     # the source identifier
  ref_resolved_to: "a1b2c3d4..."               # the commit hash this ingest pinned to
  ref_requested: "v1.2.0"                      # what the user originally asked for
  fetched_at: "2026-04-28T12:34:56Z"
  signature_status: "unsigned" | "valid" | "invalid" | "unverified"
  signed_by: "<key fingerprint>"               # if signature_status == "valid"
  publisher_verified: true | false             # set by user-trust policy
```

When a bank presents a skill to an agent (via query results or metadata fetch), it MUST include `provenance` so the agent (or user) can filter / weight by it.

### 2.6 The `args` schema

When `command_template` contains `{name}` placeholders, `args` SHOULD declare each:

```yaml
args:
  amount:
    type: integer            # see types below
    description: "amount in cents"
    range: [1, 100000]       # closed interval, for numeric types
    default: null            # optional; if present, args is optional at call time
  currency:
    type: string
    enum: ["usd", "eur", "gbp"]
    default: "usd"
  customer_id:
    type: string
    pattern: "^cus_[a-zA-Z0-9]+$"
    description: "Stripe customer ID"
  metadata:
    type: object             # arbitrary JSON object
    description: "extra fields stored alongside the charge"
    default: {}
  tags:
    type: array              # array of any sub-type
    items:
      type: string
    description: "list of category tags"
    default: []
```

Valid `type` values: `string`, `integer`, `number`, `boolean`, `array`, `object`.

**Array type** MUST declare `items` (a recursive arg schema). **Object type** MAY declare `properties` (a per-field schema map), or remain unconstrained.

**Quoting policy**: at call time, a substituted value is inserted into `command_template` as a **single shell argument**. The bank automatically wraps strings in single quotes with embedded single quotes encoded as `'\''`. Numeric values are inserted unquoted (after pattern validation rejects shell metacharacters). Booleans become the literal strings `true` / `false`. Arrays and objects are inserted as JSON-encoded strings.

**Skill authors MUST place `{placeholder}` in argument position**, never inside literal quotes:

✅ `curl -d amount={amount}` (placeholder is its own arg after `-d`)
✅ `curl -d "$(printf 'amount=%s' {amount})"` (substitution then quoting in shell)
❌ `curl -d "amount={amount}"` (placeholder inside literal double quotes — UNSAFE)

The bank's substitution does NOT attempt to "fix" templates that violate this rule; values inside literal quotes are inserted verbatim and may break or open injection. Conformant banks SHOULD detect and refuse such templates at ingest time.

**Unquoted bypass** (rarely needed):

```yaml
args:
  raw_url:
    type: string
    pattern: "^https://[a-zA-Z0-9._/-]+$"   # MANDATORY when unquoted
    unquoted: true
```

When `unquoted: true`, the bank inserts the value without single-quoting. The skill author MUST declare a strict `pattern` that rejects shell metacharacters (`;`, `&`, `|`, `$`, backtick, `(`, `)`, `<`, `>`, `*`, `?`, `[`, `]`, `\`, `"`, `'`, `{`, `}`, `#`, `~`, space, tab, newline). Banks MUST refuse `unquoted: true` args without a `pattern`, or with a pattern that allows any of the forbidden characters.

### 2.7 The `applicable_when` schema

```yaml
applicable_when:
  os: ["linux", "macos", "windows"]              # any-of
  arch: ["x86_64", "arm64"]                      # any-of
  shell_commands_present: ["jq", "curl"]         # all-of (every command MUST be on PATH)
  env_present: ["STRIPE_KEY"]                    # all-of
  env_absent: ["DRY_RUN"]                        # none-of
```

If multiple constraints are declared, all MUST be satisfied. Banks MAY evaluate `applicable_when` at retrieval time and exclude inapplicable skills from results.

OS values are lowercase. The set is open-ended; common values are `linux`, `macos`, `windows`, `freebsd`. Banks unable to determine the host OS MUST treat all skills as potentially applicable.

### 2.8 The `chains` schema

A skill can compose other skills sequentially:

```yaml
chains:
  - skill: "github.com/openai/agent-skills@a1b2c3d4.../embeddings"
    args:
      input: "{user_query}"             # references the parent skill's args
    output_var: "QEMB"                  # captures stdout of this step
  - skill: "github.com/example/agent-skills@a1b2c3d4.../vec-search"
    args:
      collection: "docs"
      embedding: "{QEMB}"               # references prior step's output
      k: 5
    output_var: "TOP_IDS"
  - skill: "github.com/example/agent-skills@a1b2c3d4.../fetch-by-ids"
    args:
      ids: "{TOP_IDS}"
```

**Placeholder syntax in chains**: `{name}` (curly braces). This is the same as `command_template`. There is no `$NAME` syntax inside `args` values; that was a draft inconsistency.

**Output capture**: `output_var` makes the step's stdout available to subsequent steps as `{NAME}`. If a step produces JSON, downstream steps may consume it raw or after `jq` processing within their own template.

**Cycle detection**: chains MUST be acyclic. The bank's executor MUST detect cycles (same skill identity invoked recursively in the same chain) and abort with a validation error.

**Retry semantics**: by default, if a step fails, the chain aborts. If `idempotent: true` is declared on the failing step's skill, the executor MAY retry up to a bank-configured limit before aborting. Non-idempotent steps MUST NOT be retried automatically.

### 2.9 Shell semantics

Skills target **bash 4.0+** unless `shell:` declares otherwise. Authors MAY rely on:

- Standard POSIX shell features (variables, pipes, redirections, command substitution).
- bashisms commonly available: `[[ ... ]]`, arrays, `${var:default}`, process substitution `<(...)`.
- Standard shell utilities (POSIX + commonly installed: `jq`, `curl`, `grep`, `awk`, etc., declared via `required_commands`).

Authors MUST NOT rely on:

- `cmd.exe` or PowerShell features.
- Fish shell or zsh-specific syntax.
- GNU extensions to standard utilities (declare via `required_commands` if needed; consumers running BSD utilities may fail).

Banks running on non-bash hosts MAY:

- Refuse skills with no explicit `shell:` field (assume bash, can't guarantee).
- Translate the template to the host shell (out of spec scope; implementation-defined).

### 2.10 The `network` allowlist

```yaml
network:
  - "https://api.stripe.com/v1/charges"
  - "https://api.stripe.com/v1/customers/*"
  - "https://*.example.com/"
```

**Matching rules**:

- Each entry is a **URL prefix or wildcard prefix**, not a full glob or regex.
- A prefix matches if the request URL **starts with** the entry's prefix (after normalizing trailing slashes).
- A single trailing `*` matches any continuation (any path, any query string).
- A `*` in the host position matches one DNS label (e.g., `*.example.com` matches `api.example.com` but not `a.b.example.com`).
- `*` is NOT supported elsewhere (no `*://`, no `*?param`).

**Banks operating in sandbox mode** MUST:

- Block any HTTP request whose URL does not match an entry in `network`.
- Treat absence of `network` as an empty allowlist (no network at all).
- Treat `network: ["*"]` as a literal entry (one wildcard URL); to permit any URL, the bank's policy SHOULD require an explicit unsafe-flag, not allow it via the spec.

Banks not in sandbox mode SHOULD still log requests outside the allowlist as warnings.

## 3. Discovery files

### 3.1 `/llms.txt` — top-level provider index

A skill provider's web root SHOULD publish `/llms.txt` following the [llmstxt.org](https://llmstxt.org/) baseline:

```
# example.com

> One-paragraph description of the provider.

## Skills

- [Skill name](URL): brief description
- ...

## Skills index (machine-readable)

- [skills-index.json](URL)
```

Every URL referenced in the `## Skills` section SHOULD be a fetchable `SKILL.md`. Every URL referenced in the `## Skills index (machine-readable)` section MUST be a fetchable `skills-index.json`.

This format is a strict superset of llms.txt — banks built for general llms.txt (not agent-skills-aware) still get a useful overview.

### 3.2 `/skills-index.json` — machine-readable manifest

```json
{
  "schema_version": "0.1",
  "publisher": {
    "name": "Example Inc.",
    "domain": "example.com",
    "github_org": "example",
    "homepage": "https://example.com",
    "agent_skills_key_url": "https://example.com/.well-known/agent-skills-key.asc"
  },
  "default_source": {
    "type": "git",
    "repo": "github.com/example/agent-skills",
    "default_branch": "main",
    "latest_release": "v1.2.0",
    "latest_commit": "a1b2c3d4e5f67890abcdef1234567890abcdef12"
  },
  "url_template": "https://cdn.jsdelivr.net/gh/example/agent-skills@{ref}/{path}/SKILL.md",
  "skills": [
    {
      "id": "charge-customer",
      "version": "1.2.0",
      "url": "https://cdn.jsdelivr.net/gh/example/agent-skills@v1.2.0/charge-customer/SKILL.md",
      "summary": "Create a Stripe charge against a customer."
    }
  ]
}
```

The `url_template` field is OPTIONAL. When present, banks SHOULD use it instead of built-in host templates (§1.1). This lets self-hosted git providers declare custom CDN paths.

Banks MUST prefer `skills-index.json` for programmatic ingest (one HTTP request) and fall back to crawling `/llms.txt` if absent.

### 3.3 Server-hosted skills (no git)

A provider MAY publish skills directly from their web server, without a git repository:

- `/llms.txt` lists each skill's URL.
- Skills live at `https://<host>/skills/<id>/SKILL.md`.
- Identity is `<host>@latest/<id>`.

This mode is **simpler to publish** but provides **weaker guarantees**:

- No commit history.
- No SHA pinning (the only `ref` is `latest`).
- No tag-based versioning.
- Banks SHOULD only accept this mode at provenance Level 0 or 1 (§5.1).

For production-grade skills, providers SHOULD host a git repository even if they only have one skill.

## 4. Skill bank behavior

### 4.1 Subscription model

A skill bank MUST persist subscriptions in some queryable storage. Each subscription record MUST capture (at minimum):

```yaml
id: "<arbitrary local identifier>"
source_type: "git" | "url"

# git subscriptions:
repo: "github.com/owner/repo"
ref_requested: "v1.2.0"               # tag, branch, or hash the user specified
ref_resolved: "a1b2c3d4..."           # commit hash at last successful sync
url_template: "..."                   # optional, overrides built-in (§3.2)

# url subscriptions:
index_url: "https://example.com/skills-index.json"

# common:
auto_update: false                    # if true, sync follows ref; if false, manual approval required
last_synced: "2026-04-28T..."
verify_signature: true                # if true, refuse to ingest unsigned tags (§5)
trusted_keys: ["fingerprint1", "..."]
```

Storage of the subscription record is implementation-defined. The reference implementation in [`IMPLEMENTATION.md`](./IMPLEMENTATION.md) uses just-bash-data's `db` collection; other banks may use SQLite, a flat file, or any equivalent.

### 4.2 Embedding text composition

The bank MUST produce one embedding vector per skill. The text fed to the embedding model MUST be the concatenation, in order, of:

1. `title`
2. `use_when`
3. `description`
4. Each entry of `examples[].intent`, joined by `\n`.
5. `tags`, joined by space.

Sections are joined by `". "` (period + space). The full string is fed as a single input to the embedding model.

**Truncation policy**: if the composed text exceeds the embedding model's max input length, the bank MUST truncate to fit by dropping later sections in this priority (LIFO):

1. Drop `tags` section.
2. Drop `examples[].intent` section, in reverse order (drop the last example first).
3. Drop trailing characters of `description` (preserving `title` + `use_when` always).

Banks MUST NOT silently produce embeddings of truncated input without recording the fact: `provenance.embedding_truncated: true` SHOULD be set in the index.

Banks MUST NOT use **different** embedding models for different skills within the same index, nor for the indexed skills versus the agent's queries. Mixed-model search is undefined.

The choice of embedding model is a per-bank deployment decision. Skills MUST NOT publish vectors. (See [`DESIGN.md`](./DESIGN.md) D4.)

### 4.3 Retrieval contract

Given a query, the bank MUST:

1. Embed the query with the same model used to index skills.
2. Run nearest-neighbor search over the index, returning top-`K` (default `K=10`).
3. Optionally re-rank by `usage_count`, `avg_rating`, `provenance.publisher_verified`, or other signals.
4. Optionally filter by `applicable_when` against the host environment.
5. Return the top-`N` (default `N=3`) skill identities + selected metadata.

Banks SHOULD support a query option to bypass `applicable_when` filtering for debugging.

### 4.4 Execution contract

Given a skill identity and arg values, the bank MUST:

1. Look up the skill in the index. If absent, exit with "not found" error.
2. Validate every arg value against the `args` schema:
   - Type check.
   - Range / enum / pattern check.
   - Reject missing required args (those without `default`).
3. Verify `applicable_when` (if declared). On mismatch, exit with "not applicable" error.
4. Substitute placeholders into `command_template`:
   - For each `{name}`, replace with the substituted form (§2.6 quoting policy).
   - Substitute occurs only at exact match `{name}` boundaries; literal `{` followed by something else is preserved.
5. Execute via the configured shell (`bash` by default).
6. Capture `stdout`, `stderr`, exit code, elapsed time.

A bank operating in sandbox mode (§2.10) MUST:

- Intercept network calls and check against `network`.
- Restrict filesystem to a per-skill scratch directory (`$AGENT_SCRATCH`).
- Prevent process spawning beyond `required_commands`.
- Block access to env vars not in `required_env`.

Sandboxing is an OPTIONAL bank feature; non-sandboxed banks MUST document the trust boundary they offer.

### 4.5 Audit contract

A conformant bank SHOULD record per-execution:

```yaml
skill_id: "<full identity>"
intent: "<original query, optional>"
args: { ... }                     # substituted values, with sensitive ones redacted
exit_code: 0
elapsed_ms: 234
rating: 5                         # optional, agent-supplied or user-supplied
notes: "..."                      # optional
timestamp: "<ISO-8601>"
```

"Sensitive values" are those whose `args.<name>.sensitive: true` is declared. Banks MUST redact these in audit records (e.g., replace with `"<redacted>"`).

Audit data lives only locally on the consumer's machine. Banks MUST NOT transmit audit data to skill providers (privacy invariant P3, §8).

## 5. Identity, signing, and trust

### 5.1 Provenance verification levels

Banks MAY require one of these levels per subscription:

- **Level 0 (no verification)**: any URL accepted. Suitable for development.
- **Level 1 (TLS only)**: HTTPS chain validates the source domain. Default for most consumers.
- **Level 2 (commit pinning)**: `<ref>` MUST be a commit hash, not a tag. Tag pins are resolved to a hash but stored only as the hash.
- **Level 3 (signed tags)**: the git tag MUST be signed by a key in `trusted_keys`. Verified via `git verify-tag`.
- **Level 4 (Sigstore + Rekor)**: signature MUST be present in the public Sigstore transparency log and not revoked.

Level 0 banks accept any source. Level 2+ banks REJECT subscriptions to server-hosted skills (§3.3) since those have no commit hashes.

### 5.2 Trusted-key management

Banks SHOULD support trusted-key configuration. Implementation-specific format; an example is given in [`IMPLEMENTATION.md`](./IMPLEMENTATION.md).

Banks MUST NOT auto-import keys. Adding a key is a deliberate user action. Banks SHOULD verify the key fingerprint matches a value the user explicitly provides (out-of-band trust establishment).

## 6. Versioning

### 6.1 Skill version semantics

Skills follow [semver](https://semver.org/):

- **PATCH** (`1.0.0 → 1.0.1`): prose / description / examples changes. NO change to `command_template`'s observable behavior, NO change to `args` schema.
- **MINOR** (`1.0.0 → 1.1.0`): additive. New optional `args`. New examples. Tighter `pattern` validation that doesn't change rejection of historically-valid values. New `tags`. Backward-compatible.
- **MAJOR** (`1.0.0 → 2.0.0`): breaking. `command_template` semantics changed. New required `args`. Removed args. Looser `pattern` (could expose injection). MUST declare `deprecates` pointing at the predecessor version.

"Observable behavior" means: given the same args, the same output is produced (idempotent skills) or the same side effect occurs (non-idempotent skills).

### 6.2 Pin policies

| Pin type | Use | Risk |
|---|---|---|
| `latest` | Personal exploration only | High — provider can swap content |
| Branch (`main`) | Following development | Medium — may break |
| Tag (`v1.2.0`) | Production | Low — only if provider re-tags |
| Hash (40+ hex) | Strict production | Zero — bit-immutable |

Banks operating in `strict_mode` MUST reject `latest`. They SHOULD warn on branch pins.

### 6.3 Deprecation cycle

When a skill is deprecated:

1. The deprecating version's `deprecates` lists the old identity.
2. Banks at next sync mark the old identity `deprecated: true` in the index.
3. Banks SHOULD continue to return deprecated skills in queries with a clear marker, allowing graceful migration.
4. After the bank operator confirms migration, they MAY explicitly remove deprecated skills.

The spec does NOT define a fixed deprecation grace period. Each provider sets policy.

## 7. Sync protocol

### 7.1 Initial subscription

```
1. User specifies repo + ref.
2. Bank resolves ref to a commit hash. Resolution method:
   a. For known git hosts: HTTP API (e.g., GitHub, GitLab REST APIs).
   b. For unknown hosts: git clone --depth 1 --branch <ref>; HEAD is the hash.
   c. For server-hosted (no git): ref is implicitly "latest"; the response's
      Last-Modified or ETag MAY serve as a pseudo-hash.
3. Bank verifies signature at the resolved hash (per provenance level §5.1).
4. Bank fetches skills-index.json (URL derivation per §1.1):
   GET <cdn-or-source>@<resolved-hash>/skills-index.json
5. For each skill in the index:
   a. Fetch SKILL.md via the URL in skills-index.json (or built-in template).
   b. Validate YAML frontmatter against the schema (schema_version match, all
      required fields present, types correct). On failure, log and skip; do
      NOT abort the whole sync.
   c. Compose embedding text per §4.2; embed via the configured model.
   d. Upsert into the index, keyed by full identity. Compute provenance
      (§2.5) from the resolved hash + signature data.
6. Persist subscription record with ref_resolved = hash from step 2.
```

### 7.2 Re-sync

```
For each subscription with auto_update == true:
  1. Resolve ref → new_hash.
  2. If new_hash == ref_resolved, skip (no change).
  3. Fetch the new skills-index.json.
  4. Compute the diff against the indexed state:
     - Added skills (new IDs).
     - Removed skills (IDs no longer in index).
     - Modified skills (same ID, new version or content).
  5. Banks SHOULD present the diff to the operator before applying.
  6. On approval, repeat ingest from §7.1 step 5 for added + modified skills.
  7. Mark removed skills as removed: true (do NOT delete; preserve audit).

For subscriptions with auto_update == false:
  Only steps 1-4. Notify operator. Do NOT apply.
```

### 7.3 Diff structure

The bank MUST surface enough information for the operator to make an informed decision. The diff structure SHOULD include:

- For each modified skill: side-by-side `command_template` before/after, args schema diff, `network` allowlist diff, list of changed sections.
- For each added skill: full `SKILL.md` rendered.
- For each removed skill: previous `SKILL.md` content + a note about why (deprecation, deletion).

### 7.4 Comparing against baseline

Banks SHOULD support a "compare against last-approved" mode. Without it, an operator approving each incremental update can miss accumulated drift. With it, every diff is shown relative to the last manually-approved state, regardless of how many auto-syncs happened in between.

## 8. Privacy invariants

The following are **structural** properties the spec is designed to preserve:

- **P1: Credentials never enter LLM context.** `command_template` references env variables; the shell substitutes values at exec time, after the LLM has emitted the command.
- **P2: Skill content is content-addressable.** Identities pinned to commit hashes are immutable. Implementations MUST NOT silently rewrite indexed skills outside of an explicit re-sync.
- **P3: No first-party telemetry to skill providers.** The bank does not POST usage data to providers. The bank's audit log lives only locally. (Providers MAY observe some traffic via CDNs / git hosts at the transport layer; this is outside the bank's control.)
- **P4: Provider downtime tolerance.** Once a skill is ingested with a hash pin, the bank MUST be able to execute it from local cache without re-fetching.
- **P5: Audit trail is structurally complete.** Every skill change is a git commit (for git-sourced skills) or a documented HTTP fetch with timestamp (for url-sourced). Implementations MUST preserve enough metadata to reconstruct the change history.

Implementations MUST NOT add features that violate these invariants without an opt-in user action.

## 9. Reserved field names

Future spec versions may add fields. To prevent collisions, the following are reserved:

- Anything under `provenance.*` (§2.5).
- `deprecated`, `removed`, `inserted_at`, `updated_at`, `last_synced`, `usage_count`, `avg_rating`, `success_rate`, `embedding_truncated` (these are bank-managed fields, not author-declared).

Vendor-specific fields MUST go in `metadata`:

```yaml
metadata:
  vendor_specific_field: "..."
```

## 10. Conformance levels

### 10.1 Skill / publisher levels

- **A1 (Author-conformant)**: produces `SKILL.md` with all required + recommended fields. NOT required to publish `/llms.txt`.
- **A2 (Publisher-conformant)**: A1 + `/llms.txt` + `/skills-index.json` + git-sourced provenance.
- **A3 (Verified publisher)**: A2 + signed tags + a published key (e.g., `.well-known/agent-skills-key.asc`).

### 10.2 Skill bank levels

- **B1 (Consumer-conformant)**: ingests A1+ skills (any source type), performs embedding per §4.2, supports retrieval and execution per §4.
- **B2 (Strict consumer)**: B1 + provenance Level 2+ enforced + sandbox mode + audit log + `applicable_when` filtering.

### 10.3 Forward compatibility

A bank declaring schema_version `0.1` ingesting a skill with schema_version `0.2` SHOULD attempt to parse known fields and ignore unknown ones. If unknown fields are required for correct execution (e.g., a hypothetical `pre_check`), the bank MUST refuse to execute and surface the schema mismatch.

## 11. Spec evolution

### 11.1 Spec versioning

The spec document version (e.g., `0.1.1` of this file) and the schema version (e.g., `"0.1"` in skill files) MAY differ:

- The spec document is the human-readable normative reference. It evolves continuously with PATCH bumps for clarification, MINOR for additive normative changes, MAJOR for breaking changes.
- The schema version embedded in `SKILL.md` files identifies which schema generation a skill targets. Schema versions are bumped only on MINOR/MAJOR-equivalent changes; PATCH-level spec edits do NOT bump the schema version.

### 11.2 Field deprecation

When a field is deprecated:

1. The spec MINOR-bumps and marks the field "deprecated as of vX.Y".
2. Banks supporting that schema_version continue to recognize the field.
3. After two MINOR bumps, the spec MAJOR-bumps and removes the field.
4. New skills published with the schema_version following the MAJOR bump MUST NOT use the removed field.

This gives a multi-version grace period for migration.

### 11.3 Backward-incompatible changes

MAJOR spec bumps require a new schema_version. Implementations MAY support multiple schema versions in parallel. Skills declaring incompatible versions are surfaced to the operator with a migration prompt; they are NOT silently ignored.

## 12. Open questions

Tracked in [`ROADMAP.md`](./ROADMAP.md). Notable items:

- Should chains support conditional execution (if/else)?
- How should the bank surface "no skill found" to mitigate retrieval-first cognitive cost (see [`DESIGN.md`](./DESIGN.md) A1)?
- i18n: separate skills per language or per-language frontmatter fields?
- Aggregator protocol formalization.
