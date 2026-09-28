# Fill Missing Category Values (Carry Forward Last Non-NULL)

## Problem
Table `brands(id, category, brand_name)`. Some `category` values are NULL. Fill each NULL with the nearest **preceding** non-NULL category (ordered by `id`).

| id | category | brand_name |
|----|----------|------------|
| 1  | Chocolates | 5-Star |
| 2  | NULL | Dairy Milk |
| 3  | NULL | Perk |
| 4  | NULL | Eclair |
| 5  | Snacks | Lays |
| 6  | NULL | Kurkure |
| 7  | NULL | Bingo |

Expected: ids 1-4 -> Chocolates, ids 5-7 -> Snacks.

**Pattern name:** Gaps and Islands (fill-forward).

**Prerequisite:** an ordering column (`id`). Without one, "preceding" has no meaning in SQL.

---

## Approach 1: Running COUNT tally + MAX (verified)

```sql
SELECT
    id,
    brand_name,
    MAX(category) OVER (PARTITION BY tally) AS category
FROM (
    SELECT
        id,
        category,
        brand_name,
        COUNT(category) OVER (ORDER BY id) AS tally
    FROM brands
) t
ORDER BY id;
```

**How it works**
- `COUNT(category)` ignores NULLs, so the running count only increases on rows with a real category.
- Each real value and the NULLs after it share the same tally, forming a group (an "island").
- `MAX(category)` per tally group picks the one real value, ignoring NULLs.

| id | category | tally |
|----|----------|-------|
| 1 | Chocolates | 1 |
| 2 | NULL | 1 |
| 3 | NULL | 1 |
| 4 | NULL | 1 |
| 5 | Snacks | 2 |
| 6 | NULL | 2 |
| 7 | NULL | 2 |

---

## Approach 2: LAST_VALUE with IGNORE NULLS (verified)

```sql
SELECT
    id,
    LAST_VALUE(category) IGNORE NULLS OVER (
        ORDER BY id
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS category,
    brand_name
FROM brands;
```

**How it works**
- The frame runs from the first row to the current row.
- `IGNORE NULLS` skips NULLs, so the last non-NULL value in the frame is the nearest preceding real category.
- Shortest approach, but relies on Snowflake's `IGNORE NULLS` support.

---

## Approach 3: Correlated subquery (verified)

```sql
SELECT
    b1.id,
    b1.brand_name,
    (
        SELECT b2.category
        FROM brands b2
        WHERE b2.id <= b1.id
          AND b2.category IS NOT NULL
        ORDER BY b2.id DESC
        LIMIT 1
    ) AS category
FROM brands b1
ORDER BY b1.id;
```

**How it works**
- The subquery sits in the SELECT list and re-runs once per `b1` row.
- `b2.id <= b1.id AND b2.category IS NOT NULL` gathers all real values at or before the current row.
- `ORDER BY b2.id DESC LIMIT 1` keeps the closest one.

**Common mistake (caught in session):** writing this as a `JOIN` with `ORDER BY ... LIMIT 1` at the end. That `LIMIT 1` applies to the whole result, not per row, so you get 1 row instead of 7. A correlated subquery scopes its own `ORDER BY`/`LIMIT` to each outer row.

---

## Approach 4: Recursive CTE (concept only, not completed)

Anchor: `SELECT id, brand_name, category FROM brands WHERE id = 1`

Recursive part idea: join `brands` on `brands.id = cte.id + 1`, and use `CASE WHEN brands.category IS NOT NULL THEN brands.category ELSE cte.category END`.

Not built or run end to end in this session. Worth revisiting later.

---

## Comparison

| Approach | Technique | Pros | Cons |
|----------|-----------|------|------|
| 1. COUNT tally + MAX | Window functions, gaps and islands | Portable, classic interview answer | Two layers (subquery) |
| 2. LAST_VALUE IGNORE NULLS | Window frame | Shortest, cleanest | Needs IGNORE NULLS support |
| 3. Correlated subquery | Per-row subquery | Easy to reason about | Slow on large tables (re-runs per row) |
| 4. Recursive CTE | Row-by-row iteration | Shows recursion | Verbose, needs gap-free ids |

## Key Takeaways
- `COUNT(col)` ignores NULLs, which is what makes the tally trick work.
- `MAX`/`MIN` ignore NULLs, so they can pull the single real value from a group.
- `LIMIT` in a join applies globally; use a correlated subquery for per-row limits.
- Always establish an ordering column before "previous/next" logic.
