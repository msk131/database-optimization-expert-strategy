# Enterprise Database Optimization Strategy

[![Oracle](https://img.shields.io/badge/Oracle-19c%2B-F80000?logo=oracle&logoColor=white)](https://www.oracle.com/database/)
[![Strategy](https://img.shields.io/badge/focus-performance%20engineering-2563eb)](docs/oracle-performance-playbook.md)
[![Contributions Welcome](https://img.shields.io/badge/contributions-welcome-16a34a)](CONTRIBUTING.md)

A production-minded, evidence-driven engineering strategy for analyzing, tuning, and safely modernizing legacy Oracle-backed, high-volume enterprise financial platforms.

---

## What This Project Solves

Regulated, high-volume environments often need the database to remain the system of record while modernization is already underway. This framework helps teams diagnose and eliminate:

- **Execution plan instability**: Prevent random plan flips with SQL Plan Management, statistics governance, and release gates.
- **Resource contention**: Reduce long batch windows, blocking sessions, lock waits, temp pressure, and runaway parallelism.
- **Suboptimal access patterns**: Remove `N+1` query loops, row-by-row processing, and accidental ORM fan-out.
- **Risky schema deployments**: Use online change patterns, rollback scripts, and post-release database validation.

---

## Architecture Pathway

```text
Production Telemetry
  Automatic Workload Repository (AWR), Active Session History (ASH), wait events, SQL IDs, bind values
        |
        v
Evidence-Based Diagnosis
  plan review, cardinality checks, blocker analysis
        |
        v
Controlled Remediation
  SQL rewrite, index strategy, stats policy, SPM baseline
        |
        v
Production Rollout
  online schema change, canary validation, rollback path
        |
        v
Post-Release Verification
  plan hash diff, waits, buffer gets, locks, batch duration
```

---

## Core Strategy Artifacts

| Artifact | Strategic Focus | Operational Value |
| :--- | :--- | :--- |
| [oracle-performance-playbook.md](docs/oracle-performance-playbook.md) | Evidence-driven tuning | SQL Plan Management, Automatic Workload Repository (AWR) and Active Session History (ASH) metrics, blocker capture, stats governance |
| [indexing-and-query-patterns.md](docs/indexing-and-query-patterns.md) | Data structures and access paths | Index tradeoffs, N+1 prevention, Oracle hints, parallel query, partition-aware ETL |
| [production-rollout-controls.md](docs/production-rollout-controls.md) | Deployment safety | Release checklists, rollback expectations, post-release database validation |
| [oracle-tuning-workflow.md](diagrams/oracle-tuning-workflow.md) | Tuning workflow | End-to-end path from business impact to regression prevention |
| [performance-governance-flow.md](diagrams/performance-governance-flow.md) | Governance flow | Pull request, migration review, plan comparison, monitoring, rollback |

---

## Core Engineering Principles

- **Telemetry over guesses**: Every performance action must map to Automatic Workload Repository (AWR), Active Session History (ASH), execution plans, wait events, bind values, and business impact.
- **Set-based dominance**: Prefer set-based SQL and staged database operations over application-layer iterative loops.
- **Deliberate indexing**: Every index must justify its read benefit against write cost, storage, redo, undo, and optimizer-plan risk.
- **Hints as exceptions**: Treat Oracle hints as controlled workarounds, not as standard production design.
- **Bulk APIs for scale**: Replace Hibernate `Session.save` loops with native SQL, jOOQ, JDBC batch, stored procedures, or Oracle-native bulk paths for high-throughput work.
- **Partition-aware ETL**: Target business-date partitions directly so batch jobs do not scan or load more data than necessary.
- **Rollback before release**: No tuning change is production-ready until the rollback or compensation path is documented.

---

## Getting Started

### Prerequisites

- Oracle Database 19c or higher
- Access to Automatic Workload Repository (AWR) and Active Session History (ASH), where licensed and available
- Representative production-scale data or a realistic performance test environment
- Permission to inspect execution plans, SQL IDs, wait events, and bind profiles

### Run A Diagnostics Strategy Audit

1. **Capture the business impact**

   Identify the affected API, report, batch window, financial process, or operational deadline.

2. **Extract workload performance metrics**

   Capture Automatic Workload Repository (AWR), Active Session History (ASH), SQL IDs, plan hashes, wait events, elapsed time, CPU time, buffer gets, physical reads, and bind samples for the affected window.

3. **Classify the bottleneck**

   Use [oracle-performance-playbook.md](docs/oracle-performance-playbook.md) to separate access-path problems, lock contention, stale statistics, N+1 patterns, temp pressure, and partition scans.

4. **Choose the smallest safe fix**

   Apply [indexing-and-query-patterns.md](docs/indexing-and-query-patterns.md) before adding indexes or hints. Confirm whether the issue is query shape, missing partition targeting, ORM fan-out, or optimizer drift.

5. **Release with controls**

   Use [production-rollout-controls.md](docs/production-rollout-controls.md) to confirm rollback scripts, plan comparison, post-release wait-event checks, and canary validation.

---

## Example Review Questions

- Does this query have a known SQL ID, plan hash, and bind profile?
- Is the proposed index selective enough to justify write-path overhead?
- Is an ORM loop causing `N+1` queries before the database even has a chance to optimize?
- Will an Oracle hint become brittle when data distribution changes?
- Does the ETL job target the correct partition window?
- Did post-release validation compare wait events, plan hash, locks, buffer gets, and batch duration?

---

## Contributing

Contributions to secure, highly available data-layer patterns are welcome. Start with [CONTRIBUTING.md](CONTRIBUTING.md), then propose changes with evidence, tradeoffs, and rollback guidance.
