# agent-skills — Specification

**Version**: 0.4.0 (draft)
**Status**: Open for comment. Schema and protocol are subject to change before v1.0.0.

**Schema version** (the value embedded in `SKILL.md` files): still `"0.1"` — v0.4 is additive (new normative section §5.4 specifying the Level 4 verification interface; existing fields unchanged). No breaking changes to the SKILL.md format. v0.2.x banks remain conformant.

**v0.4.0** formalizes the **Level 4 client-side verification interface** in §5.4. The previous patches (v0.3.1 / v0.3.2 / v0.3.3) added optional fields *describing* what a bank had observed; v0.4.0 specifies what a bank operating at Level 4 must *do* to upgrade a `"sigstore"`-method tag from "host says invalid" to "client-verified valid" — in other words, the contract that resolves the Sigstore-on-host trap. The reference CLI v0.17.0 lands the parsing primitives (Rekor entry decoding, public-instance pinning); the verification primitives (inclusion-proof Merkle math, checkpoint signature, Fulcio chain) are queued for v0.18.

**v0.3.3** added an optional `provenance.signature_identity` field for `"sigstore"`-method tags (§5.1). The Fulcio cert's Subject Alternative Name (SAN) carries the OIDC subject (email or workflow URI) and Fulcio extension OID `1.3.6.1.4.1.57264.1.1` (or `.1.8`) carries the OIDC issuer. Banks SHOULD surface both. The reference CLI v0.16.0 ships extraction (cross-impl parity validated continuously). **Extraction is not verification**: the identity is what the cert *claims*; verifying the claim against Rekor is Level 4 work and remains queued.

**v0.3.2** widened `provenance.signature_method` to `"gpg" | "ssh" | "sigstore"` (§5.1). The reference CLI v0.15.0 ships SSH-tag detection. The same patch documents the **Sigstore-on-host trap**: a properly-signed Sigstore tag may legitimately receive a `bad_cert` verdict from the host once the short-lived Fulcio cert expires, so a `"sigstore"`-method tag with `status: "invalid"` is **ambiguous** without client-side Rekor verification (Level 4) — *not* equivalent to "forged".

**v0.3.1** added an optional `provenance.signature_method` field to §5.1 (gpg / sigstore detection); the reference CLI v0.14.0 shipped it. Spec patch only — the value is informative on top of the existing Level 3a verdict. Full Sigstore Level 4 verification (Rekor inclusion proof) is queued for a future revision.

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
| `required_env` | string[] | Env var **names** the skill REQUIRES to function (presence-checked at retrieval / pre-exec). **Values are never published.** |
| `optional_env` | string[] | Env var names the skill MAY read but does not require. Used by sandboxed banks to determine the full env-access list (see §4.4). |
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

**Quoting policy** (per type):

| Type | Substitution rule |
|---|---|
| `string` | wrapped in single quotes; embedded single quotes encoded as `'\''` |
| `integer`, `number` | inserted as-is (the type system guarantees no shell metacharacters: digits, optional `-`, optional `.`, optional `e±N`) |
| `boolean` | inserted as the literal `true` or `false` (unquoted) |
| `array`, `object` | JSON-encoded, then single-quoted as a string |

The result of every substitution is **a single shell argument** suitable for direct use after a flag (e.g., `-d amount={amount}` becomes `-d 'Hello World'` for a string value, or `-d 1000` for an integer).

**Skill authors MUST place `{placeholder}` in argument position**, never inside literal `"..."` or `'...'`:

✅ `curl -d amount={amount}` — placeholder is its own arg after `-d`. Bank substitutes `{amount}` with the properly-quoted value.

❌ `curl -d "amount={amount}"` — placeholder inside literal double quotes. UNSAFE: the bank still substitutes the placeholder, but the surrounding double quotes alter how the shell parses the result. With a string value containing single quotes (which the substitution would escape with `'\''`), the parsed result is malformed.

❌ `curl -d 'amount={amount}'` — placeholder inside literal single quotes. The shell's quoting rules treat the single-quoted region as opaque; the substituted value's escaping is taken literally, producing wrong output.

The bank's substitution does NOT attempt to "fix" templates that embed placeholders inside literal quotes; the substituted value is inserted verbatim and the resulting command may be malformed or vulnerable. Conformant banks SHOULD detect this at ingest time (a literal quote character `"` or `'` immediately preceding or following a `{name}` substring is a strong signal of violation) and refuse such templates with a clear error.

**Composing more complex values**: skills that need to build a quoted string from multiple parts SHOULD do so via shell features in the template, not by embedding placeholders inside quotes. For example, to build a JSON body from multiple args:

```bash
# Conformant: jq composes the JSON, every {placeholder} is in argument position
curl -X POST https://api.example.com/charges \
  -d "$(jq -n --arg amt {amount} --arg cur {currency} '{amount: $amt, currency: $cur}')"
```

Here, every `{placeholder}` sits between two whitespace-separated arguments to `jq`. The bank's single-quoting produces well-formed `--arg` values; `jq` builds the JSON; bash double-quotes the result of `$(...)` for `-d`. No placeholder is inside a literal quote.

**Disambiguation — command substitution vs literal strings**:

The "no placeholders inside literal quotes" rule applies to **literal `"..."` and `'...'` string contexts**, where the substituted value's escaping conflicts with the surrounding quote semantics. It does NOT apply inside `$(...)` command substitution — even if the `$(...)` itself is wrapped in `"..."` for splitting/expansion control.

```bash
# ✅ Conformant: {amount} is an argument to printf, inside command substitution.
#   The outer "..." wraps the RESULT of $(...), not the placeholder.
echo "$(printf 'amount=%d' {amount})"

# ❌ Non-conformant: {amount} is inside a literal double-quoted string.
echo "amount={amount}"
```

Conformant banks SHOULD use a parser that distinguishes these two contexts when checking templates at ingest time. A naive regex looking for `{name}` adjacent to `"` will produce false positives on the conformant first form; the parser must recognize `$(...)` as a command-substitution context where placeholders are at argument boundaries to the inner command.

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

# Source-specific fields:
#   When source_type == "git": subscription targets a versioned git repository.
#     The bank resolves refs to commit hashes; can pin SHAs; supports tags;
#     supports signing. This is the production-recommended mode.
#   When source_type == "url": subscription targets a server-hosted skills-index.json
#     directly. No git semantics — no SHA, no tags, no signing. Suitable only for
#     provenance Levels 0-1 (§5.1). Server-hosted skills (§3.3) use this mode.

# git subscriptions:
repo: "github.com/owner/repo"
ref_requested: "v1.2.0"               # tag, branch, or hash the user specified
ref_resolved: "a1b2c3d4..."           # commit hash at last successful sync
url_template: "..."                   # optional, overrides built-in (§3.2)

# url subscriptions:
index_url: "https://example.com/skills-index.json"
last_etag: "..."                      # optional pseudo-hash from HTTP ETag (§7.1)

# common:
auto_update: false                    # if true, sync follows ref; if false, manual approval required
last_synced: "2026-04-28T..."
verify_signature: true                # if true, refuse to ingest unsigned tags (§5)
trusted_keys: ["fingerprint1", "..."]
```

Banks operating at Provenance Level 2 or higher (§5.1) MUST refuse subscriptions with `source_type: "url"` since those lack commit hashes for pinning.

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

If `title + use_when` ALONE exceeds the model's max input (an unusual but possible case with very small embedding models — e.g., 256-token limit), the bank MUST refuse to ingest the skill and surface an error like `embedding model context too small for skill <id> (need ≥ N tokens, model accepts M)`. The bank MUST NOT produce a degraded embedding from truncated `title + use_when` — that would silently destroy retrieval quality. Operators SHOULD then reconfigure their bank with a model that accepts longer inputs (most modern embedding models support ≥ 512 tokens; many support 8K+).

Banks MUST NOT silently produce embeddings of truncated input without recording the fact: `provenance.embedding_truncated: true` SHOULD be set in the index when truncation occurred at the `description` / `examples` / `tags` level.

Banks MUST NOT use **different** embedding models for different skills within the same index, nor for the indexed skills versus the agent's queries. Mixed-model search is undefined.

The choice of embedding model is a per-bank deployment decision. Skills MUST NOT publish vectors. (See [`DESIGN.md`](./DESIGN.md) D4.)

### 4.3 Retrieval contract

Given a query, the bank MUST:

1. Embed the query with the same model used to index skills.
2. Run nearest-neighbor search over the index, returning top-`K` (default `K=10`).
3. Optionally re-rank by audit-derived signals (see §4.3.1 for known patterns).
4. Optionally filter by `applicable_when` against the host environment.
5. Return the top-`N` (default `N=3`) skill identities + selected metadata.

Banks SHOULD support a query option to bypass `applicable_when` filtering for debugging.

#### 4.3.1 Rerank patterns *(new in v0.2)*

When the bank has audit history (§4.5), it MAY re-rank candidates by combining cosine similarity with usage signals. Two patterns are documented; banks MAY implement either, both, or neither, but SHOULD expose the choice to operators:

**Global rerank** — the simple pattern. Add a usage-count and recency boost to every candidate based on the global audit log:

```
final_score = cosine
            + α · log(1 + usage_count)
            + β · recency_boost
```

where `usage_count` is the total number of audit entries for the skill (across all past intents) and `recency_boost ∈ [0, 1]` decays exponentially with time since the last use. Suggested defaults: `α = 0.05`, `β = 0.03`, recency half-life 7 days.

**Failure mode**: under usage concentration (one skill used dramatically more than others), the boost overwhelms cosine differences and the dominant skill wins unrelated queries. Empirically observed: 50 concentrated past uses on one skill collapses top-1 accuracy from 97% to 34% on the reference 7-skill / 35-paraphrase corpus. Banks SHOULD document this risk.

**Intent-conditional rerank** — counts only past invocations whose recorded `intent` (§4.5) is semantically similar to the current query. Replaces `usage_count` with a query-conditional `n`:

```
n = |{ past_audit_entry : cos(query_vec, past_intent_vec) ≥ threshold }|
final_score = cosine
            + α · log(1 + n)
            + β · recency_boost_of_relevant_only
```

where `past_intent_vec` is the embedding of the audit entry's `intent` field and `threshold` is a similarity floor (suggested default `0.7`). Past intents below `threshold` are excluded from both the count and the recency calculation. Banks implementing this pattern MUST embed past intents at query time (lazily, cached) using the same model that indexed the skills.

Empirically validated to recover 100% top-1 under the same usage-concentration scenario where global rerank degrades to 34% — by activating the boost only on queries whose past intents are semantically related.

**No-rerank mode** — banks SHOULD support disabling rerank entirely (operator opt-out). Cosine alone is the safe baseline for adversarial / multi-tenant audit logs.

The bank's choice of rerank pattern, weights, and threshold is a deployment decision. The spec does not mandate any particular default.

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
- Block access to env vars not in `required_env ∪ optional_env`. (`required_env` is the presence-required set; `optional_env` extends the read-access set without making them mandatory. A skill that does not declare any env access at all has zero env-var visibility under sandbox mode, even if such variables exist on the host.)

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

The `intent` field, when present, enables intent-conditional rerank (§4.3.1). Banks supporting that rerank pattern MUST persist `intent` alongside other audit fields. Banks MAY also use `intent` for offline analytics, retrospective query-quality evaluation, etc. — the field is local-only like the rest of the audit log.

### 4.5.1 Per-tenant audit scoping *(new in v0.3)*

When the bank is shared by multiple agents or users (multi-tenant deployment — shared CI runners, team setups, multi-user agent infra), the audit log SHOULD record a per-call `tenant` identifier and rerank SHOULD scope past entries by tenant.

**Schema (additive to §4.5):**

```yaml
audit_entry:
  ...                       # all fields from §4.5
  tenant: "alice"           # optional, v0.3+. Free-form string identifying
                            # the agent / user / role making this exec call.
```

The field is **optional**. Audit logs and entries that omit it (e.g., everything written by v0.2-conformant banks) are valid v0.3 audit data — they represent a single, shared tenant and rerank uses every entry.

**Format**: implementations MAY enforce a tighter charset to keep the value safe across audit-log JSON, filenames, query parameters, etc. The reference CLI uses `^[a-zA-Z0-9._-]{1,64}$`. The spec does not mandate a specific regex, only that the value be a non-empty string.

**Rerank semantics (per SPEC §4.3.1):**

When the bank's query API receives a `tenant` parameter:

1. **Filter the audit log to entries where `audit_entry.tenant === query.tenant`** before computing usage_count / conditional_count / recency_boost.
2. Run the rerank pattern (global or intent-conditional) on the filtered subset.
3. Return the top-K. The result includes the per-skill counts as in v0.2; those counts now reflect the tenant-filtered universe.

When the bank's query API does NOT receive a `tenant` parameter, behaviour is identical to v0.2: every audit entry participates in the boost.

**Worked example.** Alice and Bob share a bank. Alice has called `base64-encode` 50 times for "encode credential" intents; Bob has never used base64. Without tenant scoping (v0.2), a query from Bob for "fetch URL contents" suffers the v0.4-documented stress regression — `base64-encode` ranks above `http-get` because its global usage count overwhelms cosine. Under v0.3 per-tenant scoping with `query.tenant === "bob"`, Alice's audit entries are filtered out before computing the boost, and Bob's queries return cosine-correct results.

**Privacy invariants (§8):**
- P3 still holds: per-tenant audit data still lives only locally; banks MUST NOT transmit it to skill providers.
- A new implication: when one tenant queries the bank, the bank MUST NOT leak another tenant's audit history through the rerank result. The filter described above achieves this.

**Implementations:** the reference CLI [`agent-skills-cli`](https://github.com/MauricioPerera/agent-skills-cli) v0.12.0+ implements this section. Operators set `--tenant <id>` on `exec`, `query`, and `bench`.

### 4.6 Bench protocol *(new in v0.2)*

Skill packs and operators SHOULD ship a **truth file** that pairs natural-language intents with the skill `id` an agent should retrieve for each. The format is JSONL or JSON-array, auto-detected by the first non-whitespace character (`[` ⇒ JSON array, otherwise JSONL):

```jsonl
# JSONL — one entry per line. Blank lines and lines starting with # are ignored.
{"intent": "fetch the contents of a URL", "expected": "http-get"}
{"intent": "encode a string as base64", "expected": "base64-encode"}
```

```json
[
  { "intent": "fetch the contents of a URL", "expected": "http-get" },
  { "intent": "encode a string as base64", "expected": "base64-encode" }
]
```

**Field semantics:**
- `intent` — free-form natural language; one prompt per entry. (No upper bound; banks SHOULD accept entries up to at least 2 KB.)
- `expected` — the **short** skill `id` (frontmatter `id`, e.g., `"http-get"`), NOT the full identity. Short ids make truth files portable across pack revisions; banks resolve them at run time and MUST fail fast if any `expected` doesn't resolve to exactly one installed skill.

**Operator workflow:**

A bank tool MAY implement a `bench <truth-file>` command that:
1. Loads the truth file, fails fast on parse errors or unresolvable expecteds.
2. For each entry, runs the bank's normal retrieval (same code path as a regular query).
3. Reports top-1 / top-3 / top-K accuracy, mean top-1 score, mean margin (top-1 → top-2), and per-failure breakdown (intent, expected, rank, what was retrieved at top-1).
4. Exits non-zero when any failure exists, so CI breaks on retrieval regression.

**Recommended placement:** at the pack root as `bench-truth.jsonl`. Anyone consuming the pack can validate retrieval quality against their local provider with one command.

**This is OPTIONAL.** Packs without a truth file are conformant; truth files are how publishers and consumers measure retrieval quality, not part of the loading pipeline.

### 4.7 Embedding provider abstraction (informative) *(new in v0.2)*

The spec is provider-agnostic: it requires that "the same model used to index skills is used to embed queries" (§4.3) but doesn't dictate which model. Implementations have converged on a small triplet for any provider:

```
EmbeddingProvider:
  name : string         # stable identifier including the model
                        # (e.g., "cloudflare:@cf/baai/bge-base-en-v1.5",
                        #         "ollama:nomic-embed-text",
                        #         "openai:text-embedding-3-small")
  dim  : integer        # output vector dimensionality
  embed : (text) → vec  # produces a `dim`-element vector
```

Banks SHOULD record `name` in their meta and refuse to mix vectors produced by different providers (`name` mismatch ⇒ rejected at sync time). The `dim` value is useful for early dim-mismatch detection at query time.

Three reference provider classes documented in [`agent-skills-cli`](https://github.com/MauricioPerera/agent-skills-cli):

- **Cloudflare Workers AI** — hosted, free tier available. Models: `@cf/baai/bge-{small,base,large}-en-v1.5` (384/768/1024-dim), `@cf/baai/bge-m3` (1024-dim, multilingual), `@cf/google/embeddinggemma-300m` (768-dim).
- **Ollama** — local, zero-credentials, zero network egress. Models: `nomic-embed-text` (768-dim), `mxbai-embed-large` (1024-dim), `bge-m3` (1024-dim, multilingual), `embeddinggemma` (768-dim).
- **OpenAI / OpenAI-compatible `/v1/embeddings`** — covers OpenAI, Together, Anyscale, Mistral, vLLM, infinity, TEI in compatibility mode, etc. via a shared base URL.

The CLI documents the auto-detect priority and env-var conventions for each (informative, not part of the spec).

The provider choice is a **local trust decision** for the bank operator. The spec does not impose a model; it imposes only that within one bank, indexing and query use the same model.

## 5. Identity, signing, and trust

### 5.1 Provenance verification levels

Banks MAY require one of these levels per subscription. **All levels still require URLs to conform to §1.1** (a valid host + URL template); the levels add successive trust layers on top of that.

- **Level 0 (no verification)**: source URL is accepted as-is; no signature or hash check beyond URL conformance and HTTPS transport (which the spec assumes throughout). Suitable for development.
- **Level 1 (TLS only)**: HTTPS chain validates the source domain. The bank refuses HTTP-only sources. Default for most consumers.
- **Level 2 (commit pinning)**: `<ref>` MUST be a commit hash, not a tag. Tag pins are resolved to a hash by the bank, then re-stored as the hash; the original tag is preserved only as `ref_requested` for human readability.
- **Level 3 (signed tags)**: the git tag MUST be signed and verifiable. Implies Level 2. Two verification methods are documented (banks MAY support either or both; see §5.3 for the trust trade-offs):
  - **Level 3a (host-verified)**: the bank queries the source host's API and trusts its server-side GPG verification. For GitHub: `GET /repos/{owner}/{repo}/git/tags/{tag_sha}` returns a `verification` object with `verified: true/false` based on whether the tag's signature was made by a key bound to the publisher's account. Trust assumption: the bank trusts the host to have correctly bound signing keys to publisher identities.
  - **Level 3b (client-verified)**: the bank verifies the tag signature locally with `git verify-tag` (or equivalent) against a `trusted_keys` allowlist on the subscription. The bank holds the canonical key fingerprints; the host is **not** in the trust path. Stronger guarantee than 3a, more operator burden.
- **Level 4 (Sigstore + Rekor)**: signature MUST be present in the public Sigstore transparency log and not revoked. Implies Level 3.

A bank operating at Level 3a SHOULD record the host's reason string (e.g., `"valid"`, `"unknown_key"`, `"unsigned"`) in `provenance.signature_status` so an operator can distinguish *""sloppy publisher hygiene""* (unsigned) from *""active red flag""* (signature present but unverifiable). The `signature_status` enumeration is `"valid" | "invalid" | "unsigned" | "unverified"`; the last value indicates the bank could not perform verification (non-supported host, lightweight tag with no tag object, ref is a raw SHA, etc.) and is **not** equivalent to "valid".

**Signing method detection** *(new in v0.3.1, extended in v0.3.2)*. Banks SHOULD additionally surface `provenance.signature_method` when a signature payload is present and recognisable:

  - `"gpg"` — classic OpenPGP-armored signature (PEM header `-----BEGIN PGP SIGNATURE-----`).
  - `"ssh"` *(v0.3.2)* — SSH-format git signature (PEM header `-----BEGIN SSH SIGNATURE-----`), produced by `git config gpg.format ssh && git tag -s`. Trust comes from the SSH pubkey being attached to the publisher's host account. Common in modern publisher setups; many large projects (e.g. `sigstore/cosign`'s release tags) use SSH signing rather than OpenPGP.
  - `"sigstore"` — gitsign / Sigstore CMS signature (PEM header `-----BEGIN SIGNED MESSAGE-----`), produced by `gitsign` or the cosign git-sign workflow. Uses Fulcio-issued ephemeral certs + Rekor transparency log.

Detection is structural (PEM header), not full Level 4 verification: the bank still trusts the host's verdict on the signature's validity. The method field gives operators visibility into WHICH cryptographic system signed the tag — useful for policy decisions ("we only accept Sigstore-signed packs"). Full Rekor inclusion-proof verification is what the spec calls Level 4 (below) and is queued for a future revision; the method field surfaces on top of the existing Level 3a verdict.

The field is **optional**. Banks that don't implement detection MAY omit it. Agents reading provenance MUST treat its absence as "method unknown", not as "method GPG by default".

**Sigstore-on-host trap** *(new in v0.3.2)*. A `"sigstore"`-method tag deserves special interpretation rules, because a properly-signed Sigstore tag can legitimately produce an `invalid` verdict from a Level 3a host:

  - Fulcio issues ephemeral certs that **expire ~10 minutes** after the signing event. Inclusion in Rekor is the durable proof that the signature was made when the cert was valid.
  - Hosts that re-validate the cert chain at *lookup* time (e.g., GitHub's `verification.reason: "bad_cert"`) will reject every Sigstore signature older than ~10 minutes — including correctly-signed ones.
  - Therefore, for `signature_method: "sigstore"`, banks MUST treat `status: "invalid"` with a `reason` indicating cert expiry/validity (e.g., `"bad_cert"`, `"expired"`) as **ambiguous**: the signature *may* be perfectly valid via Rekor, but the host can't tell. Banks SHOULD log a distinct hint (`"sigstore_host_unverifiable"` or similar) and SHOULD NOT treat this case identically to `"unknown_key"` (a true red flag for GPG/SSH).
  - Banks operating at Level 4 (client-side Rekor verification) resolve this ambiguity unconditionally; banks operating at Level 3a SHOULD surface the ambiguity to operators rather than silently rejecting.

This rule does **not** apply to `"gpg"` or `"ssh"` methods, where `invalid` reflects a real verification failure.

**Sigstore identity claim** *(new in v0.3.3)*. For `signature_method: "sigstore"` tags, banks SHOULD surface the cert's identity claim as `provenance.signature_identity`:

```json
{
  "subject": "billy@chainguard.dev",
  "subject_type": "email",
  "issuer": "https://accounts.google.com"
}
```

  - `subject` — the OIDC subject pulled from the Fulcio cert's first Subject Alternative Name (SAN) entry. For human signers, this is typically an email; for GitHub Actions OIDC signing, it is a workflow URI (`https://github.com/<org>/<repo>/.github/workflows/<file>@<ref>`).
  - `subject_type` — one of `"email"`, `"uri"`, `"other"`. Reflects the SAN GeneralName tag (rfc822Name, uniformResourceIdentifier, …).
  - `issuer` — the OIDC issuer URL from Fulcio extension OID `1.3.6.1.4.1.57264.1.1` (v1) or `1.3.6.1.4.1.57264.1.8` (v2). Examples:
    - `"https://accounts.google.com"` — Google OAuth
    - `"https://github.com/login/oauth"` — GitHub user OAuth
    - `"https://token.actions.githubusercontent.com"` — GitHub Actions OIDC

> **Extraction ≠ verification.** The identity claim above is what the cert *claims* it was issued to. Verifying that claim is genuine — i.e. that an attacker hasn't forged a Fulcio cert with arbitrary SAN values — requires checking the Rekor inclusion proof against Sigstore's public transparency log and validating the cert chain against the Fulcio root. That's Level 4 work (below) and is **not** done by extraction alone. Operators may surface this field for informational purposes ("publisher claims to be `<subject>` via `<issuer>`") but MUST NOT treat it as authenticated until Level 4 is implemented.

The field is **optional**. Banks that don't implement extraction MAY omit it. Banks that do implement it but encounter a malformed CMS payload SHOULD also omit the field rather than reporting partial / wrong data — extraction failure is treated identically to "no Sigstore signature".

Level 2+ banks REJECT subscriptions to server-hosted skills (§3.3), since those have no commit hashes. Server-hosted is feasible only at Levels 0–1.

### 5.2 Trusted-key management

Banks SHOULD support trusted-key configuration. Implementation-specific format; an example is given in [`IMPLEMENTATION.md`](./IMPLEMENTATION.md).

Banks MUST NOT auto-import keys. Adding a key is a deliberate user action. Banks SHOULD verify the key fingerprint matches a value the user explicitly provides (out-of-band trust establishment).

### 5.3 Verification trust trade-offs *(new in v0.2)*

The Level 3 split into 3a (host-verified) and 3b (client-verified) reflects a real operator trade-off, made explicit so banks document their choice rather than papering over it.

**Level 3a — host-verified (e.g., GitHub's `verification` API):**
- ✅ Zero operator burden: no key management, no GPG installation, no fingerprint distribution.
- ✅ Works out of the box for any GitHub-hosted pack signed by a key the publisher uploaded to their GitHub account.
- ❌ The host (GitHub) is in the trust path. A compromise of the host's key-binding system, or an account takeover that adds a new signing key, defeats verification.
- **Adversary**: an attacker who compromises the publisher's GitHub account can upload their own signing key to that account and produce *""valid""* tags. Level 3a does not detect this.

**Level 3b — client-verified (`git verify-tag` against `trusted_keys`):**
- ✅ The host is **not** in the trust path. Even a fully compromised host cannot forge a `trusted_keys`-validated tag without also stealing the publisher's private key.
- ✅ Strongest guarantee short of Sigstore + Rekor.
- ❌ Operator burden: must establish key trust out-of-band (publisher's website, in-person, signed key party, etc.), maintain `trusted_keys`, rotate on compromise.
- ❌ Doesn't catch an attacker who steals the publisher's private signing key (no system does, short of revocation).

**Recommended posture by deployment type:**
- Personal use, public packs: **3a is sufficient.** The marginal risk over 3b is small for low-stakes use.
- Production agents, internal packs: **3b for first-party packs**, 3a for third-party. Pin `trusted_keys` in subscription records.
- High-trust automation (financial, ops): **Level 4 (Sigstore + Rekor)** — adds a public, append-only audit log of every signature, detects key rotation against the transparency log.

Banks SHOULD expose `signature_status` and `signed_by` on every retrieval result so the agent (or its supervisor) can decide per-call whether to act on a skill of a given trust level.

### 5.4 Level 4 client-side verification interface *(new in v0.4)*

This section specifies the **contract** a bank MUST implement to claim Level 4 verification. It does NOT mandate a particular implementation — banks MAY use the `@sigstore/verify` library, hand-rolled primitives, or any other path that meets the contract.

The motivation is the **Sigstore-on-host trap** documented in §5.1: a properly-signed Sigstore tag will receive an `invalid` verdict from a host that re-validates the Fulcio cert at lookup time, because Fulcio certs are short-lived (~10 minutes). For `signature_method: "sigstore"` tags, the only sound trust path is client-side verification against the Rekor transparency log + Fulcio root of trust. Level 4 is that path.

**5.4.1 Inputs.** A Level 4 verifier is given:
1. The CMS payload from `verification.signature` (PEM-armored, `-----BEGIN SIGNED MESSAGE-----`).
2. The signed payload from `verification.payload` (the tag content that was hashed and signed).
3. (Optional) Subscription-level identity expectations:
   - `expected_subject` — string match on the Fulcio cert's first SAN entry.
   - `expected_issuer` — exact-match on the Fulcio OIDC-issuer extension value.

**5.4.2 Verification steps.** The verifier MUST perform all of the following, in any order, and MUST fail closed (verdict = "invalid") if any step fails:

1. **Parse the CMS payload.** Extract the first X.509 cert (the Fulcio leaf cert) and the signed digest. (v0.16+ banks already do this for identity extraction; the same parser drives Level 4.)

2. **Validate the Fulcio cert chain.** The leaf cert MUST chain to a pinned Fulcio root certificate. Banks SHOULD obtain Fulcio roots via Sigstore's TUF repository at `https://tuf-repo-cdn.sigstore.dev/`; banks MAY pin a static root for simplicity, but MUST document the rotation policy.

3. **Locate the Rekor entry.** Compute the artifact hash that Rekor expects for this signature kind (gitsign uses `hashedrekord` v0.0.1; the artifact-hash semantics are documented in `gitsign`'s source). Query the public Rekor instance at `https://rekor.sigstore.dev` (or a configured private instance) for an entry whose `body.spec.data.hash.value` matches AND whose `body.spec.signature.publicKey.content` decodes to the same Fulcio cert from step 1. There MAY be multiple matching entries (e.g., re-signing); the verifier SHOULD accept the oldest matching entry whose `integratedTime` is within the cert's validity window.

4. **Verify the inclusion proof.** Compute the Merkle root from the entry's leaf hash (defined as `RFC6962-style` SHA-256 of `0x00 || entry.body`) and the proof's audit hashes (`hashes`, leaf-to-root order). The computed root MUST equal `inclusionProof.rootHash`. The proof's `logIndex` is the **local** position within the shard's tree (identified by `entry.logID`), NOT the global `entry.logIndex` — verifiers that confuse the two will reject valid proofs. The proof's `treeSize` MUST match the size declared in the checkpoint body.

5. **Verify the checkpoint signature.** The `inclusionProof.checkpoint` is a [C2SP signed-note](https://c2sp.org/signed-note): a 4-line body (`origin / treeSize / rootHash-base64 / blank`) followed by a signature line of the form `— <key-name> <base64-encoded-blob>`. The base64 blob is `<4-byte-key-hint> || <raw-ECDSA-P-256-signature>`. The signature MUST verify against Rekor's pinned public key (also obtained via TUF, or pinned statically with documented rotation policy), with the body bytes (lines 1-4 followed by a single `\n`) as the message.

6. **Verify integrated time within cert validity.** `entry.integratedTime` MUST fall within the leaf cert's `notBefore`/`notAfter` window. (Fulcio certs are valid ~10 minutes; if Rekor recorded the entry within that window, the signing event was real even though the cert has since expired.)

7. **Verify identity (if specified).** If the subscription supplies `expected_subject` and/or `expected_issuer`, the cert's SAN and Fulcio OIDC-issuer extension MUST match exactly. Mismatch is a verification failure.

**5.4.3 Result.** On success, the bank MAY override the host's `signature_status` to `"valid"` even when the host returned `"invalid"` with reason indicating cert expiry (`"bad_cert"`, `"expired"`, etc.) — this is the **only** legitimate way to upgrade the verdict on a `"sigstore"`-method tag. Banks MUST NOT upgrade the verdict for any other reason or any other method.

A bank that successfully verifies a tag at Level 4 SHOULD record the path taken in `provenance.signature_verification_path` (e.g., `["rekor", "fulcio"]`) and the local Rekor entry coordinates (`provenance.rekor_log_index`, `provenance.rekor_uuid`) so an auditor can re-verify offline against the same entry.

**5.4.4 What v0.4.0 standardizes vs. defers.** This section specifies the verification *contract* — what banks must do, not how. Reference-implementation work proceeds in two phases:

- **Phase 1 (CLI v0.17.0)**: Rekor entry parsing + public-instance pinning + entry fetch by UUID. No verification. Operators can inspect what Rekor *claims* about an entry.
- **Phase 2 (CLI v0.18.0+)**: full verification (Merkle math + checkpoint signature + Fulcio chain + identity matching). Implementation may use `@sigstore/verify` (audited Sigstore-project library) or hand-rolled primitives — the spec does not prescribe.

Until Phase 2 ships, no bank can claim Level 4 verification. Banks operating below Level 4 MUST NOT override host verdicts for `"sigstore"`-method tags, and SHOULD continue to surface `"sigstore"`-method invalid verdicts as **ambiguous** per §5.1's Sigstore-on-host trap rule.

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

Future spec versions may add fields. To prevent collisions, the following are reserved at the top level of the frontmatter and MUST NOT be declared by skill authors:

- `provenance` (entire object — §2.5; bank-computed at ingest).
- `deprecated`, `removed` (bank-set on superseded skills during sync).
- `inserted_at`, `updated_at`, `last_synced` (bank-set timestamps).
- `usage_count`, `avg_rating`, `success_rate` (bank-computed from audit signals).

Vendor-specific or extension fields MUST go inside `metadata`:

```yaml
metadata:
  vendor_specific_field: "..."
```

Banks ingesting a skill that declares any reserved field at the top level MUST log a warning. Banks operating in strict mode MUST refuse such skills.

## 10. Conformance levels

### 10.1 Skill / publisher levels

- **A1 (Author-conformant)**: produces `SKILL.md` with all required + recommended fields. NOT required to publish `/llms.txt`.
- **A2 (Publisher-conformant)**: A1 + `/llms.txt` + `/skills-index.json` + git-sourced provenance.
- **A3 (Verified publisher)**: A2 + signed tags + a published key (e.g., `.well-known/agent-skills-key.asc`).

### 10.2 Skill bank levels

- **B1 (Consumer-conformant)**: ingests A1+ skills (any source type), performs embedding per §4.2, supports retrieval and execution per §4.
- **B2 (Strict consumer)**: B1 + provenance Level 2+ enforced + sandbox mode + audit log + `applicable_when` filtering.

### 10.3 Forward + backward compatibility

A bank declares the highest schema version it fully supports. Encountering a skill with schema version `X.Y`:

- If `X.Y` is **lower than or equal to** the bank's max supported version: the bank MUST ingest and execute the skill. Fields the bank doesn't recognize (added in PATCH-level spec edits but not visible in the bank's stale parser) MUST be preserved verbatim in the index but treated as opaque.

- If `X.Y` is **higher within the same MAJOR**: the bank SHOULD attempt to parse known fields and ignore unknown ones. The skill is ingested in "best-effort" mode — execution may proceed if all known-required fields are present. If a known-required field is missing in the parsed subset (e.g., the skill declares `command_template_v2` introduced in 0.3 but the bank only knows `command_template` from 0.1, and the skill omitted the older field intentionally), the bank MUST refuse with a clear schema-mismatch error.

- If `X.Y` differs in **MAJOR**: the bank MUST refuse to ingest. Schemas with different MAJOR versions are not assumed parsable. The operator is notified of the mismatch and SHOULD upgrade the bank or ask the publisher for a compatibility version.

This rule is concrete: it does not depend on the bank "guessing" which fields are critical. The contract is **schema_version match within MAJOR + presence of known-required fields**.

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
