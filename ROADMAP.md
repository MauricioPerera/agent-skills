# ROADMAP — agent-skills

This document tracks **what's planned, what's open, and what's deliberately out of scope**. The spec evolves in the open via PRs against this file and the canonical `SPEC.md`.

## Current state — v0.2.0 (draft)

The spec has been validated end-to-end by the reference CLI [`agent-skills-cli`](https://github.com/MauricioPerera/agent-skills-cli) through 11 minor releases (v0.5.0 → v0.11.0). v0.2 of this document formalises the patterns that crystallised during that cycle:

- **§4.3.1 Rerank patterns** (NEW) — global vs intent-conditional, with documented failure modes and empirical recovery profiles.
- **§4.5 audit `intent` field** — clarified role in intent-conditional rerank.
- **§4.6 Bench protocol** (NEW) — `bench-truth.jsonl` format for reproducible retrieval evaluation.
- **§4.7 Embedding provider abstraction** (NEW, informative) — `(name, dim, embed)` triplet + three reference provider classes.
- **§5.1 Level 3 split into 3a/3b** — host-verified vs client-verified, with explicit trust trade-offs in §5.3.

Schema version remains `"0.1"`. v0.1.x banks and packs remain conformant.

Resolved earlier (v0.1.1):
- Substitution rules formalized (C3, D16).
- Shell semantics declared (C6, D17).
- Provenance moved out of file to ingest-time computation (C1, D11 corrected).
- URL derivation rules (C2).
- Token math clarified in COMPARISON (S9).
- JSON Schema shipped.
- IMPLEMENTATION.md separated from canonical spec.

Status: **draft**. Schema is conservative — additive changes only since v0.1.0. v1.0.0 is feature-complete from the spec's perspective; the path to v1.0 is operational (more implementations, more packs, audit). See §"v1.0.0 — Stable spec" below.

## Reference implementation — shipped

The work originally listed under [v0.2.0 / v0.3.0 / v0.4.0 / v0.5.0] in earlier ROADMAP versions lives in the [`agent-skills-cli`](https://github.com/MauricioPerera/agent-skills-cli) repo and the wider ecosystem:

| Original ROADMAP item | Where it shipped |
|---|---|
| `agent-skills sync / query / exec` | CLI v0.2.0 → v0.3.0 |
| `agent-skills publish` (validate + tag) | CLI v0.8.0 (signed-tag support v0.10.0) |
| `agent-skills init` (scaffold a pack) | CLI v0.9.0 |
| `agent-skills bench` (retrieval eval) | CLI v0.7.0 |
| `agent-skills update` (refresh + GC orphans) | CLI v0.11.0 |
| Multi-provider embedding integration | CLI v0.6.0 (Cloudflare + Ollama + OpenAI-compatible) |
| GPG-signed tag verification | CLI v0.10.0 (GitHub API path; client-GPG path tracked for v0.12+) |
| Per-tenant audit / rerank | tracked for spec v0.3 + CLI v0.12+ |
| Sigstore + Rekor integration | tracked for spec v0.3 + CLI v0.13+ |
| Aggregator pattern / discovery UX | tracked for spec v0.3 |

The reference CLI's BENCHMARK.md provides the empirical numbers that back §4.3.1 in this spec.

## v0.3.0 — Per-tenant + Sigstore + aggregators

Goal: complete the trust story (Sigstore + Rekor, audit log isolation in multi-tenant deploys) and solve discovery.

Open questions:
- [ ] Per-tenant audit scoping: `audit.jsonl` schema extension or per-tenant directory? Reference CLI tracking this for v0.12.
- [ ] Sigstore + Rekor: spec-side requirement language; reference CLI tracking for v0.13.
- [ ] Aggregator pattern: solve "how does a user discover skills?" without a central registry. GitHub Pages site that crawls the `agent-skills` topic weekly is one candidate.

## v1.0.0 — Stable spec

Goal: declare the schema, identity, and protocol stable. Commit to strict semver.

Criteria for v1.0.0:
- At least 3 independent skill banks have implemented v0.x and reported back.
- At least 50 publishers have published `agent-skills`-tagged repos.
- The reference implementation has been stable for 3 months.
- A published security audit (third-party review) has been resolved.
- Interop testing across implementations passes.

## v1.x — Open extensions

Things that **could** land in 1.x without breaking the spec:

- [ ] **Conditional chains** (`chains` with branching based on previous step output).
- [ ] **Per-skill rate limiting** declared by the publisher.
- [ ] **i18n**: skills with multilingual `use_when` (one skill, N translations vs N separate skills).
- [ ] **Capability negotiation**: skills declare which features they need (e.g., "needs IVF index"); banks reject if absent.
- [ ] **Multi-modal skills**: skills that accept image inputs, audio, etc. (probably out of scope; current model is text + JSON only).

## Deliberately out of scope (won't fix)

These have been considered and **excluded**:

| Feature | Why excluded |
|---|---|
| Built-in centralized registry | Decentralization is a core design goal |
| Streaming responses | Adds protocol complexity; MCP fits this niche |
| Persistent sessions / connections | Stateless model is intentional |
| Binary skill distribution | Skills should depend on standard tools; binaries are out of scope |
| Mandatory telemetry | Privacy invariant P3 forbids it |
| Auto-update by default | Operator-driven sync is a security property |
| "Smart" command_template (Turing-complete templating) | YAGNI; bash already does this |

## Open questions

These are unresolved. PRs welcome.

### Q1: How do skills handle interactive prompts?

A skill that needs to ask the user "are you sure?" before doing something destructive currently has no way to do so. Options:
- Skill metadata declares `requires_confirmation: true`; the bank handles the prompt.
- The skill embeds a confirmation step in `command_template` (gross).
- Out of scope: skills should be non-interactive.

Current leaning: **out of scope for v1**. Confirmation is the agent's responsibility, not the skill's.

### Q2: How do skills handle large outputs?

A skill that returns 100MB of data (e.g., a database export) violates the buffered-output assumption. Options:
- Skills declare `output_size: "large"`; banks pipe to a file instead of stdout.
- Skills always write to a file; return only the path.
- Out of scope: skills must produce reasonable-sized output.

Current leaning: **convention over enforcement**. Document a "large output" pattern (write to `$AGENT_SCRATCH/<uuid>`, return path) but don't bake it into the schema.

### Q3: Should skills be cryptographically self-signed (without external CA)?

Some publishers may not want to use Sigstore/GPG. Options:
- A skill author can include `signature: <ed25519>` in frontmatter.
- The bank verifies the signature against a fingerprint declared at the publisher's `.well-known/`.
- Lower bar than GPG, but still cryptographic.

Current leaning: **defer to v0.4.0 trust + signing**. Decide based on community demand.

### Q4: What about non-git source code hosts?

Mercurial, Fossil, Bazaar, etc. Options:
- Spec is git-specific (`source: "git"` only).
- Spec is VCS-agnostic; "source" is opaque, identity is just a URL + content hash.

Current leaning: **git-specific in v1**, with a generic content-hashed escape hatch (`source: "url-hash", content_hash: "sha256:..."`). Most non-git VCSs can publish via `git svn` / mirrors anyway.

### Q5: How does a consumer know when a SHA-pinned skill is "outdated"?

**Resolved in v0.1.1**: `SPEC.md` §7.2 defines re-sync behavior with `auto_update` flag. `SPEC.md` §7.4 introduces "compare against last-approved baseline" so multiple intermediate auto-syncs don't hide cumulative drift. Banks notify operator on new versions; auto-update is opt-in.

### Q6: Skill private packages

Internal-only skills that can't go on public GitHub. Options:
- Use private GitHub repos (auth via PAT in the bank's config).
- Use self-hosted git (GitLab, Forgejo, etc.).
- Use a private CDN.

Current leaning: **all of the above are valid**. The spec doesn't care about transport mechanism; HTTPS + content-hash works on private hosts the same as public.

### Q7: Skill bundling (multiple skills in one repo vs one repo per skill)

Currently the spec assumes one repo can host many skills (each in its own subdirectory). Should it also support per-skill repos? Yes — the spec doesn't forbid it; `skills-index.json` simply lists one skill in that case. No spec change needed.

## Contributing

This is an open spec. The contribution model:

1. Open an issue first for major changes (schema additions, removed fields).
2. PRs against `SPEC.md` should bump `schema_version` if adding fields.
3. PRs adding examples to `examples/` are always welcome.
4. PRs against `DESIGN.md` should add a new "D-N" entry rather than edit existing ones.

The spec lives at `github.com/<TBD>/agent-skills`. The repo URL will be set when the spec is published publicly.

## Versioning policy

The spec itself follows semver:

- **PATCH**: documentation fixes, clarifications, typo fixes.
- **MINOR**: additive fields, optional new sections, new conformance levels.
- **MAJOR**: removed or renamed fields, changed required-field semantics.

Implementations declare the spec version they support; banks may accept a range.
