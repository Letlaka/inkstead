# Inkstead Architecture Gap Register

**Status:** Active  
**Parent:** `../master-product-and-implementation-spec.md`  
**Purpose:** Track gaps discovered while re-vetting the master specification. The master document is intentionally left unchanged.

---

## Severity

- **P0:** immediate catastrophic/data-loss/security blocker.
- **P1:** must be resolved before affected implementation.
- **P2:** important design gap; must be resolved before stable v1.
- **P3:** improvement/future hardening.

---

## Open and routed gaps

| ID | Sev | Gap | Focused specification / resolution target |
| --- | --- | --- | --- |
| GAP-001 | P1 | Random per-entry/per-blob keys are not yet defined as wrapped, synchronized objects, so a second replica could receive ciphertext without the key needed to decrypt it. | 05 + 07 |
| GAP-002 | P1 | Snapshot compaction cannot trust a client-supplied "through sequence" marker because the zero-knowledge server cannot inspect whether the snapshot actually contains those changes. | 07 |
| GAP-003 | P1 | Proposed CSP `script-src 'self'` blocks WebAssembly compilation used by likely Automerge/libsodium builds. | 03 + 04 |
| GAP-004 | P2 | ASVS matrix path is inconsistently cased in the master spec. Linux treats the variants as distinct files. | 11 |
| GAP-005 | P1 | IndexedDB is best-effort storage by default and can be evicted. The offline contract needs persistent-storage request/status/quota behavior. | 04 |
| GAP-006 | P1 | Multiple same-origin tabs can race synchronization, replica sequence allocation, migrations, and vault state. | 04 + 07 |
| GAP-007 | P1 | Service Worker update behavior, version skew, cache migration, and emergency unregister/kill-switch are not specified. | 04 |
| GAP-008 | P1 | App version, API version, Dexie schema, CRDT schema, and crypto protocol need an explicit compatibility matrix and upgrade ordering. | 04 + 07 + 11 |
| GAP-009 | P1 | New-replica bootstrap is incomplete: authenticated new browsers need a defined path to obtain/unwrap the Vault Master Key. | 05 |
| GAP-010 | P1 | Replica/session revocation cannot erase keys or plaintext already copied to a lost device. Compromise/rekey semantics need to be explicit. | 03 + 05 + 07 |
| GAP-011 | P2 | JavaScript cannot guarantee cryptographic key zeroization because of garbage collection. The security claim must be best-effort, using worker termination and reference minimization. | 05 |
| GAP-012 | P2 | Metadata leakage profile is not documented: object counts, ciphertext sizes, timestamps, access patterns, account data, and sync timing remain server-visible. | 03 + 05 + 09 |
| GAP-013 | P1 | Local IndexedDB contents are untrusted and can be modified/corrupted. AEAD verification, corruption quarantine, and resync/recovery flows need specification. | 04 + 05 + 07 |
| GAP-014 | P2 | Server rollback/withholding is only partly detectable. Existing replicas can compare known cursors/heads; a completely new replica lacks prior state. | 07 + threat model |
| GAP-015 | P1 | Long-offline/stale replicas can resurrect deleted data unless tombstone retention, replica expiry, and full-resync rules are defined. | 07 + 09 |
| GAP-016 | P1 | E2EE attachments prevent normal server-side malware/type inspection. Attachment policy must reconcile privacy with file-security requirements. | 08 |
| GAP-017 | P1 | Local save needs crash-consistent transactional outbox semantics so encrypted local persistence and pending sync cannot diverge. | 04 + 07 |
| GAP-018 | P1 | A long-offline same-origin PWA needs a defined CSRF/session refresh path when the Django session expires. | 03 + 07 |
| GAP-019 | P1 | Multiple user accounts using the same browser origin need local vault namespace isolation and mandatory vault lock on account switching. | 03 + 04 |
| GAP-020 | P1 | Browser/site-data clearing and quota exhaustion can destroy unsynced local-only data. The UI needs persistence status, storage pressure handling, and clear sync/backup state. | 04 |
| GAP-021 | P2 | Background/task-switcher screenshots can expose visible plaintext even if the vault key remains within its grace period. | 03 + 04 |
| GAP-022 | P2 | Browser history, document titles, referrers, URLs, notifications, and error reports can leak entry/journal metadata unless explicitly constrained. | 03 + 06 |
| GAP-023 | P2 | Browser extensions, OS malware, input methods, screen capture, and a compromised browser remain outside the app's protection boundary. | 03 + threat model |
| GAP-024 | P1 | Frontend build reproducibility is incomplete: generated Docker currently uses npm install and the initial repository has no committed package-lock. | 02 + 11 |
| GAP-025 | P2 | Operational backups contain account/security metadata even if journals are E2EE. Backup archives themselves need encryption/access-control policy. | 10 |
| GAP-026 | P1 | Secure first-user bootstrap/public-registration/email behavior is not fully defined for an Internet-independent deployment. | 03 |
| GAP-027 | P1 | allauth TOTP secrets are stored server-side and need encryption through the documented MFA adapter hook. | 03 |
| GAP-028 | P1 | allauth IP rate limiting depends on correctly configured trusted proxy count/header behind Traefik. | 03 + 10 |
| GAP-029 | P1 | Account/journal deletion must cascade on the server, while documentation must state that previously synchronized local copies cannot be remotely erased. | 09 |
| GAP-030 | P2 | Journal date semantics versus security/sync timestamps are underspecified. Client clocks are untrusted for security ordering. | 06 + 07 |
| GAP-031 | P2 | "Sync on the same network" should be implemented as reachability of the configured private origin, not fragile LAN-detection logic. | 04 + 10 |
| GAP-032 | P1 | Service Worker/app-shell upgrades can race IndexedDB/crypto migrations and leave incompatible code controlling open tabs. | 04 |
| GAP-033 | P2 | Failed/abandoned attachment uploads can create orphan encrypted blobs; garbage collection needs safe ownership/reference rules. | 08 + 09 |
| GAP-034 | P1 | Every API object query needs explicit owner/journal authorization and IDOR tests. | 03 + 07 + 08 |
| GAP-035 | P1 | Sync and blob APIs need pagination, request/body limits, rate limits, and abuse/DoS boundaries. | 07 + 08 + 10 |
| GAP-036 | P2 | Dependency upgrades can change binary/serialization behavior. Compatibility fixtures must protect crypto and CRDT formats. | 05 + 07 + 11 |
| GAP-037 | P1 | First-run vault state machine is incomplete: no vault, local vault, server vault but no local key, locked, recovery-required, incompatible version. | 04 + 05 |
| GAP-038 | P1 | Recovery secret format, entropy, checksum, regeneration, print/save UX, and verification are not yet specified. | 05 |
| GAP-039 | P1 | Cookiecutter currently enables DRF TokenAuthentication; the web PWA design intends session authentication only unless a future external client is approved. | 03 + 07 |
| GAP-040 | P2 | Frontend unit/type/lint test stack is not yet selected or wired into pre-merge verification. | 02 + 11 |
| GAP-041 | P2 | Privacy screen behavior on `visibilitychange`, `pagehide`, back-forward cache, and app switching needs definition. | 04 |
| GAP-042 | P2 | Journal/entry text must never appear in `document.title`, URL/query strings, notification previews, or telemetry/error payloads. | 03 + 06 |
| GAP-043 | P2 | Client attachment image handling should consider local decode/re-encode and EXIF stripping to reduce metadata and malicious payload risk. | 08 |
| GAP-044 | P1 | A stale replica needs a deterministic rule for when incremental sync is refused and a trusted full resynchronization is required. | 07 |
| GAP-045 | P2 | Server restore from backup may roll server sequence backward relative to clients. Clients need restore/rollback detection and recovery behavior. | 07 + 10 |
| GAP-046 | P2 | Local-search index lifecycle on lock, migration, corruption, and crypto-key rotation needs explicit behavior. | 09 |
| GAP-047 | P2 | Sync status needs to distinguish "saved locally", "queued", "server acknowledged", and "seen by another replica" without falsely implying backup durability. | 01 + 07 |
| GAP-048 | P2 | PWA/browser feature detection and minimum support matrix are not specified. | 05 + 11 |
| GAP-049 | P2 | Local CA/certificate rotation and WebAuthn relying-party stability need an operational migration plan. | 03 + 10 |
| GAP-050 | P2 | The server needs least-privilege database/storage credentials and a secret-management policy compatible with self-hosted Docker. | 10 |
| GAP-051 | P1 | Tiptap + Automerge rich-text integration is assumed but not proven. The official @automerge/prosemirror binding is currently beta and requires a constrained ProseMirror schema, so the chosen editor/extensions need a prototype convergence test before implementation is locked. | 06 + 07 |
| GAP-052 | P1 | django-allauth rate limits depend on Django cache availability, while generated Cookiecutter production cache uses django-redis IGNORE_EXCEPTIONS=True. A Redis outage could silently weaken authentication throttling unless failure behavior is changed and tested. | 03 |
| GAP-053 | P1 | Disabling mandatory email verification for offline-friendly accounts conflicts with allauth MFA's default refusal to enable MFA for unverified email addresses. Closed-registration provisioning and MFA_ALLOW_UNVERIFIED_EMAIL/verified-address policy must be explicit. | 03 |
| GAP-054 | P1 | A recovery secret that only unwraps a RecoveryEnvelope is not a total-loss recovery mechanism if the server, backups, and every local copy of that envelope are lost. Inkstead needs self-contained portable recovery material/Recovery Kit. | 01A + 05 + 10 |
| GAP-055 | P1 | The cached PWA shell must be account-neutral and contain no user-specific state or CSRF token; otherwise cached shell state can cross sessions or go stale. Session/CSRF bootstrap must be network-only and outside static shell persistence. | 03 + 04 |
| GAP-056 | P1 | Restoring an older server backup can lose newer vault/recovery/key-generation envelopes even when an existing client still has newer ciphertext/keys. Restore reconciliation must define how current key metadata is republished, and what happens when no current replica survives. | 05 + 07 + 10 |
| GAP-057 | P2 | Deferring destructive CRDT compaction makes the v1 server change log append-only and potentially unbounded. Storage growth must be benchmarked and bounded operationally before v1. | 07 + 10 + 11 |
| GAP-058 | P2 | Strict CSP style directives may conflict with editor/ProseMirror/Tiptap runtime styling. The final CSP must test both script/WASM and style behavior without casually enabling unsafe-inline. | 03 + 06 |
| GAP-059 | P2 | npm lifecycle/install scripts add supply-chain execution risk during builds. The reproducible frontend build policy must document when lifecycle scripts are allowed and how dependency changes are reviewed. | 02 + 11 |

---

## Key findings from external best-practice review

### Browser storage is not automatically durable

IndexedDB/Cache data is best-effort by default. Inkstead must request persistent storage when
appropriate, expose whether persistence was granted, monitor approximate quota usage, and handle
`QuotaExceededError` safely.

### Multi-tab coordination is a real correctness requirement

The Web Locks API is suitable for single-origin leader election/critical sections such as sync and
schema migration. BroadcastChannel is suitable for communicating vault/sync state across tabs.

### Service Workers are privileged code

A compromised or stale Service Worker can continue intercepting scoped requests. Inkstead needs:

- restricted scope;
- versioned caches;
- safe update activation;
- an emergency unregister/kill-switch path;
- compatibility checks before a new worker controls an old client.

### E2EE changes file-security assumptions

The server cannot inspect plaintext attachments without breaking the privacy model. The attachment
spec must therefore minimize dangerous formats, perform client-side validation/transformation where
possible, avoid server-side execution, and explicitly record any ASVS file-scanning requirement that
cannot be met because the server intentionally has no plaintext.

### Key management is a lifecycle, not an encryption call

The focused crypto spec must define:

- generation;
- wrapping;
- synchronization;
- rotation;
- compromise;
- recovery;
- retirement;
- versioning;
- migration;
- destruction semantics.

---

## Gap closure rules

A gap may be marked CLOSED only when one of the following exists:

1. an approved focused specification contains a normative resolution; or
2. an ADR accepts the risk/deviation; and
3. the associated implementation gate includes a verification test where technically possible.

Closing a gap in prose without a test is insufficient for P0/P1 items.

---

## Master specification policy

This register does not modify the master specification.

When a focused document provides more detail, it is considered a refinement.

When a focused document cannot be reconciled with the master specification, create an ADR and
decide whether a later version of the master specification should be updated. The current master
file remains untouched during this decomposition exercise.
