# ROADMAP — agent-skills

This document tracks **what's planned, what's open, and what's deliberately out of scope**. The spec evolves in the open via PRs against this file and the canonical `SPEC.md`.

## Current state — v0.1.0 (draft)

Shipped:
- Canonical `SKILL.md` schema (frontmatter + body).
- `/llms.txt` extension and `skills-index.json` machine-readable manifest.
- Sync protocol for git+CDN skill distribution.
- Threat model and conformance levels.
- Privacy invariants (one-way sync, credential isolation).
- Two reference examples (`placeholder-img`, `charge-customer`).
- Comparison vs MCP and npm packs.

Status: **draft**. Schema is unstable. Field names may change. Implementations should treat v0.1.0 as exploratory.

## v0.2.0 — Reference implementation

Goal: prove the spec runs end-to-end with real components.

Tasks:
- [ ] CLI tool: `agent-skills sync` — reads `skill_subscriptions`, fetches, validates, embeds, indexes.
- [ ] CLI tool: `agent-skills query <intent>` — embed query, return top-K skills with metadata.
- [ ] CLI tool: `agent-skills exec <id> [args...]` — execute a skill with substituted args.
- [ ] CLI tool: `agent-skills publish` — validate a `SKILL.md`, stamp provenance, prepare commit.
- [ ] Reference embedding integration: BGE-M3 via Ollama (default) + OpenAI text-embedding-3-small.
- [ ] Test suite: validate the spec against the reference impl + the example skills.

Deliverable: a npm package `@agent-skills/cli` and a working demo.

## v0.3.0 — Aggregator pattern

Goal: solve the "how does a user discover skills?" problem without inventing a registry.

Tasks:
- [ ] Define an aggregator's responsibilities (crawl public GitHub topics, validate skills, surface metadata).
- [ ] Reference aggregator implementation (could be a static GitHub Pages site that crawls weekly).
- [ ] Standardize a "skill rating" schema (community signals: stars, install counts, etc.).
- [ ] Document multiple aggregators coexisting (no central authority).

## v0.4.0 — Trust + signing

Goal: bring conformance level A3 (verified publisher) within reach.

Tasks:
- [ ] Reference implementation of GPG-signed tag verification.
- [ ] Reference implementation of Sigstore + Rekor integration.
- [ ] Trusted-key management UX (import, fingerprint display, rotation).
- [ ] Document key publication conventions (e.g., `.well-known/agent-skills-key.asc`).

## v0.5.0 — Enrichment + scoring

Goal: make retrieval better than naive nearest-neighbor.

Tasks:
- [ ] Hybrid retrieval (vector + keyword + tag filtering).
- [ ] Per-user feedback signals (`usage_count`, `avg_rating`, `last_used`).
- [ ] Re-ranking with `provenance.publisher_verified` weighting.
- [ ] Cold-start: how does a new skill bank with no audit history rank skills?

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

If you pin to SHA `abc123` and the publisher releases new commits on top, the consumer's pinned skill is "behind". The spec doesn't say how to detect this. Options:
- The bank periodically resolves the latest tag; if new, notify the operator.
- Operators must manually check.
- Aggregators publish "newer-than-X" feeds.

Current leaning: **the bank's sync daemon checks for updates without auto-applying**. UI surfaces "5 of your subscriptions have updates available".

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
