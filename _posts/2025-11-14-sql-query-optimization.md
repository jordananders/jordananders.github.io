---
layout: default
title:  "SQL Query Optimization Techniques"
date:   2025-11-14 20:00:00
categories: Database SQL Performance
---

I've optimized slow SQL queries that brought production systems to their knees. Queries that took 30 seconds reduced to 50 milliseconds. The difference between a slow query and a fast one often comes down to a few key techniques.

Here's what actually works for SQL query optimization.

## The Most Important Rule

**Fix the query, not the hardware.**

Adding more RAM or CPU power masks the problem. A poorly written query will eventually break no matter how much hardware you throw at it.

## Indexing: The Foundation

Missing indexes are the #1 reason for slow queries. The database optimizer defaults to table scans, which kill performance.

### What is an Index?

Think of it like a book index. Without an index, you read every page to find what you need (table scan). With an index, you jump directly to the right page.

### When to Add an Index

Index columns that appear in:
- `WHERE` clauses
- `JOIN` conditions
- `ORDER BY` clauses
- `GROUP BY` clauses

**Example query:**
```sql
-- Slow without index
SELECT * FROM users WHERE email = 'user@example.com';

-- Create index
CREATE INDEX idx_users_email ON users(email);

-- Now fast
```

### Covering Indexes

A covering index includes all columns referenced in a query, allowing the database to satisfy the query entirely from the index.

```sql
-- Query
SELECT user_id, email, name
FROM users
WHERE email = 'user@example.com';

-- Covering index (includes all columns in SELECT)
CREATE INDEX idx_users_email_covering ON users(email, user_id, name);

-- Database doesn't need to touch the table, just the index
```

### Composite Indexes

For queries filtering on multiple columns, use composite indexes. **Order matters!**

```sql
-- Query
SELECT * FROM orders
WHERE customer_id = 123 AND status = 'pending';

-- Good: most selective column first
CREATE INDEX idx_orders_customer_status ON orders(customer_id, status);

-- This index helps:
-- WHERE customer_id = 123
-- WHERE customer_id = 123 AND status = 'pending'

-- This index does NOT help:
-- WHERE status = 'pending' (doesn't use first column)
```

### Index Gotchas

**Functions disable indexes:**
```sql
-- BAD: Function on indexed column = no index used
SELECT * FROM users WHERE UPPER(email) = 'USER@EXAMPLE.COM';

-- GOOD: Store email in consistent case, query without function
SELECT * FROM users WHERE email = 'user@example.com';
```

**Leading wildcards disable indexes:**
```sql
-- BAD: Can't use index
SELECT * FROM users WHERE email LIKE '%@example.com';

-- GOOD: Can use index
SELECT * FROM users WHERE email LIKE 'user@%';
```

**Too many indexes slow down writes:**
- Every `INSERT`, `UPDATE`, `DELETE` must update all indexes
- Index only what you actually query

## SELECT Only What You Need

### Don't Use SELECT *

```sql
-- BAD: Returns all 50 columns
SELECT * FROM users WHERE id = 123;

-- GOOD: Return only needed columns
SELECT id, email, name FROM users WHERE id = 123;
```

**Why it matters:**
- Less data transferred over network
- Smaller result sets
- Can use covering indexes
- Clearer code

### Limit Results

```sql
-- Return only what you need
SELECT id, name FROM users
WHERE status = 'active'
LIMIT 100;

-- Or with pagination
LIMIT 100 OFFSET 200;
```

## JOIN Optimization

### Choose the Right JOIN Type

**INNER JOIN** (most efficient):
```sql
-- Only returns matching rows
SELECT orders.id, users.name
FROM orders
INNER JOIN users ON orders.user_id = users.id;
```

**LEFT JOIN**:
```sql
-- Returns all orders, even without user
SELECT orders.id, users.name
FROM orders
LEFT JOIN users ON orders.user_id = users.id;
```

### Index JOIN Columns

```sql
-- Slow without indexes
SELECT o.id, u.name
FROM orders o
JOIN users u ON o.user_id = u.id;

-- Fast with indexes
CREATE INDEX idx_orders_user_id ON orders(user_id);
CREATE INDEX idx_users_id ON users(id);  -- Usually exists as PK
```

### Avoid Cartesian Products

```sql
-- BAD: No JOIN condition = every row matched with every row
SELECT * FROM users, orders;
-- 1,000 users × 10,000 orders = 10,000,000 rows!

-- GOOD: Proper JOIN
SELECT * FROM users
JOIN orders ON users.id = orders.user_id;
```

### Small Table First (Sometimes)

Some databases optimize better when the smaller table is listed first in JOINs:

```sql
-- Small table (100 rows) first
SELECT c.name, o.total
FROM categories c
JOIN orders o ON o.category_id = c.id;
```

## WHERE Clause Optimization

### Use EXISTS Instead of IN for Subqueries

```sql
-- SLOW: IN with subquery
SELECT * FROM users
WHERE id IN (SELECT user_id FROM orders WHERE total > 100);

-- FAST: EXISTS (stops at first match)
SELECT * FROM users u
WHERE EXISTS (
    SELECT 1 FROM orders o
    WHERE o.user_id = u.id AND o.total > 100
);
```

### Avoid OR, Use UNION ALL

```sql
-- SLOW: OR prevents index usage
SELECT * FROM users
WHERE first_name = 'John' OR last_name = 'Doe';

-- FAST: UNION ALL (if no duplicates needed)
SELECT * FROM users WHERE first_name = 'John'
UNION ALL
SELECT * FROM users WHERE last_name = 'Doe';

-- Or UNION if you need to remove duplicates
```

### Use BETWEEN Instead of >= AND <=

```sql
-- GOOD: Easy to read
SELECT * FROM orders
WHERE created_at >= '2025-01-01' AND created_at < '2025-02-01';

-- BETTER: BETWEEN (more efficient)
SELECT * FROM orders
WHERE created_at BETWEEN '2025-01-01' AND '2025-01-31';
```

## Aggregation Optimization

### Filter Before Aggregating

```sql
-- BAD: Aggregate everything, then filter
SELECT user_id, COUNT(*)
FROM orders
GROUP BY user_id
HAVING COUNT(*) > 10;

-- BETTER: Filter first if possible
SELECT user_id, COUNT(*)
FROM orders
WHERE created_at > '2025-01-01'
GROUP BY user_id
HAVING COUNT(*) > 10;
```

### Use COUNT(*) Not COUNT(column)

```sql
-- SLOW: Has to check each column value for NULL
SELECT COUNT(email) FROM users;

-- FAST: Just counts rows
SELECT COUNT(*) FROM users;

-- If you need non-NULL count, be explicit
SELECT COUNT(*) FROM users WHERE email IS NOT NULL;
```

## UNION vs UNION ALL

```sql
-- SLOW: UNION removes duplicates (requires sorting)
SELECT email FROM users_active
UNION
SELECT email FROM users_inactive;

-- FAST: UNION ALL (no duplicate removal)
SELECT email FROM users_active
UNION ALL
SELECT email FROM users_inactive;
```

Only use `UNION` if you actually need duplicate removal.

## Subquery Optimization

### Avoid Correlated Subqueries

```sql
-- SLOW: Subquery runs for EVERY row
SELECT u.name, (
    SELECT COUNT(*) FROM orders o
    WHERE o.user_id = u.id
) as order_count
FROM users u;

-- FAST: JOIN instead
SELECT u.name, COUNT(o.id) as order_count
FROM users u
LEFT JOIN orders o ON o.user_id = u.id
GROUP BY u.id, u.name;
```

### Use WITH (CTEs) for Readability

```sql
-- Common Table Expressions make complex queries readable
WITH high_value_orders AS (
    SELECT user_id, SUM(total) as total_spent
    FROM orders
    WHERE total > 100
    GROUP BY user_id
)
SELECT u.name, h.total_spent
FROM users u
JOIN high_value_orders h ON h.user_id = u.id;
```

## Avoid These Common Mistakes

### 1. Functions in WHERE Clause

```sql
-- BAD: Function prevents index use
SELECT * FROM orders WHERE YEAR(created_at) = 2025;

-- GOOD: Rewrite without function
SELECT * FROM orders
WHERE created_at >= '2025-01-01' AND created_at < '2026-01-01';
```

### 2. Implicit Type Conversion

```sql
-- BAD: id is INT, but querying with string
SELECT * FROM users WHERE id = '123';

-- GOOD: Use correct type
SELECT * FROM users WHERE id = 123;
```

### 3. NOT IN with NULLs

```sql
-- DANGEROUS: NOT IN doesn't work as expected with NULLs
SELECT * FROM users WHERE id NOT IN (SELECT user_id FROM orders);
-- If ANY user_id is NULL, returns 0 rows!

-- SAFE: Use NOT EXISTS
SELECT * FROM users u
WHERE NOT EXISTS (SELECT 1 FROM orders o WHERE o.user_id = u.id);
```

### 4. Using DISTINCT to Hide Problems

```sql
-- BAD: DISTINCT hides incorrect JOIN
SELECT DISTINCT u.name
FROM users u
JOIN orders o ON u.id = o.user_id;

-- GOOD: Fix the JOIN or use proper aggregation
SELECT u.name
FROM users u
WHERE EXISTS (SELECT 1 FROM orders o WHERE o.user_id = u.id);
```

## Pagination Best Practices

### Avoid Large OFFSETs

```sql
-- SLOW: Skips 1 million rows
SELECT * FROM users
ORDER BY id
LIMIT 100 OFFSET 1000000;

-- FAST: Use WHERE to filter
SELECT * FROM users
WHERE id > 1000000
ORDER BY id
LIMIT 100;
```

### Cursor-Based Pagination

```sql
-- First page
SELECT * FROM posts
WHERE published = true
ORDER BY created_at DESC, id DESC
LIMIT 20;

-- Next page (using last row's values)
SELECT * FROM posts
WHERE published = true
  AND (created_at, id) < ('2025-01-15 10:00:00', 12345)
ORDER BY created_at DESC, id DESC
LIMIT 20;
```

## Analyzing Query Performance

### EXPLAIN Your Queries

```sql
-- PostgreSQL / MySQL
EXPLAIN SELECT * FROM users WHERE email = 'user@example.com';

-- SQL Server
SET SHOWPLAN_TEXT ON;
GO
SELECT * FROM users WHERE email = 'user@example.com';
GO
SET SHOWPLAN_TEXT OFF;
```

**Look for:**
- **Seq Scan / Table Scan**: No index used, scans entire table (bad for large tables)
- **Index Scan**: Uses index (good)
- **Index Seek**: Even better (jumps to exact index location)
- **Nested Loop**: Efficient for small datasets
- **Hash Join**: Efficient for large datasets
- **Cost numbers**: Lower is better

### Execution Statistics

```sql
-- PostgreSQL: Include actual execution stats
EXPLAIN ANALYZE SELECT * FROM users WHERE email = 'user@example.com';

-- SQL Server: Show actual execution plan
SET STATISTICS TIME ON;
SET STATISTICS IO ON;
SELECT * FROM users WHERE email = 'user@example.com';
```

## Database-Specific Tips

### PostgreSQL

```sql
-- Update statistics for better query plans
ANALYZE users;

-- Vacuum to reclaim space and update statistics
VACUUM ANALYZE users;

-- Use EXPLAIN ANALYZE to see actual performance
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM users;
```

### MySQL

```sql
-- Optimize table (defragment and update stats)
OPTIMIZE TABLE users;

-- Force index usage (if optimizer chooses wrong index)
SELECT * FROM users FORCE INDEX (idx_email) WHERE email = 'user@example.com';

-- Use query cache (if enabled)
SELECT SQL_CACHE * FROM users WHERE id = 123;
```

### SQL Server

```sql
-- Update statistics
UPDATE STATISTICS users;

-- Rebuild indexes
ALTER INDEX ALL ON users REBUILD;

-- Query hints
SELECT * FROM users WITH (INDEX(idx_email)) WHERE email = 'user@example.com';
```

## 2025 Trends: AI-Driven Optimization

Major databases now include AI-powered optimization:

**Azure SQL Database:**
- Automatic tuning based on AI/ML
- Recommends indexes automatically
- Adapts query plans based on actual execution patterns

**Oracle Autonomous Database:**
- Self-optimizing queries
- Automatic index creation
- Predictive performance tuning

**PostgreSQL Extensions:**
- `pg_stat_statements` for query analysis
- `auto_explain` for automatic EXPLAIN logging

## My Optimization Checklist

When a query is slow, I check these in order:

1. **[ ] Is there an index on WHERE/JOIN columns?**
   - Missing indexes = #1 performance killer

2. **[ ] Is SELECT * being used?**
   - Select only needed columns

3. **[ ] Are there functions in WHERE clause?**
   - Rewrite to avoid functions on indexed columns

4. **[ ] Check EXPLAIN output**
   - Look for table scans, high cost

5. **[ ] Is pagination using large OFFSET?**
   - Use cursor-based pagination instead

6. **[ ] Are there correlated subqueries?**
   - Rewrite as JOINs

7. **[ ] Is DISTINCT hiding a problem?**
   - Fix the JOIN or use proper GROUP BY

8. **[ ] Are statistics up to date?**
   - Run ANALYZE/UPDATE STATISTICS

## Real-World Example

**Before optimization:**
```sql
SELECT * FROM orders o
WHERE YEAR(o.created_at) = 2025
  AND o.status IN (
    SELECT status FROM order_statuses WHERE active = 1
  )
ORDER BY o.created_at DESC
LIMIT 100 OFFSET 1000;

-- Execution time: 28 seconds
```

**After optimization:**
```sql
-- Create indexes
CREATE INDEX idx_orders_created_status ON orders(created_at DESC, status);
CREATE INDEX idx_order_statuses_active ON order_statuses(active);

-- Optimized query
SELECT o.id, o.customer_id, o.total, o.status, o.created_at
FROM orders o
WHERE o.created_at >= '2025-01-01'
  AND o.created_at < '2026-01-01'
  AND EXISTS (
    SELECT 1 FROM order_statuses os
    WHERE os.status = o.status AND os.active = 1
  )
  AND o.created_at < (
    SELECT created_at FROM orders WHERE id = 123456  -- last_seen_id
  )
ORDER BY o.created_at DESC
LIMIT 100;

-- Execution time: 45 milliseconds
```

**Changes made:**
- Removed `SELECT *`
- Removed `YEAR()` function
- Changed `IN` to `EXISTS`
- Added appropriate indexes
- Replaced `OFFSET` with cursor pagination
- Used explicit date range

## Resources

- [SQL Query Optimization: 15 Techniques - DataCamp](https://www.datacamp.com/blog/sql-query-optimization)
- [12 SQL Query Optimization Techniques - ThoughtSpot](https://www.thoughtspot.com/data-trends/data-modeling/optimizing-sql-queries)
- [SQL Query Optimization Tips - Devart](https://blog.devart.com/how-to-optimize-sql-query.html)
- [Advanced Indexing Techniques 2025 - GUVI](https://www.guvi.in/blog/advanced-indexing-techniques-for-database/)
- [AI SQL Query Optimization 2025 - Syncfusion](https://www.syncfusion.com/blogs/post/ai-sql-query-optimization-2025/)

---

*Got a slow query to optimize? [Let me know](mailto:jordan@jordananderson.us).*
