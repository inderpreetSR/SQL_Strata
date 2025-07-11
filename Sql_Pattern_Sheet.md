# SQL Patterns Cheat Sheet

A quick reference to the most common SQL interview/exam patterns. Spot the trigger phrases in a question, then jump to the matching syntax.

---

## 1. Find Duplicates
**Phrasing:** “Which rows occur more than once…?”  
**Key:** `GROUP BY` + `HAVING COUNT(*) > 1`  
```sql
SELECT col1, col2, COUNT(*) AS cnt
FROM your_table
GROUP BY col1, col2
HAVING COUNT(*) > 1;
```

---

## 2. Top N per Group
**Phrasing:** “Give the top 3 salaries per department…”  
**Key:** `ROW_NUMBER()` window function  
```sql
SELECT *
FROM (
  SELECT
    *,
    ROW_NUMBER() OVER (PARTITION BY department
                       ORDER BY salary DESC) AS rn
  FROM employees
) t
WHERE rn <= 3;
```

---

## 3. Nth Highest Value
**Phrasing:** “Who has the 2nd highest score…?”  
**Key:** `DENSE_RANK()` or scalar subquery  
```sql
-- Using DENSE_RANK()
SELECT *
FROM (
  SELECT
    *,
    DENSE_RANK() OVER (ORDER BY score DESC) AS rnk
  FROM scores
) t
WHERE rnk = 2;

-- Or scalar subquery
SELECT *
FROM scores
WHERE score = (
  SELECT MAX(score)
  FROM scores
  WHERE score < (
    SELECT MAX(score) FROM scores
  )
);
```

---

## 4. Running Total / Cumulative Sum
**Phrasing:** “Show cumulative sales by date…”  
**Key:** `SUM() OVER (ORDER BY … ROWS UNBOUNDED PRECEDING)`  
```sql
SELECT
  sale_date,
  amount,
  SUM(amount) OVER (
    ORDER BY sale_date
    ROWS UNBOUNDED PRECEDING
  ) AS running_total
FROM sales
ORDER BY sale_date;
```

---

## 5. Filter on Aggregate
**Phrasing:** “Which customers spent above the average order value?”  
**Key:** `HAVING` or `WHERE … IN (subquery)`  
```sql
-- HAVING on grouped totals
SELECT customer_id, SUM(total) AS total_spent
FROM orders
GROUP BY customer_id
HAVING SUM(total) > (
  SELECT AVG(total) FROM orders
);

-- Or subquery filter
SELECT *
FROM customers
WHERE customer_id IN (
  SELECT customer_id
  FROM orders
  GROUP BY customer_id
  HAVING SUM(total) > (
    SELECT AVG(total) FROM orders
  )
);
```

---

## 6. Existence / Non-existence
**Phrasing:** “Which products have never sold?”  
**Key:** `NOT EXISTS` / `NOT IN` / `EXISTS`  
```sql
-- NOT EXISTS
SELECT *
FROM products p
WHERE NOT EXISTS (
  SELECT 1
  FROM sales s
  WHERE s.product_id = p.product_id
);

-- NOT IN
SELECT *
FROM products
WHERE product_id NOT IN (
  SELECT product_id FROM sales
);
```

---

## 7. Conditional Aggregation
**Phrasing:** “Count paid vs. unpaid invoices per client…”  
**Key:** `SUM(CASE WHEN … END)` or `COUNT(CASE WHEN … END)`  
```sql
SELECT
  client_id,
  SUM(CASE WHEN status = 'paid'   THEN 1 ELSE 0 END) AS paid_count,
  SUM(CASE WHEN status = 'unpaid' THEN 1 ELSE 0 END) AS unpaid_count
FROM invoices
GROUP BY client_id;
```

---

## 8. Pivot / Unpivot
**Phrasing:** “Rotate rows to columns (or vice-versa)…”  
**Key:** `PIVOT` / `UNPIVOT`  
```sql
-- Pivot example in SQL Server
SELECT *
FROM sales
PIVOT (
  SUM(amount) FOR quarter IN ([Q1], [Q2], [Q3], [Q4])
) AS p;

-- Unpivot example
SELECT product_id, quarter, amount
FROM pivoted_table
UNPIVOT (
  amount FOR quarter IN ([Q1], [Q2], [Q3], [Q4])
) AS u;
```

---

## 9. Gap & Islands
**Phrasing:** “Find consecutive date ranges per user…”  
**Key:** `LAG()`/`LEAD()` or recursive CTE  
```sql
-- Identify breaks with LAG()
SELECT
  user_id,
  date,
  CASE
    WHEN DATEADD(day, -1, date) = LAG(date) OVER (PARTITION BY user_id ORDER BY date)
    THEN 0 ELSE 1
  END AS is_new_group
FROM user_dates;
```

---

## 10. Hierarchical / Tree Data
**Phrasing:** “List all subordinates under manager X…”  
**Key:** Recursive CTE (`WITH … UNION ALL …`)  
```sql
WITH Subordinates AS (
  SELECT employee_id, manager_id
  FROM employees
  WHERE manager_id = :root_manager

  UNION ALL

  SELECT e.employee_id, e.manager_id
  FROM employees e
  JOIN Subordinates s ON e.manager_id = s.employee_id
)
SELECT * FROM Subordinates;
```

---

## 11. Pagination
**Phrasing:** “Show page 2 of orders, 50 rows per page…”  
**Key:** `OFFSET … FETCH NEXT …`  
```sql
SELECT *
FROM orders
ORDER BY order_date
OFFSET 50 ROWS FETCH NEXT 50 ROWS ONLY;
```

---

## 12. Date Bucketing
**Phrasing:** “Group sales by month/quarter/year…”  
**Key:** `GROUP BY YEAR(), MONTH(), DATEPART()`  
```sql
-- Monthly totals
SELECT
  YEAR(sale_date)  AS yr,
  MONTH(sale_date) AS mth,
  SUM(amount)      AS total
FROM sales
GROUP BY YEAR(sale_date), MONTH(sale_date)
ORDER BY yr, mth;
```

---

**How to use this sheet:**
1. **Spot trigger words** in the question (e.g. “never,” “top N,” “cumulative,” “page”).
2. **Match** to the pattern above.
3. **Copy & paste** the corresponding snippet and adjust table/column names.
4. **Test** and optimize for your SQL engine if needed.

*Save this file as* `SQL_Patterns_Cheat_Sheet.md` *in your repo!*
