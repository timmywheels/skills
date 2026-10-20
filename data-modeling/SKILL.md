---
name: data-modeling
description: Design application database schemas that can be trusted with money and history for decades — money as exact integers, ledgers instead of mutable balances, explicit state machines, immutable catalogs, point-in-time snapshots, tenant-scoped rows, and additive evolution. Use when creating or altering tables, writing migrations, modeling money, balances, payments, subscriptions, invoices, orders, or any record with a lifecycle, designing multi-tenant schemas, or reviewing a data model or migration. Triggers on tasks involving schema design, data models, ERDs, CREATE TABLE / ALTER TABLE, ledgers, audit trails, soft deletes, or status columns.
metadata:
  author: timmywheels
  version: "1.0"
---

# Data Modeling

The best financial and SaaS platforms make the same handful of schema decisions over and over, across hundreds of tables. None of them is clever on its own. Together they produce a database where every number can be explained, every past state reconstructed, and every change shipped without breaking whoever reads the data next.

This skill captures those decisions for the **storage layer** of an application: the tables, columns, keys, and invariants behind the API. It pairs with [`stripe-api-design`](../stripe-api-design/SKILL.md), which covers the wire format. Where the two differ (timestamps, for example), store the richest correct type and convert at the API boundary.

---

## The Core Philosophy

> **A row is a fact. Facts are recorded, not edited.**

Every rule below follows from it:

1. **Any past state must be explainable.** If support asks "why did this customer owe $40 on March 3rd?", the answer is a query, not an archaeology project.
2. **Derived values are derived.** A balance, a total, an MRR figure — each one is the sum of facts you can point to. Caching it is fine. Making it the only copy is not.
3. **History outlives the code that wrote it.** Rows written today will be read by code that doesn't exist yet. Name, type, and constrain them for that reader.
4. **The schema only grows.** Columns and tables are added, never renamed or repurposed in place. Old shapes stay readable.

If a table can't answer "what happened, when, and why" for its own rows, it isn't done.

---

## Think Before You Type `CREATE TABLE`

Most bad schemas are not bad SQL. They are correct answers to a question nobody asked out loud. Before reaching for patterns, work through these with whoever owns the feature. They are not a checklist to tick; they are prompts to argue over. When the honest answer is "we don't know", that is the thing to settle before writing DDL.

### Purpose

- What are we actually trying to remember here — the current state of a thing, or what happened to it?
- What decision will someone make from this data, and what would they need to see to make it well?
- What question will we be embarrassed not to be able to answer a year from now?
- If this table vanished tomorrow, what would break, and who would notice first?
- Are we modeling the business, or modeling the screen we're building this sprint?

### Truth

- Is this value something we observed, something we decided, or something we calculated? Each one deserves different handling.
- Where does the truth for this live? If we're copying it here, what keeps the copy honest — and is it a copy, or a snapshot that is *supposed* to diverge?
- If two rows disagree, which one wins, and how would we even find out they disagree?
- Whose meaning of this word are we using? Does finance mean the same thing by "active" as product, support, and the customer do?
- What would the customer call this? If our name differs from theirs, why?

### Time

- Can we reconstruct exactly what this looked like last Tuesday at 3pm?
- Is this true forever, true until someone changes it, or true only for a period?
- When did it happen versus when did we learn about it? Do we ever need both?
- In whose timezone is "today", "this month", or "the billing date"?
- What happens to this row when the rule that produced it changes next year — is it recomputed, or does it keep the old answer?

### Change

- What is the most likely next requirement, and does it fit as an addition or force a rewrite?
- What will we wish we had captured at write time that is impossible to recover later?
- Are we encoding a business rule into the shape of the data that the business will want to change?
- Is this one thing, or two things that happen to share fields today?

### People and Boundaries

- Who is allowed to see this row, and does the row itself carry enough to enforce that?
- Who changed it, and could support explain the change to an angry customer from the data alone?
- What here is personal data, and what does deleting a person mean for the records they touched?

### Failure

- What happens if this write runs twice? Half runs? Runs out of order with another one?
- What's the most damaging wrong value this column could hold, and what — constraint, type, or code — stops it?
- How will we find out this data is wrong before a customer does?
- When this fails, what will we wish the failed attempt had left behind?

### Shape and Growth

- What does this grow with — customers, time, or activity? How many rows in three years, and which query gets slow first?
- Which questions will be asked of this table constantly, and which ones once a quarter? Are we designing for the right one?

---

## The Fifteen Patterns

| # | Pattern | Example |
|---|---|---|
| 1 | **Money is integer minor units + currency** | `amount BIGINT` + `currency` on every money row; `unit_amount_decimal` for sub-cent prices |
| 2 | **Ledgers, not mutable balances** | `ledger_entries` (`amount`, `fee`, `net`, `type`, `source_id`); `ending_balance` on each entry |
| 3 | **Corrections are new rows** | `refunds` is its own table; `payments.amount_refunded` is a rollup, the payment is never rewritten |
| 4 | **Separate intent from attempt** | `payments` vs `payment_attempts`; `invoices.attempt_count` |
| 5 | **Status plus a timestamp per transition** | `finalized_at`, `paid_at`, `voided_at`, `marked_uncollectible_at` |
| 6 | **Scheduled is not the same as happened** | `cancel_at` / `cancel_at_period_end` vs `canceled_at` / `ended_at` |
| 7 | **Immutable, versioned catalog** | `products` → `prices` (never edited; `active`, `lookup_key`) → `subscription_items` |
| 8 | **Snapshot at the moment of record** | `invoices.customer_name`, `customer_email`, `customer_address_*`; line items carry `period_start`/`period_end` |
| 9 | **Stored totals that reconcile to parts** | `subtotal`, `tax`, `total`, `amount_due`, `amount_paid`, `amount_remaining` + per-line tax amounts |
| 10 | **Typed, prefixed identifiers** | `pay_`, `ref_`, `ent_` — a single `source_id` column stays readable |
| 11 | **Explicit polymorphism** | `parent_type` + `parent_id`, `source_type` + `source_id` |
| 12 | **Tenant key on every row** | `account_id` on every table, including children like `price_tiers` |
| 13 | **Metadata is a sanctioned side channel** | a `*_metadata` key/value table beside each core object |
| 14 | **Never destroy business records** | `customers.deleted_at`, `prices.active = false` |
| 15 | **Evolve by addition; record change as events** | old column kept beside its replacement, `_v2` tables; `subscription_item_change_events.mrr_change` |

The rest of this document is detail.

---

## 1. Money

- **Integer minor units.** `amount BIGINT NOT NULL` holding cents (or yen, or the currency's smallest unit). Never `FLOAT`/`DOUBLE`. `DECIMAL` is acceptable for storage but invites fractional-cent drift in application math; integers make rounding an explicit, visible decision.
- **Currency on the same row.** Every table with an amount has a `currency CHAR(3)` next to it. An amount without a currency is a bug waiting for a second market.
- **An escape hatch for sub-unit precision.** Unit prices, rates, and usage pricing sometimes need fractions of a cent (`$0.0025` per API call). Store those as `DECIMAL` (or a decimal string) in a separately named column like `unit_amount_decimal`, and round to integer minor units the moment it becomes a charge.
- **Keep both sides of a conversion.** When a customer pays in one currency and you settle in another, store both: `presentment_amount`/`presentment_currency` and `amount`/`currency`, plus the `exchange_rate` used.
- **Fee and net are columns, not arithmetic in reports.** `amount`, `fee`, `net` on the ledger row. Whoever reads it later shouldn't have to know the fee schedule in effect that day.

See [`references/money-and-ledgers.md`](references/money-and-ledgers.md).

---

## 2. Ledgers and Corrections

A balance column that gets `UPDATE`d is a number nobody can explain. Model every change to a balance as an appended entry:

```sql
CREATE TABLE customer_balance_entries (
  id              VARCHAR(32)  PRIMARY KEY,
  account_id      VARCHAR(32)  NOT NULL,
  customer_id     VARCHAR(32)  NOT NULL,
  type            VARCHAR(40)  NOT NULL,
  amount          BIGINT       NOT NULL,
  currency        CHAR(3)      NOT NULL,
  ending_balance  BIGINT       NOT NULL,
  source_type     VARCHAR(40)  NOT NULL,
  source_id       VARCHAR(32)  NOT NULL,
  created_at      TIMESTAMP(6) NOT NULL
);
```

Rules:

- **Append-only.** No `UPDATE`, no `DELETE`. A mistake is fixed by a reversing entry.
- **Signed amounts, typed entries.** Credits positive, debits negative (or the reverse — pick once). `type` says *why* (`invoice_payment`, `credit_note`, `adjustment`, `refund`).
- **Every entry points at its cause.** `source_type` + `source_id` link the entry to the invoice, refund, or adjustment that produced it.
- **`ending_balance` makes history cheap.** The balance at any moment is one row lookup instead of a sum over all history. It's derived, so it's verifiable.
- **A cached balance on the parent is allowed** (`customers.balance`) as long as it is written in the same transaction as the entry and can be rebuilt from the ledger.

**Corrections are first-class records.** A refund is a `refunds` row with its own ID, status, reason, and ledger entry — not a negative charge and not an edit to the charge. The charge keeps a rollup (`amount_refunded`, `refunded`) for convenience.

---

## 3. Lifecycle and State

### Intent vs. attempt

When an outcome may take several tries, model the goal and each try separately:

| Goal (one row) | Attempt (many rows) |
|---|---|
| `payments` | `payment_attempts` |
| `payment_method_setups` | `setup_attempts` |
| `invoices` (`attempt_count`, `next_payment_attempt`) | `payment_attempts` against the invoice |

The goal holds the desired amount and its current status; each attempt records exactly what happened, including **failures with machine-readable codes** (`failure_code`, `failure_message`, `outcome_reason`). A failed attempt is data — keep it.

### Status plus per-transition timestamps

```sql
status                    VARCHAR(32) NOT NULL,
finalized_at              TIMESTAMP(6) NULL,
paid_at                   TIMESTAMP(6) NULL,
voided_at                 TIMESTAMP(6) NULL,
marked_uncollectible_at   TIMESTAMP(6) NULL
```

- `status` is the current state; the `*_at` columns record **when each meaningful transition happened**. You can answer "how long from finalized to paid?" without an audit table.
- Status values are lowercase snake_case strings, never integers. Add new values freely; never repurpose an old one.
- The application changes status only through named transitions (`finalize`, `pay`, `void`) that also stamp the timestamp. No code path writes `status` directly.
- For rich lifecycles, add a `<entity>_status_transitions` table (`from_status`, `to_status`, `actor`, `reason`, `created_at`).

### Scheduled vs. happened

Keep the plan and the fact in different columns: `cancel_at` (when it *will* end) and `cancel_at_period_end` (the intent) vs. `canceled_at` (when the request was made) and `ended_at` (when it actually stopped). Collapsing them makes "who's churning next month?" unanswerable.

See [`references/lifecycle-and-state.md`](references/lifecycle-and-state.md).

---

## 4. Catalog, Instances, and Snapshots

### Immutable catalog

Three layers: **definition** (`products`), **terms** (`prices`: amount, currency, interval, tiers), **instance** (`subscription_items`: which customer bought which price, in what quantity).

- **Prices are never edited.** Changing a price creates a new `prices` row; the old one is set `active = false`. Existing subscriptions keep pointing at the terms they agreed to.
- **`lookup_key`** gives the current version a stable name (`pro_monthly`) so code doesn't hard-code IDs.
- **Variable structure goes in child tables**: `price_tiers` (`up_to`, `unit_amount`, `flat_amount`), `price_currency_options`.

### Snapshot at the moment of record

A finalized document (invoice, receipt, order, contract) **copies** the values it was issued with:

- `invoices.customer_name`, `customer_email`, `customer_address_*`, `customer_tax_exempt`
- `invoice_line_items.amount`, `quantity`, `price_id`, `period_start`, `period_end`, `proration`

That isn't denormalization for speed — it's correctness. The invoice must read the same in five years after the customer moves and the price is retired. Rule of thumb: **anything printed on a document lives on the document.**

### Stored totals that reconcile

Store the totals (`subtotal`, `discount`, `tax`, `total`, `amount_paid`, `amount_remaining`) *and* the breakdown that produces them (`invoice_line_items`, `invoice_line_item_tax_amounts` with `taxable_amount` and `inclusive`, `invoice_line_item_discount_amounts`). Then enforce the invariant — in a test, a constraint, or a nightly check — that the parts sum to the whole.

---

## 5. Identity, References, and Tenancy

- **Prefixed, opaque string IDs** (`cus_…`, `inv_…`). See `stripe-api-design` §1 for the format. In storage: either make it the primary key, or keep a compact internal `BIGINT` PK and a unique `public_id` column — but never expose the internal one.
- **FK columns end in `_id`** and name the thing they point at: `customer_id`, `latest_invoice_id`, `default_payment_method_id`.
- **Polymorphic references are explicit.** `source_type` + `source_id`, `parent_type` + `parent_id`. Prefixed IDs make them readable, but the `_type` column makes them joinable and checkable.
- **Tenant key on every row** — including child, join, and metadata tables. `account_id` is on `price_tiers` even though `prices` already has it. Every query can filter by tenant without a join, every index leads with it, and a missed `WHERE account_id = ?` is caught by a linter rather than a customer.
- **Metadata is its own table** (`<entity>_metadata`: tenant, entity id, `key`, `value`) or a bounded JSON column. Business logic never reads it. It is how integrators link your rows to theirs without asking for new columns.

---

## 6. Deletion, Evolution, and Change Events

### Never destroy business records

- Customers get `deleted` / `deleted_at`; catalog entries get `active = false`; documents get a terminal status (`void`, `canceled`).
- Hard-delete is for genuinely disposable data (sessions, expired tokens) and for legal erasure, which should scrub PII in place and keep the financial skeleton.

### Evolve by addition

- **Never rename or retype a column in place.** Add the new column, backfill, dual-write, move readers, then stop writing the old one. Expect the old column to sit beside its replacement for years; that's the cost of never breaking a reader.
- **A new shape is a new table** (`_v2`), not a mutation of the old one.
- **Never repurpose** a column, enum value, or table for a different meaning.

See [`references/schema-evolution.md`](references/schema-evolution.md).

### Record change as events

Reporting from snapshots means diffing them; reporting from events means summing them. For anything you'll be asked to trend (revenue, seats, status), write a change-event row with the **delta** in the same transaction as the change:

```sql
CREATE TABLE subscription_item_change_events (
  id                    VARCHAR(32)  PRIMARY KEY,
  account_id            VARCHAR(32)  NOT NULL,
  subscription_item_id  VARCHAR(32)  NOT NULL,
  event_type            VARCHAR(40)  NOT NULL,
  mrr_change            BIGINT       NOT NULL,
  quantity_change       BIGINT       NOT NULL,
  currency              CHAR(3)      NOT NULL,
  event_at              TIMESTAMP(6) NOT NULL,
  local_event_at        DATETIME(6)  NOT NULL
);
```

Monthly MRR becomes `SUM(mrr_change)`.

---

## 7. Time

- **Typed timestamps in UTC** (`TIMESTAMP` / `TIMESTAMPTZ` / `DATETIME` stored as UTC) — not integer epoch columns. The database can then compare, index, truncate, and display them correctly. Serialize to epoch seconds at the API boundary if that's your wire format.
- **`created_at` on every table**, `updated_at` on every mutable one. Pick one suffix convention (`_at`) and use it everywhere.
- **Store the local time too when the business cares about it.** "Revenue on the merchant's March 3rd" needs the local date; keep `local_event_at` or the tenant's timezone at the time of the event, not just a UTC instant.
- **Periods are half-open**: `period_start` inclusive, `period_end` exclusive.

---

## 8. Data as the Asset (DatA)

The feature that writes a row is its first reader, not its last. Finance, analytics, support, partners, and models will read it for years after that feature is forgotten, and the value of the data compounds only if it was captured well the first time. Model for those readers too.

- **Record provenance on every write.** Who or what created the row (`created_by_type` + `created_by_id`: user, staff, system, integration), through which path (`source`: `dashboard`, `api`, `import`, `job`), and the `request_id` that ties it to logs. Cheap at write time, unrecoverable later.
- **Capture context the moment it exists.** Channel, attribution, platform, app version, experiment variant, the price or offer that was shown. If a decision depended on it, store it beside the decision. Once the request ends, it's gone.
- **Store observations, not just conclusions.** Keep the raw facts (each visit, each attempt, each response) rather than only a derived flag like `is_engaged`. Definitions change; facts can be reinterpreted under the new definition. Conclusions can't.
- **One definition per term.** Every status value and business term (`active`, `churned`, `member`) has exactly one written meaning. If two teams need different meanings, that's two columns, not one argued-over column.
- **The schema documents itself.** Column and table comments (`COMMENT` in MySQL, `COMMENT ON` in Postgres) for anything non-obvious: units, allowed values, what null means. The description ships in the same migration as the column.
- **Downstream readers are consumers with a contract.** Know which tables sync to the warehouse, BI, or reverse ETL. Breaking a synced table is a blocking review item, and deprecations are announced like API changes.
- **Quality is enforced, not hoped for.** Push invariants into constraints first (`NOT NULL`, `CHECK`, foreign keys, tenant-scoped `UNIQUE`). Cover the rest with scheduled checks: totals reconcile, no orphans, data is fresh, volumes look normal.
- **Classify sensitivity when the column is born.** Tag personal, financial, and secret columns as they're added. Classification is what lets the rest of the data be shared widely, and what lets an erasure request scrub precisely instead of nervously.

### Questions to Ask

- Who besides this feature will want this row, and what would they need that we aren't storing?
- If an analyst found this table with nobody to ask, would they read it correctly?
- What context exists only during this request and disappears when it ends?
- Are we storing what happened, or only what we concluded about it? Could we recompute the conclusion if the definition changed?
- What would it cost, in money or in trust, if this data were quietly wrong for a month?
- Could we hand this table to a partner or a model tomorrow? What would we have to strip out first, and do we know which columns those are?

---

## Anti-Patterns

| Anti-pattern | Why it's bad | Do this instead |
|---|---|---|
| `price FLOAT` | Loses pennies | `amount BIGINT` minor units + `currency` |
| Amount with no currency column | Breaks on the second currency | `currency CHAR(3)` on every money row |
| `UPDATE accounts SET balance = balance - 500` | Unexplainable balance | Append a ledger entry; derive the balance |
| Refund as a negative charge or an edit to the charge | Destroys the original fact | `refunds` table + rollup on the charge |
| `status` with no transition timestamps | Can't measure the lifecycle | `*_at` per meaningful transition |
| `canceled_at` used for both "requested" and "ended" | Can't see scheduled churn | `cancel_at` / `canceled_at` / `ended_at` |
| Editing a price in place | Rewrites what existing customers agreed to | New price row, old one `active = false` |
| Invoice that joins live to `customers` for its address | Old invoices change when profiles change | Snapshot onto the invoice |
| Totals with no stored breakdown (or breakdown with no totals) | Can't audit, or can't query fast | Both, with a reconciliation check |
| `ref_id` with no type | Unjoinable, unvalidatable | `source_type` + `source_id` |
| Tenant key only on top-level tables | Every child query needs a join to be safe | Tenant key on every row |
| Custom-field requests become new columns | Schema bloat, migrations for integrators | Metadata table |
| `DELETE FROM customers` | Orphans financial history | Soft delete / `active = false` |
| Renaming a column in a migration | Breaks every reader not deployed in lockstep | Add, backfill, dual-write, deprecate |
| Integer epoch `created` columns | DB can't reason about time | Typed UTC timestamps |
| Integer status codes (`status = 3`) | Unreadable, unextendable | Snake_case string enums |
| No record of who or what wrote a row | Can't explain, attribute, or debug it | `created_by_type`, `created_by_id`, `source`, `request_id` |
| Only a derived flag (`is_engaged`) stored | Dies with today's definition | Store the underlying facts; derive the flag |
| Undocumented columns and enum values | Every reader guesses differently | Column comments in the same migration |
| Breaking a table the warehouse syncs, unannounced | Silent wrong numbers downstream | Treat synced tables as contracts |

---

## The Decision Framework

Once the open questions above are settled, walk the mechanical pass for each table:

1. **Is this a fact or a state?** Facts (payments, entries, events) are append-only. State (a subscription) mutates — so what facts record each mutation?
2. **Does it hold money?** Integer minor units, currency, and — if it changes a balance — a ledger entry.
3. **What is its lifecycle?** List the statuses and the transitions. Which transitions get a `*_at`? Which are scheduled vs. happened?
4. **Can it fail and retry?** Then split intent from attempt, and keep the failures.
5. **Is it a definition or an instance?** Definitions are versioned and immutable; instances point at a version.
6. **Will it be printed, sent, or signed?** Snapshot everything it displays.
7. **Who owns it?** Tenant key on the row, leading every index.
8. **What will integrators want to attach?** Metadata.
9. **How does it end?** Terminal status or soft delete — not `DELETE`.
10. **What will we be asked to trend?** Write change events with deltas from day one.
11. **What's the next change?** Shape it so that change is additive.
12. **Who else will read it?** Provenance, write-time context, comments, sensitivity tags, and a named downstream owner — before it ships.

For a pre-merge review of a new table or migration, use [`references/checklist.md`](references/checklist.md).

---

## Reference Documents

- **[`references/money-and-ledgers.md`](references/money-and-ledgers.md)** — Minor units, rounding, multi-currency, ledger design, refunds and reversals
- **[`references/lifecycle-and-state.md`](references/lifecycle-and-state.md)** — Intent/attempt, status columns, transition tables, scheduled vs. actual
- **[`references/schema-evolution.md`](references/schema-evolution.md)** — Expand/contract migrations, deprecations, change events
- **[`references/checklist.md`](references/checklist.md)** — Table and migration review checklist

---

## Summary

1. **A row is a fact.** Record new facts instead of editing old ones.
2. **Money is integers with a currency**, and balances are sums of ledger entries.
3. **Corrections are records**, never edits.
4. **Separate what you want from each attempt to get it**, and keep the failures.
5. **Status plus a timestamp per transition**; plans and facts in different columns.
6. **Definitions are immutable and versioned**; documents snapshot what they show.
7. **Store totals and their parts**, and check that they agree.
8. **Every row knows its tenant and its type**; references say what they point at.
9. **Nothing important is deleted**, and nothing is renamed in place.
10. **Write change events with deltas** so the questions you'll be asked are sums.
11. **DatA — Data as the Asset.** Capture provenance and context at write time, store facts over conclusions, document every column, and treat downstream readers as customers.
