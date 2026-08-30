# Inkstead Gate Evidence

This directory stores durable verification evidence for Inkstead implementation gates.

Evidence is created during implementation. Do not mark a gate PASS before the relevant focused
specification is approved and every mandatory check has been executed.

## Required metadata

Each gate evidence file begins with:

```yaml
gate: gate-XX-name
status: PASS
implementation_commit: <git-sha>
evidence_commit: <git-sha-or-pending-until-commit>
verified_at: <ISO-8601 timestamp>
environment: <summary>
```

## Required sections

- Scope
- Prerequisite gates
- Implementation commit
- Commands executed
- Automated test results
- Manual flow results
- Security checks
- Data/plaintext inspections
- Known warnings
- ADRs/deviations
- Final result

## Rule

Missing evidence means the gate has not passed.

A narrative statement such as "tested successfully" without commands/results is insufficient.
