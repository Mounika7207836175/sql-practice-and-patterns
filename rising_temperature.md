# Hard SQL Interview Question — Temperature Higher Than Previous Day

## Problem

Given a `weather` table with `id`, `recorddate`, `temperature`, find all `id`s where the temperature was **higher than the previous day**.

**Input Table**

| id | recordDate | temperature |
|---|---|---|
| 1 | 2025-10-01 | 20 |
| 2 | 2025-10-02 | 25 |
| 3 | 2025-10-04 | 22 |
| 4 | 2025-10-05 | 30 |
| 5 | 2025-10-06 | 28 |
| 6 | 2025-10-08 | 35 |

**Expected Output**

| id |
|---|
| 2 |
| 4 |

## Key Trap: Dates Are Not Consecutive

The dates have gaps — 2025-10-03 and 2025-10-07 are missing entirely. So "previous day" means the **literal calendar date minus 1 day**, not just "whatever row comes before this one in the table."

- id=3 (2025-10-04): the previous *calendar* day would be 2025-10-03 — which doesn't exist in the table. So id=3 can never qualify, even though temp (22) is higher than id=2's temp (25 → actually lower, but even if it were higher, the date gap disqualifies it).
- id=6 (2025-10-08): previous calendar day would be 2025-10-07 — also missing. Disqualified regardless of temperature.

This is why every approach below explicitly checks the **date gap equals 1**, not just "the row before this one."

---

## Approach 1: LAG() + DATEDIFF + QUALIFY

```sql
SELECT id, recorddate,
       DATEDIFF(day, LAG(recorddate,1) OVER (ORDER BY recorddate), recorddate) AS difference,
       temperature
FROM weather
QUALIFY difference = 1 
     AND temperature > LAG(temperature) OVER (ORDER BY recorddate);
```

- `LAG(recorddate,1)` looks *backward* to the previous row's date; `DATEDIFF` measures the day-gap between that and the current row.
- `QUALIFY` can reference the `difference` alias directly because it runs after the SELECT list is computed (unlike `WHERE`, which runs before aggregates/window functions and can't see them in most databases).
- Only rows where the gap is exactly 1 day AND today's temp exceeds yesterday's pass through.

---

## Approach 2: Self-Join with Date Arithmetic

```sql
SELECT w2.ID, w2.recorddate 
FROM weather w1 
JOIN weather w2
  ON w2.recorddate - w1.recorddate = 1 
 AND w2.temperature > w1.temperature;
```

- `w1` represents "yesterday," `w2` represents "today." The join condition directly encodes both requirements: exactly 1 day apart, and today's temp higher.
- `w2.recorddate - w1.recorddate = 1` relies on Snowflake allowing direct arithmetic on date types (subtracting two dates gives an integer day count). This is **dialect-specific** — not all databases support this. A portable version: `DATEADD(day, 1, w1.recorddate) = w2.recorddate`.
- Selecting `w2.ID` (not `w1.ID`) is essential — `w2` is "today," the day whose temperature actually rose.

---

## Approach 3: CASE + Subquery Filter

```sql
SELECT ID FROM (
    SELECT 
        CASE WHEN DATEDIFF(day, LAG(recorddate) OVER (ORDER BY recorddate), recorddate) = 1 
                  AND temperature > LAG(temperature) OVER (ORDER BY recorddate)
             THEN ID 
        END AS ID
    FROM weather
) AS tagged
WHERE ID IS NOT NULL;
```

- Manual equivalent of Approach 1's `QUALIFY` — the `CASE` tags qualifying rows with their `ID`, and non-qualifying rows get `NULL`.
- The outer `WHERE ID IS NOT NULL` filters out the NULLs, leaving only the qualifying ids.
- Style note: avoid naming the subquery alias the same as the derived column (e.g., both called `ID`) — rename the subquery alias (e.g., `tagged`) to avoid confusion in more complex queries.

---

## Approach 4: LEAD() — and the Bug It's Easy to Fall Into

**The trap:** it's tempting to write this using `LEAD()` symmetrically to Approach 1's `LAG()` — but naively doing so selects the **wrong id**.

**Why:** `LEAD(recorddate,1)` sitting on a given row looks *forward* to the next row. So a `difference` computed as `DATEDIFF(day, recorddate, LEAD(recorddate,1) OVER (...))` on row id=1 is really describing the gap *between id=1 and id=2* — not something that belongs to id=1 alone. Similarly, `LEAD(temperature) > temperature` evaluated on id=1's row is asking "does tomorrow's temp exceed today's" — a fact about tomorrow (id=2) rising, not about id=1.

**The incorrect version** (selects the earlier day's id — off by one row):
```sql
SELECT id FROM (
    SELECT id, recorddate,
           LEAD(id) OVER (ORDER BY recorddate) AS next_id,
           LEAD(temperature) OVER (ORDER BY recorddate) AS next_temp,
           LEAD(recorddate) OVER (ORDER BY recorddate) AS next_date
    FROM weather
)
WHERE next_temp > temperature AND DATEDIFF(day, recorddate, next_date) = 1;
-- Wrong: outputs id=1 and id=3 instead of id=2 and id=4
```
This returns id=1 and id=3 instead of id=2 and id=4 — always "one row behind" the correct answer, because it selects the *current* row's id when the condition is actually describing what happened to the *next* row.

**The corrected version** — select `LEAD(id,1)` itself, not the current row's `id`:
```sql
SELECT LEAD(id, 1) OVER (ORDER BY recorddate) AS id
FROM weather
QUALIFY DATEDIFF(day, recorddate, LEAD(recorddate,1) OVER (ORDER BY recorddate)) = 1
     AND LEAD(temperature,1) OVER (ORDER BY recorddate) > temperature;
```
- The filtering conditions stay on the *current* row's perspective ("does the next day's temp exceed mine, and is it exactly 1 day later"), but the final `SELECT` outputs `LEAD(id,1)` — the id of the day that actually rose — not the current row's own id.

**Lesson:** whichever direction (`LAG` or `LEAD`) you use to *detect* a condition, always double check which row's id you're actually supposed to report — the "detecting" row and the "reporting" row aren't always the same one.

---

## Approach 5: Correlated Subquery with EXISTS

```sql
SELECT id FROM weather w1
WHERE EXISTS (
    SELECT 1 FROM weather w2
    WHERE w2.recorddate = DATEADD(day, -1, w1.recorddate)
      AND w1.temperature > w2.temperature
);
```

- For each row `w1`, checks whether a row `w2` exists with a date exactly one day earlier and a lower temperature.
- `EXISTS` just needs to know *whether any such row exists* — it doesn't need to retrieve values from it, hence `SELECT 1`.
- Naturally selects `w1.id` — the "today" row — with no risk of the off-by-one-row mistake from Approach 4, since the correlated condition is phrased entirely from "today looking backward."

---

## Common Thread

All five approaches boil down to the same two checks: **(1) is there a row exactly one calendar day before this one, and (2) is this row's temperature higher than that one's.** The differences are just mechanical — window functions (`LAG`/`LEAD`) looking sideways at neighboring rows vs. a self-join/correlated subquery explicitly pairing rows together. The recurring lesson: when using `LAG`/`LEAD`, always verify which row's identity you intend to output — the row doing the "looking" isn't always the row that should appear in the final answer.
