# Deloitte SQL Interview Question — IPL Tournament Fixtures

## Problem

Given a `teams` table with a single column `Teamname` listing 6 IPL teams, write a query to generate all unique fixtures (each team plays every other team exactly once, no reversed duplicates, no team playing itself).

**Input Table**

| Teamname |
|---|
| RCB |
| KKR |
| SRH |
| DC |
| CSK |
| GT |

**Expected Output** (15 rows total)

| Fixtures |
|---|
| CSK VS DC |
| CSK VS GT |
| CSK VS KKR |
| CSK VS RCB |
| CSK VS SRH |
| DC VS GT |
| ... (15 total) |

**Row count check:** with 6 teams, unique pairs = `6 × 5 / 2 = 15` (each team pairs with 5 others; divide by 2 to remove reverse duplicates).

---

## What Makes This Different From the Route-Pairs Problem

In the earlier Deloitte "unique route pairs" problem, both directions of each pair **already existed as rows** in the table — the task was to filter down to one direction. Here, there's only a **single column of team names** — no pairs exist yet at all. The real challenge is **generating** every possible pair first, before any deduplication logic can apply.

---

## Approach 1: CROSS JOIN + LEAST/GREATEST + ROW_NUMBER + QUALIFY

**Logic:** Generate all ordered pairs via `CROSS JOIN`, exclude self-pairs, fingerprint each pair with `LEAST`/`GREATEST` (same trick as the route-pairs problem), then rank and filter to keep one direction per pair.

```sql
WITH CTE AS (
    SELECT t1.teamname, t2.teamname,
           LEAST(t1.teamname, t2.teamname) AS first_team,
           GREATEST(t1.teamname, t2.teamname) AS second_team,
           ROW_NUMBER() OVER (PARTITION BY first_team, second_team ORDER BY first_team, second_team) AS row_num
    FROM teams t1
    CROSS JOIN teams t2
    WHERE t1.teamname != t2.teamname
    QUALIFY row_num = 1
)
SELECT CONCAT(first_team, ' VS ', second_team) AS Fixtures FROM CTE;
```

- `CROSS JOIN ... WHERE t1.teamname != t2.teamname` produces all 30 ordered pairs (6×5), excluding self-pairs.
- `LEAST`/`GREATEST` collapses CSK-DC and DC-CSK into the identical fingerprint `(CSK, DC)`.
- `ROW_NUMBER` + `QUALIFY row_num=1` keeps one row per fingerprint — same rank-then-filter pattern used in the LatestOrder problem.

---

## Approach 2: CROSS JOIN + LEAST/GREATEST + DISTINCT (simpler)

**Logic:** Same fingerprinting idea, but since after `LEAST`/`GREATEST` the "duplicate" rows become **literally identical rows** (not just logically equivalent), `DISTINCT` alone can remove them — no `ROW_NUMBER`/`QUALIFY`/CTE needed.

```sql
SELECT DISTINCT CONCAT(LEAST(t1.teamname, t2.teamname), ' VS ', GREATEST(t1.teamname, t2.teamname)) AS Matches
FROM teams t1
CROSS JOIN teams t2
WHERE t1.teamname != t2.teamname;
```

**Why `DISTINCT` works here:** `DISTINCT` always compares whole rows, not individual columns. Before `LEAST`/`GREATEST`, the row from (t1=CSK, t2=DC) is `("CSK","DC")` and the row from (t1=DC, t2=CSK) is `("DC","CSK")` — genuinely different rows, since `DISTINCT` has no built-in concept of "these represent the same match, reversed." It's only after `LEAST`/`GREATEST` forces both rows into the same column order that they become byte-for-byte identical — at which point `DISTINCT` performs completely ordinary duplicate removal, the same as it would on any table with repeated rows. All the "cleverness" happens in `LEAST`/`GREATEST`; `DISTINCT`'s job afterward is trivial.

---

## Approach 3: Self-Join with `<` (most elegant — prevents duplicates rather than cleaning them up)

**Logic:** Instead of generating all 30 ordered pairs and then collapsing duplicates, only ever join rows where `t1.teamname < t2.teamname` — so each unique pair is produced **exactly once**, from the start.

```sql
SELECT CONCAT(t1.teamname, ' VS ', t2.teamname) AS Fixtures
FROM teams t1
JOIN teams t2 ON t1.teamname < t2.teamname;
```

- No `CROSS JOIN`, no `LEAST`/`GREATEST`, no `DISTINCT` needed — the `<` condition alone guarantees only one direction of each pair ever gets joined.
- This is the cleanest of all approaches: **preventing duplicates at the source is better than generating and filtering them out afterward.**

---

## Approach 4: Self-Join on Row Number (order-independent of alphabetical sorting)

**Logic:** Same idea as Approach 3, but instead of tying pair-selection to alphabetical order, assign each team a stable number first (via `ROW_NUMBER()`), then self-join on that number.

```sql
WITH CTE AS (
    SELECT teamname, ROW_NUMBER() OVER (ORDER BY teamname) AS rn FROM teams
)
SELECT CONCAT(t1.teamname, ' VS ', t2.teamname) AS Fixtures
FROM CTE t1
JOIN CTE t2 ON t1.rn < t2.rn;
```

- A CTE is required here because `ROW_NUMBER()` needs to be computed once per team before it can be compared across two joined instances — a window function's result can't be referenced directly inside the `JOIN ON` clause of the same query where it's computed.
- Useful when pairing needs to follow some ordering *other than* alphabetical (e.g., a seeding rank or draft order) — swap `ORDER BY teamname` for whatever ordering matters.

---

## Other Approaches (conceptual, not needed for this small dataset)

- **Recursive CTE / number generator:** if you didn't have a physical `teams` table but needed to generate combinations from a range of integers (e.g., "all pairs from players 1–10"), you'd first generate the numbers via a recursive CTE or Snowflake's `GENERATOR` function, then apply the same `<` self-join logic on top.
- **Snowflake ARRAY_AGG + FLATTEN:** collect all team names into a single array, then use `FLATTEN` twice with an index-based filter (`index1 < index2`) to generate unique pairs without a self-join — an array-native alternative, though unnecessary complexity for a simple flat table like this one.

---

## Common Thread

Every approach ultimately does one of two things: **(a) generate all ordered pairs and then filter down to one direction** (Approaches 1 & 2, via fingerprinting + ROW_NUMBER/QUALIFY or DISTINCT), or **(b) only ever generate one direction in the first place** (Approaches 3 & 4, via a `<` condition on some orderable value). Approach (b) is generally preferable when available — fewer steps, no wasted row generation, and no risk of the fingerprinting/dedup step being written incorrectly.
