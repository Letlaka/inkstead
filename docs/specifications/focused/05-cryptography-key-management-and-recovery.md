# Focused Specification 05: Cryptography, Key Management and Recovery

**Status:** DRAFT  
**Parent:** `../master-product-and-implementation-spec.md`  
**Gate:** gate-03-crypto-vault  
**Requires:** gate-02-pwa-offline PASS  
**Related gaps:** GAP-001, GAP-009, GAP-010, GAP-011, GAP-012, GAP-013, GAP-036, GAP-037, GAP-038, GAP-054, GAP-056

---

## 1. Purpose

This specification defines the client-side cryptographic system used by Inkstead.

No cryptographic implementation may begin while this document remains DRAFT.

The protocol must be independently documentable and testable.

---

## 2. Security goals

The design aims to ensure that compromise of server-side persistence alone does not reveal:

- journal names;
- entry titles;
- entry bodies;
- tags;
- mood;
- prompts/answers;
- meaningful attachment metadata;
- local search index contents;
- journal decryption keys.

The design also aims to support:

- offline unlock;
- new-device recovery;
- password/recovery credential changes;
- future algorithm upgrades;
- key compromise response;
- deterministic interoperability tests.

---

## 3. Non-goals / unavoidable boundaries

Client-side encryption cannot fully protect an unlocked vault from:

- malicious same-origin application JavaScript;
- browser extensions with page privileges;
- OS malware;
- keylogging;
- screen capture;
- a compromised browser runtime.

JavaScript also cannot guarantee byte-perfect key zeroization because garbage collection/runtime
copies are outside application control.

Inkstead therefore uses best-effort key lifetime minimization and worker termination, not claims of
guaranteed zeroization.

---

## 4. Separate account and vault credentials

Django account credentials MUST remain independent from vault cryptographic credentials.

Account password changes do not rotate journal encryption automatically.

Account recovery does not decrypt the journal.

Vault recovery material is client-side cryptographic material.

---

## 5. Recommended v1 key hierarchy

The master specification's broad hierarchy is refined here to remove unnecessary synchronized key
objects and to make total-loss recovery explicit.

Preferred v1 candidate:

```text
Vault Passphrase
      |
      | Argon2id
      v
Passphrase KEK
      |
      | unwraps
      v
Vault Master Key (VMK, random 256-bit)
      |
      +-- HKDF("generation" || N) -> Generation Key N
              |
              +-- HKDF("document" || document_uuid) -> document key
              |
              +-- HKDF("blob" || blob_uuid) -> blob key
              |
              +-- HKDF("local-index" || device_context) -> local/index key

Emergency Recovery Kit
      |
      +-- independently provides/reconstructs the same VMK without depending on Django
```

Advantages:

- a new replica needs one root key, not thousands of synchronized per-object keys;
- document/blob keys remain domain separated;
- a leaked derived object key does not reveal VMK;
- a leaked Generation Key does not reveal VMK or other generations;
- new generations can be introduced without server-side key envelopes;
- server loss does not destroy the only copy of information required to recover VMK;
- changing the normal vault passphrase only rewraps VMK.

This is a refinement of the master specification's recommended hierarchy. Approval of this focused
spec should be accompanied by an ADR recording the decision while leaving the original master
document historically intact.

---

## 6. Key classes

### 6.1 Vault Passphrase KEK

Derived from a user-chosen vault passphrase using Argon2id.

Purpose:

- unwrap VMK from an encrypted passphrase envelope;
- support normal/new-browser unlock without exposing VMK to Django.

The passphrase is distinct from the Django account password.

### 6.2 Vault Master Key (VMK)

Random 256-bit root key generated client-side.

Purpose:

- root of journal data-key derivation;
- stable vault cryptographic identity independent of Django account password.

VMK MUST never be stored plaintext by Django.

### 6.3 Generation Key N

Derived from VMK using HKDF with a canonical generation context.

Purpose:

- support versioned key generations;
- allow a compromised generation key to be retired without revealing VMK;
- provide a stable input for per-object key derivation.

Generation keys are not synchronized as independent random secrets.

### 6.4 Derived document key

Derived from the selected Generation Key and document UUID with a document-specific domain label.

It is not synchronized as a separate secret.

### 6.5 Derived blob key

Derived from the selected Generation Key and blob UUID with a distinct blob domain label.

Document and blob keys MUST NOT collide even for identical UUID bytes.

### 6.6 Local/index key

A device-purpose-separated key MAY be derived for encrypted local search/index state.

Its derivation must not let exposure of the local index key reveal VMK or content keys.

---

## 7. Why derived per-object keys are preferred for v1

PR #7 correctly identified that independently random object keys require synchronized wrapped-key
envelopes.

Inkstead does not need per-object sharing in v1. A hierarchical HKDF design therefore removes an
entire key-distribution protocol while preserving per-object key separation.

If future collaboration/sharing requires independently random item keys, introduce a separately
versioned ObjectKeyEnvelope protocol through an ADR.

There is no valid design where an independently random object key exists only on the creating
replica.

---

## 8. Emergency Recovery Kit

A recovery mechanism is not sufficient if it depends on the server retaining the only encrypted VMK
envelope.

Inkstead MUST provide portable, user-controlled recovery material that can recover VMK after:

- total server loss;
- loss/corruption of operational backups;
- loss of all browser-local Inkstead storage.

Two acceptable design families remain for review:

### Option A - bearer Recovery Kit

A versioned recovery artifact directly contains the root secret needed to reconstruct VMK.

Properties:

- simplest total-loss recovery;
- possession of the kit is equivalent to possession of a master recovery key;
- kit MUST be treated as highly sensitive;
- format needs checksum/version/vault identity and safe printable/file representation.

### Option B - Recovery Key + portable Recovery Bundle

A high-entropy Recovery Key protects a portable bundle containing the wrapped VMK and required
public parameters.

Properties:

- bundle can be backed up separately from the secret key;
- both components must be user-controlled and exportable;
- neither may exist only on the Inkstead server.

Gate 03 must choose one model.

Inkstead MUST NOT label a server-dependent passphrase envelope as the emergency recovery mechanism.

---

## 9. Recovery material requirements

Total-loss recovery material SHOULD provide at least 128 bits of effective entropy; 192 or 256 bits
is preferred.

Human/file representation MUST:

- use a standardized or carefully reviewed encoding;
- include format version;
- include error detection/checksum where practical;
- avoid ambiguous manual transcription;
- be exportable to a password manager/file;
- support print/QR representation where practical;
- be verified during initial setup.

Do not invent an unaudited mnemonic algorithm.

---

## 10. Normal vault-passphrase envelope

For pleasant new-browser unlock, Django MAY store a ciphertext envelope:

```text
VaultPassphraseEnvelope
  protocol_version
  kdf_id
  kdf_parameters
  salt
  nonce
  wrapped_vmk
  aead_metadata
```

The browser:

1. receives the envelope;
2. user enters Vault Passphrase;
3. derives Passphrase KEK locally;
4. unwraps VMK locally.

Django never receives the passphrase or KEK.

A stolen envelope permits offline passphrase guessing, so Argon2id calibration and strong
passphrase guidance are mandatory.

---

## 11. Daily/local unlock

After VMK is recovered on a browser, Inkstead may store another VMK envelope locally protected by:

- the same Vault Passphrase; or
- a device-specific local passphrase; or
- future WebAuthn PRF/device-backed wrapping.

V1 must select the least confusing safe UX.

A short numeric PIN MUST NOT be the sole offline brute-force protection unless a hardware-backed
primitive enforces attempts.

If a local unlock credential is forgotten, the Emergency Recovery Kit can re-establish local access
and allow a new local/passphrase envelope to be created.

---

## 12. Encryption algorithm

Preferred v1 AEAD:

- XChaCha20-Poly1305 through libsodium.

Requirements:

- unique 192-bit nonce per encryption operation/key;
- cryptographically secure random nonce generation;
- authenticated associated data;
- no nonce reuse;
- ciphertext version metadata.

Exact library/build is approved only after CSP/WASM/browser compatibility tests.

---

## 13. KDF

Preferred:

- Argon2id.

KDF parameters MUST be:

- calibrated against target desktop/mobile browsers;
- recorded in the envelope;
- versionable;
- high enough to resist guessing without making normal unlock unpleasant.

The calibration target should include lower-powered mobile hardware.

A future KDF upgrade must not invalidate old envelopes.

---

## 14. HKDF domain separation

Derived object keys MUST use unambiguous labels.

Conceptual examples:

```text
Inkstead/v1/document/<uuid>
Inkstead/v1/blob/<uuid>
Inkstead/v1/search/<device-id>
```

The exact binary serialization of HKDF `info` MUST be specified in the crypto protocol, not built
by ambiguous string concatenation.

---

## 15. AEAD associated data

Associated data SHOULD bind:

- protocol version;
- object type;
- vault/journal UUID where applicable;
- document/blob UUID;
- VDK generation/key ID;
- change/envelope type.

The exact encoding MUST be canonical and included in compatibility fixtures.

---

## 16. Crypto protocol versions

At minimum version independently:

- crypto envelope format;
- algorithm suite;
- KDF parameters/version;
- VDK generation.

Do not confuse crypto-protocol version with application release version.

A newer application must be able to recognize older supported crypto versions.

An older application encountering an unsupported newer crypto version MUST fail closed and MUST NOT
overwrite the data.

---

## 17. Crypto Worker

All normal vault cryptographic operations SHOULD occur in a dedicated Web Worker.

The main thread receives:

- plaintext needed for visible UI;
- ciphertext;
- operation results.

The main thread MUST NOT receive/export VMK/VDK values unless an explicit audited protocol step
requires it.

Worker messages must not be logged in production.

---

## 18. Lock behavior

On vault lock:

1. UI replaces/removes plaintext;
2. in-memory search index is destroyed;
3. Crypto Worker terminates;
4. references to plaintext buffers are released;
5. local persistent data remains ciphertext.

This is best-effort memory hygiene.

Do not claim secure memory wiping.

---

## 19. New-replica bootstrap

Normal path:

```text
authenticate to server
  -> receive public crypto parameters + VaultPassphraseEnvelope
  -> user enters Vault Passphrase
  -> derive Passphrase KEK locally
  -> unwrap VMK
  -> derive current Generation Key(s)
  -> validate encrypted vault verifier
  -> initialize local VMK envelope
  -> synchronize ciphertext
  -> decrypt locally
```

Disaster/recovery path:

```text
server envelope missing/unusable
  -> user supplies Emergency Recovery Kit
  -> recover VMK locally
  -> validate vault verifier or recovered local/export data
  -> create fresh VaultPassphraseEnvelope
  -> republish safe key metadata when server is available
```

A Django login alone never grants decryption.

The protocol must support republishing the current passphrase/key metadata after a server restore
from an older backup.

---

## 20. Vault verifier

A recovery attempt should fail quickly and deterministically without attempting to decrypt arbitrary
journal content.

Store an authenticated encrypted verifier object under the vault key hierarchy.

It must reveal no secret when stored server-side.

---

## 21. Credential change vs compromise rekey

These are distinct operations.

### 21.1 Vault passphrase change with no suspected compromise

- keep VMK;
- derive a new Passphrase KEK;
- rewrap VMK;
- replace the server/local VaultPassphraseEnvelope.

This is fast and does not change the Emergency Recovery Kit unless policy explicitly rotates root
recovery material.

### 21.2 Suspected key compromise

Rewrapping alone is insufficient if an attacker may already possess the compromised secret.

Threat-specific response:

- leaked derived object key: replace/re-encrypt the affected object if necessary;
- leaked Generation Key: move to a new generation derived from VMK and migrate affected content;
- leaked VMK or Emergency Recovery Kit: generate a new VMK and perform full-vault re-encryption;
- revoke compromised server session/replica as an additional access-control step.

V1 must define and test at least a manual full-vault rekey procedure before claiming lost-device or
Recovery-Kit compromise remediation.

---

## 22. What revocation cannot do

If a lost device already decrypted an entry, no protocol can make that copy unknowable.

If a lost device exported VMK/VDK while compromised, revoking its server session alone does not
repair cryptographic compromise.

User-facing security text must state this accurately.

---

## 23. Key rotation

The implementation MUST support key generation identifiers from day one even if routine periodic
rotation is not automated initially.

Triggers may include:

- suspected compromise;
- algorithm upgrade;
- protocol upgrade;
- deliberate user action.

Old keys MUST remain available only as long as needed for unmigrated ciphertext/backup recovery.

---

## 24. Backup and restore interaction

Operational backups may contain old VaultPassphraseEnvelopes and old ciphertext.

After restoring an older server backup:

- an existing current replica may have a newer passphrase envelope/current crypto metadata and must
  be able to republish it safely;
- the Emergency Recovery Kit must remain usable even if every server-held passphrase envelope is
  missing;
- if neither a current replica nor valid Emergency Recovery Kit exists, an old backup may require
  the older Vault Passphrase matching its envelope.

Key rotation cannot retroactively remove ciphertext/key metadata from historical backups.

Backup retention and key-retirement policy must account for this.

---

## 25. Corruption behavior

Every ciphertext read from:

- IndexedDB;
- server;
- backup/import

is untrusted.

AEAD failure MUST:

- never return partial plaintext;
- quarantine/report the affected object;
- preserve original ciphertext for recovery diagnosis where safe;
- avoid overwriting good replicas automatically.

The UI should say an item could not be verified, not invent content.

---

## 26. Metadata leakage profile

The crypto threat model must explicitly list server-visible information.

Likely visible:

- account identity/username;
- opaque vault/journal IDs;
- object/change counts;
- ciphertext lengths;
- sync timestamps;
- server sequence numbers;
- attachment ciphertext size;
- client IP/user-agent through authentication/session logs unless minimized.

Potential mitigations such as padding MAY be considered later.

V1 must document leakage even if it does not hide all of it.

---

## 27. Crypto compatibility fixtures

Before Gate 03 passes, commit deterministic test vectors containing:

- salts;
- KDF parameters;
- VMK/VDK test keys;
- recovery envelope;
- VDK envelope;
- document ciphertext;
- blob ciphertext;
- expected plaintext;
- corrupted/tampered examples.

Production code must not use fixture keys.

Fixtures must allow independent implementation verification.

---

## 28. Plaintext persistence canary

Gate 03 must create a unique plaintext marker and verify it does not appear in:

- raw IndexedDB records;
- CacheStorage;
- server requests;
- server database after sync is later added.

At Gate 03 only the local portions are required; later gates extend the same canary.

---

## 29. Crypto tests

Minimum:

1. correct Vault Passphrase unwraps VMK envelope;
2. wrong Vault Passphrase fails;
3. Emergency Recovery Kit recovers VMK with the server unavailable;
4. total-loss recovery works without server-held key envelopes;
5. Generation Key derivation is deterministic;
6. document key derivation is deterministic;
7. document/blob domain keys differ;
8. object UUID change causes different derived key;
9. nonce reuse test guard;
10. modified ciphertext fails;
11. modified AAD fails;
12. old protocol fixture decrypts;
13. unsupported future version fails closed;
14. local lock terminates worker;
15. KDF parameters round-trip;
16. Vault Passphrase rewrap does not rewrite content;
17. old-server-backup envelope can be repaired from current replica/Recovery Kit;
18. compromise-rekey fixture migrates successfully.

---

## 30. Gate 03 evidence

```text
docs/evidence/gate-03-crypto-vault.md
```

Must record:

- approved algorithm suite;
- approved KDF parameters;
- browser performance measurements;
- CSP/WASM compatibility;
- crypto fixture hashes;
- plaintext canary result;
- normal-passphrase and Emergency-Recovery-Kit flows;
- total-loss recovery test;
- restored-server key-metadata reconciliation;
- key-rotation test.

---

## 31. Blocking decisions before approval

- derived per-object key design vs random wrapped object keys;
- Emergency Recovery Kit model/format;
- recovery material entropy/encoding;
- daily local unlock mechanism;
- exact Argon2id parameters/calibration process;
- exact libsodium build;
- VMK/VDK rotation semantics;
- minimum v1 compromise-rekey capability;
- metadata leakage accepted for v1.

---

## 32. Exit criteria

Gate 03 cannot pass until:

- a second clean browser can recover all required keys;
- total-loss recovery succeeds without relying on server-held key envelopes;
- no unsynchronized random object key exists;
- local persisted journal data is ciphertext;
- recovery works with server holding no recovery secret;
- tampering is detected;
- old/future version behavior is deterministic;
- key rotation/rewrap tests exist;
- no unresolved P1 cryptographic gap remains.
