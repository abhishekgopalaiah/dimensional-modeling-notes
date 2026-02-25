Good. Factless fact tables are subtle — and very powerful when used correctly.

Let’s go structured and precise.

---

# 1️⃣ What Is a Factless Fact Table?

A **factless fact table** is:

> A fact table that contains no numeric measures, only foreign keys to dimensions.

It records that **an event occurred** or that **a condition is true**.

There is no amount, quantity, or revenue column.

---

# 2️⃣ Why Does It Exist?

Because sometimes the business question is:

* Did something happen?
* Which entities were related?
* Was something eligible?
* Was someone present?

Not:

* How much?
* How many dollars?

The fact itself is the occurrence.

---

# 3️⃣ Two Main Types of Factless Fact Tables

---

## 🔹 Type 1: Event Tracking Factless Table

Records that an event occurred.

### Example: Student Attendance

Grain:

> One row per student per class per day.

Table:

| date_key | student_key | class_key |
| -------- | ----------- | --------- |

No numeric measure.

To count attendance:

```sql
SELECT COUNT(*)
FROM fact_attendance
WHERE date_key = '2026-02-24';
```

The row itself represents attendance.

---

### Other Examples

* Website login events
* Product promotion participation
* Machine downtime events
* User app installs

---

## 🔹 Type 2: Coverage / Eligibility Table

Represents relationships or eligibility conditions.

Example:

Which products are available in which stores?

Grain:

> One row per product per store.

| product_key | store_key |
| ----------- | --------- |

No measures.

To answer:
“How many products are available in Store X?”

```sql
SELECT COUNT(*)
FROM fact_product_store
WHERE store_key = X;
```

---

# 4️⃣ Why Not Just Use a Dimension Table?

Because:

* Dimensions describe entities.
* Factless tables describe relationships or events between entities.

Example:

Customer dimension stores:

* name
* city
* segment

It does NOT store:

* Which promotions the customer participated in.

That relationship belongs in a fact (even if no numeric value exists).

---

# 5️⃣ Grain Discipline Still Applies

Even without measures, you must declare grain.

Example:

> One row per customer per promotion per campaign date.

If duplicates exist → grain broken.

Factless tables still require:

* Logical primary key
* Referential integrity
* Clear grain definition

---

# 6️⃣ Real-World Example (Retail)

### Promotion Participation

Grain:

> One row per order line per promotion.

Bridge-like factless table:

| order_line_id | promotion_key |

No allocation? Then it is purely existence tracking.

If you want revenue attribution → add allocation and treat it like a bridge.

---

# 7️⃣ Real-World Example (Banking)

### Account Compliance

Which accounts were flagged for compliance review?

Grain:

> One row per account per review event.

Table:

| account_key | review_date_key | review_type_key |

No measure.

But useful for:

* Counting flagged accounts
* Joining with balance facts
* Analyzing risk exposure

---

# 8️⃣ How BI Tools Use Factless Facts

To calculate:

* Count of students present
* Count of customers eligible
* Count of products active
* Count of accounts flagged

They simply use:

```sql
COUNT(*)
```

The row count is the metric.

---

# 9️⃣ Factless vs Bridge Table (Important Distinction)

They look similar but are conceptually different.

| Bridge Table                 | Factless Fact               |
| ---------------------------- | --------------------------- |
| Resolves many-to-many        | Records event or coverage   |
| Often includes allocation    | Usually no allocation       |
| Supports measure attribution | Supports counting existence |

Sometimes a bridge behaves like factless if no allocation exists.

---

# 🔟 When NOT to Use Factless Fact

If you already have:

Transaction fact that records the event.

Example:
Sales fact already records:

* product
* customer
* date

You don’t need another factless “purchase event” table.

Avoid redundancy.

---

# 1️⃣1️⃣ Interview-Ready Answer

If asked:

> A factless fact table is a fact table that contains no numeric measures and records the occurrence of events or relationships between dimensions. The grain must still be clearly defined, and analytics are typically performed using counts of rows.

Strong. Clear. Senior.

---

> Factless fact tables are not a separate fact table type. They can be transaction, periodic snapshot, or accumulating snapshot tables that simply do not contain numeric measures. The classification of fact tables is based on temporal behavior, whereas factless refers to the absence of measures.