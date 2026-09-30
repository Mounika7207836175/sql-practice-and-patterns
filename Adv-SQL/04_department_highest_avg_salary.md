# Department With the Highest Average Salary

## Problem
Table `employee_avg_salary(emp_id, emp_name, department, salary)`. Return the department with the highest average salary.

Result: **Finance**.

---

## Approach 1: GROUP BY + ORDER BY + LIMIT

```sql
SELECT department, AVG(salary)
FROM employee_avg_salary
GROUP BY department
ORDER BY AVG(salary) DESC
LIMIT 1;
```

**How it works**
- Classic and cleanest approach: group by department, compute each department's average, sort highest first, take the top row.

---

## Approach 2: AVG as a window function

```sql
SELECT department
FROM (
    SELECT department, AVG(salary) OVER (PARTITION BY department) AS max_avg
    FROM employee_avg_salary
)
ORDER BY max_avg DESC
LIMIT 1;
```

**How it works**
- `AVG(salary) OVER (PARTITION BY department)` computes each department's average without collapsing rows — every employee row keeps its department's average attached.
- Sorting and taking `LIMIT 1` still lands on the right department, since all rows in that department share the same top value.

**Note (discussed in session):** this approach works, but doesn't take advantage of anything a plain `GROUP BY` doesn't already do here. Window functions are more useful when individual rows need to be kept alongside an aggregate; here only the aggregate itself is needed, so Approach 1 is the more natural fit.

---

## Approach 3: MAX with a subquery

```sql
SELECT department
FROM employee_avg_salary
GROUP BY department
HAVING AVG(salary) = (
    SELECT MAX(avg_salary)
    FROM (
        SELECT AVG(salary) AS avg_salary
        FROM employee_avg_salary
        GROUP BY department
    )
);
```

**How it works**
- Innermost query: one average salary per department.
- Middle layer: `MAX(avg_salary)` — the single highest average across all departments.
- Outer query: groups by department again and uses `HAVING` (not `WHERE`, since the filter is on an aggregate) to keep the department matching that highest value.
- Naturally handles ties — if two departments shared the top average, both would be returned.

---

## Approach 4: DENSE_RANK + QUALIFY

```sql
SELECT department
FROM (
    SELECT department, AVG(salary) AS avg_salary
    FROM employee_avg_salary
    GROUP BY department
)
QUALIFY DENSE_RANK() OVER (ORDER BY avg_salary DESC) = 1;
```

**How it works**
- Inner query collapses to one row per department with its average salary.
- `DENSE_RANK()` ranks departments by that average, highest first.
- `QUALIFY ... = 1` keeps only the top-ranked department(s).
- `DENSE_RANK` (rather than `RANK` or `ROW_NUMBER`) is used so tied departments would both show up as rank 1, same reasoning as in the Second Highest Salary problem.

---

## Comparison

| Approach | Technique | Handles ties? | Notes |
|----------|-----------|----------------|-------|
| 1. GROUP BY + LIMIT | Aggregate + sort | No | Simplest, most direct |
| 2. AVG as window function | Window function | No | Works, but doesn't add value over Approach 1 here |
| 3. MAX + subquery | Nested aggregates | Yes | Three layers, but flexible and tie-safe |
| 4. DENSE_RANK + QUALIFY | Window ranking | Yes | Clean, extends easily to "top N departments" |

## Key Takeaways
- `LIMIT 1` silently drops ties; `HAVING = (subquery)` or `QUALIFY rank = 1` both return every tied department instead.
- A window function like `AVG() OVER(PARTITION BY ...)` is most useful when you need to keep individual rows alongside an aggregate — when you only need the aggregate itself, `GROUP BY` is simpler.
- `DENSE_RANK` (not `RANK` or `ROW_NUMBER`) is the right choice when ties should share the same rank.
