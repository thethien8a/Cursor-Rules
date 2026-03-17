# Cloud Database Platform Optimization

Platform-specific optimization techniques for major cloud databases.

## Snowflake

### Warehouse Sizing & Configuration

```sql
-- Create warehouse with auto-suspend
CREATE WAREHOUSE analytics_wh
  WITH WAREHOUSE_SIZE = 'MEDIUM'
  AUTO_SUSPEND = 60           -- Seconds of inactivity
  AUTO_RESUME = TRUE
  INITIALLY_SUSPENDED = TRUE;

-- Multi-cluster for concurrency
ALTER WAREHOUSE analytics_wh SET
  MIN_CLUSTER_COUNT = 1
  MAX_CLUSTER_COUNT = 3
  SCALING_POLICY = 'ECONOMY';  -- or 'STANDARD'

-- Query acceleration (for variable workloads)
ALTER WAREHOUSE analytics_wh SET
  ENABLE_QUERY_ACCELERATION = TRUE
  QUERY_ACCELERATION_MAX_SCALE_FACTOR = 8;
```

### Clustering (Instead of Indexes)

```sql
-- Add clustering key (for large tables)
ALTER TABLE sales CLUSTER BY (sale_date, region);

-- Check clustering depth (lower is better)
SELECT SYSTEM$CLUSTERING_DEPTH('sales');

-- Manual recluster if needed
ALTER TABLE sales RECLUSTER;

-- Automatic clustering (recommended)
ALTER TABLE sales RESUME RECLUSTER;
```

### Search Optimization

```sql
-- For point lookups on large tables
ALTER TABLE products ADD SEARCH OPTIMIZATION ON EQUALITY(sku, product_id);
ALTER TABLE customers ADD SEARCH OPTIMIZATION ON SUBSTRING(email);

-- Check optimization status
SELECT * FROM TABLE(INFORMATION_SCHEMA.SEARCH_OPTIMIZATION_HISTORY(
  DATE_RANGE_START => DATEADD('day', -7, CURRENT_TIMESTAMP()),
  TABLE_NAME => 'products'
));
```

### Query Optimization

```sql
-- Use COPY for bulk inserts (not INSERT)
COPY INTO sales FROM @my_stage/data/
  FILE_FORMAT = (TYPE = 'PARQUET');

-- Avoid SELECT * with semi-structured data
SELECT 
  data:customer.name::STRING as name,
  data:customer.email::STRING as email
FROM json_table;

-- Use LATERAL FLATTEN for arrays
SELECT t.id, f.value::STRING as tag
FROM products t,
LATERAL FLATTEN(input => t.tags) f;

-- Materialized views for common aggregations
CREATE MATERIALIZED VIEW mv_daily_sales AS
SELECT DATE_TRUNC('day', sale_date) as day, SUM(amount) as total
FROM sales
GROUP BY 1;
```

### Snowflake Cortex AI

```sql
-- Sentiment analysis
SELECT review_text,
  SNOWFLAKE.CORTEX.SENTIMENT(review_text) as sentiment_score
FROM reviews;

-- Summarization
SELECT SNOWFLAKE.CORTEX.SUMMARIZE(long_description) as summary
FROM articles;

-- Classification
SELECT SNOWFLAKE.CORTEX.CLASSIFY_TEXT(
  support_ticket,
  ['billing', 'technical', 'general']
) as category
FROM tickets;

-- Text completion
SELECT SNOWFLAKE.CORTEX.COMPLETE(
  'llama3.1-70b',
  'Summarize this customer feedback: ' || feedback
) as summary
FROM customer_feedback;
```

## Google BigQuery

### Partitioning & Clustering

```sql
-- Create partitioned and clustered table
CREATE TABLE dataset.events
PARTITION BY DATE(event_timestamp)
CLUSTER BY user_id, event_type
OPTIONS (
  partition_expiration_days = 90,
  require_partition_filter = TRUE
)
AS SELECT * FROM source_events;

-- Integer range partitioning
CREATE TABLE dataset.users
PARTITION BY RANGE_BUCKET(user_id, GENERATE_ARRAY(0, 1000000, 10000))
AS SELECT * FROM source_users;
```

### Query Optimization

```sql
-- Avoid SELECT * (columnar storage)
SELECT user_id, event_type, event_timestamp
FROM dataset.events
WHERE DATE(event_timestamp) = CURRENT_DATE();

-- Use approximate functions for large datasets
SELECT 
  APPROX_COUNT_DISTINCT(user_id) as unique_users,
  APPROX_QUANTILES(amount, 100)[OFFSET(50)] as median_amount
FROM large_table;

-- Avoid repeated subqueries - use WITH
WITH filtered AS (
  SELECT * FROM events WHERE date = CURRENT_DATE()
)
SELECT * FROM filtered f1
JOIN filtered f2 ON f1.session_id = f2.session_id;

-- Use STRUCT for denormalization
SELECT 
  user_id,
  ARRAY_AGG(STRUCT(event_type, timestamp)) as events
FROM user_events
GROUP BY user_id;
```

### BI Engine & Materialized Views

```sql
-- Create materialized view (auto-refreshed)
CREATE MATERIALIZED VIEW dataset.mv_hourly_metrics
OPTIONS (enable_refresh = true, refresh_interval_minutes = 30)
AS
SELECT 
  TIMESTAMP_TRUNC(event_time, HOUR) as hour,
  event_type,
  COUNT(*) as event_count
FROM dataset.events
GROUP BY 1, 2;

-- BI Engine is automatic for cached queries
-- Just ensure queries hit materialized views
```

### BigQuery ML

```sql
-- Create model directly in SQL
CREATE OR REPLACE MODEL dataset.customer_churn_model
OPTIONS (
  model_type = 'LOGISTIC_REG',
  input_label_cols = ['churned']
) AS
SELECT * FROM dataset.customer_features;

-- Predict
SELECT * FROM ML.PREDICT(
  MODEL dataset.customer_churn_model,
  (SELECT * FROM dataset.new_customers)
);
```

## Databricks / Delta Lake

### Table Optimization

```sql
-- Create Delta table with optimization
CREATE TABLE events (
  event_id BIGINT,
  event_type STRING,
  event_date DATE,
  user_id BIGINT
)
USING DELTA
PARTITIONED BY (event_date)
TBLPROPERTIES (
  'delta.autoOptimize.optimizeWrite' = 'true',
  'delta.autoOptimize.autoCompact' = 'true'
);

-- Z-ORDER for multi-dimensional clustering
OPTIMIZE events ZORDER BY (user_id, event_type);

-- Vacuum old files (default 7 days retention)
VACUUM events RETAIN 168 HOURS;

-- Analyze table for statistics
ANALYZE TABLE events COMPUTE STATISTICS FOR ALL COLUMNS;
```

### Liquid Clustering (Delta Lake 3.0+)

```sql
-- Modern clustering (replaces partitioning + Z-ORDER)
CREATE TABLE events CLUSTER BY (event_date, user_id);

-- Cluster existing table
ALTER TABLE events CLUSTER BY (event_date, user_id);

-- Trigger clustering
OPTIMIZE events;
```

### Performance Features

```sql
-- Photon engine (auto-enabled in Databricks Runtime)
-- No SQL changes needed, just use compatible operations

-- Predictive I/O (reads only needed data)
SET spark.databricks.io.cache.enabled = true;

-- Dynamic file pruning
-- Automatic for join predicates on partitioned columns

-- Bloom filters for point lookups
CREATE BLOOMFILTER INDEX ON events(user_id);
```

### Delta Lake Time Travel

```sql
-- Query historical version
SELECT * FROM events VERSION AS OF 10;
SELECT * FROM events TIMESTAMP AS OF '2025-01-01';

-- Restore to previous version
RESTORE TABLE events TO VERSION AS OF 10;

-- Describe history
DESCRIBE HISTORY events;
```

### Unity Catalog AI Functions

```sql
-- Register AI function
CREATE FUNCTION summarize_text(text STRING)
RETURNS STRING
LANGUAGE PYTHON
AS $$
  import openai
  return openai.ChatCompletion.create(
    model="gpt-4",
    messages=[{"role": "user", "content": f"Summarize: {text}"}]
  ).choices[0].message.content
$$;

-- Use in queries
SELECT summarize_text(description) FROM products;
```

## Amazon Redshift

### Distribution & Sort Keys

```sql
-- Create table with distribution
CREATE TABLE sales (
  sale_id BIGINT,
  customer_id BIGINT,
  product_id BIGINT,
  sale_date DATE,
  amount DECIMAL(12,2)
)
DISTSTYLE KEY
DISTKEY (customer_id)
SORTKEY (sale_date);

-- Distribution styles:
-- KEY: Hash by column (for joins)
-- ALL: Copy to all nodes (for small dims)
-- EVEN: Round-robin (default)
-- AUTO: Redshift chooses
```

### Compression & Encoding

```sql
-- Analyze compression
ANALYZE COMPRESSION sales;

-- Create with encoding
CREATE TABLE sales (
  sale_id BIGINT ENCODE az64,
  customer_id BIGINT ENCODE az64,
  description VARCHAR(500) ENCODE zstd,
  amount DECIMAL(12,2) ENCODE az64
);
```

### Query Optimization

```sql
-- Use materialized views
CREATE MATERIALIZED VIEW mv_daily_sales AS
SELECT DATE_TRUNC('day', sale_date) as day, SUM(amount) as total
FROM sales
GROUP BY 1;

-- Automatic refresh
ALTER MATERIALIZED VIEW mv_daily_sales AUTO REFRESH YES;

-- Analyze query plan
EXPLAIN SELECT * FROM sales WHERE customer_id = 123;

-- Check query execution
SELECT * FROM svl_query_report WHERE query = 12345;
```

### Redshift Serverless

```sql
-- Serverless workgroup (managed capacity)
-- Configure via console/API, then query normally

-- Set RPU limits
ALTER WORKGROUP my_workgroup SET MAX_RPU = 128;

-- Query isolation
ALTER WORKGROUP my_workgroup 
SET QUERY_COMPUTE_UNITS_TARGET = 64;
```

### Redshift ML

```sql
-- Create model
CREATE MODEL churn_prediction
FROM (SELECT * FROM customer_features)
TARGET churned
FUNCTION predict_churn
IAM_ROLE 'arn:aws:iam::123456789:role/RedshiftML'
SETTINGS (
  S3_BUCKET 'my-ml-bucket',
  MAX_RUNTIME 3600
);

-- Use model
SELECT customer_id, predict_churn(features) as churn_probability
FROM new_customers;
```

## Azure SQL Database

### Performance Tiers

```sql
-- Serverless tier (auto-pause)
ALTER DATABASE mydb
MODIFY (SERVICE_OBJECTIVE = 'GP_S_Gen5_2')
WITH (AUTO_PAUSE_DELAY = 60, MIN_CAPACITY = 0.5);

-- Hyperscale (large databases)
-- Configure via Azure Portal

-- Read replicas for analytics
-- Connection string: ApplicationIntent=ReadOnly
```

### Intelligent Query Processing

```sql
-- Adaptive joins (auto in compatibility 150+)
ALTER DATABASE mydb SET COMPATIBILITY_LEVEL = 160;

-- Memory grant feedback
ALTER DATABASE SCOPED CONFIGURATION SET MEMORY_GRANT_FEEDBACK_PERCENTILE_GRANT = ON;

-- Parameter sensitive plan optimization
ALTER DATABASE SCOPED CONFIGURATION SET PARAMETER_SENSITIVE_PLAN_OPTIMIZATION = ON;
```

### Columnstore Indexes

```sql
-- Clustered columnstore for analytics
CREATE CLUSTERED COLUMNSTORE INDEX cci_sales ON sales;

-- Nonclustered for hybrid workloads
CREATE NONCLUSTERED COLUMNSTORE INDEX ncci_sales 
ON sales (sale_date, amount, customer_id)
WHERE sale_date > '2024-01-01';

-- Ordered columnstore (Azure SQL only)
CREATE CLUSTERED COLUMNSTORE INDEX cci_sales ON sales
ORDER (sale_date);
```

### Query Store

```sql
-- Enable Query Store
ALTER DATABASE mydb SET QUERY_STORE = ON;

-- Force plan for query
EXEC sp_query_store_force_plan @query_id = 1, @plan_id = 1;

-- Clear Query Store
ALTER DATABASE mydb SET QUERY_STORE CLEAR;

-- Query Store views
SELECT * FROM sys.query_store_query;
SELECT * FROM sys.query_store_plan;
SELECT * FROM sys.query_store_runtime_stats;
```

## Cross-Platform Best Practices

### Cost Optimization

| Platform | Key Cost Drivers | Optimization |
|----------|-----------------|--------------|
| Snowflake | Compute time | Auto-suspend, right-size warehouse |
| BigQuery | Bytes scanned | Partitioning, clustering, caching |
| Databricks | DBU consumption | Photon, auto-scaling, spot instances |
| Redshift | Node hours | Reserved instances, pause when idle |
| Azure SQL | DTUs/vCores | Serverless, read replicas |

### Data Format Recommendations

| Platform | Best Format | Why |
|----------|-------------|-----|
| Snowflake | Parquet, CSV | Native COPY support |
| BigQuery | Parquet, Avro | Columnar, schema evolution |
| Databricks | Delta Lake | ACID, time travel |
| Redshift | Parquet, ORC | Columnar, compression |
