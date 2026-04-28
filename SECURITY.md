# SECURITY — agent-skills threat model

This document enumerates the security properties of the `agent-skills` design, the threats it mitigates by construction, the threats it does NOT mitigate, and the operator's responsibilities.

## Design properties (by construction)

These properties are **structural** to the spec — implementations cannot accidentally violate them while still being conformant.

### P1: Credentials never enter the LLM context

The skill's `command_template` references environment variables (`$STRIPE_SECRET_KEY`, `$AUTH_TOKEN`) but never embeds their **values** in the published file. The LLM emits a command containing the variable name; the shell substitutes the value at execution time.

**Why this works**: the LLM never observes the credential. The credential lives in the operator's shell environment, in `~/.config/<tool>/`, or in encrypted local storage (e.g., `db users` with `encryptionKey` set on the just-bash-data plugin).

**Counter-MCP property**: MCP tool calls require credentials to flow through tool args (which the LLM produces) or to be held by the MCP server (which loses per-user isolation). agent-skills sidesteps both.

### P2: Skill content is content-addressable

Every skill identity contains either a 40-character SHA or refers to one transitively via a tag. Bit-for-bit content at a given identity URL is verifiable.

**Why this matters**: a malicious actor cannot silently modify a previously-published skill. Their options are:
1. Publish a new version (visible in `git log`, requires re-ingest).
2. Force-push to overwrite history (detectable; GitHub logs the operation; signed tags prevent it).
3. Compromise the CDN (would require compromising both git host AND CDN simultaneously; mitigations: pin SHA, verify hash on ingest).

### P3: Sync is one-way (provider → consumer)

The skill bank fetches; it does not POST. Skill providers receive **zero telemetry** about consumers. They cannot:
- Count installations.
- See query text or intent.
- Identify users.
- Block or rate-limit specific users (beyond what the CDN does for everyone).

The skill bank's audit log lives only on the consumer's machine.

### P4: Provider downtime does not break consumers

Once a skill is ingested with a SHA pin, the consumer has a complete local copy. The provider can:
- Disappear (404, DNS removed) — consumer continues to function.
- Be acquired by a hostile party — consumer continues to function until they explicitly re-sync.
- Push malicious updates — consumer's `auto_update: false` policy blocks ingestion until operator reviews diff.

### P5: Audit trail is structurally complete

Every change to a skill is a git commit:
- Author email (verified by GitHub if "Verified" badge is enabled).
- Timestamp.
- Diff (the actual content change).
- Parent SHA (chains back to genesis).
- Optional GPG/SSH signature.

Consumers can compare any two SHAs they have observed and see the full history of changes between them. There is no "silently mutated" possibility.

## Threat model

### T1: Skill author publishes a malicious skill

A bad-faith provider publishes a skill that, when executed, exfiltrates data, runs arbitrary code, or otherwise harms the consumer.

**Mitigations**:
- **Pre-install audit**: the consumer's skill bank **MUST** show `command_template`, `network`, and `required_env` to the operator before activating a new skill. Operators inspect.
- **Sandboxing**: skill banks **SHOULD** support running skills in a constrained shell environment that enforces `network` allowlist, restricts FS access, and disallows process spawning beyond `required_commands`. The reference runtime ([`just-bash-data`](https://www.npmjs.com/package/just-bash-data)) inherits these primitives from `just-bash`.
- **Source curation**: operators **SHOULD** subscribe only to providers they trust (verified GitHub orgs, awesome-list curated packs, internal team repos).
- **Static analysis**: skill banks **MAY** scan `command_template` for known bad patterns (`curl ... | bash`, base64-encoded payloads, network calls outside the declared allowlist) and refuse to ingest.

**Residual risk**: a sufficiently obfuscated skill from a trusted provider can still cause harm. This is the same problem as `npm install` from a trusted publisher who turns rogue. agent-skills does not solve it; it provides the audit primitives so the harm is detectable post-hoc.

### T2: Provider account compromise

An attacker gains commit access to `github.com/stripe/agent-skills` and pushes a malicious update.

**Mitigations**:
- **SHA pinning**: consumers running with `auto_update: false` continue using the pre-compromise SHA. They are completely insulated.
- **Signed tags + key rotation**: if the compromise is at the commit-credential level (not the GPG key), `git verify-tag` fails for the malicious push. Skill banks with `verify_signature: true` reject the new tag.
- **Branch protection** (provider-side): GitHub branch protection rules can require signed commits, prevent force-push, require review. These are provider-side responsibilities.
- **Transparency monitoring**: third-party services (e.g., a `agent-skills-watch` aggregator) can monitor for force-pushes and alert subscribers.

**Residual risk**: a sophisticated attacker who compromises both commit credentials AND the GPG signing key can publish a signed malicious release. Mitigations: hardware-backed signing keys (YubiKey), multiple-signer requirement (e.g., release engineering team must co-sign).

### T3: CDN compromise

jsDelivr (or another CDN) is compromised and serves a different file at the same URL.

**Mitigations**:
- **Cross-CDN verification**: skill banks **MAY** fetch the same SHA-pinned file from two independent CDNs (jsDelivr + GitHub Raw + Statically) and compare hashes. Disagreement is a strong signal of compromise.
- **Direct git fetch**: consumers may bypass CDNs entirely with `git clone --depth 1`. Higher latency, fewer mutual-trust assumptions.
- **Sigstore + Rekor transparency log**: independent of CDN, the signature attestation lives in Rekor's append-only log.

**Residual risk**: zero-day compromise of a single CDN with malicious content briefly served. Mitigations are detection-oriented, not prevention.

### T4: Typosquatting

An attacker publishes `github.com/striipe/agent-skills` (note the extra "i") hoping users confuse it with `github.com/stripe/agent-skills`.

**Mitigations**:
- **Verified Publisher** badges (GitHub Verified Org, custom verification scheme): the canonical Stripe is verified, the typo squatter is not.
- **Awesome lists**: human-curated lists of canonical skill providers. Skill banks **SHOULD** ship with a default pinning of well-known publishers' canonical repos.
- **Stargazer / community-trust signals**: skill banks **MAY** display GitHub stars, dependent-skills count, or other social signals at install time.

**Residual risk**: same as `npm` — confusable names occasionally trick users. Tooling helps but cannot eliminate.

### T5: LLM emits malicious command

Even if the skill is benign, the LLM might emit values for `args` that cause harm (e.g., `customer_id: "; rm -rf /"`).

**Mitigations** (mandatory per §4.4 of `SPEC.md`):
- **Substitutions are shell-quoted by default**. The skill bank wraps every value in single quotes, escaping embedded single quotes correctly. `customer_id: "; rm -rf /"` is inserted as `'; rm -rf /'` — a literal string, not executable code.
- **Args validation**: when `args.<name>.pattern` is declared, values are matched against the regex before substitution. A `customer_id` with pattern `^cus_[a-zA-Z0-9]+$` rejects the malicious value at the validation layer.
- **`unquoted: true` opt-in**: skill authors needing raw substitution must declare it explicitly, and even then values containing shell metacharacters (`;`, `&`, `|`, `$`, `` ` ``, `(`, `)`, `<`, `>`, newline) **MUST** be rejected before insertion.

**Residual risk**: a skill author who declares `unquoted: true` AND fails to declare a strict `pattern` opens a command injection surface. Skill banks **MAY** refuse to ingest such skills, treating them as conformance violations.

### T6: Information leak via embedding API

If the consumer uses a third-party embedding API (OpenAI, Cohere, etc.), every query intent is sent to that provider.

**Mitigations**:
- **Local embedding models**: ggml-quantized BGE, GTE, MiniLM — all run locally via `ollama embed` or similar. Recommended for privacy-conscious deployments.
- **Self-hosted embedding service**: the operator runs their own embedding endpoint inside their network.
- **Per-deployment choice**: the embedding model is a deployment-time config, not a spec requirement. Operators choose their privacy-cost trade-off.

**Residual risk**: by definition, the embedding model provider sees query text. This is a property of the consumer's choice of embedding model, not the spec.

### T7: Skill bank compromise

An attacker gains access to the consumer's skill bank database and modifies indexed skills (changing a `command_template` to be malicious).

**Mitigations**:
- **Skill bank itself runs on a trusted host** (the consumer's own machine). If that host is compromised, the agent and the user's shell are also compromised — the skill bank is a footnote.
- **Re-sync from immutable source**: any time, the operator can wipe the skill bank and re-ingest from sub-pinned SHAs. Detection of tampering: compare on-disk skills to upstream by SHA.
- **Encryption at rest**: the reference runtime supports `encryptionKey` for AES-256-GCM at rest. Combined with v1.1.0's configurable `salt`, an attacker reading the disk needs the password to decrypt.

**Residual risk**: physical/host-level access defeats most software defenses. Out of scope for the spec.

## Operator responsibilities

The spec provides primitives. **Operators are responsible for**:

| Responsibility | Recommendation |
|---|---|
| Choose which providers to trust | Subscribe only to known orgs, awesome-lists, or internal repos |
| Choose pin granularity | Production: SHA pin. Development: tag pin. Avoid `latest`/`main` outside exploration |
| Audit changes before re-syncing | When a new version is available, review `git diff` between SHAs before approving |
| Manage trusted keys | Import keys deliberately; do not auto-import |
| Restrict skill bank access | Run as user-mode process; do not give skills `sudo` access |
| Encrypt sensitive state | Use `encryptionKey` + `salt` if storing skills with embedded secrets |
| Monitor audit log | Periodically review `db skill_audit` for unexpected patterns |
| Review skills with `unquoted: true` carefully | These bypass the default safety net |

## Comparison to MCP

| Threat | MCP mitigation | agent-skills mitigation |
|---|---|---|
| Credential leak via context | None (credentials are in args) | Structural — credentials never reach LLM |
| Tool definition tamper | Server-side trust | SHA-pin + signed tags |
| Provider compromise | Reissue tool defs, hope clients update | Operator-controlled pinning, no auto-update |
| Telemetry to provider | Possible (server sees usage) | None — sync is one-way |
| Audit trail | Server logs (controlled by provider) | Local audit DB (controlled by consumer) |
| Defense against malicious tool | Sandbox (varies by client) | Sandbox + provenance verification + signed tags |

## Reporting vulnerabilities

This spec repository is a draft and currently has no CVE process. If you find a flaw in the spec design itself (an attack the spec admits structurally), open a public issue or PR — the spec evolves in the open.

For implementations of the spec (skill banks, sync daemons, etc.), follow that implementation's vulnerability disclosure process.
