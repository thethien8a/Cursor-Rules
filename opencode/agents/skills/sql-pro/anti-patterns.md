# SQL Anti-Patterns Reference

Comprehensive guide to SQL anti-patterns with detection and solutions.

## Query Performance Anti-Patterns

### 1. SELECT * (Wildcard Selection)

```sql
-- ❌ Bad: Fetches all columns
SELECT * FROM orders WHERE status = 'pending';

-- ✅ Good: Explicit columns
SELECT order_id, customer_id, total_amount, created_at
FROM orders WHERE status = 'pending';
```

**Why it matters**: 
- Unnecessary I/O for unused columns
- Breaks code when schema changes
- Prevents covering index optimization

### 2. N+1 Query Pattern

```sql
-- ❌ Bad: Loop queries (N+1 problem)
-- Python pseudocode:
-- for order in orders:
--     items = db.query("SELECT * FROM items WHERE order_id = ?", order.id)

-- ✅ Good: Single JOIN query
SELECT o.*, i.*
FROM orders o
LEFT JOIN items i ON o.id = i.order_id
WHERE o.status = 'pending';

-- ✅ Alternative: Batch IN query
SELECT * FROM items WHERE order_id IN (1, 2, 3, 4, 5);
```

### 3. Functions on Indexed Columns

```sql
-- ❌ Bad: Function prevents index usage
SELECT * FROM orders WHERE YEAR(created_at) = 2025;
SELECT * FROM users WHERE LOWER(email) = 'user@example.com';
SELECT * FROM products WHERE price + tax > 100;

-- ✅ Good: Sargable predicates
SELECT * FROM orders 
WHERE created_at >= '2025-01-01' AND created_at < '2026-01-01';

SELECT * FROM users WHERE email = 'user@example.com';  -- Use case-insensitive collation

SELECT * FROM products WHERE price > 100 - tax;  -- Or use generated column
```

### 4. OR Conditions on Different Columns

```sql
-- ❌ Bad: OR prevents index intersection
SELECT * FROM products 
WHERE category_id = 5 OR brand_id = 10;

-- ✅ Good: UNION ALL
SELECT * FROM products WHERE category_id = 5
UNION ALL
SELECT * FROM products WHERE brand_id = 10 AND category_id != 5;

-- ✅ Alternative: Separate queries if cardinality is low
```

### 5. LIKE with Leading Wildcard

```sql
-- ❌ Bad: Leading wildcard = full table scan
SELECT * FROM products WHERE name LIKE '%phone%';

-- ✅ Good: Use full-text search
SELECT * FROM products 
WHERE to_tsvector('english', name) @@ to_tsquery('phone');

-- ✅ Alternative: Trigram index (pg_trgm)
CREATE INDEX idx_name_trgm ON products USING gin (name gin_trgm_ops);
```

### 6. Correlated Subqueries in SELECT

```sql
-- ❌ Bad: Executes subquery for each row
SELECT 
  o.id,
  (SELECT COUNT(*) FROM items WHERE order_id = o.id) as item_count
FROM orders o;

-- ✅ Good: Use JOIN with aggregation
SELECT o.id, COALESCE(i.item_count, 0) as item_count
FROM orders o
LEFT JOIN (
  SELECT order_id, COUNT(*) as item_count 
  FROM items GROUP BY order_id
) i ON o.id = i.order_id;

-- ✅ Better: Window function if applicable
SELECT DISTINCT o.id, COUNT(*) OVER (PARTITION BY o.id) as item_count
FROM orders o LEFT JOIN items i ON o.id = i.order_id;
```

## JOIN Anti-Patterns

### 7. Implicit Joins (ANSI-89 Syntax)

```sql
-- ❌ Bad: Comma-separated tables
SELECT * FROM orders o, customers c, items i
WHERE o.customer_id = c.id AND i.order_id = o.id;

-- ✅ Good: Explicit JOIN syntax
SELECT * FROM orders o
INNER JOIN customers c ON o.customer_id = c.id
INNER JOIN items i ON i.order_id = o.id;
```

### 8. Cartesian Product (Missing JOIN Condition)

```sql
-- ❌ Bad: Missing join condition = cartesian product
SELECT * FROM orders o
CROSS JOIN customers c  -- Intentional?
WHERE o.status = 'pending';

-- ✅ Good: Explicit intent
SELECT * FROM orders o
INNER JOIN customers c ON o.customer_id = c.id
WHERE o.status = 'pending';
```

### 9. Outer Join with WHERE on Nullable Side

```sql
-- ❌ Bad: Converts LEFT JOIN to INNER JOIN
SELECT * FROM orders o
LEFT JOIN customers c ON o.customer_id = c.id
WHERE c.country = 'US';  -- Filters out NULL rows!

-- ✅ Good: Move condition to ON clause
SELECT * FROM orders o
LEFT JOIN customers c ON o.customer_id = c.id AND c.country = 'US';

-- ✅ Alternative: Handle NULL explicitly
SELECT * FROM orders o
LEFT JOIN customers c ON o.customer_id = c.id
WHERE c.country = 'US' OR c.id IS NULL;
```

### 10. Self-Join Instead of Window Function

```sql
-- ❌ Bad: Self-join for previous row
SELECT t1.*, t2.value as prev_value
FROM timeseries t1
LEFT JOIN timeseries t2 ON t1.id = t2.id + 1;

-- ✅ Good: LAG window function
SELECT *, LAG(value) OVER (ORDER BY id) as prev_value
FROM timeseries;
```

## Aggregation Anti-Patterns

### 11. SELECT DISTINCT to Fix Duplicates

```sql
-- ❌ Bad: DISTINCT hides join problem
SELECT DISTINCT o.id, o.total
FROM orders o
JOIN items i ON o.id = i.order_id;

-- ✅ Good: Fix the join/query logic
SELECT o.id, o.total, COUNT(i.id) as item_count
FROM orders o
LEFT JOIN items i ON o.id = i.order_id
GROUP BY o.id, o.total;
```

### 12. GROUP BY with SELECT *

```sql
-- ❌ Bad: Non-aggregated columns in SELECT
SELECT *, COUNT(*)
FROM orders
GROUP BY customer_id;  -- Other columns undefined!

-- ✅ Good: Explicit columns
SELECT customer_id, COUNT(*) as order_count, MAX(created_at) as last_order
FROM orders
GROUP BY customer_id;
```

### 13. UNION Instead of UNION ALL

```sql
-- ❌ Bad: UNION removes duplicates (expensive sort)
SELECT id FROM table_a
UNION
SELECT id FROM table_b;

-- ✅ Good: Use UNION ALL when duplicates OK or impossible
SELECT id, 'A' as source FROM table_a
UNION ALL
SELECT id, 'B' as source FROM table_b;
```

### 14. COUNT(*) vs COUNT(column) Confusion

```sql
-- Different meanings!
SELECT 
  COUNT(*) as total_rows,           -- Counts all rows
  COUNT(email) as rows_with_email,  -- Counts non-NULL values
  COUNT(DISTINCT email) as unique_emails
FROM users;
```

## NULL Handling Anti-Patterns

### 15. NULL in Equality Comparisons

```sql
-- ❌ Bad: NULL != NULL in SQL
SELECT * FROM users WHERE department = NULL;      -- Always empty!
SELECT * FROM users WHERE department != 'Sales';  -- Excludes NULLs!

-- ✅ Good: Use IS NULL / IS NOT NULL
SELECT * FROM users WHERE department IS NULL;
SELECT * FROM users WHERE department != 'Sales' OR department IS NULL;

-- ✅ Alternative: COALESCE for defaults
SELECT * FROM users WHERE COALESCE(department, 'Unknown') != 'Sales';
```

### 16. NULL in NOT IN Subquery

```sql
-- ❌ Bad: Returns empty if subquery has NULL
SELECT * FROM products
WHERE category_id NOT IN (SELECT id FROM categories);  -- Fails if any id is NULL

-- ✅ Good: Use NOT EXISTS
SELECT * FROM products p
WHERE NOT EXISTS (SELECT 1 FROM categories c WHERE c.id = p.category_id);

-- ✅ Alternative: Filter NULLs
SELECT * FROM products
WHERE category_id NOT IN (SELECT id FROM categories WHERE id IS NOT NULL);
```

### 17. NULL in Arithmetic

```sql
-- ❌ Bad: NULL propagates
SELECT price * quantity as total  -- NULL if either is NULL
FROM items;

-- ✅ Good: Handle NULL explicitly
SELECT COALESCE(price, 0) * COALESCE(quantity, 0) as total
FROM items;
```

## Schema Anti-Patterns

### 18. VARCHAR for Everything

```sql
-- ❌ Bad: Wrong data types
CREATE TABLE orders (
  id VARCHAR(50),           -- Should be INT/BIGINT
  amount VARCHAR(20),       -- Should be DECIMAL
  created_at VARCHAR(30),   -- Should be TIMESTAMP
  is_active VARCHAR(5)      -- Should be BOOLEAN
);

-- ✅ Good: Appropriate types
CREATE TABLE orders (
  id BIGINT PRIMARY KEY,
  amount DECIMAL(12, 2),
  created_at TIMESTAMP WITH TIME ZONE,
  is_active BOOLEAN DEFAULT true
);
```

### 19. Entity-Attribute-Value (EAV) Pattern

```sql
-- ❌ Bad: EAV table
CREATE TABLE product_attributes (
  product_id INT,
  attribute_name VARCHAR(100),
  attribute_value VARCHAR(500)
);

-- Problems: No type safety, complex queries, poor performance

-- ✅ Good: JSONB for flexible attributes
CREATE TABLE products (
  id SERIAL PRIMARY KEY,
  name VARCHAR(255),
  attributes JSONB  -- {"color": "red", "size": "L", "weight": 2.5}
);

-- ✅ Alternative: Proper normalization for known attributes
```

### 20. Missing Foreign Key Constraints

```sql
-- ❌ Bad: No referential integrity
CREATE TABLE orders (
  id SERIAL PRIMARY KEY,
  customer_id INT  -- No FK constraint!
);

-- ✅ Good: Enforce integrity
CREATE TABLE orders (
  id SERIAL PRIMARY KEY,
  customer_id INT NOT NULL REFERENCES customers(id)
);
```

### 21. Using FLOAT for Money

```sql
-- ❌ Bad: Floating point precision issues
SELECT 0.1 + 0.2;  -- Returns 0.30000000000000004

-- ✅ Good: Use DECIMAL/NUMERIC
CREATE TABLE transactions (
  amount DECIMAL(15, 2)  -- Exact precision for money
);
```

## Index Anti-Patterns

### 22. Over-Indexing

```sql
-- ❌ Bad: Index on every column
CREATE INDEX idx_1 ON orders(id);          -- Already indexed (PK)
CREATE INDEX idx_2 ON orders(customer_id);
CREATE INDEX idx_3 ON orders(status);
CREATE INDEX idx_4 ON orders(created_at);
CREATE INDEX idx_5 ON orders(customer_id, status);  -- Redundant
CREATE INDEX idx_6 ON orders(status, customer_id);  -- Maybe redundant

-- ✅ Good: Index based on query patterns
-- Analyze actual queries before creating indexes
```

### 23. Wrong Column Order in Composite Index

```sql
-- For query: WHERE status = 'active' AND created_at > '2025-01-01'

-- ❌ Bad: Low cardinality column first
CREATE INDEX idx ON orders(status, created_at);

-- ✅ Good: Consider cardinality and query patterns
-- If status has few values, this order works for equality + range
-- But consider both query types before deciding
```

### 24. Indexing Heavily Updated Columns

```sql
-- ❌ Bad: Index on frequently updated column
CREATE INDEX idx ON counters(view_count);  -- Updated every page view

-- ✅ Good: Index on stable columns, materialize for queries
-- Use materialized views or summary tables for analytics
```

## CTE Anti-Patterns

### 25. Unused CTEs

```sql
-- ❌ Bad: CTE defined but never used
WITH 
  active_users AS (SELECT * FROM users WHERE status = 'active'),
  premium_users AS (SELECT * FROM users WHERE plan = 'premium')  -- Never used!
SELECT * FROM active_users;

-- ✅ Good: Only define needed CTEs
WITH active_users AS (SELECT * FROM users WHERE status = 'active')
SELECT * FROM active_users;
```

### 26. CTE for Single-Use Subquery

```sql
-- ❌ Bad: Unnecessary CTE (may prevent optimization)
WITH filtered AS (SELECT * FROM orders WHERE status = 'pending')
SELECT * FROM filtered WHERE total > 100;

-- ✅ Good: Direct query (optimizer can merge)
SELECT * FROM orders WHERE status = 'pending' AND total > 100;
```

## Transaction Anti-Patterns

### 27. Long-Running Transactions

```sql
-- ❌ Bad: Transaction holds locks too long
BEGIN;
SELECT * FROM large_table;  -- Long read
-- User goes to lunch...
UPDATE orders SET status = 'done' WHERE id = 1;
COMMIT;

-- ✅ Good: Keep transactions short
BEGIN;
UPDATE orders SET status = 'done' WHERE id = 1;
COMMIT;
-- Do reads outside transaction when possible
```

### 28. Missing Transaction for Related Updates

```sql
-- ❌ Bad: Related updates without transaction
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
-- App crashes here!
UPDATE accounts SET balance = balance + 100 WHERE id = 2;

-- ✅ Good: Atomic transaction
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;
```

## Query Writing Anti-Patterns

### 29. Ordinal Numbers in ORDER BY

```sql
-- ❌ Bad: Fragile, unclear
SELECT name, created_at, status FROM orders ORDER BY 2 DESC, 3;

-- ✅ Good: Explicit column names
SELECT name, created_at, status FROM orders 
ORDER BY created_at DESC, status;
```

### 30. Implicit Type Conversion

```sql
-- ❌ Bad: String compared to integer
SELECT * FROM orders WHERE order_id = '12345';  -- May not use index

-- ✅ Good: Matching types
SELECT * FROM orders WHERE order_id = 12345;
```

### 31. OFFSET for Pagination

```sql
-- ❌ Bad: OFFSET scans skipped rows
SELECT * FROM products ORDER BY id LIMIT 20 OFFSET 10000;  -- Scans 10020 rows

-- ✅ Good: Keyset pagination
SELECT * FROM products WHERE id > 10000 ORDER BY id LIMIT 20;
```

### 32. EXISTS vs IN for Large Sets

```sql
-- For large subquery results, EXISTS often performs better

-- ✅ Good: EXISTS (stops at first match)
SELECT * FROM orders o
WHERE EXISTS (SELECT 1 FROM vip_customers v WHERE v.id = o.customer_id);

-- May be slower: IN (materializes all results)
SELECT * FROM orders
WHERE customer_id IN (SELECT id FROM vip_customers);
```

## Detection Checklist

Use this checklist when reviewing SQL:

- [ ] No `SELECT *` in production code
- [ ] No functions on indexed columns in WHERE
- [ ] No implicit joins (comma syntax)
- [ ] No `DISTINCT` without justification
- [ ] No `UNION` where `UNION ALL` works
- [ ] NULL handling is explicit
- [ ] Appropriate data types used
- [ ] Transactions are short and atomic
- [ ] Pagination uses keyset, not OFFSET
- [ ] JOINs have proper conditions
