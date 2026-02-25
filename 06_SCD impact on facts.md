You mean **SCD impact on facts** (Slowly Changing Dimensions).
This is a senior-level topic. It’s where dimensional modeling meets temporal correctness.

Let’s go deep and structured.

---

# 1️⃣ First: What Is the Core Problem?

Dimensions change over time.

Example:

* Customer changes city
* Product changes category
* Account changes risk rating
* Trader changes desk

But fact tables record events at a specific point in time.

Question:

> When joining fact to dimension, which version of the dimension should be used?

That’s the SCD impact.

---

# 2️⃣ SCD Type 1 Impact (Overwrite)

Type 1:
Old value is overwritten.
No history maintained.

Example:
Customer moves from Mumbai → Delhi.
Old value replaced.

Impact on facts:

All historical sales will now show Delhi.

You lose historical truth.

This may be acceptable for:

* Correcting spelling errors
* Non-analytical attributes

But dangerous for:

* Region-based historical reporting

---

# 3️⃣ SCD Type 2 Impact (Most Important)

Type 2:
New row inserted with new surrogate key.
History preserved.
Good. Let’s slow this down and make it very concrete.

I’ll walk you through a full story with numbers, tables, and exactly what breaks if SCD2 is not handled correctly.

---

# Step 1: The Business Scenario

Customer C1 lives in Mumbai.

On **Jan 10, 2024**, they move to Delhi.

We want to preserve history:

* Sales before Jan 10 → count under Mumbai
* Sales after Jan 10 → count under Delhi

That’s why we use **SCD Type 2**.

---

# Step 2: Dimension Table (SCD Type 2)

dim_customer:

| customer_key | customer_id | city   | effective_from | effective_to |
| ------------ | ----------- | ------ | -------------- | ------------ |
| 101          | C1          | Mumbai | 2023-01-01     | 2024-01-10   |
| 205          | C1          | Delhi  | 2024-01-11     | NULL         |

Notice:

* Same customer_id
* Different surrogate keys
* Time ranges do not overlap

---

# Step 3: Two Orders

Order A:

* order_id = O1
* customer_id = C1
* order_date = 2023-06-01
* revenue = 1000

Order B:

* order_id = O2
* customer_id = C1
* order_date = 2024-02-01
* revenue = 2000

---

# Step 4: What the Fact Table SHOULD Look Like

fact_order:

| order_id | customer_key | order_date | revenue |
| -------- | ------------ | ---------- | ------- |
| O1       | 101          | 2023-06-01 | 1000    |
| O2       | 205          | 2024-02-01 | 2000    |

Why?

Because:

* O1 happened when customer was in Mumbai → key 101
* O2 happened when customer was in Delhi → key 205

Now reporting works:

Revenue by city:

Mumbai → 1000
Delhi → 2000

Correct.

---

# Step 5: What Happens If You Do It WRONG

Wrong logic:
Join fact to dimension using only customer_id, and take latest row.

Then fact becomes:

| order_id | customer_key | order_date | revenue |
| -------- | ------------ | ---------- | ------- |
| O1       | 205          | 2023-06-01 | 1000    |
| O2       | 205          | 2024-02-01 | 2000    |

Now both orders link to Delhi.

Revenue by city:

Mumbai → 0
Delhi → 3000

This is historically wrong.

That is the impact of mishandling SCD2.

---

# Step 6: How You Actually Handle It (ETL Logic)

When loading fact:

You do NOT join like this:

```sql
JOIN dim_customer d
ON s.customer_id = d.customer_id
```

You must join like this:

```sql
JOIN dim_customer d
ON s.customer_id = d.customer_id
AND s.order_date >= d.effective_from
AND (s.order_date <= d.effective_to OR d.effective_to IS NULL)
```

This ensures:
You get the dimension version valid at event time.

That’s it. That’s the core idea.

---

# Step 7: Late Arriving Fact (Important)

Now imagine:

Customer moved Jan 10.

Order from Jan 5 arrives late on Jan 20.

If your ETL uses:
load_date instead of order_date

You will assign Delhi key (205).

Wrong.

Correct approach:
Always use business event date in the lookup.

Event time > load time.

This is the most common production mistake.

---

# Step 8: Why Surrogate Key Is Mandatory

Why not store customer_id in fact?

Because if you do:

Fact:
| order_id | customer_id | order_date | revenue |

Then when you join to dim_customer,
you’ll get 2 rows (Mumbai + Delhi).

Ambiguous.

Surrogate key freezes the correct historical version.

That is why SCD2 requires surrogate keys.

---

# Step 9: Visual Understanding

Think of dimension as:

A timeline of customer states.

Fact row must connect to the correct point on that timeline.

Like this:

Time →
[ Mumbai ] --------- [ Delhi ]
↑ O1                 ↑ O2

Fact row attaches to the state at its time.

That’s SCD2 impact.

---

# Step 10: Why This Is So Important

If you don’t handle this correctly:

* Historical reports become wrong
* Revenue by region becomes wrong
* Performance by segment becomes wrong
* ML features get corrupted

Executives lose trust in data.

---

# Step 11: Final Clean Understanding

SCD2 impact on facts means:

Facts must store the surrogate key of the dimension version that was valid at the time of the event.

The join must include:

* business key
* event date within effective date range

Without this, history collapses.


---

# 4️⃣ How Facts Handle Type 2 Correctly

During ETL:

1. Lookup dimension row
2. Match business key
3. Filter by:
   event_date BETWEEN effective_from AND effective_to
4. Insert surrogate key into fact

This preserves historical truth.

This is critical.

---

# 5️⃣ Late Arriving Facts + SCD2 (Real-World Complexity)

Suppose:

* Customer moved city on Jan 10
* A Jan 5 order arrives late on Jan 15

If ETL incorrectly uses current dimension record,
it will assign wrong city.

Correct solution:
Dimension lookup must use order_date,
not load date.

This is where many pipelines fail.

---

# 6️⃣ SCD Type 3 Impact

Type 3:
Limited history stored in same row.

Example:
current_city, previous_city

Impact:
Only limited historical comparison possible.
Fact linking doesn’t change much.

Less common in serious warehouses.

---

# 7️⃣ SCD and Accumulating Snapshots

If fact row gets updated (e.g., order lifecycle),
and dimension changes mid-process:

Decision needed:

Do you:

* Freeze dimension at order creation?
* Or update to reflect latest dimension?

This depends on business semantics.

Example:
Salesperson changes region after order placed.
Should revenue belong to old or new region?

Architectural decision required.

---

# 8️⃣ SCD and Aggregation

If Type 2 is used correctly:

You can analyze:

* Revenue by historical city
* Revenue by current city
* Revenue before vs after segment change

If Type 1:
You lose historical segmentation capability.

This affects executive reporting.

---

# 9️⃣ SCD Impact on Feature Engineering

Suppose:

You’re building churn model.

Feature:
Revenue while customer was in premium segment.

If you don’t use historical surrogate keys:
You mislabel past transactions.

Model learns wrong patterns.

Temporal correctness matters for ML.

---

# 🔟 Common Mistakes

❌ Joining fact to dimension on business key only
❌ Ignoring effective dates
❌ Updating fact rows when dimension changes
❌ Using current dimension snapshot for all historical facts
❌ Not handling late-arriving facts

---

# 1️⃣1️⃣ Partitioning & Engineering Impact

When using SCD2:

* Fact table stores surrogate keys
* Dimension table stores history
* Joins become heavier
* Partitioning often based on fact event date
* Dimension typically smaller → broadcast join in Spark

Correct physical design matters.

---

# 1️⃣2️⃣ Interview-Ready Answer

If asked:

> SCDs impact fact tables because facts must link to the correct historical version of a dimension. In Type 2 dimensions, the fact table stores the surrogate key corresponding to the dimension state at the time of the event. Incorrect handling leads to loss of historical accuracy and aggregation errors.

That’s senior-level clarity.

---

# 1️⃣3️⃣ The Core Principle

Facts represent events at time T.

Dimensions represent descriptive context.

SCD2 ensures:
Context at time T is preserved.

Without it:
Historical truth collapses.

---

If you want, next we can:

* Walk through a complete SCD2 ETL example in SQL
* Or simulate a tricky late-arriving fact scenario
* Or discuss when NOT to use SCD2 (very important strategic decision)
