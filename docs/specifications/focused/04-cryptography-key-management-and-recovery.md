# Focused Specification 04: Cryptography, Key Management and Recovery

**Status:** DRAFT  
**Parent:** `../master-product-and-implementation-spec.md`  
**Gate:** gate-03-crypto-vault  
**Requires:** gate-02-pwa-offline PASS  
**Related gaps:** GAP-001, GAP-009, GAP-010, GAP-011, GAP-012, GAP-013, GAP-036, GAP-037, GAP-038

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

The master spec suggested independent random per-document keys. The focused review identifies a key
distribution cost: independently random keys must themselves be synchronized as wrapped encrypted
objects.

For v1, the preferred refinement is a smaller, versioned key hierarchy with deterministic
domain-separated object keys.

```text
Recovery Secret
      |
      | Argon2id
      v
Recovery KEK
      |
      | unwraps
      v
Vault Master Key (VMK, random 256-bit)
      |
      | unwraps
      v
Vault Data Key Generation N (VDK-N, random 256-bit)
      |
      +-- HKDF("document" || document_uuid) -> document key
      |
      +-- HKDF("blob" || blob_uuid) -> blob key
      |
      +-- HKDF("local-index" || device_context) -> index key where appropriate
```

This has several advantages:

- new replicas synchronize only a small set of wrapped VDK envelopes;
- no per-entry key envelope can be missing;
- every object still has a distinct domain-separated encryption key;
- key generations allow future rotation/migration;
- credential changes rewrap VMK/VDKs rather than rewriting all content.

Before approval, this design must receive explicit cryptographic review.

---

## 6. Key classes

### 6.1 Recovery Key-Encryption Key (Recovery KEK)

Derived from high-entropy recovery material using Argon2id.

Purpose:

- unwrap VMK for disaster/new-device recovery.

MUST NOT be transmitted to server.

### 6.2 Vault Master Key (VMK)

Random 256-bit key generated client-side.

Purpose:

- wrap/unlock VDK generations;
- provide stable vault root independent of account password.

VMK itself MUST be stored only in encrypted/wrapped form when persisted.

### 6.3 Vault Data Key generation (VDK-N)

Random 256-bit key.

Purpose:

- root for domain-separated document/blob keys for one key generation.

Multiple old generations MAY exist during migration.

### 6.4 Derived document key

Derived using HKDF from VDK and a domain-separated context containing document UUID.

It is never synchronized as a separate random secret.

### 6.5 Derived blob key

Derived using HKDF from VDK and blob UUID with a distinct domain label.

Document and blob keys MUST NOT collide even for the same UUID bytes.

### 6.6 Device-local unlock KEK

A device-local passphrase or future WebAuthn PRF may derive/provide a KEK that wraps a local VMK
copy.

It MUST remain separate from the recovery KEK.

---

## 7. Why derived per-object keys are preferred for v1

The PR #7 review correctly identified that random object keys need synchronized wrapped-key
envelopes.

Rather than add an envelope for every object, this spec proposes deterministic per-object key
derivation from a synchronized wrapped VDK generation.

If later review prefers independently random per-object keys, then a normative `ObjectKeyEnvelope`
must be added containing:

- object UUID;
- key generation/version;
- wrapped object key;
- wrapping-key ID;
- algorithm/protocol version;
- authenticated context.

There is no valid design where an independently random object key exists only on the creating
replica.

---

## 8. Recovery secret requirements

Recovery material SHOULD provide at least 128 bits of brute-force-resistant entropy; 192 or 256 bits
is preferred where usable.

The human representation MUST:

- use a standardized/audited encoding or carefully reviewed library;
- include error-detection/checksum capability where practical;
- avoid ambiguous characters;
- be printable;
- be copyable to a password manager;
- be confirmable during setup.

Do not invent a home-grown mnemonic scheme casually.

Blocking review decision:

- exact entropy;
- exact encoding;
- exact dependency/library.

---

## 9. Local daily unlock

V1 should avoid requiring the long recovery secret every day.

Proposed flow:

1. new device recovers VMK using recovery secret;
2. user creates a local vault-unlock passphrase;
3. Argon2id derives a device-local KEK;
4. device-local KEK wraps VMK for that browser profile;
5. daily unlock uses the local passphrase.

Requirements:

- a short numeric PIN alone is NOT sufficient as the sole offline brute-force barrier unless
  protected by a hardware/platform primitive;
- KDF parameters are stored with the local envelope;
- failed local unlock attempts are delayed in UI, but designers must assume an attacker with a
  copied browser profile can perform offline guessing;
- therefore user guidance must favor a meaningful passphrase.

Future progressive enhancement:

- WebAuthn PRF/device-backed wrapping key.

---

## 10. Recovery envelope

The server MAY store:

```text
RecoveryEnvelope
  protocol_version
  kdf_id
  kdf_parameters
  salt
  nonce
  wrapped_vmk
  aead_metadata
```

The server MUST NOT store the recovery secret/KEK.

The envelope must be authenticated.

---

## 11. VDK envelope

The server synchronizes encrypted VDK generations.

Example:

```text
VaultDataKeyEnvelope
  key_generation
  key_id
  created_at
  status
  nonce
  wrapped_vdk
  protocol_version
```

`wrapped_vdk` is encrypted under VMK.

A new replica that recovers VMK can therefore unwrap all required VDK generations and decrypt
existing content.

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

Required flow:

```text
authenticate to server
  -> receive public crypto parameters + RecoveryEnvelope + VDK envelopes
  -> user provides recovery secret
  -> derive Recovery KEK
  -> unwrap VMK
  -> unwrap VDK generations
  -> validate a known encrypted verifier
  -> initialize device-local unlock envelope
  -> synchronize ciphertext
  -> decrypt locally
```

No server plaintext/key exposure is required.

---

## 20. Vault verifier

A recovery attempt should fail quickly and deterministically without attempting to decrypt arbitrary
journal content.

Store an authenticated encrypted verifier object under the vault key hierarchy.

It must reveal no secret when stored server-side.

---

## 21. Credential change vs compromise rekey

These are distinct operations.

### 21.1 Recovery/passphrase change with no suspected compromise

- keep VMK;
- derive a new KEK;
- rewrap VMK;
- old recovery envelope is invalidated.

This is fast.

### 21.2 Suspected key compromise

Rewrapping alone is insufficient because an attacker may already possess VMK/VDK.

Compromise rekey requires:

- new VMK and/or new VDK generation according to threat;
- new content encrypted under uncompromised keys;
- migration of existing documents/blobs if retrospective protection is desired;
- retirement of old key generations after migration;
- revocation of compromised replica/session.

V1 must at least define and test a manual full-vault rekey procedure before claiming lost-device
compromise remediation.

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

## 24. Backup interaction

Operational server backups may contain old wrapped key envelopes and old ciphertext.

Key rotation cannot retroactively remove material from historical backups.

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

1. correct recovery secret unwraps VMK;
2. wrong recovery secret fails;
3. VDK envelope unwrap succeeds;
4. document key derivation is deterministic;
5. document/blob domain keys differ;
6. object UUID change causes different derived key;
7. nonce reuse test guard;
8. modified ciphertext fails;
9. modified AAD fails;
10. old protocol fixture decrypts;
11. unsupported future version fails closed;
12. local lock terminates worker;
13. KDF parameters round-trip;
14. recovery rewrap does not rewrite content;
15. compromise-rekey fixture migrates successfully.

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
- recovery/new-replica flow;
- key-rotation test.

---

## 31. Blocking decisions before approval

- derived per-object key design vs random wrapped object keys;
- recovery secret entropy/encoding;
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
- no unsynchronized random object key exists;
- local persisted journal data is ciphertext;
- recovery works with server holding no recovery secret;
- tampering is detected;
- old/future version behavior is deterministic;
- key rotation/rewrap tests exist;
- no unresolved P1 cryptographic gap remains.
