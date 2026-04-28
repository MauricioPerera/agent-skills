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

## 5. Identity, signing, and trust

### 5.1 Provenance verification levels

Banks MAY require one of these levels per subscription. **All levels still require URLs to conform to §1.1** (a valid host + URL template); the levels add successive trust layers on top of that.

- **Level 0 (no verification)**: source URL is accepted as-is; no signature or hash check beyond URL conformance and HTTPS transport (which the spec assumes throughout). Suitable for development.
- **Level 1 (TLS only)**: HTTPS chain validates the source domain. The bank refuses HTTP-only sources. Default for most consumers.
- **Level 2 (commit pinning)**: `<ref>` MUST be a commit hash, not a tag. Tag pins are resolved to a hash by the bank, then re-stored as the hash; the original tag is preserved only as `ref_requested` for human readability.
- **Level 3 (signed tags)**: the git tag MUST be signed by a key in `trusted_keys`. Verified via `git verify-tag` or equivalent. Implies Level 2.
- **Level 4 (Sigstore + Rekor)**: signature MUST be present in the public Sigstore transparency log and not revoked. Implies Level 3.

Level 2+ banks REJECT subscriptions to server-hosted skills (§3.3), since those have no commit hashes. Server-hosted is feasible only at Levels 0–1.

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
