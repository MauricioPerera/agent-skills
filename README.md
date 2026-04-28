# agent-skills

> An open, decentralized specification for distributing tools to LLM agents — an alternative to MCP that is **token-efficient**, **transparent**, **immutable**, and **web-native**.

## The thesis

LLM agents today learn tools via **injection**: each tool's name, description, and JSON schema is loaded into the system prompt at session start. This works for small catalogs but breaks down when agents need access to dozens or hundreds of tools — the context window fills with tool definitions before the user asks anything.

`agent-skills` proposes a different model: **retrieval over injection**. Tools are described as `SKILL.md` files hosted at stable URLs (typically a git repository served via a CDN). The agent learns *one* convention — how to query its local skill bank — and discovers tools on demand via vector search. Token cost stays roughly constant regardless of catalog size.

The same convention also gives us:

- **Credential isolation**: secrets never enter the LLM's context. Skill commands reference environment variables that only the shell sees.
- **Transparency**: every skill is a markdown file in a public git repo. Users can read, fork, modify, and audit.
- **Cryptographic provenance**: skills are pinned to git commit SHAs. Changes are traceable, signable, and immutable.
- **Decentralization**: no central registry. GitHub, GitLab, or self-hosted git plus a CDN are the only infrastructure required.
- **Composability**: skills can chain other skills by SHA reference. Recipes become first-class artifacts.

## Architecture in one diagram

```
┌─────────────────────────────────────────────────────────────────┐
│  Layer 1 — Skill providers                                      │
│                                                                 │
│  github.com/stripe/agent-skills @v1.2.0                         │
│  github.com/openai/agent-skills @v3.4.5                         │
│  gitlab.com/yourcompany/internal-skills @main                   │
└─────────────────────────────────────────────────────────────────┘
                             ↓
              git clone / cdn.jsdelivr.net@SHA
                             ↓
┌─────────────────────────────────────────────────────────────────┐
│  Layer 2 — Discovery                                            │
│                                                                 │
│  - GitHub topic: `agent-skills`                                 │
│  - Awesome lists (curated)                                      │
│  - Aggregator sites (optional, multiple competing)              │
└─────────────────────────────────────────────────────────────────┘
                             ↓
                  User picks repo + version
                             ↓
┌─────────────────────────────────────────────────────────────────┐
│  Layer 3 — Local skill bank (e.g., just-bash-data)              │
│                                                                 │
│  db skill_subscriptions     → which repos + SHAs                │
│  db skills                  → indexed metadata                  │
│  vec skills                 → embeddings (local model)          │
│  db skill_audit             → usage + ratings + feedback        │
└─────────────────────────────────────────────────────────────────┘
                             ↓
                       Query / execute
                             ↓
┌─────────────────────────────────────────────────────────────────┐
│  Layer 4 — Agent runtime (any shell-capable runtime)            │
│                                                                 │
│  1. Embed user intent                                           │
│  2. vec search skills                                           │
│  3. db skills find (top hits)                                   │
│  4. Pick + execute command_template                             │
│  5. db skill_audit insert (feedback loop)                       │
└─────────────────────────────────────────────────────────────────┘
```

## Document map

This repository is **a specification + reference artifacts**, not (yet) a runtime implementation. The runtime is delegated to [`just-bash-data`](https://www.npmjs.com/package/just-bash-data), which already provides the `db` + `vec` primitives this spec relies on.

| File | Role |
|---|---|
| [`SPEC.md`](./SPEC.md) | Canonical specification — `SKILL.md` schema, `/llms.txt` extension, sync protocol, identity |
| [`SECURITY.md`](./SECURITY.md) | Threat model, provenance, signing, sandbox boundaries |
| [`COMPARISON.md`](./COMPARISON.md) | Detailed comparison vs MCP, vs npm skill packs, vs registry-based approaches |
| [`DESIGN.md`](./DESIGN.md) | Architectural decisions and rationale |
| [`ROADMAP.md`](./ROADMAP.md) | Spec versioning, future work, open questions |
| [`CHANGELOG.md`](./CHANGELOG.md) | Spec version history (semver) |
| [`examples/llms.txt`](./examples/llms.txt) | Example top-level discovery file |
| [`examples/skills/`](./examples/skills/) | Two complete `SKILL.md` examples (placeholder-img, charge-customer) |

## Quick start (consumer side)

You have an agent that runs in [`just-bash-data`](https://www.npmjs.com/package/just-bash-data). You want to install Stripe's agent skills:

```bash
# 1. Subscribe to a skill pack (pinned to a SHA for immutability)
db skill_subscriptions insert '{
  "_id": "stripe-skills",
  "source": "git",
  "repo": "github.com/stripe/agent-skills",
  "version_pin": "v1.2.0",
  "sha_pin": "a1b2c3d4e5f67890abcdef1234567890abcdef12",
  "cdn_base": "https://cdn.jsdelivr.net/gh/stripe/agent-skills@a1b2c3d4e5f67890abcdef1234567890abcdef12"
}'

# 2. Run the sync daemon (manual or cron)
sync-skills.sh

# 3. The agent finds and executes
QEMB=$(curl -s "$EMB_API" -d "{\"input\":\"charge a customer for $100\"}" | jq '.data[0].embedding')
TOP=$(vec search skills "$QEMB" --k 5 | jq -r '.[0].id')
db skills find "{\"_id\":\"$TOP\"}" | jq '.[0].command_template'
# → "stripe charges create --amount {amount} --currency {currency} --customer {customer_id}"
```

The skill itself never reaches Stripe's secret key into the LLM's context. The LLM emits the templated command; the local shell substitutes `$STRIPE_SECRET_KEY` at exec time.

## Quick start (publisher side)

You want to publish your tool as agent-skills:

```bash
mkdir my-agent-skills && cd my-agent-skills
mkdir skills && touch skills/my-tool/SKILL.md skills/my-tool/SKILL.md

# Edit skills/my-tool/SKILL.md (see examples/ for the format)
# Add a top-level llms.txt and skills-index.json (see examples/)

git init && git add . && git commit -m "initial release"
git tag v1.0.0
gh repo create my-agent-skills --public
git push --tags
```

That's it. Your skill pack is now discoverable via:
- Direct URL: `cdn.jsdelivr.net/gh/<you>/my-agent-skills@v1.0.0/skills/my-tool/SKILL.md`
- GitHub topic: `agent-skills` (apply via `gh repo edit --add-topic agent-skills`)
- The world's pull request system

## Status

**This is v0.1.0 — a draft specification**. Nothing here is final. Schema, protocol, and naming are open for iteration. The reference primitives (`db` + `vec`) are stable in [`just-bash-data@1.1.0`](https://www.npmjs.com/package/just-bash-data); the *spec on top of them* is what this repo defines.

See [`ROADMAP.md`](./ROADMAP.md) for what's planned and [`CHANGELOG.md`](./CHANGELOG.md) for how the spec evolves.

## License

[MIT](./LICENSE) — for both the spec text and the example artifacts. Implementations choose their own.
