+++
title = "Database Indexing: Beyond the Basics"
date = 2024-12-15
description = "Understanding B-trees, covering indexes, and when NOT to use indexes."
[taxonomies]
tags = ["databases", "performance", "postgresql"]
+++

Every developer knows indexes make queries faster. But do you know *why* they work, when they hurt, and how to design them effectively?

<!-- more -->

## How B-Tree Indexes Work

Most databases use B-tree indexes. Think of it as a sorted phonebook for your data:

```
                    [M]
                   /   \
              [D,G]     [R,U]
             /  |  \    /  |  \
          [A-C][E-F][H-L][N-Q][S-T][V-Z]
                              ↓
                         Leaf nodes
                     (pointers to rows)
```

**Time complexity**: O(log n) for lookups vs O(n) for full table scan.

## The Query Optimizer's Perspective

The database decides whether to use an index based on **selectivity**:

```sql
-- Index WILL be used (low selectivity)
SELECT * FROM users WHERE email = 'john@example.com';

-- Index might NOT be used (high selectivity)
SELECT * FROM users WHERE is_active = true;  -- If 90% are active
```

Rule of thumb: If a query returns more than ~10-15% of rows, a full scan is often faster.

## Covering Indexes

A covering index contains all columns needed by a query, eliminating the need to read the actual table:

```sql
-- Query
SELECT name, email FROM users WHERE status = 'active';

-- Covering index (PostgreSQL)
CREATE INDEX idx_users_status_covering 
ON users (status) INCLUDE (name, email);
```

The query can be satisfied entirely from the index—no table lookup needed.

## Composite Index Column Order Matters

```sql
-- Index on (country, city, name)

-- ✅ Uses index effectively
WHERE country = 'USA'
WHERE country = 'USA' AND city = 'NYC'
WHERE country = 'USA' AND city = 'NYC' AND name = 'John'

-- ⚠️ Uses index partially
WHERE country = 'USA' AND name = 'John'  -- Skips city

-- ❌ Cannot use index
WHERE city = 'NYC'  -- Doesn't start with country
WHERE name = 'John' -- Doesn't start with country
```

Think of it like a phone book: you can't skip to "Smith" without first knowing the region.

## When Indexes Hurt

Every index has costs:

1. **Write overhead**: INSERT, UPDATE, DELETE must update indexes
2. **Storage**: Indexes consume disk space
3. **Memory**: Hot indexes should fit in memory

```sql
-- Don't index these columns:
-- - Low cardinality (boolean, status with few values)
-- - Rarely queried columns
-- - Tables with heavy write traffic and rare reads
```

## Practical Query Analysis

```sql
-- PostgreSQL: See the query plan
EXPLAIN ANALYZE 
SELECT * FROM orders WHERE customer_id = 123;

-- Output to look for:
-- ✅ Index Scan using idx_orders_customer_id
-- ❌ Seq Scan on orders
```

## Key Takeaways

1. **Understand selectivity** - Indexes don't help if you're returning most rows
2. **Use covering indexes** - Include columns to avoid table lookups
3. **Order matters** - Put high-selectivity columns first in composite indexes
4. **Measure everything** - Use `EXPLAIN ANALYZE`, don't guess

Indexing is an art of trade-offs. Master it, and your databases will thank you.
