# Focused Specification 09: Search, Export, Import, Deletion and Data Lifecycle

**Status:** DRAFT  
**Parent:** `../master-product-and-implementation-spec.md`  
**Gate:** gate-07-data-lifecycle  
**Requires:** gate-06-attachments PASS  
**Related gaps:** GAP-012, GAP-015, GAP-029, GAP-033, GAP-044, GAP-046

---

## 1. Purpose

This specification covers secondary data derived from the journal and the complete lifecycle from
creation through search, export, deletion and eventual purge.

The main rule is simple:

> Derived convenience data must never become a privacy bypass or a second unencrypted source of
> truth.

---

## 2. Search architecture

Search is client-side.

Django does not index plaintext journal content.

Initial engine candidate:

- MiniSearch.

Search runs only over decrypted local domain data.

---

## 3. Search index persistence

Plaintext index MAY exist in memory while vault unlocked.

If persisted for startup performance:

- serialize;
- encrypt with a purpose-separated local/search key;
- store ciphertext in IndexedDB.

On vault lock:

- destroy in-memory index;
- terminate/release plaintext references.

---

## 4. Search index is rebuildable

Search index is never authoritative.

If:

- corrupt;
- missing;
- incompatible;
- cryptographic generation changed;

then delete encrypted index and rebuild from decrypted journal documents after unlock.

Never risk journal data to save an index.

---

## 5. Search query privacy

Search queries MUST NOT:

- be sent to Django;
- appear in server logs;
- appear in URL query parameters;
- be stored in browser history.

Search state MAY be ephemeral local UI state.

---

## 6. Search filters

V1 candidate filters:

- journal;
- tag;
- date range;
- favourite;
- mood;
- has attachment;
- has photo;
- has audio.

All evaluated locally.

---

## 7. Export boundary

Readable export is one of the clearest transitions from encrypted to plaintext data.

Export requires:

- unlocked vault;
- explicit user action;
- sensitive-action confirmation/reauthentication;
- clear warning that the resulting archive is plaintext unless user separately encrypts it.

---

## 8. V1 export formats

### Structured JSON

Must preserve enough data for lossless Inkstead re-import:

- schema version;
- journals;
- entries;
- dates/timezones;
- tags;
- favourites;
- mood;
- prompts/templates where user-created;
- attachment relationships.

### Markdown

Human-portable representation.

Attachments included using safe relative paths.

Readable export is constructed entirely client-side.

---

## 9. Export temporary data

While building an archive:

- avoid server upload;
- minimize long-lived plaintext temporary storage;
- revoke Blob URLs after download;
- do not write export contents to logs.

Large export implementation may require streaming APIs; exact approach is benchmarked.

---

## 10. Import security

Imports are hostile input.

Pipeline:

```text
select file
  -> validate archive/schema
  -> bound total size/file count
  -> prevent zip/path traversal
  -> parse
  -> sanitize rich content
  -> preview summary
  -> create new local encrypted objects transactionally
```

Import MUST NOT overwrite an existing object merely because an attacker supplied the same UUID
unless the format/procedure explicitly treats it as a trusted Inkstead restore.

---

## 11. Import staging

Prefer staging imported records before committing them to active vault.

If import fails:

- existing journal remains unchanged;
- partial imported data is discarded or clearly quarantined.

---

## 12. Trash

Default trash retention target:

- 30 days.

Trash is synchronized encrypted domain state.

A user may restore before purge.

---

## 13. Permanent purge

Permanent purge is a separate operation from trash.

Flow:

1. confirm;
2. ensure required local state committed;
3. synchronize purge request/control state;
4. server removes active ciphertext for object;
5. server retains opaque purge marker;
6. referenced blobs are purged according to attachment rules;
7. local replicas remove active object on next sync.

---

## 14. Purge marker

Server-visible opaque marker prevents stale resurrection.

It contains no title/content.

It remains long enough to cover all future stale-replica behavior; preferred v1 is until account
deletion.

---

## 15. Stale replica handling

A stale replica MUST process purge/control state before uploading old work.

If it has edits to a purged object:

- do not restore automatically;
- show conflict/recovery message;
- optionally let user explicitly copy content into a new UUID after local unlock.

---

## 16. Deletion and backups

"Permanent delete" cannot truthfully mean immediate erasure from historical backups.

User-facing wording must distinguish:

- active server deletion;
- local replica deletion;
- backup retention expiration.

Backup policy defines when historical encrypted copies age out.

---

## 17. Cryptographic deletion

Deleting a key can contribute to making ciphertext inaccessible, but Inkstead MUST NOT market key
deletion as guaranteed secure erasure across:

- devices;
- backups;
- memory;
- exported files.

Data lifecycle language remains precise.

---

## 18. Account deletion

Server account deletion requires:

- reauthentication;
- vault-sensitive confirmation;
- clear impact statement.

Server must remove:

- account;
- active sessions;
- MFA secrets;
- vault key envelopes;
- encrypted document/change data;
- encrypted blobs;
- replica records;
- purge markers;
- audit records according to documented security retention policy.

Audit/security retention must not contain journal plaintext.

---

## 19. Local copies on account deletion

Before server account deletion, current browser SHOULD offer to wipe its local encrypted vault.

Other replicas cannot be remotely guaranteed wiped.

Documentation must state that.

If user later opens a disconnected old browser, it may still possess its encrypted local data until
locally cleared.

---

## 20. Local wipe

Local wipe removes:

- vault IndexedDB records;
- encrypted search index;
- local attachment ciphertext;
- device-local wrapped key material;
- application caches where safe.

It must not depend on server availability.

A local wipe is destructive and requires explicit confirmation.

---

## 21. Backup retention

Operational backup retention must be documented, for example:

- daily;
- weekly;
- monthly generations.

Exact policy is deployment-configurable.

User-visible deletion documentation should link to the configured retention concept rather than
promise immediate physical erasure.

---

## 22. Key rotation and search index

When relevant local encryption generation changes:

- invalidate/rebuild search index;
- do not try to keep stale index encrypted under retired key indefinitely.

---

## 23. Data portability

Inkstead must avoid lock-in.

At minimum:

- documented JSON schema;
- Markdown export;
- attachment preservation;
- import of its own export.

Future competitor imports are post-v1 unless separately approved.

---

## 24. Gate 07 tests

Minimum:

1. local search works offline;
2. search query absent from network/server logs;
3. persisted search index is ciphertext;
4. index corruption triggers rebuild;
5. JSON export round-trips;
6. Markdown export contains expected content;
7. export created with server offline;
8. hostile import HTML sanitized;
9. zip-slip/path traversal input rejected;
10. failed import leaves existing vault unchanged;
11. trash/restore across replicas;
12. purge marker prevents resurrection;
13. stale-replica old edit cannot recreate purged UUID;
14. account deletion cascades server data;
15. local copy behavior after account deletion matches documentation.

---

## 25. Gate 07 evidence

```text
docs/evidence/gate-07-data-lifecycle.md
```

Include:

- search network trace;
- encrypted index inspection;
- export/import fixtures;
- hostile import tests;
- purge/stale-replica test;
- server deletion inventory;
- retention-language review.

---

## 26. Blocking decisions before approval

- exact search index persistence strategy;
- exact JSON export schema;
- archive format/streaming library;
- trash retention default;
- purge marker lifetime;
- account-deletion audit retention;
- local wipe UX;
- future import scope.

---

## 27. Exit criteria

Gate 07 passes only when:

- search is fully local and encrypted at rest;
- readable export needs no server plaintext;
- hostile import cannot corrupt active journal;
- deleted data cannot silently resurrect from a stale replica;
- account deletion removes active server data;
- backup/local-copy deletion limitations are accurately communicated.
