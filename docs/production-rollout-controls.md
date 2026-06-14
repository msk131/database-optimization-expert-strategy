# Production Rollout Controls

Database optimization changes can improve performance, but they can also create regressions if released without controls.

## Change Types

| Change | Risk | Control |
| --- | --- | --- |
| SQL rewrite | Functional mismatch | Regression tests and result comparison |
| New index | DML slowdown or plan change | Workload impact review |
| Drop index | Query regression | Usage analysis and rollback script |
| Partitioning | Data movement or operational complexity | Rehearsal and runtime estimate |
| Statistics change | Plan instability | Baseline comparison and rollback strategy |
| Batch tuning | Partial processing or locking | Restartability and reconciliation |
| Oracle hint | Brittle plan control | Document SQL ID, plan hash, bind profile, and rollback query text |
| Parallel execution | CPU, IO, or temp starvation | Resource limits, batch-window approval, and OLTP wait-event monitoring |
| ORM persistence change | N+1 queries or excessive flush cost | Query-count tests, native SQL review, and production-like batch volume |
| ETL partition change | Wide scans or wrong partition load | Partition predicates, staged validation, and partition-level reconciliation |

## Release Checklist

- Baseline captured.
- SQL and execution plan reviewed.
- Bind values and cardinality assumptions captured.
- N+1 and ORM lazy-loading risk reviewed.
- Index read benefit compared against write-path cost.
- Partition pruning or partition targeting verified where relevant.
- Parallel query or DML degree approved where relevant.
- Functional test passed.
- Performance test passed with representative volume.
- Rollback script or compensation path exists.
- SQL Plan Management baseline or rollback plan exists for critical SQL.
- Monitoring dashboard is ready.
- Production support team is briefed.
- Post-release validation window is defined.

## Post-Release Validation

Measure:

- Top SQL elapsed time
- Buffer gets
- Physical reads
- CPU time
- Wait events
- ASH samples for short spikes hidden by broad AWR windows
- Blocking sessions and lock wait chains
- Plan hash changes for critical SQL
- Partition pruning behavior for ETL and batch queries
- Parallel server usage, temp pressure, and Resource Manager throttling
- Batch duration
- Error rate
- Lock wait trends
- Application response time

Do not close a database optimization release only because the application has no HTTP 500 errors. A release can be functionally healthy while silently increasing buffer gets, lock waits, temp spills, or full partition scans that only surface during the next heavy batch window.

## Rollback Expectations

Every change must define the exact rollback or compensation path before deployment:

| Change | Rollback Direction |
| --- | --- |
| New index | Drop index or mark invisible after plan impact review |
| Dropped index | Recreate with original definition and statistics policy |
| SQL rewrite | Restore previous SQL text or feature flag |
| SQL hint | Remove hint or restore previous hinted statement |
| Statistics change | Restore exported statistics or locked baseline statistics |
| Partition change | Use rehearsed partition exchange or online redefinition rollback plan |
| Parallel degree change | Revert object, statement, or job-level parallel settings |
