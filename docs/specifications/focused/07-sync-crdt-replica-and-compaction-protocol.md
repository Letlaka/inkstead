# Focused Specification 07: CRDT, Sync, Replica and Compaction Protocol

**Status:** DRAFT  
**Parent:** `../master-product-and-implementation-spec.md`  
**Gate:** gate-05-crdt-sync  
**Requires:** gate-04-local-journal PASS  
**Related gaps:** GAP-001, GAP-002, GAP-006, GAP-008, GAP-013, GAP-014, GAP-015, GAP-017, GAP-018, GAP-030, GAP-034, GAP-035, GAP-036, GAP-044, GAP-045, GAP-047

---

## 1. Purpose

This specification defines how independently edited encrypted local journals converge through a
Django server that cannot inspect journal plaintext.

The protocol must prefer data preservation over clever compaction.

---

## 2. Synchronization principles

1. local writes complete before network sync;
2. Django stores opaque encrypted changes;
3. server sequences are transport ordering, not document truth;
4. Automerge causality resolves document convergence;
5. uploads are idempotent;
6. downloads are resumable;
7. server failure never invalidates local writes;
8. destructive server compaction is not allowed until independently proven safe.

---

## 3. CRDT unit

One journal entry SHOULD map to one Automerge document.

Other encrypted domain objects may later use their own Automerge documents if concurrent edits
justify it.

Avoid one vault-wide CRDT.

---

## 4. Replica

A replica is one initialized browser/PWA installation.

Server-visible fields may include:

- replica UUID;
- owner;
- created time;
- last successful sync;
- last accepted client sequence;
- revoked time;
- stale state.

Replica identity is not an authentication token.

Django session authentication remains authoritative.

---

## 5. Local durable outbox

A local edit that is reported as "Saved on this device" MUST create durable sync work.

Conceptual transaction:

```text
generate Automerge change
  -> encrypt change
  -> update encrypted local document snapshot/state
  -> append encrypted ChangeEnvelope to outbox
  -> COMMIT one local transaction
```

If the transaction fails, the UI MUST NOT claim the edit is durably saved.

---

## 6. Local durable inbox

Downloads should not advance the fully applied cursor before local persistence/validation.

Preferred model:

```text
download page
  -> atomically store encrypted inbox envelopes + downloaded cursor
  -> decrypt/validate/apply
  -> persist resulting encrypted local state
  -> advance applied cursor
  -> remove/mark inbox processed
```

A crash between download and apply therefore replays safely.

---

## 7. Client sequence

Each replica maintains a monotonically increasing client sequence for uploads.

The sequence is allocated under the local writer/sync coordination rules.

Server behavior:

- accept expected new sequence;
- accept exact duplicate idempotently if change UUID/hash matches;
- reject conflicting reuse of a sequence;
- reject unexplained gaps unless the protocol explicitly supports a recovery command.

---

## 8. Change identity

Every change envelope has a random opaque change UUID and ciphertext hash.

Server uniqueness should cover at least:

- replica + client sequence;
- change UUID.

A duplicate retry must not create a second logical server change.

---

## 9. Server sequence

The server assigns a monotonically increasing sequence within a vault/account sync stream.

Use database transaction/locking appropriate to PostgreSQL.

The server sequence supports:

- incremental pull;
- acknowledgements;
- restore/rollback detection;
- pagination.

It does not decide CRDT conflict outcome.

---

## 10. Change envelope

Conceptual public metadata:

```text
change_uuid
vault_uuid
document_uuid
replica_uuid
client_sequence
server_sequence
crypto_protocol_version
vdk_generation
ciphertext_hash
ciphertext_length
created_on_server
ciphertext
```

Meaningful document content remains encrypted.

Metadata minimization should be revisited before implementation.

---

## 11. Wrapped key distribution

Under the preferred crypto design, document/blob keys are derived from synchronized VDK
generations.

The sync protocol MUST provide VDK envelopes to authorized replicas before they are expected to
decrypt content using that generation.

If the crypto spec later chooses random per-object keys, sync MUST transport authenticated wrapped
ObjectKeyEnvelopes first.

The server must never deliver ciphertext whose necessary key metadata has no recovery path.

---

## 12. Pull protocol

Suggested behavior:

```text
GET /api/v1/sync/changes/?after=<server-seq>&limit=<n>
```

Response includes:

- server instance/restore identity;
- page of opaque changes;
- next cursor;
- whether more pages remain;
- purge/control records where applicable;
- key-generation envelopes required by those changes.

Limit is server-controlled and bounded.

---

## 13. Push protocol

Suggested:

```text
POST /api/v1/sync/changes/
```

Request contains bounded batch of encrypted change envelopes.

Server validates:

- authenticated owner;
- replica authorization;
- vault ownership;
- request size;
- count;
- UUID formats;
- sequence/idempotency;
- supported public protocol metadata.

It cannot validate plaintext correctness.

---

## 14. CSRF/session recovery

If push fails because session/CSRF expired:

- outbox remains untouched;
- client obtains fresh authentication/CSRF when user restores server login;
- same idempotent outbox batch retries.

No journal re-encryption is required.

---

## 15. Automerge validation

After decrypting incoming change bytes, the client validates them through Automerge.

Malformed or incompatible changes are quarantined.

Do not advance applied cursor past an unrecoverable required change without an explicit recovery
state.

---

## 16. Snapshots

Snapshots are encrypted full-document/bootstrap artifacts.

They MAY be used to reduce new-replica startup cost.

A snapshot contains encrypted internal metadata describing:

- document UUID;
- included known server sequence/cutoff;
- Automerge heads/change state;
- schema version;
- crypto generation.

Outer metadata may duplicate enough information for routing but is not trusted for content coverage.

---

## 17. V1 compaction safety decision

For v1:

> A client-uploaded snapshot MUST NOT authorize deletion of historical ChangeEnvelopes merely because
> it claims to cover them.

Django cannot decrypt the snapshot and cannot prove the claim.

Therefore:

- snapshots are additive bootstrap optimizations;
- historical changes remain retained in v1;
- automatic destructive change compaction is deferred.

This intentionally trades server storage for recoverability.

---

## 18. Future destructive compaction

Any future change-log pruning requires a separate ADR and protocol proving at least:

- snapshot integrity;
- replica acknowledgement semantics;
- active/stale replica definition;
- crash safety;
- rollback safety;
- backup interaction;
- new-replica recovery;
- local unsynced-change preservation.

Do not implement deletion first and invent acknowledgement afterward.

---

## 19. Snapshot selection

A new client may request a latest snapshot for each document and then pull later changes.

Client behavior:

1. decrypt snapshot;
2. verify internal metadata/AAD;
3. load Automerge document;
4. apply retained later changes;
5. compare expected heads/consistency;
6. fall back to older snapshot/change history if verification fails.

Because historical changes remain available in v1, a bad snapshot is recoverable.

---

## 20. Stale replica

A replica becomes stale after a configurable inactivity period or when protocol migrations require
full rebootstrap.

Suggested initial policy for evaluation:

- 90 days without successful sync.

Stale status does not delete local data.

---

## 21. Stale-replica reconnect

Before a stale replica may upload new changes:

1. authenticate;
2. fetch current key-generation/control state;
3. fetch purge markers;
4. full-resync affected documents/snapshots;
5. merge local unsynced CRDT work;
6. only then upload valid resulting changes.

This prevents an old device from blindly replaying state before learning about permanent deletions
or protocol changes.

---

## 22. Permanent-purge marker

To prevent resurrection after a document is permanently deleted, server SHOULD retain an opaque
purge marker:

```text
PurgedObject
  vault_uuid
  object_uuid
  purge_server_sequence
  purged_at
```

No plaintext title/content.

Future uploads for a purged UUID are rejected.

Purge markers may remain until account deletion.

---

## 23. Purged object on stale client

If a stale client has local edits for a server-purged object:

- it MUST NOT silently recreate the same object UUID;
- server rejects upload;
- client explains the object was permanently deleted elsewhere;
- client MAY offer explicit recovery as a new entry with a new UUID if the user deliberately
  chooses it.

No silent resurrection.

---

## 24. Replica revocation

Revoked replica:

- cannot push/pull after server session/auth checks;
- remains unable to erase local data remotely;
- may require full vault rekey if compromise is suspected.

Server rejects change uploads associated with revoked replica IDs.

---

## 25. Server restore/rollback detection

Existing clients remember:

- last server instance identity where applicable;
- highest acknowledged server sequence.

If server returns a lower sequence than previously seen, client MUST treat this as potential
rollback/backup restore.

Do not blindly upload.

Recovery flow should reconcile local changes with restored server state.

---

## 26. Restore generation

Supported restore tooling SHOULD create/rotate a server restore-generation identifier after a
deliberate backup restore.

Clients seeing a new restore generation enter explicit reconciliation mode.

If an administrator restores outside supported tooling, sequence regression still provides a
fallback warning to existing clients.

A completely new client cannot detect history it has never seen; this limitation belongs in the
threat model.

---

## 27. Server withholding threat

AEAD detects modification but cannot force an untrusted server to return data it chooses to hide.

Existing replicas can detect some rollback/withholding relative to known cursors/heads.

New replicas have weaker detection.

V1 accepts this within the documented web-server trust boundary rather than claiming Byzantine
server transparency.

---

## 28. API authorization

Every sync query is scoped to authenticated owner.

Tests include:

- cross-user vault;
- cross-user document;
- cross-user replica;
- guessed UUID;
- revoked replica.

Never trust opaque UUID secrecy as authorization.

---

## 29. API abuse limits

Server MUST bound:

- batch change count;
- per-envelope ciphertext size;
- total request size;
- pagination limit;
- concurrent requests as appropriate;
- blob operations separately.

Rate limiting must not prevent normal reconnect after long offline use; use generous authenticated
sync limits with hard size ceilings rather than login-style tiny limits.

---

## 30. Sync protocol versioning

API/sync protocol version is explicit and independent of crypto version.

Clients send supported version.

Unsupported future/old protocol must fail with a machine-readable upgrade response, not partial
mutation.

---

## 31. Sync status semantics

Client tracks separately:

- local_saved;
- outbox_pending;
- server_acknowledged;
- last_pull_applied;
- offline/session_required.

"Synced" means local outbox acknowledged and all known server changes applied at that moment.

It does not mean every other replica is current.

---

## 32. No WebSocket requirement

V1 uses ordinary HTTPS request/response sync.

WebSockets/Channels are unnecessary for a private journal's correctness.

Foreground polling/reachability triggers are enough.

---

## 33. Gate 05 tests

Minimum:

1. outbox survives reload;
2. duplicate upload is idempotent;
3. conflicting client-sequence reuse rejected;
4. interrupted push retries safely;
5. interrupted pull replays inbox;
6. cursor not advanced before apply;
7. two offline replicas edit different text and converge;
8. same-region concurrent edits converge deterministically;
9. server unavailable never blocks local save;
10. session expiry leaves outbox intact;
11. cross-user UUID access denied;
12. revoked replica rejected;
13. new replica obtains required VDK generation;
14. bad snapshot rejected without data loss;
15. snapshot upload does not delete history;
16. stale replica full-resync flow;
17. purged object cannot resurrect;
18. server-sequence regression triggers restore mode;
19. unsupported protocol fails closed;
20. large/abusive batches rejected.

---

## 34. Gate 05 evidence

```text
docs/evidence/gate-05-crdt-sync.md
```

Include:

- API schema version;
- database model summary;
- outbox/inbox crash tests;
- convergence fixtures;
- IDOR tests;
- snapshot tests;
- stale-replica tests;
- restore/rollback test.

---

## 35. Blocking decisions before approval

- exact global/per-vault server sequence implementation;
- stale-replica threshold;
- snapshot creation frequency;
- exact new-replica bootstrap endpoint shape;
- purge-marker retention;
- whether journal metadata uses same change stream or separate objects;
- maximum batch sizes;
- server restore-generation mechanism.

---

## 36. Exit criteria

Gate 05 passes only when:

- two clean replicas converge after offline edits;
- retries cannot duplicate logical changes;
- crashes cannot lose acknowledged local writes;
- snapshots cannot cause destructive data loss;
- stale replicas cannot resurrect purged objects;
- rollback/restore is detected for existing clients;
- authorization tests pass;
- no unresolved P1 sync/CRDT gap remains.
