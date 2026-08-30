# Inkstead Focused Specification and Stage-Gate Framework

**Status:** Baseline process specification  
**Parent:** `../master-product-and-implementation-spec.md`  
**Purpose:** Decompose the master specification into reviewable, testable, dependency-ordered specifications without modifying the master document.

---

## 1. Why this framework exists

The master specification intentionally describes the complete Inkstead product. It is too broad to
use safely as a single implementation checklist.

Inkstead will therefore develop through focused specifications. Each focused specification defines:

- one bounded architectural area;
- its upstream assumptions;
- decisions that must be made before implementation;
- security and privacy invariants;
- user flows;
- failure flows;
- required tests;
- objective exit criteria;
- evidence that must be committed before the next implementation stage can begin.

Later focused specifications may exist in draft form so the whole architecture can be inspected.
However, a later stage MUST NOT be approved for implementation until every prerequisite gate has
passed.

The master specification remains unchanged and serves as the umbrella requirement. Focused
specifications refine it. If a focused specification needs to contradict or replace a normative
master requirement, an ADR is required.

---

## 2. Specification state machine

Every focused specification uses one of these states:

```text
DRAFT
  |
  v
IN REVIEW
  |
  v
APPROVED FOR IMPLEMENTATION
  |
  v
IMPLEMENTED
  |
  v
VERIFIED
```

### DRAFT

The scope exists but may contain unresolved decisions.

### IN REVIEW

The focused design is actively being challenged against:

- user flows;
- failure flows;
- threat model;
- browser/runtime constraints;
- upstream framework behavior;
- maintained package capabilities;
- previous gate evidence.

### APPROVED FOR IMPLEMENTATION

All blocking decisions are resolved.

No implementation work for a stage should begin before this state.

### IMPLEMENTED

The required code/configuration exists on a dedicated implementation branch.

### VERIFIED

Every mandatory gate check has passed and evidence has been committed.

Only VERIFIED satisfies a downstream gate dependency.

---

## 3. Evidence contract

Each implementation gate produces a durable evidence document:

```text
docs/evidence/gate-00-foundation.md
docs/evidence/gate-01-security-auth.md
docs/evidence/gate-02-pwa-offline.md
...
```

Evidence documents MUST record:

- gate ID;
- implementation commit SHA;
- tested environment;
- dependency lockfile hashes or commit state;
- commands executed;
- test results;
- security checks;
- manual verification performed;
- known warnings;
- deviations/ADRs;
- reviewer/approval state;
- final PASS or FAIL.

Screenshots MAY supplement evidence but MUST NOT replace executable checks.

A gate is not passed because a developer writes "works for me."

---

## 4. Enforcement model

Inkstead will implement a repository-local gate verifier during Gate 00.

The target command is:

```text
uv run python scripts/verify_stage_gate.py --target <gate-id>
```

The verifier MUST:

1. read `docs/specifications/focused/stage-gates.yaml`;
2. determine the prerequisite gates;
3. confirm each prerequisite evidence file exists;
4. confirm each evidence file declares `status: PASS`;
5. confirm evidence references a real repository commit;
6. fail closed when required evidence is missing or malformed.

The verifier SHOULD later be wired into:

- local `just` commands;
- pre-merge CI;
- self-hosted CI runners;
- pull-request checks.

Cloud CI availability MUST NOT be required to run the gate locally.

---

## 5. Branch discipline

Suggested branch convention:

```text
spec/NN-<area>
phase/NN-<implementation>
fix/NN-<issue>
```

Each implementation PR MUST identify:

- focused specification;
- gate being satisfied;
- prerequisite evidence;
- required verification commands.

Architecture-changing implementation MUST include an ADR when required.

---

## 6. Focused specification sequence

| Order | Focused specification | Implementation gate |
| --- | --- | --- |
| 00 | Framework and gap register | Gate 00 |
| 01 | Product scope and end-to-end user flows | Gate 00 dependency |
| 01A | Security/privacy threat model and data classification | Gate 00 dependency |
| 02 | Cookiecutter platform foundation | Gate 00 |
| 03 | Authentication, sessions and browser security | Gate 01 |
| 04 | PWA, local storage, offline and update lifecycle | Gate 02 |
| 05 | Cryptography, key management and recovery | Gate 03 |
| 06 | Journal domain, editor and metadata semantics | Gate 04 |
| 07 | CRDT, sync, replica and compaction protocol | Gate 05 |
| 08 | Attachments, media and file security | Gate 06 |
| 09 | Search, export, import, deletion and data lifecycle | Gate 07 |
| 10 | Deployment, TLS, backup and operations | Gate 08 |
| 11 | Security assurance, release verification and v1 gates | Gate 09/10 |

The numbering of specifications and implementation gates is intentionally not identical in every
case. The machine-readable manifest is authoritative for prerequisites.

---

## 7. Development order

The implementation order is:

```text
verified Cookiecutter baseline
        |
        v
auth/server/browser security
        |
        v
PWA shell and durable local storage behavior
        |
        v
cryptographic vault
        |
        v
local journal/editor
        |
        v
CRDT semantics
        |
        v
server synchronization
        |
        v
attachments
        |
        v
search/data lifecycle/export
        |
        v
backup/restore/operations
        |
        v
full security assurance and v1
```

A later implementation stage MUST NOT be used to hide a failed earlier stage.

For example:

- synchronization MUST NOT be built to compensate for broken local persistence;
- backup MUST NOT be used to compensate for unreliable sync;
- UI polish MUST NOT conceal cryptographic recovery problems.

---

## 8. Review order

The focused documents are created now as drafts so architectural cross-dependencies can be seen.

They will then be refined one at a time.

Recommended review loop:

1. select the next focused spec;
2. inspect current repository/upstream package behavior;
3. walk every happy-path user flow;
4. walk offline/failure/security flows;
5. challenge the design against the gap register;
6. resolve blocking decisions;
7. mark the focused spec APPROVED FOR IMPLEMENTATION;
8. implement;
9. execute gate checks;
10. commit evidence;
11. mark spec VERIFIED;
12. only then approve the next implementation stage.

---

## 9. Mandatory cross-cutting checks

Every focused specification review MUST answer:

### Data

- What plaintext exists?
- Where can it exist?
- For how long?
- What is persisted?
- What is logged?
- What metadata leaks by design?
- How is corruption detected?
- How is migration handled?

### Identity

- Who is acting?
- What proves authorization?
- What happens offline?
- What happens after logout?
- What happens after account switch?
- What happens after replica/session revocation?

### Failure

- What if the server disappears?
- What if the browser closes mid-operation?
- What if storage quota is exhausted?
- What if the tab crashes?
- What if two tabs act concurrently?
- What if an old client reconnects?
- What if a dependency upgrade changes serialization?

### Security

- What can XSS do?
- What can a stolen browser profile reveal?
- What can a stolen database reveal?
- What can a malicious attachment do?
- What can a compromised reverse proxy do?
- Which assumption cannot be enforced technically?

### Recovery

- Can the state be reconstructed?
- Which keys are required?
- What if one replica is stale?
- What if a backup predates a crypto/schema migration?
- Can deletion accidentally be undone?

### User experience

- Does security interrupt writing?
- Is a warning actionable?
- Can users understand whether data is only local versus synchronized?
- Is a destructive operation explicit?
- Does the flow still work on mobile?

---

## 10. Stop conditions

Development MUST stop at the current gate when any of the following is true:

- a cryptographic invariant is ambiguous;
- required browser behavior has not been proven on a target browser;
- prior gate evidence is missing;
- a test demonstrates possible silent data loss;
- plaintext appears in a prohibited persistence/logging location;
- a database/schema migration lacks backward-compatibility evidence;
- a security dependency is unmaintained or incompatible;
- a required backup cannot be restored;
- a known P1 security/design issue remains unresolved.

The correct response to a failed gate is to fix the current stage, not to proceed and promise to
"circle back."

---

## 11. Relationship to PR #7 review findings

The review of the master-specification PR identified several concrete gaps:

1. random document/blob keys need synchronized wrapped-key envelopes;
2. zero-knowledge snapshot compaction cannot trust an unverifiable client coverage marker;
3. CSP must explicitly accommodate WebAssembly or the selected WASM dependencies must be replaced;
4. ASVS matrix path casing must be normalized.

These are recorded in the gap register and resolved in the relevant focused specifications without
editing the master specification.

---

## 12. Definition of a passed gate

A gate is PASS only when:

- all prerequisite gates are PASS;
- its focused spec is approved;
- implementation is present;
- automated checks pass;
- mandatory manual flows pass;
- security checks pass;
- evidence is committed;
- no unresolved blocking gap remains.

"Mostly passing" is FAIL.
