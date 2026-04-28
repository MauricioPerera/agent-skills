# Changelog

All notable changes to the `agent-skills` specification are documented here. The format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/) and the spec adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

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

[0.1.1]: https://github.com/MauricioPerera/agent-skills/releases/tag/v0.1.1
[0.1.0]: https://github.com/MauricioPerera/agent-skills/releases/tag/v0.1.0
