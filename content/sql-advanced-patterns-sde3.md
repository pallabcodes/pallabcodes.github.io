+++
title = "Advanced SQL Patterns Every Senior Engineer Should Know"
date = 2025-12-24
description = "SQL quirks, gotchas, and production patterns that separate senior engineers—from self-joins to partitioning strategies"
[taxonomies]
tags = ["sql", "database", "production", "best-practices", "postgresql", "security"]
+++

Knowing SQL syntax is one thing. Understanding the subtle quirks, performance traps, and security pitfalls that emerge in production is what separates senior engineers from the rest. This article covers the advanced patterns you need.

<!-- more -->

## Self-Joins and Their Gotchas

### What is a Self-Join?

A self-join is when a table joins with itself. Common use cases: hierarchical data, comparing rows within the same table.

```sql
-- Employee hierarchy (managers are also employees)
CREATE TABLE employees (
    id INT PRIMARY KEY,
    name VARCHAR(100),
    manager_id INT  -- References employees.id
);

-- Self-join to get employee + manager names
SELECT 
    e.name as employee,
    m.name as manager
FROM employees e
LEFT JOIN employees m ON e.manager_id = m.id;
```

**Result:**
```
┌──────────┬──────────┐
│ employee │ manager  │
├──────────┼──────────┤
│ Alice    │ Carol    │
│ Bob      │ Carol    │
│ Carol    │ NULL     │ ← CEO has no manager
└──────────┴──────────┘
```

### Critical Rule: Always Use Different Aliases

```sql
-- ❌ WRONG: Same alias causes ambiguity
SELECT e.name, e.name as manager_name
FROM employees e
JOIN employees e ON e.manager_id = e.id;  -- ERROR: ambiguous

-- ✅ CORRECT: Different aliases
SELECT e.name as employee, m.name as manager_name
FROM employees e
LEFT JOIN employees m ON e.manager_id = m.id;
```

### Finding Gaps in Sequences

```sql
-- Find missing order IDs
SELECT a.order_id + 1 as missing_start
FROM orders a
LEFT JOIN orders b ON a.order_id + 1 = b.order_id
WHERE b.order_id IS NULL
  AND a.order_id < (SELECT MAX(order_id) FROM orders);
```

## Derived Tables: When and Why

### What Are Derived Tables?

Subqueries in the `FROM` clause that act as temporary tables.

```sql
FROM (
    SELECT seller_id, SUM(sales) as total
    FROM sales
    GROUP BY seller_id
) derived_table_name  -- ← Required alias
```

**Aliases are mandatory** in SQL standard—the database needs a name to reference the temporary result set.

### Why Can't Window Functions Go in WHERE?

```sql
-- ❌ This doesn't work:
SELECT name, salary
FROM employees
WHERE ROW_NUMBER() OVER (ORDER BY salary DESC) <= 10;  -- ERROR
```

**Query execution order:**
1. `FROM` + `JOIN`
2. `WHERE` ← Evaluated before window functions
3. `GROUP BY`
4. Window functions ← Happens here
5. `SELECT`
6. `ORDER BY`

**Solution: Use a derived table or CTE:**

```sql
-- ✅ Derived table
SELECT * FROM (
    SELECT name, salary, ROW_NUMBER() OVER (ORDER BY salary DESC) as rn
    FROM employees
) WHERE rn <= 10;

-- ✅ CTE (clearer for complex queries)
WITH ranked AS (
    SELECT name, salary, ROW_NUMBER() OVER (ORDER BY salary DESC) as rn
    FROM employees
)
SELECT * FROM ranked WHERE rn <= 10;
```

## Outer JOIN Pitfalls

### The WHERE Clause Trap

```sql
-- ❌ COMMON MISTAKE: This effectively becomes INNER JOIN
SELECT u.name, o.total
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
WHERE o.total > 100;  -- Filters out NULLs from the LEFT JOIN!
```

**What happens:**
- `LEFT JOIN` includes all users (even without orders)
- `WHERE o.total > 100` filters out rows where `o.total IS NULL`
- Result: Only users with orders >100 (same as INNER JOIN)

**Solutions:**

```sql
-- ✅ Option 1: Move condition to JOIN
SELECT u.name, o.total
FROM users u
LEFT JOIN orders o ON u.id = o.user_id AND o.total > 100;

-- ✅ Option  2: Explicitly handle NULLs
SELECT u.name, o.total
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
WHERE o.total > 100 OR o.total IS NULL;
```

### CROSS JOIN Dangers

```sql
-- ⚠️ DANGER: Cartesian product
SELECT * FROM products p1
CROSS JOIN products p2;  -- 1000 products = 1,000,000 rows!

-- Implicit CROSS JOIN (even more dangerous because it's hidden)
SELECT * FROM products, categories;  -- Same as CROSS JOIN
```

**Use CROSS JOIN only when you truly need all combinations** (e.g., generating date ranges, combinatorial analysis).

```sql
-- ✅ Valid use case: Generate all date-seller combinations
SELECT d.date, s.seller_id
FROM generate_series('2024-01-01'::date, '2024-12-31'::date, '1 day') d(date)
CROSS JOIN sellers s;
```

## Stored Procedures: Best Practices

### PostgreSQL vs MySQL Syntax

**PostgreSQL (PL/pgSQL):**
```sql
CREATE OR REPLACE FUNCTION get_top_sellers(
    p_month DATE DEFAULT CURRENT_DATE,
    p_limit INT DEFAULT 10
) RETURNS TABLE (
    seller_id INT,
    seller_name VARCHAR,
    total_sales DECIMAL
) AS $$
BEGIN
    RETURN QUERY
    SELECT 
        s.id,
        s.seller_name,
        SUM(sa.total_sales)::DECIMAL
    FROM sellers s
    JOIN sales sa ON s.id = sa.seller_id
    WHERE DATE_TRUNC('month', sa.created_at) = DATE_TRUNC('month', p_month)
    GROUP BY s.id, s.seller_name
    ORDER BY SUM(sa.total_sales) DESC
    LIMIT p_limit;
END;
$$ LANGUAGE plpgsql;

-- Usage
SELECT * FROM get_top_sellers('2024-12-01', 5);
```

**MySQL:**
```sql
DELIMITER //
CREATE PROCEDURE get_top_sellers(
    IN p_month DATE,
    IN p_limit INT
)
BEGIN
    SELECT 
        s.id as seller_id,
        s.seller_name,
        SUM(sa.total_sales) as total_sales
    FROM sellers s
    JOIN sales sa ON s.id = sa.seller_id
    WHERE DATE_FORMAT(sa.created_at, '%Y-%m-01') = p_month
    GROUP BY s.id, s.seller_name
    ORDER BY total_sales DESC
    LIMIT p_limit;
END //
DELIMITER ;

-- Usage
CALL get_top_sellers('2024-12-01', 5);
```

### Security: Always Use Parameters

```sql
-- ❌ VULNERABLE: SQL Injection
CREATE FUNCTION unsafe_search(user_input TEXT) RETURNS SETOF products AS $$
BEGIN
    RETURN QUERY EXECUTE 'SELECT * FROM products WHERE name = ''' || user_input || '''';
    -- user_input = "'; DROP TABLE products; --" = disaster
END;
$$ LANGUAGE plpgsql;

-- ✅ SAFE: Parameterized query
CREATE FUNCTION safe_search(user_input TEXT) RETURNS SETOF products AS $$
BEGIN
    RETURN QUERY EXECUTE 'SELECT * FROM products WHERE name = $1' USING user_input;
END;
$$ LANGUAGE plpgsql;
```

**Golden Rule:** Never concatenate user input into SQL strings. Always use parameters.

## Views vs Materialized Views

### Regular Views

```sql
CREATE VIEW active_users AS
SELECT * FROM users WHERE status = 'active';

-- Query always evaluates in real-time
SELECT * FROM active_users WHERE created_at > '2024-01-01';
```

**Characteristics:**
- No storage overhead (just a saved query)
- Always up-to-date
- Performance same as underlying query

### Materialized Views

```sql
CREATE MATERIALIZED VIEW monthly_stats AS
SELECT 
    DATE_TRUNC('month', created_at) as month,
    COUNT(*) as total_orders,
    SUM(amount) as revenue
FROM orders
GROUP BY DATE_TRUNC('month', created_at);

-- Must be refreshed manually or on schedule
REFRESH MATERIALIZED VIEW monthly_stats;

-- Optionally refresh concurrently (allows queries during refresh - PostgreSQL only)
REFRESH MATERIALIZED VIEW CONCURRENTLY monthly_stats;
```

**Characteristics:**
- Stores actual data (disk space required)
- Can be stale (refresh lag)
- Fast queries (pre-computed)
- Requires refresh strategy

**MySQL Note:** No native materialized views. Use triggers + tables:

```sql
-- MySQL workaround
CREATE TABLE monthly_stats_cache (
    month DATE PRIMARY KEY,
    total_orders INT,
    revenue DECIMAL(10,2)
);

-- Trigger to update cache on INSERT
DELIMITER //
CREATE TRIGGER update_monthly_stats
AFTER INSERT ON orders
FOR EACH ROW
BEGIN
    INSERT INTO monthly_stats_cache (month, total_orders, revenue)
    VALUES (DATE_FORMAT(NEW.created_at, '%Y-%m-01'), 1, NEW.amount)
    ON DUPLICATE KEY UPDATE
        total_orders = total_orders + 1,
        revenue = revenue + NEW.amount;
END //
DELIMITER ;
```

## Partitioning Strategies

### Why Partition?

- **Performance**: Query only relevant partitions (partition pruning)
- **Maintenance**: Drop old partitions instead of DELETE
- **Scalability**: Distribute large tables across multiple tablespaces

### PostgreSQL Declarative Partitioning

```sql
-- Range partitioning by date
CREATE TABLE sales (
    id SERIAL,
    seller_id INT,
    total_sales DECIMAL(10,2),
    created_at DATE NOT NULL
) PARTITION BY RANGE (created_at);

-- Create partitions
CREATE TABLE sales_2024_01 PARTITION OF sales
    FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');

CREATE TABLE sales_2024_02 PARTITION OF sales
    FOR VALUES FROM ('2024-02-01') TO ('2024-03-01');

-- Queries automatically use correct partition
SELECT * FROM sales WHERE created_at = '2024-01-15';  
-- Only scans sales_2024_01
```

### Critical Partitioning Rules

**1. Partition key must be in unique constraints:**

```sql
-- ❌ WRONG: id alone isn't enough
CREATE UNIQUE INDEX idx_sales_id ON sales (id);  -- ERROR in partitioned table

-- ✅ CORRECT: Include partition key
CREATE UNIQUE INDEX idx_sales_id_date ON sales (id, created_at);

-- ✅ CORRECT: If id is globally unique without partition key
CREATE UNIQUE INDEX idx_sales_id ON sales (id);  -- Works if id is unique across all partitions
```

**2. All partitions must be defined:**

```sql
-- ⚠️ PROBLEM: No partition for future dates
-- Insert for '2024-03-15' will fail if no partition exists

-- ✅ SOLUTION: Create future partitions in advance or use DEFAULT
CREATE TABLE sales_default PARTITION OF sales DEFAULT;
```

### Hash Partitioning for Even Distribution

```sql
CREATE TABLE orders (
    id SERIAL,
    user_id INT NOT NULL,
    amount DECIMAL(10,2)
) PARTITION BY HASH (user_id);

CREATE TABLE orders_p0 PARTITION OF orders FOR VALUES WITH (MODULUS 4, REMAINDER 0);
CREATE TABLE orders_p1 PARTITION OF orders FOR VALUES WITH (MODULUS 4, REMAINDER 1);
CREATE TABLE orders_p2 PARTITION OF orders FOR VALUES WITH (MODULUS 4, REMAINDER 2);
CREATE TABLE orders_p3 PARTITION OF orders FOR VALUES WITH (MODULUS 4, REMAINDER 3);
```

**Use when:** Even data distribution more important than range queries.

## Index Usage Patterns

### Composite Index Column Order Matters

```sql
CREATE INDEX idx_users_dept_salary ON users (department, salary);

-- ✅ Uses index: Leading column (department) present
SELECT * FROM users WHERE department = 'Engineering';

-- ❌ Doesn't use index: Leading column missing
SELECT * FROM users WHERE salary > 100000;

-- ✅ Uses index: Both columns leverage index
SELECT * FROM users WHERE department = 'Engineering' AND salary > 100000;
```

**Rule**: Index must match query filters from left to right.

### Covering Indexes

```sql
-- Query needs: department, salary, name
CREATE INDEX idx_users_dept_salary_name ON users (department, salary, name);

-- Index-only scan (no table lookup needed)
SELECT department, salary, name 
FROM users 
WHERE department = 'Engineering' AND salary > 100000;
```

### Partial Indexes for Specific Queries

```sql
-- Index only active users
CREATE INDEX idx_active_users ON users (created_at) 
WHERE status = 'active';

-- This query benefits:
SELECT * FROM users WHERE status = 'active' AND created_at > '2024-01-01';
-- Smaller index, faster queries
```

## CTE (Common Table Expression) Patterns

### Basic CTE

```sql
WITH dept_totals AS (
    SELECT department, SUM(salary) as total 
    FROM employees 
    GROUP BY department
)
SELECT * FROM dept_totals WHERE total > 1000000;
```

### Chaining CTEs

```sql
WITH dept_totals AS (
    SELECT department, SUM(salary) as total
    FROM employees
    GROUP BY department
),
dept_avg AS (
    SELECT AVG(total) as company_avg FROM dept_totals
)
SELECT d.*
FROM dept_totals d
CROSS JOIN dept_avg a
WHERE d.total > a.company_avg;
```

### Recursive CTEs

```sql
-- Organizational hierarchy traversal
WITH RECURSIVE org_chart AS (
    -- Base case: CEO
    SELECT id, name, manager_id, 1 as level
    FROM employees
    WHERE manager_id IS NULL
    
    UNION ALL
    
    -- Recursive case: Direct reports
    SELECT e.id, e.name, e.manager_id, oc.level + 1
    FROM employees e
    JOIN org_chart oc ON e.manager_id = oc.id
)
SELECT * FROM org_chart ORDER BY level, name;
```

### CTE Limitations

```sql
-- ❌ Can't reference CTE in separate query
WITH my_cte AS (SELECT * FROM users)
SELECT * FROM my_cte;
-- Later in same script:
SELECT * FROM my_cte;  -- ERROR: CTE doesn't exist here

-- ✅ Must define within each query
-- Or use temporary tables for cross-query reuse
```

## Transaction Isolation Levels

### Read Committed (Default in PostgreSQL, MySQL)

```sql
-- Transaction 1
BEGIN;
SELECT balance FROM accounts WHERE id = 1;  -- Returns 1000
-- Meanwhile...

-- Transaction 2
UPDATE accounts SET balance = 500 WHERE id = 1;
COMMIT;

-- Back to Transaction 1
SELECT balance FROM accounts WHERE id = 1;  -- Returns 500 (non-repeatable read!)
COMMIT;
```

### Serializable (Strictest)

```sql
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
BEGIN;
SELECT * FROM inventory WHERE product_id = 10;  -- stock = 5

-- Transaction 2 tries to update same product
UPDATE inventory SET stock = stock - 3 WHERE product_id = 10;
-- Will block or fail with serialization error

COMMIT;
```

**Trade-offs:**
- **Read Committed**: Fast, allows concurrent updates, non-repeatable reads possible
- **Repeatable Read**: Consistent reads within transaction, phantom reads possible
- **Serializable**: Full isolation, expensive, serialization errors

## Performance Anti-Patterns

### N+1 Query Problem

```sql
-- ❌ BAD: N+1 queries
SELECT * FROM users;  -- 1 query
-- For each user (in application code):
SELECT * FROM orders WHERE user_id = ?;  -- N queries

-- ✅ GOOD: Single query with JOIN
SELECT u.*, o.*
FROM users u
LEFT JOIN orders o ON u.id = o.user_id;
```

### Correlated Subqueries in SELECT

```sql
-- ❌ SLOW: Subquery runs for each row
SELECT 
    name,
    (SELECT AVG(salary) FROM employees e2 WHERE e2.dept = e1.dept) as avg_dept_salary
FROM employees e1;

-- ✅ FAST: Window function or JOIN
SELECT 
    name,
    AVG(salary) OVER (PARTITION BY dept) as avg_dept_salary
FROM employees;
```

### UNION vs UNION ALL

```sql
-- ❌ SLOW: UNION removes duplicates (implicit DISTINCT)
SELECT name FROM customers
UNION
SELECT name FROM suppliers;

-- ✅ FAST: UNION ALL keeps duplicates
SELECT name FROM customers
UNION ALL
SELECT name FROM suppliers;
```

**Use UNION ALL unless you specifically need deduplication.**

## Database-Specific Quirks

### PostgreSQL

```sql
-- Array operations
SELECT * FROM users WHERE 'admin' = ANY(roles);

-- JSON operators
SELECT data->>'name' as name FROM events WHERE data->>'type' = 'click';

-- RETURNING clause
UPDATE users SET status = 'active' WHERE id = 1 RETURNING *;
```

### MySQL

```sql
-- Backticks for reserved words
SELECT `order`, `user` FROM `table`;

-- LIMIT with offset
SELECT * FROM products LIMIT 10 OFFSET 20;

-- INSERT ... ON DUPLICATE KEY UPDATE
INSERT INTO stats (id, count) VALUES (1, 1)
ON DUPLICATE KEY UPDATE count = count + 1;
```

### SQL Server

```sql
-- Square brackets for identifiers
SELECT [order], [user] FROM [table];

-- TOP instead of LIMIT
SELECT TOP 10 * FROM products ORDER BY price DESC;

-- Pagination with OFFSET/FETCH
SELECT * FROM products 
ORDER BY id 
OFFSET 20 ROWS FETCH NEXT 10 ROWS ONLY;
```

## SQL Injection Prevention

### Common Injection Vectors

```sql
-- ❌ VULNERABLE: String concatenation
sql = "SELECT * FROM users WHERE id = " + user_input;
-- user_input = "1 OR 1=1" → Returns all users

-- ❌ VULNERABLE: ORDER BY injection
sql = "SELECT * FROM products ORDER BY " + sort_column;
-- sort_column = "price; DROP TABLE products; --"

-- ✅ SAFE: Parameterized queries (Python example)
cursor.execute("SELECT * FROM users WHERE id = %s", (user_input,))

-- ✅ SAFE: Whitelist for dynamic columns
allowed_columns = ['name', 'price', 'created_at']
if sort_column in allowed_columns:
    sql = f"SELECT * FROM products ORDER BY {sort_column}"
```

### Stored Procedure Safety

```sql
-- ✅ SAFE: Use USING or parameter binding
CREATE FUNCTION search_products(search_term TEXT) ...
BEGIN
    RETURN QUERY EXECUTE 'SELECT * FROM products WHERE name ILIKE $1' 
    USING '%' || search_term || '%';
END;
```

## Key Takeaways

1. **Self-joins require different aliases** to avoid ambiguity
2. **Derived tables are required** when filtering on window functions
3. **WHERE clauses filter out NULLs from LEFT JOINs**—move conditions to JOIN clause
4. **CROSS JOIN creates Cartesian products**—use only when needed
5. **Always use parameterized queries**—never concatenate user input
6. **Views are live queries, materialized views are cached**—choose based on staleness tolerance
7. **Partition key must be in unique constraints** for partitioned tables
8. **Composite index column order matters**—match your query filters left-to-right
9. **CTEs improve readability** but don't persist across queries
10. **UNION ALL is faster than UNION**—unless you need deduplication
11. **Transaction isolation affects consistency**—choose based on requirements  
12. **Profile with EXPLAIN ANALYZE**—always validate performance assumptions

---

**Series complete!** You've now covered:
- [Part 1: SQL Window Functions Core Concepts](/sql-window-functions-part-1)
- [Part 2: Top-N Per Group Problems](/sql-window-functions-part-2)
- [Part 3: Scaling Database Analytics](/sql-window-functions-part-3)
- [Part 4: Advanced SQL Patterns for Senior Engineers](/sql-advanced-patterns-sde3)

These patterns represent production-tested knowledge from systems at Google, Stripe, Uber, Amazon, and beyond. Master them, and you'll handle any SQL challenge that comes your way.
