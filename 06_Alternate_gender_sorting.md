# Deloitte SQL Interview Question — Sort Result in Alternate Order of Gender

## Problem

Given an `employees` table with `id`, `EmpName`, `Gender`, sort the result so that rows **alternate** between genders, while preserving each gender's original relative order (by `id`).

**Input Table**

| id | EmpName | Gender |
|---|---|---|
| 1 | Allen | MALE |
| 2 | Benjamin | MALE |
| 3 | Charle | MALE |
| 4 | Daisy | FEMALE |
| 5 | Emma | FEMALE |

**Expected Output**

| id | EmpName | Gender |
|---|---|---|
| 1 | Allen | MALE |
| 4 | Daisy | FEMALE |
| 2 | Benjamin | MALE |
| 5 | Emma | FEMALE |
| 3 | Charle | MALE |

## Core Idea

First give each employee a **position number within their own gender** (1st male, 2nd male, 3rd male; 1st female, 2nd female) using `ROW_NUMBER() OVER (PARTITION BY Gender ORDER BY id)`. Once every row has this "rank within gender," the three approaches below differ only in *how* they turn that rank into the final interleaved order.

---

## Approach 1: PARTITION BY + ORDER BY (rank, tiebreak)

```sql
SELECT id, empname, gender FROM (
    SELECT id, empname, gender,
           ROW_NUMBER() OVER (PARTITION BY gender ORDER BY id) AS row_num
    FROM employees
)
ORDER BY row_num, gender DESC;
```

**Why it works:**
- After ranking, both Allen (MALE) and Daisy (FEMALE) have `row_num = 1`; Benjamin and Emma both have `row_num = 2`; Charle alone has `row_num = 3`.
- `ORDER BY row_num` groups all the "1st in their gender" rows together, all "2nd in their gender" together, etc. — this alone produces "1,1,2,2,3", which is most of the interleaving.
- The remaining question is which gender wins the tie when `row_num` is equal. Plain `ORDER BY gender` (ascending) would sort alphabetically — "FEMALE" < "MALE" since F comes before M — putting FEMALE first, which is backwards from the expected output. Adding `DESC` flips the tiebreak so MALE sorts first within each tied `row_num`.

---

## Approach 2: UNION ALL of Two Independently-Ranked Queries

```sql
SELECT id, empname, gender FROM (
    SELECT id, empname, gender,
           ROW_NUMBER() OVER (ORDER BY id) AS row_num
    FROM employees WHERE gender = 'MALE'
    UNION ALL
    SELECT id, empname, gender,
           ROW_NUMBER() OVER (ORDER BY id) AS row_num
    FROM employees WHERE gender = 'FEMALE'
)
ORDER BY row_num, gender DESC;
```

- Ranks MALE and FEMALE employees completely independently (in two separate queries), each producing its own `row_num` starting at 1.
- Since each half's `WHERE` clause already filters to a single gender, `PARTITION BY gender` inside `ROW_NUMBER()` is unnecessary here — `ROW_NUMBER() OVER (ORDER BY id)` alone is sufficient, since there's no cross-gender contamination to guard against within either half.
- `UNION ALL` stacks the two ranked halves together; the same final `ORDER BY row_num, gender DESC` from Approach 1 interleaves them.
- Useful when the two groups need genuinely different logic (different filters, different tiebreakers, etc.) rather than a single shared rule.

---

## Approach 3: Explicit Interleave-Key Formula

**Logic:** instead of relying on `ORDER BY (rank, tiebreak)` to *happen* to interleave correctly, compute a single number per row that directly encodes its final position, then sort by that alone.

**The formula:** for a MALE row with within-gender rank `n`, the final position is `2n - 1` (giving 1, 3, 5). For a FEMALE row with rank `n`, the final position is `2n` (giving 2, 4).

```sql
WITH CTE AS (
    SELECT id, gender, empname,
           COALESCE(
               CASE WHEN gender = 'MALE' THEN 2*row_num - 1 END,
               CASE WHEN gender = 'FEMALE' THEN 2*row_num END
           ) AS row_num
    FROM (
        SELECT id, empname, gender,
               ROW_NUMBER() OVER (PARTITION BY gender ORDER BY id) AS row_num
        FROM employees
    )
)
SELECT id, empname, gender FROM CTE
ORDER BY row_num;
```

- Each `CASE WHEN` only handles its own gender and has no `ELSE`, so for any given row, exactly one of the two `CASE` expressions evaluates to a real number and the other evaluates to NULL (e.g., Allen gets `(1, NULL)`).
- `COALESCE` picks the first non-NULL value between the two, collapsing `(1, NULL)` down to `1`. This is the same "extract the real value out of a NULL/value pair" idea used in the `MAX(CASE WHEN...)` pattern from the LatestOrder problem — just applied across two columns on the same row instead of across multiple rows in a group.
- Once every row has its final position pre-computed, a single `ORDER BY row_num` (no tiebreak column needed) produces the fully interleaved result directly.

---

## Common Thread

All three approaches rely on the same foundation — **rank each row within its own group** — and differ only in how that rank becomes the final row order: via a tiebreak column (Approach 1), via combining independently-ranked subsets (Approach 2), or via directly computing the final position as a formula (Approach 3). This "alternate/interleave two groups" pattern generalizes to any number of categories — the interleave formula in Approach 3 would need one `CASE WHEN` branch per category, each spaced by the total number of categories instead of just 2.
