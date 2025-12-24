+++
title = "Solving Top-N Per Group Problems: A Deep Dive"
date = 2025-12-24
description = "Master the top-N per group pattern with window functions, complete with visualizations and alternative approaches for production systems"
[taxonomies]
tags = ["sql", "database", "window-functions", "top-n", "postgresql", "optimization"]
+++

The "top N per group" problem is one of the most common patterns in analytics: find the top 3 products per category, the 5 highest-rated items per seller, or the most recent orders per customer. Window functions make this elegant and efficient.

<!-- more -->

## The Problem

**Scenario**: You have a sales database and need to find:
1. Top 10 sellers by revenue this month
2. Top 10 products overall
3. Top 3 products for each seller

Traditional approaches lead to complex self-joins or multiple queries. Window functions solve this in a single, readable query.

## Sample Schema

```sql
CREATE TABLE sales (
    id SERIAL PRIMARY KEY,
    seller_id INT,
    product_id INT,
    total_sales DECIMAL(10,2),
    created_at TIMESTAMP
);

CREATE TABLE sellers (
    id INT PRIMARY KEY,
    seller_name VARCHAR(255)
);

CREATE TABLE products (
    id INT PRIMARY KEY,
    product_name VARCHAR(255)
);
```

## The Window Function Solution

Here's the complete solution for finding the top 3 products per seller:

```sql
SELECT *
FROM (
    SELECT 
        sl.id as seller_id,
        sl.seller_name,
        p.id as product_id,
        p.product_name,
        COUNT(*) as total_orders,
        SUM(s.total_sales) as product_sales,
        ROW_NUMBER() OVER (
            PARTITION BY sl.id 
            ORDER BY SUM(s.total_sales) DESC
        ) as product_rank
    FROM sales s 
    JOIN products p ON s.product_id = p.id
    JOIN sellers sl ON s.seller_id = sl.id
    WHERE DATE_TRUNC('month', s.created_at) = DATE_TRUNC('month', NOW())
    GROUP BY sl.id, sl.seller_name, p.id, p.product_name
) ranked
WHERE product_rank <= 3
ORDER BY seller_id, product_rank;
```

## Step-by-Step Execution Flow

Let's visualize how this query executes with sample data.

### Step 1: Raw Sales Data

```
Sales Table (Current Month):
┌────────────┬───────────┬────────────┬─────────────┐
│ created_at │ seller_id │ product_id │ total_sales │
├────────────┼───────────┼────────────┼─────────────┤
│ 2024-12-15 │    1      │     10     │    100      │
│ 2024-12-16 │    1      │     20     │    200      │
│ 2024-12-17 │    1      │     10     │    150      │
│ 2024-12-18 │    2      │     30     │    300      │
│ 2024-12-19 │    1      │     20     │    250      │
│ 2024-12-20 │    2      │     40     │    400      │
│ 2024-12-21 │    1      │     30     │     75      │
│ 2024-12-22 │    2      │     30     │    350      │
└────────────┴───────────┴────────────┴─────────────┘
```

### Step 2: After JOINs and GROUP BY

The query first joins tables and aggregates by seller and product:

```
After JOIN + GROUP BY:
┌───────────┬─────────────┬────────────┬──────────────┬──────────────┬───────────────┐
│ seller_id │ seller_name │ product_id │ product_name │ total_orders │ product_sales │
├───────────┼─────────────┼────────────┼──────────────┼──────────────┼───────────────┤
│    1      │  Seller A   │     10     │  Product X   │      2       │     250       │
│    1      │  Seller A   │     20     │  Product Y   │      2       │     450       │
│    1      │  Seller A   │     30     │  Product Z   │      1       │      75       │
│    2      │  Seller B   │     30     │  Product Z   │      2       │     650       │
│    2      │  Seller B   │     40     │  Product W   │      1       │     400       │
└───────────┴─────────────┴────────────┴──────────────┴──────────────┴───────────────┘
```

### Step 3: Window Function Creates Partitions

`PARTITION BY sl.id` splits the data into independent groups:

```
╔═══════════════════════════════════════════════════════════╗
║ Partition 1: seller_id = 1 (Seller A)                    ║
╠═══════════════════════════════════════════════════════════╣
│ seller_id │ seller_name │ product_id │ product_name │ sales │
├───────────┼─────────────┼────────────┼──────────────┼───────┤
│    1      │  Seller A   │     10     │  Product X   │  250  │
│    1      │  Seller A   │     20     │  Product Y   │  450  │
│    1      │  Seller A   │     30     │  Product Z   │   75  │
╚═══════════════════════════════════════════════════════════╝

╔═══════════════════════════════════════════════════════════╗
║ Partition 2: seller_id = 2 (Seller B)                    ║
╠═══════════════════════════════════════════════════════════╣
│ seller_id │ seller_name │ product_id │ product_name │ sales │
├───────────┼─────────────┼────────────┼──────────────┼───────┤
│    2      │  Seller B   │     30     │  Product Z   │  650  │
│    2      │  Seller B   │     40     │  Product W   │  400  │
╚═══════════════════════════════════════════════════════════╝
```

### Step 4: ORDER BY Sorts Within Each Partition

Within each partition, rows are sorted by `product_sales DESC`:

```
Partition 1 (sorted):
┌───────────┬─────────────┬────────────┬──────────────┬───────────────┐
│ seller_id │ seller_name │ product_id │ product_name │ product_sales │
├───────────┼─────────────┼────────────┼──────────────┼───────────────┤
│    1      │  Seller A   │     20     │  Product Y   │     450       │ ← Highest
│    1      │  Seller A   │     10     │  Product X   │     250       │
│    1      │  Seller A   │     30     │  Product Z   │      75       │ ← Lowest
└───────────┴─────────────┴────────────┴──────────────┴───────────────┘

Partition 2 (sorted):
┌───────────┬─────────────┬────────────┬──────────────┬───────────────┐
│ seller_id │ seller_name │ product_id │ product_name │ product_sales │
├───────────┼─────────────┼────────────┼──────────────┼───────────────┤
│    2      │  Seller B   │     30     │  Product Z   │     650       │ ← Highest
│    2      │  Seller B   │     40     │  Product W   │     400       │ ← Lowest
└───────────┴─────────────┴────────────┴──────────────┴───────────────┘
```

### Step 5: ROW_NUMBER() Assigns Ranks

`ROW_NUMBER()` assigns sequential integers within each partition:

```
After ROW_NUMBER():
┌───────────┬─────────────┬────────────┬──────────────┬───────┬──────────────┐
│ seller_id │ seller_name │ product_id │ product_name │ sales │ product_rank │
├───────────┼─────────────┼────────────┼──────────────┼───────┼──────────────┤
│    1      │  Seller A   │     20     │  Product Y   │  450  │      1       │
│    1      │  Seller A   │     10     │  Product X   │  250  │      2       │
│    1      │  Seller A   │     30     │  Product Z   │   75  │      3       │
│    2      │  Seller B   │     30     │  Product Z   │  650  │      1       │ ← Rank resets
│    2      │  Seller B   │     40     │  Product W   │  400  │      2       │
└───────────┴─────────────┴────────────┴──────────────┴───────┴──────────────┘
```

**Key insight**: Ranking resets to 1 for each seller because of `PARTITION BY`.

### Step 6: Filter WHERE product_rank <= 3

The outer query filters for top 3 products per seller:

```
FINAL RESULT:
┌───────────┬─────────────┬────────────┬──────────────┬───────┬──────────────┐
│ seller_id │ seller_name │ product_id │ product_name │ sales │ product_rank │
├───────────┼─────────────┼────────────┼──────────────┼───────┼──────────────┤
│    1      │  Seller A   │     20     │  Product Y   │  450  │      1       │
│    1      │  Seller A   │     10     │  Product X   │  250  │      2       │
│    1      │  Seller A   │     30     │  Product Z   │   75  │      3       │
│    2      │  Seller B   │     30     │  Product Z   │  650  │      1       │
│    2      │  Seller B   │     40     │  Product W   │  400  │      2       │
└───────────┴─────────────┴────────────┴──────────────┴───────┴──────────────┘
```

## Understanding Derived Tables

### What is `ranked`?

```sql
FROM (
    SELECT ...
) ranked  -- ← This is a derived table alias
```

`ranked` is a **required alias** for the subquery. In SQL, any subquery in the `FROM` clause must have an alias—it's part of the SQL standard.

**Common aliases for this pattern:**
- `ranked` (descriptive of the ranking operation)
- `top_products`
- `results`
- `r` (concise)

The name is arbitrary—choose what makes your code readable.

### Why Can't You Use Window Functions in WHERE?

```sql
-- ❌ This doesn't work:
SELECT ...
FROM sales
WHERE ROW_NUMBER() OVER (...) <= 3;  -- ERROR
```

**Why**: The `WHERE` clause is evaluated before window functions. The query execution order is:

1. `FROM` + `JOIN`
2. `WHERE`
3. `GROUP BY`
4. Window functions
5. `SELECT`
6. `ORDER BY`

**Solution**: Use a subquery or CTE:

```sql
-- ✅ Subquery approach:
SELECT * FROM (
    SELECT *, ROW_NUMBER() OVER (...) as rn
    FROM sales
) WHERE rn <= 3;

-- ✅ CTE approach:
WITH ranked AS (
    SELECT *, ROW_NUMBER() OVER (...) as rn
    FROM sales
)
SELECT * FROM ranked WHERE rn <= 3;
```

## Alternative Approaches

### Approach 1: Multiple Separate Queries

```sql
-- Query 1: Top sellers
SELECT seller_id, seller_name, SUM(total_sales) as total
FROM sales s JOIN sellers sl ON s.seller_id = sl.id
WHERE DATE_TRUNC('month', created_at) = DATE_TRUNC('month', NOW())
GROUP BY seller_id, seller_name
ORDER BY total DESC
LIMIT 10;

-- Query 2: Top products
SELECT product_id, product_name, SUM(total_sales) as total
FROM sales s JOIN products p ON s.product_id = p.id
WHERE DATE_TRUNC('month', created_at) = DATE_TRUNC('month', NOW())
GROUP BY product_id, product_name
ORDER BY total DESC
LIMIT 10;

-- Query 3: Top products per seller (requires window function anyway)
```

**Pros:**
- Simple and straightforward
- Easy to cache individually
- Can run in parallel

**Cons:**
- Multiple round trips to database
- Potential data inconsistency between queries
- Can't get all data in one transaction

### Approach 2: Self-Join

```sql
SELECT s1.seller_id, s1.product_id, s1.product_sales
FROM (
    SELECT seller_id, product_id, SUM(total_sales) as product_sales
    FROM sales
    WHERE DATE_TRUNC('month', created_at) = DATE_TRUNC('month', NOW())
    GROUP BY seller_id, product_id
) s1
LEFT JOIN (
    SELECT seller_id, product_id, SUM(total_sales) as product_sales
    FROM sales
    WHERE DATE_TRUNC('month', created_at) = DATE_TRUNC('month', NOW())
    GROUP BY seller_id, product_id
) s2 ON s1.seller_id = s2.seller_id AND s1.product_sales < s2.product_sales
GROUP BY s1.seller_id, s1.product_id, s1.product_sales
HAVING COUNT(s2.product_id) < 3
ORDER BY s1.seller_id, s1.product_sales DESC;
```

**Pros:**
- Works in older SQL databases without window functions

**Cons:**
- Complex and hard to read
- Poor performance (O(n²) comparisons)
- Difficult to maintain

**Verdict**: Avoid unless you're stuck on SQL Server 2000 or similar legacy systems.

### Approach 3: Correlated Subqueries

Avoid in production—extremely slow as subquery executes for each row.

### Approach 4: Using LATERAL Joins (PostgreSQL)

```sql
SELECT sl.id, sl.seller_name, top_products.*
FROM sellers sl
CROSS JOIN LATERAL (
    SELECT p.product_name, SUM(s.total_sales) as sales
    FROM sales s
    JOIN products p ON s.product_id = p.id
    WHERE s.seller_id = sl.id
      AND DATE_TRUNC('month', s.created_at) = DATE_TRUNC('month', NOW())
    GROUP BY p.id, p.product_name
    ORDER BY sales DESC
    LIMIT 3
) top_products;
```

**Pros:**
- Readable for those familiar with lateral joins
- Can be efficient with proper indexes

**Cons:**
- PostgreSQL-specific syntax
- Less flexible than window functions

## Choosing RANK() vs ROW_NUMBER() vs DENSE_RANK()

For top-N problems, the choice matters when there are ties:

```sql
-- Sales data with ties:
┌───────────┬────────────┬───────┐
│ seller_id │ product_id │ sales │
├───────────┼────────────┼───────┤
│    1      │     10     │  500  │
│    1      │     20     │  500  │ ← Tie
│    1      │     30     │  400  │
│    1      │     40     │  300  │
└───────────┴────────────┴───────┘

ROW_NUMBER() - Arbitrary ordering for ties:
┌────────────┬───────┬──────┐
│ product_id │ sales │ rank │
├────────────┼───────┼──────┤
│     10     │  500  │  1   │
│     20     │  500  │  2   │ ← Tie broken arbitrarily
│     30     │  400  │  3   │
└────────────┴───────┴──────┘

RANK() - Gaps after ties:
┌────────────┬───────┬──────┐
│ product_id │ sales │ rank │
├────────────┼───────┼──────┤
│     10     │  500  │  1   │
│     20     │  500  │  1   │ ← Same rank
│     30     │  400  │  3   │ ← Skipped 2
└────────────┴───────┴──────┘

DENSE_RANK() - No gaps:
┌────────────┬───────┬──────┐
│ product_id │ sales │ rank │
├────────────┼───────┼──────┤
│     10     │  500  │  1   │
│     20     │  500  │  1   │ ← Same rank
│     30     │  400  │  2   │ ← No gap
└────────────┴───────┴──────┘
```

**For top-N filtering:**
- **ROW_NUMBER()**: Guarantees exactly N results per group
- **RANK()**: May return >N results if ties at boundary
- **DENSE_RANK()**: May return >N results if ties

**Recommendation**: Use `ROW_NUMBER()` unless ties must share the same rank.

## Performance Optimization

### 1. Index Partition and Order Columns

```sql
CREATE INDEX idx_sales_seller_sales 
ON sales (seller_id, total_sales DESC, created_at);
```

This index supports:
- `PARTITION BY seller_id`
- `ORDER BY total_sales DESC`
- `WHERE created_at` filtering

### 2. Use CTEs for Readability

```sql
WITH monthly_sales AS (
    SELECT seller_id, product_id, SUM(total_sales) as sales
    FROM sales
    WHERE DATE_TRUNC('month', created_at) = DATE_TRUNC('month', NOW())
    GROUP BY seller_id, product_id
),
ranked_products AS (
    SELECT 
        *,
        ROW_NUMBER() OVER (PARTITION BY seller_id ORDER BY sales DESC) as rank
    FROM monthly_sales
)
SELECT * FROM ranked_products WHERE rank <= 3;
```

### 3. Filter Early

```sql
-- ✅ Good: Filter before window function
SELECT * FROM (
    SELECT 
        *,
        ROW_NUMBER() OVER (PARTITION BY seller_id ORDER BY sales DESC) as rn
    FROM sales
    WHERE created_at >= '2024-12-01'  -- Filter first
) WHERE rn <= 3;

-- ❌ Bad: Window function on full table
SELECT * FROM (
    SELECT 
        *,
        ROW_NUMBER() OVER (PARTITION BY seller_id ORDER BY sales DESC) as rn
    FROM sales  -- All historical data
) WHERE rn <= 3 AND created_at >= '2024-12-01';
```

### 4. Profile with EXPLAIN ANALYZE

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM (
    SELECT *, ROW_NUMBER() OVER (PARTITION BY seller_id ORDER BY sales DESC) as rn
    FROM sales
    WHERE created_at >= '2024-12-01'
) WHERE rn <= 3;
```

Look for:
- WindowAgg node execution time
- Sort operations
- Index usage

## Common Patterns

### Top N Recent Records Per User

```sql
SELECT * FROM (
    SELECT 
        user_id,
        order_id,
        created_at,
        ROW_NUMBER() OVER (
            PARTITION BY user_id 
            ORDER BY created_at DESC
        ) as recency_rank
    FROM orders
) WHERE recency_rank <= 5;
```

### Top N by Multiple Criteria

```sql
SELECT * FROM (
    SELECT 
        category_id,
        product_id,
        rating,
        sales,
        ROW_NUMBER() OVER (
            PARTITION BY category_id 
            ORDER BY rating DESC, sales DESC  -- Tiebreaker
        ) as rank
    FROM products
) WHERE rank <= 10;
```

### Top N with Percentage Threshold

```sql
WITH category_totals AS (
    SELECT 
        category_id,
        product_id,
        sales,
        SUM(sales) OVER (PARTITION BY category_id) as category_total
    FROM products
)
SELECT * FROM (
    SELECT 
        *,
        (sales::FLOAT / category_total * 100) as pct_of_category,
        ROW_NUMBER() OVER (PARTITION BY category_id ORDER BY sales DESC) as rank
    FROM category_totals
) WHERE rank <= 5 OR pct_of_category >= 20;  -- Top 5 or 20%+ of category
```

## Key Takeaways

1. **Window functions are the elegant solution** for top-N per group problems
2. **PARTITION BY creates independent ranking contexts** for each group
3. **Derived tables are required** to filter on window function results
4. **ROW_NUMBER() guarantees exactly N results**, RANK/DENSE_RANK may return more with ties
5. **Index partition and order columns** for best performance
6. **Filter data early** before applying window functions
7. **CTEs improve readability** for complex queries

---

**Next in this series**: [Part 3: Scaling Database Analytics](/sql-window-functions-part-3) - When window functions aren't enough and how to scale from millions to billions of records using materialized views, columnar databases, and distributed systems.
