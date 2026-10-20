# Schema Evolution

Rows outlive code. The schema has to change without breaking the readers you've forgotten about: other services, analytics, warehouse syncs, partners reading exports.

---

## The Rule

**Additive changes ship any time. Everything else is an expand/contract sequence.**

| Safe (additive) | Unsafe (needs expand/contract) |
|---|---|
| New table | Rename a table or column |
| New nullable column, or one with a default | Change a column's type or meaning |
| New enum / status value | Remove or repurpose an enum value |
| New index | Drop a column or table |
| Widen a column (`VARCHAR(32)` → `VARCHAR(64)`) | Narrow a column or add `NOT NULL` to existing data |

---

## Expand / Contract

To rename `plan_id` to `price_id`:

1. **Expand.** Add `price_id`. Deploy.
2. **Dual-write.** Application writes both columns. Deploy.
3. **Backfill.** Copy `plan_id` → `price_id` for old rows, in batches.
4. **Move readers.** Every reader switches to `price_id`. Include analytics, exports, and other services.
5. **Stop writing the old column.** Deploy.
6. **Contract** — only if you're sure nothing reads it. Many mature platforms never do this step for data that outside readers depend on, and carry the deprecated column beside its replacement indefinitely.

Each step is its own deploy and reversible on its own.

---

## New Shape, New Table

When a concept is redesigned rather than tweaked, create a sibling (`terminal_readers_v2`, `item_change_events_v2`) instead of migrating the old table into a different meaning. Run both, move consumers, and retire the old one on a schedule. Readers can tell from the name which semantics they're getting.

---

## Never Repurpose

The most dangerous migration has no DDL at all: deciding that an existing column, flag, or status now means something different. Every historical row silently changes meaning. If the meaning changes, the name changes — which means a new column.

---

## Record Change as Events

Snapshots tell you what is. Events tell you what happened. Reporting needs the second.

```sql
CREATE TABLE subscription_item_change_events (
  id                    VARCHAR(32)  PRIMARY KEY,
  account_id            VARCHAR(32)  NOT NULL,
  customer_id           VARCHAR(32)  NOT NULL,
  subscription_id       VARCHAR(32)  NOT NULL,
  subscription_item_id  VARCHAR(32)  NOT NULL,
  product_id            VARCHAR(32)  NOT NULL,
  price_id              VARCHAR(32)  NOT NULL,
  event_type            VARCHAR(40)  NOT NULL,
  mrr_change            BIGINT       NOT NULL,
  quantity_change       BIGINT       NOT NULL,
  currency              CHAR(3)      NOT NULL,
  event_at              TIMESTAMP(6) NOT NULL,
  local_event_at        DATETIME(6)  NOT NULL
);
```

- **Store the delta**, not just the new value. `SUM(mrr_change)` over a window is MRR movement; no window functions over snapshots required.
- **Write it in the same transaction** as the change it describes, so it can't drift.
- **Denormalize the dimensions** you'll group by (`customer_id`, `product_id`, `price_id`) onto the event, as of the moment it happened.
- **Doubles as an outbox.** A relay can publish these rows to a queue or webhook system, giving you reliable at-least-once delivery without dual-writes to a broker.

---

## Questions to Ask

- Who reads this table that isn't in this repo?
- If this migration is deployed but the code isn't (or the reverse), what breaks?
- Is this change really a rename, or has the meaning changed too?
- What will someone in two years need to know about rows written before today's change, and where will they find it?
- What are we going to be asked to put on a chart, and are we writing the events that make it a sum?
