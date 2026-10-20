# Table and Migration Review Checklist

Use this before merging a new table or a migration. Unlike the questions in `SKILL.md`, these items have yes/no answers. If you can't tick a box, write down why.

---

## 1. Purpose

- [ ] I can say in one sentence what this table remembers, and whether it records **facts** (append-only) or **state** (mutable).
- [ ] I know which questions it must answer, and the indexes support the frequent ones.
- [ ] The table and column names match the language the business and customers use.
- [ ] Naming follows the schema-wide conventions (`_id` on every foreign key, `_at` on every timestamp), and a linter checks it rather than reviewers.

## 2. Identity and References

- [ ] Primary key is a prefixed opaque string, or an internal key with a unique `public_id`. The internal key is never exposed.
- [ ] The ID prefix doesn't collide with an existing one.
- [ ] Every foreign key column ends in `_id` and names its target.
- [ ] Every polymorphic reference has a `_type` column beside its `_id`.
- [ ] Foreign key constraints exist, or there's a written reason they don't.

## 3. Tenancy

- [ ] The tenant key is on this table, including if it's a child, join, or metadata table.
- [ ] Every composite index that serves a tenant query leads with the tenant key.
- [ ] Unique constraints are scoped by tenant where they should be (`UNIQUE (account_id, lookup_key)`).

## 4. Money

- [ ] Amounts are `BIGINT` minor units. No `FLOAT`/`DOUBLE`.
- [ ] Every amount has a `currency` on the same row.
- [ ] Sub-unit rates live in a separately named decimal column.
- [ ] Changes to any balance are written as ledger entries; any cached balance is updated in the same transaction.
- [ ] Corrections (refunds, reversals, credits) are their own records, not edits or negative rows.
- [ ] Stored totals have a stored breakdown, and something checks that they agree.

## 5. Lifecycle

- [ ] `status` is a snake_case string with constrained values.
- [ ] Every transition someone will want to measure has a `*_at` column.
- [ ] Scheduled values and actual values are separate columns (`cancel_at` vs `ended_at`).
- [ ] If outcomes can be retried, intent and attempt are separate tables, and attempts keep failure codes.
- [ ] Code changes `status` only through named transitions.

## 6. Catalog and Snapshots

- [ ] Definitions (prices, plans, templates) are immutable and versioned; they're retired with `active = false`, not edited.
- [ ] Documents (invoices, receipts, orders, contracts) copy every value they display.

## 7. Time

- [ ] Timestamps are typed and stored in UTC; no integer epoch columns.
- [ ] `created_at` exists; `updated_at` exists if rows can change.
- [ ] Where the business cares about local dates, the local value or the tenant's timezone at the time is stored.
- [ ] Periods are half-open (`start` inclusive, `end` exclusive), and that's documented.

## 8. Deletion and Privacy

- [ ] Business records are never hard-deleted; there's a soft-delete column or a terminal status.
- [ ] Personal data columns are identified, and there's a plan to scrub them while keeping the financial record.

## 9. Extensibility and Change

- [ ] There's a metadata side table or bounded JSON column for caller-defined data, and no business logic reads it.
- [ ] Anything we'll be asked to trend writes a change event with a delta in the same transaction.

## 10. Data as the Asset (DatA)

- [ ] Rows record who or what wrote them (`created_by_type`, `created_by_id`, `source`) and a `request_id`.
- [ ] Context that only exists at write time (channel, attribution, platform, app version, variant, offer shown) is stored where a decision depended on it.
- [ ] Underlying facts are stored, not only derived flags.
- [ ] Every non-obvious column and every enum value has a comment, added in this migration.
- [ ] Each business term used here has one written definition.
- [ ] Personal, financial, and secret columns are tagged.
- [ ] Invariants are enforced as constraints where the database can, and by scheduled checks where it can't.
- [ ] If this table syncs downstream, its owner there knows about this change.

## 11. The Migration Itself

- [ ] It's purely additive, or it's one step of an expand/contract sequence that's written down.
- [ ] No column, enum value, or table is repurposed to mean something new.
- [ ] It's safe if code and schema deploy in either order.
- [ ] Backfills run in batches and can resume.
- [ ] I've listed readers outside this codebase (analytics, warehouse, exports, other services) and checked them.
