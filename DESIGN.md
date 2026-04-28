# DESIGN — architectural decisions

This document records the architectural decisions in `agent-skills` and the reasoning behind each. The intent is that future contributors can read this and understand **why** the spec looks the way it does, not just what it specifies.

Each decision is presented as: **decision → alternatives considered → rationale**.

---

## D1: Retrieval over injection

**Decision**: The agent learns *one* convention (how to query a skill bank) and discovers tools on demand via vector search. Tools are not loaded into the system prompt at session start.

**Alternatives considered**:
- **Inject all tool definitions** (MCP, OpenAI function calling): the LLM sees every tool's schema upfront.
- **Hybrid (cache top-N tools)**: pre-load the N most-used tools, retrieve the rest.
- **Tag-based filtering**: load only tools matching the user's declared "task category".

**Rationale**:
- Token cost in injection scales linearly with catalog size; retrieval is O(1).
- The agent doesn't know what tools it might need at session start. Just-in-time discovery matches user intent better.
- The LLM is an excellent intent-matcher; let it formulate queries, let infrastructure do retrieval. This mirrors the shift from "fine-tune knowledge" to "RAG".

**Cost paid**: per-task overhead (~150 tokens for retrieval), one external API call for query embedding (or local model).

---

## D2: Plain-text content-addressable distribution

**Decision**: Skills are markdown files in git repos, served via CDNs that respect content-addressable URLs (e.g., jsDelivr's `@<sha>` syntax).

**Alternatives considered**:
- **Centralized skill registry**: a `skills.io` that hosts everything.
- **npm packages**: skills as npm dependencies.
- **OCI artifacts**: skills as container-registry blobs.
- **Provider-hosted on own domain**: each provider serves `/skills/<name>/SKILL.md`.

**Rationale**:
- Git provides cryptographic content-addressability "for free" (every commit SHA hashes the content).
- jsDelivr provides global CDN caching with SHA-pinning support, also "for free".
- No vendor backend dependency: providers commit a markdown file, that's it.
- Decentralized: no single registry to fail or be compromised.
- Plain text: human-readable, diffable, AI-friendly, version-controllable.

**Cost paid**: providers must use git (universal), consumers must do HTTP fetches (universal).

---

## D3: YAML frontmatter + markdown body

**Decision**: `SKILL.md` files use YAML frontmatter for machine-readable fields and markdown body for human-readable prose.

**Alternatives considered**:
- **Pure JSON**: easier parsing, but ugly for humans and bad for prose-heavy fields.
- **Pure YAML**: parses cleanly but markdown is more familiar for documentation.
- **TOML frontmatter**: less common in the markdown ecosystem.
- **Multiple files**: `skill.json` + `README.md` separately.

**Rationale**:
- Markdown frontmatter is the dominant convention in static-site generators (Jekyll, Hugo, Astro, MDX). Skill authors already know the pattern.
- One file = one skill = one URL. Simpler mental model.
- YAML handles structured data cleanly; markdown handles prose. Best of both.
- The body is optional context for LLMs that need more than the frontmatter to understand a skill.

**Cost paid**: parsers must handle both YAML and markdown (gray-matter / yaml-front-matter libraries are mature).

---

## D4: Local embedding generation

**Decision**: Skill banks generate embeddings at ingest time using their locally configured model. Skills do not publish vectors.

**Alternatives considered**:
- **Publishers ship pre-computed embeddings**: requires global agreement on model.
- **Standardize one embedding model in the spec**: locks the ecosystem to a single vendor.
- **Multi-model with hash-based selection**: complexity nightmare.

**Rationale** (from earlier conversation):
- **Provider doesn't know which model the consumer uses**. If a publisher ships embeddings from `text-embedding-3-small` and a consumer runs `bge-m3`, the vector spaces are incompatible. Search returns garbage.
- **Models change; skill content persists**. Forcing a model freeze on the ecosystem is brittle.
- **Privacy**: if a query were embedded with a publisher-supplied model, intent would leak to the publisher. Local embeddings keep queries on the consumer's hardware.
- **Cost amortization**: publishers don't pay embedding costs; consumers pay only for skills they install. Per-publisher economics aren't externalized.

**Cost paid**: every consumer runs an embedding step on every ingested skill. With 1000 skills × 1ms locally, this is sub-second. With cloud APIs and large catalogs, this is a one-time cost.

---

## D5: Credential isolation via env substitution

**Decision**: `command_template` references environment variables (`$STRIPE_KEY`); the **shell** (not the LLM) substitutes values at execution time.

**Alternatives considered**:
- **Args carry credentials**: standard MCP approach. Credential value passes through LLM context.
- **Skill bank holds credentials, injects post-LLM**: requires the bank to know which arg is "secret". Schema complexity.
- **External credential broker**: a separate service holds creds, returns one-time tokens. Adds infrastructure.

**Rationale**:
- The LLM should never see secrets. Period. Once in the context, they're in the conversation log, possibly in upstream caches, possibly leakable via jailbreak.
- Bash already has this concept built-in via `$VAR` expansion. We didn't invent it; we honored it.
- Env vars + per-shell scope = natural multi-tenant isolation. Two users on the same machine each have their own `$STRIPE_KEY`.
- `required_env` declares the dependency without revealing the value. The agent can know "this skill needs `$AUTH_TOKEN`" and offer to log in if absent — without ever seeing the token.

**Cost paid**: skills that genuinely need to pass dynamic secrets (rare) must do it via stdin or a temp file rather than args. Acceptable trade-off.

---

## D6: SHA pinning as the canonical identity

**Decision**: A skill's canonical identity includes a 40-character git commit SHA. Tag pinning is allowed but resolved to a SHA at ingest time.

**Alternatives considered**:
- **Tag-only pinning**: simpler to read but mutable.
- **Semantic identity** (`stripe/charge-customer@1.2`): more user-friendly but requires registry.
- **URL-only**: simplest but no immutability.

**Rationale**:
- Immutability is the property that prevents silent updates. Without it, a provider can swap content under us.
- Tags are pointers; they can be moved (`git push --force`). SHAs cannot.
- Reading 40 hex characters is admittedly worse UX than `v1.2.0`. We accept this for safety. UI can hide SHAs and show the resolved tag for display.
- The spec explicitly requires "the bank resolves the tag to a SHA at ingest" — so the user writes `v1.2.0` but the bank stores the SHA.

**Cost paid**: identity strings are long. Display layers must abstract them.

---

## D7: No first-party telemetry (one-way sync)

**Decision**: The skill bank fetches from providers but never POSTs telemetry back. Providers receive zero **first-party** data about consumers.

**Alternatives considered**:
- **Optional opt-in telemetry**: providers might want install counts, error reports.
- **Anonymous aggregated telemetry**: a third-party broker collects + anonymizes.
- **Mandatory telemetry**: standard "phone home" pattern.

**Rationale**:
- Privacy is hard to retrofit. Default to zero first-party telemetry; let opt-in mechanisms layer on later if demanded.
- Providers can infer popularity from GitHub stars, npm downloads of related packages, etc. They don't need first-party data.
- Consumer privacy is a feature. Many enterprises explicitly need it for compliance.

**Cost paid**: providers don't have install counts. Tooling like aggregators can fill the gap optionally.

**Important caveat (added in v0.1.1)**: "no first-party" does NOT mean "no telemetry at all." Transport-layer observers see traffic regardless of the spec:
- The git host (GitHub, GitLab) logs every clone/fetch with IP + User-Agent.
- The CDN (jsDelivr, Cloudflare) logs every request.
- The embedding API (if remote) sees every query intent.

The spec's invariant is that **the skill author / provider does not receive bank-originated reports** — usage counts, ratings, errors, telemetry beacons. What network operators see at the transport layer is outside the spec's control. P3 of `SPEC.md` §8 was reworded to make this distinction explicit.

---

## D8: Sync is operator-driven, not auto-update by default

**Decision**: Subscriptions default to `auto_update: false`. New versions require operator approval.

**Alternatives considered**:
- **Auto-update by default**: latest = best.
- **Auto-update with rollback**: try new version, revert if errors spike.
- **Channel-based** (stable / beta / nightly): user picks risk level.

**Rationale**:
- Auto-update is hostile to security-conscious operators. Pushing new content into running systems without review is how supply chain attacks succeed.
- Operators who want auto-update can opt in per-subscription. Defaults are conservative.
- Channel-based releases can be implemented atop the basic primitive (subscriptions to specific tags or branches).

**Cost paid**: operators must periodically review and approve updates. UX work needed (a sync daemon that emails diffs, a TUI that walks operator through changes).

---

## D9: Skills depend on commands already in PATH

**Decision**: Skills do not bundle binaries. They reference standard tools (`curl`, `jq`, `gh`, `psql`, etc.) via `required_commands`.

**Alternatives considered**:
- **Bundle binaries**: skills include compiled helpers (Linux x86_64 only? Multi-arch?).
- **Bundle scripts**: skills include Python/Bash/Node scripts that run alongside.
- **Reference an external image** (Docker / OCI): heavy but portable.

**Rationale**:
- Skill files stay tiny (markdown only).
- Versioning binaries inside markdown files is awkward.
- The shell ecosystem already has tools for binary distribution (apt, brew, asdf, etc.). Don't reinvent.
- Skills declare their dependencies; the operator ensures the host has them. Standard separation.

**Cost paid**: skills can't run on hosts without the prerequisite tools. `applicable_when.shell_commands_present` lets the bank filter inapplicable skills.

---

## D10: Composition via `chains` referencing other skills by SHA

**Decision**: A skill can declare `chains` — a sequence of references to other skills (each with its own SHA pin). The bank resolves and executes the chain.

**Alternatives considered**:
- **No composition**: each skill is atomic; agents orchestrate.
- **Composition via plain bash pipes**: `command_template` is one big pipeline.
- **Composition via DAG language** (a la Airflow): higher expressivity, much more complexity.

**Rationale**:
- Pipeline-in-bash works for linear flows but breaks down for fan-out / fan-in.
- Cross-provider chains need referential identity, which `<provider>@<sha>/<skill>` provides.
- DAG languages are overkill for the current need. Linear chains cover most practical cases. If complexity grows, chains can be extended.
- Cycle detection is mandatory (otherwise a malicious skill could DoS the bank).

**Cost paid**: chains add executor complexity. Cycles, errors mid-chain, partial outputs all need careful handling.

---

## D11: Provenance is computed at ingest, not stored in the file

**Decision (revised in v0.1.1)**: `provenance` fields are computed by the **skill bank at ingest time** from the source's git metadata + signature data. They are NOT placed in the `SKILL.md` file by the author or by CI.

**Alternatives considered**:
- **Authors declare provenance manually**: error-prone, easy to forge, and impossible to do correctly (the SHA depends on the file content).
- **CI auto-populates after commit**: chicken-and-egg — stamping the SHA produces a new commit with a different SHA, which then needs another stamp, ad infinitum.
- **Two-commit dance**: commit with placeholder, then second commit replacing placeholder with previous SHA. Produces noisy history; the stamped SHA is one commit behind reality; fragile.
- **External provenance file alongside SKILL.md**: clutters the repo and still has chicken-and-egg with itself.
- **No provenance at all**: loses the property of "who published this and when."

**Rationale**:
- An author cannot reliably know their own commit SHA at the moment they write the file. The SHA is **derived** from the file content + commit metadata; it cannot be embedded in the content without a fixed-point operation that git does not provide.
- The skill bank already has access to authoritative provenance at ingest time (git API responses, HTTP headers, signature artifacts). Re-deriving from these sources is reliable and forgery-resistant.
- Authors writing provenance manually opens forgery — they could claim any SHA. Auto-population by the bank closes that hole.

**Cost paid**: provenance is not visible in the `SKILL.md` file's source. Authors who want to indicate "this skill expects to be served from `github.com/example/agent-skills`" can do so via a comment in the markdown body or via the `homepage` field; the spec does not provide a frontmatter field for this expressly.

**Implementation note**: §2.5 of `SPEC.md` was rewritten in v0.1.1 to clarify that `provenance` is bank-managed metadata, not author-declared content. Banks MUST compute it at ingest. Earlier draft examples that placed `provenance:` in the frontmatter were incorrect and have been removed.

---

## D12: Conformance levels (A1/A2/A3, B1/B2)

**Decision**: The spec defines progressive conformance levels rather than one all-or-nothing standard.

**Alternatives considered**:
- **Single conformance**: a skill either is or isn't valid.
- **Profile-based**: many independent profiles a skill can claim.

**Rationale**:
- Adoption is gradual. A solo developer should be able to publish a "Level A1" skill (just `SKILL.md`, no signing, no llms.txt). An enterprise should be able to require "Level A3" (signed + verified).
- Levels create natural maturity progression: A1 for prototypes, A3 for production.
- Consumers configure their `verify_level` per subscription, matching their trust model.

**Cost paid**: the spec is more complex to read. Compensated by clearer "what do I need to do" for each adoption level.

---

## D13: No streaming, no statefulness

**Decision**: Skills produce buffered stdout. No streaming, no persistent connections, no session state across executions.

**Alternatives considered**:
- **WebSocket-style streaming**: matches MCP capability.
- **HTTP/2 server-sent events**: lighter than WS.
- **Stateful via a session ID + sequential calls**: complex but flexible.

**Rationale**:
- Streaming and statefulness add significant complexity and weren't critical for the agent-shell use case targeted by `just-bash-data`.
- Most LLM tool needs are: "do X, return result". Buffered is fine.
- For tools that genuinely need streaming, MCP exists and remains a valid choice.
- Keeping the surface small allows the spec to be confidently fitted to ~100% of stateless tool use cases. Trying to cover streaming would balloon scope.

**Cost paid**: certain workflows (long-running queries, real-time logs) are not a fit. We're explicit about this in `COMPARISON.md`.

---

## D14: Spec versioning via `schema_version` field

**Decision**: Every `SKILL.md` declares the spec version it conforms to. Skill banks reject mismatched versions or apply migration shims.

**Alternatives considered**:
- **Implicit versioning**: assume latest, break old skills silently.
- **Backward compatibility forever**: never bump major version.
- **Multiple namespaces**: `agent-skills-v1`, `agent-skills-v2`.

**Rationale**:
- Schemas evolve. Explicitly declaring the version makes evolution safe.
- Banks can support multiple `schema_version`s simultaneously, applying transformations as needed.
- Authors don't have to migrate every skill on every spec bump; they migrate at their own pace.

**Cost paid**: spec maintainers must support old versions for some grace period. Standard semver-style discipline applies.

---

## D15: Default-rejection for unsigned skills in strict mode

**Decision**: A skill bank operating in `strict_mode` defaults to rejecting subscriptions without `verify_signature: true` + `trusted_keys`.

**Alternatives considered**:
- **Always lenient**: convenience first; security as opt-in.
- **Always strict**: security first; require crypto setup.

**Rationale**:
- Different deployments have different needs. Casual / development: lenient. Production / enterprise: strict.
- Strict mode forces deliberate trust decisions. Operators must import a key and assert "I trust this signer for this subscription".
- Lenient mode allows quick prototyping without ceremony.
- Default is lenient for the bank as a whole; strict is opt-in. But within strict mode, defaults flip — verification is the default, opt-out is explicit.

**Cost paid**: setup overhead for strict mode (key management, etc.). Reference tooling should make this easy.

---

## D16: Substitution rule — placeholders go in argument position, not inside literal quotes

**Decision (added v0.1.1)**: `{placeholder}` MUST occupy a shell argument position. The bank inserts the substituted value as a single, properly-escaped shell argument. Templates that embed placeholders inside literal `"..."` or `'...'` are non-conformant.

**Alternatives considered**:
- **Context-aware substitution**: detect surrounding quotes and adapt escaping. Implementations diverge; depends on parsing the template at substitution time.
- **No escaping (raw substitution)**: simplest but command injection guaranteed.
- **JSON-only substitutions** (e.g., `--data-raw '{json}'`): forces all skills into a JSON-shaped flow, awkward for many tools.

**Rationale**:
- A unified, simple rule (single-quoted substitution into argument position) sidesteps every quoting edge case.
- Skill authors learn one rule, applied everywhere.
- Banks parse templates trivially; no shell-aware parser needed.
- Conformance can be checked at ingest by detecting `{placeholder}` inside literal quote characters.

**Examples**:
```bash
# Conformant
curl -X POST https://api.example.com/v1/charges \
  -d amount={amount} \
  -d currency={currency} \
  -d customer={customer_id}

# Non-conformant (bank rejects at ingest)
curl -X POST "https://api.example.com/v1/{path}" \
  -d "amount={amount}"
```

**Cost paid**: skill authors must re-learn standard shell argument quoting (which is a habit anyway). The first batch of v0.1.0 examples were rewritten to comply.

---

## D17: Shell semantics fixed to bash 4.0+

**Decision (added v0.1.1)**: Skills target POSIX shell, specifically bash 4.0+. Authors MAY rely on common bashisms (`[[ ]]`, arrays, brace expansion). Authors MUST NOT rely on `cmd.exe`, PowerShell, fish, or zsh-specific syntax.

**Alternatives considered**:
- **Pure POSIX `/bin/sh`**: too restrictive; loses arrays, `[[ ]]`, brace expansion, etc., which most real-world tools use.
- **No fixed shell**: every skill declares its own. Operationally a nightmare.
- **Container-based execution**: ship skills with a `Dockerfile` declaring the runtime. Heavy; out of scope for v1.
- **Shell-agnostic intermediate language**: invent a constrained DSL. Too much complexity for the niche.

**Rationale**:
- bash 4.0+ is the de-facto baseline shell on Linux and macOS (with the well-known macOS-ships-3.x exception, easily remedied via Homebrew).
- Most CLI tools assume bash semantics for their examples.
- Authors and banks share one mental model.

**Cost paid**: skills are not portable to Windows-native shells. `applicable_when.os` filtering excludes Windows skills from non-Windows banks (and vice versa); `shell:` field allows authors to declare alternative shells if desired.

**Practical implication**: a Windows user wanting agent-skills runs the bank inside WSL or Git Bash. The reference implementation explicitly does not target native Windows shells.

---

## What we deliberately do NOT specify

- **The embedding model**: each consumer chooses. Spec only requires consistency between skill index and query embeddings.
- **The vector store backend**: anything that can do nearest-neighbor search qualifies. The reference uses `js-vector-store` via `just-bash-data`.
- **The agent runtime**: any shell-capable runtime. The reference is `just-bash`. Could be Python's subprocess, Bash itself, etc.
- **The hosting CDN**: jsDelivr is one option. GitHub Raw, GitLab raw, self-hosted nginx all work.
- **Discovery aggregators**: third-party services (or none). Spec doesn't mandate or specify any.
- **UI / UX**: how a user browses, installs, or audits skills is implementation-defined.

This is intentional. The spec defines a **format and protocol**, not a **product**.

## Decisions still in flux (see ROADMAP.md)

- Should `chains` support conditional execution (if/else based on prior step output)?
- Should there be a built-in `dry-run` mode for skills?
- How to handle internationalization (skills in non-English `use_when` fields)?
- Should `examples` support negative examples ("DO NOT pick this skill when ...")?
- Is binary distribution (e.g., a Python helper) ever in scope?

These are recorded in `ROADMAP.md` for community input.

---

## Architectural caveats (added v0.1.1)

These are real trade-offs the design accepts, not gaps. Documented here for honesty.

### A1: Retrieval-first imposes a cognitive cost on the agent

The spec assumes the LLM can formulate good queries to find the right skill. In practice, LLMs vary at this:
- An LLM that has never seen the skill bank may not know what categories of skill exist, and therefore may not know what to ask for.
- A vague query produces vague vector matches; the agent picks an irrelevant skill, or worse, hallucinates one.
- Without a pre-loaded catalog, the agent cannot plan multi-step tasks knowing what tools are available.

MCP avoids this by injecting the full catalog. Agent-skills accepts the cognitive cost in exchange for token economy + privacy + decentralization. **Mitigations** the spec leaves to implementations:
- Bank exposes a meta-skill `list-skill-categories` returning a coarse taxonomy.
- After a failed query, bank surfaces "did you mean..." suggestions.
- Bank periodically re-injects "top-N most-used skills in your past sessions" into the system prompt.
- Pre-bake a small "core" catalog (5-10 essentials) into the system prompt, keep the rest behind retrieval.

These are out of spec but documented in `IMPLEMENTATION.md` as recommended practices.

### A2: Chain retry semantics depend on `idempotent`

A chain executor faced with a mid-chain failure must decide: abort, retry, skip. The spec adds `idempotent: true|false` to skills (D2.4) so the executor can decide safely:
- **Idempotent**: retry up to bank-configured limit before aborting.
- **Non-idempotent**: abort immediately. NEVER auto-retry (could double-charge, double-deploy, etc.).

Skills authoring non-idempotent operations (charge a customer, deploy code) MUST declare `idempotent: false` or the spec's default kicks in (which is `false`). This is a defense against careless retries.

### A3: SHA pinning protects against silent updates, not against rugged providers

A provider can publish a great v1.0 today and a malicious v2.0 in six months. SHA pinning protects you from changes you didn't approve, but the moment you bless v2.0's diff and update your pin, you adopt whatever the provider shipped. The spec does NOT prevent rug-pulls; it forces them to be visible.

`SPEC.md` §7.4 introduces "compare against last-approved baseline" — diff is shown relative to the most recent operator-approved state, regardless of intermediate auto-syncs. This makes drift visible across multiple updates. Banks that only diff "previous → new" miss accumulated changes.

### A4: Discovery via GitHub topics has scale limits

GitHub topics are flat. Once `agent-skills` topic has thousands of repos, the topic page becomes a hopeless wall of cards. Search is keyword-based, not semantic.

Mitigations:
- Multiple narrower topics (`agent-skills-payments`, `agent-skills-data`, etc.) — but this requires community coordination.
- Aggregator services that index the topic and offer better UX (planned for `agent-skills` v0.3.0 in `ROADMAP.md`).
- Awesome lists for high-signal curation.

The spec does not commit to a discovery mechanism beyond "use the web." Aggregators are an implementation pattern, not a normative requirement.
