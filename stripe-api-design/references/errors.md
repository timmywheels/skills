# Errors Deep Dive

Errors are the most-read part of your API. An integrator stares at error responses while their feature is broken and a deadline is approaching. The error you serve them is the difference between a 5-minute fix and a 5-hour debugging session.

Treat error design with the same seriousness as the success path.

---

## The Error Envelope

Every error response is a single object with one top-level key:

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

Why a single `error` wrapper instead of fields at the top level?

- **Distinguishable.** Success and error responses have *different shapes*. A client checking "does the response have an `error` key?" is unambiguous.
- **Future-proof.** Lets you add error-level metadata (`error.retry_after`, `error.suggestion`) without polluting the success namespace.
- **Maps cleanly to exceptions.** SDKs convert this to a typed exception class trivially.

---

## The Six Required Fields

### `type` — coarse category

Branchable, stable, finite set. Tells the integrator *which catch block* to use.

The taxonomy:

| Type | Meaning | HTTP status (typical) |
|---|---|---|
| `api_error` | Server-side problem; not the caller's fault | 5xx |
| `authentication_error` | Auth missing, invalid, or insufficient | 401, 403 |
| `invalid_request_error` | Caller sent a malformed or invalid request | 400, 404 |
| `rate_limit_error` | Caller is rate-limited | 429 |
| `idempotency_error` | Idempotency key violation | 400, 409 |
| `card_error` | Payment-specific failure (declined, expired, etc.) | 402 |
| `validation_error` | Optional alternative to invalid_request_error for schema-level issues | 400 |

Pick a small set (~5–8) and stick to it. Each is a *different problem class* that a thoughtful integrator handles differently:

- `api_error` → log it, retry with backoff, alert on persistence
- `authentication_error` → check credentials; don't retry
- `invalid_request_error` → fix the code that constructed the request; don't retry
- `rate_limit_error` → back off and retry
- `idempotency_error` → fix the key generation logic
- `card_error` → surface to the end user; offer to try another card

### `code` — fine-grained identifier

Machine-readable, stable across versions, scoped under `type`. Examples:

For `card_error`:
- `card_declined`
- `expired_card`
- `incorrect_cvc`
- `processing_error`
- `incorrect_number`

For `invalid_request_error`:
- `parameter_missing`
- `parameter_invalid_empty`
- `parameter_invalid_integer`
- `resource_missing`
- `resource_already_exists`

For `authentication_error`:
- `api_key_expired`
- `api_key_invalid`
- `permission_denied`

The relationship: `code` is a refinement of `type`. Integrators branch on `type` first, then drill into `code` only if they need to.

### `message` — human-readable

Safe to surface to *engineers* (in logs, in dashboards). Generally **NOT safe to surface to end users**:

- Leaks implementation details ("Database connection failed")
- Leaks security info ("User exists but password is wrong")
- Untranslated (always in English by default)
- Verbose for engineers, confusing for end users

If you want user-safe messages, that's a separate concept — `user_message`, perhaps, with explicit i18n. The default `message` is for the developer building the integration.

### `param` — which input caused it (if any)

Identifies the request parameter that caused the validation error:

```json
{
  "error": {
    "type": "invalid_request_error",
    "code": "parameter_missing",
    "message": "Missing required param: customer.",
    "param": "customer"
  }
}
```

Lets the integrator's UI map errors to form fields automatically. The value is the *parameter path* — for nested objects, dotted notation: `"shipping.address.line1"`.

Include `param` only when the error is caused by a specific input. For server errors, omit it.

### `doc_url` — link to the runbook

```json
"doc_url": "https://docs.example.com/error-codes#card-declined"
```

This is the highest-leverage field after `type`. The integrator clicks once and reads:

- What this error means
- Common causes
- How to handle it in code
- Whether to retry, surface to user, or escalate

If you don't have docs for an error code, you don't fully understand it yet. Write the doc first, then ship the code.

### `request_id` — correlation

```json
"request_id": "req_NffrFeUfNV2Hib"
```

Every response — success and error — gets a unique request ID. Stripe puts it in both:

- The response body (in `error.request_id` on errors, top-level on successes)
- The `Request-Id` header on every response

Why both? The body version makes it visible in client-side logs that may not capture headers. The header version is universally available.

The request ID is the bridge between the integrator's report and your server logs. When a customer files a support ticket with `req_NffrFeUfNV2Hib`, you should be able to pull the entire request trace from your logs by that ID. Plan the logging backend around this.

---

## Optional but Useful Fields

### `decline_code` — payment-specific subcode

Only on card errors. Distinguishes between very different failure modes that share `code: card_declined`:

```json
"code": "card_declined",
"decline_code": "insufficient_funds"
```

```json
"code": "card_declined",
"decline_code": "stolen_card"
```

The integrator handles these very differently — `insufficient_funds` deserves "try another card", `stolen_card` deserves "contact your bank and *don't* offer to retry."

This pattern (a domain-specific subcode under a general code) generalizes. If your API has a domain with rich failure modes — fraud detection, content moderation, identity verification — give that domain its own subcode field.

### `retry_after` — when to retry

For `rate_limit_error`:

```json
{
  "error": {
    "type": "rate_limit_error",
    "code": "rate_limit_exceeded",
    "message": "Too many requests.",
    "retry_after": 30
  }
}
```

Number of seconds. Also surface in the standard `Retry-After` HTTP header.

### `payment_intent` (or other resource) — embedded context

Some errors come *with the resource* they happened on:

```json
{
  "error": {
    "type": "card_error",
    "code": "card_declined",
    "message": "Your card was declined.",
    "payment_intent": {
      "id": "pi_xxx",
      "object": "payment_intent",
      "status": "requires_payment_method",
      "...": "..."
    }
  }
}
```

The integrator now has the full PaymentIntent state without a follow-up GET. Useful when an action endpoint fails partway and the resource is in a meaningful intermediate state.

---

## HTTP Status Code Mapping

Status codes tell the *category*; the body tells the *specifics*. Don't make integrators branch on status alone — give them `type` and `code`.

| Status | When | Notes |
|---|---|---|
| 200 OK | Success | Even some "soft failures" are 200 (a charge that's `requires_action` is still a successful API call) |
| 201 Created | Optional alternative to 200 for create endpoints; Stripe uses 200 throughout | Pick one and stay consistent |
| 204 No Content | DELETE that returns nothing | Some APIs return the deleted object; Stripe does. Both are valid. |
| 400 Bad Request | Caller sent something invalid | The most common error response |
| 401 Unauthorized | No auth or invalid auth | "I don't know who you are" |
| 402 Payment Required | Stripe-ism — usually a card decline | Standard HTTP says this is reserved/unused; Stripe overloaded it |
| 403 Forbidden | Authenticated but lacks permission | "I know who you are; you can't do this" |
| 404 Not Found | Resource doesn't exist or you can't see it | Use when the resource genuinely doesn't exist OR when revealing existence would leak info |
| 409 Conflict | Idempotency conflict, or state-machine violation | "You can't cancel an already-canceled subscription" |
| 422 Unprocessable Entity | Optional — semantic validation error | Stripe doesn't really use 422; many APIs use 400 for everything client-side |
| 429 Too Many Requests | Rate limited | Always include `Retry-After` |
| 500 Internal Server Error | Generic server fault | Should be rare in steady state |
| 502 Bad Gateway | Upstream dependency failed | |
| 503 Service Unavailable | Server overloaded or in maintenance | Include `Retry-After` |
| 504 Gateway Timeout | Upstream timed out | Caller should retry with idempotency key |

**Don't invent custom statuses.** Use the standard ones. Integrators' HTTP libraries already know how to handle them.

**Don't return 200 for errors.** Some APIs do this ("everything is 200, check the body"). It defeats every standard tool: load balancers, monitoring, retry libraries, even browsers' devtools all use status codes to make decisions.

---

## Retry Signaling

For ambiguous failures (5xx, network errors), tell the client whether to retry:

```http
HTTP/1.1 503 Service Unavailable
Stripe-Should-Retry: true
Retry-After: 30
```

```http
HTTP/1.1 500 Internal Server Error
Stripe-Should-Retry: false
```

Why this matters: a 500 might be a transient blip (retry) or a deterministic bug (don't retry, you'll just blow up the logs). The server knows which; the client shouldn't have to guess.

The header is `Stripe-Should-Retry: true | false` plus the absence of the header (interpret as "use your own judgment, default backoff"). Document the precedence clearly.

For 4xx errors, generally **don't retry** — the request was bad. The caller needs to fix something.

---

## Validation Errors

When a request has multiple problems, you have two options:

### Option A: First-error-wins (Stripe's choice)

Return one error for the first failure encountered, with `param` pointing to the offending field:

```json
{
  "error": {
    "type": "invalid_request_error",
    "code": "parameter_missing",
    "message": "Missing required param: customer.",
    "param": "customer"
  }
}
```

Pros: simple, fast, predictable. Cons: integrator fixes one error, retries, gets the next one.

### Option B: All-errors-at-once

Return an array of validation errors:

```json
{
  "error": {
    "type": "validation_error",
    "code": "multiple_errors",
    "message": "The request failed validation.",
    "errors": [
      { "code": "parameter_missing", "param": "customer" },
      { "code": "parameter_invalid_integer", "param": "amount" }
    ]
  }
}
```

Pros: integrator sees everything at once. Cons: more complex error structure; some errors mask others.

**Recommendation: Option A by default**, especially for state-changing endpoints. Use Option B only for endpoints that are clearly schema-validating bulk submissions (CSV imports, batch creates).

---

## Errors in SDKs

The error envelope should map cleanly to typed exceptions in every official SDK:

```python
try:
    stripe.Charge.create(amount=2000, currency="usd", source="tok_xxx")
except stripe.error.CardError as e:
    # User's card was declined
    print(e.code, e.decline_code, e.user_message)
except stripe.error.RateLimitError as e:
    # Too many requests
    time.sleep(e.retry_after)
except stripe.error.InvalidRequestError as e:
    # Bug in our code constructing the request
    raise
except stripe.error.AuthenticationError as e:
    # API key is wrong
    raise
except stripe.error.APIError as e:
    # Server-side problem
    if e.should_retry:
        retry_with_backoff()
```

The mapping is:

```
error.type "card_error"           → CardError
error.type "rate_limit_error"     → RateLimitError
error.type "invalid_request_error"→ InvalidRequestError
error.type "authentication_error" → AuthenticationError
error.type "idempotency_error"    → IdempotencyError
error.type "api_error"            → APIError
```

This is why `type` must be a small, stable set. Each one becomes a class in every SDK in every language. Adding a new `type` is a coordinated, breaking-ish change.

---

## Error Documentation

For each error code, the docs should answer:

1. **What does this mean?** Plain English explanation.
2. **What causes it?** Common scenarios.
3. **How do I fix it?** Either "fix your request like X" or "show this UI to the user."
4. **Should I retry?** Yes / no / yes with backoff.
5. **Code example.** Idiomatic handler in each SDK language.

If you can't write all five, you don't fully understand the error yet — go figure it out before shipping.

---

## Anti-Patterns

| Anti-pattern | Why bad |
|---|---|
| Returning 200 with an error body | Defeats every standard tool |
| Inconsistent error shapes per endpoint | Integrators write a different handler per endpoint |
| Errors as plain strings (`"error": "Card declined"`) | Not branchable, not actionable, not internationalizable |
| Stack traces in error messages | Leaks implementation, hostile to debugging |
| Generic 500 for every server-side issue | Operations team can't diagnose without logs |
| `error.code` that varies by version | Breaks integrators every release |
| No request ID | Support tickets become impossible |
| Surfacing raw DB errors | "duplicate key value violates unique constraint customers_email_key" — what does an integrator do with that? Translate to a domain code. |
| HTTP 200 + `{"success": false}` instead of 4xx | Same as "200 with error body" — anti-pattern. |

---

## Summary

1. **Single `error` wrapper** with a stable shape across every endpoint.
2. **Required fields:** `type`, `code`, `message`, `request_id`. Add `param`, `doc_url`, `decline_code` where appropriate.
3. **Coarse `type`** for catch-block routing; fine `code` for specific handling.
4. **Status code matches category;** body matches specifics. Branch on `type`, not status.
5. **Every error has docs** with a `doc_url`. Write them first.
6. **`Stripe-Should-Retry`** disambiguates retryable vs not.
7. **Errors are part of the SDK type system** — each `type` is a class.
8. **`request_id` is the bridge** to your server logs. Plan around it.
