# Changelog

All notable changes to the `agent-skills` specification are documented here. The format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/) and the spec adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [0.2.0] — 2026-04-28

Additive specification update. **Schema version remains `"0.1"`** (no SKILL.md format changes); this document is bumped 0.1.1 → 0.2.0 to formalise patterns that emerged from the reference CLI's v0.5.0–v0.11.0 implementation cycle. Existing v0.1.x banks and packs remain conformant.

### New normative sections

- **§4.3.1 Rerank patterns**. Describes the **global** and **intent-conditional** rerank algorithms, with their formulas, the empirical failure mode of global rerank under usage concentration (50 concentrated past uses → top-1 collapses from 97% to 34% on the reference 7-skill / 35-paraphrase corpus), and the recovery profile of intent-conditional (100% top-1 under the same scenario via a `cos(query, past_intent) ≥ threshold` filter). Suggested defaults: `α=0.05`, `β=0.03`, `threshold=0.7`. Banks SHOULD expose the choice to operators and SHOULD support a no-rerank mode.
- **§4.5 audit `intent` field clarification**. The `intent` field was already in §4.5; v0.2 makes its role in intent-conditional rerank explicit and requires banks supporting that pattern to persist it.
- **§4.6 Bench protocol**. Standardises the `bench-truth.jsonl` format (JSONL or JSON-array, auto-detected, `{intent, expected}` pairs where `expected` is the **short** skill id for portability). Optional but recommended placement at pack root; an executable convention for measuring retrieval quality. CI integration via non-zero exit on any failure.
- **§5.1 Level 3 split into 3a (host-verified) and 3b (client-verified)**. Reflects the real operator trade-off between zero-burden trust delegation (3a) and host-independent verification (3b). Documented because the reference CLI implements 3a; v0.11.0+ pre-stages 3b for v0.12+.
- **§5.3 Verification trust trade-offs**. Explicit table of who's in the trust path under each Level, what each level catches and misses, and recommended posture per deployment type. Makes the cost of choosing a level visible rather than papering over it.

### New informative sections

- **§4.7 Embedding provider abstraction**. Documents the `(name, dim, embed)` triplet that all known providers expose, the requirement that `name` be persisted with the bank's index (so mixing models is detected at load time), and the three reference provider classes (Cloudflare Workers AI, Ollama, OpenAI-compatible `/v1/embeddings`). The provider choice is a local trust decision; the spec only requires consistency.

### Status

- v0.2 of this document is **back-compatible** with v0.1.1. Implementations that conform to v0.1.1 MUST remain conformant under v0.2 by ignoring the additions. The reverse is not guaranteed: a bank that doesn't implement intent-conditional rerank will produce different retrieval results than one that does.
- The reference CLI [`agent-skills-cli`](https://github.com/MauricioPerera/agent-skills-cli) v0.11.0+ implements every concrete pattern described in v0.2 of this document. The empirical numbers cited in §4.3.1 (97% → 34% under stress, 100% recovery) come from its [BENCHMARK.md](https://github.com/MauricioPerera/agent-skills-cli/blob/main/BENCHMARK.md) on live Cloudflare Workers AI.

### Why no schema bump

A schema-version bump would be required if v0.2 changed the SKILL.md format. It doesn't — every new section either (a) describes bank-internal behaviour (§4.3.1, §4.6, §4.7), (b) clarifies an existing field (§4.5), or (c) refines an existing trust level into sub-levels with the same enumeration values (§5.1). Authors of v0.1.x SKILL.md files have nothing to update.

## [0.1.1] — 2026-04-28

Substantial rewrite of the v0.1.0 draft after a critical self-review identified 6 critical issues, 10 significant issues, and several missing sections. Schema version remains `"0.1"` (no skill-author-visible breaks); spec document is bumped from 0.1.0 → 0.1.1.

### Critical fixes

- **C1**: `provenance` fields are now bank-managed metadata computed at ingest time, NOT author-declared in `SKILL.md`. Resolves the chicken-and-egg of "the SHA depends on the file, the file would have to know its SHA". §2.5 rewritten; D11 in DESIGN.md corrected.
- **C2**: identity → URL derivation now formally specified. `SPEC.md` §1.1 lists URL templates for known git hosts (GitHub, GitLab, Bitbucket) and requires self-hosted providers to declare a `url_template` in `skills-index.json`.
- **C3**: substitution rule rewritten. Placeholders MUST appear in argument position (not inside literal `"..."` or `'...'`). Banks SHOULD reject non-conformant templates at ingest. New decision D16 documents the rationale.
- **C4**: `args.<name>.unquoted` and `args.<name>.sensitive` now formally listed in §2.6 (were referenced but undeclared in v0.1.0).
- **C5**: embedding model truncation policy formalized. `SPEC.md` §4.2 prescribes a LIFO drop order (tags → trailing examples → trailing description) with a `provenance.embedding_truncated` flag.
- **C6**: shell semantics now explicit. Skills target bash 4.0+ unless `shell:` declares otherwise. New decision D17.

### Significant fixes

- **S1**: SHA-256 git transition supported. `<ref>` accepts ≥ 40 hex characters (40 for SHA-1, 64 for SHA-256).
- **S2**: examples indexing strategy specified (single concatenated text, examples joined with `\n` between intents).
- **S3**: `network` allowlist syntax formalized as URL-prefix matching with single trailing `*` and host-label `*`. No glob, no regex.
- **S4**: `args.type` extended to include `array` and `object`. Recursive sub-schemas via `items` / `properties`.
- **S5**: chains placeholder syntax unified to `{name}` (was a draft inconsistency between `{name}` and `$NAME`).
- **S6**: spec text is now implementation-agnostic. References to `db` / `vec` collections removed from `SPEC.md` and migrated to the new `IMPLEMENTATION.md`.
- **S7**: example provenance no longer uses null SHA placeholder (it's now computed at ingest, not in the file).
- **S8**: conformance levels clarified. B1 ingests A1+ skills; B2 adds Level 2+ provenance + sandbox + audit + applicable_when filtering.
- **S9**: `COMPARISON.md` token math redone with explicit axes (catalog size × tasks × queries-per-task), real numbers, multiple realistic scenarios.
- **S10**: substitution metacharacter handling switched from denylist to required pattern allowlist. `unquoted: true` requires a strict `pattern`; banks MUST refuse skills that violate this.

### New artifacts

- **`schemas/skill.schema.json`**: JSON Schema (Draft 2020-12) for SKILL.md frontmatter validation.
- **`IMPLEMENTATION.md`**: Reference implementation guide using `just-bash-data` v1.1.0+. All bank-specific code examples (sync daemon, query, exec) live here, separate from the canonical spec.
- **DESIGN.md additions**: D16 (substitution rule), D17 (shell semantics), and "Architectural caveats" section (A1–A4) recording trade-offs the design accepts honestly.

### Honest concession

- **Privacy invariant P3** reworded: no longer claims "zero telemetry" but precisely "no first-party telemetry to skill providers". Transport-layer observers (CDNs, git hosts, embedding APIs) see traffic regardless; that is outside the spec's reach.

### Spec evolution policy

`SPEC.md` §11 now defines:
- The spec document version (semver of the document).
- The schema version embedded in `SKILL.md` files (independent, only bumped on additive/breaking schema changes).
- Field deprecation cycle: 2 MINOR bumps before removal in a MAJOR.

### Compatibility

This is a draft → draft revision. Producers of v0.1.0 SKILL.md files MUST update `command_template`s if their placeholders sit inside literal quotes (per C3). All other v0.1.0 fields are accepted unchanged.

## [0.1.0] — 2026-04-28

### Added

Initial draft specification.

#### Core spec
- `SKILL.md` schema: YAML frontmatter + markdown body.
- Required fields: `schema_version`, `id`, `version`, `title`, `description`, `use_when`, `command_template`.
- Recommended fields: `license`, `author`, `homepage`, `category`, `tags`, `args`, `examples`.
- Optional fields: `required_commands`, `required_env`, `network`, `applicable_when`, `deprecates`, `migration_notes`, `related`, `chains`.
- Provenance fields (auto-populated): `provenance.source`, `provenance.commit`, `provenance.tag`, `provenance.signed_by`, etc.
- Skill identity scheme: `<source>@<version-or-sha>/<path>`.

#### Distribution protocol
- `/llms.txt` extension for human-readable provider index.
- `/skills-index.json` machine-readable manifest format.
- Sync protocol for git-source + CDN distribution.
- SHA pinning as canonical immutability mechanism.
- Subscription record schema with version + SHA pinning.

#### Trust model
- Five conformance levels (A1, A2, A3, B1, B2).
- Five provenance verification levels (0–4).
- Trusted-key management contract.
- Threat model covering author malice, account compromise, CDN compromise, typosquatting, command injection, query leak, skill bank compromise.

#### Privacy invariants
- P1: credentials never enter LLM context.
- P2: skill content is content-addressable.
- P3: sync is one-way (provider → consumer); no telemetry.
- P4: provider downtime does not break consumers.
- P5: audit trail is structurally complete via git history.

### Reference artifacts
- `examples/llms.txt` — top-level provider discovery.
- `examples/skills-index.json` — machine-readable manifest.
- `examples/skills/placeholder-img/SKILL.md` — simple no-auth skill.
- `examples/skills/charge-customer/SKILL.md` — credential-isolated skill with provenance.

### Documentation
- `README.md` — pitch, ToC, quick-start.
- `SPEC.md` — canonical specification.
- `SECURITY.md` — threat model and mitigations.
- `COMPARISON.md` — vs MCP, vs npm packs, vs centralized registries.
- `DESIGN.md` — 15 architectural decisions with rationale.
- `ROADMAP.md` — versioning, planned work, open questions.

### Status

**Draft**. Schema, field names, and protocol details are subject to change. v1.0.0 will be tagged once:
- 3+ independent skill bank implementations exist.
- 50+ public skill packs exist.
- Reference implementation has been stable for 3 months.
- A third-party security audit has been resolved.

[0.2.0]: https://github.com/MauricioPerera/agent-skills/releases/tag/v0.2.0
[0.1.1]: https://github.com/MauricioPerera/agent-skills/releases/tag/v0.1.1
[0.1.0]: https://github.com/MauricioPerera/agent-skills/releases/tag/v0.1.0
