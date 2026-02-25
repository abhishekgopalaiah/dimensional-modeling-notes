# Data Warehouse Core Concepts: Interview Guide

> **Target Audience**: Senior Data Engineers, Data Architects (8+ years experience)  
> **Purpose**: Master fundamental concepts for technical interviews  
> **Last Updated**: February 2026

---

## Table of Contents

1. [Entities](#entities)
2. [Facts & Fact Tables](#facts--fact-tables)
3. [Grain Discipline](#grain-discipline)
4. [Fact Additivity](#fact-additivity)
5. [Interview Questions](#interview-questions)

---

## Entities

### Definition

**Entity**: A distinct business object or concept about which you want to store data. It represents a real-world thing that has attributes and identity.

### In Dimensional Modeling

- **Dimensions** represent descriptive entities (Customer, Product, Date)
- **Facts** represent measurable events related to entities

### Examples by Domain

| Domain | Entity | Key Attributes |
|--------|--------|---------------|
| **Retail** | Customer | customer_id, name, segment, lifetime_value |
| | Product | product_id, name, category, brand, price |
| | Order | order_id, order_date, status, total_amount |
| **Banking** | Account | account_id, account_type, open_date, balance |
| | Transaction | transaction_id, amount, type, timestamp |
| **Trading** | Instrument | symbol, asset_class, exchange, currency |
| | Trade | trade_id, quantity, price, execution_time |

### Entity Characteristics

- ✅ Has attributes (columns)
- ✅ Has a primary key (natural or surrogate)
- ✅ Exists independently in the business domain
- ✅ Can be related to other entities

---

## Facts & Fact Tables

### What is a Fact?

A **fact** is a measurable business event stored at a defined grain. It is typically numeric and aggregatable.

**Facts answer:**
- **How much?** → revenue, cost, profit
- **How many?** → quantity, count, units
- **How often?** → frequency, occurrences

### What is a Fact Table?

A **fact table** is the central table in a star schema that stores measurable business events at a defined grain.

**Contains:**
- Numeric measures (facts)
- Foreign keys to dimension tables
- Degenerate dimensions (optional - IDs stored directly)

### Fact Table Structure Example

```sql
CREATE TABLE fact_sales (
    -- Dimension Foreign Keys
    date_key            INT,
    product_key         INT,
    customer_key        INT,
    store_key           INT,
    
    -- Degenerate Dimension
    order_number        VARCHAR(20),
    line_number         INT,
    
    -- Additive Facts
    quantity            DECIMAL(10,2),
    sales_amount        DECIMAL(15,2),
    cost_amount         DECIMAL(15,2),
    profit_amount       DECIMAL(15,2),
    
    -- Metadata
    load_timestamp      TIMESTAMP
)
PARTITION BY RANGE(date_key);
```

---

## Grain Discipline

> 🎯 **Critical Rule**: Grain is the single most important concept in dimensional modeling.

### What is Grain?

**Grain** defines exactly what one row in the fact table represents.

> Grain is not just documentation—it's a constraint system that determines:
> - What the logical primary key is
> - What dimensions are allowed
> - What measures are valid
> - What aggregations are mathematically safe

### Grain Declaration Pattern

Always state grain as: **"One row per..."**

| Business Process | Grain Statement | Logical Primary Key |
|-----------------|-----------------|-------------------|
| **Retail Sales** | One row per order line item | order_id + line_number |
| **Banking Transactions** | One row per transaction | transaction_id |
| **Daily Account Balance** | One row per account per day | account_id + date |
| **Trade Execution** | One row per trade | trade_id |
| **Website Clickstream** | One row per page view | session_id + timestamp + page_id |

### Why Grain Matters Mathematically

Aggregation assumes rows are at identical detail level. If grain is inconsistent, `SUM()` becomes invalid.

**Example of Grain Violation:**

```sql
-- WRONG: Mixing order header and line details
-- Some rows represent orders, some represent lines
SELECT SUM(amount) FROM mixed_grain_table;  -- INCORRECT RESULT!
```

**Correct Approach:**

```sql
-- Separate tables with clear grain
fact_order_header:  -- One row per order
fact_order_line:    -- One row per order line item
```

### Common Grain Violations

#### ❌ Violation 1: Mixing Header and Line Grain

```sql
-- WRONG: Same table contains both
| order_id | line_number | product | amount |
| O1       | NULL        | NULL    | 1000   |  -- Order header
| O1       | 1           | P1      | 500    |  -- Line item
| O1       | 2           | P2      | 500    |  -- Line item
-- SUM(amount) = 2000 (WRONG! Should be 1000)
```

#### ❌ Violation 2: Adding Pre-Aggregated Columns

```sql
-- WRONG: Mixing grains in columns
CREATE TABLE fact_order_line (
    order_id        VARCHAR(20),
    line_number     INT,
    line_amount     DECIMAL(15,2),  -- Line grain
    order_total     DECIMAL(15,2)   -- Order grain (WRONG!)
);
-- Summing line_amount will multiply order_total
```

#### ❌ Violation 3: Mixing Snapshot and Transaction Data

```sql
-- WRONG: Same table contains both
-- Individual transactions AND monthly summaries
-- This destroys aggregation correctness
```

### Grain Validation Checklist

Every fact table should pass these checks:

- [ ] Can you describe grain in one sentence?
- [ ] Can you define the logical primary key?
- [ ] Does `SUM()` behave correctly?
- [ ] Are all measures at the same level?
- [ ] Are all dimension keys valid for that level?
- [ ] Are there duplicates violating grain?

### Grain and Primary Key Thinking

Even if not physically enforced, every fact table has a **logical primary key** defined by grain.

**Example:**
- Grain: One row per account per day
- Logical key: (account_id, date)
- If duplicates exist → grain is broken

**Test with SQL:**
```sql
-- Check for grain violations
SELECT account_id, date, COUNT(*) as row_count
FROM fact_account_balance
GROUP BY account_id, date
HAVING COUNT(*) > 1;  -- Should return 0 rows
```

### Multi-Grain Modeling (Correct Pattern)

Different grains coexist in **separate tables** (constellation schema):

```sql
-- Transaction grain
fact_sales_transaction:  -- One row per order line

-- Daily summary grain  
fact_sales_daily:        -- One row per product per day

-- Both valid, but NEVER mixed in one table
```

---

## Fact Additivity

Understanding additivity is critical for correct aggregation and query design.

### Three Types of Facts

| Type | Sum Across Entities? | Sum Across Time? | Example |
|------|---------------------|------------------|---------|
| **Additive** | ✅ Yes | ✅ Yes | sales_amount, quantity |
| **Semi-Additive** | ✅ Yes | ❌ No | account_balance, inventory |
| **Non-Additive** | ❌ No | ❌ No | margin%, ratios, averages |

### 1. Additive Facts

**Definition**: Can be summed across **all dimensions**, including time.

**Example:**
```sql
-- All these aggregations are valid
SELECT SUM(sales_amount) FROM fact_sales;  -- Total sales
SELECT SUM(sales_amount) FROM fact_sales WHERE date_key = 20240115;  -- Daily
SELECT product_key, SUM(sales_amount) FROM fact_sales GROUP BY product_key;  -- By product
```

**Why Preferred:**
- ✅ Flexible aggregation at any level
- ✅ Supports drill-down and roll-up
- ✅ No special handling required
- ✅ BI tools work naturally

### 2. Semi-Additive Facts

**Definition**: Can be summed across **some dimensions** but **NOT across time**.

> ⚠️ **Key Issue**: Time dimension breaks additivity because these represent **state**, not **flow**.

**Example: Account Balance**

```sql
-- ✅ CORRECT: Sum across accounts (single day)
SELECT date_key, SUM(account_balance) as total_exposure
FROM fact_account_balance
WHERE date_key = 20240115
GROUP BY date_key;

-- ❌ WRONG: Cannot sum balance across time
SELECT account_key, SUM(account_balance)  -- Meaningless!
FROM fact_account_balance
WHERE date_key BETWEEN 20240101 AND 20240131
GROUP BY account_key;
```

**Correct Time Aggregations:**
```sql
-- ✅ Last value (ending balance)
SELECT account_key, 
       LAST_VALUE(account_balance) OVER (
           PARTITION BY account_key 
           ORDER BY date_key
       ) as ending_balance
FROM fact_account_balance;

-- ✅ Average balance
SELECT account_key, AVG(account_balance) as avg_balance
FROM fact_account_balance
GROUP BY account_key;
```

**Impact on Data Engineering:**

```python
# Partition overwrite strategy (not append)
def load_daily_balance(business_date):
    date_key = format_date_key(business_date)
    
    # Overwrite partition (idempotent)
    df_snapshot.write \
        .mode("overwrite") \
        .option("partitionOverwriteMode", "dynamic") \
        .partitionBy("date_key") \
        .save("fact_account_balance")
```

### 3. Non-Additive Facts

**Definition**: Cannot be summed across **any dimension**.

> ❌ **Rule**: Never use `SUM()` - must recalculate from base measures.

**Example: Profit Margin**

```sql
-- ❌ WRONG: Cannot sum percentages
SELECT SUM(profit_margin_pct) FROM fact_sales;

-- ✅ CORRECT: Store base facts, calculate ratio
CREATE TABLE fact_sales (
    sales_amount    DECIMAL(15,2),  -- Additive
    cost_amount     DECIMAL(15,2),  -- Additive
    profit_amount   DECIMAL(15,2)   -- Additive
    -- Do NOT store: profit_margin_pct
);

-- Calculate in query layer
SELECT 
    product_key,
    SUM(profit_amount) / NULLIF(SUM(sales_amount), 0) as margin_pct
FROM fact_sales
GROUP BY product_key;
```

**Common Non-Additive Metrics:**

| Metric | Correct Calculation |
|--------|-------------------|
| Conversion Rate | `SUM(conversions) / SUM(visits)` |
| Click-Through Rate | `SUM(clicks) / SUM(impressions)` |
| Average Order Value | `SUM(order_amount) / COUNT(orders)` |
| Utilization % | `SUM(used_capacity) / SUM(total_capacity)` |

### Architecture Principles

#### 1. Store Additive Facts When Possible
```sql
-- ✅ Preferred
sales_amount, cost_amount, profit_amount

-- ❌ Avoid
profit_margin_pct
```

#### 2. Handle Semi-Additive with Care
```python
# Partition strategy
PARTITION BY date_key

# Load strategy
mode = "overwrite"  # Not append

# Query pattern
WHERE date_key = specific_date  # Always filter
```

#### 3. Never Store Non-Additive Directly
```sql
-- ✅ Compute in view/BI layer
CREATE VIEW v_sales_metrics AS
SELECT 
    product_key,
    SUM(sales_amount) as total_sales,
    SUM(profit_amount) / NULLIF(SUM(sales_amount), 0) as margin_pct
FROM fact_sales
GROUP BY product_key;
```

---

## Interview Questions

### Q1: What is grain in dimensional modeling?

**Model Answer:**
> "Grain defines the exact level of detail represented by each row in a fact table, typically stated as 'one row per X'. It must be declared before adding any measures or dimensions because it determines the logical primary key, valid dimensions, and mathematically correct aggregations. Violating grain discipline leads to double counting and broken aggregations."

### Q2: Explain the difference between additive, semi-additive, and non-additive facts.

**Model Answer:**
> "Additive facts like sales_amount can be summed across all dimensions including time. Semi-additive facts like account_balance represent state and can be summed across entities but not across time—you'd use LAST_VALUE or AVG instead. Non-additive facts like profit margin percentage cannot be summed at all and must be recalculated from base additive measures. In practice, we store base additive facts and compute ratios in the BI layer to preserve mathematical integrity."

### Q3: How do you validate that a fact table has correct grain?

**Model Answer:**
> "First, verify you can state the grain in one sentence starting with 'one row per'. Then check: (1) Can you define the logical primary key? (2) Does a query for duplicates on that key return zero rows? (3) Are all measures at the same level of detail? (4) Does SUM() produce correct results? (5) Are all dimension foreign keys valid for that grain? If any check fails, the grain is violated and must be redesigned."

### Q4: What happens if you mix order header and line item grain in one table?

**Model Answer:**
> "Mixing grains causes double counting. If you store both order totals and line items in the same table, summing the amount column will count order totals multiple times—once for the header row and again across all line items. This breaks aggregation correctness. The solution is to create separate fact tables: fact_order_header at order grain and fact_order_line at line item grain."

### Q5: Why can't you sum account balances across time?

**Model Answer:**
> "Account balance is a semi-additive fact representing state at a point in time, not a flow. If an account has $1,000 on Jan 1 and $1,200 on Jan 2, summing gives $2,200 which is meaningless. The correct approach is to use LAST_VALUE for ending balance, AVG for average balance, or MAX/MIN depending on business needs. You can sum balances across accounts on a single date because that represents total exposure at that moment."

### Q6: How does grain discipline relate to partitioning strategy?

**Model Answer:**
> "Grain must align with physical design. If the grain is 'one row per account per day', you should partition by business date (the date in the grain), not ingestion date. This ensures partition pruning works correctly, late-arriving facts go to the right partition, and queries filtering on business date are efficient. Misalignment between grain and partitioning breaks both performance and temporal correctness."

### Q7: What is a degenerate dimension?

**Model Answer:**
> "A degenerate dimension is a dimension attribute stored directly in the fact table with no separate dimension table. Common examples are invoice numbers or order IDs that have no additional descriptive attributes. Storing them in the fact table avoids an unnecessary join while still providing the ability to filter and group by that identifier."

### Q8: Can you have multiple grains in a data warehouse?

**Model Answer:**
> "Yes, through a multi-fact constellation schema where different fact tables operate at different grains but share conformed dimensions. For example, fact_sales_transaction at 'one row per order line' and fact_inventory_snapshot at 'one row per product per store per day'. Each fact maintains its own grain discipline, but they can be analyzed together through shared dimensions like product and date."

---

## Key Takeaways

1. **Grain is a contract** - Each row represents exactly one thing
2. **Declare grain first** - Before adding any measures or dimensions
3. **Validate grain rigorously** - Test for duplicates and aggregation correctness
4. **Additivity matters** - Understand which facts can be summed and how
5. **Physical design follows logical design** - Partitioning must align with grain
6. **Separate grains = separate tables** - Never mix grains in one fact table

---

*This guide covers the fundamental concepts that form the foundation of all dimensional modeling. Master these before moving to advanced topics.*
