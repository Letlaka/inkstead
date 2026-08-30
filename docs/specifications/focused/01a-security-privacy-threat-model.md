# Focused Specification 01A: Security, Privacy Threat Model and Data Classification

**Status:** DRAFT  
**Parent:** `../master-product-and-implementation-spec.md`  
**Implementation prerequisite:** MUST be APPROVED before Gate 00 can pass  
**Related gaps:** GAP-010, GAP-011, GAP-012, GAP-014, GAP-016, GAP-021, GAP-022, GAP-023, GAP-025, GAP-029, GAP-042

---

## 1. Purpose

This specification defines Inkstead's security assets, trust boundaries, attacker models, privacy
classes, explicit non-goals, and claims.

Every later focused specification MUST use this document's terminology.

The purpose is not to make Inkstead "secure against everything." The purpose is to say exactly what
Inkstead protects, from whom, under which conditions, and where the unavoidable boundaries are.

---

## 2. Security goals

Inkstead prioritizes:

1. confidentiality of journal plaintext;
2. integrity of journal data and synchronization;
3. availability of local writing even when the server is unavailable;
4. recoverability from server failure;
5. resistance to credential theft and account takeover;
6. minimization of sensitive metadata exposed to the server;
7. safe failure instead of silent data corruption/loss;
8. security controls that do not routinely interrupt journaling.

---

## 3. Protected assets

### Class S0 - Root cryptographic secrets

Highest sensitivity:

- Vault Master Key;
- Vault Data Keys/key generations;
- recovery secret/recovery KEK;
- device-local vault-unlock keys;
- server-side MFA field-encryption keys;
- TLS/private-CA private keys;
- backup repository encryption credentials.

These MUST never be logged.

Journal root keys MUST never be persistently available to Django.

### Class S1 - Authentication secrets

- account password;
- TOTP seeds;
- TOTP codes;
- recovery codes;
- WebAuthn private credential material;
- authenticated session cookies;
- CSRF secrets/tokens where sensitive.

### Class S2 - Journal plaintext

- journal names/descriptions;
- entry titles;
- entry bodies;
- tags;
- mood;
- prompts/answers;
- attachment captions;
- original attachment filenames;
- decrypted attachments;
- local search queries;
- local search index plaintext.

### Class S3 - Sensitive operational metadata

Potentially revealing even without plaintext:

- opaque vault/document/blob IDs;
- ciphertext sizes;
- entry/change counts;
- upload/download timing;
- sync timestamps;
- replica activity;
- user-agent/device information;
- client IP;
- account username/email if configured;
- storage quota/backup metadata.

### Class S4 - Public/non-sensitive application data

- static application bundles;
- generic product text;
- public documentation;
- non-user-specific icons/fonts.

---

## 4. Primary trust boundaries

```text
USER / DEVICE
+----------------------------------------------------------+
| Browser runtime                                           |
|  +-- React/editor                     [plaintext visible] |
|  +-- Crypto Worker                    [vault keys]        |
|  +-- IndexedDB                        [ciphertext]        |
|  +-- Service Worker                   [app shell only]    |
|  +-- browser extensions / OS          [outside control]   |
+---------------------------+------------------------------+
                            |
                            | HTTPS
                            v
SERVER
+----------------------------------------------------------+
| Traefik                                                   |
| Django / allauth / DRF                                    |
| PostgreSQL                     [opaque encrypted journal] |
| Redis                          [no journal plaintext]      |
| blob storage / Nginx           [ciphertext]               |
| operational backup             [ciphertext + metadata]    |
+----------------------------------------------------------+
```

The application-delivery server is part of the trusted computing base because it serves executable
JavaScript.

---

## 5. Attacker models

### A1 - Opportunistic network attacker

Can observe/tamper with unencrypted LAN traffic.

Mitigation target:

- HTTPS;
- secure cookies;
- HSTS where appropriate.

### A2 - Remote unauthenticated attacker

Relevant if the self-host exposes Inkstead through Internet/VPN ingress.

Can:

- probe login;
- guess credentials;
- submit malformed requests;
- attempt XSS/CSRF/DoS.

### A3 - Malicious LAN client

Can reach the Inkstead hostname but has no account.

Must not gain access merely because it shares a private network.

### A4 - Stolen server database/storage/backup

Attacker obtains PostgreSQL, blob storage or operational backup.

Goal:

- journal plaintext remains unavailable without client-side vault keys.

### A5 - Honest-but-curious server administrator

Has legitimate server/DB access but should not ordinarily be able to read journal plaintext.

This is a primary design goal.

### A6 - Actively compromised Inkstead server

Attacker controls Django/static application delivery.

Limitation:

- because Inkstead is a web application, attacker may serve malicious JavaScript on the next load
  and target an unlocked vault.

Inkstead does NOT claim cryptographic protection against this active web-delivery attack.

Existing offline/cached clients may retain some protection until hostile code is loaded, but this is
not a security guarantee.

### A7 - Stolen browser profile/device at rest

Attacker obtains browser files/site data while vault is locked.

Goal:

- journal persistent data remains encrypted;
- offline guessing is constrained by strong local unlock KDF/material.

### A8 - Physical access to an unlocked browser

Attacker can see what the legitimate user can see.

Auto-lock/privacy cover reduce exposure but cannot retroactively hide observed plaintext.

### A9 - Malicious browser extension / compromised browser / OS malware

May observe DOM, keystrokes, memory, clipboard or decrypted content.

This is outside Inkstead's protection boundary.

### A10 - Supply-chain attacker

Compromises an application/dependency/container build artifact.

Mitigation:

- lockfiles;
- minimal dependencies;
- scanning;
- reproducible/pinned builds;
- SBOM;
- review;
- CSP limits where possible.

CSP cannot make a deliberately malicious bundled dependency safe.

### A11 - Malicious imported content/attachment

Attempts:

- XSS;
- path traversal;
- decompression bombs;
- dangerous file execution;
- metadata leakage.

### A12 - Buggy/stale legitimate replica

Not malicious but may:

- send old changes;
- miss purges;
- upload duplicate data;
- use an old protocol;
- submit a bad snapshot.

Protocol MUST be safe against this class without assuming every client is current.

---

## 6. Security properties by state

### Vault locked

Expected:

- no journal plaintext rendered;
- no active in-memory search index;
- Crypto Worker terminated;
- persistent journal state ciphertext only.

### Vault unlocked

Expected:

- plaintext exists in browser memory/DOM as necessary;
- keys exist in Crypto Worker;
- persistent local writes remain encrypted;
- server still receives ciphertext.

### Server unavailable

Expected:

- local journal remains usable;
- no weakening of local encryption;
- sync simply queues.

### Server session expired

Expected:

- local vault remains independently usable if unlocked;
- sync pauses;
- reauthentication required only to resume server operations.

---

## 7. Explicit metadata leakage

V1 does not attempt full traffic-analysis resistance.

The server may learn:

- account identity;
- that a vault exists;
- opaque object IDs;
- number/size of encrypted envelopes;
- approximate edit/upload timing;
- attachment ciphertext size;
- replica last-sync time;
- network/IP/session metadata.

Inkstead MUST document this rather than calling all metadata "zero knowledge."

Future padding/batching may reduce some leakage but is not a v1 requirement.

---

## 8. Explicit non-goals

Inkstead does not promise protection from:

- a malicious browser extension with page permissions;
- OS-level malware/keylogging;
- an actively malicious web server serving modified client code;
- screenshots/photos of an unlocked journal;
- a user intentionally exporting plaintext;
- plaintext copied to another application/clipboard by the user;
- local data already copied from a device before revocation;
- historical plaintext exports stored outside Inkstead;
- immediate erasure from all historical backups;
- traffic-analysis-resistant anonymity.

---

## 9. Availability model

Local writing availability is more important than server availability.

The server may fail closed for authentication/security operations while the local journal continues
to work.

Examples:

- Redis unavailable: server login/rate-limited operations may fail safely; local writing continues;
- PostgreSQL unavailable: sync unavailable; local writing continues;
- TLS certificate expired: sync unavailable; local writing continues on already installed/cached
  client where browser rules permit; the UI must not encourage bypassing certificate validation.

---

## 10. Integrity model

Integrity protections include:

- AEAD for encrypted journal objects;
- deterministic CRDT validation;
- client sequence/idempotency;
- server sequence/cursor;
- purge markers;
- versioned protocols;
- migration compatibility fixtures;
- backup restore verification.

The server cannot inspect plaintext correctness.

---

## 11. Recovery model

Inkstead MUST distinguish:

- account recovery;
- vault cryptographic recovery;
- local-device recovery;
- server backup recovery;
- disaster recovery after total server loss.

A "recovery secret" is not considered sufficient merely because it unlocks a server-held envelope.

Before crypto approval, Inkstead must define a recovery package that remains usable when:

- server is lost;
- server backup is lost/corrupt;
- all local browser state is lost;
- user still possesses the intended recovery material.

Either the recovery material itself or a clearly named self-contained Recovery Kit must include all
non-secret/wrapped information required to reconstruct vault key access.

---

## 12. Revocation model

Revocation can prevent future server access.

It cannot guarantee remote deletion of:

- local ciphertext;
- previously decrypted plaintext;
- keys extracted before revocation.

Compromise response may therefore require cryptographic rekey, not just session revocation.

---

## 13. Privacy UX rule

Security language must describe actual state.

Allowed examples:

- "Saved on this device"
- "Synced to server"
- "Journal locked"
- "This browser's local storage is not persistent"

Avoid:

- "Completely anonymous"
- "Impossible to decrypt"
- "Remote wipe successful"
- "Permanently erased everywhere"

unless a future design can prove those claims.

---

## 14. Security-control hierarchy

Where possible:

1. prevent unsafe state;
2. fail closed;
3. recover automatically without data loss;
4. inform user only when an action is required;
5. never train users to click through meaningless warnings.

---

## 15. Security regression philosophy

Every security property that can be mechanically checked should eventually become a test.

Examples:

- plaintext canary;
- IDOR;
- CSP;
- Service Worker scope;
- encrypted IndexedDB;
- corrupted envelope rejection;
- stale replica purge;
- backup restore.

Human documentation is necessary for boundaries that cannot be mechanically enforced.

---

## 16. Threat-model review triggers

Review this document when adding:

- public sharing;
- collaboration;
- plugins;
- native apps;
- browser extensions;
- remote AI;
- external integrations;
- federation;
- public API clients;
- new plaintext server processing.

---

## 17. Blocking decisions before approval

- exact total-loss recovery material/Recovery Kit;
- accepted v1 metadata leakage;
- whether multi-user self-hosting remains in v1;
- exact external/Internet deployment support statement;
- browser-extension/OS boundary wording;
- backup erasure/retention security claim;
- availability behavior when Redis/allauth rate-limit cache is unavailable.

---

## 18. Approval criteria

This threat model can be marked APPROVED when:

- all assets are classified;
- all trust boundaries have an owner;
- total-loss recovery expectation is explicit;
- server metadata leakage is documented;
- revocation limitations are explicit;
- actively compromised web-delivery limitation is explicit;
- later focused specs use consistent definitions;
- no public product claim exceeds this threat model.
