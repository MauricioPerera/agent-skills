# IMPLEMENTATION — reference skill bank using just-bash-data

This document describes **one** way to build a conformant agent-skills skill bank. It is a **reference implementation**, not part of the canonical spec. The spec ([`SPEC.md`](./SPEC.md)) is implementation-agnostic; this guide shows how to satisfy it using a specific stack.

The reference stack is:

- **Storage + retrieval primitives**: [`just-bash-data`](https://www.npmjs.com/package/just-bash-data) — a [`just-bash`](https://github.com/vercel-labs/just-bash) plugin that provides `db` (MongoDB-style document store) and `vec` (vector similarity search) commands.
- **Shell**: bash 4.x+ (macOS/Linux native; Windows users can use WSL or Git Bash).
- **Embedding model**: any HTTPS-accessible API (OpenAI, Cohere, local Ollama). Spec requires consistency between indexing and querying; nothing else is dictated.

This is one viable choice. Other valid stacks include:

- SQLite + sqlite-vss + bash
- Postgres + pgvector + Python
- Redis + Redis Stack + Go
- A pure in-memory skill bank in any language

The stack does not change skill behavior; it changes operator ergonomics.

## Why just-bash-data

- The runtime that birthed the agent-skills design.
- Already implements every primitive the spec needs: structured docs (`db`), vector search (`vec`), authentication (JWT/RBAC), encryption-at-rest (AES-256-GCM), atomic persistence, IVF for scaling.
- Stable on npm at v1.1.0; CI green; 264 tests + 243 E2E assertions.
- Single-file install, zero infrastructure.

If your deployment context already has Postgres or Redis, those are equally valid. This guide doesn't argue you should adopt just-bash-data; it shows how to build a reference bank atop it for those who do.

## Schema mapping

The spec describes operations abstractly ("the bank persists subscriptions", "the bank embeds and indexes"). This section maps those operations to concrete just-bash-data collections.

### Subscriptions

```bash
# One-time bootstrap
db skill_subscriptions index create id --unique
db skill_subscriptions index create source_type --sorted

# Subscribe (full record matches SPEC §4.1)
db skill_subscriptions insert '{
  "_id": "stripe-skills",
  "source_type": "git",
  "repo": "github.com/stripe/agent-skills",
  "ref_requested": "v1.2.0",
  "ref_resolved": "a1b2c3d4e5f67890abcdef1234567890abcdef12",
  "url_template": null,
  "auto_update": false,
  "last_synced": "2026-04-28T12:00:00Z",
  "verify_signature": true,
  "trusted_keys": ["B5A4 9C28 D9F1 ..."]
}'

# List subscriptions (for sync daemon)
db skill_subscriptions find '{}' --sort last_synced:1

# Update last_synced after a sync
db skill_subscriptions update '{"_id": "stripe-skills"}' '{"$set": {"last_synced": "..."}}'
```

### Skill index

```bash
# One-time bootstrap
db skills index create category --sorted
db skills index create version --sorted
vec create skills --dim 1024 --quantize int8 --ivf-clusters 100

# Upsert a skill at ingest time (id = full identity per SPEC §1)
db skills insert '{
  "_id": "github.com/stripe/agent-skills@a1b2c3d4.../charge-customer",
  "schema_version": "0.1",
  "id": "charge-customer",
  "version": "1.2.0",
  "title": "Charge a customer via Stripe",
  "description": "...",
  "use_when": "...",
  "command_template": "...",
  "args": { ... },
  "examples": [ ... ],
  "category": "payments",
  "tags": ["stripe", "billing"],
  "license": "MIT",
  "required_commands": ["curl"],
  "required_env": ["STRIPE_SECRET_KEY"],
  "network": ["https://api.stripe.com/v1/charges"],
  "applicable_when": { ... },
  "provenance": {
    "source_type": "git",
    "source": "github.com/stripe/agent-skills",
    "ref_resolved_to": "a1b2c3d4...",
    "ref_requested": "v1.2.0",
    "fetched_at": "2026-04-28T12:00:00Z",
    "signature_status": "valid",
    "signed_by": "B5A4 9C28 D9F1 ..."
  },
  "usage_count": 0,
  "avg_rating": null
}'

# Store the embedding (computed locally from the spec §4.2 composition)
vec store skills "github.com/stripe/agent-skills@a1b2c3d4.../charge-customer" "$EMBEDDING_VECTOR"
```

### Audit log

```bash
db skill_audit insert '{
  "_id": "<auto>",
  "skill_id": "github.com/stripe/agent-skills@a1b2c3d4.../charge-customer",
  "intent": "charge cus_X 50 USD",
  "args": {
    "customer_id": "cus_X",
    "amount": 5000,
    "currency": "usd"
  },
  "exit_code": 0,
  "elapsed_ms": 234,
  "rating": 5,
  "timestamp": "2026-04-28T12:34:56Z"
}'

# Aggregate ratings to bias future retrieval
db skill_audit aggregate '[
  {"$group": {
    "_id": "$skill_id",
    "n_uses": {"$count": 1},
    "avg_rating": {"$avg": "$rating"}
  }},
  {"$sort": {"avg_rating": -1}}
]'

# Propagate aggregated stats back to the skill index
db skills update '{"_id": "..."}' '{"$set": {"avg_rating": 4.7, "usage_count": 142}}'
```

### Trusted keys (provenance Level 3+)

```bash
db trusted_keys index create fingerprint --unique

db trusted_keys insert '{
  "_id": "stripe-llm.gpg",
  "fingerprint": "B5A4 9C28 D9F1 ...",
  "public_key": "<armored>",
  "imported_at": "2026-04-28T12:00:00Z",
  "imported_from": "https://stripe.com/.well-known/agent-skills-key.asc"
}'
```

## Sync daemon (sketch)

The sync daemon is the operational glue that pulls skills from sources and updates the index. Below is a simplified bash implementation.

```bash
#!/usr/bin/env bash
# sync-skills.sh — fetch + ingest all subscribed skills
set -euo pipefail

EMBED_API="${EMBED_API:-http://localhost:11434/api/embeddings}"
EMBED_MODEL="${EMBED_MODEL:-bge-m3}"

embed() {
  curl -sS "$EMBED_API" \
    -d "{\"model\":\"$EMBED_MODEL\",\"prompt\":$(jq -Rs . <<<"$1")}" \
    | jq -c '.embedding'
}

resolve_ref() {
  local repo="$1" ref="$2"
  case "$repo" in
    github.com/*)
      gh api "repos/${repo#github.com/}/git/refs/tags/$ref" \
        --jq '.object.sha' 2>/dev/null \
      || gh api "repos/${repo#github.com/}/commits/$ref" --jq '.sha'
      ;;
    gitlab.com/*)
      curl -sS "https://gitlab.com/api/v4/projects/$(echo "${repo#gitlab.com/}" | jq -sRr @uri)/repository/commits/$ref" \
        | jq -r '.id'
      ;;
    *)
      echo "Unsupported host: $repo" >&2; return 1
      ;;
  esac
}

build_url() {
  local repo="$1" ref="$2" path="$3"
  case "$repo" in
    github.com/*)
      echo "https://cdn.jsdelivr.net/gh/${repo#github.com/}@${ref}/${path}/SKILL.md"
      ;;
    gitlab.com/*)
      echo "https://gitlab.com/${repo#gitlab.com/}/-/raw/${ref}/${path}/SKILL.md"
      ;;
  esac
}

ingest_skill() {
  local sub_id="$1" repo="$2" ref="$3" path="$4"
  local url id_full md frontmatter body title use_when description tags examples_intents embed_text emb

  url=$(build_url "$repo" "$ref" "$path")
  md=$(curl -fsSL "$url") || { echo "fetch failed: $url" >&2; return 1; }

  # Split frontmatter / body
  if ! [[ "$md" =~ ^---$ ]]; then
    echo "no frontmatter in $url" >&2; return 1
  fi
  frontmatter=$(awk '/^---$/{c++; next} c==1' <<<"$md")

  # Parse minimal fields with yq (recommended) or python.
  title=$(yq -r '.title' <<<"$frontmatter")
  use_when=$(yq -r '.use_when' <<<"$frontmatter")
  description=$(yq -r '.description' <<<"$frontmatter")
  tags=$(yq -r '.tags // [] | join(" ")' <<<"$frontmatter")
  examples_intents=$(yq -r '.examples // [] | map(.intent) | join("\n")' <<<"$frontmatter")

  # Compose embedding text (SPEC §4.2)
  embed_text="${title}. ${use_when}. ${description}. ${examples_intents}. ${tags}"
  emb=$(embed "$embed_text")

  # Compute identity
  id_full="${repo}@${ref}/${path}"

  # Upsert into index (here using just-bash-data's db + vec commands)
  db skills insert "$(jq -c \
    --arg id "$id_full" \
    --argjson md "$(yq -o=json '.' <<<"$frontmatter")" \
    '$md + {_id: $id}'
  )"
  vec store skills "$id_full" "$emb"
}

# Main loop
for sub in $(db skill_subscriptions find '{}' | jq -c '.[]'); do
  sub_id=$(jq -r '._id' <<<"$sub")
  repo=$(jq -r '.repo' <<<"$sub")
  ref_req=$(jq -r '.ref_requested' <<<"$sub")
  ref=$(resolve_ref "$repo" "$ref_req")

  echo "[$sub_id] resolved $ref_req → $ref"

  # Fetch skills-index.json
  index_url=$(build_url "$repo" "$ref" "" | sed 's/SKILL.md/skills-index.json/' | sed 's|/$||')
  index=$(curl -fsSL "$index_url")

  echo "$index" | jq -c '.skills[]' | while read -r skill; do
    path=$(jq -r '.id' <<<"$skill")
    ingest_skill "$sub_id" "$repo" "$ref" "$path" || true
  done

  db skill_subscriptions update "{\"_id\":\"$sub_id\"}" \
    "{\"\$set\":{\"ref_resolved\":\"$ref\",\"last_synced\":\"$(date -u +%FT%TZ)\"}}"
done
```

This is intentionally simple; production banks would add:

- Diff against previous SHA before applying changes.
- Signature verification (gpg) per SPEC §5.
- Concurrency / parallelization.
- Retry with backoff on network errors.
- Schema validation against [`schemas/skill.schema.json`](./schemas/skill.schema.json).
- Truncation handling for long embedding inputs (SPEC §4.2).

## Query daemon (sketch)

```bash
#!/usr/bin/env bash
# query-skills.sh — find skills matching a user intent
set -euo pipefail

INTENT="$1"
K="${K:-5}"

# Embed the query with the SAME model used at ingest
QEMB=$(curl -sS "$EMBED_API" -d "{\"model\":\"$EMBED_MODEL\",\"prompt\":$(jq -Rs . <<<"$INTENT")}" | jq -c '.embedding')

# Vector search
TOP_IDS=$(vec search skills "$QEMB" --k "$K" | jq -r '.[].id')

# Fetch metadata + filter by applicable_when (simple example: env_present check)
for id in $TOP_IDS; do
  skill=$(db skills find "{\"_id\":$(jq -Rs . <<<"$id")}" | jq '.[0]')
  required_env=$(jq -r '.required_env // [] | .[]' <<<"$skill")

  applicable=true
  for env in $required_env; do
    if [[ -z "${!env:-}" ]]; then applicable=false; break; fi
  done

  if "$applicable"; then
    jq -c '{id: ._id, title, command_template, use_when}' <<<"$skill"
  fi
done
```

## Execution daemon (sketch)

```bash
#!/usr/bin/env bash
# exec-skill.sh — run a skill with given args
set -euo pipefail

SKILL_ID="$1"
shift
ARGS_JSON="$1"  # e.g., '{"amount":1000,"currency":"usd","customer_id":"cus_X"}'

skill=$(db skills find "{\"_id\":$(jq -Rs . <<<"$SKILL_ID")}" | jq '.[0]')
if [[ "$skill" == "null" ]]; then
  echo "skill not found: $SKILL_ID" >&2
  exit 3
fi

# Validate args against args schema (omitted here; see SPEC §4.4 step 2)

# Substitute placeholders in command_template
template=$(jq -r '.command_template' <<<"$skill")
cmd="$template"
for key in $(jq -r '.args // {} | keys[]' <<<"$ARGS_JSON"); do
  value=$(jq -r ".$key" <<<"$ARGS_JSON")
  # Simple substitution; production uses single-quote escaping per SPEC §2.6
  escaped=$(printf "'%s'" "${value//\'/\'\\\'\'}")
  cmd="${cmd//\{$key\}/$escaped}"
done

# Execute and capture
start=$(date +%s%3N)
stdout=$(bash -c "$cmd" 2> /tmp/stderr.$$)
exit_code=$?
elapsed=$(( $(date +%s%3N) - start ))

# Audit
db skill_audit insert "$(jq -c -n \
  --arg sid "$SKILL_ID" \
  --argjson args "$ARGS_JSON" \
  --argjson code "$exit_code" \
  --argjson ms "$elapsed" \
  --arg ts "$(date -u +%FT%TZ)" \
  '{skill_id:$sid, args:$args, exit_code:$code, elapsed_ms:$ms, timestamp:$ts}'
)"

echo "$stdout"
exit "$exit_code"
```

## Encryption + salt

just-bash-data v1.1.0 supports configurable salts (`PluginOptions.salt`). For multi-tenant deployments where each tenant should have isolated keys derived from a shared password:

```typescript
import { Bash } from "just-bash";
import { createDataPlugin } from "just-bash-data";

const bash = new Bash({
  customCommands: createDataPlugin({
    encryptionKey: process.env.SHARED_PASSWORD,
    salt: process.env.TENANT_ID,    // per-tenant salt
  })
});
```

Skill banks deployed in a multi-tenant agent service can encrypt each tenant's skill_audit + skill_subscriptions independently while sharing the same password material.

## Sandboxing

just-bash inherits primitives for restricting commands' filesystem access and network calls:

- `ReadOnlyFs` wrapper restricts a sandboxed filesystem to specific paths.
- `AllowList` for network calls (matches against the skill's `network` field).
- Custom `CustomCommand[]` for the agent — the `bash.exec()` runtime only knows about explicitly-registered commands; arbitrary `/bin/sh` is not reachable from inside the sandbox.

For B2 (strict consumer) conformance, deploy with these primitives enabled. For B1 (consumer-conformant), they are optional but recommended.

## Performance notes

| Operation | Reference cost |
|---|---|
| Subscribe (one-time) | ~5s for a 50-skill index (network-dominated) |
| Sync (no changes) | ~100ms (HEAD requests + stale-check) |
| Sync (50 skills changed) | ~5s + N×embedding-API cost |
| Query (vec search 1000 skills) | ~5ms (without IVF) / ~1ms (with IVF) |
| Execute skill (no exec) | ~1ms (validation + substitution) |
| Execute skill (with exec) | dominated by the underlying command |

For banks growing beyond ~10K skills, enable IVF at vec collection creation time:

```bash
vec create skills --dim 1024 --quantize int8 --ivf-clusters 100 --ivf-probes 10
# After ingest:
vec ivf build skills
```

## Validation against the JSON schema

The spec ships [`schemas/skill.schema.json`](./schemas/skill.schema.json) — a JSON Schema describing valid frontmatter. Validate before ingest:

```bash
yq -o=json '.' SKILL.md \
  | npx ajv validate -s schemas/skill.schema.json --spec=draft2020 || {
    echo "invalid skill" >&2; exit 1;
  }
```

The reference sync daemon runs this check on every fetched `SKILL.md`. Skills failing validation are skipped, logged, and reported in the diff.

## Limitations of this reference

- **Single-process**: the reference doesn't address concurrent execution by multiple agents. just-bash-data isolates state per `IFileSystem` instance; cross-process coordination needs a shared FS or a separate coordination layer.
- **No streaming**: per [`COMPARISON.md`](./COMPARISON.md), agent-skills doesn't support streaming output. just-bash returns buffered stdout.
- **Bash-only**: just-bash interprets bash; running on Windows needs WSL or Git Bash.
- **No retry policy for non-idempotent skills**: implementing the chain executor correctly requires policy decisions (when to abort vs. when to skip). The reference daemon punts on this.

For other limitations, see [`COMPARISON.md`](./COMPARISON.md) and [`SECURITY.md`](./SECURITY.md).

## Where to go from here

- The reference sync + query + exec scripts above are sketches. A production-ready agent-skills CLI atop just-bash-data is planned for `agent-skills` v0.2.0 (see [`ROADMAP.md`](./ROADMAP.md)).
- For implementations atop other stacks (SQLite, Postgres, Redis), follow the same SPEC.md operational contracts. The mapping from spec operations to your DB primitives is mechanical.
- For testing your implementation against the spec, [`schemas/skill.schema.json`](./schemas/skill.schema.json) + the conformance test fixtures (planned for v0.2.0) provide the ground truth.
