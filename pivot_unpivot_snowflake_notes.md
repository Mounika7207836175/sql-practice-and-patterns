# PIVOT and UNPIVOT in SQL (Snowflake) — Revision Notes

## 1. The Core Idea

- **PIVOT**: rows → columns. Distinct values inside one column become new column headers. Used to build summary/report-style tables.
- **UNPIVOT**: columns → rows. Multiple columns get "unstacked" back into rows. Used to normalize wide data (common in ETL when a source system hands you a wide export).

---

## 2. PIVOT — Rows to Columns

### Example data (long format)

| Region | Quarter | Sales |
|--------|---------|-------|
| East   | Q1      | 1000  |
| East   | Q2      | 1200  |
| East   | Q3      | 1100  |
| East   | Q4      | 1400  |
| West   | Q1      | 2000  |
| West   | Q2      | 2200  |
| West   | Q3      | 2100  |
| West   | Q4      | 2500  |
| North  | Q1      | 1500  |
| North  | Q2      | 1600  |
| North  | Q3      | 1550  |
| North  | Q4      | 1700  |

### Target (wide format)

| Region | Q1   | Q2   | Q3   | Q4   |
|--------|------|------|------|------|
| East   | 1000 | 1200 | 1100 | 1400 |
| West   | 2000 | 2200 | 2100 | 2500 |
| North  | 1500 | 1600 | 1550 | 1700 |

### Manual / portable version (works on Postgres, MySQL, Snowflake — anywhere)

```sql
SELECT 
    Region,
    SUM(CASE WHEN Quarter = 'Q1' THEN Sales ELSE 0 END) AS Q1,
    SUM(CASE WHEN Quarter = 'Q2' THEN Sales ELSE 0 END) AS Q2,
    SUM(CASE WHEN Quarter = 'Q3' THEN Sales ELSE 0 END) AS Q3,
    SUM(CASE WHEN Quarter = 'Q4' THEN Sales ELSE 0 END) AS Q4
FROM SalesTable
GROUP BY Region;
```

**How to read it:** for each row, the `CASE WHEN` checks if `Quarter` matches the target value. If yes, grab `Sales`; if no, use 0. `SUM` collapses all matching (and zero) values into one number per group. `GROUP BY Region` shrinks 12 rows down to 3 — one per region.

### Native Snowflake PIVOT (static — you know the values ahead of time)

```sql
SELECT *
FROM SalesTable
PIVOT (
    SUM(Sales)
    FOR Quarter IN ('Q1', 'Q2', 'Q3', 'Q4')
) AS p (Region, Q1, Q2, Q3, Q4);
```

- `SUM(Sales)` — aggregate function, required even if only one row matches each cell.
- `FOR Quarter IN ('Q1','Q2','Q3','Q4')` — the column whose **values** become new columns. These must be actual data values, in quotes.
- `AS p (Region, Q1, Q2, Q3, Q4)` — renames the output columns. Any column NOT referenced inside PIVOT (like `Region`) automatically stays as-is on the left.

### Dynamic PIVOT (values not known ahead of time)

Problem it solves: if a new value (e.g. a `Q5`) appears in the data and your static `IN (...)` list doesn't include it, that data silently gets dropped.

```sql
SELECT *
FROM SalesTable
PIVOT (
    SUM(Sales)
    FOR Quarter IN (ANY ORDER BY Quarter)
) AS p;
```

- `ANY` — tells Snowflake to look at the actual data and find every distinct value in `Quarter` itself (like running `SELECT DISTINCT Quarter` behind the scenes).
- `ORDER BY Quarter` — controls column order in the output; required syntax, can't be skipped.

**Limitation:** because the output schema isn't fixed in advance, dynamic pivot **cannot** be used inside a `CREATE VIEW`, and some BI tools that expect a stable schema won't work with it. Best for ad hoc exploration, not permanent pipeline objects.

**When to use which:**

| Situation | Use |
|---|---|
| Known, fixed set of values | Static PIVOT — `IN ('Q1','Q2',...)` |
| Values change / unknown ahead of time | Dynamic PIVOT — `IN (ANY ORDER BY ...)` |
| Feeding a view or BI tool needing stable schema | Static PIVOT only |

---

## 3. Worked Practice Problem — Static PIVOT

### Raw data

| Department | TrainingType | Hours |
|------------|--------------|-------|
| Sales | Onboarding | 5 |
| Sales | Compliance | 3 |
| Sales | Technical | 8 |
| IT | Onboarding | 4 |
| IT | Compliance | 2 |
| IT | Technical | 12 |
| HR | Onboarding | 6 |
| HR | Compliance | 5 |
| HR | Technical | 1 |

### Correct solution

```sql
SELECT *
FROM Employee_Training
PIVOT (
    SUM(Hours)
    FOR TrainingType IN ('Onboarding', 'Compliance', 'Technical')
) AS p (Dept, OB, Comp, Tech);
```

**Key rule to remember:** values inside `IN (...)` in the `PIVOT` clause must match the **actual data values exactly** (case-sensitive, no abbreviating). The abbreviation/renaming only happens in the final `AS p(...)` alias list — that part can be anything you like.

---

## 4. UNPIVOT — Columns to Rows

### Example: start from wide data

| Department | OB | Comp | Tech |
|------------|----|----|----|
| Sales | 5 | 3 | 8 |
| IT | 4 | 2 | 12 |
| HR | 6 | 5 | 1 |

### Target (long format)

| Department | TrainingType | Hours |
|------------|--------------|-------|
| Sales | OB | 5 |
| Sales | Comp | 3 |
| Sales | Tech | 8 |
| IT | OB | 4 |
| IT | Comp | 2 |
| IT | Tech | 12 |
| HR | OB | 6 |
| HR | Comp | 5 |
| HR | Tech | 1 |

### Snowflake syntax

```sql
SELECT Department, TrainingType, Hours
FROM Employee_Training_Wide
UNPIVOT (
    Hours
    FOR TrainingType IN (OB, Comp, Tech)
) AS up;
```

- `Hours` — a **new column name you invent**; holds all the stacked values.
- `TrainingType` — a **new column name you invent**; holds the label of which original column each value came from.
- `IN (OB, Comp, Tech)` — the **actual existing column names** to collapse into rows. No quotes (these are column names, not data values — opposite of PIVOT).
- `Department` stays untouched automatically, same logic as PIVOT.

---

## 5. PIVOT vs UNPIVOT — Quick Comparison

| | PIVOT | UNPIVOT |
|---|---|---|
| Direction | rows → columns | columns → rows |
| `IN (...)` contains | data values (quoted, e.g. `'Q1'`) | column names (no quotes, e.g. `OB`) |
| Aggregate function needed? | Yes (`SUM`, always) | No |
| Row count | shrinks | grows |
| Typical use case | reporting / BI layer | ETL / normalizing source data |

---

## 6. Practice Problem — UNPIVOT (try this one)

### Raw data (wide)

| Product | Quality | Durability | Price |
|---------|---------|------------|-------|
| ProductA | 8 | 7 | 6 |
| ProductB | 9 | 8 | 5 |
| ProductC | 6 | 9 | 8 |

### Target (long)

| Product | Metric | Score |
|---------|--------|-------|
| ProductA | Quality | 8 |
| ProductA | Durability | 7 |
| ProductA | Price | 6 |
| ProductB | Quality | 9 |
| ProductB | Durability | 8 |
| ProductB | Price | 5 |
| ProductC | Quality | 6 |
| ProductC | Durability | 9 |
| ProductC | Price | 8 |

**Task:** write the `UNPIVOT` query yourself using the pattern above.

### Correct solution

```sql
SELECT Product, Feature, Score
FROM ProductRatings
UNPIVOT (
    Score
    FOR Feature IN (Quality, Durability, Price)
) AS unpivottable;
```

**Key points:**
- `FROM ProductRatings` — must name the actual source (wide) table; without it, Snowflake has no idea where `Quality`, `Durability`, `Price` come from.
- `Score` — new column invented to hold the stacked values. No aggregate function needed (unlike PIVOT).
- `FOR Feature IN (Quality, Durability, Price)` — these are **actual column names** from the wide table, no quotes.
- Match casing to the real column names exactly (e.g. `Price`, not `price`) as a good habit, even though Snowflake is generally case-insensitive for unquoted identifiers.
