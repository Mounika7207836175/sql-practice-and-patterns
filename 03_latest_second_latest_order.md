# SQL Scenario Interview Question — Latest & Second Latest Order Amount

## Problem

Given an `orders` table with `order_id`, `customer_id`, `order_date`, `order_amount`, find each customer's **latest** and **second latest** order amount — based on `order_date`, not on the order amount's size.

**Input Table**

| order_id | customer_id | order_date | order_amount |
|---|---|---|---|
| 1 | 101 | 2024-01-10 | 150.00 |
| 2 | 101 | 2024-02-15 | 200.00 |
| 3 | 101 | 2024-03-20 | 180.00 |
| 4 | 102 | 2024-01-12 | 200.00 |
| 5 | 102 | 2024-02-25 | 250.00 |
| 6 | 102 | 2024-03-10 | 320.00 |
| 7 | 103 | 2024-01-25 | 400.00 |
| 8 | 103 | 2024-02-15 | 420.00 |

**Expected Output**

| customer_id | latestorder | secondhighest |
|---|---|---|
| 101 | 180.00 | 200.00 |
| 102 | 320.00 | 250.00 |
| 103 | 420.00 | 400.00 |

**Key clarification:** "latest" is decided by `order_date` (most recent), not by the size of `order_amount`. For customer 101, the most recent order (2024-03-20) has amount 180 — even though 200 is a bigger number, it's the *older* of the two top orders.

---

## Core Pattern (used across all four approaches)

The general shape of this problem is: **rank/identify rows per customer → spread the ranked rows into separate columns using `MAX(CASE WHEN ... THEN value END)` + `GROUP BY customer_id`.**

Once you have two rows per customer tagged 0/1 or 1/2 (however you produced that tag), the exact same wrap-up step applies every time:

```sql
SELECT customer_id,
       MAX(CASE WHEN <tag indicates "latest"> THEN order_amount END) AS latestorder,
       MAX(CASE WHEN <tag indicates "second latest"> THEN order_amount END) AS secondhighest
FROM <ranked source>
GROUP BY customer_id;
```

Why this works: `CASE WHEN` produces the real amount for the row matching that tag, and NULL for the other row. `MAX()` ignores NULLs and just returns the one real value sitting in the group — effectively "picking" the right value out of two rows and squashing them into one.

**GROUP BY rule to remember:** every column in `SELECT` must either be listed in `GROUP BY` or wrapped in an aggregate (`MAX`, `COUNT`, etc.) — this is why `order_amount` and the tag column (`row_num` / `counting`) don't appear directly in the final SELECT once they've done their job upstream.

---

## Approach 1: CTE + ROW_NUMBER() + CASE + MAX + GROUP BY

**Logic:** Rank each customer's orders by date (most recent first), keep the top 2, then pivot them into two columns.

```sql
WITH CTE AS (
    SELECT customer_id, 
           ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY order_date DESC) AS row_num, 
           order_date, order_amount 
    FROM orders
    QUALIFY row_num <= 2
)
SELECT customer_id,
       MAX(CASE WHEN row_num = 1 THEN order_amount END) AS latestorder,
       MAX(CASE WHEN row_num = 2 THEN order_amount END) AS secondhighest
FROM CTE
GROUP BY customer_id;
```

- `ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY order_date DESC)` numbers each customer's orders 1 (most recent) upward.
- `QUALIFY row_num <= 2` keeps only the top 2 rows per customer (Snowflake's window-function equivalent of `WHERE`).
- The final `CASE + MAX + GROUP BY` spreads `row_num = 1/2` into two columns.

---

## Approach 2: ROW_NUMBER() + LEAD() (no row-collapsing needed)

**Logic:** Instead of producing two separate rows and then merging them, use `LEAD()` to pull the *next* row's amount directly onto the current row — so latest and second-latest sit together from the start.

```sql
SELECT customer_id, order_date, order_amount AS first,
       LEAD(order_amount) OVER (PARTITION BY customer_id ORDER BY order_date DESC) AS second
FROM orders
QUALIFY ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY order_date DESC) = 1
ORDER BY customer_id;
```

- Rows are ordered `order_date DESC` per customer, so row 1 = latest order.
- `LEAD(order_amount, 1)` looks one row *forward* (down) within that ordering — from the latest row, that's the second latest order's amount, landing in the same row.
- `QUALIFY ROW_NUMBER() ... = 1` keeps only the "latest" row per customer (which now already carries the second-latest value via `LEAD`), since without it you'd get one row per original order, each showing its own LEAD value.
- **Note:** without an explicit final `ORDER BY`, SQL does not guarantee row order in the output — always add `ORDER BY` if the output order matters, since window function `ORDER BY` only affects ranking, not final row order.

---

## Approach 3: Self-Join + COUNT + HAVING + CASE + MAX + GROUP BY

**Logic:** Instead of ranking, count — for each order, how many *other* orders for the same customer have a later date. 0 later orders = latest; 1 later order = second latest.

```sql
WITH ranked AS (
    SELECT a.customer_id, a.order_date, a.order_amount,
           COUNT(b.order_date) AS later_orders_count
    FROM orders a
    LEFT JOIN orders b
      ON a.customer_id = b.customer_id
     AND a.order_date < b.order_date
    GROUP BY a.customer_id, a.order_date, a.order_amount
    HAVING COUNT(b.order_date) <= 1
)
SELECT customer_id,
       MAX(CASE WHEN later_orders_count = 0 THEN order_amount END) AS latestorder,
       MAX(CASE WHEN later_orders_count = 1 THEN order_amount END) AS secondhighest
FROM ranked
GROUP BY customer_id;
```

**Key details and pitfalls:**
- `LEFT JOIN` (not `INNER JOIN`) is essential — it preserves rows even when no "later" match exists (e.g., a customer's most recent order has zero later orders, but must still appear in the result).
- The join condition `a.customer_id = b.customer_id AND a.order_date < b.order_date` finds, for each row `a`, every row `b` from the same customer with a strictly later date.
- **Must use `COUNT(b.order_date)`, not `COUNT(a.order_date)`.** `a.order_date` is never NULL (it's the "current" row), so counting it would always return at least 1, even when there are zero actual matches — this was a real bug hit while building this approach. `b.order_date` is NULL exactly when no match was found (thanks to the LEFT JOIN), so `COUNT(b.order_date)` correctly ignores those phantom rows and returns 0.
- `HAVING`, not `WHERE`, is required to filter on `COUNT(...)` — `WHERE` runs before aggregation happens and can't see the aggregate's result; `HAVING` runs after `GROUP BY`/aggregates are computed.
- The final `CASE + MAX + GROUP BY` step is identical in shape to Approach 1 — just swapping `row_num = 1/2` for `later_orders_count = 0/1`.
- This can also be written using a subquery in the `FROM` clause instead of a named CTE — functionally identical, just without the `WITH` name.

---

## Common Thread

Every approach reduces to the same two-stage idea:
1. **Identify or rank** each customer's orders by recency (via `ROW_NUMBER()`, `LEAD()`, or a self-join `COUNT`).
2. **Spread the result into columns** using `MAX(CASE WHEN <rank/tag> THEN value END)` + `GROUP BY customer_id` (or, with `LEAD()`, skip this step entirely since the values already sit on one row).

This "rank-then-pivot" pattern is one of the most common shapes in SQL interviews — anytime you see "find the Nth most recent / highest / lowest per group," this is the toolkit to reach for.
