# SQL Practice & Query Patterns

Welcome to my SQL repository! This project serves as a practical library where I document SQL queries, database concepts, optimization techniques, and solutions to real-world data retrieval problems.

## 📌 Repository Purpose
- Document end-to-end solutions for relational database queries (PostgreSQL / SQL Server / MySQL).
- Explore multiple solution techniques (Subqueries, Joins, Set Operations, Aggregations) for the same problem.
- Track common SQL pitfalls, duplicate handling (`DISTINCT`), and execution behavior.

## 🛠️ Topics Covered
- **Filtering & Subqueries:** `IN`, `EXISTS`, correlated subqueries
- **Set Operations:** `INTERSECT`, `UNION`, `EXCEPT`
- **Joins:** `INNER JOIN`, `LEFT JOIN`, Self-Joins
- **Aggregations:** `GROUP BY`, `HAVING`, `COUNT(DISTINCT ...)`

## 💡 Highlighted Problem Patterns

### Finding Customers Who Purchased Multiple Specific Items
**Scenario:** Identify customers who purchased both `Apple` AND `Orange`.

- **Approach 1: Subquery (`IN`)**
  ```sql
  SELECT DISTINCT CustomerName 
  FROM CustomerPurchases 
  WHERE Fruit = 'Orange' 
    AND CustomerName IN (
        SELECT CustomerName 
        FROM CustomerPurchases 
        WHERE Fruit = 'Apple'
    );
