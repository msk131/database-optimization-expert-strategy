# Oracle Performance Playbook

## Diagnostic Workflow

1. Confirm business impact.
2. Capture baseline metrics.
3. Identify top SQL and wait events.
4. Review execution plans and table/index statistics.
5. Identify the smallest safe change.
6. Validate with production-like data volume.
7. Deploy with monitoring and rollback.
8. Compare post-release metrics to baseline.

## Evidence To Capture

| Evidence | Why It Matters |
| --- | --- |
| SQL ID | Links analysis to the exact statement |
| Execution plan | Shows join method, access path, and cost model |
| Elapsed time | Measures user or batch impact |
| CPU time | Identifies compute pressure |
| Buffer gets | Shows logical IO intensity |
| Physical reads | Shows disk IO pressure |
| Rows processed | Helps validate cardinality and selectivity |
| Wait events | Shows where sessions spend time |
| Bind variables | Explains plan variation and selectivity |
| Table/index stats | Determines optimizer quality |
| Active Session History (ASH) samples | Captures short lock, latch, CPU, and IO spikes that may be averaged out of hourly AWR snapshots |
| Blocking session details | Identifies root blockers instead of only the sessions waiting behind them |

## Common Patterns

| Symptom | Likely Cause | Possible Fix |
| --- | --- | --- |
| High buffer gets | Poor access path | Index review, predicate rewrite |
| Full table scan | Missing index or low selectivity | Index, partitioning, query rewrite |
| Plan instability | Bind peeking, skew, stale stats | Stats strategy, SQL plan management |
| Slow batch | Row-by-row processing | Set-based SQL, batching, commit tuning |
| Lock waits | Long transactions or hot rows | Transaction redesign, concurrency control |
| Temp pressure | Large sorts or hash joins | Query rewrite, indexing, memory review |
| High parse time | Literal SQL or poor cursor reuse | Bind variables, cursor sharing review |
| N+1 query pattern | ORM lazy loading or repository loops | Explicit joins, projections, native SQL, or set-based fetch |
| Parallel query regression | Parallel slaves competing with OLTP | Resource Manager limits, lower degree, or `NO_PARALLEL` on critical paths |
| Partition scan too wide | Missing business-date predicate or non-partition-aware ETL | Partition pruning, staged load, partition exchange, targeted extract |

## Tuning Principles

- Tune from evidence, not guesswork.
- Fix the biggest bottleneck first.
- Prefer small, reversible changes.
- Avoid adding indexes without workload impact review.
- Validate functional equivalence after SQL rewrites.
- Compare before-and-after metrics.
- Monitor production after release.

## SQL Plan Baseline And Optimizer Statistics Governance

| Control | Standard |
| --- | --- |
| Critical SQL list | Maintain top 20 Structured Query Language (SQL) statements by elapsed time, central processing unit (CPU), buffer gets, physical reads, and business criticality |
| Plan capture | Store SQL identifier, plan hash, bind profile, row counts, and before/after metrics |
| Baseline trigger | Use SQL Plan Management when a critical query has repeated plan regression or cardinality instability |
| Statistics refresh | Use non-blocking scheduled statistics refresh with production-safe sampling and runtime limits |
| Large partitioned tables | Use incremental statistics and refresh only changed partitions where possible |
| Stable core schemas | Lock statistics for critical tables when plan stability is more important than automatic plan adaptation |
| Histogram policy | Use histograms only where data skew is proven and monitored |
| Deployment gate | Block release when a critical plan hash changes without approval |

Hourly Automatic Workload Repository (AWR) reports are not enough for high-frequency financial batches. A row-lock spike that lasts under a minute can be averaged into the broader snapshot. Supplement AWR with ASH, targeted `v$session` sampling, and blocker capture during batch windows and release validation.

## Online Index And Partition Strategy

| Change | Production-Safe Direction |
| --- | --- |
| New index | Create online where supported; validate Data Manipulation Language (DML) overhead before release |
| Index rebuild | Rebuild online during approved low-risk window and monitor locks, undo, and input/output |
| Partition conversion | Use online redefinition where supported and rehearse with production-like volume |
| Partition key | Prefer business date for transaction, audit, ledger, and history tables |
| Archive policy | Move aged partitions to approved cold storage only after retention and audit review |

Use `DBMS_REDEFINITION` for high-risk table shape changes on live financial tables. Standard DDL can request locks that block behind long reports and then stall incoming writes. Online redefinition lets the team rehearse, synchronize, and swap metadata with a much shorter final lock window.

## Blocking Lock Interceptor

Use this query during release windows or batch incidents to find the root blocker and the SQL waiting behind it:

```sql
SELECT
    blocker.sid AS blocker_session_id,
    blocker.username AS blocker_user,
    blocker.program AS blocker_application,
    waiter.sid AS stalled_session_id,
    waiter.wait_time_micro / 1000000 AS wait_duration_seconds,
    sql_text.sql_fulltext AS stalled_sql_query
FROM v$session blocker
JOIN v$session waiter
  ON blocker.sid = waiter.blocking_session
JOIN v$sql sql_text
  ON waiter.sql_id = sql_text.sql_id
WHERE waiter.blocking_session IS NOT NULL
ORDER BY wait_duration_seconds DESC;
```

Capture the result with timestamp, release identifier, batch name, module, action, and service name so the incident can be traced back to the exact workload.

## Regression Detection

Every database change must compare:

- Plan hash value.
- Elapsed time.
- Buffer gets per execution.
- Physical reads.
- Rows processed.
- Lock wait time.
- Batch duration.
