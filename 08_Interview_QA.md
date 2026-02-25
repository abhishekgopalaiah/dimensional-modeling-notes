# Data Warehouse Interview Questions & Answers

> **Target**: Senior Data Engineer / Data Architect roles  
> **Coverage**: 50 most common interview questions with model answers  
> **Format**: Question → Model Answer → Follow-up Topics

---

## Table of Contents

1. [Core Concepts (Q1-Q10)](#core-concepts)
2. [Dimensions (Q11-Q20)](#dimensions)
3. [Fact Tables (Q21-Q30)](#fact-tables)
4. [SCD & Temporal (Q31-Q40)](#scd--temporal)
5. [ETL & Performance (Q41-Q50)](#etl--performance)

---

## Core Concepts

### Q1: What is dimensional modeling and why is it used?

**Model Answer:**
> "Dimensional modeling is a data warehouse design technique that organizes data into fact tables (containing measurable business events) and dimension tables (containing descriptive context). It's used because it optimizes for analytical query performance, is intuitive for business users, supports flexible drill-down analysis, and enables consistent reporting across the enterprise through conformed dimensions."

**Follow-up Topics:** Star schema vs snowflake, Kimball vs Inmon

---

### Q2: Explain star schema vs snowflake schema.

**Model Answer:**
> "A star schema has denormalized dimension tables directly connected to fact tables, creating a star-like structure. A snowflake schema normalizes dimensions into multiple related tables. Star schemas are preferred for most use cases because they're simpler to query, perform better with fewer joins, and are easier for BI tools to navigate. Snowflake schemas reduce redundancy but add query complexity and join overhead. In modern columnar databases, storage savings from normalization are minimal."

**Follow-up Topics:** When to use snowflake, query performance implications

---

### Q3: What is grain and why is it critical?

**Model Answer:**
> "Grain defines the exact level of detail represented by each row in a fact table, stated as 'one row per X'. It's critical because it determines the logical primary key, constrains which dimensions and measures are valid, and ensures aggregation correctness. Grain must be declared before adding any columns. Violating grain discipline—like mixing order headers and line items—causes double counting and broken analytics."

**Follow-up Topics:** Grain validation, multi-grain scenarios

---

### Q4: What are conformed dimensions?

**Model Answer:**
> "Conformed dimensions are dimension tables shared across multiple fact tables with identical keys, attributes, and business meaning. For example, dim_date used by sales, inventory, and returns facts must have the same fiscal calendar logic. Conformed dimensions enable cross-process analysis and ensure consistent KPIs across the enterprise. Without them, different teams report different numbers for the same metric."

**Follow-up Topics:** Implementing conformed dimensions, governance challenges

---

### Q5: Explain surrogate keys vs natural keys.

**Model Answer:**
> "A natural key is the business identifier from source systems (like customer_id). A surrogate key is a system-generated integer used as the primary key in the warehouse. Surrogate keys are preferred because they: (1) handle SCD Type 2 history by creating new rows for the same business key, (2) insulate the warehouse from source system key changes, (3) improve join performance with small integer keys, and (4) simplify integration when multiple sources have conflicting natural keys."

**Follow-up Topics:** Surrogate key generation strategies, distributed systems

---

### Q6: What is a degenerate dimension?

**Model Answer:**
> "A degenerate dimension is a dimension attribute stored directly in the fact table without a separate dimension table. Common examples are invoice numbers or order IDs that have no additional descriptive attributes beyond the identifier itself. Storing them in the fact table avoids unnecessary joins while still enabling filtering and grouping."

**Follow-up Topics:** When to use vs avoid, impact on fact table width

---

### Q7: Explain additive, semi-additive, and non-additive facts.

**Model Answer:**
> "Additive facts like sales_amount can be summed across all dimensions including time. Semi-additive facts like account_balance represent state and can be summed across entities but not time—you'd use LAST_VALUE or AVG for time aggregations. Non-additive facts like profit margin percentage cannot be summed at all and must be recalculated from base measures. Best practice is to store base additive facts and compute ratios in the BI layer."

**Follow-up Topics:** Impact on ETL, query patterns, BI tool configuration

---

### Q8: What is a junk dimension?

**Model Answer:**
> "A junk dimension combines multiple low-cardinality flag columns into a single dimension table to reduce fact table width and improve compression. For example, instead of storing is_promo, is_online, is_discounted as separate boolean columns in the fact, you create a junk dimension with all possible combinations (2×2×2=8 rows) and store only the junk_key in the fact. This works well when flags have low cardinality and are frequently queried together."

**Follow-up Topics:** Cardinality limits, when not to use

---

### Q9: What is a factless fact table?

**Model Answer:**
> "A factless fact table contains only dimension foreign keys with no numeric measures. It records that an event occurred or a relationship exists. Examples include student attendance (date, student, class) or product availability (product, store). Analytics use COUNT(*) to measure occurrences. Factless facts are still classified as transaction, periodic snapshot, or accumulating snapshot based on temporal behavior—'factless' only means no numeric measures."

**Follow-up Topics:** Use cases, vs bridge tables

---

### Q10: How do you validate fact table grain?

**Model Answer:**
> "First, state the grain in one sentence: 'one row per X'. Then validate: (1) Define the logical primary key from the grain. (2) Query for duplicates on that key—should return zero rows. (3) Verify all measures are at the same detail level. (4) Check that SUM() produces correct results. (5) Confirm all dimension keys are valid for that grain. If any check fails, the grain is violated and requires redesign."

**Follow-up Topics:** Automated testing, data quality checks

---

## Dimensions

### Q11: Explain SCD Type 1, Type 2, and Type 3.

**Model Answer:**
> "Type 1 overwrites old values with new ones, losing history—used for corrections or non-analytical attributes. Type 2 inserts a new row with a new surrogate key when tracked attributes change, preserving full history through effective_from/effective_to dates and is_current flags. Type 3 stores limited history by adding previous_value columns. Type 2 is most common for regulatory reporting and historical analysis, but requires careful fact table joins using event dates to link to the correct historical dimension version."

**Follow-up Topics:** Type 2 implementation, performance implications

---

### Q12: How do facts join to SCD Type 2 dimensions?

**Model Answer:**
> "Facts must store the surrogate key of the dimension version valid at event time. During ETL, the dimension lookup joins on business key AND event date within the effective date range. For example: JOIN dim_customer ON fact.customer_id = dim.customer_id AND fact.order_date BETWEEN dim.effective_from AND dim.effective_to. This ensures historical accuracy—a sale from when the customer lived in Mumbai links to the Mumbai dimension row, not the current Delhi row."

**Follow-up Topics:** Late-arriving facts, retroactive corrections

---

### Q13: What is a mini-dimension?

**Model Answer:**
> "A mini-dimension separates rapidly changing attributes from a main dimension to prevent SCD Type 2 explosion. For example, if customer churn_score changes daily, storing it in the main customer dimension would create thousands of rows per customer. Instead, create dim_customer_behavior with churn_score and behavior_segment, and reference both customer_key and behavior_key in facts. This maintains historical accuracy without bloating the core dimension."

**Follow-up Topics:** When to use, design patterns

---

### Q14: What is a role-playing dimension?

**Model Answer:**
> "A role-playing dimension is one physical dimension table used multiple times in a fact table under different semantic roles. For example, dim_date used as order_date, ship_date, and delivery_date in fact_orders. The same dimension table is joined three times with different aliases. This avoids duplicating dimension data while supporting multiple date contexts in analysis."

**Follow-up Topics:** Query patterns, BI tool handling

---

### Q15: How do you handle late-arriving dimensions?

**Model Answer:**
> "When a fact arrives before its dimension, create a placeholder dimension row with unknown values and a special surrogate key (like -1). Load the fact referencing this placeholder. When the real dimension arrives, update the placeholder row with actual values or use Type 2 logic to insert a new row and update affected facts. This maintains referential integrity while handling timing mismatches between source systems."

**Follow-up Topics:** Inferred members, ETL patterns

---

### Q16: What is a bridge table and when is it used?

**Model Answer:**
> "A bridge table resolves many-to-many relationships between facts and dimensions while preserving fact grain. For example, if an order can have multiple sales reps, a bridge_order_salesrep table contains (order_key, salesrep_key, allocation_pct). The allocation percentage ensures measures aren't duplicated—revenue is split proportionally. Bridge tables are used for multi-valued attributes, skills per employee, diagnoses per patient, or campaign attribution."

**Follow-up Topics:** Allocation logic, performance considerations

---

### Q17: How do you prevent SCD Type 2 explosion?

**Model Answer:**
> "Strategies include: (1) Use mini-dimensions for rapidly changing attributes. (2) Apply Type 2 only to analytically important attributes—use Type 1 for others. (3) Implement change detection with hash comparisons to avoid unnecessary new rows. (4) Set business rules for meaningful change thresholds (e.g., price change >5%). (5) Archive old dimension versions after a retention period. (6) Monitor dimension growth and adjust tracked attributes based on actual query patterns."

**Follow-up Topics:** Monitoring, governance

---

### Q18: What is an outrigger dimension?

**Model Answer:**
> "An outrigger dimension is a dimension referenced by another dimension rather than directly by the fact. For example, dim_customer might reference dim_geography. While this creates a snowflake pattern, it's sometimes necessary for shared reference data. The risk is increased SCD complexity—if geography changes, you must decide whether to update the customer dimension or create a new customer row. Generally avoided unless there's strong justification."

**Follow-up Topics:** When justified, alternatives

---

### Q19: Explain hierarchical dimensions.

**Model Answer:**
> "Hierarchical dimensions represent relationships like Country → State → City → Store. The preferred approach is a flattened structure storing all hierarchy levels in one dimension table (store_id, city, state, country). This enables easy drill-down and avoids complex recursive queries. For ragged hierarchies (varying depth), use bridge tables or path enumeration. Parent-child tables are avoided in dimensional models due to query complexity."

**Follow-up Topics:** Ragged hierarchies, query patterns

---

### Q20: What is a shrunken dimension?

**Model Answer:**
> "A shrunken dimension is a subset of a larger dimension used with aggregate fact tables. For example, dim_customer_full has 50 columns, but dim_customer_aggregate for monthly summaries only needs (customer_key, region, segment). This reduces storage and improves query performance for pre-aggregated facts while maintaining key consistency with the full dimension."

**Follow-up Topics:** Aggregate fact design, conformance

---

## Fact Tables

### Q21: Explain the three main fact table types.

**Model Answer:**
> "Transaction facts store one row per business event (order line, trade, payment) with append-only loads and fully additive measures. Periodic snapshot facts store one row per entity per period (account per day, inventory per day) with overwrite loads and semi-additive measures representing state. Accumulating snapshot facts store one row per process instance (order lifecycle) updated as milestones occur, with multiple date columns for each stage. The type is determined by temporal behavior, not aggregation level."

**Follow-up Topics:** Choosing the right type, hybrid approaches

---

### Q22: When would you use a periodic snapshot vs transaction fact?

**Model Answer:**
> "Use transaction facts when you need atomic event detail for drill-down analysis and measures are flows (sales, trades). Use periodic snapshots when you need point-in-time state for trend analysis and measures are balances or levels (account balance, inventory). Often you need both: transaction facts for detailed analysis and periodic snapshots derived from transactions for efficient time-series queries and regulatory reporting."

**Follow-up Topics:** Deriving snapshots from transactions

---

### Q23: What is an accumulating snapshot and when is it used?

**Model Answer:**
> "An accumulating snapshot tracks a process lifecycle with one row per instance, updated as milestones occur. It has multiple date columns (order_date, ship_date, delivery_date) and duration metrics between stages. Used for SLA monitoring, funnel analysis, and bottleneck identification in processes like order fulfillment, loan applications, or support tickets. Load pattern is MERGE/UPSERT, not append-only."

**Follow-up Topics:** ETL patterns, vs transaction facts

---

### Q24: How do you handle late-arriving facts?

**Model Answer:**
> "Late-arriving facts must use event date, not load date, for dimension lookups to maintain historical accuracy. The ETL must: (1) Join dimensions using event_date BETWEEN effective_from AND effective_to. (2) Write to the correct business date partition, not current date. (3) Support partition backfills. (4) Use MERGE logic for idempotency. (5) Implement reconciliation checks to detect late data. Failure to handle this correctly causes historical corruption."

**Follow-up Topics:** Partition strategies, reconciliation

---

### Q25: What is the difference between a fact and an aggregate fact?

**Model Answer:**
> "An atomic fact table stores data at the lowest grain (order line item). An aggregate fact table is a pre-summarized version at higher grain (daily sales per product) for query performance. Aggregates are derived from atomic facts through scheduled ETL. The atomic fact remains the source of truth. Aggregates are still classified as transaction, snapshot, or accumulating based on temporal behavior—aggregation doesn't change the type."

**Follow-up Topics:** When to create aggregates, maintenance

---

### Q26: How do you design a multi-fact constellation schema?

**Model Answer:**
> "A constellation schema has multiple fact tables at different grains sharing conformed dimensions. Each fact represents a distinct business process with its own grain. Key principles: (1) Each fact has explicit grain. (2) Shared dimensions must be conformed (same keys, meaning, SCD logic). (3) Never join facts directly—aggregate to compatible grain first, then join through dimensions. (4) Use consistent dimension keys across all facts. This enables cross-process analysis while maintaining grain discipline."

**Follow-up Topics:** Conformed dimension management, query patterns

---

### Q27: What are the key considerations for fact table partitioning?

**Model Answer:**
> "Partition by business date column in the grain (order_date, snapshot_date), not ingestion date. This enables: (1) Efficient query pruning when filtering by date. (2) Correct placement of late-arriving facts. (3) Incremental load patterns (overwrite last N days). (4) Alignment with grain discipline. Use daily partitions for most cases. Combine with clustering on high-cardinality filter columns (customer_id, product_id). Avoid partitioning by high-cardinality dimensions."

**Follow-up Topics:** BigQuery vs Spark strategies, optimization

---

### Q28: How do you ensure fact table idempotency?

**Model Answer:**
> "Idempotency means running the same load multiple times produces the same result. Achieve this through: (1) MERGE logic using natural keys (order_id, transaction_id). (2) Partition overwrite for snapshot facts. (3) Deduplication logic before loading. (4) Batch tracking with load_id. (5) Testing reprocessing scenarios. For transaction facts, use MERGE ON natural_key WHEN NOT MATCHED THEN INSERT. This prevents duplicates during reruns or late data corrections."

**Follow-up Topics:** CDC patterns, testing strategies

---

### Q29: What is a coverage fact table?

**Model Answer:**
> "A coverage fact table (also called factless fact) records relationships or eligibility without numeric measures. Examples: product availability per store, promotion eligibility per customer, student enrollment per class. The grain is clearly defined (one row per product per store), and analytics use COUNT(*) to measure coverage. Despite having no measures, grain discipline and referential integrity still apply."

**Follow-up Topics:** Use cases, vs bridge tables

---

### Q30: How do you handle fact table updates vs inserts?

**Model Answer:**
> "Transaction facts are typically insert-only (append) since events are immutable. Periodic snapshots use partition overwrite or MERGE to replace state. Accumulating snapshots use MERGE to update existing rows as milestones occur. For corrections, use MERGE with natural keys. The pattern depends on fact type and business requirements. Always maintain audit columns (load_timestamp, batch_id) to track changes."

**Follow-up Topics:** Audit patterns, data quality

---

## SCD & Temporal

### Q31: How do you implement SCD Type 2 in Spark/BigQuery?

**Model Answer:**
> "Implementation steps: (1) Read current dimension and incoming changes. (2) Detect changes using hash comparison or column-by-column check. (3) For changed rows: close old record by setting effective_to = current_date and is_current = false. (4) Insert new record with new surrogate key, effective_from = current_date, effective_to = NULL, is_current = true. (5) For new records, insert with initial effective dates. Use MERGE statement in BigQuery or Delta Lake in Spark for ACID compliance."

**Follow-up Topics:** Hash-based change detection, performance optimization

---

### Q32: What is retroactive dimension correction?

**Model Answer:**
> "Retroactive correction occurs when a dimension change effective in the past is discovered late. For example, discovering in March that a customer's segment changed in February. This requires: (1) Correcting the dimension with proper effective dates. (2) Identifying affected facts. (3) Updating fact surrogate keys to reference the correct historical dimension version. (4) Reprocessing affected partitions. (5) Reconciling downstream aggregates. This is complex and expensive, so prevention through data quality checks is preferred."

**Follow-up Topics:** Impact analysis, prevention strategies

---

### Q33: How do you handle slowly changing facts?

**Model Answer:**
> "Facts can change due to corrections, adjustments, or late data. Handle through: (1) MERGE logic using natural keys to update existing rows. (2) Audit columns tracking original vs corrected values. (3) Separate adjustment facts if business needs to see both original and adjusted. (4) Effective dating if facts have temporal validity. (5) Idempotent loads to handle reprocessing. Unlike dimensions, fact updates are usually corrections rather than historical tracking."

**Follow-up Topics:** Audit patterns, reconciliation

---

### Q34: What is point-in-time join logic?

**Model Answer:**
> "Point-in-time joins ensure facts link to dimension versions valid at event time. The join condition includes: fact.business_key = dim.business_key AND fact.event_date BETWEEN dim.effective_from AND dim.effective_to. This is critical for SCD Type 2 dimensions. Without it, historical facts would incorrectly link to current dimension values, breaking historical accuracy. Performance optimization includes indexing effective dates and broadcasting small dimensions in Spark."

**Follow-up Topics:** Performance optimization, query patterns

---

### Q35: How do you test SCD Type 2 implementation?

**Model Answer:**
> "Test scenarios: (1) New dimension record—verify initial effective dates and is_current flag. (2) Attribute change—verify old row closed and new row inserted with new surrogate key. (3) No change—verify no new row created. (4) Multiple changes same day—verify only one new row. (5) Fact join correctness—verify facts link to correct historical version. (6) Effective date gaps—verify no overlaps or gaps. (7) Surrogate key uniqueness. Automate these tests in CI/CD."

**Follow-up Topics:** Data quality framework, automation

---

### Q36: What is the impact of SCD Type 2 on query performance?

**Model Answer:**
> "Type 2 increases dimension size (multiple rows per business key) and adds join complexity (must filter on effective dates). Mitigation strategies: (1) Index on business_key and effective dates. (2) Partition large dimensions by effective_from year. (3) Broadcast small dimensions in Spark. (4) Create current-only views for queries not needing history. (5) Use clustering in BigQuery. (6) Limit Type 2 to analytically important attributes. Despite overhead, Type 2 is essential for historical accuracy."

**Follow-up Topics:** Optimization techniques, when to avoid

---

### Q37: How do you handle dimension deletes?

**Model Answer:**
> "True deletes are rare in dimensional models. Options: (1) Soft delete—set is_deleted flag and effective_to date, keep row for referential integrity. (2) Keep dimension row even if entity deleted in source—facts still reference it. (3) For Type 2, close current row with effective_to = delete_date. (4) Never physically delete if facts reference it—breaks referential integrity. (5) Archive old versions after retention period if needed. Most 'deletes' are actually status changes (active → inactive)."

**Follow-up Topics:** Referential integrity, archival strategies

---

### Q38: What is a hybrid SCD approach?

**Model Answer:**
> "Hybrid SCD applies different strategies to different attributes in the same dimension. For example, customer dimension might use: Type 2 for segment and region (need history), Type 1 for email and phone (corrections), Type 3 for previous_segment (limited history). This balances historical accuracy with dimension size. Implementation tracks which attributes trigger new rows. Requires clear documentation of which attributes use which type."

**Follow-up Topics:** Design patterns, documentation

---

### Q39: How do you handle timezone issues in temporal logic?

**Model Answer:**
> "Best practices: (1) Store all timestamps in UTC in the warehouse. (2) Convert to business timezone only in presentation layer. (3) Use date_key (integer YYYYMMDD) for partitioning and joins, not timestamps. (4) For effective dates, use DATE type, not TIMESTAMP. (5) Document timezone assumptions clearly. (6) Handle daylight saving time transitions carefully. (7) For global businesses, store timezone in dimension if needed. Consistent timezone handling prevents temporal join errors."

**Follow-up Topics:** Global deployments, DST handling

---

### Q40: What is temporal consistency in dimensional models?

**Model Answer:**
> "Temporal consistency means all related data reflects the same point in time. For example, a fact from January 2024 must join to dimension versions valid in January 2024, not current versions. This requires: (1) Event date in facts. (2) Effective dates in dimensions. (3) Point-in-time join logic. (4) Consistent date handling across all tables. (5) Late data handling that maintains temporal correctness. Breaking temporal consistency causes historical reports to show incorrect values."

**Follow-up Topics:** Testing, data quality checks

---

## ETL & Performance

### Q41: Explain incremental loading strategies.

**Model Answer:**
> "Main strategies: (1) Append-only—for immutable transaction facts, load only new records using watermark. (2) MERGE/UPSERT—for facts with updates, use natural keys to insert new or update existing. (3) Partition overwrite—rebuild specific date partitions, efficient for snapshots. (4) CDC—stream changes from source, apply incrementally. (5) Rolling window—reprocess last N days to capture late data. Strategy depends on fact type, data volume, and lateness requirements. All must be idempotent."

**Follow-up Topics:** CDC implementation, performance tuning

---

### Q42: How do you handle very large fact tables (billions of rows)?

**Model Answer:**
> "Strategies: (1) Partition by business date for query pruning. (2) Cluster by frequently filtered columns. (3) Use columnar format (Parquet/ORC). (4) Create aggregate facts for common queries. (5) Implement partition archival for old data. (6) Use appropriate compression. (7) Optimize file sizes (100-500MB in Spark). (8) Broadcast small dimensions. (9) Incremental loads, not full refreshes. (10) Monitor query patterns and optimize accordingly. Consider tiered storage for historical data."

**Follow-up Topics:** Cost optimization, archival strategies

---

### Q43: What is the difference between clustering and partitioning?

**Model Answer:**
> "Partitioning physically divides data into separate files/segments by one column (typically date), enabling entire partition pruning. Clustering sorts data within partitions by one or more columns, enabling finer-grained pruning. Example: partition by order_date (daily), cluster by customer_id and product_id. Queries filtering on date skip entire partitions; filters on customer_id within a partition scan less data due to clustering. Use partitioning for time-based pruning, clustering for high-cardinality filters."

**Follow-up Topics:** BigQuery vs Spark differences, optimization

---

### Q44: How do you ensure data quality in fact tables?

**Model Answer:**
> "Implement checks: (1) Grain validation—no duplicates on logical primary key. (2) Referential integrity—all dimension keys exist. (3) Measure validation—no nulls in required facts, values in expected ranges. (4) Reconciliation—source totals match warehouse totals. (5) Partition completeness—all expected dates present. (6) Late data monitoring—track arrival delays. (7) Incremental drift—compare daily loads to full refresh. (8) Automated tests in CI/CD. (9) Data quality dashboards. (10) Alerts for anomalies."

**Follow-up Topics:** Testing frameworks, monitoring

---

### Q45: What is the role of staging tables in dimensional ETL?

**Model Answer:**
> "Staging tables provide: (1) Isolation—load and validate before affecting production. (2) Transformation workspace—perform complex logic without impacting source or target. (3) Restart capability—if load fails, restart from staging. (4) Audit trail—track what was loaded. (5) Data quality checks—validate before promoting. Pattern: source → staging → dimension/fact. Staging can be persistent or temporary. In cloud warehouses, staging is often in separate schema or dataset."

**Follow-up Topics:** Staging patterns, cloud-specific approaches

---

### Q46: How do you handle fact table reconciliation?

**Model Answer:**
> "Reconciliation ensures warehouse matches source. Process: (1) Compare source totals to warehouse totals by date/partition. (2) Track row counts, sum of key measures. (3) Identify discrepancies—missing data, duplicates, late arrivals. (4) Investigate root cause—ETL bugs, late data, source issues. (5) Implement automated reconciliation reports. (6) Set tolerance thresholds. (7) Alert on breaches. (8) Document resolution procedures. Run reconciliation after each load and periodically for historical data."

**Follow-up Topics:** Automation, root cause analysis

---

### Q47: What is the impact of compression on fact tables?

**Model Answer:**
> "Compression reduces storage and I/O, improving query performance and reducing costs. Columnar formats (Parquet, ORC) compress extremely well due to repeated values. Considerations: (1) Choose appropriate compression codec—Snappy for speed, Gzip for size. (2) Wider fact tables compress better (more repeated values). (3) Sorted/clustered data compresses better. (4) Compression is transparent to queries. (5) Trade-off between compression ratio and CPU. In BigQuery, compression is automatic. In Spark, configure at write time."

**Follow-up Topics:** Codec selection, performance trade-offs

---

### Q48: How do you design for query performance in dimensional models?

**Model Answer:**
> "Optimization strategies: (1) Denormalize dimensions (star not snowflake). (2) Partition facts by business date. (3) Cluster by frequently filtered columns. (4) Create aggregate facts for common queries. (5) Use appropriate data types (integers for keys). (6) Implement materialized views for complex joins. (7) Broadcast small dimensions in Spark. (8) Optimize join order (fact last). (9) Push predicates down. (10) Monitor query patterns and adjust design. Performance is a continuous process, not one-time."

**Follow-up Topics:** Query optimization, monitoring

---

### Q49: What is the role of metadata in dimensional models?

**Model Answer:**
> "Metadata provides: (1) Data lineage—source to target mappings. (2) Business definitions—what each measure means. (3) Grain documentation—what each row represents. (4) SCD strategy—which attributes use which type. (5) Refresh schedules—when data updates. (6) Data quality rules—validation logic. (7) Usage tracking—which tables/columns are queried. (8) Impact analysis—what breaks if schema changes. Store in data catalog or metadata repository. Essential for governance and self-service analytics."

**Follow-up Topics:** Data catalogs, governance

---

### Q50: How do you migrate from a legacy data warehouse to modern architecture?

**Model Answer:**
> "Migration approach: (1) Assess current state—document existing models, dependencies, usage. (2) Define target architecture—cloud platform, modeling approach. (3) Prioritize—migrate high-value, low-complexity first. (4) Implement dual-run—run old and new in parallel. (5) Migrate incrementally—table by table or subject area by subject area. (6) Validate—reconcile results between old and new. (7) Cutover—switch users to new platform. (8) Decommission—retire old system. (9) Optimize—tune new platform. (10) Document—capture lessons learned. Expect 12-24 months for enterprise migrations."

**Follow-up Topics:** Cloud migration, change management

---

## Interview Preparation Tips

### How to Use This Guide

1. **Read each question and formulate your own answer first**
2. **Compare with the model answer**
3. **Practice explaining out loud**
4. **Prepare follow-up examples from your experience**
5. **Focus on the "why" not just the "what"**

### Common Interview Patterns

- **Scenario-based**: "How would you design a warehouse for..."
- **Debugging**: "This query is slow, how would you optimize..."
- **Trade-offs**: "When would you use X vs Y..."
- **Experience**: "Tell me about a time you handled..."
- **Architecture**: "Design a solution for..."

### Red Flags to Avoid

- ❌ Saying "it depends" without explaining what it depends on
- ❌ Not asking clarifying questions
- ❌ Focusing only on tools, not concepts
- ❌ Not admitting when you don't know something
- ❌ Giving textbook answers without real-world context

### Green Flags to Show

- ✅ Asking about business requirements before designing
- ✅ Discussing trade-offs explicitly
- ✅ Mentioning data quality and testing
- ✅ Considering performance and cost
- ✅ Referencing real projects (without violating NDAs)
- ✅ Showing continuous learning

---

*Master these 50 questions and you'll be well-prepared for senior data engineering interviews. Remember: interviewers want to see how you think, not just what you know.*
