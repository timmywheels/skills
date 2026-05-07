# Versioning Deep Dive

> "We try to get the design right the first time, but we know we won't."

The hardest constraint in API design is that **you can never break a customer**. Once a client is in production, their code is frozen until they choose to update it. Your API has to keep doing what it did the day they integrated, possibly forever.

Stripe's date-based versioning is the most successful answer to this constraint. This document explains how it works and how to implement it.

---

## Why Date-Based Versions?

Most APIs version with major numbers: `v1`, `v2`, `v3`. Each new major is a curated rewrite that callers have to migrate to in a "big bang." Stripe's observation is that this is bimodal in the worst way:

- **Within a major version**, you can't make any breaking changes — so you accumulate technical debt and bad design choices.
- **Between major versions**, you make *too many* changes at once — so callers face a giant migration that's hard to schedule.

Date-based versions invert this. Every release is small (often just one or two breaking changes). Callers upgrade incrementally, one release at a time, on their own schedule. There is no "v3 migration day."

```
2024-06-20    ← release: deprecated `customer.account_balance`, added `customer.balance`
2024-09-30    ← release: changed `subscription.status` enum to add "paused"
2024-10-28    ← release: removed `card.fingerprint` (replaced by payment_method.card.fingerprint)
2025-01-15    ← release: ...
```

Each release is described by its delta from the previous one. The current version is just "the latest date."

---

## Account Pinning

When a new account makes its first API request, the server pins them to the *current* version:

```
Account created on 2025-03-04 → pinned to whatever version was current that day
```

Every request from that account gets responses formatted in their pinned version, even years later. They can:

1. **Override per-request** with `Stripe-Version: 2024-10-28`. Useful for testing migrations.
2. **Upgrade their pin** via dashboard or admin API. Once they upgrade, they can't go back without contacting support.
3. **Pin their webhook endpoints separately.** Each webhook endpoint has its own version pin, so the API can be upgraded independently of webhook handlers.

The key property: **the account is the unit of versioning**, not the request. A request without `Stripe-Version` falls back to the account's pin.

---

## What Counts as Backward-Compatible

These changes can ship at any time, on any version, no upgrade required:

| Change | Why it's safe |
|---|---|
| Adding a new endpoint | Old clients don't call it |
| Adding a new optional request parameter | Old clients don't send it; servers default it |
| Adding a new response field | Old clients ignore unknown fields (this is part of the contract) |
| Adding a new event type | Old webhook handlers ignore unknown event types |
| Adding a new enum value | Old clients are required to tolerate unknown enum values |
| Reordering response fields | JSON is unordered |
| Lengthening opaque IDs | The format is opaque; the prefix is stable |
| Performance / latency changes | Not part of the wire contract |
| Adding new error codes within an existing error type | Clients should branch on `type` first, treat unknown `code` as the type |

The "old clients tolerate unknown X" assumption is a **contract you set with integrators**, not a coincidence. Your docs must say:

> Clients MUST ignore unknown response fields, unknown enum values, and unknown event types. Failure to do so will break under additive changes.

This is non-negotiable. If you don't tell integrators this, they'll write strict code that breaks the moment you add a field.

---

## What Counts as Breaking

These require a new dated version:

| Change | Why it breaks |
|---|---|
| Removing a field | Old clients depend on it |
| Renaming a field | Same as remove + add |
| Changing a field's type | `boolean` → `string` is a parsing failure |
| Removing or renaming an enum value | Old clients have switch statements with the old value |
| Tightening validation | Inputs that used to be accepted now error |
| Changing default values | Behavior changes silently for existing callers |
| Changing the meaning of a field | Even if the type is the same, semantic shifts break logic |
| Adding a required request parameter | Existing requests start erroring |
| Changing the structure of an existing object (nested vs flat) | Parsing breaks |

If you find yourself wanting to do any of these, the move is: **add the new shape alongside the old, and use a version transformer to maintain both views.**

---

## The Response Compatibility Layer

This is the architectural trick that makes the whole system tractable.

### Without it (the bad version)

```python
def get_customer(id, version):
    customer = db.fetch(id)
    if version < "2024-06-20":
        customer.account_balance = customer.balance  # legacy field
        del customer.balance
    if version < "2023-08-16":
        customer.subscription = customer.subscriptions[0]  # used to be singular
        del customer.subscriptions
    if version < "2022-11-15":
        # ...
    return customer
```

Every endpoint accumulates `if version <` branches. New engineers learning the codebase have to reason about every prior version. Eventually nobody understands any code path anymore.

### With it (the good version)

Core business logic only writes the **current** version:

```python
def get_customer(id):
    return db.fetch(id)   # returns the modern shape, no version awareness
```

A separate **transformer chain** post-processes the response, walking backward through time:

```python
def serve(request, response):
    target_version = resolve_version(request)
    transformers = TRANSFORMERS_BY_VERSION  # ordered list, newest first

    for transformer in transformers:
        if transformer.version > target_version:
            response = transformer.apply_backward(response)

    return response
```

Each transformer is a small, isolated module:

```python
# transformers/2024_06_20_balance_rename.py

VERSION = "2024-06-20"
APPLIES_TO = ["customer"]   # which object types this affects

def apply_backward(obj):
    """Transform from post-2024-06-20 shape to pre-2024-06-20 shape."""
    if obj["object"] != "customer":
        return obj
    obj["account_balance"] = obj.pop("balance")
    return obj
```

When this transformer ships, it goes into the chain. Six months later, when someone forgets the `balance` rename ever happened, the transformer is still there doing its job. The core code never had to change.

### Properties of good transformers

- **Pure functions.** No side effects, no DB access. Just shape manipulation.
- **One concern per transformer.** A single rename, a single enum addition, a single field type change. Don't bundle.
- **Backward only.** You only need to go from "modern → ancient." You never serve an "ancient" response from "ancient" code.
- **Composable.** Applying transformers in order from newest version to target version produces the correct final shape.
- **Small.** Most transformers are <50 lines. If yours is bigger, you might be hiding a non-versionable change inside it.

### Request transformers

You also need *forward* transformers for incoming requests — translating "old request shape" to "modern request shape" before it hits core logic:

```python
# transformers/2024_06_20_balance_rename.py

def apply_forward(request_body):
    """Transform from pre-2024-06-20 input to current input."""
    if "account_balance" in request_body:
        request_body["balance"] = request_body.pop("account_balance")
    return request_body
```

Same chain, reversed direction.

---

## Version Resolution

Where the server gets the version for a given request, in order:

1. **`Stripe-Version` header** if present (per-request override)
2. **OAuth app's pinned version** if the request is on behalf of a connected app
3. **Account's pinned version** (the default)
4. **Latest version** for unauthenticated or new-account requests

Implementations should compute this once per request and pass it through the response pipeline.

---

## Webhook Versioning

Webhooks are *outbound* — the server constructs them, not the client. They need their own version system:

```
Webhook endpoint:    https://example.com/stripe-webhook
Pinned version:      2024-09-30
```

Each webhook endpoint has its own version pin, separate from the account's API version. This lets a customer:

- Upgrade their API integration without re-testing webhook handlers
- Test new webhook formats on a staging endpoint before flipping production
- Run multiple endpoints at different versions for migration

Server logic:

```python
def deliver_webhook(event, endpoint):
    # event is in current version
    payload = transform_to_version(event, endpoint.api_version)
    payload["api_version"] = endpoint.api_version
    sign_and_post(payload, endpoint.url, endpoint.signing_secret)
```

The `api_version` field on every event tells the receiver which schema they're getting. Useful for receivers that handle multiple versions during migration.

---

## When You Can't Version

Some changes can't be expressed as a forward/backward transformation:

- **Behavioral changes.** "Charges now retry on insufficient_funds." There's no transformer that makes the old behavior reappear.
- **Underlying calculation changes.** "We now compute tax differently."
- **Security fixes.** "We now validate the signature differently." Versioning a security fix would defeat the fix.

For these, the answer is one of:

1. **Make it opt-in via a request parameter.** New behavior only fires when caller explicitly enables it.
2. **Ship as a new endpoint.** `/v1/charges/v2/create` (yes, ugly) or `/v1/payment_intents` (cleaner — replace charges with a new resource).
3. **Announce, deprecate, and migrate with explicit communication.** Sometimes there's no clean way; you have to email customers and ship a migration deadline. Use this sparingly — every time you do, you erode trust.

---

## Documentation Strategy

Each version has its own changelog entry. Each entry includes:

- **Date** (the version identifier)
- **What changed** (concrete: "field renamed", "enum value added")
- **Migration guide** (how to update code from the previous version)
- **Examples** (before/after JSON for affected objects)

Stripe's docs render the entire reference at the user's pinned version. Field names, examples, even the listed enum values change based on which version you're viewing. This is achievable because the version system is a first-class part of the codebase, not an afterthought.

---

## Deprecation

Even within a versioning system, you sometimes want to say "please stop using this." Two mechanisms:

### Soft deprecation (no version bump)

Add a `Deprecation` header to responses that use a deprecated endpoint or field:

```
Deprecation: true
Link: <https://docs.example.com/migrating-from-account-balance>; rel="deprecation"
Sunset: Sat, 31 Dec 2026 23:59:59 GMT
```

Don't break anything. Just tell integrators in a place their tooling can pick up automatically (the `Sunset` header is consumed by some HTTP clients).

### Hard deprecation (version bump)

Ship a new version that removes the deprecated thing. Old accounts keep the old behavior. New accounts (and upgraders) don't see the deprecated form at all.

---

## When to Make a New Major

You will, eventually, want a `v2`. Stripe did. Reasons it's sometimes worth it:

- **Foundational shift in request/response format** (form-encoded → JSON, REST → GraphQL).
- **Coordinated rename of many resources** that aren't reasonable as transformers.
- **New auth model** that can't be transparently bridged.

Even then, **don't sunset v1**. Run both indefinitely. The point of date-based versioning is that you never force a customer to migrate. A v2 is just a parallel API; v1 keeps working forever.

---

## Summary

1. **Date-based rolling versions** — small deltas, callers upgrade incrementally.
2. **Pin the account on first request** — the account is the unit of versioning.
3. **Backward-compatible changes ship anytime** — additive only, with a documented contract that clients tolerate unknowns.
4. **Breaking changes require a new dated version** — and they should be small.
5. **Response compatibility layer** — internal code only writes current; transformer chain produces older shapes on the way out.
6. **Webhook endpoints version independently** of account API version.
7. **Old versions live forever** — never sunset, never force migration.
