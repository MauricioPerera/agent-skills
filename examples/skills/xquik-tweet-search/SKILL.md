---
schema_version: "0.1"
id: "xquik-tweet-search"
version: "1.0.0"
title: "Search X posts via Xquik"
description: "Searches public X posts through the Xquik REST API using a query string and result limit. Reads the API key from the shell environment so it never reaches the LLM context."
use_when: "the user wants to search public X posts by keyword, hashtag, username, or advanced X search operator through Xquik"
command_template: "python3 -c 'import os, sys, urllib.parse, urllib.request; query=sys.argv[1]; limit=sys.argv[2]; url=\"https://xquik.com/api/v1/x/tweets/search?\" + urllib.parse.urlencode({\"q\": query, \"limit\": limit}); req=urllib.request.Request(url, headers={\"X-API-Key\": os.environ[\"XQUIK_API_KEY\"]}); print(urllib.request.urlopen(req, timeout=30).read().decode())' {query} {limit}"

args:
  query:
    type: string
    description: "X search query, such as a keyword, hashtag, username, or supported X search operator"
    pattern: "^.{1,240}$"
  limit:
    type: integer
    description: "maximum number of posts to return"
    range: [1, 100]
    default: 10

license: "MIT"
author:
  name: "Xquik"
  url: "https://docs.xquik.com"
homepage: "https://docs.xquik.com/api-reference/overview"

category: "social-data"
tags: ["x", "twitter", "search", "social-data", "api"]

shell: "bash"
idempotent: true
required_commands: ["python3"]
required_env:
  - "XQUIK_API_KEY"
network:
  - "https://xquik.com/api/v1/x/tweets/search"

applicable_when:
  shell_commands_present: ["python3"]
  env_present: ["XQUIK_API_KEY"]

examples:
  - intent: "Find recent public posts about AI agents"
    command: "python3 -c 'import os, sys, urllib.parse, urllib.request; query=sys.argv[1]; limit=sys.argv[2]; url=\"https://xquik.com/api/v1/x/tweets/search?\" + urllib.parse.urlencode({\"q\": query, \"limit\": limit}); req=urllib.request.Request(url, headers={\"X-API-Key\": os.environ[\"XQUIK_API_KEY\"]}); print(urllib.request.urlopen(req, timeout=30).read().decode())' 'AI agents' 10"
  - intent: "Search hashtag posts for a launch"
    command: "python3 -c 'import os, sys, urllib.parse, urllib.request; query=sys.argv[1]; limit=sys.argv[2]; url=\"https://xquik.com/api/v1/x/tweets/search?\" + urllib.parse.urlencode({\"q\": query, \"limit\": limit}); req=urllib.request.Request(url, headers={\"X-API-Key\": os.environ[\"XQUIK_API_KEY\"]}); print(urllib.request.urlopen(req, timeout=30).read().decode())' '#launch' 25"
---

# Xquik Tweet Search

Search public X posts through the Xquik REST API. This example demonstrates
credential isolation for an API-key service while keeping query text and result
limits validated before execution.

## Credential Boundary

- `$XQUIK_API_KEY` is read by the shell process at execution time.
- The skill bank substitutes only `{query}` and `{limit}`.
- The LLM receives the command template and argument schema, not the key value.

## When To Use This

- The user asks to search public X posts by keyword, hashtag, username, or
  supported X search operator.
- The user has approved using their Xquik API key from the shell environment.
- The task only needs read-only public post search.

## When Not To Use This

- The user wants private reads, monitors, webhooks, exports, or write actions.
- The user has not configured `$XQUIK_API_KEY`.
- The task needs a full workflow guide instead of a single read-only command.

## Output

Stdout: the JSON response from `GET /api/v1/x/tweets/search`.

See the full Xquik skill and API docs for broader workflows:

- https://github.com/Xquik-dev/x-twitter-scraper/tree/master/skills/x-twitter-scraper
- https://docs.xquik.com/api-reference/overview
