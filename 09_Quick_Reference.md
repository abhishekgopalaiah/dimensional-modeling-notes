# Data Warehouse Quick Reference Guide

> **Purpose**: Last-minute interview prep cheat sheet  
> **Time to Review**: 15-20 minutes  
> **Focus**: Core concepts, definitions, and common patterns

---

## 🎯 Must-Know Definitions (Memorize These)

### Grain
> "The exact level of detail represented by one row in a fact table, stated as 'one row per X'"

### Conformed Dimension
> "A dimension shared across multiple fact tables with identical keys, attributes, and business meaning"

### Surrogate Key
> "A system-generated integer primary key that insulates the warehouse from source system changes and enables SCD Type 2"

### Degenerate Dimension
> "A dimension attribute stored directly in the fact table without a separate dimension table (e.g., invoice_number)"

### Semi-Additive Fact
> "A measure that can be summed across entities but not across time (e.g., account_balance)"

### SCD Type 2
> "Preserves full history by inserting new rows with new surrogate keys when tracked attributes change"

---

## 📊 Fact Table Types (3 Core Types)

| Type | Grain | Load Pattern | Use Case | Example |
|------|-------|--------------|----------|---------|
| **Transaction** | One row per event | Append-only | Event capture | Sales, trades, clicks |
| **Periodic Snapshot** | One row per entity per period | Overwrite | State tracking | Daily balance, inventory |
| **Accumulating Snapshot** | One row per process instance | Update (MERGE) | Lifecycle tracking | Order fulfillment, loan application |

**Key Point**: Type is determined by **temporal behavior**, not aggregation level.

---

## 🔑 Dimension Types (Top 10)

| Type | Description | When to Use |
|------|-------------|-------------|
| **Conformed** | Shared across facts | Always for common dimensions |
| **Role-Playing** | Same dimension, multiple roles | Multiple dates in one fact |
| **SCD Type 1** | Overwrite, no history | Corrections, non-analytical attributes |
| **SCD Type 2** | Full history, new rows | Regulatory, historical analysis |
| **SCD Type 3** | Limited history columns | Previous value tracking |
| **Junk** | Combined low-cardinality flags | Multiple boolean flags |
| **Degenerate** | Stored in fact table | Transaction IDs with no attributes |
| **Mini** | Rapidly changing attributes | Prevent SCD2 explosion |
| **Bridge** | Many-to-many resolution | Multiple values per fact |
| **Factless** | No measures, only keys | Event occurrence, coverage |

---

## 🎨 Schema Patterns

### Star Schema (Preferred)
```
        dim_customer
             |
fact_sales --|-- dim_product
             |
        dim_date
```
- Denormalized dimensions
- Fewer joins
- Better performance
- Easier for BI tools

### Snowflake Schema (Rare)
```
fact_sales -- dim_product -- dim_category -- dim_department
```
- Normalized dimensions
- More joins
- Harder to query
- Minimal storage savings in columnar DBs

### Constellation Schema (Enterprise)
```
Multiple fact tables sharing conformed dimensions
```
- Different grains per fact
- Shared dimensions
- Cross-process analysis

---

## ⚡ Additivity Rules

| Fact Type | Sum Across Entities? | Sum Across Time? | Correct Time Aggregation |
|-----------|---------------------|------------------|------------------------|
| **Additive** | ✅ Yes | ✅ Yes | SUM, AVG, COUNT |
| **Semi-Additive** | ✅ Yes | ❌ No | LAST_VALUE, AVG, MAX, MIN |
| **Non-Additive** | ❌ No | ❌ No | Recalculate from base facts |

**Examples:**
- Additive: sales_amount, quantity, count
- Semi-Additive: balance, inventory, open_positions
- Non-Additive: margin%, conversion_rate, ratios

**Golden Rule**: Store base additive facts, compute ratios in BI layer

---

## 🔄 SCD Type 2 Pattern

### Dimension Structure
```sql
| surrogate_key | business_key | attribute | effective_from | effective_to | is_current |
|---------------|--------------|-----------|----------------|--------------|------------|
| 101           | C1           | Mumbai    | 2023-01-01     | 2024-01-10   | false      |
| 205           | C1           | Delhi     | 2024-01-11     | NULL         | true       |
```

### Fact Join Logic
```sql
JOIN dim_customer d
  ON f.customer_id = d.customer_id
 AND f.order_date BETWEEN d.effective_from AND COALESCE(d.effective_to, '9999-12-31')
```

### ETL Steps
1. Detect change (hash comparison)
2. Close old record (set effective_to, is_current = false)
3. Insert new record (new surrogate key, new effective_from)

---

## 🚀 ETL Patterns

### Incremental Loading Strategies

| Strategy | Use Case | Pattern |
|----------|----------|---------|
| **Append-Only** | Immutable transactions | `WHERE updated_at > last_watermark` |
| **MERGE/UPSERT** | Updates possible | `MERGE ON natural_key` |
| **Partition Overwrite** | Time-based snapshots | Rebuild last N days |
| **CDC** | Real-time changes | Stream inserts/updates/deletes |
| **Rolling Window** | Late data handling | Reprocess last 7 days |

### Idempotency Checklist
- [ ] MERGE logic using natural keys
- [ ] Deduplication before load
- [ ] Partition overwrite for snapshots
- [ ] Batch tracking with load_id
- [ ] Tested reprocessing scenarios

---

## 🎯 Partitioning Best Practices

### What to Partition By
✅ **DO**: Business date (order_date, snapshot_date, trade_date)  
❌ **DON'T**: Ingestion date, surrogate keys, high-cardinality dimensions

### Partitioning vs Clustering

| Aspect | Partitioning | Clustering |
|--------|--------------|------------|
| **Purpose** | Large-scale pruning | Fine-grained pruning |
| **Column** | One (typically date) | Multiple (1-4 columns) |
| **Granularity** | Coarse (daily/monthly) | Fine (within partitions) |
| **Use For** | Time-based queries | High-cardinality filters |

### Example
```sql
-- Partition by date, cluster by customer and product
PARTITION BY RANGE(order_date)
CLUSTER BY (customer_id, product_id)
```

---

## 🔍 Common Grain Violations

| Violation | Problem | Solution |
|-----------|---------|----------|
| **Mixed header/line** | Double counting | Separate fact tables |
| **Pre-aggregated columns** | Grain mismatch | Remove or separate table |
| **Mixed snapshot/transaction** | Aggregation breaks | Separate fact tables |
| **Duplicates** | Grain broken | Fix logical primary key |

### Grain Validation SQL
```sql
-- Should return 0 rows
SELECT logical_key_columns, COUNT(*) as cnt
FROM fact_table
GROUP BY logical_key_columns
HAVING COUNT(*) > 1;
```

---

## 🌉 Bridge Table Pattern

### Structure
```sql
CREATE TABLE bridge_order_promotion (
    order_line_id   BIGINT,
    promotion_key   INT,
    allocation_pct  NUMERIC(8,6),  -- Sums to 1.0 per order_line
    PRIMARY KEY (order_line_id, promotion_key)
);
```

### Query with Allocation
```sql
SELECT p.promotion_name,
       SUM(f.revenue * b.allocation_pct) as allocated_revenue
FROM fact_order_line f
JOIN bridge_order_promotion b ON f.order_line_id = b.order_line_id
JOIN dim_promotion p ON b.promotion_key = p.promotion_key
GROUP BY p.promotion_name;
```

### Validation Checks
- Allocation sums to 1.0 per source row
- No duplicate bridge rows
- Referential integrity maintained

---

## 💡 Interview Answer Templates

### "What is grain?"
> "Grain defines the exact level of detail represented by each row in a fact table, stated as 'one row per X'. It must be declared before adding measures or dimensions because it determines the logical primary key and ensures aggregation correctness."

### "Explain SCD Type 2"
> "Type 2 preserves full history by inserting new rows with new surrogate keys when tracked attributes change. Facts must join using event date within effective date ranges to maintain historical accuracy. This is critical for regulatory reporting and historical analysis."

### "Additive vs Semi-Additive"
> "Additive facts like sales_amount can be summed across all dimensions including time. Semi-additive facts like account_balance represent state and can be summed across entities but not time—you'd use LAST_VALUE or AVG for time aggregations. Store base additive facts and compute ratios in the BI layer."

### "Transaction vs Snapshot"
> "Transaction facts store one row per event with append-only loads and fully additive measures. Periodic snapshots store one row per entity per period with overwrite loads and semi-additive measures representing state. The type is determined by temporal behavior, not aggregation level."

### "How to handle late-arriving facts?"
> "Use event date, not load date, for dimension lookups. Join on business_key AND event_date BETWEEN effective_from AND effective_to. Write to correct business date partition. Support partition backfills. Use MERGE for idempotency. Implement reconciliation checks."

---

## 🚨 Common Mistakes to Avoid

| Mistake | Impact | Fix |
|---------|--------|-----|
| Partition by ingestion_date | Late data goes to wrong partition | Partition by business date |
| Sum balance across time | Meaningless results | Use LAST_VALUE or AVG |
| Store ratios in facts | Non-additive, breaks aggregation | Store base facts, compute in BI |
| Mix grains in one table | Double counting | Separate fact tables |
| Join facts directly | Row explosion | Aggregate first, join through dimensions |
| No grain documentation | Confusion, errors | Document grain explicitly |
| Type 1 for analytical attributes | Lose history | Use Type 2 |
| No idempotency | Duplicates on rerun | MERGE logic |

---

## 📈 Performance Optimization Checklist

- [ ] Partition by business date
- [ ] Cluster by frequently filtered columns
- [ ] Use columnar format (Parquet/ORC)
- [ ] Denormalize dimensions (star schema)
- [ ] Create aggregate facts for common queries
- [ ] Broadcast small dimensions (Spark)
- [ ] Implement incremental loads
- [ ] Optimize file sizes (100-500MB)
- [ ] Use appropriate compression
- [ ] Monitor and tune based on query patterns

---

## 🎓 Senior-Level Talking Points

### Grain Discipline
"Grain is a constraint system, not just documentation. It determines what aggregations are mathematically valid."

### Temporal Correctness
"Facts must link to dimension versions valid at event time, not load time. This requires point-in-time join logic."

### Additivity Awareness
"Semi-additive facts require special handling—partition overwrite, not append, and time-aware aggregations."

### Idempotency
"All ETL must be idempotent. Running the same load twice should produce identical results."

### Conformed Dimensions
"Conformed dimensions enable cross-process analysis and ensure consistent KPIs across the enterprise."

### Performance Trade-offs
"Star schema trades storage for query performance. In columnar databases, this trade-off heavily favors denormalization."

---

## 🔥 Last-Minute Checklist

**15 Minutes Before Interview:**

- [ ] Review grain definition and validation
- [ ] Memorize fact table types (3 core types)
- [ ] Recall SCD Type 2 join logic
- [ ] Remember additivity rules
- [ ] Know partitioning best practices
- [ ] Understand bridge table pattern
- [ ] Prepare 2-3 real project examples
- [ ] Review common mistakes to avoid

**During Interview:**

- [ ] Ask clarifying questions about requirements
- [ ] State grain explicitly when designing
- [ ] Discuss trade-offs (performance vs complexity)
- [ ] Mention data quality and testing
- [ ] Reference real-world experience
- [ ] Admit when you don't know something
- [ ] Think out loud—show your reasoning

---

## 📚 Key Formulas

### Grain Validation
```sql
COUNT(DISTINCT logical_key) = COUNT(*) → Grain is correct
```

### Semi-Additive Aggregation
```sql
-- Ending balance
LAST_VALUE(balance) OVER (PARTITION BY account ORDER BY date)

-- Average balance
AVG(balance) OVER (PARTITION BY account ORDER BY date ROWS BETWEEN 29 PRECEDING AND CURRENT ROW)
```

### Non-Additive Calculation
```sql
-- Margin percentage
SUM(profit) / NULLIF(SUM(sales), 0)

-- Conversion rate
SUM(conversions) / NULLIF(SUM(visits), 0)
```

### Bridge Allocation
```sql
SUM(fact_measure * bridge.allocation_pct)
```

---

## 🎯 Final Tips

1. **Always start with grain** when designing fact tables
2. **Use Type 2 for analytical attributes** that need history
3. **Partition by business date**, not ingestion date
4. **Store base additive facts**, compute ratios in BI layer
5. **Test for idempotency** in all ETL processes
6. **Document everything** - grain, SCD strategy, refresh schedules
7. **Monitor query patterns** and optimize accordingly
8. **Think in terms of trade-offs** - there's no perfect solution

---

*Print this guide and review it 15 minutes before your interview. Focus on understanding the "why" behind each concept, not just memorizing definitions.*
