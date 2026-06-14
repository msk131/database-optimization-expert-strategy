# Database Optimization Expert Strategy

This project defines how Oracle-backed enterprise financial systems can be analyzed, tuned, and changed safely before and during modernization.

## What This Project Solves

Legacy financial platforms often suffer from slow Structured Query Language (SQL), unstable execution plans, long batch windows, locking, blocking, N+1 access patterns, row-by-row extract-transform-load (ETL), and risky schema changes. This project provides a controlled approach for diagnosing database bottlenecks, improving performance, and releasing tuning changes without turning the database into the outage source.

The strategy is aimed at regulated, high-volume environments where the database is still the system of record and performance changes must be explainable, reversible, and measurable.

## Key Artifacts

| Artifact | Purpose |
| --- | --- |
| [oracle-performance-playbook.md](docs/oracle-performance-playbook.md) | Evidence-driven SQL tuning, execution plan review, statistics governance, and online change strategy |
| [indexing-and-query-patterns.md](docs/indexing-and-query-patterns.md) | Index selection rules, N+1 prevention, Oracle hints, ETL partition targeting, and native SQL guidance |
| [production-rollout-controls.md](docs/production-rollout-controls.md) | Release checklist, rollback expectations, and post-release validation |
| [oracle-tuning-workflow.md](diagrams/oracle-tuning-workflow.md) | Tuning workflow diagram |
| [performance-governance-flow.md](diagrams/performance-governance-flow.md) | Performance governance flow |

## Operating Principles

- Tune from AWR, ASH, execution plans, wait events, bind values, and business impact, not from guesses.
- Prefer set-based SQL over row-by-row application loops.
- Use indexes deliberately; every index must justify read benefit against write, storage, and optimizer-plan cost.
- Treat Oracle hints as controlled exceptions, not as the default tuning method.
- Keep ETL jobs partition-aware so large loads and extracts target the smallest safe data slice.
- Avoid Hibernate `Session.save` loops for high-volume database work; use native SQL, jOOQ, JDBC batch, stored procedures, or database-native bulk APIs where correctness and throughput require it.
- Validate every change with production-like volume and a rollback path.

## Outcome

The goal is to reduce migration risk by stabilizing the legacy database, improving critical workloads, and preventing performance regressions during phased rollout.
