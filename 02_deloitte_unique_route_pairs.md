# Deloitte SQL Interview Question — Unique Route Pairs

## Problem

Given a table of routes with `Start_Location`, `End_Location`, and `Distance`, some routes are recorded twice — once in each direction (e.g., Delhi→Pune and Pune→Delhi). Write SQL to return only **unique combinations** of the two locations, regardless of direction.

**Input Table**

| Start_Location | End_Location | Distance |
|---|---|---|
| Delhi | Pune | 1400 |
| Pune | Delhi | 1400 |
| Bangalore | Chennai | 350 |
| Mumbai | Ahmedabad | 500 |
| Chennai | Bangalore | 350 |
| Patna | Ranchi | 300 |

**Expected Output**

| Start_Location | End_Location | Distance |
|---|---|---|
| Delhi | Pune | 1400 |
| Bangalore | Chennai | 350 |
| Mumbai | Ahmedabad | 500 |
| Patna | Ranchi | 300 |

Delhi↔Pune and Bangalore↔Chennai are reversed duplicates of the same route. Mumbai→Ahmedabad and Patna→Ranchi have no reverse — they're standalone.

---

## Core Idea (common to all three approaches)

Delhi→Pune and Pune→Delhi are the *same route* if you ignore direction. To make SQL treat them as identical, we need a way to describe the pair of cities that doesn't depend on which one is Start and which is End — an **order-independent fingerprint**.

`LEAST(Start_Location, End_Location)` and `GREATEST(Start_Location, End_Location)` give us exactly that: for both Delhi/Pune rows, `LEAST = Delhi` and `GREATEST = Pune`, no matter which order they appear in.

Once rows share the same fingerprint, we need a way to **collapse them into one row**. That's where the three approaches differ.

---

## Approach 1: CTE + LEAST/GREATEST + GROUP BY

**Logic:** Compute the fingerprint columns first (in a CTE), then group by them and pull out one representative value per group using `MIN`/`MAX`.

```sql
WITH fingerprint AS (
    SELECT 
        Start_Location,
        End_Location,
        Distance,
        LEAST(Start_Location, End_Location) AS sorted_start,
        GREATEST(Start_Location, End_Location) AS sorted_end
    FROM location_distances
)
SELECT 
    MIN(Start_Location) AS Start_Location,
    MAX(End_Location) AS End_Location,
    MIN(Distance) AS Distance
FROM fingerprint
GROUP BY sorted_start, sorted_end;
```

**Why it works:**
- `GROUP BY sorted_start, sorted_end` collapses Delhi/Pune and Pune/Delhi into a single group (since both have `sorted_start = Delhi`, `sorted_end = Pune`).
- `MIN`/`MAX` just grab *a* representative value from the group — since Distance is identical either way, it doesn't matter which one is picked.

---

## Approach 2: ROW_NUMBER() + PARTITION BY + QUALIFY

**Logic:** Instead of collapsing rows via grouping, tag every row with a number based on its position within its fingerprint group, then keep only the first-tagged row per group.

```sql
SELECT
    Start_Location,
    End_Location,
    Distance
FROM location_distances
QUALIFY ROW_NUMBER() OVER (
    PARTITION BY LEAST(Start_Location, End_Location), GREATEST(Start_Location, End_Location)
    ORDER BY Start_Location
) = 1;
```

**Why it works:**
- `PARTITION BY` groups rows the same way `GROUP BY` did above, but `ROW_NUMBER()` numbers rows *within* each partition (1, 2, 3...) instead of collapsing them.
- `QUALIFY` is Snowflake's version of `WHERE`, but for window functions — it filters *after* the window function is calculated, which a normal `WHERE` clause can't do.
- `ORDER BY` inside `OVER()` is mandatory syntax for `ROW_NUMBER()`. It decides *which* row in a tied pair gets `rn = 1` — in this problem it doesn't change the final output (Distance is the same either direction), but it would matter if the two directions had different data.

---

## Approach 3: Self-Join (LEFT JOIN)

**Logic:** Join the table to itself to find each row's "reverse" (if one exists), then keep only one direction of each pair — plus any row that has no reverse at all.

```sql
SELECT a.Start_Location, a.End_Location, a.Distance 
FROM location_distances a
LEFT JOIN location_distances b
    ON a.Start_Location = b.End_Location 
   AND b.Start_Location = a.End_Location
WHERE a.Start_Location < a.End_Location 
   OR b.Start_Location IS NULL;
```

**Why it works, step by step:**
1. The `ON` condition matches each row `a` to its reverse `b` (e.g., Delhi→Pune matches to Pune→Delhi).
2. Using `LEFT JOIN` (not `INNER JOIN`) is critical — it keeps rows from `a` even when no reverse match exists (like Mumbai→Ahmedabad), instead of silently dropping them.
3. After the join, duplicate pairs appear twice (once as `a`=Delhi→Pune, once as `a`=Pune→Delhi). The `WHERE` clause picks one direction:
   - `a.Start_Location < a.End_Location` — keeps the alphabetically "forward" direction of a duplicate pair (Delhi < Pune → keeps Delhi→Pune, drops Pune→Delhi).
   - `OR b.Start_Location IS NULL` — this rescues standalone rows that would otherwise get wrongly filtered out by the first condition (e.g., Mumbai→Ahmedabad fails `Mumbai < Ahmedabad`, but since it has no reverse, `b` is NULL, so it's kept anyway).
4. `IS NULL` (not `= NULL`) is required in SQL to test for NULL, since `NULL = NULL` evaluates to unknown, never true.

---

## Common Thread

All three approaches solve the same underlying problem: **treat (A, B) and (B, A) as the same, then keep exactly one representative.**
- Approach 1 collapses via `GROUP BY`.
- Approach 2 tags via `ROW_NUMBER()` and filters via `QUALIFY`.
- Approach 3 compares rows directly via a self-join and filters via `WHERE`.

This pattern reappears anytime you're deduplicating **unordered pairs** — symmetric flight/route tables, undirected graph edges, mutual friend pairs, etc.
