# Money and Ledgers

Money is the one domain where "close enough" is a defect. These rules keep every cent explainable.

---

## Representation

### Integer minor units

```sql
amount    BIGINT  NOT NULL,
currency  CHAR(3) NOT NULL
```

- `1099` with `usd` means $10.99. `1099` with `jpy` means ¥1,099, because yen has no minor unit. Look up each currency's exponent (ISO 4217) instead of assuming two decimals.
- `BIGINT`, not `INT`. A platform summing a year of volume in cents overflows 32 bits quickly.
- `FLOAT` and `DOUBLE` are never acceptable for money. `0.1 + 0.2` is not `0.3`.

### When you need fractions of a minor unit

Usage prices, per-unit rates, FX rates, and tax rates legitimately need more precision than a cent. Store them separately and visibly:

```sql
unit_amount          BIGINT        NULL,
unit_amount_decimal  DECIMAL(19,12) NULL
```

The same split applies to tiers (`price_tiers.amount` + `amount_decimal`). The precise value is the *rate*; the integer is what gets *charged*. Round once, at the moment a rate becomes an amount, with a documented rounding mode, and store the rounded result.

### Multi-currency

Keep both sides of every conversion on the row that performed it:

| Column | Meaning |
|---|---|
| `presentment_amount`, `presentment_currency` | What the customer saw and paid |
| `amount`, `currency` | What you settled in |
| `exchange_rate` | The rate actually applied |

Never recompute historical conversions with today's rate.

---

## Ledgers

### Why

A mutable balance answers one question: "what is it now?" A ledger answers every question: "what was it then, what changed it, and who caused that?" The ledger is the source of truth; any balance column is a cache of it.

### Shape

```sql
CREATE TABLE ledger_entries (
  id              VARCHAR(32)  PRIMARY KEY,
  account_id      VARCHAR(32)  NOT NULL,
  balance_type    VARCHAR(32)  NOT NULL,
  type            VARCHAR(40)  NOT NULL,
  reporting_category VARCHAR(40) NOT NULL,
  amount          BIGINT       NOT NULL,
  fee             BIGINT       NOT NULL DEFAULT 0,
  net             BIGINT       NOT NULL,
  currency        CHAR(3)      NOT NULL,
  status          VARCHAR(20)  NOT NULL,
  available_on    TIMESTAMP(6) NOT NULL,
  source_type     VARCHAR(40)  NOT NULL,
  source_id       VARCHAR(32)  NOT NULL,
  created_at      TIMESTAMP(6) NOT NULL
);
```

What each column earns:

- **`type`** is operational (`payment`, `refund`, `payout`, `platform_fee`, `processor_fee`, `adjustment`); **`reporting_category`** is how finance groups it. They change for different reasons, so keep both.
- **`amount`, `fee`, `net`** are stored, not derived in reports. `net = amount - fee` is an invariant you can check.
- **`status` + `available_on`** separate "recorded" from "spendable". Money that is pending is still a fact.
- **`balance_type`** lets one table hold several balances (available, pending, reserved, customer credit) without separate tables per balance.

### Invariants worth enforcing

- Entries are never updated or deleted. Grant the application role `INSERT` and `SELECT` only on the table, if your database allows it.
- `net = amount - fee` on every row.
- If you cache a balance (`accounts.balance`, `customers.balance`), it is updated in the **same transaction** as the entry, and a periodic job compares it to `SUM(net)`.
- For double-entry systems, every transaction's entries sum to zero across accounts.

### Running balances

Storing `ending_balance` on each entry makes point-in-time lookups trivial and makes gaps or races visible: each row's `ending_balance` minus its `amount` must equal the previous row's `ending_balance`. It requires serializing writes per balance (a row lock on the balance owner), which is usually the right call anyway.

---

## Corrections

### Refunds, reversals, adjustments

The original fact stays as it was. The correction is a new record with:

- its own ID and lifecycle (`refunds.status`: `pending`, `succeeded`, `failed`, `canceled`)
- a reason (`reason`, `failure_reason`)
- a pointer to what it corrects (`payment_id`)
- its own ledger entry (`ledger_entry_id`), and a second one if the correction itself fails (`failure_ledger_entry_id`)

The original may carry a **rollup** for convenience — `payments.amount_refunded`, `payments.refunded` — updated alongside the new record.

### Why not just negate?

A negative payment loses the reason, the separate lifecycle (refunds can fail days later), partial refunds, and the ability to list "all refunds last month" without filtering signs out of a payments table.

---

## Documents and Totals

An invoice stores its totals **and** the parts that produce them:

| Level | Columns |
|---|---|
| Invoice | `subtotal`, `tax`, `total`, `amount_due`, `amount_paid`, `amount_remaining`, `starting_balance`, `ending_balance` |
| Line item | `amount`, `quantity`, `price_id`, `period_start`, `period_end`, `proration`, `discountable` |
| Line-item tax | `amount`, `taxable_amount`, `tax_rate`, `inclusive` |
| Line-item discount | `amount`, `discount_id` |

Totals make reporting fast; the breakdown makes totals auditable. A test or scheduled check confirms `subtotal = SUM(line_items.amount)` and `amount_remaining = amount_due - amount_paid`.

---

## Questions to Ask

- If finance asks for every movement of money on one account last quarter, is that one query?
- Which number on this screen could be wrong without anyone noticing, and what would catch it?
- Where does rounding happen, how many times, and who decided the rounding mode?
- If a refund fails three days later, where is that recorded and what does the balance show in between?
- Could we rebuild every cached balance from the ledger today? Have we actually tried?
