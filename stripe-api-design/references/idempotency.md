# Idempotency Deep Dive

Network errors are inevitable. A robust API treats every retry as a first-class scenario, not an edge case. This is the implementation guide for getting it right.

---

## The Problem

A client sends `POST /v1/charges` with `{amount: 5000}`. The server processes it. The response gets lost mid-flight (TCP reset, gateway timeout, client timeout). The client has no idea whether the charge happened.

Three bad outcomes if you do nothing:

1. **Client retries.** Server processes again. Customer is double-charged.
2. **Client gives up.** Customer was charged but client thinks it failed. Inventory de-syncs.
3. **Client retries with different data.** Now you have two charges with conflicting amounts and no audit trail of which was the "real" one.

Idempotency keys solve all three.

---

## The Mechanism

The client generates a unique key per *logical operation* and includes it as a header:

```http
POST /v1/charges HTTP/1.1
Host: api.stripe.com
Authorization: Bearer sk_test_xxx
Idempotency-Key: a3b8f2c4-9d5e-4f7a-bc1d-0e2f6a8d9b3c
Content-Type: application/json

{"amount": 5000, "currency": "usd", "customer": "cus_xxx"}
```

The server stores the result of the first request keyed by `(account_id, idempotency_key)`:

```
{
  status_code: 200,
  body: { "id": "ch_xxx", "amount": 5000, ... },
  request_fingerprint: hash(request_body),
  created_at: 1680893993
}
```

Subsequent requests with the same key return the *cached result*, regardless of whether the original succeeded or failed.

---

## The Five Cases

### Case 1: First request

Client sends, server has no record of this key. Process normally, save result, return.

```
Server: 200 OK
{"id": "ch_xxx", ...}
```

### Case 2: Retry after success

Client retries (network blip, client crash, whatever). Server finds cached success.

```
Server: 200 OK
Idempotent-Replayed: true
{"id": "ch_xxx", ...}     ← same response as before
```

The `Idempotent-Replayed: true` header signals "this was a replay, not a fresh execution" — useful for client logging and metrics.

### Case 3: Retry after failure

Client retries. Server finds cached *failure*.

```
Server: 402 Payment Required
Idempotent-Replayed: true
{"error": {"type": "card_error", "code": "card_declined", ...}}
```

Yes, **failures are cached too**. Why? Because the client retrying after a 402 might be a bug in their retry logic, or it might be intentional. Either way, returning the same answer is the safe behavior. If they want a fresh attempt, they generate a new key.

### Case 4: Same key, different parameters

Client sends the same key but different request body.

```
Server: 400 Bad Request
{
  "error": {
    "type": "idempotency_error",
    "message": "Keys for idempotent requests can only be used with the same parameters they were first used with."
  }
}
```

This protects against bugs where a key gets reused accidentally. The server compares a fingerprint (hash) of the request body to the original; if they don't match, refuse.

### Case 5: Same key, original still in flight

Client sends a retry while the first request is still processing.

```
Server: 409 Conflict
{
  "error": {
    "type": "idempotency_error",
    "message": "There is currently a request in flight for this idempotency key. Try again later."
  }
}
```

This requires a lock per key. The lock is released when the first request finishes (either committing the cache entry or failing in a way that allows retry).

---

## Implementation

### Storage

```sql
CREATE TABLE idempotency_keys (
  account_id      TEXT NOT NULL,
  key             TEXT NOT NULL,
  request_path    TEXT NOT NULL,
  request_hash    TEXT NOT NULL,            -- SHA-256 of request body
  status_code     INTEGER,
  response_body   JSONB,
  locked_at       TIMESTAMP,                -- non-null while in flight
  created_at      TIMESTAMP NOT NULL DEFAULT NOW(),
  expires_at      TIMESTAMP NOT NULL,       -- created_at + 24h

  PRIMARY KEY (account_id, key)
);

CREATE INDEX idx_idempotency_expires ON idempotency_keys(expires_at);
```

### Request flow

```python
def handle_request(request):
    key = request.headers.get("Idempotency-Key")
    if not key:
        return process_normally(request)

    if request.method not in ("POST", "PATCH", "DELETE"):
        return error("Idempotency-Key not supported on GET")

    request_hash = sha256(canonical(request.body))

    with idempotency_lock(account_id, key):
        existing = lookup(account_id, key)

        if existing is None:
            # Case 1: first time
            insert_lock(account_id, key, request_path, request_hash)
            try:
                response = process_normally(request)
            except Exception as e:
                response = error_response(e)
            save_response(account_id, key, response)
            return response

        if existing.is_locked():
            # Case 5: original still running
            return error_409("request in flight")

        if existing.request_hash != request_hash:
            # Case 4: parameter mismatch
            return error_400("idempotency_error: parameter mismatch")

        if existing.request_path != request_path:
            # Same key on a different endpoint — also a mismatch
            return error_400("idempotency_error: endpoint mismatch")

        # Cases 2 & 3: replay
        return cached(existing.status_code, existing.response_body,
                      headers={"Idempotent-Replayed": "true"})
```

### The lock

`idempotency_lock` is critical. Without it, two concurrent retries can both miss the cache, both execute, and both commit. Implementations:

- **Postgres advisory locks** keyed by hash of `(account_id, key)`.
- **Redis SETNX** with a TTL.
- **A `locked_at` column** with a unique constraint that fails the second insert.

The lock must hold until the response is saved or the request fails in a recoverable way. If the server crashes mid-request, the lock should expire (use a TTL on the lock, ~30 seconds longer than your max request timeout).

### What to fingerprint

Hash the *canonicalized* request body — sorted keys, normalized whitespace, normalized booleans/numbers. Don't hash:

- Headers (other than the body content type)
- The auth token (it's already part of `account_id`)
- The Idempotency-Key itself (that's the lookup key)
- Query strings on POST endpoints (rarely meaningful)

```python
def canonical(body: dict) -> str:
    return json.dumps(body, sort_keys=True, separators=(',', ':'))
```

---

## Key Generation (Client Side)

Document this for integrators. Bad keys defeat the purpose.

### Good keys

- **UUIDv4.** The default choice. Cryptographically random, ~122 bits of entropy.
- **Deterministic from a business identifier:** `f"checkout-{order_id}"`, `f"refund-{charge_id}-1"`. This way, *retries from any process* — including a totally different worker — converge on the same key.
- **Per-attempt UUIDs** stored in your database before the request fires, so you can recover the key after a crash.

### Bad keys

- **Timestamps.** Two retries in the same millisecond collide.
- **Auto-increment IDs.** Same problem; also leak business data.
- **Hashes of the request body.** A hash is *not* an idempotency key. Different operations may have identical bodies legitimately. Use a UUID.
- **The Idempotency-Key from a previous endpoint.** Each operation needs its own key. Don't reuse "the request ID."
- **Long-lived "session" keys.** Any reuse beyond a single logical operation is wrong.

---

## Scope and Lifetime

### Scope

The lookup key is `(account_id, idempotency_key)` at minimum. Some implementations also include the endpoint path, so the same key on `/charges` and `/refunds` doesn't collide. Either approach works as long as it's documented.

**Do not scope by user/IP/session.** The whole point is that retries from different processes should converge.

### Lifetime

Stripe's v1 keys live 24 hours. v2 lives 30 days. Pick one:

- **Short (24h)** keeps the storage small and forces clients to use fresh keys for retries beyond a day. Reasonable for high-volume APIs.
- **Long (30d)** is more forgiving for batch retry logic and offline clients. Higher storage cost.

After expiry, the same key starts fresh. **Document this clearly** — clients sometimes assume keys live forever.

### Cleanup

Run a periodic job to delete expired entries. Don't query the table without a `created_at` predicate; it'll grow unbounded.

---

## What Endpoints Need Idempotency Keys

| Endpoint type | Needs key? |
|---|---|
| `POST` that creates state | **Yes** |
| `POST` that mutates state (action endpoints) | **Yes** |
| `PATCH` | **Yes** (technically PATCH should be idempotent, but in practice retry semantics require the key for the cache replay) |
| `DELETE` | **Optional** — DELETE is supposed to be idempotent by HTTP spec, but adding key support gives clients a consistent retry story |
| `GET` | **No** — already idempotent |
| `HEAD` / `OPTIONS` | **No** |

**Make it accept-but-not-require for the first version.** Then encourage it loudly in docs and in the SDK. Then make it required when you ship v2.

---

## The Subtle Bugs

### Don't cache the response *before* fully committing the side effect

Wrong:

```python
charge = create_charge(...)          # money moves
log_audit(charge)                    # audit log written
save_idempotency_response(charge)    # cached
notify_webhooks(charge)              # ← if this fails and the client retries,
                                     #   we'll replay the cache without re-emitting
```

If anything between "money moved" and "cache committed" can fail and matter, you'll quietly skip it on retry. Either:

- Put everything in a single transaction including the cache write, OR
- Make every step idempotent on the resource ID (e.g., webhook delivery dedupes by event ID).

### The compare-parameters check must canonicalize

If your fingerprint includes JSON object key order, two semantically identical requests will look different. Always canonicalize before hashing.

### Don't include the Idempotency-Key in the hash

Otherwise a client retrying with the same key always matches itself trivially, and parameter mismatch detection breaks.

### Be careful what you store in the response cache

The cached response will be served back, possibly hours later. If it contains time-sensitive fields (signed URLs, ephemeral tokens), the replay returns expired values. Either:

- Mark such fields as not cacheable and regenerate on replay, OR
- Document that idempotency replays may contain stale time-sensitive data, OR
- Don't cache the response body for endpoints that produce time-sensitive output.

---

## Returning Idempotency Errors

Errors caused by idempotency violations get their own type:

```json
{
  "error": {
    "type": "idempotency_error",
    "message": "Keys for idempotent requests can only be used with the same parameters they were first used with.",
    "doc_url": "https://docs.example.com/errors#idempotency-error"
  }
}
```

This is distinct from `invalid_request_error` because the *fix* is different: integrator must change either the key or the parameters, not the auth or the routing.

---

## Summary

1. **Caller-generated keys, server-cached responses.** That's the whole protocol.
2. **Cache failures too** — replays return the same answer regardless.
3. **Lock per key while in flight** — otherwise concurrent retries both execute.
4. **Fingerprint the body** — different params with the same key is an error.
5. **24h–30d lifetime** — long enough for retries, short enough for storage.
6. **Required on every state-changing endpoint** — accept-then-encourage-then-require.
7. **Document the contract clearly** — keys are a contract with the integrator, not an implementation detail.
