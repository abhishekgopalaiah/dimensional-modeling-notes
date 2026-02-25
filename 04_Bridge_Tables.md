# Bridge tables — compact, practical guide

A **bridge table** (aka associative table) resolves **many-to-many** relationships while preserving fact grain and summarizability. Use them whenever joining a fact to a dimension directly would duplicate fact rows or break aggregations.

Below is everything you need to know as a senior engineer: definition, structure, SQL patterns, allocation logic, SCD interactions, performance, testing, and common pitfalls.

---

# 1. Purpose (short)

Keep the original fact grain intact while modeling relationships that are many-to-many (fact↔dim or dim↔dim). Optionally carry allocation or weighting so measures can be fairly attributed.

---

# 2. Typical bridge structures & logical keys

### A. Fact ↔ Dimension bridge (promotion example)

When one `order_line` can have multiple `promotion`s:

Columns:

* `order_line_id` (FK → fact)
* `promotion_key` (FK → dim_promotion)
* `allocation_pct` (optional, numeric, sums to 1 per order_line)
* optional audit/validity columns (`effective_from`, `effective_to`, `load_ts`)

Logical PK: `(order_line_id, promotion_key)`

### B. Dimension ↔ Dimension bridge (customer ↔ segment)

When a customer belongs to multiple segments:

Columns:

* `customer_key`
* `segment_key`
* (optional) `weight` or `start_date`, `end_date`

Logical PK: `(customer_key, segment_key)`

### Surrogate key?

Optional. Use a composite natural PK normally; add a surrogate `bridge_id` only for operational reasons (CDC, soft deletes, tooling that expects single-column PK).

---

# 3. Allocation & preservation of totals

If bridge rows will be used to attribute numeric measures (revenue, quantity), include an `allocation_pct` so joining does not multiply facts.

Example:

Fact `order_line`:

* `order_line_id = 1`, `revenue = 100`

Bridge:

* (1, P1, 0.6)
* (1, P2, 0.4)

Query to allocate revenue by promotion:

```sql
SELECT b.promotion_key,
       SUM(f.revenue * b.allocation_pct) AS revenue_allocated
FROM fact_order_line f
JOIN bridge_order_promotion b ON f.order_line_id = b.order_line_id
GROUP BY b.promotion_key;
```

This preserves total: 100 * (0.6 + 0.4) = 100.

If no allocation column exists, joining will duplicate the revenue across promotions (wrong).

---

# 4. SQL examples: create, join, aggregate

Create (simple):

```sql
CREATE TABLE bridge_order_promotion (
  order_line_id   BIGINT NOT NULL,
  promotion_key   INT   NOT NULL,
  allocation_pct  NUMERIC(8,6) DEFAULT 1.0,
  PRIMARY KEY (order_line_id, promotion_key)
);
```

Join + safe aggregation:

```sql
SELECT p.promotion_name,
       SUM(f.revenue * b.allocation_pct) as revenue_for_promo
FROM fact_order_line f
JOIN bridge_order_promotion b ON f.order_line_id = b.order_line_id
JOIN dim_promotion p ON b.promotion_key = p.promotion_key
GROUP BY p.promotion_name;
```

Detect missing allocation sums (data quality):

```sql
SELECT order_line_id,
       SUM(allocation_pct) AS total_alloc
FROM bridge_order_promotion
GROUP BY order_line_id
HAVING ABS(SUM(allocation_pct) - 1.0) > 0.0001;
```

Detect duplicate bridge rows:

```sql
SELECT order_line_id, promotion_key, COUNT(*) AS cnt
FROM bridge_order_promotion
GROUP BY order_line_id, promotion_key
HAVING COUNT(*) > 1;
```

---

# 5. Interaction with SCDs and time

* If the dimension is SCD2, the bridge should reference the appropriate `dim_key` (surrogate) for the effective time of the fact event.
* If the relationship itself is time–varying (customer moves between segments), include `effective_from` / `effective_to` in the bridge and join using the fact event date:

```sql
JOIN bridge b
  ON f.customer_key = b.customer_key
 AND f.event_date BETWEEN b.effective_from AND b.effective_to
```

* For historical correctness, choose whether the bridge records historical versions (recommended) or only current mappings (dangerous for historical analysis).

---

# 6. When to materialize vs compute on the fly

Materialize bridge tables when:

* Relationship is large and reused widely
* Joins are frequent and performance critical
* Aggregations require consistent allocation

Compute on the fly when:

* Relationship is small and seldom queried
* Up-to-date dynamic logic is required and materialization cost outweighs benefits

Materialized bridges are common in warehouses/marts for performance and repeatability.

---

# 7. Performance & storage considerations

* Bridges increase join complexity; index appropriately (cluster/partition) by the FK most used in joins (e.g., `order_line_id`).
* If `order_line_id` cardinality is huge, watch file sizes and partitioning strategies in Spark/BigQuery.
* Pre-aggregate if downstream queries always need allocation results by certain dimensions (e.g., promo × month).

---

# 8. Common pitfalls and how to avoid them

* **No allocation column**: multiplying measures on join — always consider allocation or design queries to divide.
* **Not enforcing uniqueness**: duplicate bridge rows break sums — validate uniqueness.
* **Wrong SCD key used**: linking to current dimension surrogate will corrupt historical analysis — join on correct effective surrogate or include effective dates.
* **Using bridge for non-M2M**: unnecessary complexity if relation is 1-to-many — don’t overuse.
* **Bridge row count explosion**: storing full cross-product without pruning or weighting can bloat tables.

---

# 9. Testing and monitoring (must-have checks)

* Uniqueness check for PK (duplicates).
* Allocation completeness check (allocation_pct sums to 1 per source row).
* Referential integrity checks (every bridge FK exists in fact/dim).
* Cardinality monitoring (growth trends).
* Spot check totals: compare SUM(fact.revenue) before and after join/allocations to ensure conservation.

---

# 10. Interview-ready concise answer

> A bridge table is an associative table that resolves many-to-many relationships (between a fact and a dimension or two dimensions) while preserving the original fact grain. It should have a logical primary key (usually the composite of the foreign keys), optionally a surrogate key for operational reasons, and often an allocation/weight column so measures can be attributed accurately without duplication.

---

If you want, I can:

* Draft a concrete `fact + dim + bridge` schema for an ecommerce or trading example with DDL and sample aggregation tests, or
* Build a checklist of SQL assertions you should run in CI to catch bridge-related data quality issues. Which would you prefer?
