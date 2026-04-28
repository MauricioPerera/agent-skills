---
schema_version: "0.1"
id: "charge-customer"
version: "1.0.0"
title: "Charge a customer via Stripe"
description: "Creates a one-time charge against an existing Stripe customer using the Stripe Charges API. Reads the secret key from the shell environment so it never reaches the LLM context."
use_when: "the user wants to charge an existing customer a specific amount in a specific currency, and the Stripe API is the payment processor"

# Note the placeholder positioning: each {arg} is a separate value passed to
# curl's -d option. The bank substitutes each as a single shell argument;
# the template does NOT pre-quote them. See SPEC.md §2.6 + DESIGN.md D16.
#
# $STRIPE_SECRET_KEY is a shell variable expanded at exec time by the SHELL,
# not a placeholder substituted by the bank. The LLM never sees its value.
command_template: "curl -fsSL --request POST https://api.stripe.com/v1/charges --user $STRIPE_SECRET_KEY: --data amount={amount} --data currency={currency} --data customer={customer_id} --data description={description}"

args:
  amount:
    type: integer
    description: "amount in the smallest currency unit (e.g., cents for USD, yen for JPY)"
    range: [1, 99999999]
  currency:
    type: string
    description: "ISO 4217 lowercase currency code"
    enum: ["usd", "eur", "gbp", "jpy", "cad", "aud"]
    default: "usd"
  customer_id:
    type: string
    description: "existing Stripe customer ID (format: cus_*)"
    pattern: "^cus_[a-zA-Z0-9]+$"
  description:
    type: string
    description: "human-readable note shown in the Stripe dashboard"
    default: "Agent-initiated charge"
    pattern: "^[a-zA-Z0-9 .,_:-]{1,256}$"

license: "MIT"
author:
  name: "Example Inc."
  url: "https://example.com"
homepage: "https://example.com/docs/agent-skills"

category: "payments"
tags: ["stripe", "billing", "charge", "payment"]

shell: "bash"
idempotent: false   # creating a charge is NOT idempotent without an idempotency_key

required_commands: ["curl"]
required_env:
  - "STRIPE_SECRET_KEY"
network:
  - "https://api.stripe.com/v1/charges"

applicable_when:
  shell_commands_present: ["curl"]
  env_present: ["STRIPE_SECRET_KEY"]

examples:
  - intent: "Charge customer cus_ABC123 ten dollars"
    command: "curl -fsSL --request POST https://api.stripe.com/v1/charges --user $STRIPE_SECRET_KEY: --data amount=1000 --data currency=usd --data customer=cus_ABC123 --data description='Agent-initiated charge'"
    expected_output: '{"id":"ch_xxx","amount":1000,"currency":"usd",...}'
  - intent: "Bill customer cus_XYZ789 fifty euros for monthly subscription"
    command: "curl -fsSL --request POST https://api.stripe.com/v1/charges --user $STRIPE_SECRET_KEY: --data amount=5000 --data currency=eur --data customer=cus_XYZ789 --data description='Monthly subscription'"
  - intent: "Process payment for customer cus_DEF456 in Japanese yen, 1500 yen"
    command: "curl -fsSL --request POST https://api.stripe.com/v1/charges --user $STRIPE_SECRET_KEY: --data amount=1500 --data currency=jpy --data customer=cus_DEF456 --data description='Agent-initiated charge'"

related:
  - "github.com/example/agent-skills@v1.0.0/skills/refund-charge"
  - "github.com/example/agent-skills@v1.0.0/skills/list-customer-charges"
---

# Charge a Customer

Creates a one-time charge against an existing Stripe customer. The customer must already exist in your Stripe account.

## Privacy property — the credential isolation invariant

This skill demonstrates [SPEC.md §8 P1](../../../SPEC.md#8-privacy-invariants):

- The `command_template` references `$STRIPE_SECRET_KEY` as a **shell variable**, not as a `{placeholder}`.
- The bank does NOT substitute `$STRIPE_SECRET_KEY`. The bank's substitution only handles `{placeholder}` patterns.
- When the bank emits the (substituted) command to the shell for execution, the shell sees `$STRIPE_SECRET_KEY` as a normal variable reference and expands it from the environment **at exec time**, after the LLM is no longer involved.
- The LLM receives only `command_template` (with `$STRIPE_SECRET_KEY` literal) — it never sees the key's value.

For this skill to work, the operator must have set `$STRIPE_SECRET_KEY` in the shell environment **before** the agent runs. Common mechanisms:

```bash
# Option A: dotenv-style file sourced at agent start
source ~/.stripe-credentials

# Option B: direnv per-project
echo 'export STRIPE_SECRET_KEY=sk_test_xxx' > .envrc
direnv allow

# Option C: 1Password CLI on demand
export STRIPE_SECRET_KEY=$(op read 'op://vault/Stripe/api-key')
```

The skill does NOT include a "set the key" step intentionally — credential bootstrap is the operator's responsibility, not the agent's.

## When to use this

- The user has explicitly named a customer (with `cus_*` ID) and an amount.
- The intent is a one-off charge, not a subscription or invoice.
- The Stripe account is configured to accept the currency requested.

## When NOT to use this

- The user wants a subscription (use `create-subscription` instead).
- The user wants to charge a card directly without an existing customer (use `create-charge-with-card`, although this is generally discouraged).
- The user wants to refund (use `refund-charge`).
- The amount or customer ID is uncertain — the agent should confirm with the user first.

## `idempotent: false`

Creating a charge is intrinsically non-idempotent: re-running this skill with the same args creates a **second** charge. Operators using chains MUST NOT auto-retry this skill (the chain executor checks `idempotent` before deciding). To make a charge retry-safe, add a `--data idempotency_key=<UUID>` to the template (see Stripe's docs on idempotency keys).

A future version of this skill may add an `idempotency_key` arg and flip to `idempotent: true`. That would be a MINOR version bump (it relaxes a constraint without breaking existing usage).

## Input validation

Before substitution, the bank validates:
- `amount`: positive integer ≤ 99,999,999.
- `currency`: one of the listed ISO codes.
- `customer_id`: matches `^cus_[a-zA-Z0-9]+$` — prevents injection of arbitrary URLs.
- `description`: matches the safe-character pattern, length ≤ 256.

Validation failure produces **exit code 5** (validation error) with a clear stderr message. The skill is never invoked with bad args.

## Output

Stdout: the JSON response from Stripe's `/v1/charges` endpoint. Notable fields:
- `id`: the charge ID (`ch_*`).
- `status`: `succeeded`, `pending`, or `failed`.
- `amount`, `currency`, `customer`: echoed back.
- `failure_code`, `failure_message`: present if `status == "failed"`.

## Errors

- **HTTP 401 Unauthorized**: `$STRIPE_SECRET_KEY` is invalid or empty. Re-check the env.
- **HTTP 402 Payment Required**: card declined; `failure_code` explains why.
- **HTTP 404 Not Found**: `customer_id` does not exist in this Stripe account.
- **HTTP 429 Too Many Requests**: hit Stripe's rate limit; retry with backoff.

## Audit considerations

Every successful charge is recorded in Stripe's dashboard. To make agent-emitted charges identifiable in the dashboard, consider extending the template with `--data 'metadata[source]=agent-skills'` — Stripe metadata is searchable.

The skill bank's local audit log (per [SPEC.md §4.5](../../../SPEC.md#45-audit-contract)) is a complementary record but not a substitute for Stripe's canonical activity log.
