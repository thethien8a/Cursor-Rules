---
name: sql-pro
description: Master modern SQL with AI-powered databases, cloud-native platforms, vector search, data lakehouse patterns, and advanced optimization. Expert in performance tuning, streaming SQL, dbt transformations, and anti-pattern detection. Use PROACTIVELY for database optimization, complex analytics, query debugging, or data engineering tasks.
---

# SQL Pro - Modern Database Mastery

Expert SQL specialist for modern database systems, AI integration, performance optimization, and data engineering best practices.

## When to Use

- Writing/optimizing complex SQL queries
- Designing schemas for OLTP/OLAP/HTAP workloads
- Debugging slow queries with execution plans
- Building data pipelines with dbt
- Implementing vector search or AI features
- Working with data lakehouses (Iceberg, Delta Lake, Hudi)

## When NOT to Use

- Pure ORM-level guidance without SQL
- Document-only databases (MongoDB without SQL interface)
- No access to query plans or schema

## Quick Reference

### Execution Plan Analysis Checklist

```sql
-- PostgreSQL: Always use ANALYZE for actual runtime stats
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT) SELECT ...;

-- Key metrics to check:
-- 1. Seq Scan on large tables → Need index?
-- 2. Nested Loop with high rows → Consider Hash/Merge Join
-- 3. Sort with high memory → Add index or increase work_mem
-- 4. Buffers: shared hit vs read → Cache efficiency
```

### Index Strategy Quick Guide

| Access Pattern | Index Type | When to Use |
|---------------|------------|-------------|
| Exact match (=) | B-tree | Primary keys, unique lookups |
| Range queries (<, >, BETWEEN) | B-tree | Date ranges, numeric ranges |
| Text search (LIKE 'prefix%') | B-tree | Prefix matching |
| Full-text search | GIN/GiST | Natural language search |
| JSON/Array containment | GIN | JSONB queries, array ops |
| Vector similarity | HNSW/IVFFlat | Semantic search, embeddings |
| Geospatial | GiST/SP-GiST | Location-based queries |
| Analytics (large scans) | Columnstore | Aggregations, OLAP |

### Modern SQL Features Cheatsheet

```sql
-- Window Functions (avoid self-joins)
SELECT *,
  LAG(value) OVER (PARTITION BY category ORDER BY date) as prev_value,
  SUM(value) OVER (PARTITION BY category ORDER BY date ROWS UNBOUNDED PRECEDING) as running_total,
  RANK() OVER (PARTITION BY category ORDER BY value DESC) as rank
FROM table;

-- Recursive CTE (hierarchical data)
WITH RECURSIVE hierarchy AS (
  SELECT id, parent_id, name, 1 as level
  FROM org WHERE parent_id IS NULL
  UNION ALL
  SELECT o.id, o.parent_id, o.name, h.level + 1
  FROM org o JOIN hierarchy h ON o.parent_id = h.id
)
SELECT * FROM hierarchy;

-- MERGE/UPSERT (PostgreSQL 15+)
MERGE INTO target t
USING source s ON t.id = s.id
WHEN MATCHED THEN UPDATE SET value = s.value
WHEN NOT MATCHED THEN INSERT (id, value) VALUES (s.id, s.value);

-- JSON_TABLE (PostgreSQL 17+)
SELECT jt.*
FROM api_responses,
  JSON_TABLE(response, '$.items[*]' COLUMNS (
    id INT PATH '$.id',
    name TEXT PATH '$.name'
  )) AS jt;
```

## Critical Anti-Patterns to Avoid

| Anti-Pattern | Problem | Solution |
|-------------|---------|----------|
| `SELECT *` | Unnecessary I/O, breaks schema changes | List specific columns |
| N+1 queries | Loop queries = N+1 round trips | Use JOINs or batch queries |
| Functions in WHERE | `WHERE YEAR(date) = 2025` prevents index use | `WHERE date >= '2025-01-01'` |
| `SELECT DISTINCT` to fix duplicates | Masks JOIN issues | Fix the JOIN logic |
| `UNION` instead of `UNION ALL` | Unnecessary deduplication | Use `UNION ALL` when possible |
| Implicit type conversion | Index bypass, wrong results | Explicit casting |
| Missing NULL handling | Silent bugs in comparisons | Use `COALESCE`, `IS NULL` |

> **Deep dive**: See `./anti-patterns.md` for 30+ patterns with examples.

## AI & Vector Database Integration

### pgvector Semantic Search

```sql
-- Enable extension
CREATE EXTENSION vector;

-- Create table with embedding column
CREATE TABLE documents (
  id SERIAL PRIMARY KEY,
  content TEXT,
  embedding vector(1536)  -- OpenAI ada-002 dimension
);

-- Create HNSW index for fast similarity search
CREATE INDEX ON documents 
USING hnsw (embedding vector_cosine_ops);

-- Semantic search query
SELECT content, 1 - (embedding <=> $query_embedding) AS similarity
FROM documents
ORDER BY embedding <=> $query_embedding
LIMIT 10;
```

### Native AI Functions (2025+)

```sql
-- Snowflake Cortex AI
SELECT SNOWFLAKE.CORTEX.SENTIMENT(review_text) as sentiment,
       SNOWFLAKE.CORTEX.SUMMARIZE(description) as summary
FROM products;

-- Oracle Select AI (natural language to SQL)
SELECT AI "Show top 10 customers by revenue last quarter";

-- SQL Server 2025 Vector Search
SELECT * FROM products
ORDER BY VECTOR_DISTANCE('cosine', embedding, @query_vector)
OFFSET 0 ROWS FETCH NEXT 10 ROWS ONLY;
```

## Data Lakehouse Patterns

### Open Table Format Comparison

| Feature | Apache Iceberg | Delta Lake | Apache Hudi |
|---------|---------------|------------|-------------|
| ACID | Yes | Yes | Yes |
| Time Travel | Snapshot-based | Version-based | Timeline-based |
| Schema Evolution | Full support | Full support | Full support |
| Best For | Multi-engine analytics | Spark/Databricks | Upserts, CDC |
| Partition Evolution | Yes (hidden) | Manual | Manual |

### Iceberg SQL Operations

```sql
-- Time travel query
SELECT * FROM catalog.db.table
FOR SYSTEM_TIME AS OF TIMESTAMP '2025-01-01 00:00:00';

-- Schema evolution
ALTER TABLE catalog.db.table 
ADD COLUMN new_col STRING AFTER existing_col;

-- Partition evolution (no rewrite needed)
ALTER TABLE catalog.db.table 
ADD PARTITION FIELD months(event_date);

-- Maintenance operations
CALL catalog.system.rewrite_data_files('db.table');
CALL catalog.system.expire_snapshots('db.table', TIMESTAMP '2025-01-01');
```

## dbt Best Practices

### Project Structure

```
models/
├── staging/          # 1:1 with source tables, light cleaning
│   └── stg_*.sql
├── intermediate/     # Business logic, joins
│   └── int_*.sql
└── marts/           # Business-ready aggregations
    ├── dim_*.sql    # Dimensions
    └── fct_*.sql    # Facts
```

### Materialization Strategy

| Model Type | Materialization | When |
|-----------|-----------------|------|
| Staging | View | Always fresh, low cost |
| Intermediate | Ephemeral/View | Reduce redundancy |
| Marts | Table | Query performance |
| Large facts | Incremental | Build time > 1 hour |

```sql
-- Incremental model pattern
{{
  config(
    materialized='incremental',
    unique_key='id',
    incremental_strategy='merge'
  )
}}

SELECT * FROM {{ ref('stg_events') }}
{% if is_incremental() %}
WHERE updated_at > (SELECT MAX(updated_at) FROM {{ this }})
{% endif %}
```

## Streaming SQL (Real-time)

### ksqlDB Patterns

```sql
-- Create stream from Kafka topic
CREATE STREAM orders (
  order_id VARCHAR KEY,
  customer_id VARCHAR,
  amount DECIMAL(10,2),
  ts TIMESTAMP
) WITH (
  KAFKA_TOPIC='orders',
  VALUE_FORMAT='JSON',
  TIMESTAMP='ts'
);

-- Real-time aggregation (materialized view)
CREATE TABLE hourly_sales AS
SELECT customer_id,
  WINDOWSTART as window_start,
  SUM(amount) as total_sales,
  COUNT(*) as order_count
FROM orders
WINDOW TUMBLING (SIZE 1 HOUR)
GROUP BY customer_id
EMIT CHANGES;

-- Push query (continuous results)
SELECT * FROM hourly_sales EMIT CHANGES;

-- Pull query (point-in-time lookup)
SELECT * FROM hourly_sales WHERE customer_id = 'C123';
```

## Cloud Platform Optimization

### Snowflake

```sql
-- Query acceleration for ad-hoc analytics
ALTER WAREHOUSE my_wh SET QUERY_ACCELERATION_MAX_SCALE_FACTOR = 8;

-- Clustering for large tables (instead of indexes)
ALTER TABLE large_fact CLUSTER BY (date_key, region);

-- Search optimization for point lookups
ALTER TABLE products ADD SEARCH OPTIMIZATION ON EQUALITY(sku);

-- Result caching (automatic, but can force)
ALTER SESSION SET USE_CACHED_RESULT = TRUE;
```

### BigQuery

```sql
-- Partition and cluster for cost/performance
CREATE TABLE dataset.events
PARTITION BY DATE(event_timestamp)
CLUSTER BY user_id, event_type
AS SELECT * FROM source;

-- Approximate aggregations for speed
SELECT APPROX_COUNT_DISTINCT(user_id) as unique_users
FROM events WHERE date = CURRENT_DATE();

-- Materialized views for common aggregations
CREATE MATERIALIZED VIEW mv_daily_stats AS
SELECT DATE(ts) as date, COUNT(*) as events
FROM events GROUP BY 1;
```

> **Deep dive**: See `./cloud-platforms.md` for Databricks, Aurora, Azure SQL.

## PostgreSQL 17/18 Features

### PostgreSQL 18 (2025)

```sql
-- Virtual generated columns (computed at query time)
ALTER TABLE orders ADD COLUMN total_with_tax NUMERIC 
  GENERATED ALWAYS AS (amount * 1.1) VIRTUAL;

-- UUIDv7 (timestamp-ordered, better for indexes)
SELECT uuidv7();  -- e.g., 019234ab-5678-7def-8901-234567890abc

-- Enhanced RETURNING with OLD/NEW
UPDATE products SET price = price * 1.1
RETURNING OLD.price as old_price, NEW.price as new_price;

-- Asynchronous I/O (up to 3x faster reads)
-- Enabled by default for seq scans, bitmap scans, vacuum
```

### PostgreSQL 17

```sql
-- JSON_TABLE (convert JSON to relational)
SELECT jt.* FROM api_data,
  JSON_TABLE(payload, '$.users[*]' COLUMNS (
    user_id INT PATH '$.id',
    email TEXT PATH '$.email',
    active BOOLEAN PATH '$.is_active'
  )) AS jt;

-- Incremental MERGE support
-- Improved CTE optimization
-- Better parallel query execution
```

## Performance Tuning Workflow

```
1. IDENTIFY → Query taking > expected time?
   └─ Check: pg_stat_statements, slow query log

2. ANALYZE → Run EXPLAIN (ANALYZE, BUFFERS)
   └─ Look for: Seq Scans, High Rows, Sorts, Nested Loops

3. DIAGNOSE → What's the bottleneck?
   ├─ I/O bound → Add indexes, increase shared_buffers
   ├─ CPU bound → Simplify query, add materialized view
   └─ Memory bound → Increase work_mem, partition table

4. OPTIMIZE → Apply fix
   ├─ Add/modify indexes
   ├─ Rewrite query (CTEs, window functions)
   ├─ Update statistics (ANALYZE table)
   └─ Partition large tables

5. VALIDATE → Re-run EXPLAIN, compare metrics
   └─ Test with production-like data volume
```

> **Deep dive**: See `./execution-plan-guide.md` for detailed examples.

## Interview-Ready Patterns

### Top 5 SQL Patterns (Asked 80% of time)

```sql
-- 1. Running Total / Rolling Average
SELECT date, value,
  AVG(value) OVER (ORDER BY date ROWS BETWEEN 6 PRECEDING AND CURRENT ROW) as rolling_7d_avg
FROM metrics;

-- 2. Rank / Top N per Group
WITH ranked AS (
  SELECT *, ROW_NUMBER() OVER (PARTITION BY category ORDER BY sales DESC) as rn
  FROM products
)
SELECT * FROM ranked WHERE rn <= 3;

-- 3. Year-over-Year Comparison
SELECT current.month,
  current.revenue,
  previous.revenue as prev_year,
  (current.revenue - previous.revenue) / previous.revenue * 100 as yoy_growth
FROM monthly_sales current
LEFT JOIN monthly_sales previous 
  ON current.month = previous.month + INTERVAL '1 year';

-- 4. Gap Analysis (Missing Sequences)
WITH all_dates AS (
  SELECT generate_series(MIN(date), MAX(date), '1 day'::interval)::date as date
  FROM events
)
SELECT a.date FROM all_dates a
LEFT JOIN events e ON a.date = e.date
WHERE e.date IS NULL;

-- 5. Deduplication (Keep Latest)
DELETE FROM users u
USING (
  SELECT email, MAX(created_at) as max_created
  FROM users GROUP BY email HAVING COUNT(*) > 1
) d
WHERE u.email = d.email AND u.created_at < d.max_created;
```

> **Deep dive**: See `./interview-patterns.md` for 20+ patterns.

## Safety Guidelines

- **Production queries**: Always use `LIMIT` for exploratory queries
- **Write operations**: Test on staging first, use transactions
- **Large deletes**: Batch with `LIMIT` to avoid locks
- **Schema changes**: Use online DDL when available
- **Credentials**: Never hardcode, use secrets management

## Response Approach

1. **Clarify requirements** → Data volume, latency needs, platform
2. **Check schema** → Existing indexes, data types, constraints  
3. **Write query** → Use modern SQL features appropriately
4. **Optimize** → Add indexes, check execution plan
5. **Test** → Validate with realistic data volume
6. **Document** → Explain assumptions, edge cases

## Additional Resources

- `./anti-patterns.md` - 30+ SQL anti-patterns with fixes
- `./execution-plan-guide.md` - Deep dive into EXPLAIN
- `./cloud-platforms.md` - Platform-specific optimization
- `./interview-patterns.md` - Common interview questions
