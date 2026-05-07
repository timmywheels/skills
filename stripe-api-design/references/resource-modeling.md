# Resource Modeling Deep Dive

The hardest part of API design is choosing what your resources are. The patterns below are how to make that choice consistently.

---

## The Resource Test

Before every new endpoint, ask: **is this a noun with a lifecycle, or is it an action?**

A resource is a noun with:

- An **identity** (a stable ID that survives across requests)
- A **lifecycle** (it gets created, updated, possibly canceled or deleted)
- **Properties** (fields that describe its state)
- **References** (foreign keys to other resources)

If you can't articulate identity + lifecycle + properties, you're not designing a resource — you're designing an RPC. RPCs occasionally belong (see "action endpoints" below), but most things are nouns and should be modeled as such.

### Example: refunds

A naive design says "refund a charge" is an action on the charge:

```
POST /v1/charges/:id/refund   ← RPC-shaped
```

But refunds have IDs (`re_xxx`), they have status, they emit webhooks, they can be partial, they have their own metadata. They're nouns. So:

```
POST /v1/refunds              ← create a Refund (with charge_id in body)
GET  /v1/refunds/:id          ← retrieve
GET  /v1/refunds              ← list (filterable by charge)
POST /v1/refunds/:id/cancel   ← state transition
```

The Refund is a first-class resource. The original charge becomes a foreign key reference, not the URL parent.

### When NOT to make something a resource

Some operations really are pure actions: `capture`, `cancel`, `confirm`, `void`. They mutate an existing resource without producing a new persistent thing. These belong as **action endpoints** on the existing resource:

```
POST /v1/payment_intents/:id/capture
POST /v1/payment_intents/:id/cancel
POST /v1/payment_intents/:id/confirm
```

The test: does this operation produce something with its own ID, lifecycle, and webhook events? If yes, it's a resource. If no (it's just transitioning the parent's state), it's an action.

---

## ID Design

### Anatomy of a good ID

```
cus_NffrFeUfNV2Hib
└┬┘ └─────┬──────┘
 │        └── 14+ chars of unguessable random
 └── 2-4 char type prefix + underscore
```

**Prefix conventions** (from Stripe's actual API):

| Prefix | Resource |
|---|---|
| `cus_` | Customer |
| `ch_`  | Charge |
| `pi_`  | PaymentIntent |
| `pm_`  | PaymentMethod |
| `re_`  | Refund |
| `sub_` | Subscription |
| `in_`  | Invoice |
| `ii_`  | InvoiceItem |
| `evt_` | Event |
| `req_` | Request (correlation ID) |
| `sk_`/`pk_` | Secret/Publishable API key |

Pick prefixes that are short but distinct. Avoid collisions: don't ship `c_` for both Customer and Charge.

### Test mode IDs

Test resources should be visually distinguishable:

```
Live:  cus_NffrFeUfNV2Hib
Test:  cus_test_NffrFeUfNV2Hib    or    cus_NffrFEUFTEST...
```

The exact convention doesn't matter much — what matters is that an engineer eyeballing a log can tell instantly whether they're looking at production data.

### Why not UUIDs?

UUIDs work, and they're better than autoincrement. But they're worse than prefixed IDs because:

1. **No type info.** `7c9e6679-7425-40de-944b-e07fc1f90ae7` — is that a customer? A charge? You have to look at context. Prefix tells you immediately.
2. **Visually noisy.** Hyphens, hex, fixed length. Prefixed IDs read easier in logs.
3. **Harder to grep.** `grep cus_` finds every customer. `grep` for UUIDs finds everything.

If you must use UUIDs internally for storage reasons, still expose prefixed IDs at the API boundary:

```python
internal_id = uuid4()                      # 7c9e6679-7425-40de-944b-e07fc1f90ae7
external_id = f"cus_{base62(internal_id)}" # cus_2eX9kPq3aR7nB1mLvZyT4u
```

### Why strings, not integers?

- **No overflow.** Integer IDs eventually exceed 2^53 and JavaScript clients silently corrupt them.
- **No leading-zero loss.** `00042` survives as a string; gets stripped as a number.
- **No coercion bugs.** PHP, JavaScript, and several JSON libraries coerce numeric strings to numbers and back, sometimes lossily.
- **Future-proof.** Strings can encode anything later (sharding, partitioning, region prefixes). Integers can't.

---

## The Object Envelope

Every resource shares a common spine. This is non-negotiable for a Stripe-style API.

```typescript
interface BaseObject {
  id: string;             // prefixed string
  object: string;         // discriminator: "customer", "charge", etc.
  created: number;        // Unix epoch seconds
  livemode: boolean;      // true for production, false for test
  metadata: Record<string, string>;  // caller-defined data
}
```

### Why each field is required

- **`id`** — obvious. Every resource is addressable.
- **`object`** — the polymorphic discriminator. Without it, a generic webhook receiver can't tell a `Customer` from a `Charge` from `event.data.object`. This single field is what makes Stripe's event system work without per-resource handlers.
- **`created`** — present on every object so generic sort/filter logic works. Stripe uses `created` (no `_at`) for historical reasons. New APIs should use `created_at` for the modern convention; pick one and never mix.
- **`livemode`** — the production-vs-test sentinel. Trivially small but catches expensive mistakes.
- **`metadata`** — the extensibility escape hatch. Even if you can't think of a use, ship it. Removing it later is a breaking change.

### Adding type-specific fields

Type-specific fields go alongside the spine, not nested:

```json
{
  "id": "ch_xxx",
  "object": "charge",
  "created": 1680893993,
  "livemode": false,
  "metadata": {},

  "amount": 2000,
  "currency": "usd",
  "customer": "cus_xxx",
  "status": "succeeded",
  "payment_method": "pm_xxx"
}
```

Resist the temptation to namespace under `attributes` or `properties`. Flat is better — it makes the JSON shorter, the SDK ergonomics simpler, and the docs easier to read.

---

## State Machines and Action Endpoints

Most non-trivial resources have a state field with a constrained lifecycle:

```
PaymentIntent:
   requires_payment_method → requires_confirmation → requires_action
        → processing → succeeded
                    → canceled (terminal)
                    → requires_capture (if manual capture)
                        → succeeded (after capture)
```

### Don't expose `status` as writable

The wrong design:

```
PATCH /v1/payment_intents/:id
{ "status": "succeeded" }       ← NEVER allow this
```

Why it's wrong:

- **No business logic.** A real "confirm" must validate the payment method, charge the card, and record the result. A status write skips all of that.
- **No audit.** "Why is this PI succeeded?" becomes unanswerable.
- **No webhooks.** State transitions emit events. A bare PATCH might skip the event chain.
- **No idempotency boundary.** Each transition has different retry semantics.

### Use action sub-resources for transitions

```
POST /v1/payment_intents/:id/confirm    → triggers the validate-and-charge flow
POST /v1/payment_intents/:id/capture    → captures an authorized payment
POST /v1/payment_intents/:id/cancel     → cancels (only valid in some states)
```

Each is its own endpoint with its own:

- Authorization scope (capture may require a different permission than confirm)
- Idempotency contract
- Validation rules (`cancel` is invalid if status is already `succeeded`)
- Webhook event (`payment_intent.canceled`, `payment_intent.succeeded`)
- Documentation page

### What about generic field updates?

A generic `PATCH /v1/payment_intents/:id` is fine for **non-state fields** — `description`, `metadata`, `receipt_email`. The rule is: anything that wouldn't fit naturally as a verb stays as PATCH.

```
PATCH /v1/payment_intents/:id        ← edit description, metadata, etc.
POST  /v1/payment_intents/:id/cancel ← cancel state transition
```

The PATCH endpoint validates that the caller isn't trying to write reserved fields like `status`, `amount` (after creation), or `currency`.

---

## Relationships

### One-to-many: filter on the list endpoint

```
GET /v1/charges?customer=cus_xxx
GET /v1/invoices?subscription=sub_xxx
```

Don't nest:

```
GET /v1/customers/:id/charges          ← AVOID
```

Reasons covered in SKILL.md §3.

### Many-to-many: a join resource

If two resources have a many-to-many relationship with its own attributes (a Subscription has many SubscriptionItems with quantity, price, etc.), the join is its own resource:

```
GET    /v1/subscription_items
POST   /v1/subscription_items
PATCH  /v1/subscription_items/:id
DELETE /v1/subscription_items/:id
```

`SubscriptionItem` has `subscription_id` and `price_id` as foreign keys, plus its own fields. It's a first-class resource with its own ID, webhook events, and metadata.

### One-to-one ownership: embed or expand

Owned resources that have no independent lifecycle (a Charge's `outcome` data) can be embedded directly on the parent:

```json
{
  "id": "ch_xxx",
  "object": "charge",
  "outcome": {
    "network_status": "approved_by_network",
    "risk_level": "normal",
    "seller_message": "Payment complete."
  }
}
```

No separate ID, no separate endpoint. Just a nested object on the parent.

If the owned resource is large but rarely needed, hide it behind `expand`:

```
GET /v1/charges/:id?expand[]=balance_transaction
```

---

## Singletons

Some resources are singletons — there's exactly one per account:

```
GET /v1/account
GET /v1/balance
```

No ID in the URL. Still has the standard envelope (`id`, `object`, etc.) — just no list endpoint and the URL doesn't take a path parameter.

---

## A Worked Example: Designing a "Reservation" Resource

Suppose you're building a booking platform and need a `Reservation` resource. Walk the framework:

**1. Is it a resource?** Yes — it has identity, lifecycle, and properties.

**2. ID prefix:** `res_` (short for reservation).

**3. Lifecycle:**

```
draft → pending_payment → confirmed → checked_in → completed
                       → canceled (terminal)
                       → no_show (terminal)
```

**4. Envelope:**

```json
{
  "id": "res_NffrFeUfNV2Hib",
  "object": "reservation",
  "created": 1680893993,
  "livemode": false,
  "metadata": {},

  "status": "confirmed",
  "guest_id": "gst_xxx",
  "property_id": "prp_xxx",
  "check_in": 1681516800,
  "check_out": 1681862400,
  "amount": 24500,
  "currency": "usd",
  "payment_intent_id": "pi_xxx"
}
```

**5. Endpoints:**

```
POST   /v1/reservations                       create (status starts as draft)
GET    /v1/reservations                       list, filterable by guest, property, status
GET    /v1/reservations/:id                   retrieve
PATCH  /v1/reservations/:id                   update non-state fields (metadata, notes)
POST   /v1/reservations/:id/confirm           draft → confirmed (triggers payment)
POST   /v1/reservations/:id/check_in          confirmed → checked_in
POST   /v1/reservations/:id/check_out         checked_in → completed
POST   /v1/reservations/:id/cancel            any non-terminal → canceled
POST   /v1/reservations/:id/mark_no_show      confirmed (past check_in) → no_show
```

**6. Webhook events:**

```
reservation.created
reservation.confirmed
reservation.checked_in
reservation.completed
reservation.canceled
reservation.no_show
```

**7. Filters on list:**

```
GET /v1/reservations?guest=gst_xxx&status=confirmed&check_in[gte]=1680000000
```

**8. Expand:**

```
GET /v1/reservations/:id?expand[]=guest&expand[]=property&expand[]=payment_intent
```

That's a complete resource design. Every endpoint is predictable from the framework. Integrators learn it once.
