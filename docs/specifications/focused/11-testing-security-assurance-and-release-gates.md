# Focused Specification 11: Testing, Security Assurance and Release Gates

**Status:** DRAFT  
**Parent:** `../master-product-and-implementation-spec.md`  
**Gates:** gate-09-assurance, gate-10-v1  
**Requires:** gate-08-operations PASS  
**Related gaps:** GAP-004, GAP-036, GAP-040, GAP-048

---

## 1. Purpose

This specification defines how Inkstead proves its claims.

Security and offline correctness are release requirements, not documentation adjectives.

---

## 2. Canonical ASVS path

Use one case-sensitive path everywhere:

```text
docs/security/ASVS-5-MATRIX.md
```

This resolves the casing ambiguity discovered in master-spec review.

No lowercase duplicate should be created.

---

## 3. Test layers

Inkstead requires multiple test layers.

### Backend unit/integration

- pytest;
- pytest-django;
- Django test client/API client;
- database tests on PostgreSQL.

### Python static quality

- Ruff;
- mypy;
- pre-commit.

### Frontend

Gate 00 selects:

- TypeScript strict type checking;
- lint;
- React/component tests;
- crypto/storage unit tests.

### Browser E2E

- Playwright;
- Chromium;
- Firefox;
- WebKit.

### Security/property tests

- crypto fixtures;
- corrupted envelope tests;
- hostile HTML/import cases;
- sync interruption/convergence;
- plaintext canaries.

---

## 4. Local-first CI philosophy

Every mandatory check MUST have a repository-local command.

Cloud CI is an execution environment, not the source of truth.

A self-hosted/local runner must be able to run the same gates.

---

## 5. Stage-gate verifier

The verifier defined in Focused Specification 00 MUST enforce prerequisite evidence.

It should support:

```bash
uv run python scripts/verify_stage_gate.py --target gate-05-crdt-sync
```

Failure conditions:

- prerequisite missing;
- evidence FAIL;
- malformed evidence;
- unknown gate;
- dependency cycle;
- referenced evidence path absent.

Tests for the verifier are mandatory.

---

## 6. Evidence integrity

Evidence records a commit SHA.

A future enhancement SHOULD verify that the evidence commit contains or descends from the
implementation commit it claims to verify.

Evidence is not a substitute for rerunning tests, but it prevents invisible gate skipping.

---

## 7. Backend test policy

All security-sensitive server logic must include negative tests.

Examples:

- unauthorized object;
- wrong owner;
- revoked replica;
- invalid CSRF;
- oversized request;
- duplicate sequence conflict;
- unsupported protocol;
- malformed UUID;
- rate-limit path.

Happy-path-only tests are insufficient.

---

## 8. Frontend test policy

Security-sensitive client logic requires tests for:

- wrong vault key;
- corrupted ciphertext;
- lock/unlock;
- IndexedDB failure;
- quota failure;
- Service Worker update;
- multi-tab lock;
- unsafe pasted content;
- unsafe link schemes;
- search index corruption;
- migration interruption.

---

## 9. Crypto test vectors

Crypto fixtures are permanent compatibility contracts.

Every supported protocol version remains testable until deliberately retired through an ADR and
migration policy.

Do not regenerate old expected ciphertext casually when tests fail; that can hide a protocol break.

---

## 10. CRDT/sync fixtures

Keep deterministic scenarios for:

- concurrent edits;
- offline edits;
- delete/edit;
- stale replica;
- duplicate upload;
- server restore;
- purge markers;
- snapshot bootstrap.

A dependency upgrade of Automerge must pass existing fixtures before merge.

---

## 11. Plaintext canary

Permanent release gate.

Create unique markers and search:

- IndexedDB raw export;
- CacheStorage;
- HTTP traces;
- PostgreSQL dump;
- Redis inspection where practical;
- blob filesystem;
- Django logs;
- Nginx logs;
- Traefik logs;
- backup archive.

Expected occurrences are only deliberately decrypted test-client memory/output.

Any prohibited persistence occurrence is release-blocking.

---

## 12. XSS security suite

Test:

- script paste;
- event handlers;
- SVG payloads;
- iframe;
- javascript URLs;
- malformed HTML;
- DOM clobbering candidates;
- unsafe imported rich text.

CSP browser tests must prove execution is blocked even if sanitization regresses.

Defense in depth is intentional.

---

## 13. CSP/WASM regression

Every browser build test must prove:

- Automerge initializes;
- libsodium initializes;
- Crypto Worker works;
- ordinary eval/Function remains blocked;
- no unexpected remote scripts execute.

This prevents future dependency upgrades from silently forcing a weaker CSP.

---

## 14. Authentication assurance

Test:

- password login;
- TOTP;
- recovery code;
- passkey/WebAuthn on supported targets;
- admin MFA;
- enumeration behavior;
- brute-force/rate limits;
- trusted proxy;
- session revocation;
- account switch.

Inspect database to confirm protected MFA secret-at-rest behavior.

---

## 15. CSRF/session assurance

Test long-offline sequence:

1. unlock local journal;
2. let/force server session expire;
3. edit locally;
4. reconnect;
5. sync receives auth/CSRF failure;
6. outbox remains;
7. reauthenticate;
8. obtain fresh CSRF;
9. retry;
10. verify no duplicate/lost changes.

---

## 16. Offline E2E

Permanent scenario:

```text
online login
 -> unlock
 -> create
 -> offline
 -> edit
 -> close
 -> reopen offline
 -> unlock
 -> verify/edit
 -> reconnect
 -> sync
 -> second replica
 -> verify convergence
```

Run on all supported browser engines where practical.

---

## 17. Storage durability tests

Cover:

- `persist()` granted;
- denied;
- quota near full;
- quota exceeded;
- IndexedDB transaction abort;
- simulated corrupted record;
- site-data clear documentation/manual flow.

---

## 18. Service Worker tests

Cover:

- scope;
- cache inventory;
- offline shell;
- waiting update;
- compatible activation;
- incompatible schema;
- emergency unregister;
- stale cache cleanup.

Authentication/API responses must not appear in PWA cache.

---

## 19. Attachment security tests

Cover:

- allowed types;
- blocked unsafe types;
- image metadata stripping;
- huge file;
- upload interruption;
- IDOR;
- filename injection;
- HTML/SVG rendering prohibition;
- decrypted data cache inspection.

---

## 20. Import fuzz/property testing

Import parsers should receive:

- truncated archives;
- duplicate paths;
- path traversal;
- excessive nesting;
- malformed JSON;
- huge declared sizes;
- invalid rich text;
- duplicate UUIDs.

Existing vault must remain unchanged on failed import.

---

## 21. Backup/restore assurance

Release candidate requires:

- fresh backup;
- integrity check;
- destructive restore;
- existing-client reconciliation;
- new-client recovery;
- plaintext canary scan.

---

## 22. Migration assurance

Maintain fixtures/databases from previous supported release.

Test:

- server DB upgrade;
- Dexie upgrade;
- crypto protocol compatibility;
- CRDT schema compatibility;
- rollback refusal where unsafe;
- interrupted migration resume.

---

## 23. Performance assurance

Before v1 test:

- 50,000 entries;
- large tag set;
- multiple journals;
- several GB attachment scenario where feasible;
- cold search rebuild;
- new-replica bootstrap;
- long offline outbox;
- mobile browser memory.

Performance failures that create data-loss risk are security/reliability blockers.

---

## 24. Accessibility assurance

Automated tools may help but do not replace manual keyboard/screen-reader checks.

V1 target:

- WCAG 2.2 AA for core flows.

Test:

- account login/MFA;
- vault unlock;
- editor;
- navigation;
- search;
- export/deletion confirmation.

---

## 25. Dependency scanning

Select maintained tools during Gate 00/assurance setup.

Candidate capabilities:

- Python vulnerability scan;
- npm vulnerability/OSV scan;
- container scan;
- secret scan.

No one scanner is treated as complete.

---

## 26. Container scanning

Use a maintained scanner such as Trivy or equivalent.

Block stable release on unresolved critical vulnerabilities unless:

- not exploitable;
- documented;
- approved via risk record.

---

## 27. SBOM

Stable release SHOULD publish an SBOM.

Preferred formats:

- CycloneDX;
- SPDX.

Include application and container dependencies where tooling supports it.

---

## 28. Secret scanning

Repository history/PR checks should detect:

- private keys;
- tokens;
- passwords;
- accidentally committed env files.

Known test fixtures must be safely allowlisted rather than disabling scanning broadly.

---

## 29. ASVS 5.0

Maintain:

```text
docs/security/ASVS-5-MATRIX.md
```

Target:

- all applicable L1;
- all applicable L2;
- selected/applicable L3 for:
  - cryptography;
  - authentication;
  - session management;
  - client security;
  - sensitive data;
  - administrative operations;
  - deployment.

Each item is:

- PASS;
- N/A with rationale;
- ACCEPTED RISK with ADR/rationale;
- PLANNED (not allowed for mandatory stable-v1 target items).

---

## 30. Threat-model review

Threat model is reviewed:

- before crypto implementation;
- before sync;
- before attachments;
- before stable v1;
- after any architecture-changing feature.

New external integrations require threat-model update before implementation.

---

## 31. Security claims review

Before release, verify public documentation does not overclaim.

Specifically do not claim:

- protection against malicious browser extensions;
- guaranteed remote wipe;
- guaranteed memory zeroization;
- immediate erasure from historical backups;
- protection from a malicious web server serving altered JS unless separately achieved;
- server-side malware scanning of E2EE attachments when it cannot occur.

---

## 32. Release artifact integrity

Mature release process SHOULD provide:

- source tag;
- checksums;
- container image digest;
- SBOM;
- signed artifacts/provenance when implemented.

Build inputs use lockfiles.

---

## 33. Independent review roadmap

Before stable v1:

- internal architecture review;
- public threat model;
- public crypto spec;
- community review where possible.

After v1:

- targeted penetration test;
- cryptographic review;
- professional assessment when funding permits.

Findings must enter the gap register.

---

## 34. P1/P0 policy

Stable release cannot ship with unresolved P0/P1 gap unless:

- severity is re-evaluated with evidence;
- explicit ADR/risk acceptance exists;
- user safety is not misrepresented.

P1 is not "we will fix after launch."

---

## 35. Gate 09 required evidence

```text
docs/evidence/gate-09-assurance.md
```

Must include:

- full test matrix;
- browser matrix;
- ASVS status;
- threat-model review;
- vulnerability scans;
- container scan;
- SBOM result;
- canary results;
- migration compatibility;
- backup restore result.

---

## 36. Gate 10 v1 checklist

V1 requires:

- all earlier gates PASS;
- all blocking gaps CLOSED;
- product flows approved;
- documentation complete;
- recovery docs complete;
- installation docs complete;
- backup/restore docs complete;
- security claims reviewed;
- supported-browser matrix published;
- release artifacts verified.

---

## 37. Definition of stable v1

A user can:

1. self-host Inkstead;
2. establish trusted HTTPS;
3. securely authenticate;
4. initialize/recover a vault;
5. write entirely offline;
6. reopen offline;
7. use desktop/mobile responsive UI;
8. synchronize later;
9. converge multiple replicas;
10. use safe attachments;
11. search locally;
12. export/import;
13. delete/purge predictably;
14. back up and restore the server;
15. verify no journal plaintext is stored server-side during normal operation.

---

## 38. Exit criteria

Gate 09 passes when security assurance is complete for a release candidate.

Gate 10 passes when:

- all mandatory gates pass;
- release documentation matches tested reality;
- no blocking gap remains;
- the tagged release is reproducible and recoverable enough for supported use.
