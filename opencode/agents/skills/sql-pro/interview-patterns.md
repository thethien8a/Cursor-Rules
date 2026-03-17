# SQL Interview Patterns

Common SQL patterns frequently asked in data engineering and analytics interviews.

## Pattern 1: Running Totals & Rolling Averages

### Running Total

```sql
-- Running total of sales by date
SELECT 
  sale_date,
  daily_amount,
  SUM(daily_amount) OVER (ORDER BY sale_date) as running_total
FROM daily_sales;

-- Running total partitioned by category
SELECT 
  category,
  sale_date,
  amount,
  SUM(amount) OVER (
    PARTITION BY category 
    ORDER BY sale_date
  ) as category_running_total
FROM sales;
```

### Rolling Average (7-day, 28-day)

```sql
-- 7-day rolling average
SELECT 
  date,
  value,
  AVG(value) OVER (
    ORDER BY date 
    ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
  ) as rolling_7d_avg
FROM metrics;

-- 28-day rolling average (common for DAU/MAU)
SELECT 
  date,
  dau,
  AVG(dau) OVER (
    ORDER BY date 
    ROWS BETWEEN 27 PRECEDING AND CURRENT ROW
  ) as rolling_28d_avg
FROM daily_active_users;
```

## Pattern 2: Ranking & Top N

### Basic Ranking

```sql
-- ROW_NUMBER: Unique ranking (1, 2, 3)
-- RANK: Ties get same rank, gaps after (1, 1, 3)
-- DENSE_RANK: Ties get same rank, no gaps (1, 1, 2)

SELECT 
  product_name,
  sales,
  ROW_NUMBER() OVER (ORDER BY sales DESC) as row_num,
  RANK() OVER (ORDER BY sales DESC) as rank,
  DENSE_RANK() OVER (ORDER BY sales DESC) as dense_rank
FROM products;
```

### Top N per Group

```sql
-- Top 3 products per category
WITH ranked AS (
  SELECT 
    category,
    product_name,
    sales,
    ROW_NUMBER() OVER (
      PARTITION BY category 
      ORDER BY sales DESC
    ) as rn
  FROM products
)
SELECT * FROM ranked WHERE rn <= 3;

-- Alternative: QUALIFY (Snowflake, BigQuery)
SELECT * FROM products
QUALIFY ROW_NUMBER() OVER (PARTITION BY category ORDER BY sales DESC) <= 3;
```

### Percentile Ranking

```sql
-- Percentile of each value
SELECT 
  employee_id,
  salary,
  PERCENT_RANK() OVER (ORDER BY salary) as percentile,
  NTILE(4) OVER (ORDER BY salary) as quartile
FROM employees;
```

## Pattern 3: Year-over-Year Comparison

### Basic YoY

```sql
-- Revenue comparison YoY
SELECT 
  current_year.month,
  current_year.revenue as current_revenue,
  previous_year.revenue as prev_year_revenue,
  ROUND(
    (current_year.revenue - previous_year.revenue) / 
    NULLIF(previous_year.revenue, 0) * 100, 2
  ) as yoy_growth_pct
FROM monthly_revenue current_year
LEFT JOIN monthly_revenue previous_year 
  ON current_year.month = previous_year.month + INTERVAL '1 year';
```

### Using LAG

```sql
-- YoY using window function
SELECT 
  year,
  month,
  revenue,
  LAG(revenue, 12) OVER (ORDER BY year, month) as prev_year_revenue,
  revenue - LAG(revenue, 12) OVER (ORDER BY year, month) as yoy_change
FROM monthly_revenue;
```

### Month-over-Month

```sql
SELECT 
  month,
  revenue,
  LAG(revenue, 1) OVER (ORDER BY month) as prev_month,
  ROUND(
    (revenue - LAG(revenue, 1) OVER (ORDER BY month)) /
    NULLIF(LAG(revenue, 1) OVER (ORDER BY month), 0) * 100, 2
  ) as mom_growth_pct
FROM monthly_revenue;
```

## Pattern 4: Gap Analysis

### Find Missing Dates

```sql
-- Find days with no events
WITH date_range AS (
  SELECT generate_series(
    (SELECT MIN(event_date) FROM events),
    (SELECT MAX(event_date) FROM events),
    '1 day'::interval
  )::date as date
)
SELECT d.date as missing_date
FROM date_range d
LEFT JOIN events e ON d.date = e.event_date
WHERE e.event_date IS NULL;
```

### Find Missing Sequences

```sql
-- Find gaps in sequential IDs
WITH id_range AS (
  SELECT generate_series(
    (SELECT MIN(id) FROM orders),
    (SELECT MAX(id) FROM orders)
  ) as id
)
SELECT r.id as missing_id
FROM id_range r
LEFT JOIN orders o ON r.id = o.id
WHERE o.id IS NULL;
```

### Session Gap Analysis

```sql
-- Find sessions (30 min gap = new session)
WITH events_with_lag AS (
  SELECT 
    user_id,
    event_time,
    LAG(event_time) OVER (PARTITION BY user_id ORDER BY event_time) as prev_event
  FROM user_events
),
session_markers AS (
  SELECT 
    *,
    CASE 
      WHEN prev_event IS NULL 
        OR event_time - prev_event > INTERVAL '30 minutes'
      THEN 1 ELSE 0 
    END as is_new_session
  FROM events_with_lag
)
SELECT 
  user_id,
  event_time,
  SUM(is_new_session) OVER (
    PARTITION BY user_id 
    ORDER BY event_time
  ) as session_id
FROM session_markers;
```

## Pattern 5: Cohort Analysis

### User Retention Cohort

```sql
-- Monthly retention cohort
WITH first_purchase AS (
  SELECT 
    user_id,
    DATE_TRUNC('month', MIN(purchase_date)) as cohort_month
  FROM purchases
  GROUP BY user_id
),
activity AS (
  SELECT 
    p.user_id,
    f.cohort_month,
    DATE_TRUNC('month', p.purchase_date) as activity_month
  FROM purchases p
  JOIN first_purchase f ON p.user_id = f.user_id
)
SELECT 
  cohort_month,
  EXTRACT(MONTH FROM AGE(activity_month, cohort_month)) as months_since_cohort,
  COUNT(DISTINCT user_id) as users
FROM activity
GROUP BY 1, 2
ORDER BY 1, 2;

-- Pivot to retention matrix
-- (Use CASE WHEN or PIVOT depending on platform)
```

### Revenue Cohort

```sql
-- Revenue by cohort month
WITH first_purchase AS (
  SELECT 
    user_id,
    DATE_TRUNC('month', MIN(purchase_date)) as cohort_month
  FROM purchases
  GROUP BY user_id
)
SELECT 
  f.cohort_month,
  DATE_TRUNC('month', p.purchase_date) as purchase_month,
  SUM(p.amount) as revenue
FROM purchases p
JOIN first_purchase f ON p.user_id = f.user_id
GROUP BY 1, 2
ORDER BY 1, 2;
```

## Pattern 6: Funnel Analysis

### Conversion Funnel

```sql
-- E-commerce funnel
WITH funnel AS (
  SELECT 
    session_id,
    MAX(CASE WHEN event_type = 'page_view' THEN 1 ELSE 0 END) as viewed,
    MAX(CASE WHEN event_type = 'add_to_cart' THEN 1 ELSE 0 END) as added,
    MAX(CASE WHEN event_type = 'checkout' THEN 1 ELSE 0 END) as checkout,
    MAX(CASE WHEN event_type = 'purchase' THEN 1 ELSE 0 END) as purchased
  FROM events
  GROUP BY session_id
)
SELECT 
  SUM(viewed) as step1_view,
  SUM(added) as step2_cart,
  SUM(checkout) as step3_checkout,
  SUM(purchased) as step4_purchase,
  ROUND(SUM(added)::numeric / NULLIF(SUM(viewed), 0) * 100, 2) as view_to_cart_pct,
  ROUND(SUM(purchased)::numeric / NULLIF(SUM(viewed), 0) * 100, 2) as overall_conversion_pct
FROM funnel;
```

### Time-Ordered Funnel

```sql
-- Funnel with time ordering (user must complete steps in order)
WITH ordered_events AS (
  SELECT 
    user_id,
    event_type,
    event_time,
    LAG(event_type) OVER (PARTITION BY user_id ORDER BY event_time) as prev_event
  FROM events
  WHERE event_type IN ('signup', 'activation', 'purchase')
)
SELECT 
  event_type,
  COUNT(DISTINCT user_id) as users,
  COUNT(DISTINCT CASE 
    WHEN prev_event = 'signup' AND event_type = 'activation' THEN user_id
    WHEN prev_event = 'activation' AND event_type = 'purchase' THEN user_id
  END) as proper_sequence_users
FROM ordered_events
GROUP BY event_type;
```

## Pattern 7: Deduplication

### Keep Latest Record

```sql
-- Delete duplicates, keep latest
DELETE FROM users u
USING (
  SELECT email, MAX(created_at) as max_created
  FROM users
  GROUP BY email
  HAVING COUNT(*) > 1
) d
WHERE u.email = d.email AND u.created_at < d.max_created;

-- Alternative: Keep with row_number
WITH ranked AS (
  SELECT *, ROW_NUMBER() OVER (PARTITION BY email ORDER BY created_at DESC) as rn
  FROM users
)
DELETE FROM users WHERE id IN (SELECT id FROM ranked WHERE rn > 1);
```

### Deduplicate with Aggregation

```sql
-- Merge duplicate records
SELECT 
  email,
  MAX(name) as name,  -- or use FIRST_VALUE
  MIN(created_at) as first_seen,
  MAX(last_login) as last_login,
  SUM(purchase_count) as total_purchases
FROM users
GROUP BY email;
```

## Pattern 8: Recursive Queries

### Hierarchy Traversal

```sql
-- Organization chart
WITH RECURSIVE org_tree AS (
  -- Base case: root nodes
  SELECT id, name, manager_id, 1 as level, name::text as path
  FROM employees
  WHERE manager_id IS NULL
  
  UNION ALL
  
  -- Recursive case
  SELECT e.id, e.name, e.manager_id, t.level + 1, t.path || ' > ' || e.name
  FROM employees e
  JOIN org_tree t ON e.manager_id = t.id
)
SELECT * FROM org_tree ORDER BY path;
```

### Bill of Materials

```sql
-- Product component hierarchy
WITH RECURSIVE bom AS (
  SELECT 
    product_id, 
    component_id, 
    quantity,
    1 as level
  FROM product_components
  WHERE product_id = 100  -- Starting product
  
  UNION ALL
  
  SELECT 
    b.product_id,
    pc.component_id,
    b.quantity * pc.quantity,
    b.level + 1
  FROM bom b
  JOIN product_components pc ON b.component_id = pc.product_id
  WHERE b.level < 10  -- Prevent infinite loops
)
SELECT component_id, SUM(quantity) as total_needed
FROM bom
GROUP BY component_id;
```

## Pattern 9: Time-Based Analysis

### Consecutive Days

```sql
-- Find users with 7+ consecutive login days
WITH login_groups AS (
  SELECT 
    user_id,
    login_date,
    login_date - ROW_NUMBER() OVER (
      PARTITION BY user_id ORDER BY login_date
    )::int as grp
  FROM (SELECT DISTINCT user_id, login_date::date FROM logins) t
)
SELECT 
  user_id,
  MIN(login_date) as streak_start,
  MAX(login_date) as streak_end,
  COUNT(*) as consecutive_days
FROM login_groups
GROUP BY user_id, grp
HAVING COUNT(*) >= 7;
```

### First/Last Activity

```sql
-- User first and last activity
SELECT 
  user_id,
  MIN(event_time) as first_activity,
  MAX(event_time) as last_activity,
  MAX(event_time) - MIN(event_time) as active_span,
  COUNT(DISTINCT DATE(event_time)) as active_days
FROM user_events
GROUP BY user_id;
```

## Pattern 10: Median & Percentiles

### Median Calculation

```sql
-- PostgreSQL: PERCENTILE_CONT
SELECT PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY salary) as median_salary
FROM employees;

-- Alternative: Using NTILE
WITH quartiles AS (
  SELECT salary, NTILE(2) OVER (ORDER BY salary) as half
  FROM employees
)
SELECT MAX(salary) as median
FROM quartiles WHERE half = 1;
```

### Multiple Percentiles

```sql
SELECT 
  PERCENTILE_CONT(0.25) WITHIN GROUP (ORDER BY response_time) as p25,
  PERCENTILE_CONT(0.50) WITHIN GROUP (ORDER BY response_time) as p50,
  PERCENTILE_CONT(0.75) WITHIN GROUP (ORDER BY response_time) as p75,
  PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY response_time) as p95,
  PERCENTILE_CONT(0.99) WITHIN GROUP (ORDER BY response_time) as p99
FROM api_requests;
```

## Pattern 11: Pivot / Unpivot

### Pivot (Rows to Columns)

```sql
-- PostgreSQL: FILTER clause
SELECT 
  product_id,
  SUM(amount) FILTER (WHERE quarter = 'Q1') as q1_sales,
  SUM(amount) FILTER (WHERE quarter = 'Q2') as q2_sales,
  SUM(amount) FILTER (WHERE quarter = 'Q3') as q3_sales,
  SUM(amount) FILTER (WHERE quarter = 'Q4') as q4_sales
FROM sales
GROUP BY product_id;

-- Standard SQL: CASE WHEN
SELECT 
  product_id,
  SUM(CASE WHEN quarter = 'Q1' THEN amount END) as q1_sales,
  SUM(CASE WHEN quarter = 'Q2' THEN amount END) as q2_sales
FROM sales
GROUP BY product_id;
```

### Unpivot (Columns to Rows)

```sql
-- PostgreSQL: LATERAL with VALUES
SELECT product_id, quarter, sales
FROM quarterly_sales,
LATERAL (VALUES 
  ('Q1', q1_sales),
  ('Q2', q2_sales),
  ('Q3', q3_sales),
  ('Q4', q4_sales)
) AS t(quarter, sales);

-- Standard: UNION ALL
SELECT product_id, 'Q1' as quarter, q1_sales as sales FROM quarterly_sales
UNION ALL
SELECT product_id, 'Q2', q2_sales FROM quarterly_sales
UNION ALL
SELECT product_id, 'Q3', q3_sales FROM quarterly_sales
UNION ALL
SELECT product_id, 'Q4', q4_sales FROM quarterly_sales;
```

## Pattern 12: Window Frame Specifications

### Frame Types

```sql
SELECT 
  date,
  value,
  -- Default: RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
  SUM(value) OVER (ORDER BY date) as running_total,
  
  -- ROWS: Physical rows
  AVG(value) OVER (ORDER BY date ROWS BETWEEN 2 PRECEDING AND CURRENT ROW) as moving_avg_3,
  
  -- RANGE: Logical range (same date values treated as peers)
  SUM(value) OVER (ORDER BY date RANGE BETWEEN INTERVAL '7 days' PRECEDING AND CURRENT ROW) as rolling_7d,
  
  -- Exclude current row
  AVG(value) OVER (ORDER BY date ROWS BETWEEN 2 PRECEDING AND 1 PRECEDING) as prev_2_avg
FROM metrics;
```

## Interview Question Checklist

Before answering, always clarify:

- [ ] What database platform? (Syntax varies)
- [ ] Table structure and sample data?
- [ ] Expected output format?
- [ ] NULL handling requirements?
- [ ] Performance constraints (data volume)?
- [ ] Edge cases (empty tables, duplicates)?

Common follow-up questions:

- "How would you optimize this for billions of rows?"
- "What indexes would help?"
- "How would you test this query?"
- "What if there are NULL values?"
