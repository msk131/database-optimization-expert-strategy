# Indexing And Query Pattern Strategy

This guide defines when to use different indexing, query, hinting, and ETL patterns in Oracle-backed financial systems.

## Indexing Strategy

Indexes are not free. They speed selected reads but add storage, redo, undo, maintenance, and Data Manipulation Language (DML) cost. Add an index only when the workload evidence shows that the read path matters more than the write overhead.

| Strategy | Use When | Avoid When | Notes |
| --- | --- | --- | --- |
| B-tree index | Equality or range predicates are selective, joins use stable keys, and Online Transaction Processing (OLTP) reads need low latency | Column has very low selectivity, table is tiny, or DML pressure is more important than read speed | Default choice for account, transaction, customer, and reference lookups |
| Composite B-tree index | Queries repeatedly filter or join on the same ordered column set | Predicate order varies heavily or leading column is rarely filtered | Put the most selective and consistently used leading predicates first, unless range access requires a different order |
| Covering index | A critical query can be answered from index columns without table access | Index becomes wide enough to hurt DML and cache efficiency | Useful for hot read APIs and reconciliation lookups |
| Function-based index | Query filters on a deterministic expression such as `UPPER(account_number)` or date truncation | Application can store a normalized searchable column instead | Make sure session settings do not change expression semantics |
| Bitmap index | Read-mostly warehouse or reporting dimensions with low cardinality | High-concurrency OLTP tables | Bitmap indexes can cause severe locking and DML contention in transactional systems |
| Local partitioned index | Large partitioned tables where queries and ETL target business-date partitions | Global uniqueness across partitions is required without partition key support | Preferred for transaction, ledger, audit, history, and event tables partitioned by business date |
| Global index | Cross-partition lookup is business-critical and selective | Frequent partition maintenance would make global index upkeep risky | Use carefully; partition operations can invalidate or stress global indexes |
| Invisible index | You need to test optimizer behavior without exposing the index to all sessions | The team cannot control test sessions or plan capture | Useful for canary validation before making an index visible |

## When Not To Add An Index

- The query runs rarely and can use an existing batch/reporting path.
- The predicate returns a large percentage of the table.
- The table is small enough that full scan is cheaper.
- The query can be fixed by removing an N+1 pattern or rewriting a join.
- The index duplicates another index with the same useful leading columns.
- The extra DML cost would slow a critical write path such as payments, ledger posting, or account holds.

## N+1 Query Prevention

N+1 happens when application code loads one parent row and then issues one additional query per child or related row. It is one of the fastest ways to turn a healthy database into a latency amplifier.

| Symptom | Example | Preferred Fix |
| --- | --- | --- |
| Many repeated single-row selects | Load 500 accounts, then query balances 500 times | Join or fetch all balances with one `WHERE account_id IN (...)` statement |
| ORM lazy-loading storm | Transaction list triggers one select per customer, account, or ledger entry | Use explicit fetch joins, projections, or purpose-built read queries |
| Batch row loop | ETL reads IDs and calls the database once per ID | Use set-based SQL, staging tables, `MERGE`, or partition-wise processing |
| API fan-out | Endpoint builds response by calling repositories in loops | Create one read model query for the response shape |

For high-volume financial paths, prefer explicit read models over generic object graph traversal. The database should receive a small number of intentional queries, not thousands of accidental ORM queries.

## Hibernate And Native SQL Guidance

Hibernate is useful for ordinary transactional state changes, but it should not be the default tool for high-volume tuning, ETL, reconciliation, or batch posting.

| Workload | Avoid | Prefer |
| --- | --- | --- |
| Bulk insert or update | `Session.save` or entity loop with periodic flush | JDBC batch, jOOQ batch, Oracle direct-path insert, SQL loader, or staged `MERGE` |
| Reconciliation | Loading entities and summing in Java | Aggregate SQL grouped by account, partition, date, or transaction type |
| Ledger posting | Saving many detached objects one by one | One transaction with explicit SQL, strict ordering, and invariant checks |
| Reporting read model | Lazy-loaded entity graph | Native SQL projection, materialized view, or dedicated read table |
| ETL landing | ORM persistence context | Staging tables, external tables, partition exchange, and set-based validation |

Keep ORM sessions short. Do not let a persistence context grow across large files, partition loads, or batch windows. Long sessions increase memory pressure, dirty checking time, lock duration, and rollback cost.

## Oracle Hints

Hints can stabilize a critical query, but they can also hide stale statistics, bad indexing, or poor SQL shape. Use them only after evidence shows the optimizer needs guidance and after a no-hint solution has been considered.

| Hint Type | Use When | Risk |
| --- | --- | --- |
| `INDEX(table index_name)` | A proven selective index is being ignored because of plan drift or skew | Can force a bad path when data distribution changes |
| `FULL(table)` | Full scan is cheaper for a large-range analytic or batch query | Can crush OLTP if used on hot transactional tables |
| `LEADING` / `ORDERED` | Join order must be controlled for a critical query with known cardinality | Can become brittle as data volumes change |
| `USE_NL` | Small driving rowset joins to selective indexed lookups | Bad for large joins and can produce nested-loop explosions |
| `USE_HASH` | Large joins benefit from hash join and memory is available | Can create temp pressure if estimates are wrong |
| `PARALLEL(table n)` | Large read or batch operation can safely use multiple parallel slaves | Can starve OLTP, inflate CPU, and overload IO |
| `NO_PARALLEL` | Critical OLTP query must avoid accidental parallel execution | May slow legitimate analytic workloads |

Hints must be documented with the SQL ID, expected plan hash, before/after metrics, bind profile, and rollback rule. For durable plan control, prefer SQL Plan Management baselines over scattered application SQL hints.

## Parallel Query And Parallel DML

Parallelism is a capacity tradeoff, not a free speed boost.

Use parallel execution when:

- The operation scans or transforms a large partition or table.
- The job runs in a controlled batch window.
- CPU, IO, temp, and parallel server limits have headroom.
- The result is validated against serial execution.
- OLTP service-level objectives are protected with Resource Manager controls.

Avoid parallel execution when:

- The query is selective and should use an index.
- The system is already CPU or IO bound.
- The statement runs frequently from an API path.
- Parallel slaves would compete with payment, ledger, or customer-facing workloads.
- The query touches many hot blocks or lock-prone rows.

## ETL Partition Targeting

ETL should target partitions deliberately instead of scanning or loading the whole table.

Recommended patterns:

- Partition large transaction, ledger, audit, and history tables by business date.
- Load data into staging tables that match the target partition shape.
- Validate row counts, sums, keys, and duplicate checks in staging before publishing.
- Use partition exchange for large immutable loads when the data for a partition is complete and validated.
- Use local indexes where partition maintenance is frequent.
- Extract with date and partition predicates so jobs read only the intended business window.
- Run independent partitions in parallel when downstream constraints, IO, and reconciliation rules allow it.

Example partition-targeted extract:

```sql
SELECT /*+ PARALLEL(t 8) */
       t.transaction_id,
       t.account_id,
       t.amount,
       t.business_date
FROM transactions t
WHERE t.business_date >= DATE '2026-06-01'
  AND t.business_date <  DATE '2026-06-02';
```

Example staged load shape:

```sql
MERGE INTO ledger_entries target
USING staged_ledger_entries source
   ON (target.ledger_entry_id = source.ledger_entry_id)
WHEN NOT MATCHED THEN
  INSERT (
      ledger_entry_id,
      transaction_id,
      account_id,
      amount,
      business_date
  )
  VALUES (
      source.ledger_entry_id,
      source.transaction_id,
      source.account_id,
      source.amount,
      source.business_date
  );
```

## Review Checklist

- Does the query have a known SQL ID and plan hash?
- Are bind values and cardinality estimates captured?
- Is this a query-shape problem before it is an index problem?
- Will the index hurt critical writes?
- Is the access path partition-aware?
- Are ORM loops avoided on high-volume paths?
- Is any hint documented and reversible?
- Are before/after metrics captured for elapsed time, CPU, buffer gets, physical reads, rows processed, and wait events?
