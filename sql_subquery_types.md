# SQL Subquery Types

## 1. Scalar Subquery

Returns **one value** (one row and one column).

```sql
SELECT *
FROM employees
WHERE salary > (
    SELECT AVG(salary)
    FROM employees
);
```

**Remember:** One value.

---

## 2. Single-Row Subquery

Returns **one row**, potentially with multiple columns.

```sql
SELECT *
FROM employees
WHERE (department_id, salary) =
      (SELECT department_id, salary
       FROM employees
       WHERE emp_id = 101);
```

**Remember:** One row.

---

## 3. Multiple-Row Subquery

Returns **multiple rows**.

Commonly used with `IN`, `ANY`, or `ALL`.

```sql
SELECT *
FROM employees
WHERE department_id IN (
    SELECT department_id
    FROM departments
    WHERE location = 'CHENNAI'
);
```

**Remember:** Many rows.

---

## 4. Correlated Subquery

The inner query depends on the outer query.

```sql
SELECT E1.*
FROM employees E1
WHERE salary > (
    SELECT AVG(E2.salary)
    FROM employees E2
    WHERE E2.department_id = E1.department_id
);
```

The inner query refers to `E1`, which belongs to the outer query.

**Remember:** Inner query depends on outer query.

---

## 5. Non-Correlated Subquery

The inner query is independent of the outer query.

```sql
SELECT *
FROM employees
WHERE salary > (
    SELECT AVG(salary)
    FROM employees
);
```

The inner query does not refer to the outer query.

**Remember:** Independent inner query.

---

## 6. Nested Subquery

A subquery exists inside another subquery.

```sql
SELECT *
FROM employees
WHERE salary > (
    SELECT AVG(salary)
    FROM employees
    WHERE department_id IN (
        SELECT department_id
        FROM departments
        WHERE location = 'CHENNAI'
    )
);
```

**Remember:** Subquery inside subquery.

---

## 7. Subquery in WHERE

Used mainly for filtering rows.

```sql
SELECT *
FROM employees
WHERE department_id IN (
    SELECT department_id
    FROM departments
);
```

**Remember:** Filtering.

---

## 8. Subquery in FROM

The subquery result acts like a temporary table.

```sql
SELECT *
FROM (
    SELECT department_id, AVG(salary) AS avg_salary
    FROM employees
    GROUP BY department_id
) A;
```

**Remember:** Subquery becomes a table.

---

## 9. Subquery in SELECT

The subquery produces a value/column.

```sql
SELECT
    emp_id,
    salary,
    (SELECT AVG(salary) FROM employees) AS company_avg_salary
FROM employees;
```

**Remember:** Subquery becomes a column/value.

---

## 10. EXISTS Subquery

Checks whether at least one matching row exists.

```sql
SELECT *
FROM employees E
WHERE EXISTS (
    SELECT 1
    FROM departments D
    WHERE D.department_id = E.department_id
);
```

**Remember:** "Does a matching row exist?"

---

# Quick Revision Table

| Type | Meaning |
|---|---|
| Scalar | Returns one value |
| Single-row | Returns one row |
| Multiple-row | Returns multiple rows |
| Correlated | Depends on outer query |
| Non-correlated | Independent of outer query |
| Nested | Subquery inside another subquery |
| WHERE subquery | Used for filtering |
| FROM subquery | Acts like a temporary table |
| SELECT subquery | Produces a column/value |
| EXISTS | Checks whether a matching row exists |

## Important Note

These categories can overlap.

For example, a subquery can be both **correlated** and **scalar**.
