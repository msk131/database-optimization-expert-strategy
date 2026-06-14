# Contributing

Contributions should improve practical database performance engineering guidance for Oracle-backed enterprise systems.

## Contribution Standards

- Tie recommendations to evidence such as SQL IDs, execution plans, AWR, ASH, wait events, cardinality, or production-like test results.
- Include operational tradeoffs, especially write-path cost, lock risk, rollback complexity, and observability requirements.
- Avoid generic tuning advice that cannot be validated.
- Prefer clear examples, checklists, and decision tables.
- Document when a technique should not be used.

## Pull Request Checklist

- The change explains the workload or failure mode it addresses.
- The guidance is reversible or includes a rollback path.
- Any Oracle hint, index, parallelism, or statistics recommendation includes risk notes.
- Links are relative and work inside the repository.
