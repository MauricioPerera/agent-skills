# COMPARISON — agent-skills vs alternatives

This document is honest about where `agent-skills` wins, where it doesn't, and where the trade-offs land. The goal is to help operators choose deliberately.

## TL;DR matrix

| Property | MCP | npm skill packs | Centralized registry | **agent-skills** |
|---|---|---|---|---|
| Tool definitions in LLM context | ✗ all loaded upfront | ✗ all loaded upfront | ✗ all loaded upfront | ✓ retrieved on demand |
| Token cost in system prompt | O(N tools) | O(N tools) | O(N tools) | O(1) |
| Per-task token cost | ~100 tokens | ~100 tokens | ~100 tokens | ~150–300 tokens |
| Credentials visible to LLM | ✗ in args | ✗ in args | ✗ in args | ✓ env-substituted |
| Provider hosts a server | ✗ required | — | ✗ registry hosts | ✓ static files only |
| Content immutability | ✗ | ✓ npm tarballs | ✓ registry-pinned | ✓ SHA-pinned |
| Audit trail | partial (server logs) | partial (npm metadata) | partial (registry logs) | ✓ git log |
| User can fork/modify | ✗ hard | ✓ npm fork | ✗ governance | ✓ git fork |
| Decentralized | ✗ | partial (npm is central) | ✗ | ✓ |
| Streaming responses | ✓ native | ✗ | varies | ✗ |
| Stateful sessions | ✓ | ✗ | varies | ✗ |
| Multi-language clients | ✓ | language-specific | varies | ✓ (any HTTP+shell) |
| Tool composition by reference | partial | partial | partial | ✓ chains by SHA |
| Discoverability | ✗ ad-hoc | ✓ npm search | ✓ registry search | ✓ GitHub topic + awesome lists |

The headline: **agent-skills wins on token economy, privacy, transparency, and decentralization. MCP wins on streaming and statefulness.** They occupy different niches.

## vs MCP (Model Context Protocol)

### Where MCP wins

1. **Streaming responses**. MCP defines a streaming protocol; tools can emit partial results progressively. agent-skills returns buffered output (the underlying shell is buffer-oriented). For tools that produce tens of MB of output progressively (e.g., a long-running SQL query, a build log), MCP gives better UX.

2. **Stateful sessions**. MCP servers can hold connections, sessions, transactional state across multiple tool calls. agent-skills is stateless per execution; long-lived state lives in the consumer's `db` collections. For workflows like "open a database transaction, do N writes, commit", MCP's connection model is more natural.

3. **Multi-language client maturity**. MCP has SDKs in Python, TypeScript, Go, Rust as of late 2025. agent-skills's reference runtime (`just-bash-data`) is Node-only today. Consumers in other ecosystems would need to implement the spec from scratch.

4. **Anthropic's adoption**. MCP has Anthropic backing, Claude Desktop integration, established server registry. agent-skills is a new pattern with zero brand recognition. Network effects favor MCP for now.

5. **Tool definitions are visible to the LLM at decision time**. The LLM sees the full schema for every tool when deciding what to call. agent-skills requires a retrieval step first; the LLM sees only the top-K results, which may exclude a relevant tool that ranked too low.

### Where agent-skills wins

1. **Token economics scale**. With 100+ tools, MCP's tool definitions consume thousands of tokens **before the user's question**. agent-skills uses a fixed ~200 tokens regardless. Crossover point: ~10–15 tools. Above that, agent-skills is significantly cheaper.

   | Catalog size | MCP tokens | agent-skills tokens | Skills win |
   |---:|---:|---:|---:|
   | 5 | 400 | 200 + 5×200 | 1.2× MCP |
   | 20 | 1,600 | 200 + 5×200 | 1.3× cheaper |
   | 100 | 8,000 | 200 + 5×200 | 6.6× cheaper |
   | 500 | 40,000 | 200 + 5×200 | 33× cheaper |

   (Numbers are rough; per-task agent-skills cost is for 5 retrievals @ 200 tokens. Real values depend on top-K and embedding API call.)

2. **Credential isolation is structural**. Per §P1 of `SECURITY.md`: secrets never reach the LLM context. With MCP, secrets either flow through args (LLM sees them) or are held by the server (no per-user isolation). agent-skills sidesteps the dilemma by letting the **shell** handle substitution.

3. **No vendor backend required**. An MCP tool needs a server (`@modelcontextprotocol/server-filesystem`, `@modelcontextprotocol/server-postgres`, etc.). The provider must run, monitor, and update that server. agent-skills providers commit a markdown file to a git repo — that's it. The deployment overhead asymptotes to zero.

4. **Transparency by default**. Every agent-skills skill is plain markdown that humans read. MCP tools are JSON schemas + opaque server logic. Auditing what a tool *actually does* requires reading source code; auditing an agent-skills skill requires reading one paragraph of YAML.

5. **Update workflow is git-native**. New version of a skill = git commit + tag + push. Consumers see the diff via `git diff`, choose to ingest or hold. MCP server upgrades require coordination between server operator and clients (re-fetch tool definitions, restart sessions).

6. **Composability**. agent-skills supports `chains` — a skill can reference other skills by SHA. The skill bank executes the chain transparently to the LLM. MCP composability requires the LLM to orchestrate multiple tool calls, paying tokens at each hop.

7. **Decentralization**. MCP servers are addressable by URL but "the MCP ecosystem" is largely an Anthropic-curated set. agent-skills inherits the web's decentralization: any git repo + CDN works.

8. **Auth without leakage**. The most striking property: an agent can use a skill that requires `$STRIPE_SECRET_KEY` without that key ever being mentioned to the LLM. The LLM emits `stripe charges create --customer cus_X`, the shell does the substitution before exec. MCP cannot replicate this without putting the credential in either the tool args or the server config (both have downsides).

### When to pick which

**Pick MCP when**:
- You have ≤20 tools and they're stable.
- You need streaming output.
- You need stateful sessions (DB connections, etc.).
- You're building inside Claude Desktop or another MCP-first client.
- Your tools genuinely need a server (not just a shell command).

**Pick agent-skills when**:
- You have many tools or want unbounded growth.
- Token cost matters.
- Credentials must not reach the LLM.
- You need user-modifiable, auditable tools.
- You want zero vendor backend dependency.
- Your tools are essentially shell commands (REST APIs via curl, CLIs, file operations).
- You need cryptographic provenance / audit trails.

**Use both**: they aren't mutually exclusive. An agent can have an MCP client for the 5 tools that need streaming + a skill bank for the 200 tools that don't. The patterns compose.

## vs npm skill packs

A naive alternative I considered before settling on the git+CDN model was: distribute skills as npm packages. Each skill pack is an npm dependency that exports a `Skill[]` array.

### Why it lost

1. **Forces consumer to use Node**. agent-skills works for any consumer that can do HTTPS + run a shell command. A Python agent can use it. A Go agent can use it. npm-pack consumers must include `node_modules` or fork the publishing model per language.

2. **Versioning weaker than git**. npm gives semver ranges (`^1.2.0`) but no SHA-level pinning unless you use `package-lock.json` carefully. git gives bit-immutable SHAs.

3. **No native diff workflow**. Comparing skill versions across npm releases requires `npm pack` + `tar` + `diff`. With git, it's `git diff v1.2.0 v1.3.0`.

4. **npm registry is centralized**. agent-skills inherits the web's decentralization; npm has a single canonical registry that could refuse to host or be coerced.

5. **Cost of publishing**. Authors must `npm publish`. agent-skills authors `git push` (already part of their workflow if they use git for source).

### Where npm packs would win

- If a skill includes binary helpers (a Python script, a compiled binary), npm's `files` field handles it cleanly. agent-skills explicitly excludes binary distribution; skills depend on commands already in the user's `PATH`.
- npm's installation flow is familiar and tooling-rich.

For the use cases we care about (skill = shell command + metadata), the overhead of npm doesn't pay off.

## vs centralized registry (e.g., a hypothetical `skills.io`)

Some projects propose a central registry where all skills are published, indexed, rated, and downloaded.

### Why it lost

1. **Single point of failure**. Registry down → ecosystem down. agent-skills has no central dependency.
2. **Gatekeeper risk**. Whoever runs the registry can refuse to host, deplatform users, or be compelled by regulation. Decentralized hosting routes around this.
3. **Trust concentration**. A central registry's compromise compromises everyone. Distributed git repos compromise individually.
4. **Operational burden**. Registry needs uptime, search, moderation, abuse handling. None of this needed when the web does the work.
5. **Discovery doesn't need a registry**. GitHub topics + search engines index the open web. There's no specific reason to build a parallel infrastructure.

### Where a central registry would win

- Faster, more consistent search (indexed, real-time updates).
- Quality curation (gatekeeper rejects low-quality or malicious skills).
- Standardized rating / popularity signals.

agent-skills can replicate these via aggregator services (third-party, optional, multiple competing) without baking centralization into the spec.

## vs no-spec / ad-hoc

Some teams will look at this and think "I'll just have my agent shell out to whatever, no formal spec needed".

### What you lose

- **Discoverability**: ad-hoc setups don't share a vocabulary; one team's "deploy script" is another's "release function".
- **Composability**: without `chains` or schema, every workflow is bespoke.
- **Audit**: without `provenance`, you can't tell what a command was, who shipped it, when it changed.
- **Privacy**: without `command_template` + env substitution, secrets sneak into prompts.
- **Portability**: skills written for one team are useless to another.

### What you gain

- Zero ceremony. Just write bash and call it.

For internal-only, single-team, throwaway use cases, the spec is overkill. For anything intended to outlast a single sprint or to be reused across teams, the spec earns its weight.

## The honest verdict

agent-skills is **not a universal replacement for MCP**. It's a pattern optimized for:

- Large, growing tool catalogs.
- Privacy-sensitive deployments.
- Audit-heavy compliance environments.
- Distributed teams without central tooling.
- Open-source toolkits where transparency matters.

It is **not a fit for**:

- Tools requiring streaming output or persistent sessions.
- Single-team setups with <10 stable tools (MCP or ad-hoc is simpler).
- Workflows that depend on Anthropic-specific MCP features.

For everything in between, agent-skills offers a path that scales further than MCP can while preserving properties (decentralization, provenance, privacy) that MCP gives up by design.
