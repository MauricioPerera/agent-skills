# agent-skills

> An open, decentralized **specification** for distributing tools to LLM agents — an alternative to MCP that is **token-efficient**, **transparent**, **immutable**, and **web-native**.

> **Empirical proof of concept**: 7 / 7 agent intents retrieved their intended skill as top-1 from the public [`agent-skills-pack`](https://github.com/MauricioPerera/agent-skills-pack) using `@cf/baai/bge-base-en-v1.5` on Cloudflare Workers AI. Full benchmark: [agent-skills-cli/BENCHMARK.md](https://github.com/MauricioPerera/agent-skills-cli/blob/main/BENCHMARK.md).

This repository defines a **format and a protocol**. It does not include a runtime. Conformant skill banks can be built atop any sufficient infrastructure (filesystem + vector index + shell). One reference runtime — built on the parallel project [`just-bash-data`](https://github.com/MauricioPerera/just-bash-data) — is described in [`IMPLEMENTATION.md`](./IMPLEMENTATION.md), but the spec proper is implementation-agnostic.

## The thesis

LLM agents today learn tools via **injection**: each tool's name, description, and JSON schema is loaded into the system prompt at session start. This works for small catalogs but breaks down when agents need access to dozens or hundreds of tools — the context window fills with tool definitions before the user asks anything.

`agent-skills` proposes a different model: **retrieval over injection**. Tools are described as `SKILL.md` files hosted at stable URLs (typically a git repository served via a CDN). The agent learns *one* convention — how to query its local skill bank — and discovers tools on demand via vector search. Token cost stays roughly constant regardless of catalog size.

The same pattern gives us:

- **Credential isolation**: secrets never enter the LLM's context. Skill commands reference environment variables that only the shell sees.
- **Transparency**: every skill is a markdown file in a public git repo. Users can read, fork, modify, audit.
- **Cryptographic provenance**: skills are pinned to git commit hashes. Changes are traceable, signable, immutable.
- **Decentralization**: no central registry. GitHub, GitLab, or self-hosted git plus a CDN are the only infrastructure required.
- **Composability**: skills can chain other skills by hash reference. Recipes become first-class artifacts.

## Architecture in one diagram

```
┌─────────────────────────────────────────────────────────────────┐
│  Layer 1 — Skill providers (any git host)                       │
│                                                                 │
│  github.com/stripe/agent-skills @v1.2.0                         │
│  github.com/openai/agent-skills @v3.4.5                         │
│  gitlab.com/yourcompany/internal-skills @main                   │
└─────────────────────────────────────────────────────────────────┘
                             ↓
              git clone / cdn.jsdelivr.net@<commit-sha>
                             ↓
┌─────────────────────────────────────────────────────────────────┐
│  Layer 2 — Discovery                                            │
│                                                                 │
│  - GitHub topic: `agent-skills`                                 │
│  - Awesome lists (community-curated)                            │
│  - Aggregator sites (third-party, optional)                     │
└─────────────────────────────────────────────────────────────────┘
                             ↓
                  User picks repo + version
                             ↓
┌─────────────────────────────────────────────────────────────────┐
│  Layer 3 — Local skill bank (any conformant implementation)     │
│                                                                 │
│  subscriptions storage  → which repos + hashes                  │
│  skill index            → indexed metadata                      │
│  vector index           → embeddings (locally chosen model)     │
│  audit log              → usage + ratings + feedback            │
└─────────────────────────────────────────────────────────────────┘
                             ↓
                       Query / execute
                             ↓
┌─────────────────────────────────────────────────────────────────┐
│  Layer 4 — Agent runtime (any shell-capable runtime)            │
│                                                                 │
│  1. Embed user intent                                           │
│  2. Query skill bank → top-K                                    │
│  3. Fetch metadata for chosen skill                             │
│  4. Execute command_template with substituted args              │
│  5. Record audit + feedback                                     │
└─────────────────────────────────────────────────────────────────┘
```

## Document map

| File | Role |
|---|---|
| [`SPEC.md`](./SPEC.md) | **Canonical specification** — schema, identity, protocol, conformance |
| [`SECURITY.md`](./SECURITY.md) | Threat model, provenance, signing, sandbox boundaries |
| [`COMPARISON.md`](./COMPARISON.md) | Detailed comparison vs MCP, vs npm packs, vs registries; migration sketch |
| [`DESIGN.md`](./DESIGN.md) | Architectural decisions and rationale |
| [`IMPLEMENTATION.md`](./IMPLEMENTATION.md) | Reference skill bank built on `just-bash-data` (one of many possible) |
| [`ROADMAP.md`](./ROADMAP.md) | Spec versioning, future work, open questions |
| [`CHANGELOG.md`](./CHANGELOG.md) | Spec version history (semver) |
| [`schemas/skill.schema.json`](./schemas/skill.schema.json) | JSON Schema for `SKILL.md` frontmatter validation |
| [`examples/llms.txt`](./examples/llms.txt) | Example top-level discovery file |
| [`examples/skills/`](./examples/skills/) | Two complete `SKILL.md` examples |

## Relationship to `just-bash-data`

`agent-skills` is the **specification**. `just-bash-data` is **one possible reference runtime** that already implements the primitives (`db` for structured docs, `vec` for similarity search, encryption-at-rest, etc.) a conformant skill bank needs.

- The spec text is independent of any runtime.
- The reference implementation guide ([`IMPLEMENTATION.md`](./IMPLEMENTATION.md)) shows how to build a B1/B2-conformant skill bank atop just-bash-data v1.1.0+.
- Other implementations atop SQLite + sqlite-vss, Postgres + pgvector, Redis Stack, or in-memory stores are equally valid; the spec is what they must conform to.

The two repositories evolve in parallel:
- **`just-bash-data`** stabilizes and extends storage/retrieval primitives.
- **`agent-skills`** stabilizes the format/protocol on top.

## Quick start (consumer side, using just-bash-data)

```bash
# Install the runtime
npm i just-bash-data@1.1.0 just-bash

# Bootstrap (one-time)
db skill_subscriptions index create id --unique
db skills index create category --sorted
vec create skills --dim 1024 --quantize int8 --ivf-clusters 100

# Subscribe to a skill pack (pinned to a hash for immutability)
db skill_subscriptions insert '{
  "_id": "stripe-skills",
  "source_type": "git",
  "repo": "github.com/stripe/agent-skills",
  "ref_requested": "v1.2.0",
  "ref_resolved": "a1b2c3d4e5f67890abcdef1234567890abcdef12",
  "auto_update": false,
  "verify_signature": true,
  "trusted_keys": ["B5A4 9C28 D9F1 ..."]
}'

# Run the sync daemon (sketched in IMPLEMENTATION.md)
./sync-skills.sh

# The agent finds and executes
QEMB=$(curl -s "$EMB_API" -d "{\"input\":\"charge customer cus_X $50\"}" | jq '.data[0].embedding')
TOP=$(vec search skills "$QEMB" --k 5 | jq -r '.[0].id')
db skills find "{\"_id\":\"$TOP\"}" | jq '.[0] | {title, command_template, args}'
```

For a production-grade walk-through, see [`IMPLEMENTATION.md`](./IMPLEMENTATION.md).

## Quick start (publisher side)

You want to publish your tool as agent-skills:

```bash
mkdir my-agent-skills && cd my-agent-skills
mkdir skills/my-tool

# Edit skills/my-tool/SKILL.md — see examples/skills/ for the format
vim skills/my-tool/SKILL.md

# Add /llms.txt and /skills-index.json at repo root (see examples/)

# Validate against the JSON schema before committing:
yq -o=json '.' skills/my-tool/SKILL.md \
  | head -n -1 | tail -n +2 \
  | npx ajv validate -s https://raw.githubusercontent.com/MauricioPerera/agent-skills/v0.1.1/schemas/skill.schema.json --spec=draft2020

git init && git add . && git commit -m "initial release"
git tag -s v1.0.0   # signed tag for Level 3 conformance
gh repo create my-agent-skills --public --topic agent-skills
git push --tags
```

That's it. Your skill pack is now discoverable via:
- Direct URL: `cdn.jsdelivr.net/gh/<you>/my-agent-skills@v1.0.0/skills/my-tool/SKILL.md`
- GitHub topic: `agent-skills`
- Pull requests from the world

## Sister projects

- [`agent-skills-cli`](https://github.com/MauricioPerera/agent-skills-cli) — **reference CLI implementation**. Validates SKILL.md files against this spec and resolves command_template against arg values. v0.2.0-alpha shipped covering local-only operations (validate + resolve); v0.2.0 final adds sync + query + exec.
- [`agent-skills-pack`](https://github.com/MauricioPerera/agent-skills-pack) — **example skill pack** with 7 production-ready skills (HTTP, GitHub CLI, ripgrep, jq, base64, …). Each demonstrates a different pattern from this spec; intended as a copy-paste-and-fork baseline for new pack authors.
- [`just-bash-data`](https://github.com/MauricioPerera/just-bash-data) — **storage runtime** providing the `db` (document store) and `vec` (vector search) primitives a conformant skill bank needs. The reference CLI's future `sync` / `query` / `exec` commands integrate with this.

## Status

**v0.1.1 — draft.** Schema, protocol, and naming are open for iteration. The reference primitives (`db` + `vec` + encryption + IVF) are stable in [`just-bash-data@1.1.0`](https://www.npmjs.com/package/just-bash-data); the spec on top of them is what this repo defines.

See [`ROADMAP.md`](./ROADMAP.md) for what's planned and [`CHANGELOG.md`](./CHANGELOG.md) for how the spec evolves.

## License

[MIT](./LICENSE) for both spec text and example artifacts. Implementations choose their own.
