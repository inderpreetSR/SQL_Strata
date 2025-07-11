# PostgreSQL SQL Patterns Cheat Sheet

A quick reference to the most common SQL interview/exam patterns in PostgreSQL. Spot the trigger phrases in the question, then jump to the matching syntax.

---

## 1. Find Duplicates
**Phrasing:** “Which rows occur more than once…?”  
**Key:** GROUP BY + HAVING COUNT(*) > 1  
    SELECT col1, col2, COUNT(*) AS cnt
    FROM your_table
    GROUP BY col1, col2
    HAVING COUNT(*) > 1;

---

## 2. Top N per Group
**Phrasing:** “Give the top 3 salaries per department…”  
**Key:** ROW_NUMBER() or DISTINCT ON  
    -- Using ROW_NUMBER()
    SELECT *
    FROM (
        SELECT
            *,
            ROW_NUMBER() OVER (PARTITION BY department ORDER BY salary DESC) AS rn
        FROM employees
    ) t
    WHERE rn <= 3;

    -- Fast top-1 per group
    SELECT DISTINCT ON (department)
        department, employee_id, salary
    FROM employees
    ORDER BY department, salary DESC;

---

## 3. Nth Highest Value
**Phrasing:** “Who has the 2nd highest score…?”  
**Key:** DENSE_RANK() or subquery  
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
    FROM scores s
    WHERE s.score = (
        SELECT MAX(score)
        FROM scores
        WHERE score < (SELECT MAX(score) FROM scores)
    );

---

## 4. Running Total / Cumulative Sum
**Phrasing:** “Show cumulative sales by date…”  
**Key:** SUM() OVER (ORDER BY date ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)  
    SELECT
        sale_date,
        amount,
        SUM(amount) OVER (
            ORDER BY sale_date
            ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
        ) AS running_total
    FROM sales
    ORDER BY sale_date;

---

## 5. Filter on Aggregate
**Phrasing:** “Which customers spent above the average order value?”  
**Key:** HAVING or WHERE … IN (subquery)  
    -- HAVING on grouped totals
    SELECT customer_id, SUM(total) AS total_spent
    FROM orders
    GROUP BY customer_id
    HAVING SUM(total) > (SELECT AVG(total) FROM orders);

    -- Or subquery filter
    SELECT *
    FROM customers
    WHERE customer_id IN (
        SELECT customer_id
        FROM orders
        GROUP BY customer_id
        HAVING SUM(total) > (SELECT AVG(total) FROM orders)
    );

---

## 6. Existence / Non-existence
**Phrasing:** “Which products have never sold?”  
**Key:** NOT EXISTS / NOT IN  
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
    WHERE product_id NOT IN (SELECT product_id FROM sales);

---

## 7. Conditional Aggregation
**Phrasing:** “Count paid vs. unpaid invoices per client…”  
**Key:** FILTER clause  
    SELECT
        client_id,
        COUNT(*) FILTER (WHERE status = 'paid')   AS paid_count,
        COUNT(*) FILTER (WHERE status = 'unpaid') AS unpaid_count
    FROM invoices
    GROUP BY client_id;

---

## 8. Pivot / Unpivot
**Phrasing:** “Rotate rows to columns…”  
**Key:** CASE aggregation or crosstab()  
    SELECT
        product_id,
        SUM(CASE WHEN quarter = 'Q1' THEN amount ELSE 0 END) AS Q1,
        SUM(CASE WHEN quarter = 'Q2' THEN amount ELSE 0 END) AS Q2,
        SUM(CASE WHEN quarter = 'Q3' THEN amount ELSE 0 END) AS Q3,
        SUM(CASE WHEN quarter = 'Q4' THEN amount ELSE 0 END) AS Q4
    FROM sales
    GROUP BY product_id;

---

## 9. Gap & Islands
**Phrasing:** “Find consecutive date ranges per user…”  
**Key:** LAG()/LEAD() or recursive CTE  
    SELECT
        user_id,
        date,
        CASE
            WHEN LAG(date) OVER (PARTITION BY user_id ORDER BY date) = date - INTERVAL '1 day'
            THEN 0 ELSE 1
        END AS is_new_group
    FROM user_dates;

---

## 10. Hierarchical / Tree Data
**Phrasing:** “List all subordinates under manager X…”  
**Key:** Recursive CTE  
    WITH RECURSIVE subordinates AS (
        SELECT employee_id, manager_id
        FROM employees
        WHERE manager_id = :root_manager

        UNION ALL

        SELECT e.employee_id, e.manager_id
        FROM employee
