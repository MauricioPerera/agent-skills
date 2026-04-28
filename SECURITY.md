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

### P3: No first-party telemetry (one-way sync)

The skill bank fetches; it does not POST telemetry back. Skill providers receive **zero first-party data** about consumers from the bank itself. They cannot:
- Receive bank-originated install counts, ratings, or error reports.
- Receive query text or intent.
- Identify users via bank-emitted IDs.

**Important caveat**: this is *first-party* zero, not absolute. Transport-layer observers see traffic regardless:
- The git host (GitHub, GitLab) logs every clone with IP + User-Agent.
- The CDN (jsDelivr, Cloudflare) logs every request.
- The embedding model API (if remote) sees every query intent.

The spec's invariant is that **the bank does not phone home to the skill author/provider**. What network operators see at the transport layer depends on the operator's choice of CDN, embedding model, and network configuration. Operators concerned about transport-level observability should: use locally-hosted embedding models, fetch via Tor or VPN, or self-host the CDN.

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

**Mitigations** (consumer-side, what the spec controls):
- **SHA pinning**: consumers running with `auto_update: false` continue using the pre-compromise SHA. They are completely insulated until they manually re-pin.
- **Signed tag verification**: if the compromise is at the commit-credential level (not the GPG key), `git verify-tag` fails for the malicious push. Skill banks with `verify_signature: true` reject the new tag.
- **Compare against last-approved baseline**: per `SPEC.md` §7.4, banks SHOULD diff every prospective update against the operator's last manually-approved state, not just the previous auto-synced state. This makes accumulated drift visible across multiple sync cycles.
- **Transparency monitoring**: third-party watch services can subscribe to repos and alert on force-pushes, tag movements, or unsigned commits. Out-of-spec but enabled by the architecture.

**Provider-side recommendations** (not controlled by the spec, but worth recording):
- GitHub branch protection rules requiring signed commits + reviews + linear history.
- Hardware-backed signing keys (YubiKey, secure enclave).
- Multi-signer release process (e.g., release engineering team must co-sign).
- Sigstore / cosign for transparency-log-backed signatures (Level 4 conformance).

**Residual risk**: an attacker compromising both commit credentials AND signing key material can publish a signed malicious release. The Level-4 (Sigstore) mitigation reduces this further by requiring signatures to land in a public append-only log; revocation becomes detectable after the fact. Defense-in-depth at the provider organization level remains essential — the spec cannot replace good ops.

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

**Mitigations** (mandatory per §4.4 + §2.6 of `SPEC.md`):

- **Substitutions are single-quoted by default**. The skill bank wraps every string value in single quotes, escaping embedded single quotes as `'\''`. `customer_id: "; rm -rf /"` is inserted as `'; rm -rf /'` — a literal string passed to the next argument, not executable code.

- **Placeholders MUST appear in argument position** (D16). Templates that embed `{placeholder}` inside literal `"..."` or `'...'` are non-conformant; banks SHOULD refuse them at ingest. This eliminates the "what context am I in?" ambiguity that complex shell quoting otherwise creates.

- **Args validation via positive patterns**. When `args.<name>.pattern` is declared, the value is matched against the regex BEFORE substitution. A `customer_id` with pattern `^cus_[a-zA-Z0-9]+$` rejects malicious values at the validation layer, regardless of how they would be quoted later. **Pattern allowlisting is preferred over metacharacter denylisting** because allowlists fail closed (anything not explicitly permitted is rejected) while denylists fail open (any character not enumerated slips through).

- **`unquoted: true` opt-in**: skill authors needing raw substitution MUST declare it explicitly. The spec REQUIRES that any `unquoted: true` arg also declare a `pattern` that rejects ALL shell metacharacters. Banks MUST refuse to ingest skills that violate this constraint. The full set of characters that an unquoted-arg pattern MUST reject:

  ```
  ; & | $ ` ( ) < > * ? [ ] \ " ' { } # ~ space tab newline
  ```

  An author writing `pattern: "^[a-zA-Z0-9_-]+$"` for an `unquoted` arg is safe (none of the forbidden chars match). An author writing `pattern: ".*"` is rejected by conformant banks.

**Residual risk**: a non-conformant bank that does NOT enforce these constraints is vulnerable to a malicious skill author. Spec compliance and a JSON-Schema validation step at ingest are the recommended defense. Operators SHOULD verify their bank implementation enforces §2.6 before deploying in trust-sensitive contexts.

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
