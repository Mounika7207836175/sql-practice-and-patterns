# Problem 01: Find Customers Who Purchased Both Apple and Orange

## 📌 Problem Description
Given a table of customer purchases, write a SQL query to identify all customers who purchased **both** an `'Apple'` **and** an `'Orange'`.

### Sample Schema & Dataset
```
CREATE TABLE CustomerPurchases (
    CustomerName VARCHAR(50),
    Fruit VARCHAR(50)
);

INSERT INTO CustomerPurchases (CustomerName, Fruit) 
VALUES 
    ('Alice', 'Apple'),
    ('Alice', 'Orange'),
    ('Bob', 'Apple'),
    ('Charlie', 'Orange'),
    ('David', 'Apple'),
    ('David', 'Orange'),
    ('Emma', 'Banana'),
    ('Alice', 'Apple'); -- Duplicate entry for testing
```
**🎯 Expected Output**
CustomerName
Alice
David

**Approach 1: Subquery with IN (My Initial Approach)**
```
SELECT DISTINCT CustomerName 
FROM CustomerPurchases 
WHERE Fruit = 'Orange' 
  AND CustomerName IN (
      SELECT CustomerName 
      FROM CustomerPurchases 
      WHERE Fruit = 'Apple'
  );
```
**Approach 2: Set Operations (INTERSECT)**
```
SELECT CustomerName FROM CustomerPurchases WHERE Fruit = 'Apple'
INTERSECT
SELECT CustomerName FROM CustomerPurchases WHERE Fruit = 'Orange'
ORDER BY CustomerName ASC;
```
**Approach 3: Conditional Aggregation (GROUP BY + HAVING)**
```
SELECT CustomerName
FROM CustomerPurchases
WHERE Fruit IN ('Apple', 'Orange')
GROUP BY CustomerName
HAVING COUNT(DISTINCT Fruit) = 2;
```
**🧠 Doubts Faced & Key Learnings**
1. Why DISTINCT is necessary in the Subquery approach
Observation: Running WHERE Fruit = 'Orange' checks every single matching row.

Pitfall: If Alice buys Orange twice in the table, the query without DISTINCT evaluates to True for both rows and outputs Alice twice.

Takeaway: Always use DISTINCT when filtering with IN subqueries if a single entity can have duplicate action rows.

2. Why INTERSECT changed the row output order (e.g., David before Alice)
Observation: In PostgreSQL, INTERSECT outputted David then Alice, unlike the insertion order.

Reason: SQL relational tables and set operations have no guaranteed row order by default. Set operations like INTERSECT remove duplicates by using internal hash tables or sorting algorithms, which alters row output sequence.

Takeaway: Never rely on implicit output ordering in SQL; always append an explicit ORDER BY clause if order matters.
