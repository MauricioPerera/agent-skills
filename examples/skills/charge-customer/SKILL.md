---
schema_version: "1.0"
id: "charge-customer"
version: "1.0.0"
title: "Charge a customer via Stripe"
description: "Creates a one-time charge against an existing Stripe customer using the Stripe Charges API. Reads the secret key from the shell environment so it never reaches the LLM context."
use_when: "the user wants to charge an existing customer a specific amount in a specific currency, and the Stripe API is the payment processor"

command_template: |
  curl -fsS -X POST "https://api.stripe.com/v1/charges" \
    -u "$STRIPE_SECRET_KEY:" \
    -d "amount={amount}" \
    -d "currency={currency}" \
    -d "customer={customer_id}" \
    -d "description={description}"

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

license: "MIT"
author:
  name: "Example Inc."
  url: "https://example.com"
homepage: "https://example.com/docs/agent-skills"

category: "payments"
tags: ["stripe", "billing", "charge", "payment"]

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
    command: |
      curl -fsS -X POST "https://api.stripe.com/v1/charges" \
        -u "$STRIPE_SECRET_KEY:" \
        -d "amount=1000" \
        -d "currency=usd" \
        -d "customer=cus_ABC123" \
        -d "description=Agent-initiated charge"
    expected_output: '{"id":"ch_xxx","amount":1000,"currency":"usd",...}'
  - intent: "Bill customer cus_XYZ789 fifty euros for monthly subscription"
    command: |
      curl -fsS -X POST "https://api.stripe.com/v1/charges" \
        -u "$STRIPE_SECRET_KEY:" \
        -d "amount=5000" \
        -d "currency=eur" \
        -d "customer=cus_XYZ789" \
        -d "description=Monthly subscription"
  - intent: "Process payment for customer cus_DEF456 in Japanese yen, 1500 yen"
    command: |
      curl -fsS -X POST "https://api.stripe.com/v1/charges" \
        -u "$STRIPE_SECRET_KEY:" \
        -d "amount=1500" \
        -d "currency=jpy" \
        -d "customer=cus_DEF456" \
        -d "description=Agent-initiated charge"

provenance:
  source: "git"
  repo: "github.com/example/agent-skills"
  commit: "0000000000000000000000000000000000000000"
  tag: "v1.0.0"
  signed_by: "example-inc.gpg"
  published_at: "2026-04-28T00:00:00Z"

related:
  - "github.com/example/agent-skills@v1.0.0/skills/refund-charge"
  - "github.com/example/agent-skills@v1.0.0/skills/list-customer-charges"
---

# Charge a Customer

Creates a one-time charge against an existing Stripe customer. The customer must already exist in your Stripe account (use `create-customer` or the Stripe dashboard to create one first).

## Privacy property

This skill demonstrates the **credential isolation invariant** of agent-skills:

- The `command_template` references `$STRIPE_SECRET_KEY` as a shell variable.
- When the LLM emits this command, it does so verbatim — without substituting the secret.
- The shell substitutes `$STRIPE_SECRET_KEY` immediately before exec.
- **The secret never enters the LLM's context, the conversation log, or any upstream API call.**

For this skill to work, the operator must have set `$STRIPE_SECRET_KEY` in the shell environment **before** the agent runs (e.g., via `source ~/.stripe-credentials` or `direnv` or `1Password CLI`).

## When to use this

- The user has explicitly named a customer (with `cus_*` ID) and an amount.
- The intent is a one-off charge, not a subscription or invoice.
- The Stripe account is configured to accept the currency requested.

## When NOT to use this

- The user wants a subscription (use `create-subscription` instead).
- The user wants to charge a card directly without an existing customer (use `create-charge-with-card`, although this is generally discouraged).
- The user wants to refund (use `refund-charge`).
- The amount or customer ID is uncertain — confirm with the user first.

## Input validation

The skill bank validates `args` before execution:

- `amount` must be a positive integer ≤ 99,999,999 (Stripe's per-charge limit varies by account).
- `currency` must be one of the listed ISO codes.
- `customer_id` must match `^cus_[a-zA-Z0-9]+$` to prevent injection of arbitrary URLs.

A failed validation results in **exit code 5** (validation error) with a clear stderr message.

## Output

Stdout: the JSON response from Stripe's `/v1/charges` endpoint. Notable fields:
- `id`: the charge ID (`ch_*`).
- `status`: `succeeded`, `pending`, or `failed`.
- `amount`, `currency`, `customer`: echoed back.
- `failure_code`, `failure_message`: present if `status == "failed"`.

## Errors

- **HTTP 401 Unauthorized**: `$STRIPE_SECRET_KEY` is invalid. Re-check the env.
- **HTTP 402 Payment Required**: card declined; `failure_code` explains why.
- **HTTP 404 Not Found**: `customer_id` does not exist in this Stripe account.
- **HTTP 429 Too Many Requests**: hit Stripe's rate limit; retry with backoff.

## Audit considerations

Every successful charge is recorded in Stripe's dashboard with:
- The `description` you pass.
- A `metadata.source = "agent-skills"` field if you set it (consider adding `-d 'metadata[source]=agent-skills'` to the template if you want unambiguous attribution).

Stripe dashboard activity is the canonical audit log; the agent-skills audit log (`db skill_audit`) is a complementary local record.
