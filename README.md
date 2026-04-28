# agent-skills

> An open, decentralized **specification** for distributing tools to LLM agents — an alternative to MCP that is **token-efficient**, **transparent**, **immutable**, and **web-native**.

> **Empirical proof of concept (live Cloudflare Workers AI, 35 paraphrased intents × 7 skills)**:
> - **Cosine baseline**: 34/35 = **97.1 %** top-1, 100 % top-3.
> - **v0.4.0 global rerank under stress** (50 concentrated past uses on one skill): collapses to **34.3 %** top-1 — the boost overwhelms cosine. Honestly documented as a failure mode of naive global-count rerank.
> - **v0.5.0 intent-conditional rerank** (sim ≥ 0.7 filter): **35/35 = 100 %** top-1 *under the same stress that broke global rerank*. The audit log only contributes a boost when its past intents are semantically similar to the current query.
>
> Full methodology, all 5 strategies compared, raw run JSON: [BENCHMARK.md](https://github.com/MauricioPerera/agent-skills-cli/blob/main/BENCHMARK.md).

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

You want to publish your tool as agent-skills. With [`agent-skills-cli`](https://github.com/MauricioPerera/agent-skills-cli) v0.9.0+:

```bash
# 1. Scaffold a complete pack (skills/, llms.txt, README, CI workflow).
agent-skills init my-pack --pack --author "Your Name"
cd my-pack

# 2. Edit the scaffolded skills/hello-world/SKILL.md, then:
agent-skills publish --check-only   # validate everything

# 3. Initial commit + signed release tag.
git init && git add . && git commit -m "initial release"
agent-skills publish --tag v1.0.0 --sign  # generates skills-index.json + signed tag
gh repo create my-pack --public --topic agent-skills
git push --follow-tags
```

That's it — your pack is now discoverable via:
- Direct URL: `cdn.jsdelivr.net/gh/<you>/my-pack@v1.0.0/skills/hello-world/SKILL.md`
- GitHub topic: `agent-skills`

The scaffolded `SKILL.md` includes every optional frontmatter field commented with a short explanation, so authors can discover the spec by editing the scaffold rather than reading this document end-to-end.

If you prefer to scaffold by hand without the CLI, the [`examples/skills/`](./examples/skills/) directory has two complete `SKILL.md` files to copy from.

That's it. Your skill pack is now discoverable via:
- Direct URL: `cdn.jsdelivr.net/gh/<you>/my-agent-skills@v1.0.0/skills/my-tool/SKILL.md`
- GitHub topic: `agent-skills`
- Pull requests from the world

## Two independent implementations

The spec is **specified** (not just documented) — two implementations satisfy it with bit-identical retrieval behaviour:

| Implementation | Language | Scope | Use it for |
|---|---|---|---|
| [`agent-skills-cli`](https://github.com/MauricioPerera/agent-skills-cli) | TypeScript | full agent loop (validate, resolve, sync, query, exec, audit, bench, publish, init, update, signature verification) | production |
| [`agent-skills-py-proof`](https://github.com/MauricioPerera/agent-skills-py-proof) | Python (510 LOC, single file) | retrieval only (parse, sync, query, bench) | reading the spec; reference for porting to a third language |

Both produce **identical scores to 4 decimal places** on the canonical benchmark (34/35 top-1 = 97.1 %, 35/35 top-3 = 100 %, mean margin +0.175, identical sole failure on the same paraphrase). If a third implementation produces different numbers on the same setup, either it's doing something different or it has found a spec gap — that's the point of having two.

The parity is **continuously validated** by [`agent-skills-cli`'s `e2e.yml` workflow](https://github.com/MauricioPerera/agent-skills-cli/actions/workflows/e2e.yml), which runs both implementations against the same Ollama setup on every push and weekly via cron, and fails if their numerical results ever diverge.

## Sister projects

- [`agent-skills-cli`](https://github.com/MauricioPerera/agent-skills-cli) — **reference TypeScript CLI**. Ships the full agent loop (validate, resolve, sync, query, exec, audit) with intent-conditional rerank by default; `bench` (v0.7.0+) for reproducible retrieval evaluation; `publish` (v0.8.0+) for pack authors; `init` (v0.9.0+) to scaffold a new pack; `update` (v0.11.0+) to refresh subscribed packs + GC orphans; signed-tag verification (v0.10.0+). Multi-provider embeddings: Cloudflare Workers AI, Ollama (local), OpenAI / OpenAI-compatible.
- [`agent-skills-py-proof`](https://github.com/MauricioPerera/agent-skills-py-proof) — **510-line Python proof** that the spec is sufficient for an independent implementation. Bit-identical retrieval scores to the TS CLI on the canonical benchmark.
- [`agent-skills-pack`](https://github.com/MauricioPerera/agent-skills-pack) — **example skill pack** with 7 production-ready skills (HTTP, GitHub CLI, ripgrep, jq, base64, …). Each demonstrates a different pattern from this spec; intended as a copy-paste-and-fork baseline for new pack authors.
- [`just-bash-data`](https://github.com/MauricioPerera/just-bash-data) — **storage runtime** providing the `db` (document store) and `vec` (vector search) primitives a conformant skill bank needs. The reference CLI's future `sync` / `query` / `exec` commands integrate with this.

## Status

**v0.2.0 — draft.** Additive update to v0.1.1. **Schema version remains `"0.1"`** (no SKILL.md changes). v0.2 formalises three patterns that emerged from the reference-CLI implementation cycle:

- **Rerank patterns** (§4.3.1): global vs intent-conditional, with empirical failure modes documented.
- **Bench protocol** (§4.6): `bench-truth.jsonl` format for reproducible retrieval evaluation.
- **Signature verification trust split** (§5.1, §5.3): Level 3a (host-verified, e.g. GitHub) vs Level 3b (client-verified, `trusted_keys`).

Plus an informative §4.7 documenting the embedding-provider abstraction (name, dim, embed) and three reference provider classes (Cloudflare Workers AI, Ollama, OpenAI-compatible).

The reference primitives (`db` + `vec` + encryption + IVF) are stable in [`just-bash-data@1.1.0`](https://www.npmjs.com/package/just-bash-data); the spec on top of them is what this repo defines.

See [`ROADMAP.md`](./ROADMAP.md) for what's planned and [`CHANGELOG.md`](./CHANGELOG.md) for how the spec evolves.

## License

[MIT](./LICENSE) for both spec text and example artifacts. Implementations choose their own.
