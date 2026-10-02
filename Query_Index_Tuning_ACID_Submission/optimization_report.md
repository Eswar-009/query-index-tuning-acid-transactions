# Query Index Tuning, Execution Plans & ACID Transactions

## Objective

Analyze slow database queries with `EXPLAIN ANALYZE`, identify sequential scans, create strategic B-tree, composite, and partial indexes, compare execution performance, and implement safe ACID transaction blocks for concurrent inventory updates.

## Database

The scripts are written for PostgreSQL 14+.

### Assumed schema

```sql
customers(
    customer_id BIGINT PRIMARY KEY,
    customer_name TEXT,
    region TEXT
);

orders(
    order_id BIGINT PRIMARY KEY,
    customer_id BIGINT,
    order_date DATE,
    status TEXT,
    total_amount NUMERIC(12,2)
);

products(
    product_id BIGINT PRIMARY KEY,
    product_name TEXT,
    category TEXT
);

inventory(
    product_id BIGINT PRIMARY KEY,
    quantity INTEGER NOT NULL,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

## 1. Generate 50,000+ simulated rows

The supporting SQL script inserts 60,000 customers and 120,000 orders. `ANALYZE` is then executed so the PostgreSQL planner has current statistics.

## 2. Baseline query and EXPLAIN ANALYZE

Example unindexed query:

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT order_id, customer_id, order_date, total_amount
FROM orders
WHERE customer_id = 25000
  AND order_date >= DATE '2026-01-01'
ORDER BY order_date DESC;
```

### Typical baseline plan

Before adding a suitable index, PostgreSQL may choose:

```text
Seq Scan on orders
  Filter: (customer_id = 25000 AND order_date >= '2026-01-01')
Sort
```

The important signal is the `Seq Scan`, which means PostgreSQL reads a large portion of the table to find matching rows.

> Exact execution time and row counts depend on the machine, PostgreSQL version, cache state, and generated data. Run the supplied commands to capture the actual values in your environment.

## 3. Composite B-tree index

Create:

```sql
CREATE INDEX idx_orders_customer_date
ON orders (customer_id, order_date DESC);
```

Then rerun:

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT order_id, customer_id, order_date, total_amount
FROM orders
WHERE customer_id = 25000
  AND order_date >= DATE '2026-01-01'
ORDER BY order_date DESC;
```

### Expected optimized plan

The planner can now use an index scan:

```text
Index Scan using idx_orders_customer_date on orders
  Index Cond: (customer_id = 25000
              AND order_date >= '2026-01-01')
```

This avoids scanning unrelated orders and can also avoid a separate sort because the index matches the filtering and ordering pattern.

## 4. B-tree index for status filtering

Query:

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT order_id, customer_id, total_amount
FROM orders
WHERE status = 'PENDING';
```

Index:

```sql
CREATE INDEX idx_orders_status
ON orders (status);
```

This is useful when status filtering is selective enough to justify index access.

## 5. Partial index

If the application frequently searches only pending orders:

```sql
CREATE INDEX idx_orders_pending_customer_date
ON orders (customer_id, order_date DESC)
WHERE status = 'PENDING';
```

Query:

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT order_id, customer_id, order_date, total_amount
FROM orders
WHERE status = 'PENDING'
  AND customer_id = 25000
ORDER BY order_date DESC;
```

A partial index stores only rows satisfying the predicate, reducing index size and potentially improving lookup performance.

## 6. Performance comparison

Record the actual values from `EXPLAIN ANALYZE`.

| Query | Before | After | Improvement |
|---|---:|---:|---:|
| Customer + date lookup | Run locally | Run locally | Calculate locally |
| Pending orders | Run locally | Run locally | Calculate locally |
| Pending customer history | Run locally | Run locally | Calculate locally |

### Calculation

```text
Improvement % =
((Before Execution Time - After Execution Time)
 / Before Execution Time) * 100
```

Do not use fabricated benchmark numbers. The actual execution times should be copied from your PostgreSQL `EXPLAIN ANALYZE` output.

## 7. ACID transaction design

Inventory updates must be atomic. The transaction script uses:

- `BEGIN` to start the transaction.
- `SELECT ... FOR UPDATE` to lock the inventory row.
- Validation to prevent negative stock.
- `UPDATE` to decrease inventory.
- `COMMIT` after all operations succeed.
- `ROLLBACK` when validation or another operation fails.

### Why `FOR UPDATE`?

Two concurrent transactions should not both read the same available quantity and then oversell the product. Row-level locking serializes updates to the same inventory record.

## 8. ACID properties demonstrated

### Atomicity
Either the complete inventory update succeeds or all changes are rolled back.

### Consistency
The transaction prevents inventory from becoming negative.

### Isolation
`FOR UPDATE` prevents conflicting concurrent transactions from modifying the same inventory row simultaneously.

### Durability
After `COMMIT`, PostgreSQL makes the committed transaction durable according to its WAL/recovery guarantees.

## 9. Deliverables

- `optimization_report.md` — analysis, execution-plan interpretation, indexes, benchmark procedure, and ACID explanation.
- `transaction_scripts.sql` — setup data, baseline queries, index creation, EXPLAIN ANALYZE statements, and safe transaction blocks.

## Submission checklist

- [x] 50,000+ simulated rows
- [x] EXPLAIN ANALYZE before indexing
- [x] B-tree index
- [x] Composite index
- [x] Partial index
- [x] Before/after performance measurement procedure
- [x] BEGIN / COMMIT transaction
- [x] ROLLBACK handling
- [x] Concurrent inventory update protection
- [x] Execution plan documentation
