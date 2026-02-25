# Data Warehouse Dimensions: Architecture Guide

> **Purpose**: Deep, architecture-level explanation of major dimension types for senior data engineering and architect interviews  
> **Scope**: Formal definitions, structural behavior, ETL mechanics, query patterns, performance implications, and real-world examples

---

## Table of Contents

1. [Conformed Dimension](#1-conformed-dimension)
2. [Role-Playing Dimension](#2-role-playing-dimension)
3. [Slowly Changing Dimension (SCD)](#3-slowly-changing-dimension-scd)
   - [Type 1 - Overwrite](#scd-type-1--overwrite)
   - [Type 2 - Full History](#scd-type-2--full-history)
   - [Type 3 - Limited History](#scd-type-3--limited-history)
4. [Junk Dimension](#4-junk-dimension)
5. [Degenerate Dimension](#5-degenerate-dimension)
6. [Mini-Dimension](#6-mini-dimension)
7. [Snowflaked Dimension](#7-snowflaked-dimension)
8. [Outrigger Dimension](#8-outrigger-dimension)
9. [Inferred / Late Arriving Dimension](#9-inferred--late-arriving-dimension)
10. [Bridge Dimension (Many-to-Many)](#🔟-bridge-dimension-many-to-many)
11. [Hierarchical Dimension](#1️⃣1️⃣-hierarchical-dimension)
12. [Shrunken Dimension](#1️⃣2️⃣-shrunken-dimension)
13. [Static Dimension](#1️⃣3️⃣-static-dimension)
14. [Architecture Summary](#architecture-perspective-summary)
15. [Real-World Example](#real-retail-example-full-stack)
16. [Advanced Topics](#are-topics-left-under-dimensions)

---

## 1️⃣ Conformed Dimension

### Definition
A dimension that is shared across multiple fact tables or data marts, maintaining identical meaning, keys, and attributes. It guarantees semantic consistency across the enterprise.

### Example
```sql
-- dim_date used by multiple fact tables
fact_sales      → dim_date (date_key)
fact_inventory  → dim_date (date_key)  
fact_returns    → dim_date (date_key)
fact_forecast   → dim_date (date_key)
```

All use the same `date_key` and fiscal calendar logic.

### Why It Matters
- **Finance** calculates revenue by month
- **Supply Chain** calculates inventory by month

Both must use identical fiscal calendar logic → **KPI mismatch** if inconsistent.

### Technical Requirements
- ✅ Same surrogate keys
- ✅ Same attribute definitions  
- ✅ Same grain
- ✅ Same business meaning

### Failure Example
Two teams build separate date tables:
- Team A: Fiscal year starts **April**
- Team B: Fiscal year starts **January**

**Result**: Revenue reports disagree.

### Performance Benefits
- 🔄 **BI tool consistency**
- ⚡ **Cache efficiency** 
- 🔄 **Query reuse**

---

## 2️⃣ Role-Playing Dimension

### Definition
One physical dimension reused multiple times in a fact table under different semantic roles.

### Example
```sql
-- Order fact with multiple date references
fact_orders:
| order_date_key | ship_date_key | delivery_date_key |
```

All reference the same `dim_date` table.

### Query Example
```sql
SELECT 
    od.month AS order_month,
    sd.month AS ship_month
FROM fact_orders f
JOIN dim_date od ON f.order_date_key = od.date_key
JOIN dim_date sd ON f.ship_date_key = sd.date_key;
```

### Common Use Cases
- **Employee**: sales_rep, approver, reviewer
- **Location**: origin, destination
- **Date**: order, ship, delivery

### ETL Consideration
- No dimension table duplication
- Just multiple foreign keys in fact table

---

## 3️⃣ Slowly Changing Dimension (SCD)

Handles attribute change over time.

### SCD Type 1 — Overwrite

#### Definition
Old value replaced by new value. No history maintained.

#### Example
```sql
-- Customer phone number change
Before: customer_key=1, phone=999
After:  customer_key=1, phone=888
```

#### Use Cases
- 📝 Data corrections
- 🔄 Non-analytical attributes  
- 🛡️ GDPR compliance cleanup

#### Risk
Historical reports reflect **new value** for past transactions.

### SCD Type 2 — Full History

#### Definition
Insert new row when tracked attribute changes.

#### Structural Pattern
```sql
| surrogate_key | natural_key | attr | effective_from | effective_to | is_current |
```

#### Detailed Example
```sql
-- Customer moves from NY to CA
sk | cust_id | state | eff_from   | eff_to     | current
---|---------|-------|------------|------------|--------
1  | C101    | NY    | 2023-01-01 | 2024-05-01 | false
2  | C101    | CA    | 2024-05-01 | NULL       | true
```

#### Fact Join Logic
```sql
JOIN dim_customer d
ON f.customer_id = d.customer_id
AND f.transaction_date BETWEEN d.effective_from AND d.effective_to
```

#### ETL Logic
1. 🔍 Detect change (hash comparison)
2. 📝 Close old record  
3. ➕ Insert new record

#### When Required
- 📊 Regulatory reporting
- 👥 Customer segmentation history
- ⚠️ Risk band tracking
- 💰 Pricing category history

#### Performance Considerations
Large SCD2 dimensions require:
- 📁 Clustering/partitioning
- 🔑 Index on natural_key
- 🎯 Control explosion

### SCD Type 3 — Limited History

#### Definition
Add previous value column to store limited history.

```sql
| customer_id | current_segment | previous_segment |
```

#### Use Cases
- 📈 Track last marketing campaign
- 💳 Previous pricing tier

#### Limitation
Only one (or limited) historical version.

---

## 4️⃣ Junk Dimension

### Definition
Combines multiple low-cardinality flags into one dimension.

### Example
```sql
-- Instead of multiple flag columns in fact:
is_promo, is_online, is_discounted, is_first_order

-- Create junk dimension:
| junk_key | is_promo | is_online | is_discounted | is_first_order |
```

Fact table stores only `junk_key`.

### Benefits
- 📏 **Reduces fact table width**
- 🗜️ **Improves compression**
- 🧹 **Cleaner modeling**

### Cardinality Check
If flags have: 2 × 2 × 2 × 2 = **16 possible combinations**  
Dimension has maximum 16 rows.

---

## 5️⃣ Degenerate Dimension

### Definition
Dimension attribute stored in fact table with no separate dimension.

### Example
```sql
-- Invoice number has no additional attributes
fact_sales:
| invoice_number | product_key | customer_key | revenue |
```

### When Appropriate
- 🚫 No descriptive attributes
- 🏷️ Pure transaction identifier
- ⚡ Avoid unnecessary join

---

## 6️⃣ Mini-Dimension

### Definition
Separate rapidly changing attributes into dedicated dimension.

### Problem It Solves
**SCD2 explosion** - if `churn_score` changes daily, customer dimension grows uncontrollably.

### Solution
```sql
-- Main dimension (stable)
dim_customer_main:
| customer_key | name | dob | signup_date |

-- Mini dimension (volatile)  
dim_customer_behavior:
| behavior_key | churn_score | behavior_segment | risk_band |
```

Fact table references both keys.

### Benefit
🎯 Historical accuracy without bloating core dimension.

---

## 7️⃣ Snowflaked Dimension

### Definition
Normalized dimension structure.

### Example
```sql
-- Instead of denormalized:
dim_product:
| product_key | product | category | department |

-- Normalized (snowflaked):
dim_product → dim_category → dim_department
```

### Pros & Cons
| Pros | Cons |
|------|------|
| ✅ Reduced redundancy | ❌ More joins |
| | ❌ Slower BI queries |
| | ❌ Harder SCD handling |

**Usage**: Applied sparingly.

---

## 8️⃣ Outrigger Dimension

### Definition
Dimension linked to another dimension.

### Example
```sql
-- Customer dimension references geography
fact_sales → dim_customer → dim_geography
```

### Risk
🔄 **SCD complexity increases** significantly.

---

## 9️⃣ Inferred / Late Arriving Dimension

### Definition
Create placeholder dimension when fact arrives before dimension.

### Example
Order received but customer not loaded yet:

```sql
-- Insert placeholder
INSERT INTO dim_customer VALUES ('C999', 'Unknown', ...);

-- Later update with real data
UPDATE dim_customer SET name = 'John Doe' WHERE customer_id = 'C999';
```

### Why Important
🔗 **Maintains referential integrity**.

---

## 🔟 Bridge Dimension (Many-to-Many)

### Problem
Order can have multiple sales reps, but fact can't store multiple keys.

### Solution
```sql
-- Bridge table
bridge_order_salesrep:
| order_key | salesrep_key | allocation_pct |
```

### Query Pattern
```sql
FROM fact_orders f
JOIN bridge_order_salesrep b ON f.order_key = b.order_key  
JOIN dim_salesrep s ON b.salesrep_key = s.salesrep_key
```

### Use Cases
- 🏷️ Multi-valued attributes
- 🏥 Healthcare diagnosis
- 💼 Skills per employee
- 📢 Multiple campaign attribution

---

## 1️⃣1️⃣ Hierarchical Dimension

### Definition
Represents hierarchy like: **Country → State → City → Store**

### Modeling Options
1. 📊 **Flattened** (preferred)
2. 🌳 **Parent-child**  
3. 🌉 **Bridge** for ragged hierarchies

---

## 1️⃣2️⃣ Shrunken Dimension

### Definition
Subset of dimension used for aggregate fact tables.

### Example
```sql
-- Full customer dimension (50 columns)
dim_customer_full: [name, dob, email, phone, address, ...]

-- Shrunken dimension for aggregates  
dim_customer_agg: [customer_key, region, segment]
```

---

## 1️⃣3️⃣ Static Dimension

### Definition
Never changes.

### Example
📅 **Calendar dimension** - no SCD logic required.

---

## Architecture Perspective Summary

Dimension design decisions impact:

| Aspect | Impact |
|--------|--------|
| 💾 **Storage size** | Table width and row count |
| ⚡ **Query speed** | Join complexity and indexing |
| 📊 **Historical correctness** | SCD type selection |
| 🔧 **ETL complexity** | Change detection and processing |
| 🛡️ **Regulatory compliance** | Audit trail requirements |

---

## Real Retail Example (Full Stack)

| Dimension | Type | Reason |
|-----------|------|--------|
| **Customer** | SCD2 | Regulatory reporting, segmentation history |
| **Product** | SCD2 | Category changes, pricing history |
| **Date** | Static, Role-playing | Calendar consistency, multiple date roles |
| **Promo flags** | Junk | Low-cardinality flags |
| **Invoice number** | Degenerate | Transaction identifier only |
| **Churn score** | Mini-dim | Rapidly changing behavior metrics |
| **Sales reps** | Bridge | Many-to-many relationship |

---

## Are Topics Left Under Dimensions?

**Yes** — for comprehensive preparation, you must cover:

1. 🔧 **Advanced SCD mechanics** - hybrid types, effective dating
2. 🌳 **Hierarchy modeling** - ragged hierarchies, recursive queries  
3. 🌉 **Many-to-many bridges** - allocation percentages, weighting
4. ⚙️ **Dimension ETL engineering** - change detection, performance
5. ⚡ **Performance tuning** - indexing, partitioning, clustering
6. ☁️ **Cloud/lakehouse implementation** - dimensional modeling in modern platforms
7. 🔗 **Dimension-fact interaction** - grain consistency, referential integrity
8. 🤖 **ML/feature-store relevance** - dimensions as features

---

*This guide serves as a comprehensive reference for data warehouse dimension design in enterprise environments.*