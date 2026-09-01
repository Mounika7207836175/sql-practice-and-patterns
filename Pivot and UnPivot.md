**Syntax for pivot in snowflake**
```select * from Region
PIVOT(
  sum(Sales)
  for Quarter IN ('Q1', 'Q2', 'Q3', 'Q4')
) AS p(Region, Q1, Q2, Q3,Q4);
```
