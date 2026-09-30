# Employees Who Joined in the Same Month & Year as Their Manager

## Problem
Table `employee_manager(employeeid, employeename, joiningdate, managerid)`. `managerid` points back to another `employeeid` in the same table (self-referencing). Return employees who joined in the same month and year as their manager.

| employeeid | employeename | joiningdate | managerid |
|------------|--------------|-------------|-----------|
| 1 | Rajesh | 2022-05-10 | NULL (manager) |
| 2 | Anitha | 2022-05-15 | 1 — same month & year as manager |
| 3 | Kiran | 2022-06-20 | 1 — different month |
| 4 | Suresh | 2023-01-05 | NULL (manager) |
| 5 | Priya | 2023-01-18 | 4 — same month & year as manager |
| 6 | Deepak | 2023-02-12 | 4 — different month |

Expected result: **Anitha** and **Priya**.

---

## Approach 1: Self join

```sql
SELECT 
    e1.employeeid AS empId, 
    e1.employeename AS empName, 
    e1.joiningdate AS empJoiningDate,
    e2.employeename AS managerName,
    e2.joiningdate AS managerJoiningDate
FROM employee_manager e1
JOIN employee_manager e2
    ON e1.managerid = e2.employeeid
WHERE DATE_TRUNC('month', e1.joiningdate) = DATE_TRUNC('month', e2.joiningdate);
```

**How it works**
- The same table is read twice with two aliases: `e1` plays the role of the employee, `e2` plays the role of the manager.
- The join condition `e1.managerid = e2.employeeid` lines up each employee with their manager's row.
- `DATE_TRUNC('month', ...)` rounds each date down to the 1st of its month, so one `=` comparison checks both month and year at once (replaces writing two separate `DATE_PART` checks for year and month).

**Common mistake (caught in session):** writing the join condition backwards as `e1.employeeid = managerid`, which matches each person to their *reports* instead of their *manager*. The `manager_id` column lives on the employee's row, so it must be matched against the manager's `employeeid`.

---

## Approach 2: Correlated subquery

```sql
SELECT *
FROM (
    SELECT 
        e1.employeeid,
        e1.employeename,
        e1.joiningdate AS employeejoiningdate,
        (
            SELECT e2.joiningdate 
            FROM employee_manager e2 
            WHERE e2.employeeid = e1.managerid
        ) AS managerjoiningdate
    FROM employee_manager e1
)
WHERE DATE_TRUNC('month', employeejoiningdate) = DATE_TRUNC('month', managerjoiningdate);
```

**How it works**
- The subquery in the `SELECT` list fetches the manager's `joiningdate` for each employee row, using `e1.managerid` to look it up.
- This attaches the manager's date as a plain column first (inner query), then a separate outer query filters using `DATE_TRUNC` on both plain columns.

**Errors hit along the way (session debugging):**
- Wrapping the correlated subquery directly inside `DATE_TRUNC(...)` in the `WHERE` clause caused: *"Unsupported subquery type cannot be evaluated."* Snowflake can't evaluate a correlated subquery nested inside a function call used in a comparison.
- **Fix:** split into two layers — an inner query that returns the manager's date as a plain column, and an outer query that does the `DATE_TRUNC` comparison on that already-resolved column.
- The subquery's `WHERE` clause order also mattered: `e2.employeeid = e1.managerid` (searched column on the left) worked, while the reversed order caused the same evaluation error.
- `DATE_TRUNC('month', ...)` needs `'month'` as a quoted string, not a bare word.

---

## Comparison

| Approach | Technique | Notes |
|----------|-----------|-------|
| 1. Self join | JOIN on `manager_id = employeeid` | Straightforward, single query, no nesting issues |
| 2. Correlated subquery | Subquery in SELECT list | Needs an inner/outer split to avoid the "unsupported subquery type" error |

## Key Takeaways
- A self-referencing table (a column pointing back to the same table's key) needs two aliases of the same table to compare a row to its related row.
- `DATE_TRUNC('month', date)` rounds a date to the 1st of its month, letting one `=` replace separate year and month checks.
- Snowflake can reject a correlated subquery when it's nested inside another function call in a `WHERE` clause; resolving the subquery in an inner query first, then filtering in an outer query, avoids this.
