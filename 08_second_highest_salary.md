# Second Highest Salary (Distinct Version)

## Problem
Table `employee_salary(emp_id, emp_name, salary)`. Return the second highest **distinct** salary. If there is none, the result should be NULL.

Example salaries: 500, 500, 300, 200. Ties count as one level, so the answer is **300**.

| salary | level |
|--------|-------|
| 500 (two rows) | 1st |
| 300 | 2nd |
| 200 | 3rd |

---

## Approach 1: DENSE_RANK

```sql
SELECT emp_id, emp_name, salary,
       DENSE_RANK() OVER (ORDER BY salary DESC) AS rank
FROM employee_salary
QUALIFY rank = 2;
```

- `DENSE_RANK` gives tied salaries the same rank with no gaps (500, 500, 300, 200 becomes 1, 1, 2, 3).
- `RANK()` would give 1, 1, 3, 4 (a gap), and `ROW_NUMBER()` would give 1, 2, 3, 4 (ties split), so neither works here.
- Returns all employees who earn the second highest salary.

---

## Approach 2: MAX with a subquery

```sql
SELECT MAX(salary)
FROM employee_salary
WHERE salary < (SELECT MAX(salary) FROM employee_salary);
```

- The inner query finds the top salary. The outer query takes the highest salary below it.
- No window functions needed.
- Returns NULL (one row) when there is no second salary, because an aggregate without `GROUP BY` always returns one row.

---

## Approach 3: ORDER BY + LIMIT/OFFSET

```sql
SELECT DISTINCT salary
FROM employee_salary
ORDER BY salary DESC
LIMIT 1 OFFSET 1;
```

- `DISTINCT` collapses tied salaries into one level before sorting.
- `OFFSET 1` skips the top salary, and `LIMIT 1` takes the next one.
- Without `DISTINCT`, the tie at the top would return 500 instead of 300.

---

## Approach 4: Correlated subquery with COUNT

```sql
SELECT DISTINCT e1.salary
FROM employee_salary e1
WHERE (
    SELECT COUNT(DISTINCT e2.salary)
    FROM employee_salary e2
    WHERE e2.salary > e1.salary
) = 1;
```

- For each row, count how many **different** salaries are higher.
- The second highest distinct salary has exactly one higher salary above it.
- `COUNT(DISTINCT ...)` makes the two 500s count once.

| salary | distinct salaries above it |
|--------|----------------------------|
| 500 | 0 |
| 300 | 1 |
| 200 | 2 |

---

## Approach 5: Self join

```sql
SELECT e1.salary
FROM employee_salary e1
LEFT JOIN employee_salary e2
  ON e2.salary > e1.salary
GROUP BY e1.salary
HAVING COUNT(DISTINCT e2.salary) = 1;
```

- Each row is paired with every row that earns more.
- Grouping by `e1.salary` collapses ties into one group.
- Count `e2.salary`, not `e1.salary`. `e1.salary` is never NULL, so counting it is always at least 1 (the same bug as Problem 2).

---

## NULL handling (when there is no second highest salary)

| Approach | No second salary returns |
|----------|--------------------------|
| 1. DENSE_RANK + QUALIFY | Zero rows (QUALIFY filters everything out) |
| 2. MAX + subquery | One row with NULL |
| 3. LIMIT/OFFSET | Zero rows |
| 4. Correlated subquery | Zero rows |
| 5. Self join + HAVING | Zero rows |

Idea to get NULL instead of zero rows: wrap the query as a scalar subquery, for example `SELECT (SELECT DISTINCT salary FROM employee_salary ORDER BY salary DESC LIMIT 1 OFFSET 1) AS second_highest;`. A scalar subquery that finds nothing returns NULL. This was discussed but not run in the session.

---

## Comparison

| Approach | Technique | Pros | Cons |
|----------|-----------|------|------|
| 1. DENSE_RANK | Window function | Easy to extend to Nth highest | Returns zero rows when missing |
| 2. MAX + subquery | Aggregate | Returns NULL naturally | Only good for 2nd highest |
| 3. LIMIT/OFFSET | Sort and skip | Very short | Needs DISTINCT, zero rows when missing |
| 4. Correlated subquery | Per-row COUNT | Generalizes (change `= 1` to `= N-1`) | Slow on large tables |
| 5. Self join | Join + GROUP BY | Same idea as approach 4 | Slow on large tables |

## Key Takeaways
- Ties at the top are why `ROW_NUMBER()` and plain `OFFSET` fail. Use `DENSE_RANK` or `DISTINCT`.
- `COUNT(DISTINCT ...)` counts levels, not rows.
- Aggregates without `GROUP BY` always return one row, which is what gives NULL for free.
- Count the column from the joined side (`e2`) in a self join.

## Still to do
- The **non-distinct (500) version**, where a tie at the top counts as the second highest.
