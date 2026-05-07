# API Design Review Checklist

Stripe is famous for funneling every new endpoint through a lightweight cross-functional review before launch — written up in a brief design doc, circulated to a mailing list, blocked on objections. The reason it works: a published endpoint is permanent, so the cost of getting it wrong compounds forever, while the cost of an extra day of review is one day.

Use this checklist before publishing any new endpoint, resource, or breaking change. If you can't tick every box, the design isn't done.

---

## 1. Resource Identity

- [ ] **The resource is a noun**, not a verb-disguised-as-a-noun. (`Refund`, not `RefundOperation`.)
- [ ] **The name is plural** in URLs (`/refunds`, not `/refund`).
- [ ] **The name uses snake_case** (`/payment_intents`, not `/paymentIntents`).
- [ ] **The name is one I can live with for a decade.** No clever or trendy terms.
- [ ] **No abbreviations** that aren't industry-standard (`customer`, not `cust`).

## 2. ID Format

- [ ] **Prefixed string** with a 2-4 character type prefix (`cus_`, `re_`, `pi_`).
- [ ] **The prefix doesn't collide** with any existing resource's prefix.
- [ ] **Test mode IDs are visually distinguishable** from live mode IDs.
- [ ] **The body is opaque** — no embedded business data, no sequential ordering.
- [ ] **The body has enough entropy** that IDs are unguessable (≥16 chars of base62, or equivalent).
- [ ] **IDs are strings, not integers**, in both API responses and SDK types.

## 3. Object Envelope

Every response object includes:

- [ ] `id` — the prefixed string identifier.
- [ ] `object` — discriminator string equal to the resource name (`"customer"`, `"refund"`).
- [ ] `created` (or `created_at`) — Unix epoch seconds, integer.
- [ ] `livemode` — boolean.
- [ ] `metadata` — string→string map, caller-defined.
- [ ] **Type-specific fields are flat**, not nested under `attributes` or `properties`.

## 4. CRUD Endpoints

For a typical resource, you have:

- [ ] **Create:** `POST /v1/{resource}` returns the created object.
- [ ] **Retrieve:** `GET /v1/{resource}/{id}` returns the object or 404.
- [ ] **Update:** `PATCH /v1/{resource}/{id}` (or `POST` for v1-style APIs) returns the updated object.
- [ ] **Delete:** `DELETE /v1/{resource}/{id}` — returns the deleted object or `{deleted: true}`.
- [ ] **List:** `GET /v1/{resource}` returns the list envelope.
- [ ] **No `/getX`, `/createX` URLs.** Methods are HTTP verbs.

## 5. State Machine (if applicable)

- [ ] **The lifecycle is documented** as a state diagram.
- [ ] **Each state transition has its own action endpoint**: `POST /v1/{resource}/{id}/{action}`.
- [ ] **The status field is read-only** — never PATCHable directly.
- [ ] **Each transition emits a webhook event**: `{resource}.{verb}`.
- [ ] **Invalid transitions return 409** with a clear error message.

## 6. List Endpoint

- [ ] **List envelope:** `{ object: "list", url, has_more, data }`.
- [ ] `data` is the array (not `items`, `results`, or the resource name).
- [ ] **Cursor pagination** with `starting_after` and `ending_before`. **No offset/page params.**
- [ ] `limit` parameter with a documented default (typically 10) and max (typically 100).
- [ ] **Filterable by every meaningful field**, using the field's name as the query param.
- [ ] **Range filters** use bracket suffixes: `created[gte]`, `created[lt]`.
- [ ] **Sort order is stable and documented** (typically reverse-chronological by `created`).

## 7. Expansion

- [ ] **Foreign keys default to ID strings** in responses.
- [ ] `expand` parameter is supported on retrieve, create, update, and list endpoints.
- [ ] **Dot notation** for nested expansion: `expand[]=invoice.subscription`.
- [ ] **Maximum depth is documented** (Stripe uses 4).
- [ ] **List endpoints use `data.` prefix**: `expand[]=data.customer`.
- [ ] **Default response is not expanded** — opt-in only.

## 8. Idempotency

- [ ] `Idempotency-Key` header is **accepted on every state-changing endpoint** (POST, PATCH, DELETE).
- [ ] **First request executes normally**; saved response keyed by `(account_id, key)`.
- [ ] **Retries with the same key + same body** return the cached response.
- [ ] **Retries with the same key + different body** return `idempotency_error`.
- [ ] **Retries while original is in flight** return 409.
- [ ] **Replays include `Idempotent-Replayed: true`** header.
- [ ] **Failures are cached too**, not just successes.
- [ ] **Key lifetime is documented** (24h or 30d typical).

## 9. Errors

- [ ] **Every error response is a single `error` object** with stable shape.
- [ ] **Required fields:** `type`, `code`, `message`, `request_id`.
- [ ] `type` is from a small, stable taxonomy (~5–8 values).
- [ ] `code` is fine-grained, scoped under `type`, machine-readable.
- [ ] `param` identifies the offending field for input errors.
- [ ] `doc_url` links to a real, written runbook page.
- [ ] **HTTP status code matches category** (4xx for caller errors, 5xx for server errors).
- [ ] **Never return 200 with an error body.**
- [ ] `Stripe-Should-Retry` header signals retryability for ambiguous failures.
- [ ] **Every error code has documentation** that answers: what is it, what causes it, how to fix, should I retry, code example.

## 10. Versioning

- [ ] **The change is classified** as backward-compatible or breaking.
- [ ] **If backward-compatible:** can ship anytime; no version bump needed.
- [ ] **If breaking:** new dated version is created; transformer chain updated; old version still works for pinned accounts.
- [ ] **Transformers are pure functions**, one concern each.
- [ ] **The contract that clients tolerate unknown fields/enums/events is clearly documented.**
- [ ] **No version-conditional code in core business logic.** Versioning lives in the transformer layer.
- [ ] **Old versions are never sunset** — pinned accounts keep working forever.

## 11. Webhooks (if applicable)

- [ ] **Event type follows `{resource}.{verb}` convention.**
- [ ] **Event envelope** includes `id`, `object: "event"`, `type`, `created`, `livemode`, `data.object`, `api_version`, `request`.
- [ ] **`data.object` is the full resource** at the time of the event — receivers don't need to fetch.
- [ ] **Events are signed** with HMAC-SHA256 over `timestamp + body`, sent as `X-Signature: t=…,v1=…`.
- [ ] **Receivers are documented to dedupe by `event.id`** (at-least-once delivery).
- [ ] **Receivers are documented to not rely on event order.**
- [ ] **Webhook endpoints have their own version pin**, separate from account API version.
- [ ] **Retry policy is documented** — exponential backoff, ~3 days, then dead-letter.
- [ ] **Signature verification rejects timestamps older than 5 minutes** (replay protection).

## 12. Metadata

- [ ] `metadata` field is **present on every meaningful resource**.
- [ ] **Type:** string → string only. No nesting, no other types.
- [ ] **Bounds documented:** key count limit, value length limit.
- [ ] **Server never reads metadata** — it's pure caller-owned storage.
- [ ] **Metadata is settable on every write** (create, update, action endpoints).
- [ ] **List endpoints filter by metadata:** `metadata[key]=value`.

## 13. Test Mode

- [ ] **Test API keys are visually distinct** from live keys (`sk_test_` vs `sk_live_`).
- [ ] **Test resources have visually distinct IDs.**
- [ ] **`livemode` field on every object** correctly reflects which mode produced it.
- [ ] **Same endpoints in both modes** — no `/test/` URL prefix.
- [ ] **Test fixtures exist** for triggering every error path (declined cards, failed bank transfers, etc.).
- [ ] **Time-based features have a test-mode time-travel mechanism** (subscriptions, scheduled jobs).
- [ ] **Dashboard shows TEST MODE prominently.**

## 14. Documentation

- [ ] **Every endpoint has a documented page** with: description, parameters, returns, example request, example response, possible errors.
- [ ] **Every error code has a doc page** that the `doc_url` field points to.
- [ ] **Code samples in at least 3 languages** (curl + 2 SDKs).
- [ ] **Three-column layout** with navigation, prose, and code samples (Stripe convention; other layouts also fine if they keep code visible).
- [ ] **Versioning notes** on any field that changed across versions.
- [ ] **Webhook event reference** lists every event type, when it fires, and the shape of `data.object`.
- [ ] **Migration guides** for every breaking version.

## 15. SDK Ergonomics

- [ ] **The wire format maps cleanly to typed objects** in every official SDK language.
- [ ] **Errors map to typed exception classes** — one class per `error.type`.
- [ ] **List endpoints expose auto-pagination iterators**: `for x in client.customers.list().auto_paging_iter():`.
- [ ] **Idempotency keys are first-class options**, not buried in headers.
- [ ] **`expand` is exposed as a typed parameter**, with union types for "string | object" fields.
- [ ] **No language-specific quirks bleed through** to the wire format.

## 16. Security

- [ ] **Auth is via bearer token** in `Authorization: Bearer sk_xxx`.
- [ ] **Test and live keys are different** and can never be mixed in one request.
- [ ] **Webhook signatures are HMAC with timestamp**; receivers reject old timestamps.
- [ ] **No sensitive data in URLs** — everything goes in headers or body.
- [ ] **No sensitive data in error messages** that could leak to end users.
- [ ] **Rate limits per account**, with `Retry-After` and `X-RateLimit-Remaining` headers.
- [ ] **PII / PCI fields are gated** behind explicit permissions or expand options.

## 17. Operational

- [ ] **Every request gets a `request_id`** in both response body and `Request-Id` header.
- [ ] **Server logs are queryable by request ID.**
- [ ] **Latency targets are documented** (p50, p99) and monitored.
- [ ] **Idempotency cache size and TTL are monitored.**
- [ ] **Webhook delivery success rate is monitored** per endpoint.
- [ ] **Deprecated fields/endpoints are tracked** and have a sunset plan.

---

## The Five Hardest Questions

If you can answer these five, the design is probably solid. If you can't, go back.

1. **What does this look like five years from now?** If the resource has accumulated 30 fields, will today's structure still make sense? Are the "core" fields the ones future callers will need most, or the ones that were convenient to ship today?

2. **What's the smallest breaking change that would force a v2?** If the answer is "almost any change", the model is too rigid. If the answer is "we'd need to rewrite everything", the model is too monolithic. The right answer is "we have escape hatches (metadata, expand, action endpoints) that make most foreseeable changes additive."

3. **How does an integrator handle every error path?** Walk through each error type. For each: do they retry? Surface to user? Fix code? Escalate to support? If any answer is "I'm not sure", the docs aren't done.

4. **Can I generate an SDK from this?** A good API has a tight enough spec that a code generator can produce idiomatic SDKs in every major language. If the SDK requires per-endpoint hand-tuning, the API is inconsistent.

5. **Would I want to integrate with this on a Friday at 5pm?** The "Friday afternoon test." If a developer hitting a deadline can integrate the happy path in 30 minutes by reading three doc pages, the API passes. If they need to read the whole reference and Slack their friend at the company, it doesn't.

---

## When You Find a Mistake After Shipping

You will. Stripe has. Everyone has. The playbook:

1. **Don't break existing callers.** The old behavior keeps working at the old version.
2. **Add the corrected version** as a new dated release.
3. **Document the difference** clearly in the migration guide.
4. **Quietly deprecate the old version** but never sunset it.
5. **Watch new accounts** — they pin to the new version automatically.
6. **Email any high-volume customers still on the old version** when their next migration is convenient.

Permanence is a feature, not a constraint. It's the thing that makes the API trustworthy.
