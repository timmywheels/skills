# Lifecycle and State

Most records that matter have a lifecycle. How you store it decides whether you can ever measure, debug, or explain it.

---

## Status Columns

```sql
status VARCHAR(32) NOT NULL
```

- **Strings, not integers.** `past_due` reads itself; `4` needs a lookup table that someone will forget to update.
- **Lowercase snake_case values**, matching the API.
- **Add values, never repurpose them.** If `inactive` used to mean "paused" and now needs to mean "canceled", that's two new values and a migration, not a redefinition.
- **Enforce allowed values** with a `CHECK` constraint, an enum type, or a lookup table with a foreign key — but prefer the option that makes adding a value cheap.
- **Only named transitions write status.** Application code calls `invoice.finalize()`, `invoice.pay()`, `invoice.void()`; each validates the source state, stamps a timestamp, and emits an event. Nothing does a bare `UPDATE … SET status = ?`.

---

## A Timestamp Per Transition

`status` tells you where a record is. Timestamps tell you how it got there.

```sql
status                   VARCHAR(32)  NOT NULL,
created_at               TIMESTAMP(6) NOT NULL,
finalized_at             TIMESTAMP(6) NULL,
paid_at                  TIMESTAMP(6) NULL,
voided_at                TIMESTAMP(6) NULL,
marked_uncollectible_at  TIMESTAMP(6) NULL
```

It makes these trivial:

- Time-to-pay: `paid_at - finalized_at`
- "Invoices voided after finalization last month"
- Funnel drop-off by stage

Stamp a column for each transition someone will want to measure. Not every transition needs one; the ones that end up on a dashboard do.

### When to graduate to a transitions table

Add `<entity>_status_transitions` when you need *who* and *why*, or a state can be re-entered:

```sql
CREATE TABLE subscription_status_transitions (
  id               VARCHAR(32)  PRIMARY KEY,
  account_id       VARCHAR(32)  NOT NULL,
  subscription_id  VARCHAR(32)  NOT NULL,
  from_status      VARCHAR(32)  NULL,
  to_status        VARCHAR(32)  NOT NULL,
  actor_type       VARCHAR(20)  NOT NULL,
  actor_id         VARCHAR(32)  NULL,
  reason           VARCHAR(64)  NULL,
  created_at       TIMESTAMP(6) NOT NULL
);
```

Keep the `*_at` columns on the parent anyway; they are the cheap path for the common query.

---

## Intent vs. Attempt

When reaching an outcome can take several tries, model them separately.

| | Intent | Attempt |
|---|---|---|
| Cardinality | One per goal | Many per intent |
| Holds | What should happen: amount, currency, customer, capture method | What did happen: result, failure code, network response, risk outcome |
| Mutates | Yes — status advances | No — written once, maybe with one completion update |
| Examples | `payments`, `payment_method_setups`, `invoices`, `outbound_messages` | `payment_attempts`, `setup_attempts`, `delivery_attempts` |

Why it matters:

- Retries don't overwrite each other, so "how many attempts until success?" and "which decline codes do we see most?" are answerable.
- The intent can carry `last_payment_error_*` and `latest_attempt_id` for convenience without losing the full history.
- Retry scheduling lives on the intent (`attempt_count`, `next_payment_attempt`), not in a job queue only.

Record failures in full: `failure_code`, `failure_message`, and any external decline or advice codes (`network_decline_code`, `network_advice_code`). A failed attempt is often the most useful row in the table.

---

## Scheduled vs. Happened

Keep the plan and the fact in separate columns.

| Column | Meaning |
|---|---|
| `cancel_at_period_end` | Intent: the customer asked to stop at renewal |
| `cancel_at` | Plan: when it is scheduled to stop |
| `canceled_at` | Fact: when cancellation was requested or performed |
| `ended_at` | Fact: when service actually stopped |
| `trial_end` | Plan, until it passes |
| `pause_collection_resumes_at` | Plan |

Collapsing these destroys forecasting ("who churns next month?") and makes "undo cancellation" ambiguous.

The same split applies to anything scheduled: `scheduled_for` vs `sent_at`, `due_date` vs `paid_at`, `expires_at` vs `expired_at`.

---

## Pending Changes

If a change is requested but not yet in force (a downgrade at period end, a payment that must succeed before an upgrade applies), don't overwrite the live fields. Store them as `pending_update_*` columns on the record, or, when changes stack up over time, as a schedule of phases (`subscription_schedules` → `subscription_schedule_phases`). Live fields always describe what is true now.

---

## Questions to Ask

- If this record is stuck, can we tell from the data alone which step it's stuck on and since when?
- Which transitions will someone want to measure the time between? Do they each have a timestamp?
- Can a record leave a terminal state? If "yes, sometimes", is it really terminal?
- Who is allowed to cause each transition, and would we know afterward who did?
- What's the difference between "the customer canceled" and "we canceled them", and does the schema care?
- When the plan and the fact disagree (scheduled to end Friday, still running Monday), which part of the system notices?
