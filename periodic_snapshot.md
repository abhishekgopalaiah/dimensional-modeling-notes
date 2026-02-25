# Periodic snapshot fact table — deep dive (with examples)

Short answer up front:
A **periodic snapshot fact** captures the *state of an entity at regular intervals* (daily, weekly, monthly). Measures in a periodic snapshot are **usually semi-additive** — you can sum them across many dimensions but **not safely across the time dimension** without using an appropriate aggregation (LAST_VALUE, AVG, etc.).

Now the full breakdown.

---

# 1. Definition & purpose

A **periodic snapshot fact table** stores one row per entity per regular time period (the period defines the grain). It is used to capture point-in-time state so you can analyze time-series trends, balances, inventory, portfolio positions, daily exposures, etc.

Typical grain sentence examples:

* “One row per account per day.”
* “One row per product per store per day.”
* “One row per portfolio per end-of-week.”

---

# 2. Key characteristics

* **Grain:** entity × period (e.g., account_id × date).
* **Row meaning:** state *as of* the snapshot time (end of day, start of day, end of month).
* **Measures:** mostly state-like (account_balance, stock_quantity, open_positions).
* **Additivity:** usually **semi-additive** — summable across non-time dimensions, **not** summable across time.
* **Load pattern:** typically periodic full/partial refresh of each period (partition overwrite for the date), or derived from transaction facts each period.
* **Primary key (logical):** (entity_id, snapshot_date).
* **Late data:** must allow writes/overwrites to historical partitions (backfills).
* **Use cases:** trend analysis, regulatory snapshots, daily positions, reconciliation, KPI time series.

---

# 3. Why it’s usually semi-additive

A semi-additive measure represents a *state* at a point in time, not a flow. Summing balances across dates produces nonsense:

Example:

```
account_id A: balance on Jan 1 = 1,000
                balance on Jan 2 = 1,200
SUM over Jan1–2 = 2,200  ← meaningless
```

Correct time aggregations:

* use LAST_VALUE(balance) for “balance as of end of period”
* use AVG(balance) for average balance over the window
* use MAX/MIN depending on business need

But you can sum across other dimensions (accounts, products, stores) at a fixed date:

```
total_balance_on_Jan_2 = SUM(balance) WHERE snapshot_date = '2026-01-02'
```

---

# 4. Concrete examples

### Example A — Bank account daily snapshot (classic)

**Grain:** one row per account per day.
Columns:

```
snapshot_date DATE,
account_id    STRING,
balance       NUMERIC,
available_cash NUMERIC,
num_transactions INT,
...
PRIMARY KEY (account_id, snapshot_date)
```

Queries:

* Total bank balance on a day:

  ```sql
  SELECT SUM(balance) FROM fact_account_snapshot WHERE snapshot_date='2026-02-01';
  ```
* Average balance last 30 days (per account):

  ```sql
  SELECT account_id, AVG(balance) FROM fact_account_snapshot
  WHERE snapshot_date BETWEEN DATE_SUB('2026-02-01', INTERVAL 30 DAY) AND '2026-02-01'
  GROUP BY account_id;
  ```
* Balance as of a date (per account): LAST_VALUE via window or pick row with max(snapshot_date) ≤ target_date.

### Example B — Inventory snapshot daily

**Grain:** one row per product per store per day.
Columns: `snapshot_date, store_id, product_id, stock_qty, reserved_qty`.
Use for stock reports and restock triggers. `stock_qty` is semi-additive.

### Example C — Portfolio snapshot (end-of-day)

**Grain:** one row per portfolio per day.
Measures: `market_value`, `unrealized_pnl`, `open_positions_count`.

---

# 5. How to build them (ETL patterns)

Two main approaches:

**A. Recompute snapshot from transaction/atomic fact per period (recommended)**

* For each snapshot_date, aggregate the transaction fact (up to that business time) to derive the state.
* Use partition overwrite for that snapshot_date.

**B. Maintain via incremental updates**

* Keep running state and apply deltas each day (careful: must be idempotent and handle late transactions).

Typical ETL steps (recompute approach):

1. Select transactions up to the end of the period (or transactions for the period depending on domain).
2. Aggregate into one row per entity for that period.
3. Overwrite the snapshot partition for that date (or MERGE).

Important: use business date/time for cutoffs (event_time), not ingestion time.

---

# 6. Partitioning & storage

* Partition by `snapshot_date` (daily) — this supports efficient pruning and easy backfills.
* Combine with clustering by high-filter columns (account_id, product_id) if queries filter on them heavily.
* Keep file sizes balanced to avoid small file problems (Spark) or too many partitions (BigQuery limit considerations).

---

# 7. Late data & backfills

* Your pipeline must allow writing historical partitions (e.g., Jan 5) when late transactions arrive.
* Typical operational pattern: daily job overwrites last N days partitions (rolling reprocessing window) to capture late facts; keep archival raw transactions to recompute older snapshots when needed.

---

# 8. Aggregation semantics & BI guidance

* Do **not** SUM balances across time in dashboards. Provide BI users with safe aggregated metrics:

  * `balance_on_date` = SUM(balance) WHERE snapshot_date = X
  * `avg_balance_30d` = AVG(balance) over last 30 days
  * `mtd_balance_change` = balance_on_last_day - balance_on_first_day
* Provide documented helper views/functions for LAST_VALUE or point-in-time retrievals so downstream consumers don’t misuse the snapshot.

---

# 9. How it interacts with SCD and dimensions

* Snapshots reference surrogate keys or business keys for dimensions valid at snapshot_date.
* If a dimension changed retroactively, you may need to reprocess affected snapshot partitions to maintain historical correctness.

---

# 10. Pros / Cons — quick checklist

Pros:

* Easy historical trend analysis
* Good for KPI time series and compliance
* Predictable size growth (period × entities)

Cons:

* Storage heavy for high cardinality entities × frequency
* Semi-additive measures require care in aggregation
* Requires backfill support for late transactions

---

# 11. Interview-ready concise answer

> A periodic snapshot fact table captures the state of an entity at regular time intervals (e.g., account × day). Its measures are usually semi-additive — you can aggregate them across dimensions but not across time (use LAST_VALUE/AVG instead). Build snapshots by recomputing from atomic transactions per period, partition by snapshot_date, and support backfills for late data.

---

# 12. Example SQL (build a daily account snapshot from transactions)

```sql
-- transaction_fact: (txn_id, account_id, txn_time, txn_amount, txn_type)
-- We want account balance as of end-of-day (EOD) for each date.

WITH daily_cumulative AS (
  SELECT
    DATE(txn_time) AS txn_date,
    account_id,
    SUM(CASE WHEN txn_type='credit' THEN txn_amount ELSE -txn_amount END) AS net_flow
  FROM transaction_fact
  WHERE txn_time <= TIMESTAMP('2026-02-01 23:59:59')  -- for a single date job; typically parametrized
  GROUP BY DATE(txn_time), account_id
),
account_balance AS (
  -- cumulative sum up to the snapshot_date per account (if transactions only for period are present,
  -- you may need to aggregate over all prior txns or maintain running balance)
  SELECT
    account_id,
    SUM(net_flow) OVER (PARTITION BY account_id ORDER BY txn_date ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS balance,
    txn_date AS snapshot_date
  FROM daily_cumulative
)
-- Then write account_balance rows into fact_account_snapshot partitioned by snapshot_date
```

(Production pipelines usually compute cumulative state more efficiently, possibly maintaining running aggregates to avoid full historical scans.)

---

Good question — and this is where many people get confused.

Short answer:

> A transaction fact table absolutely has a date/time column, but it is not “period-based” in the way a periodic snapshot is.

Let’s clarify precisely.

---
# so Transaction Fact Table does not have period?
> A transaction fact table absolutely has a date/time column, but it is not “period-based” in the way a periodic snapshot is.

# 1️⃣ Transaction Fact Table

Grain:

> One row per business event.

Examples:

* One row per order line
* One row per trade
* One row per payment
* One row per click

Each row represents an event that happened at a specific moment.

So yes — it has:

* order_date
* trade_timestamp
* payment_time

But that date belongs to the event itself.

---

# 2️⃣ Periodic Snapshot Fact Table

Grain:

> One row per entity per time period.

Examples:

* One row per account per day
* One row per product per store per day

Here the “period” defines the grain.

Even if nothing happened that day,
a row may still exist to represent state.

---

# 3️⃣ The Core Difference

### Transaction Fact

Time is when something happened.

Rows exist only when events occur.

If no sale happens on Jan 5 → no row.

---

### Periodic Snapshot

Time is the reporting period.

Rows exist every period for each entity.

Even if no transaction occurred.

---

# 4️⃣ Example to Make It Clear

## Transaction Fact (Sales)

| order_id | order_date | product | revenue |
| -------- | ---------- | ------- | ------- |
| O1       | Jan 1      | P1      | 100     |
| O2       | Jan 5      | P2      | 200     |

If no order on Jan 3 → no row.

---

## Periodic Snapshot (Inventory)

| snapshot_date | product | stock_qty |
| ------------- | ------- | --------- |
| Jan 1         | P1      | 50        |
| Jan 2         | P1      | 48        |
| Jan 3         | P1      | 48        |
| Jan 4         | P1      | 60        |

Even if no sale happened on Jan 3,
stock level is recorded.

---

# 5️⃣ Additivity Difference

Transaction facts:
Usually fully additive across time.
You can sum revenue across days.

Snapshot facts:
Usually semi-additive.
You cannot sum stock_qty across days.

---

# 6️⃣ Important Insight

Transaction fact has event time.

Periodic snapshot has reporting time.

That’s the real conceptual difference.

---

# 7️⃣ Why This Matters for Modeling

If business asks:

“Total sales in Jan”

Use transaction fact.

If business asks:

“Inventory level on Jan 31”

Use snapshot fact.

If business asks:

“Average daily inventory in Jan”

Use snapshot fact + AVG over time.

---

# 8️⃣ Interview-Ready Answer

If asked:

> A transaction fact table includes an event date or timestamp, but its grain is defined by individual business events, not by fixed time periods. In contrast, a periodic snapshot fact table has time as part of its grain and records state at regular intervals regardless of whether events occurred.

That shows clarity.

---

# 9️⃣ Quick Check for You

If you wanted to calculate:

“Total account balance in January”

Would you:
A) Sum daily balances
B) Take balance on Jan 31
C) Average daily balances
D) It depends on business definition

Think carefully — this tests understanding of semi-additive behavior.
---

# cant we do periodic snapshot of transaction table? like revenue per day?
> Yes, you can create a daily summary of a transaction fact (e.g., revenue per day), but that is an aggregated transaction fact — not a true periodic snapshot in the state sense.

Very good question. This is where many people mix up concepts.

Short answer:

> Yes, you can create a daily summary of a transaction fact (e.g., revenue per day), but that is an aggregated transaction fact — not a true periodic snapshot in the state sense.

Let’s break this carefully.

---

# 1️⃣ Two Different Ideas That Look Similar

### A) Daily Revenue per Product

From transaction fact:

```sql
SELECT order_date, product_id, SUM(revenue)
FROM fact_order_line
GROUP BY order_date, product_id
```

This creates:

| order_date | product_id | daily_revenue |

This is:

* Aggregated transaction fact
* Still additive across time
* Derived from atomic events

It is NOT a state snapshot.

---

### B) Daily Account Balance

Snapshot table:

| snapshot_date | account_id | balance |

This captures:

* State at end of day
* Semi-additive measure
* Cannot sum across days

This is a true periodic snapshot.

---

# 2️⃣ Why Daily Revenue Is Not a “True” Snapshot

Revenue is a flow.

It accumulates through events.

If you store:

| date | revenue |

You can safely sum across dates:

Jan 1 → 100
Jan 2 → 200
Total Jan → 300

This is additive.

Snapshot tables usually represent state, not flow.

---

# 3️⃣ So What Is “Revenue Per Day”?

It is:

> A summarized transaction fact at day grain.

Grain:

One row per product per day.

But it is still fundamentally event-based data that was aggregated.

---

# 4️⃣ True Conceptual Difference

| Type                   | Represents        | Example         |
| ---------------------- | ----------------- | --------------- |
| Transaction fact       | Events            | Each order      |
| Aggregated transaction | Summarized events | Revenue per day |
| Periodic snapshot      | State at time T   | Balance per day |

Aggregation does not automatically make something a snapshot.

---

# 5️⃣ When You Would Build Daily Revenue Table

Reasons:

* Performance optimization
* Dashboard speed
* Reduce scan cost
* Pre-aggregate heavy fact

But it is derived from atomic transaction fact.

Atomic fact remains source of truth.

---

# 6️⃣ Can a Periodic Snapshot Contain Additive Measures?

Yes — but the conceptual purpose matters.

Example:

You might store:

| snapshot_date | total_active_users |

This is count at end of day.
It is state.
Summing across days is meaningless.

Even though number looks additive,
its meaning is state-based.

---

# 7️⃣ Senior-Level Insight

The classification is about semantic meaning, not just grain.

Ask:

Is this measuring something that happened?
→ Transaction fact.

Is this measuring what exists at a point in time?
→ Periodic snapshot.

Revenue per day measures what happened during that day.
Balance per day measures what exists at that time.

That’s the difference.

---

# 8️⃣ Interview-Ready Answer

If asked:

> A daily revenue table derived from transactions is an aggregated transaction fact, not a true periodic snapshot. Transaction facts represent flows, which are fully additive across time, whereas periodic snapshots represent state at a specific point in time and are typically semi-additive.

That’s a very strong answer.

---
