# Changelog

All notable changes to the `agent-skills` specification are documented here. The format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/) and the spec adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

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

[0.1.0]: https://github.com/MauricioPerera/agent-skills/releases/tag/v0.1.0
