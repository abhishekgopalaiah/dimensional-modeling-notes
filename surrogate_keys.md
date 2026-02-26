1️⃣ What Is a Surrogate Key?

A surrogate key (SK) is a system-generated, meaningless, unique identifier used as the primary key of a dimension table.

It has:
	•	No business meaning
	•	No dependency on source system
	•	Typically integer / bigint
	•	Generated inside warehouse

Example:

Customer Dimension

| customer_sk	| customer_id (BK)	| name	| city	| effective_from | effective_to
| 101	| C1001	| John	| NY	| 2024-01-01	| 2024-06-01
| 102	| C1001	| John	| LA	| 2024-06-01	| 9999-12-31

	•	customer_id = business key (from source)
	•	customer_sk = surrogate key (warehouse key)

Fact table will reference customer_sk, not customer_id.

⸻

2️⃣ Why Surrogate Keys Are Required

🔹 Reason 1: Handle SCD Type 2

If you use business key:

Fact → joins to multiple dimension rows
Ambiguity happens.

Surrogate key guarantees:
Fact joins to correct historical version.

⸻

🔹 Reason 2: Source System Changes

Business keys can:
	•	Change format
	•	Be reused
	•	Merge
	•	Be null
	•	Be composite

Warehouse needs stable key.

⸻

🔹 Reason 3: Performance
	•	Integer joins are faster
	•	Smaller storage
	•	Better indexing
	•	Better partition pruning

⸻

🔹 Reason 4: Multi-Source Integration

Two systems may have:
	•	Same customer_id for different customers
	•	Different formats

Surrogate key creates unified identity.

⸻

3️⃣ Interview Questions (With Answers)

⸻

Q1: Why not use business key as primary key?

Answer:
Because business keys:
	•	Change
	•	May be reused
	•	Cannot support SCD Type 2 cleanly
	•	May be composite
	•	May collide across systems

Surrogate key provides warehouse stability and historical tracking.

⸻

Q2: How does surrogate key help in SCD Type 2?

Example:

Customer moves from NY → LA.

If using business key:

Fact:

order_id	customer_id


Join produces two rows because two versions exist.

With surrogate key:

Fact:
| order_id | customer_sk |

Now it points to exact historical version.

This ensures historical correctness.

⸻

Q3: What happens if you don’t use surrogate keys in Type 2 dimension?

You get:
	•	Duplicate joins
	•	Wrong aggregations
	•	Historical corruption
	•	Impossible back-dating fixes

Senior-level red flag if someone says SK not needed for Type 2.

⸻

Q4: When might surrogate key NOT be required?
	•	Type 1 dimension only
	•	Static lookup tables
	•	Small reference tables
	•	Date dimension (sometimes optional but still recommended)

But in enterprise DWH → always recommended.

⸻

Q5: What is surrogate key generation strategy?

Common strategies:
	•	Auto increment sequence
	•	UUID (not ideal for performance)
	•	Spark monotonically_increasing_id (not ideal for deterministic loads)
	•	Hash-based keys (rare in star schema)

Best practice:
Use sequence or identity column in warehouse.

⸻

Q6: Can fact table have surrogate key?

Yes.

But usually:
Fact table primary key = composite (dim_sk + timestamp)

Some teams add fact_sk for:
	•	Deduplication
	•	CDC tracking
	•	Easier updates

Not mandatory in dimensional modeling.

⸻

4️⃣ Advanced / Scenario-Based Questions

These are what differentiate senior engineers.

⸻

Q7: How do you handle late arriving dimension with surrogate key?

Scenario:
Fact arrives first.
Customer dimension not yet loaded.

Solution:
	•	Assign “unknown” surrogate key (-1)
	•	Later update fact row when dimension arrives
OR
	•	Stage fact until dimension available

Important:
Never use business key directly in fact.

⸻

Q8: What if business key changes in source?

Example:
Customer ID changes from C1001 → C9001.

With surrogate key:
You update dimension row.
Fact table unaffected.

If using business key:
All historical facts break.

⸻

Q9: How do you handle surrogate key during reprocessing?

If full reload:
	•	Preserve surrogate key mapping
	•	Do NOT regenerate blindly
	•	Use business key + effective date to match

Otherwise historical joins break.

This is a common production mistake.

⸻

Q10: What are pitfalls of surrogate keys?
	•	Duplicate business keys if dedup logic weak
	•	Sequence gap issues (not serious)
	•	Need strong natural key constraints
	•	Reprocessing challenges

⸻

5️⃣ Very Important Senior-Level Concept

Surrogate Key ≠ Business Identity

Warehouse identity and business identity must be decoupled.

Business key = logical identity
Surrogate key = physical identity

This separation gives:
	•	Historical stability
	•	Integration flexibility
	•	Architecture resilience

⸻

6️⃣ Real Interview Deep Question

“Explain how surrogate keys impact fact table design.”

Strong answer:
	•	Fact table stores surrogate keys of dimensions.
	•	Ensures time-consistent joins.
	•	Avoids ambiguity in SCD Type 2.
	•	Improves join performance.
	•	Protects against business key changes.

Weak answer:

“It improves performance.”

Too shallow.

⸻

7️⃣ Rapid Fire Interview Drill

You should be able to answer these in under 30 seconds:
	•	Why surrogate key?
	•	What breaks without it?
	•	How does it help SCD2?
	•	Can fact have surrogate key?
	•	What about late arriving dimension?
	•	What if business key changes?
	•	How do you generate it in Spark?

⸻

8️⃣ Spark-Specific Interview Angle (Since You Know Spark)

They may ask:

“How do you generate surrogate keys in Spark?”

Answer options:
	•	Use window + row_number over deterministic ordering
	•	Use Delta identity column
	•	Maintain sequence table
	•	Avoid monotonically_increasing_id for production deterministic pipelines


	•	Or real production failure cases involving surrogate keys

Your choice.
