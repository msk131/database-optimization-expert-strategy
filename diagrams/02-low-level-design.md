# Performance Governance Flow

```mermaid
flowchart TD
    PR[Pull Request]
    SQL[SQL Review]
    MIG[Migration Review]
    PERF[Performance Test]
    PLAN[Plan Comparison]
    APPROVE{Approved?}
    DEPLOY[Deploy]
    MONITOR[Post-Release Monitoring]
    REGRESS{Regression?}
    FIX[Rollback or Fix Forward]
    DONE[Done]

    PR --> SQL
    SQL --> MIG
    MIG --> PERF
    PERF --> PLAN
    PLAN --> APPROVE
    APPROVE -- Yes --> DEPLOY
    APPROVE -- No --> PR
    DEPLOY --> MONITOR
    MONITOR --> REGRESS
    REGRESS -- Yes --> FIX
    REGRESS -- No --> DONE
```

## Governance Controls

- Critical SQL reviewed before release.
- Database migrations dry-run in lower environments.
- Execution plan changes reviewed for high-volume queries.
- Post-release monitoring confirms expected improvement.

