# Data Warehouse Fundamentals: Architecture Guide

> **Audience**: Senior Data Engineers, Data Architects, Data Platform Engineers  
> **Focus**: Dimensional modeling, fact table design, ETL patterns, performance optimization  
> **Level**: Architecture and interview preparation

---

## Table of Contents

1. [Core Concepts](#core-concepts)
   - [Entities](#entities)
   - [Facts & Fact Tables](#facts--fact-tables)
   - [Grain Definition](#grain-definition)
2. [Fact Additivity](#fact-additivity)
3. [Fact Table Types](#fact-table-types)
4. [Advanced Topics](#advanced-topics-for-senior-engineers)
5. [Senior Readiness Checklist](#-senior-readiness-checklist)

---

## Core Concepts

### Entities

**Definition**: An entity is a distinct business object or concept about which you want to store data. It represents a real-world thing that has attributes and identity.

In dimensional modeling:
- **Dimensions** represent descriptive entities (Customer, Product, Date)
- **Facts** represent measurable events related to entities

#### Examples by Domain

| Domain | Entity | Attributes Example |
|--------|--------|-------------------|
| **Retail** | Customer | customer_id, name, segment, lifetime_value, signup_date |
| | Product | product_id, name, category, department, brand |
| | Order | order_id, order_date, order_status |
| **Banking** | Account | account_id, account_type, open_date, branch_id |
| | Transaction | transaction_id, amount, transaction_type, timestamp |
| **Trading** | Instrument | symbol, asset_class, exchange, currency |
| | Trade | trade_id, quantity, price, execution_time |
| **HR** | Employee | employee_id, name, department, hire_date, salary_band |
| | Department | dept_id, dept_name, manager_id, cost_center |

#### Entity Characteristics
- Has attributes (columns)
- Has a primary key (natural or surrogate)
- Exists independently in the business domain
- Can be related to other entities

---

### Facts & Fact Tables

#### What is a Fact?

A **fact** is a measurable business event stored at a defined grain. It is typically numeric and aggregatable, representing the quantitative outcome of a business process.

Facts answer questions like:
- **How much?** → revenue, cost, profit
- **How many?** → quantity, count, units
- **How often?** → frequency, occurrences

#### What is a Fact Table?

A **fact table** is the central table in a star schema that stores measurable business events at a defined grain. It contains:
- **Numeric measures** (facts)
- **Foreign keys** to dimension tables
- **Degenerate dimensions** (optional - transaction IDs stored directly)

#### Fact Table Structure

```sql
-- Example: Retail Sales Fact Table
CREATE TABLE fact_sales (
    -- Dimension Foreign Keys
    date_key            INT,
    product_key         INT,
    customer_key        INT,
    store_key           INT,
    promotion_key       INT,
    
    -- Degenerate Dimension
    order_number        VARCHAR(20),
    line_number         INT,
    
    -- Additive Facts
    quantity            DECIMAL(10,2),
    sales_amount        DECIMAL(15,2),
    discount_amount     DECIMAL(15,2),
    cost_amount         DECIMAL(15,2),
    profit_amount       DECIMAL(15,2),
    
    -- Semi-Additive Facts (if snapshot)
    -- inventory_quantity  INT,
    
    -- Non-Additive Facts (avoid storing directly)
    -- profit_margin_pct   DECIMAL(5,2),  -- Calculate as profit/sales
    
    -- Metadata
    load_timestamp      TIMESTAMP
)
PARTITION BY RANGE(date_key);
```

---

### Grain Definition

**Grain** defines exactly what one row in the fact table represents.

> **Critical Rule**: The grain must be declared **before** adding any measures or dimensions.

#### Grain Declaration Examples

| Business Process | Grain Statement | Logical Primary Key |
|-----------------|-----------------|-------------------|
| **Retail Sales** | One row per order line item | order_id + line_number |
| **Banking Transactions** | One row per transaction | transaction_id |
| **Daily Account Balance** | One row per account per day | account_id + date |
| **Trade Execution** | One row per trade | trade_id |
| **Website Clickstream** | One row per page view | session_id + timestamp + page_id |
| **Order Lifecycle** | One row per order (updated) | order_id |

#### Why Grain Matters

**Correct Grain**:
```sql
-- Grain: One row per order line item
SELECT order_id, line_number, product_id, quantity, amount
FROM fact_sales
WHERE order_id = 'ORD12345';

-- Result:
-- ORD12345, 1, PROD_A, 2, 100.00
-- ORD12345, 2, PROD_B, 1, 50.00
```

**Incorrect Mixed Grain** (violation):
```sql
-- WRONG: Mixing order header and line details
-- Some rows are order-level, some are line-level
-- This breaks aggregation logic
```

#### Grain Discipline Checklist

- [ ] Grain declared explicitly before design
- [ ] All measures conform to grain
- [ ] All dimensions conform to grain
- [ ] No mixed-grain violations
- [ ] Logical primary key identified
- [ ] No duplicate grain rows possible

## Fact Additivity

Understanding additivity is critical for correct aggregation and query design.

### 1️⃣ Additive Facts

#### Definition
A fact is **additive** if it can be summed across **all dimensions**, including time.

> ✅ **Rule**: `SUM()` works correctly across every dimension.

#### Example 1: Retail Sales

**Grain**: One row per order line item

**Additive Facts**:
- `quantity` - units sold
- `sales_amount` - revenue
- `discount_amount` - discounts applied
- `cost_amount` - cost of goods
- `profit_amount` - profit earned

**Valid Aggregations**:
```sql
-- Sum across time
SELECT SUM(sales_amount) FROM fact_sales 
WHERE date_key BETWEEN 20240101 AND 20240131;

-- Sum across products
SELECT product_key, SUM(sales_amount) FROM fact_sales GROUP BY product_key;

-- Sum across stores
SELECT store_key, SUM(sales_amount) FROM fact_sales GROUP BY store_key;

-- Sum across all dimensions
SELECT SUM(sales_amount) FROM fact_sales;
```

All aggregations are mathematically correct.

#### Example 2: Trading System

**Grain**: One row per trade execution

**Additive Facts**:
- `trade_quantity` - shares/contracts traded
- `trade_value` - notional value
- `commission_amount` - fees paid

```sql
-- Sum by instrument
SELECT instrument_id, SUM(trade_quantity) as total_volume
FROM fact_trades
WHERE trade_date = '2024-01-15'
GROUP BY instrument_id;

-- Sum by time period
SELECT DATE_TRUNC('hour', trade_timestamp) as hour,
       SUM(trade_value) as hourly_volume
FROM fact_trades
GROUP BY 1;
```

#### Why Additive Facts Are Preferred
- ✅ Flexible aggregation at any level
- ✅ Supports drill-down and roll-up
- ✅ No special handling required
- ✅ BI tools work naturally

---

### 2️⃣ Semi-Additive Facts

#### Definition
A fact is **semi-additive** if it can be summed across **some dimensions** but **NOT across time**.

> ⚠️ **Key Issue**: Time dimension breaks additivity because these represent **state**, not **flow**.

#### Example 1: Bank Account Balance

**Grain**: One row per account per day

**Semi-Additive Fact**: `account_balance`

**Valid Aggregations**:
```sql
-- ✅ Sum across accounts (on a single day)
SELECT date_key, SUM(account_balance) as total_bank_exposure
FROM fact_account_balance
WHERE date_key = 20240115
GROUP BY date_key;

-- ✅ Sum across regions (on a single day)
SELECT region, SUM(account_balance) as regional_exposure
FROM fact_account_balance f
JOIN dim_account a ON f.account_key = a.account_key
WHERE date_key = 20240115
GROUP BY region;
```

**Invalid Aggregation**:
```sql
-- ❌ WRONG: Cannot sum balance across time
SELECT account_key, SUM(account_balance) as total_balance
FROM fact_account_balance
WHERE date_key BETWEEN 20240101 AND 20240131
GROUP BY account_key;

-- Why wrong?
-- If account balance is:
-- Jan 1: $1,000
-- Jan 2: $1,200  
-- Jan 3: $1,100
-- SUM = $3,300 (meaningless!)
```

**Correct Time Aggregations**:
```sql
-- ✅ Last value (ending balance)
SELECT account_key, 
       LAST_VALUE(account_balance) OVER (
           PARTITION BY account_key 
           ORDER BY date_key
       ) as ending_balance
FROM fact_account_balance
WHERE date_key BETWEEN 20240101 AND 20240131;

-- ✅ Average balance
SELECT account_key, AVG(account_balance) as avg_balance
FROM fact_account_balance
WHERE date_key BETWEEN 20240101 AND 20240131
GROUP BY account_key;

-- ✅ Max/Min balance
SELECT account_key, 
       MAX(account_balance) as peak_balance,
       MIN(account_balance) as low_balance
FROM fact_account_balance
WHERE date_key BETWEEN 20240101 AND 20240131
GROUP BY account_key;
```

#### Example 2: Inventory Snapshot

**Grain**: One row per product per store per day

**Semi-Additive Fact**: `on_hand_quantity`

```sql
-- ✅ Total inventory across all stores (single day)
SELECT date_key, SUM(on_hand_quantity) as total_inventory
FROM fact_inventory
WHERE date_key = 20240115
GROUP BY date_key;

-- ❌ WRONG: Sum across days
SELECT product_key, SUM(on_hand_quantity)
FROM fact_inventory
WHERE date_key BETWEEN 20240101 AND 20240131
GROUP BY product_key;
```

#### Impact on Data Engineering

**Partitioning Strategy**:
```sql
-- Partition by business date (not ingestion date)
CREATE TABLE fact_account_balance (
    date_key INT,
    account_key INT,
    account_balance DECIMAL(15,2)
)
PARTITION BY RANGE(date_key);
```

**Incremental Load Pattern**:
```python
# Overwrite partition strategy (not append)
def load_daily_balance(business_date):
    # Delete existing partition
    spark.sql(f"""
        DELETE FROM fact_account_balance 
        WHERE date_key = {business_date}
    """)
    
    # Insert new snapshot
    df_balance.write.mode("append").insertInto("fact_account_balance")
```

**Query Optimization**:
```sql
-- Always filter by specific date for semi-additive facts
SELECT store_key, SUM(on_hand_quantity)
FROM fact_inventory
WHERE date_key = 20240115  -- Critical filter
GROUP BY store_key;
```

---

### 3️⃣ Non-Additive Facts

#### Definition
A fact is **non-additive** if it **cannot be summed across any dimension**.

> ❌ **Rule**: Never use `SUM()` - must recalculate from base measures.

#### Example 1: Profit Margin Percentage

**Problem**:
```sql
-- ❌ WRONG: Cannot sum percentages
SELECT SUM(profit_margin_pct) FROM fact_sales;

-- If Product A has 20% margin and Product B has 30% margin
-- Total margin is NOT 50%!
```

**Correct Approach**:
```sql
-- ✅ Store base additive facts
CREATE TABLE fact_sales (
    sales_amount    DECIMAL(15,2),  -- Additive
    cost_amount     DECIMAL(15,2),  -- Additive
    profit_amount   DECIMAL(15,2)   -- Additive (sales - cost)
    -- Do NOT store: profit_margin_pct
);

-- ✅ Calculate ratio in query layer
SELECT 
    product_key,
    SUM(profit_amount) / NULLIF(SUM(sales_amount), 0) as profit_margin_pct
FROM fact_sales
GROUP BY product_key;
```

#### Example 2: Other Non-Additive Metrics

| Metric | Type | Correct Calculation |
|--------|------|-------------------|
| **Conversion Rate** | Ratio | `SUM(conversions) / SUM(visits)` |
| **Click-Through Rate (CTR)** | Ratio | `SUM(clicks) / SUM(impressions)` |
| **Average Order Value** | Average | `SUM(order_amount) / COUNT(orders)` |
| **Utilization %** | Ratio | `SUM(used_capacity) / SUM(total_capacity)` |
| **Risk Score** | Weighted Avg | `SUM(risk * weight) / SUM(weight)` |
| **Temperature** | Average | `AVG(temperature)` not `SUM()` |

#### Storage Pattern

```sql
-- ✅ GOOD: Store base additive facts
CREATE TABLE fact_marketing (
    impressions     BIGINT,      -- Additive
    clicks          BIGINT,      -- Additive
    conversions     INT,         -- Additive
    spend_amount    DECIMAL(15,2) -- Additive
    -- Calculate in BI layer:
    -- ctr = clicks / impressions
    -- conversion_rate = conversions / clicks
    -- cpa = spend_amount / conversions
);

-- ❌ BAD: Storing derived ratios
CREATE TABLE fact_marketing_bad (
    ctr_pct             DECIMAL(5,2),  -- Non-additive
    conversion_rate_pct DECIMAL(5,2),  -- Non-additive
    cpa_amount          DECIMAL(10,2)  -- Non-additive
);
```

---

### Comparison Matrix

| Fact Type | Sum Across Entities? | Sum Across Time? | Example | Aggregation Functions |
|-----------|---------------------|------------------|---------|---------------------|
| **Additive** | ✅ Yes | ✅ Yes | sales_amount, quantity | SUM, AVG, COUNT |
| **Semi-Additive** | ✅ Yes | ❌ No | account_balance, inventory | SUM (single date), AVG, LAST_VALUE, MAX, MIN |
| **Non-Additive** | ❌ No | ❌ No | margin%, ratios, averages | Recalculate from base facts |

---

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
    SUM(profit_amount) as total_profit,
    SUM(profit_amount) / NULLIF(SUM(sales_amount), 0) as margin_pct
FROM fact_sales
GROUP BY product_key;
```

---

### Interview Answer Template

**Question**: "Explain the difference between additive, semi-additive, and non-additive facts."

**Answer**:
> "Additive facts like sales_amount can be summed across all dimensions including time. Semi-additive facts like account_balance represent state and can be summed across entities but not across time—you'd use LAST_VALUE or AVG instead. Non-additive facts like profit margin percentage cannot be summed at all and must be recalculated from base additive measures. In practice, we store base additive facts and compute ratios in the BI layer to preserve mathematical integrity."

---

### Why This Matters in Modern Platforms

#### Spark/BigQuery Context
```python
# ❌ Wrong: Pre-aggregating semi-additive incorrectly
df_wrong = spark.sql("""
    SELECT account_id, SUM(balance) as total_balance
    FROM daily_balance
    WHERE date BETWEEN '2024-01-01' AND '2024-01-31'
    GROUP BY account_id
""")

# ✅ Correct: Use window functions
df_correct = spark.sql("""
    SELECT account_id,
           LAST_VALUE(balance) OVER (
               PARTITION BY account_id 
               ORDER BY date
           ) as ending_balance
    FROM daily_balance
    WHERE date BETWEEN '2024-01-01' AND '2024-01-31'
""")
```

#### ML/Feature Engineering
```python
# ❌ Wrong: Training on incorrectly aggregated balances
features = df.groupBy("customer_id").agg(
    F.sum("daily_balance").alias("total_balance")  # Meaningless
)

# ✅ Correct: Use proper aggregations
features = df.groupBy("customer_id").agg(
    F.avg("daily_balance").alias("avg_balance"),
    F.max("daily_balance").alias("peak_balance"),
    F.last("daily_balance").alias("current_balance")
)
```

**Impact**: Incorrect additivity handling leads to **silent data corruption** that propagates through analytics and ML models.

## Fact Table Types

Fact tables are classified based on **how they capture business processes over time**, not by aggregation level.

> 🎯 **Key Principle**: Fact type is determined by **behavioral pattern**, not grain or aggregation.

### Three Core Types

| Type | Grain Pattern | Load Pattern | Use Case |
|------|--------------|--------------|----------|
| **Transaction** | One row per event | Append-only | Capture individual business events |
| **Periodic Snapshot** | One row per entity per period | Overwrite/Append | Track state at regular intervals |
| **Accumulating Snapshot** | One row per process instance | Update | Monitor lifecycle progression |

---

### 1️⃣ Transaction Fact Table

#### Definition
Stores **one row per business event**. This is the most common and atomic form of fact table.

**Grain**: One row per transaction/event

**Characteristics**:
- ✅ Fully additive (typically)
- 📈 High volume (millions to billions of rows)
- ➕ Append-only load pattern
- 🔍 Most granular form
- 🎯 Best for drill-down analysis

#### Real-World Examples

**Retail Sales**:
```sql
CREATE TABLE fact_sales (
    -- Grain: One row per order line item
    order_id            VARCHAR(20),
    line_number         INT,
    
    -- Dimension Keys
    order_date_key      INT,
    product_key         INT,
    customer_key        INT,
    store_key           INT,
    promotion_key       INT,
    
    -- Additive Measures
    quantity            DECIMAL(10,2),
    unit_price          DECIMAL(10,2),
    sales_amount        DECIMAL(15,2),
    discount_amount     DECIMAL(15,2),
    cost_amount         DECIMAL(15,2),
    profit_amount       DECIMAL(15,2),
    
    -- Metadata
    transaction_timestamp TIMESTAMP,
    load_timestamp        TIMESTAMP
)
PARTITION BY RANGE(order_date_key);
```

**Banking Transactions**:
```sql
CREATE TABLE fact_transactions (
    -- Grain: One row per transaction
    transaction_id      VARCHAR(50),
    
    -- Dimension Keys
    transaction_date_key INT,
    account_key         INT,
    transaction_type_key INT,
    channel_key         INT,
    merchant_key        INT,
    
    -- Additive Measures
    transaction_amount  DECIMAL(15,2),
    fee_amount          DECIMAL(10,2),
    
    -- Degenerate Dimensions
    authorization_code  VARCHAR(20),
    
    transaction_timestamp TIMESTAMP
)
PARTITION BY RANGE(transaction_date_key);
```

**Trading System**:
```sql
CREATE TABLE fact_trades (
    -- Grain: One row per trade execution
    trade_id            VARCHAR(50),
    
    -- Dimension Keys
    trade_date_key      INT,
    instrument_key      INT,
    trader_key          INT,
    counterparty_key    INT,
    strategy_key        INT,
    
    -- Additive Measures
    quantity            DECIMAL(18,4),
    execution_price     DECIMAL(18,6),
    trade_value         DECIMAL(20,2),
    commission_amount   DECIMAL(10,2),
    
    execution_timestamp TIMESTAMP
)
PARTITION BY RANGE(trade_date_key);
```

#### ETL Pattern

```python
# Append-only incremental load
def load_transaction_fact(business_date):
    """
    Transaction facts use append-only pattern
    """
    # Extract new transactions
    df_new_transactions = extract_transactions(business_date)
    
    # Transform and validate
    df_transformed = transform_transactions(df_new_transactions)
    
    # Append to fact table (no updates)
    df_transformed.write \
        .mode("append") \
        .partitionBy("order_date_key") \
        .format("parquet") \
        .save("fact_sales")
    
    # Idempotency: Use MERGE if reprocessing
    # spark.sql("""
    #     MERGE INTO fact_sales t
    #     USING staging_sales s
    #     ON t.order_id = s.order_id AND t.line_number = s.line_number
    #     WHEN NOT MATCHED THEN INSERT *
    # """)
```

#### Query Patterns

```sql
-- Flexible aggregation at any level
SELECT 
    d.year,
    d.quarter,
    p.category,
    SUM(f.sales_amount) as total_sales,
    SUM(f.quantity) as total_units,
    COUNT(DISTINCT f.order_id) as order_count
FROM fact_sales f
JOIN dim_date d ON f.order_date_key = d.date_key
JOIN dim_product p ON f.product_key = p.product_key
WHERE d.year = 2024
GROUP BY d.year, d.quarter, p.category;

-- Drill-down to transaction detail
SELECT 
    f.order_id,
    f.line_number,
    d.full_date,
    p.product_name,
    f.quantity,
    f.sales_amount
FROM fact_sales f
JOIN dim_date d ON f.order_date_key = d.date_key
JOIN dim_product p ON f.product_key = p.product_key
WHERE f.order_id = 'ORD123456';
```

#### When to Use
- ✅ Capturing individual business events
- ✅ Need for detailed drill-down
- ✅ Flexible aggregation requirements
- ✅ ML feature engineering
- ✅ Root cause analysis
- ✅ Regulatory audit trails

---

### 2️⃣ Periodic Snapshot Fact Table

#### Definition
Stores the **state of something at regular intervals** (daily, weekly, monthly).

**Grain**: One row per entity per time period

**Characteristics**:
- ⚠️ Semi-additive measures (typically)
- 📊 Predictable growth (rows = entities × periods)
- 🔄 Overwrite or append per period
- 📅 Time-series analysis
- 🎯 Trend tracking

#### Real-World Examples

**Daily Account Balance**:
```sql
CREATE TABLE fact_account_balance_daily (
    -- Grain: One row per account per day
    date_key            INT,
    account_key         INT,
    
    -- Semi-Additive Measures (state)
    opening_balance     DECIMAL(15,2),
    closing_balance     DECIMAL(15,2),
    available_balance   DECIMAL(15,2),
    
    -- Additive Measures (flow during day)
    total_deposits      DECIMAL(15,2),
    total_withdrawals   DECIMAL(15,2),
    transaction_count   INT,
    
    -- Metadata
    snapshot_timestamp  TIMESTAMP,
    
    PRIMARY KEY (date_key, account_key)
)
PARTITION BY RANGE(date_key);
```

**Daily Inventory Snapshot**:
```sql
CREATE TABLE fact_inventory_daily (
    -- Grain: One row per product per store per day
    date_key            INT,
    product_key         INT,
    store_key           INT,
    
    -- Semi-Additive (state)
    on_hand_quantity    INT,
    on_order_quantity   INT,
    reserved_quantity   INT,
    available_quantity  INT,
    
    -- Additive (flow during day)
    units_received      INT,
    units_sold          INT,
    units_adjusted      INT,
    
    -- Metadata
    snapshot_timestamp  TIMESTAMP,
    
    PRIMARY KEY (date_key, product_key, store_key)
)
PARTITION BY RANGE(date_key);
```

**Monthly Subscription Snapshot**:
```sql
CREATE TABLE fact_subscription_monthly (
    -- Grain: One row per customer per month
    month_key           INT,
    customer_key        INT,
    
    -- Semi-Additive (state at month-end)
    active_subscriptions INT,
    mrr_amount          DECIMAL(10,2),  -- Monthly Recurring Revenue
    
    -- Additive (flow during month)
    new_subscriptions   INT,
    cancelled_subscriptions INT,
    upgrade_count       INT,
    downgrade_count     INT,
    
    PRIMARY KEY (month_key, customer_key)
)
PARTITION BY RANGE(month_key);
```

#### ETL Pattern

```python
# Partition overwrite pattern
def load_periodic_snapshot(business_date):
    """
    Periodic snapshots use overwrite pattern per partition
    """
    date_key = format_date_key(business_date)
    
    # Calculate snapshot for the date
    df_snapshot = calculate_daily_snapshot(business_date)
    
    # Overwrite partition (idempotent)
    df_snapshot.write \
        .mode("overwrite") \
        .option("partitionOverwriteMode", "dynamic") \
        .partitionBy("date_key") \
        .format("parquet") \
        .save("fact_inventory_daily")
    
    # Alternative: Delete + Insert pattern
    # spark.sql(f"DELETE FROM fact_inventory_daily WHERE date_key = {date_key}")
    # df_snapshot.write.mode("append").insertInto("fact_inventory_daily")
```

#### Query Patterns

```sql
-- ✅ Correct: Sum across entities (single date)
SELECT 
    date_key,
    SUM(closing_balance) as total_bank_exposure,
    COUNT(DISTINCT account_key) as active_accounts
FROM fact_account_balance_daily
WHERE date_key = 20240115
GROUP BY date_key;

-- ✅ Correct: Trend analysis (use proper aggregation)
SELECT 
    account_key,
    AVG(closing_balance) as avg_balance,
    MAX(closing_balance) as peak_balance,
    MIN(closing_balance) as low_balance
FROM fact_account_balance_daily
WHERE date_key BETWEEN 20240101 AND 20240131
GROUP BY account_key;

-- ✅ Correct: Month-over-month comparison
SELECT 
    account_key,
    MAX(CASE WHEN date_key = 20240131 THEN closing_balance END) as jan_balance,
    MAX(CASE WHEN date_key = 20240229 THEN closing_balance END) as feb_balance
FROM fact_account_balance_daily
WHERE date_key IN (20240131, 20240229)
GROUP BY account_key;

-- ❌ WRONG: Summing balance across time
SELECT account_key, SUM(closing_balance)  -- Meaningless!
FROM fact_account_balance_daily
WHERE date_key BETWEEN 20240101 AND 20240131
GROUP BY account_key;
```

#### When to Use
- ✅ Tracking state over time
- ✅ Point-in-time reporting
- ✅ Trend analysis
- ✅ Inventory management
- ✅ Portfolio tracking
- ✅ Subscription metrics

---

### 3️⃣ Accumulating Snapshot Fact Table

#### Definition
Tracks the **lifecycle of a process** across multiple stages/milestones.

**Grain**: One row per process instance (updated as it progresses)

**Characteristics**:
- 🔄 Frequently updated (not append-only)
- 📅 Multiple date columns (one per milestone)
- ⏱️ Duration metrics between stages
- 🎯 Pipeline/funnel tracking
- 📊 SLA monitoring

#### Real-World Examples

**Order Fulfillment Pipeline**:
```sql
CREATE TABLE fact_order_pipeline (
    -- Grain: One row per order (updated as it progresses)
    order_id            VARCHAR(50) PRIMARY KEY,
    
    -- Dimension Keys
    customer_key        INT,
    product_key         INT,
    store_key           INT,
    
    -- Milestone Date Keys (updated as milestones reached)
    order_date_key      INT,        -- Set when order placed
    payment_date_key    INT,        -- Set when payment confirmed
    pick_date_key       INT,        -- Set when picked from warehouse
    pack_date_key       INT,        -- Set when packed
    ship_date_key       INT,        -- Set when shipped
    delivery_date_key   INT,        -- Set when delivered
    
    -- Status (updated)
    current_status      VARCHAR(20), -- 'ordered', 'paid', 'shipped', 'delivered'
    
    -- Additive Measures
    order_amount        DECIMAL(15,2),
    shipping_cost       DECIMAL(10,2),
    
    -- Duration Metrics (calculated)
    order_to_ship_days  INT,
    ship_to_delivery_days INT,
    total_fulfillment_days INT,
    
    -- Metadata
    last_updated_timestamp TIMESTAMP
);
```

**Loan Application Lifecycle**:
```sql
CREATE TABLE fact_loan_application (
    -- Grain: One row per loan application
    application_id      VARCHAR(50) PRIMARY KEY,
    
    -- Dimension Keys
    applicant_key       INT,
    loan_product_key    INT,
    branch_key          INT,
    
    -- Milestone Dates
    application_date_key    INT,
    document_submit_date_key INT,
    credit_check_date_key   INT,
    underwriting_date_key   INT,
    approval_date_key       INT,
    funding_date_key        INT,
    
    -- Status
    current_stage       VARCHAR(30),
    final_decision      VARCHAR(20), -- 'approved', 'rejected', 'withdrawn'
    
    -- Measures
    requested_amount    DECIMAL(15,2),
    approved_amount     DECIMAL(15,2),
    
    -- Duration Metrics
    application_to_decision_days INT,
    approval_to_funding_days INT,
    
    last_updated_timestamp TIMESTAMP
);
```

**Support Ticket Resolution**:
```sql
CREATE TABLE fact_ticket_lifecycle (
    -- Grain: One row per support ticket
    ticket_id           VARCHAR(50) PRIMARY KEY,
    
    -- Dimension Keys
    customer_key        INT,
    agent_key           INT,
    category_key        INT,
    priority_key        INT,
    
    -- Milestone Dates
    created_date_key        INT,
    assigned_date_key       INT,
    first_response_date_key INT,
    resolved_date_key       INT,
    closed_date_key         INT,
    
    -- Status
    current_status      VARCHAR(20),
    resolution_type     VARCHAR(30),
    
    -- Duration Metrics (for SLA tracking)
    time_to_first_response_hours INT,
    time_to_resolution_hours     INT,
    
    last_updated_timestamp TIMESTAMP
);
```

#### ETL Pattern

```python
# Update pattern (MERGE/UPSERT)
def load_accumulating_snapshot(business_date):
    """
    Accumulating snapshots use MERGE pattern to update existing rows
    """
    # Get updates for the day
    df_updates = extract_order_updates(business_date)
    
    # Calculate duration metrics
    df_updates = df_updates.withColumn(
        "order_to_ship_days",
        F.datediff("ship_date", "order_date")
    )
    
    # MERGE pattern (upsert)
    spark.sql("""
        MERGE INTO fact_order_pipeline t
        USING staging_order_updates s
        ON t.order_id = s.order_id
        WHEN MATCHED THEN UPDATE SET
            t.ship_date_key = s.ship_date_key,
            t.delivery_date_key = s.delivery_date_key,
            t.current_status = s.current_status,
            t.order_to_ship_days = s.order_to_ship_days,
            t.last_updated_timestamp = CURRENT_TIMESTAMP()
        WHEN NOT MATCHED THEN INSERT *
    """)
```

#### Query Patterns

```sql
-- SLA compliance analysis
SELECT 
    current_status,
    COUNT(*) as order_count,
    AVG(order_to_ship_days) as avg_fulfillment_time,
    SUM(CASE WHEN order_to_ship_days <= 2 THEN 1 ELSE 0 END) as within_sla,
    SUM(CASE WHEN order_to_ship_days > 2 THEN 1 ELSE 0 END) as breached_sla
FROM fact_order_pipeline
WHERE order_date_key >= 20240101
GROUP BY current_status;

-- Funnel analysis
SELECT 
    COUNT(*) as total_orders,
    SUM(CASE WHEN payment_date_key IS NOT NULL THEN 1 ELSE 0 END) as paid,
    SUM(CASE WHEN ship_date_key IS NOT NULL THEN 1 ELSE 0 END) as shipped,
    SUM(CASE WHEN delivery_date_key IS NOT NULL THEN 1 ELSE 0 END) as delivered
FROM fact_order_pipeline
WHERE order_date_key = 20240115;

-- Bottleneck identification
SELECT 
    AVG(DATEDIFF(payment_date, order_date)) as order_to_payment_days,
    AVG(DATEDIFF(ship_date, payment_date)) as payment_to_ship_days,
    AVG(DATEDIFF(delivery_date, ship_date)) as ship_to_delivery_days
FROM fact_order_pipeline
WHERE delivery_date_key IS NOT NULL
  AND order_date_key >= 20240101;
```

#### When to Use
- ✅ Process lifecycle tracking
- ✅ SLA monitoring and compliance
- ✅ Funnel/pipeline analysis
- ✅ Bottleneck identification
- ✅ Duration analysis between stages
- ✅ Workflow optimization

---

### Fact Table Type Comparison

| Aspect | Transaction | Periodic Snapshot | Accumulating Snapshot |
|--------|------------|-------------------|----------------------|
| **Grain** | Per event | Per entity per period | Per process instance |
| **Load Pattern** | Append-only | Overwrite/Append | Update (MERGE) |
| **Row Count** | Very high | Predictable (entities × periods) | Moderate (active processes) |
| **Additivity** | Fully additive | Semi-additive | Mixed |
| **Update Frequency** | Never | Per period | As milestones occur |
| **Date Columns** | 1 (event date) | 1 (snapshot date) | Multiple (milestone dates) |
| **Use Case** | Event capture | State tracking | Lifecycle monitoring |
| **Example** | Sales transactions | Daily balances | Order fulfillment |
| **Query Type** | Drill-down, aggregation | Trend analysis | Duration, SLA analysis |

---

### Advanced Fact Patterns

#### 1️⃣ Factless Fact Table

**Definition**: Stores only foreign keys with no numeric measures. Used to track events or coverage.

**Example: Student Attendance**:
```sql
CREATE TABLE fact_student_attendance (
    -- Grain: One row per student per class per day
    date_key        INT,
    student_key     INT,
    class_key       INT,
    instructor_key  INT,
    
    -- No measures! Just the fact that attendance occurred
    PRIMARY KEY (date_key, student_key, class_key)
);

-- Query: Which students attended which classes?
SELECT 
    s.student_name,
    c.class_name,
    d.full_date
FROM fact_student_attendance f
JOIN dim_student s ON f.student_key = s.student_key
JOIN dim_class c ON f.class_key = c.class_key
JOIN dim_date d ON f.date_key = d.date_key;

-- Query: Attendance rate
SELECT 
    s.student_name,
    COUNT(*) as classes_attended,
    COUNT(*) * 100.0 / (SELECT COUNT(*) FROM dim_class) as attendance_pct
FROM fact_student_attendance f
JOIN dim_student s ON f.student_key = s.student_key
GROUP BY s.student_name;
```

**Example: Product-Store Coverage**:
```sql
CREATE TABLE fact_product_store_assignment (
    -- Grain: One row per product per store (if assigned)
    product_key     INT,
    store_key       INT,
    effective_date_key INT,
    
    -- No measures - just tracks which products are sold in which stores
    PRIMARY KEY (product_key, store_key)
);
```

**Use Cases**:
- Event tracking (attendance, eligibility)
- Coverage tracking (product availability)
- Many-to-many relationships
- Promotion eligibility

---

#### 2️⃣ Aggregated Fact Table

**Definition**: Pre-summarized version of a transaction fact table for query performance.

> ⚠️ **Important**: This is NOT a new fact type—it's still a transaction fact, just at a higher grain.

**Example**:
```sql
-- Base transaction fact (atomic)
CREATE TABLE fact_sales (
    -- Grain: One row per order line item
    order_id        VARCHAR(20),
    line_number     INT,
    order_date_key  INT,
    product_key     INT,
    store_key       INT,
    quantity        DECIMAL(10,2),
    sales_amount    DECIMAL(15,2)
);
-- Rows: Billions

-- Aggregated fact (for performance)
CREATE TABLE fact_sales_daily_product (
    -- Grain: One row per product per day
    date_key        INT,
    product_key     INT,
    store_key       INT,
    
    -- Pre-aggregated measures
    total_quantity  DECIMAL(15,2),
    total_sales     DECIMAL(18,2),
    order_count     INT,
    
    PRIMARY KEY (date_key, product_key, store_key)
);
-- Rows: Millions (much smaller)

-- Populated via:
INSERT INTO fact_sales_daily_product
SELECT 
    order_date_key as date_key,
    product_key,
    store_key,
    SUM(quantity) as total_quantity,
    SUM(sales_amount) as total_sales,
    COUNT(DISTINCT order_id) as order_count
FROM fact_sales
WHERE order_date_key = 20240115
GROUP BY order_date_key, product_key, store_key;
```

**Characteristics**:

* Derived from transaction fact
* Fully additive (usually)
* Used for query optimization
* Does NOT represent a state snapshot

---

# 6️⃣ Real-World Example (Finance / Trading Context)

If you model a trading system:

### Transaction Fact

* One row per trade

### Periodic Snapshot

* Daily portfolio position

### Accumulating Snapshot

* Trade lifecycle (placed → executed → settled)

A mature warehouse often uses all three together.

---

# 7️⃣ Interview-Level Answer (Concise)

If asked:

> There are three main types of fact tables: transaction facts that store individual business events, periodic snapshots that store the state of something at regular intervals, and accumulating snapshots that track a process lifecycle across milestones. The choice depends on whether you’re capturing events, state over time, or process progression.

That’s a strong, structured answer.

---

If you want, next we can:

* Design all three types for a single business scenario
* Or discuss which type is best for ML pipelines
* Or go into factless fact tables in depth (commonly asked at senior interviews)


Good questions. Let’s separate them clearly.

---



# 2️⃣ Is Aggregated Fact Table Same as Periodic Snapshot?

No. They are fundamentally different.

Let’s compare carefully.

---

# 🔹 Periodic Snapshot Fact Table

### Purpose:

Capture the **state of an entity at regular time intervals**.

Grain:

> One row per entity per period.

Example:

Daily account balance:

| date | account_id | balance |
| ---- | ---------- | ------- |

Each row represents the balance **as of that day**.

Characteristics:

* Usually semi-additive
* Time-based grain
* Captures state, not flow

---
More detail:
Excellent — this is exactly the right confusion to clarify.

You are correct:

The **three core fact table types** are:

1. Transaction fact
2. Periodic snapshot fact
3. Accumulating snapshot fact

Now your question is:

> If I create “daily revenue per product”, is that a new type of fact?

The answer is:

> No. It is not a new fact type.
> It is a summarized (aggregated) transaction fact.

---

# 1️⃣ Fact Types Are Based on Behavior, Not Aggregation

Fact types are classified by **how the business process behaves over time**, not by whether data is aggregated.

So:

### Transaction fact

One row per event
(append-only)

### Periodic snapshot

One row per entity per fixed time period
(state-based)

### Accumulating snapshot

One row per lifecycle entity
(updated over time)

Aggregation does NOT create a new type.

---

# 2️⃣ What Is “Daily Revenue per Product”?

If you build:

| order_date | product_id | total_revenue |

Grain:

> One row per product per day

This is:

* Derived from transaction fact
* Still representing flow during the day
* Fully additive across time

So it is:

> A summarized transaction fact

Not a new fact type.

---

# 3️⃣ Why It Is NOT a Periodic Snapshot

Periodic snapshot represents state.

Example:
Inventory at end of day.

If no sale happens:
Stock still exists.

Revenue per day represents events that occurred during that day.

If no sale happens:
Revenue = 0.

That’s flow, not state.

---

# 4️⃣ Think of It Like This

Fact type classification axis:

How does the process evolve over time?

Aggregation axis:

What is the grain of storage?

These are separate.

You can have:

* Atomic transaction fact
* Summarized transaction fact
* Atomic snapshot fact
* Summarized snapshot fact

Still only 3 behavioral types.

---

# 5️⃣ Clean Classification Table

| Table                      | Type                     | Why               |
| -------------------------- | ------------------------ | ----------------- |
| fact_order_line            | Transaction              | One row per event |
| fact_daily_revenue         | Transaction (aggregated) | Still flow        |
| fact_account_balance_daily | Periodic snapshot        | State             |
| fact_order_pipeline        | Accumulating snapshot    | Lifecycle         |

---

# 6️⃣ Interview-Ready Answer

If interviewer asks:

“Is daily revenue per product a periodic snapshot?”

You say:

> No. It is an aggregated transaction fact. Periodic snapshots represent state at a point in time, whereas daily revenue represents flow accumulated from transactions during that period.

That’s precise and senior.

---

You’re asking the right kind of questions now — the confusion you had is exactly where conceptual maturity forms.

Now let me push you one level deeper:

If we store **cumulative revenue till date (running total)** per day:

| date | cumulative_revenue |

Is that:

A) Transaction fact
B) Periodic snapshot
C) Accumulating snapshot
D) Something else

Think carefully.


----

Grain discipline (deep)

Additivity & summarizability

Fact table types

Degenerate dimensions

Bridge tables

Factless facts

SCD impact on facts

Late arriving facts

Incremental loading strategies

Partitioning & performance

Conformed dimensions

Multi-fact constellation design

--- More to learn

Yes — and this is the right question to ask at your level.

You already understand the core definitions. What separates a senior data engineer from someone who just “knows dimensional modeling” is depth in fact table design and operational handling.

Here is a structured list of **fact-table–specific topics** you should master.

---

# 1️⃣ Grain Discipline (Deep Mastery)

You must be able to:

* Declare grain precisely
* Detect mixed-grain violations
* Separate header vs line facts
* Align grain with business process
* Identify logical primary key from grain
* Validate no duplicate grain rows

This is the foundation.

---

# 2️⃣ Additivity & Summarizability

You should deeply understand:

* Additive vs semi-additive vs non-additive
* Why summing snapshots breaks math
* Why ratios should not be stored directly
* How to preserve summarizability
* Distinct count challenges

Interviewers love summarization traps.

---

# 3️⃣ Fact Table Types (Beyond Definitions)

Know the practical implications of:

* Transaction facts (append-only)
* Periodic snapshots (state)
* Accumulating snapshots (lifecycle updates)
* Factless facts (event/coverage)

More importantly:
Know when to use each.

---

# 4️⃣ Fact Table Keys & Degenerate Dimensions

You should understand:

* Surrogate keys vs natural keys
* Why fact tables usually don’t enforce PK physically
* Degenerate dimensions (e.g., order_id in fact)
* Composite logical keys
* Handling high-cardinality keys

---

# 5️⃣ SCD Impact on Facts

This is advanced but essential:

* Type 2 surrogate key lookup
* Event time vs load time
* Late arriving facts
* Late arriving dimensions
* Retroactive correction
* Temporal consistency guarantees

This is where modeling meets real engineering.

---

# 6️⃣ Many-to-Many & Bridge Tables

You must know:

* Why direct joins duplicate measures
* Bridge grain definition
* Allocation factor logic
* How to test allocation correctness
* Performance impact of bridges

This is frequently asked in senior interviews.

---

# 7️⃣ Incremental Loading Strategies for Facts

From engineering perspective:

* Append-only strategy
* MERGE / Upsert strategy
* Partition overwrite strategy
* CDC integration
* Idempotency
* Backfill logic
* Late data windows

This is highly practical.

---

# 8️⃣ Partitioning & Performance

You should know:

* Why partition by business date
* Why not ingestion date
* Partition vs clustering
* File sizing strategy (Spark)
* Query pruning
* Aggregation before cross-fact joins

This is critical in BigQuery/Spark roles.

---

# 9️⃣ Multi-Fact Constellation Design

Enterprise modeling includes:

* Multiple fact tables
* Conformed dimensions
* Cross-fact analysis
* Aggregation alignment
* Avoiding fact-to-fact joins

You should be comfortable designing one.

---

# 🔟 Fact Table Testing & Data Quality

Very underrated topic.

You should know how to:

* Detect duplicate grain rows
* Reconcile source totals
* Validate allocation sums
* Detect missing surrogate keys
* Identify broken partitions
* Monitor incremental drift

Senior engineers think about validation, not just modeling.

---

# 1️⃣1️⃣ Advanced Topics (Optional but Strong Signal)

If you want to be very strong:

* Distinct count handling at scale
* Fact table archiving strategies
* Data vault vs dimensional fact mapping
* Feature engineering alignment with fact grain
* Streaming fact ingestion patterns
* Slowly changing fact concepts (rare but exists)
* Fact table compression & storage optimization

---

# 1️⃣2️⃣ What You Don’t Need to Overdo

Avoid:

* Memorizing textbook definitions only
* Over-focusing on ER diagrams
* Learning 50 theoretical modeling patterns without practical alignment

Focus on:
Grain + correctness + incremental + performance.

---

# 🎯 If You Master These, You’re Senior-Ready

You should be able to:

* Design a fact table from vague business requirements
* Spot modeling mistakes instantly
* Propose incremental strategy
* Handle SCD2 properly
* Design partition strategy
* Explain why an aggregation is wrong
* Architect multi-fact warehouse

That’s real-world readiness.

---

If you want, I can now:

* Create a structured 2–3 week learning plan for fact-table mastery
* Or simulate a 45-minute senior data modeling interview focused only on fact tables
* Or assess your readiness with scenario-based modeling questions

Your move.
