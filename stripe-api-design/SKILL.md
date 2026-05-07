---
name: stripe-api-design
description: Design REST APIs the way Stripe does — resource-oriented, predictable, extensible, and built for indefinite backward compatibility. Use when designing new endpoints, modeling resources, choosing request/response shapes, defining error contracts, planning pagination or idempotency, evolving an existing API, or reviewing API designs. Triggers on tasks involving REST APIs, public APIs, SDK design, webhook delivery, API versioning, or "Stripe-like" API ergonomics.
metadata:
  author: timmywheels
  version: "1.0"
---

# Stripe-Style API Design

Stripe's API is the modern reference standard for developer-facing REST APIs. This skill captures the conventions, decision frameworks, and design tradeoffs that make their API feel inevitable — predictable resources, painless pagination, exactly-once writes, and indefinite backward compatibility.

Apply these patterns when you are **designing the public surface** of any system: a SaaS REST API, an internal platform API, a webhook system, or a typed SDK. The goal is an API that integrators can learn once and reason about everywhere.

---

## The Core Philosophy

> **APIs are products. Developers are the customers.**

Every other rule in this skill flows from that single sentence. It implies:

1. **Optimize for the integrator's first hour and tenth year.** A new developer should grasp the API by reading three endpoints. A ten-year customer should never have to rewrite working code.
2. **Predictability beats cleverness.** If `/customers` works one way, `/charges` must work the same way. Surprise is a defect.
3. **Each endpoint is a permanent contract.** Once it ships, you support it forever. Design accordingly — slowly, with review, and with a versioning escape hatch.
4. **Errors are part of the product.** A confusing error message is a UX bug, not a server log line.
5. **Make the right thing the easy thing.** Idempotency keys, expand, metadata — the API should make robust integrations trivial and fragile ones harder than necessary.

If you can't justify a design choice in those terms, it isn't ready.

---

## The Ten Patterns

These are the load-bearing patterns. The rest of this document is detail.

| # | Pattern | Why it matters |
|---|---|---|
| 1 | **Resource-oriented URLs** | Nouns, not verbs. `/customers/cus_123`, never `/getCustomer?id=123`. |
| 2 | **Prefixed string IDs** | `cus_`, `ch_`, `pi_`. Type-encoded, debuggable, opaque. |
| 3 | **Consistent object envelope** | Every object has `id`, `object`, `created`, `livemode`, `metadata`. |
| 4 | **List envelope** | `{ object: "list", data: [...], has_more: bool, url }`. |
| 5 | **Cursor pagination** | `starting_after` / `ending_before` over object IDs. Never offset. |
| 6 | **`expand` parameter** | Inline related objects on demand. Avoid N+1 round-trips. |
| 7 | **Idempotency keys** | `Idempotency-Key` header. Exactly-once semantics for every POST. |
| 8 | **Structured errors** | `type`, `code`, `message`, `param`, `doc_url`, `request_id`. |
| 9 | **Date-based versioning** | `Stripe-Version: 2024-10-28`. Pinned per account. Never breaks. |
| 10 | **Metadata everywhere** | Every object accepts arbitrary key-value `metadata` from the caller. |

If your API has all ten, it will feel like Stripe's. If it's missing any, you have homework.

---

## 1. Resource Modeling

The foundation. Get this wrong and no convention can save the API.

### Naming

- **Plural, lowercase, snake_case.** `/customers`, `/payment_intents`, `/subscription_items`.
- **Nouns, not verbs.** Actions are HTTP methods, not URL segments — *with one critical exception* (state transitions, below).
- **Stable names.** Once published, the resource name is permanent. `account` cannot become `organization` later, even if "organization" is more accurate. Pick the name you can live with for a decade.

### IDs

Every resource has a string ID with a stable, type-encoded prefix:

```
cus_NffrFeUfNV2Hib       # Customer
ch_3MtwBwLkdIwHu7ix28a3tqPa   # Charge
pi_3MtwBwLkdIwHu7ix2QFi9zPx   # PaymentIntent
sub_1MowQVLkdIwHu7ixeRlqHVzs  # Subscription
```

Properties of good IDs:

- **Prefix encodes type.** `cus_` always means Customer. Pasting an ID into a log gives instant context.
- **Opaque body.** Random, non-sequential, no info about scale or order. Numeric autoincrement IDs leak business data and create enumeration vulnerabilities.
- **Stringly typed.** Always strings, never numbers. Numeric IDs eventually overflow, get coerced to floats, or get stripped of leading zeros by a caller's JSON parser.
- **Reasonable length.** Long enough to be unguessable, short enough to fit in a tweet.
- **Distinct test/live IDs.** Test mode IDs should be visually distinguishable so a test ID never accidentally lands in production logs. Stripe uses prefix variations (e.g., `tok_test_…`).

### The object envelope

Every resource shares a common spine:

```json
{
  "id": "cus_NffrFeUfNV2Hib",
  "object": "customer",
  "created": 1680893993,
  "livemode": false,
  "metadata": { "internal_user_id": "9f4c..." },
  "...": "type-specific fields"
}
```

Required on every object:

- **`id`** — the prefixed string identifier
- **`object`** — discriminator string. Lets clients route polymorphically without inspecting URL or context.
- **`created`** — Unix epoch seconds (integer). When this resource was created.
- **`livemode`** — boolean. Was this object created with a live key or a test key? Critical for sanity checks.
- **`metadata`** — string→string map for caller-defined data. See §10.

The `object` field is non-negotiable. It enables polymorphic events (`event.data.object` can be any resource type), enables generic SDK handling, and makes any JSON blob self-describing.

### State transitions: the verb exception

Most mutations are `POST` to the resource for create, `POST` to the instance for update (Stripe uses POST for both — see §3), and `DELETE` for delete. But **state changes get their own sub-resource action endpoints**:

```
POST /v1/payment_intents/:id/capture
POST /v1/payment_intents/:id/cancel
POST /v1/payment_intents/:id/confirm
POST /v1/refunds/:id/cancel
POST /v1/subscriptions/:id/cancel
```

Why not just `PATCH /payment_intents/:id` with `{status: "captured"}`?

- **Atomic intent.** "Capture this payment" is a verb the business cares about. Hiding it inside a generic PATCH obscures audit logs and webhooks.
- **Distinct authorization.** Capturing may require different permissions than editing metadata.
- **Distinct side effects.** Capture moves money. Update metadata doesn't. They shouldn't share an endpoint.
- **Distinct idempotency.** A retried capture must be safe; a retried metadata edit has different semantics.

**Rule of thumb:** if a state field has a constrained lifecycle (`requires_action → processing → succeeded`), each transition gets its own action endpoint. Never let clients write the status field directly.

See [`references/resource-modeling.md`](references/resource-modeling.md) for deeper examples.

---

## 2. Request and Response Conventions

### Field naming

- **`snake_case`** for every field. Never camelCase, never PascalCase, never abbreviation. `payment_method`, not `paymentMethod` or `pmt_method`.
- **Spell out acronyms in field names.** `customer_id`, not `cust_id`. The two extra characters cost nothing; the lost guess costs an integration.
- **Suffix conventions:**
  - `_at` for absolute timestamps: `created_at`, `canceled_at` — though Stripe historically uses `created` (no suffix); pick one and stay consistent.
  - `_id` for foreign keys: `customer_id`, `latest_invoice_id`.
  - `_url` for URL strings.

### Primitive types

| Concept | Representation | Reason |
|---|---|---|
| Money | **Integer in smallest currency unit** (`amount: 1099` for $10.99) | Floats lose pennies. JSON parsers vary. Cents are exact. |
| Currency | **ISO 4217 lowercase** (`"usd"`, `"eur"`) | Lowercase keeps it noiseless and consistent. |
| Time | **Unix epoch seconds** (integer) | Universal, timezone-free, sortable, parses everywhere. |
| Booleans | **Real booleans**, never `"true"` strings | A boolean is a boolean. |
| Enums | **Lowercase snake_case strings** (`"requires_action"`) | Strings are extensible; numeric enums are not. |
| Optional → absent | **`null`**, not omitted | Present-with-null is unambiguous. |

### Request format

- **`POST` for create AND update.** Stripe uses POST for both create and update on v1, partly historical and partly because form-encoded bodies don't cleanly support PATCH semantics. v2 uses JSON and supports both POST and DELETE with idempotency. **For new APIs in 2026: use JSON request bodies, POST for create, PATCH for partial update, DELETE for delete.** That's the modern convention; Stripe's v1 form encoding is the artifact of when it shipped.
- **JSON request bodies** with `Content-Type: application/json` (v2 standard).
- **Query parameters for read-time concerns:** filtering, pagination, expansion. Never for state-changing input.

### Request/response symmetry

The shape you `POST` should be the shape you `GET` back, modulo server-generated fields like `id`, `created`, and computed status. Caller writes `metadata.internal_id`; caller reads `metadata.internal_id`. No mysterious renames, no `request_metadata` vs `response_metadata`.

### Sparse responses

Return *complete* objects by default. Don't make callers do a second fetch to get the fields they actually wanted. If a field is large or sensitive (raw card numbers, encrypted payloads), gate it behind `expand` (§5) instead of omitting it silently.

---

## 3. Listing, Filtering, and Pagination

Every list endpoint shares a single shape.

### The list envelope

```json
{
  "object": "list",
  "url": "/v1/customers",
  "has_more": true,
  "data": [
    { "id": "cus_NffrFeUfNV2Hib", "object": "customer", "...": "..." },
    { "id": "cus_Nf9...", "object": "customer", "...": "..." }
  ]
}
```

Fields:

- `object: "list"` — the polymorphic discriminator, same as on resources.
- `url` — the canonical path of this list. Lets generic SDKs auto-paginate without reverse-engineering URLs.
- `has_more` — boolean. The signal a paginating loop watches.
- `data` — the array. Always called `data`, never `items` / `results` / `customers`.

### Cursor pagination

Two parameters, mutually exclusive:

- `starting_after=cus_xxx` — return the page *after* this object.
- `ending_before=cus_xxx` — return the page *before* this object.

Plus `limit` (default 10, max 100).

```
GET /v1/customers?limit=25&starting_after=cus_NffrFeUfNV2Hib
```

**Never use `offset`/`page` pagination for high-volume lists.** Offsets are stable only on frozen data. Insert one row and every subsequent page either duplicates a record or skips one. Cursor-on-ID pagination is consistent under concurrent writes because the cursor is anchored to a real object.

### Filtering

Every list endpoint accepts filters as query parameters, named identically to the response field they filter on:

```
GET /v1/charges?customer=cus_xxx&status=succeeded&created[gte]=1680000000
```

For range filters on comparable fields, use bracket suffixes: `[gte]`, `[gt]`, `[lte]`, `[lt]`. This is the one place bracket syntax pays off — it composes cleanly without inventing new field names per range type.

**Don't nest list endpoints.** Prefer `GET /v1/charges?customer=cus_xxx` over `GET /v1/customers/cus_xxx/charges`. Reasons:

- One endpoint to test, document, and version.
- Filters compose: `?customer=cus_xxx&status=succeeded`.
- Pagination cursors stay valid across filter combinations.

### Auto-pagination in SDKs

The list envelope's `has_more` + `data[].id` is sufficient for SDKs to expose `for charge in stripe.Charge.list().auto_paging_iter():`. Design for the SDK ergonomics, not just the wire format.

---

## 4. Expansion: solving N+1 at the protocol layer

Most APIs return foreign keys and force callers to make follow-up requests:

```json
{ "id": "ch_xxx", "customer": "cus_xxx" }   // now fetch the customer separately
```

This is the N+1 problem in protocol form. Stripe's answer is the `expand` parameter:

```
GET /v1/charges/ch_xxx?expand[]=customer&expand[]=invoice.subscription
```

```json
{
  "id": "ch_xxx",
  "customer": { "id": "cus_xxx", "object": "customer", "...": "..." },
  "invoice": {
    "id": "in_xxx",
    "object": "invoice",
    "subscription": { "id": "sub_xxx", "object": "subscription", "...": "..." }
  }
}
```

### Rules for `expand`

- **Default is the ID string.** Without `expand`, every reference is a string ID. Predictable, cheap.
- **Caller opt-in only.** Never expand by default — it inflates payloads and creates implicit cost.
- **Dot notation for nested expansion.** `expand[]=invoice.subscription`.
- **Cap depth.** Stripe caps at 4 levels. Pick a number; document it.
- **Works on list endpoints too.** Use the `data.` prefix: `expand[]=data.customer`.
- **Same field name, different type.** This is the trickiest part. `customer` is sometimes `"cus_xxx"` (string) and sometimes `{...}` (object). Document this clearly. SDKs should expose both forms via union types.

### When NOT to use expand

- For one-to-many relationships ("a customer's charges"). Use a list endpoint with a filter.
- For collections that may be huge. Expansion of unbounded lists is a footgun.
- For fields you'd want every time. If 99% of callers expand `customer`, just include the customer object by default — the expand parameter exists for the 1% case where you don't.

---

## 5. Idempotency: exactly-once semantics for free

Network errors are inevitable. Without idempotency, retrying a failed `POST /charges` either double-charges the customer (if the original request succeeded) or never charges at all (if the caller gives up). Both are wrong.

### The mechanism

Caller generates a unique key per logical operation and sends it as a header:

```
POST /v1/charges
Idempotency-Key: a3b8f2c4-9d5e-4f7a-bc1d-0e2f6a8d9b3c
```

Server behavior:

1. **First time seeing this key:** execute normally, save the (status code + response body) tuple keyed by `(account_id, key)`.
2. **Same key, same parameters:** return the saved response. Set `Idempotent-Replayed: true`.
3. **Same key, different parameters:** return an error. The key is locked to its first body to prevent accidental misuse.
4. **Same key, original request still in flight:** return 409 Conflict. Caller should retry after a backoff.
5. **Key expires after 24 hours.** After that, the same key starts fresh.

### Design rules

- **Required for every state-changing operation.** POST, PATCH, DELETE that creates side effects.
- **Caller-generated, not server-generated.** The whole point is the *caller* recovers from a failed response. UUIDv4 is the conventional choice.
- **Scope: per (account, endpoint, key)** at minimum. Don't let one customer's keys collide with another's.
- **Cache failures, not just successes.** A 4xx response is also cached — otherwise retries on validation errors get inconsistent answers.
- **Never reuse keys across logically distinct operations.** "Charge $50" and "Charge $50 again" need different keys.
- **GET and DELETE are inherently idempotent** by HTTP spec. Don't add idempotency keys there; it adds confusion without benefit.

This is one of the highest-leverage patterns in the API. The full implementation discussion is in [`references/idempotency.md`](references/idempotency.md).

---

## 6. Errors: the most-read part of your docs

Integrators spend more time staring at errors than success responses. Treat error design with the same care.

### The envelope

```json
{
  "error": {
    "type": "card_error",
    "code": "card_declined",
    "decline_code": "insufficient_funds",
    "message": "Your card has insufficient funds.",
    "param": "source",
    "doc_url": "https://docs.stripe.com/error-codes#card-declined",
    "request_id": "req_NffrFeUfNV2Hib"
  }
}
```

Every field earns its place:

- **`type`** — coarse-grained category. Tells the integrator *which catch block* to use. Examples: `api_error`, `card_error`, `invalid_request_error`, `authentication_error`, `idempotency_error`, `rate_limit_error`.
- **`code`** — fine-grained machine-readable identifier. Stable across versions. Branch on this, not the message.
- **`decline_code`** — domain-specific subcode (only on certain errors). Lets the integrator distinguish `insufficient_funds` from `lost_card` from `expired_card`.
- **`message`** — human-readable. Safe to surface to *engineers*. Generally NOT safe to surface to end users without translation.
- **`param`** — which input field caused the error. Lights up form fields in the integrator's UI.
- **`doc_url`** — direct link to the error explanation. The integrator clicks once and reads the runbook.
- **`request_id`** — opaque server-side correlation ID. Goes into every response (success too, via header). Critical for support tickets.

### HTTP status codes

| Status | Meaning | Common `error.type` |
|---|---|---|
| 200 | Success | (no error) |
| 400 | Caller sent something invalid | `invalid_request_error` |
| 401 | Authentication failed | `authentication_error` |
| 402 | Payment-specific failure (Stripe-ism — usually card declined) | `card_error` |
| 403 | Authenticated but unauthorized | `authentication_error` |
| 404 | Resource doesn't exist (or you can't see it) | `invalid_request_error` |
| 409 | Idempotency conflict | `idempotency_error` |
| 429 | Rate limited | `rate_limit_error` |
| 5xx | Server fault | `api_error` |

The status code tells *category*. The body tells *specifics*. Integrators should branch on `error.type` and `error.code`, not on status.

### The `Stripe-Should-Retry` signal

For ambiguous failures (5xx, network errors), include guidance in a response header:

```
Stripe-Should-Retry: false
```

This shifts retry policy from heuristic to declarative. The server knows whether the operation is safe to retry; the client shouldn't have to guess.

Full reference: [`references/errors.md`](references/errors.md).

---

## 7. Versioning: never break a customer

The hardest part of API design isn't shipping v1 — it's shipping v2 without breaking v1 callers.

### Date-based rolling versions

```
Stripe-Version: 2024-10-28
```

Not `v1`/`v2`/`v3`. Each released version is a date. Each new release is a small, isolated set of breaking changes — never a complete rewrite. Accounts pin to a specific version on first request and stay there until they explicitly upgrade.

### What's backward-compatible (no version bump)

These changes can ship at any time, on any version:

- **Adding new endpoints**
- **Adding new optional request parameters**
- **Adding new response fields** to existing objects
- **Adding new event types** (caller must already handle unknown events)
- **Adding new enum values** (caller must already handle unknown enums — see below)
- **Reordering response fields**
- **Changing the format of opaque IDs** (`ch_xxx` getting longer is fine; the prefix stays)

### What's breaking (requires version bump)

- Removing a field
- Renaming a field
- Changing a field's type (boolean → enum, string → object)
- Removing or renaming an enum value
- Tightening validation (newly rejecting input that used to be accepted)
- Changing default behavior

### The response compatibility layer

This is the architectural trick that makes indefinite backward compatibility tractable.

```
[Core business logic — only knows the latest schema]
              ↓ produces a "modern" response
[Version transformer chain]
   2024-10-28 → 2024-09-30 → 2024-06-20 → ... → caller's pinned version
              ↓ transformed response
[Wire]
```

Internal code only ever writes the *current* version. A pipeline of small transformers walks the response backward in time, applying one delta per release until it matches the caller's pinned version. New engineers don't have to reason about old versions; old customers don't have to migrate.

For this to work, every breaking change must be expressible as a pure transformation from "new schema" to "old schema". If a change can't be expressed that way, it's not a versioning candidate — it's a new endpoint.

### Enum forward-compatibility

Document at the top of every SDK and API reference: **clients must tolerate unknown enum values**. Otherwise adding `status: "requires_capture"` next year breaks every client written this year. This is a *contract* you set with integrators, not a bug.

Full strategy in [`references/versioning.md`](references/versioning.md).

---

## 8. Webhooks

If your API has any async work, customers need webhooks. Same design philosophy applies.

### The event envelope

```json
{
  "id": "evt_1NG8Du2eZvKYlo2C9XMqbR8f",
  "object": "event",
  "type": "payment_intent.succeeded",
  "created": 1680893993,
  "livemode": false,
  "data": {
    "object": { "id": "pi_xxx", "object": "payment_intent", "...": "..." }
  },
  "api_version": "2024-10-28",
  "request": { "id": "req_xxx", "idempotency_key": "..." }
}
```

Properties:

- **`type`** is a dotted string: `<resource>.<verb>`. `customer.created`, `payment_intent.succeeded`, `invoice.payment_failed`. Let integrators subscribe to wildcards (`payment_intent.*`).
- **`data.object`** is the full resource at the time of the event. Don't make webhooks force a follow-up GET.
- **`api_version`** is pinned to the *endpoint's* version, not the account's. Lets integrators upgrade webhook handlers independently of API calls.
- **`request.idempotency_key`** echoes back the original write's key. Lets webhook handlers correlate to the originating call.

### Reliability rules

- **Sign every webhook.** HMAC-SHA256 over `timestamp + body`, sent as `Stripe-Signature: t=…,v1=…`. Include a timestamp to prevent replay attacks. Receivers reject signatures older than 5 minutes.
- **At-least-once delivery.** Webhooks WILL be redelivered. Receivers must dedupe by `event.id`.
- **Exponential backoff with cap.** Retry for ~3 days, exponentially backing off. After that, surface failures in a dashboard.
- **Order is not guaranteed.** A `customer.deleted` may arrive before `customer.created` if retries reorder them. Receivers must reconcile from the data, not from arrival order.
- **Idempotent receivers.** The receiver must process the same event twice safely. The `event.id` is the natural dedupe key.

### What NOT to webhook

- Synchronous responses to a request the caller is awaiting. That's just the response.
- Internal state changes that don't affect the customer's data model.
- Anything that would be cheaper to poll than to subscribe to.

---

## 9. Metadata: the extensibility escape hatch

Every meaningful resource has a `metadata` field:

```json
"metadata": {
  "internal_user_id": "9f4c2a8b",
  "campaign": "summer-launch-2026",
  "plan_grandfathered": "true"
}
```

Rules:

- **String → string.** No nesting, no numbers, no arrays. (Yes, integrators want richer types. The constraint forces them to encode their own and keeps the API stable.)
- **Bounded.** Stripe caps at ~50 keys, ~500 chars per value. Pick limits and document them.
- **Caller-owned.** The server never reads or interprets metadata. It's stored verbatim and returned verbatim. This is a *promise* — the moment you start branching server logic on metadata, it stops being metadata and you've broken the contract.
- **Indexable.** Allow filtering by `metadata[key]=value` in list endpoints. This is what makes metadata useful instead of just decorative.
- **Settable on every write.** Create, update, every action endpoint — metadata is always an accepted parameter.

Why this matters: every API will eventually face requests for "can you store this one extra field?" Metadata is the answer to 90% of them. Without it, every such request is a new schema field, a versioning concern, and a documentation burden.

---

## 10. Test mode as a first-class concept

The best APIs are the ones you can play with. Stripe's test mode rules:

- **Separate keys, separate IDs.** Test API keys produce test IDs that can never be confused with live IDs. The `livemode` field on every object makes this auditable.
- **Same endpoints.** No `/test/` prefix, no different URL. Just a different key. Code paths are identical.
- **Test fixtures.** Special card numbers (`4242 4242 4242 4242`), test bank accounts, simulated webhook events. Integrators need ways to trigger every error path without real money.
- **Time travel.** For subscriptions and other time-based resources, give test mode a way to advance the clock. Otherwise integrators can't test renewals without waiting a month.
- **Visible everywhere.** The dashboard, the docs, the request logs all show "TEST MODE" prominently. Mistakes between modes are catastrophic.

If your API doesn't have a test mode, you don't have a developer-friendly API.

---

## Anti-Patterns to Avoid

The mirror of the patterns above. If you catch yourself doing any of these, stop.

| Anti-pattern | Why it's bad | Do this instead |
|---|---|---|
| Numeric autoincrement IDs | Leak business scale, enable enumeration, type-coerce | Prefixed opaque strings |
| `/getUser?id=123` | Verb in URL, query param for a path concern | `GET /users/usr_123` |
| `{ "items": [...], "count": 100, "page": 2 }` | Offset pagination breaks under writes | Cursor pagination, `data` array, `has_more` |
| Per-resource list shapes (`{ customers: [...] }`) | Generic SDK code can't iterate | Universal list envelope |
| Letting clients write `status` directly | Bypasses business logic, breaks audit | Action sub-resources |
| Errors as plain strings | Not branchable, not localizable, not loggable | Structured `{type, code, message, param, doc_url}` |
| Fields renamed across versions | Breaks every client written before the rename | Add new field, deprecate old, never delete |
| Optional fields that are sometimes omitted, sometimes null | Two ways to mean "absent" — clients must handle both | Always present, value is `null` when absent |
| `created_at` as ISO string | Timezone ambiguity, parser variance | Unix epoch integer |
| Money as float | `0.1 + 0.2 !== 0.3` | Integer minor units (cents) |
| Webhooks without signatures | Replay/forgery vulnerable | HMAC-signed with timestamp |
| Webhooks expecting in-order delivery | Will never be in order under retries | Idempotent receivers, reconcile from data |
| `expand` by default | Inflates payloads, hides cost | Opt-in only |
| New required fields on existing endpoints | Breaks every existing client | Either optional with default, or new endpoint |

---

## The Decision Framework

When designing a new endpoint or resource, walk this:

1. **What is the noun?** If you can't name it without a verb, you're modeling an action, not a resource. Either find the noun (a `Refund` *is* the noun for "refunding") or make it an action sub-resource.
2. **What is its lifecycle?** Map the states. Each state transition that matters is a sub-resource action endpoint.
3. **What's its envelope?** `id`, `object`, `created`, `livemode`, `metadata`, then type-specific fields. No exceptions.
4. **What does it list with?** Define filters. Define sort order (almost always reverse-chronological by `created`). Confirm cursor pagination works.
5. **What references it? What does it reference?** Every reference is a `{thing}_id` string by default, expandable via `expand`.
6. **What can fail?** Enumerate every error type and code. Each error gets a `doc_url`.
7. **Is the create idempotent-safe?** All POSTs accept `Idempotency-Key`. Always.
8. **What webhook events does it emit?** `<resource>.<verb>` for every state transition.
9. **What's in metadata?** Confirm there's an extensibility escape hatch.
10. **What if we need to change this?** Walk the next-2-years scenarios. Which fields are likely to evolve? Are they shaped to allow additive change?

If any answer is "I'll figure it out later," you're not done. Once shipped, the answer becomes permanent.

For a complete pre-launch review, use [`references/checklist.md`](references/checklist.md).

---

## Reference Documents

- **[`references/resource-modeling.md`](references/resource-modeling.md)** — Deep dive on resource design, IDs, object envelopes, and state-machine endpoints
- **[`references/idempotency.md`](references/idempotency.md)** — Implementing idempotency keys end-to-end
- **[`references/versioning.md`](references/versioning.md)** — Date-based versioning and the response compatibility layer
- **[`references/errors.md`](references/errors.md)** — Error envelope, types, codes, and HTTP status mapping
- **[`references/checklist.md`](references/checklist.md)** — Pre-launch design review checklist

---

## Summary

1. **APIs are products. Developers are customers.** Every decision flows from that.
2. **Resource-oriented URLs, prefixed string IDs, consistent envelopes.** Predictability above all.
3. **One pagination shape, one list shape, one error shape.** Sameness is a feature.
4. **Idempotency keys on every write.** Make the right thing the easy thing.
5. **Date-based versioning + response compatibility layer.** Never break a customer.
6. **`expand` and `metadata`** are the two extensibility levers that prevent most "add this field" requests.
7. **Test mode is part of the product.** A staging environment is not a substitute.
8. **Errors are the most-read docs.** Treat them with the same care as the success path.
9. **Webhooks are signed, idempotent, at-least-once.** Receivers reconcile from data, not arrival order.
10. **Every endpoint shipped is a forever-promise.** Design slowly, version everything, never delete.
