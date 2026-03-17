# Execution Plan Analysis Guide

Deep dive into reading and optimizing SQL execution plans.

## Running EXPLAIN

### PostgreSQL

```sql
-- Basic plan (estimates only)
EXPLAIN SELECT * FROM orders WHERE status = 'pending';

-- Full analysis (actually runs query!)
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT) 
SELECT * FROM orders WHERE status = 'pending';

-- JSON format for tools
EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON) SELECT ...;

-- Verbose mode (more details)
EXPLAIN (ANALYZE, VERBOSE, BUFFERS) SELECT ...;
```

### MySQL

```sql
-- Basic plan
EXPLAIN SELECT * FROM orders WHERE status = 'pending';

-- Extended format
EXPLAIN FORMAT=JSON SELECT * FROM orders WHERE status = 'pending';

-- Analyze (MySQL 8.0.18+)
EXPLAIN ANALYZE SELECT * FROM orders WHERE status = 'pending';
```

### SQL Server

```sql
-- Estimated plan (doesn't run)
SET SHOWPLAN_XML ON;
GO
SELECT * FROM orders WHERE status = 'pending';
GO
SET SHOWPLAN_XML OFF;

-- Actual plan (runs query)
SET STATISTICS PROFILE ON;
SELECT * FROM orders WHERE status = 'pending';
SET STATISTICS PROFILE OFF;
```

## Understanding Plan Output

### PostgreSQL Example

```
Seq Scan on orders  (cost=0.00..1520.00 rows=50000 width=120) (actual time=0.015..25.432 rows=48532 loops=1)
  Filter: (status = 'pending')
  Rows Removed by Filter: 1468
  Buffers: shared hit=520
Planning Time: 0.085 ms
Execution Time: 28.234 ms
```

**Key Metrics:**

| Metric | Meaning |
|--------|---------|
| `cost=0.00..1520.00` | Startup cost..Total cost (arbitrary units) |
| `rows=50000` | Estimated rows |
| `actual rows=48532` | Actual rows (with ANALYZE) |
| `loops=1` | Number of times this node executed |
| `Buffers: shared hit=520` | Pages read from cache |
| `Rows Removed by Filter` | Rows scanned but rejected |

### Cost Model

```
Total Cost = (disk pages read * seq_page_cost) 
           + (rows scanned * cpu_tuple_cost)
           + (index scanned * cpu_index_tuple_cost)
           + (function calls * cpu_operator_cost)
```

Default PostgreSQL costs:
- `seq_page_cost = 1.0`
- `random_page_cost = 4.0`
- `cpu_tuple_cost = 0.01`
- `cpu_index_tuple_cost = 0.005`
- `cpu_operator_cost = 0.0025`

## Scan Types

### Sequential Scan

```
Seq Scan on large_table  (cost=0.00..18334.00 rows=1000000 width=36)
```

**When used**: No suitable index, or optimizer estimates seq scan faster
**Red flag if**: Table is large AND query is selective

**Fix options**:
1. Add appropriate index
2. Partition table
3. Check if statistics are stale (`ANALYZE table`)

### Index Scan

```
Index Scan using idx_orders_status on orders  (cost=0.43..8.45 rows=1 width=120)
  Index Cond: (status = 'pending')
```

**When used**: Index exists and query is selective enough
**Good**: Low cost, precise access

### Index Only Scan

```
Index Only Scan using idx_orders_status_date on orders  (cost=0.43..4.45 rows=1 width=12)
  Index Cond: (status = 'pending')
  Heap Fetches: 0
```

**Best case**: All needed columns in index (covering index)
**Watch**: `Heap Fetches` should be low (vacuum keeps visibility map fresh)

### Bitmap Index Scan

```
Bitmap Heap Scan on orders  (cost=4.48..35.12 rows=10 width=120)
  Recheck Cond: (status = 'pending')
  ->  Bitmap Index Scan on idx_status  (cost=0.00..4.48 rows=10 width=0)
        Index Cond: (status = 'pending')
```

**When used**: Multiple conditions OR moderate selectivity
**Two-phase**: Build bitmap from index, then fetch from heap

### Parallel Scans

```
Gather  (cost=1000.00..15000.00 rows=100000 width=36)
  Workers Planned: 4
  Workers Launched: 4
  ->  Parallel Seq Scan on large_table  (cost=0.00..14000.00 rows=25000 width=36)
```

**When used**: Large tables, complex operations
**Tune**: `max_parallel_workers_per_gather`

## Join Types

### Nested Loop

```
Nested Loop  (cost=0.43..16.52 rows=1 width=240)
  ->  Index Scan using idx_orders_id on orders  (cost=0.29..8.30 rows=1 width=120)
  ->  Index Scan using idx_items_order on items  (cost=0.14..8.16 rows=1 width=120)
        Index Cond: (order_id = orders.id)
```

**Best for**: Small outer table, indexed inner table
**Red flag**: High `loops` count on inner scan

### Hash Join

```
Hash Join  (cost=1.25..2.50 rows=10 width=240)
  Hash Cond: (orders.customer_id = customers.id)
  ->  Seq Scan on orders  (cost=0.00..1.10 rows=10 width=120)
  ->  Hash  (cost=1.10..1.10 rows=10 width=120)
        ->  Seq Scan on customers  (cost=0.00..1.10 rows=10 width=120)
```

**Best for**: Larger tables, equality joins
**Memory**: Builds hash table in memory (increase `work_mem` if needed)

### Merge Join

```
Merge Join  (cost=0.85..2.25 rows=10 width=240)
  Merge Cond: (orders.id = items.order_id)
  ->  Index Scan using orders_pkey on orders  (cost=0.29..1.00 rows=10 width=120)
  ->  Index Scan using idx_items_order on items  (cost=0.29..1.00 rows=10 width=120)
```

**Best for**: Pre-sorted data (from indexes), large datasets
**Requirement**: Both inputs sorted on join key

### Join Selection Guide

| Scenario | Best Join | Why |
|----------|-----------|-----|
| Small outer, indexed inner | Nested Loop | Few iterations, fast lookups |
| Large tables, equality | Hash Join | One-time hash build, O(n) lookups |
| Sorted data, large tables | Merge Join | No hash build, streaming |
| Cross join (intentional) | Nested Loop | Must examine all pairs |

## Aggregation Operations

### HashAggregate

```
HashAggregate  (cost=1.15..1.20 rows=5 width=12)
  Group Key: status
  ->  Seq Scan on orders  (cost=0.00..1.10 rows=10 width=4)
```

**When used**: Few groups relative to rows
**Memory**: Hash table size depends on group count

### GroupAggregate

```
GroupAggregate  (cost=0.29..1.50 rows=5 width=12)
  Group Key: status
  ->  Index Scan using idx_status on orders  (cost=0.29..1.00 rows=10 width=4)
```

**When used**: Data already sorted by group key
**Better when**: Many groups, limited memory

## Sort Operations

### Sort

```
Sort  (cost=1.27..1.29 rows=10 width=120)
  Sort Key: created_at DESC
  Sort Method: quicksort  Memory: 25kB
  ->  Seq Scan on orders  (cost=0.00..1.10 rows=10 width=120)
```

**Watch for**:
- `Sort Method: external merge Disk: XXkB` = spilled to disk
- High memory usage = increase `work_mem`

### Top-N Sort

```
Limit  (cost=1.20..1.22 rows=10 width=120)
  ->  Sort  (cost=1.20..1.45 rows=100 width=120)
        Sort Key: created_at DESC
```

**Optimization**: Only keeps top N rows in memory

## Common Problems & Fixes

### Problem 1: Seq Scan on Large Table

```
Seq Scan on orders  (cost=0.00..185000.00 rows=5000000 width=120)
  Filter: (status = 'pending')
  Rows Removed by Filter: 4900000
```

**Diagnosis**: Scanning 5M rows, keeping only 100K
**Fixes**:
1. `CREATE INDEX idx_status ON orders(status);`
2. Check if statistics outdated: `ANALYZE orders;`
3. Partition by status if appropriate

### Problem 2: Nested Loop with High Loops

```
Nested Loop  (cost=0.29..50000.00 rows=10000 width=240)
  ->  Seq Scan on orders  (cost=0.00..1000.00 rows=10000 width=120)
  ->  Index Scan on items  (cost=0.29..4.00 rows=1 width=120) (actual loops=10000)
```

**Diagnosis**: 10,000 index lookups
**Fixes**:
1. Add composite index
2. Force Hash Join: `SET enable_nestloop = off;` (temporary)
3. Rewrite query to batch lookups

### Problem 3: Hash Join Spill to Disk

```
Hash Join  (cost=25000.00..50000.00 rows=100000 width=240)
  ->  Hash  (cost=20000.00..20000.00 rows=500000 width=120)
        Buckets: 65536  Batches: 16  Memory Usage: 32MB
```

**Diagnosis**: `Batches > 1` means spilling to disk
**Fixes**:
1. Increase `work_mem` for session: `SET work_mem = '256MB';`
2. Add filter to reduce hash table size
3. Consider pre-aggregation

### Problem 4: Sort Spill

```
Sort  (cost=50000.00..55000.00 rows=1000000 width=120)
  Sort Method: external merge  Disk: 125000kB
```

**Diagnosis**: 125MB sorted on disk
**Fixes**:
1. Increase `work_mem`
2. Add index on sort columns
3. Use LIMIT if only top rows needed

### Problem 5: Bitmap Heap Scan with High Recheck

```
Bitmap Heap Scan on orders  (cost=100.00..5000.00 rows=10000 width=120)
  Recheck Cond: (status = 'pending')
  Rows Removed by Index Recheck: 50000
```

**Diagnosis**: Index returns lossy bitmap, many rechecks
**Fixes**:
1. Increase `work_mem` for larger bitmap
2. Make index more selective
3. Consider different index type

## Optimization Workflow

```
Step 1: Get Baseline
┌─────────────────────────────────────────┐
│ EXPLAIN (ANALYZE, BUFFERS)              │
│ SELECT ... FROM ... WHERE ...;          │
│                                         │
│ Note: execution time, rows, buffers     │
└─────────────────────────────────────────┘
                    │
                    ▼
Step 2: Identify Bottleneck
┌─────────────────────────────────────────┐
│ Look for:                               │
│ - Seq Scan on large tables              │
│ - High loops in Nested Loop             │
│ - Disk spills (external sort/hash)      │
│ - Large gap: estimated vs actual rows   │
└─────────────────────────────────────────┘
                    │
                    ▼
Step 3: Apply Fix
┌─────────────────────────────────────────┐
│ - Add/modify index                      │
│ - Update statistics (ANALYZE)           │
│ - Rewrite query                         │
│ - Adjust work_mem/other settings        │
└─────────────────────────────────────────┘
                    │
                    ▼
Step 4: Validate
┌─────────────────────────────────────────┐
│ Re-run EXPLAIN (ANALYZE, BUFFERS)       │
│ Compare: time, buffers, plan shape      │
│ Test with production-like data volume   │
└─────────────────────────────────────────┘
```

## Quick Reference: Plan Node Red Flags

| Node | Red Flag | Action |
|------|----------|--------|
| Seq Scan | Large table, selective filter | Add index |
| Nested Loop | High loops (>1000) | Check indexes, consider Hash Join |
| Hash Join | Batches > 1 | Increase work_mem |
| Sort | External merge | Increase work_mem, add index |
| Bitmap Heap Scan | High recheck ratio | More selective index |
| Gather | Workers Launched < Planned | Check max_parallel_workers |
| Any | actual rows >> estimated rows | Run ANALYZE |

## PostgreSQL Performance Views

```sql
-- Slow queries
SELECT query, calls, mean_exec_time, total_exec_time
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 10;

-- Table scan stats
SELECT relname, seq_scan, seq_tup_read, idx_scan, idx_tup_fetch
FROM pg_stat_user_tables
ORDER BY seq_tup_read DESC;

-- Index usage
SELECT indexrelname, idx_scan, idx_tup_read, idx_tup_fetch
FROM pg_stat_user_indexes
ORDER BY idx_scan DESC;

-- Cache hit ratio
SELECT 
  sum(heap_blks_hit) / (sum(heap_blks_hit) + sum(heap_blks_read)) as cache_hit_ratio
FROM pg_statio_user_tables;
```
