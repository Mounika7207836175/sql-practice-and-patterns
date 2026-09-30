# Snowflake DATE_TRUNC

## 1. What is DATE_TRUNC?

`DATE_TRUNC` moves a date or timestamp back to the **beginning of a specified time period**.

### Syntax

```sql
DATE_TRUNC(<part>, <date_or_timestamp>)
```

Example:

```sql
SELECT DATE_TRUNC('MONTH', '2026-09-29'::DATE);
```

Result:

```text
2026-09-01
```

The day is reset to the first day of the month.

---

## 2. How DATE_TRUNC works

Think of a timestamp like this:

```text
2026-09-29 14:35:47
│    │  │  │  │  │
│    │  │  │  │  └── Second
│    │  │  │  └───── Minute
│    │  │  └──────── Hour
│    │  └─────────── Day
│    └────────────── Month
└─────────────────── Year
```

`DATE_TRUNC` keeps the selected level and resets everything smaller than it.

| Part | Example result |
|---|---|
| `YEAR` | `2026-01-01 00:00:00` |
| `QUARTER` | `2026-07-01 00:00:00` |
| `MONTH` | `2026-09-01 00:00:00` |
| `WEEK` | Start of the week |
| `DAY` | `2026-09-29 00:00:00` |
| `HOUR` | `2026-09-29 14:00:00` |
| `MINUTE` | `2026-09-29 14:35:00` |
| `SECOND` | `2026-09-29 14:35:47` |

For example:

```sql
DATE_TRUNC('HOUR', '2026-09-29 14:35:47'::TIMESTAMP)
```

returns:

```text
2026-09-29 14:00:00
```

---

## 3. Common date parts

The most commonly used parts are:

```text
YEAR
QUARTER
MONTH
WEEK
DAY
HOUR
MINUTE
SECOND
```

---

## 4. YEAR

```sql
SELECT DATE_TRUNC('YEAR', '2026-09-29'::DATE);
```

Result:

```text
2026-01-01
```

Use it when you need the beginning of the year, such as for **YTD (Year-To-Date)** calculations.

---

## 5. QUARTER

A year has four quarters:

```text
Q1 → January - March
Q2 → April - June
Q3 → July - September
Q4 → October - December
```

Example:

```sql
SELECT DATE_TRUNC('QUARTER', '2026-09-29'::DATE);
```

Result:

```text
2026-07-01
```

Other examples:

```text
2026-02-15 → 2026-01-01
2026-05-20 → 2026-04-01
2026-08-10 → 2026-07-01
2026-12-25 → 2026-10-01
```

---

## 6. MONTH

```sql
SELECT DATE_TRUNC('MONTH', '2026-09-29'::DATE);
```

Result:

```text
2026-09-01
```

It moves any date in the month to the **first day of that month**.

Examples:

```text
2026-01-15 → 2026-01-01
2026-02-28 → 2026-02-01
2026-09-29 → 2026-09-01
2026-12-31 → 2026-12-01
```

---

## 7. WEEK

```sql
SELECT DATE_TRUNC('WEEK', '2026-09-29'::DATE);
```

This returns the beginning of the week.

The exact first day of the week is affected by Snowflake's `WEEK_START` session parameter.

You can check it with:

```sql
SHOW PARAMETERS LIKE 'WEEK_START';
```

---

## 8. DAY

For a timestamp:

```sql
SELECT DATE_TRUNC('DAY', '2026-09-29 14:35:47'::TIMESTAMP);
```

Result:

```text
2026-09-29 00:00:00
```

It removes the time portion by moving the timestamp to the beginning of the day.

Useful for daily reports and filtering.

---

## 9. HOUR

```sql
SELECT DATE_TRUNC('HOUR', '2026-09-29 14:35:47'::TIMESTAMP);
```

Result:

```text
2026-09-29 14:00:00
```

All timestamps within the 14:00 hour can therefore be grouped into the same hourly bucket.

---

## 10. MINUTE

```sql
SELECT DATE_TRUNC('MINUTE', '2026-09-29 14:35:47'::TIMESTAMP);
```

Result:

```text
2026-09-29 14:35:00
```

Seconds are removed.

---

## 11. SECOND

```sql
SELECT DATE_TRUNC(
    'SECOND',
    '2026-09-29 14:35:47.123'::TIMESTAMP
);
```

Result:

```text
2026-09-29 14:35:47
```

Fractional seconds are removed.

---

# 12. DATE vs TIMESTAMP

### DATE

A `DATE` contains only:

```text
YYYY-MM-DD
```

Example:

```text
2026-09-29
```

### TIMESTAMP

A timestamp contains date and time:

```text
YYYY-MM-DD HH:MI:SS
```

Example:

```text
2026-09-29 14:35:47
```

---

# 13. Important limitation: DATE vs TIMESTAMP

`DATE_TRUNC` can only work with information that exists in the input data type.

A `DATE` does **not** contain hour, minute, or second information.

For example:

```text
DATE:
2026-09-29
```

There is no hour to truncate.

A timestamp does contain time:

```text
TIMESTAMP:
2026-09-29 14:35:47
```

Therefore:

```sql
DATE_TRUNC('HOUR', timestamp_col)
```

can return:

```text
2026-09-29 14:00:00
```

### Easy rule

> **DATE → date-level information**  
> **TIMESTAMP → date + time information**

So you should use a timestamp when you need hour/minute/second-level truncation.

---

# 14. DATE_TRUNC with GROUP BY

One of the most common uses of `DATE_TRUNC` is grouping data by a time period.

Suppose:

```text
ORDERS

ORDER_DATE              AMOUNT
2026-09-01 10:15:00       100
2026-09-03 12:30:00       200
2026-09-15 09:10:00       150
2026-09-29 18:20:00       300
2026-10-01 10:00:00       500
```

To calculate monthly sales:

```sql
SELECT
    DATE_TRUNC('MONTH', order_date) AS month,
    SUM(amount) AS total_amount
FROM orders
GROUP BY DATE_TRUNC('MONTH', order_date)
ORDER BY month;
```

Result:

```text
MONTH       TOTAL_AMOUNT
2026-09-01      750
2026-10-01      500
```

All dates in the same month become the same truncated value.

---

# 15. DATE_TRUNC with CURRENT_DATE / CURRENT_TIMESTAMP

### Beginning of today

```sql
DATE_TRUNC('DAY', CURRENT_TIMESTAMP())
```

If the current timestamp is:

```text
2026-09-29 14:30:00
```

the result is:

```text
2026-09-29 00:00:00
```

Example:

```sql
WHERE order_date >= DATE_TRUNC('DAY', CURRENT_TIMESTAMP())
```

This can be used to retrieve records from the beginning of today.

### Beginning of the current month

```sql
DATE_TRUNC('MONTH', CURRENT_DATE())
```

If today is:

```text
2026-09-29
```

the result is:

```text
2026-09-01
```

### Beginning of the current year

```sql
DATE_TRUNC('YEAR', CURRENT_DATE())
```

If today is:

```text
2026-09-29
```

the result is:

```text
2026-01-01
```

---

# 16. DATE_TRUNC vs DATE_PART

These functions are different.

### DATE_PART

Extracts a component/value.

```sql
SELECT DATE_PART('MONTH', '2026-09-29'::DATE);
```

Result:

```text
9
```

### DATE_TRUNC

Moves the date to the beginning of that period.

```sql
SELECT DATE_TRUNC('MONTH', '2026-09-29'::DATE);
```

Result:

```text
2026-09-01
```

### Easy way to remember

```text
DATE_PART → "Give me the part."

DATE_TRUNC → "Take me to the start of the part."
```

---

# 17. DATE_TRUNC vs DATEDIFF

### DATE_TRUNC

Normalizes a date/time to the beginning of a period.

```sql
DATE_TRUNC('MONTH', date_col)
```

Example:

```text
2026-09-29 → 2026-09-01
```

### DATEDIFF

Calculates the difference between two dates/times.

```sql
DATEDIFF(
    'DAY',
    '2026-09-01'::DATE,
    '2026-09-29'::DATE
)
```

Result:

```text
28
```

### Easy way to remember

```text
DATE_TRUNC → changes/normalizes the date

DATEDIFF → calculates a difference
```

---

# 18. DATE_TRUNC vs TIME_SLICE

### DATE_TRUNC

Used for calendar boundaries:

```text
MONTH → first day of month
DAY   → beginning of day
HOUR  → beginning of hour
```

### TIME_SLICE

Used to divide time into fixed-size intervals, such as:

```text
10-minute intervals
15-minute intervals
30-minute intervals
```

Easy distinction:

```text
DATE_TRUNC → calendar boundaries

TIME_SLICE → fixed-size time intervals
```

---

# 19. Common real-world uses

`DATE_TRUNC` is commonly used for:

- Monthly sales/revenue reports
- Daily reports
- Hourly monitoring
- Weekly analysis
- Quarterly reporting
- MTD (Month-To-Date)
- YTD (Year-To-Date)
- Time-series analysis
- Grouping timestamps
- Incremental data processing
- Filtering data from the beginning of a period

---

# 20. Quick interview revision

### What does DATE_TRUNC do?

It truncates a date/time to the beginning of the specified period.

### Syntax

```sql
DATE_TRUNC(<part>, <date_or_timestamp>)
```

### Example

```sql
DATE_TRUNC('MONTH', '2026-09-29'::DATE)
```

Result:

```text
2026-09-01
```

### Difference from DATE_PART

```text
DATE_PART → extracts a value
DATE_TRUNC → returns the beginning of a period
```

### Difference from DATEDIFF

```text
DATE_TRUNC → normalizes a date/time
DATEDIFF → calculates the difference
```

### Difference from TIME_SLICE

```text
DATE_TRUNC → calendar-based boundaries
TIME_SLICE → fixed-size time intervals
```

### Key limitation

```text
DATE → contains date only
TIMESTAMP → contains date + time
```

Therefore, hour/minute/second-level truncation requires time information in the input, such as a timestamp.

---

# 21. One-line memory trick

> **DATE_TRUNC = Take a date/time and move it back to the beginning of the specified period.**
