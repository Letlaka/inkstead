# Focused Specification 05: PWA, Local Storage, Offline and Update Lifecycle

**Status:** DRAFT  
**Parent:** `../master-product-and-implementation-spec.md`  
**Gate:** gate-02-pwa-offline  
**Requires:** gate-01-security-auth PASS  
**Related gaps:** GAP-005, GAP-006, GAP-007, GAP-008, GAP-013, GAP-017, GAP-019, GAP-020, GAP-021, GAP-031, GAP-032, GAP-037, GAP-041, GAP-048

---

## 1. Purpose

This specification proves that Inkstead is genuinely local-first before the cryptographic vault and
journal domain are layered on top.

The PWA is not an offline cache for a server application. It is a local application that later
synchronizes with Django.

---

## 2. PWA scope

Service Worker scope SHOULD be restricted to:

```text
/journal/
```

The worker MUST NOT control:

```text
/accounts/
/admin/
/users/
/api/
```

unless a future review proves a specific need.

Use the registration scope and server headers to prevent accidental scope broadening.

---

## 3. Cached application shell

Cache only safe application resources:

- journal shell HTML designed for offline use;
- hashed JS bundles;
- CSS;
- icons;
- local fonts;
- manifest;
- static images required by UI.

Do not CacheStorage-cache:

- authentication responses;
- server API data;
- plaintext journal content;
- decrypted blobs;
- dynamic account pages.

---

## 4. IndexedDB

Use Dexie over IndexedDB.

The initial schema should separate concerns:

```text
meta
vaults
documents
outbox
inbox
blobs
search
migration_state
sync_state
```

Exact tables evolve with focused specs, but schema versions MUST be explicit.

---

## 5. User/vault namespace isolation

All local keys/storage records must be scoped by stable local vault/account identity.

Switching authenticated users MUST NOT cause the new user to enumerate another vault's metadata.

Opaque internal records may coexist in one origin database only if every access path enforces the
namespace.

A simpler separate Dexie database per vault MAY be preferred if browser behavior/testing is cleaner.

This is a blocking design decision.

---

## 6. Storage durability

Browser origin data is best-effort by default.

After vault initialization Inkstead SHOULD call:

```js
navigator.storage.persist()
```

when available.

The result must be recorded in local application state.

If persistence is not granted:

- Inkstead still works;
- the user receives a calm, actionable warning;
- unsynced local-only data is clearly distinguished from server-synced data;
- repeated intrusive prompts are avoided.

---

## 7. Storage quota

Use:

```js
navigator.storage.estimate()
```

where supported.

The UI SHOULD monitor broad usage pressure, especially before adding large blobs.

Every IndexedDB write must handle `QuotaExceededError`.

Text persistence receives priority over optional large media.

---

## 8. Site-data deletion

No web application can prevent a user/browser from clearing site data.

Documentation and UI MUST state:

- clearing Inkstead site data removes the local encrypted copy;
- synchronized server ciphertext remains recoverable with valid recovery material;
- unsynced local-only changes may be permanently lost.

Inkstead should not imply browser storage is equivalent to a backup.

---

## 9. First local database initialization

Startup sequence:

```text
load static shell
  -> feature detection
  -> acquire migration/startup lock
  -> open Dexie
  -> run supported schema migration
  -> release migration lock
  -> determine local vault state
  -> render appropriate screen
```

No network dependency.

---

## 10. Multi-tab coordination

Multiple same-origin contexts can race:

- DB migrations;
- replica sequence allocation;
- sync;
- local lock state.

Use Web Locks where supported.

Suggested lock names:

```text
inkstead:<vault-id>:db-migration
inkstead:<vault-id>:sync
inkstead:<vault-id>:writer
```

BroadcastChannel may communicate non-secret state:

- "vault locked";
- "sync completed";
- "new local revision";
- "takeover requested".

Never broadcast cryptographic keys.

---

## 11. Single-writer decision for v1

Preferred v1 simplification:

- only one active writer context per vault/browser profile at a time;
- additional tabs show a safe handoff/read-only screen or request takeover;
- sync leader is therefore deterministic.

This materially reduces CRDT actor/sequence races within one browser.

If multi-writer tabs are desired, the sync spec must model them as independent actors and prove
correctness.

This remains a blocking UX decision.

---

## 12. Fallback when Web Locks unavailable

The browser support matrix must not silently assume Web Locks.

Fallback options:

- IndexedDB lease with expiry + BroadcastChannel;
- explicitly unsupported browser state.

A fallback must be tested against crash/restart. Do not build a forever-held localStorage mutex.

---

## 13. Privacy cover

On `visibilitychange` to hidden:

- render privacy cover/remove sensitive DOM immediately;
- do not automatically destroy the key until configured lock timeout;
- pause expensive UI work;
- keep local committed state intact.

On return:

- if lock grace remains, restore;
- otherwise require vault unlock.

Evaluate `pagehide` and bfcache restoration as well.

---

## 14. Offline startup requirement

After one successful installation/bootstrap, this sequence must pass:

1. close Inkstead;
2. stop Django/Traefik or disconnect network;
3. open installed PWA/browser route;
4. application shell starts;
5. local vault state is discovered;
6. user can unlock once crypto stage exists.

This is mandatory Gate 02 evidence.

---

## 15. Feature detection

At startup detect required browser capabilities, not user-agent strings.

Core requirements likely include:

- IndexedDB;
- Service Worker;
- Web Workers;
- Web Crypto;
- necessary WASM capability if selected;
- TextEncoder/TextDecoder;
- secure context.

Enhancements:

- StorageManager persistence;
- Web Locks;
- BroadcastChannel;
- WebAuthn PRF.

Unsupported core capability should produce a clear compatibility page before journal creation.

---

## 16. Browser support matrix

Before Gate 02 approval define minimum tested targets for:

- Chromium;
- Firefox;
- WebKit/Safari.

"Last two browsers" is not enough for a security/offline contract.

Record:

- tested versions;
- unsupported/degraded capabilities;
- PWA install behavior;
- persistent storage behavior.

---

## 17. Service Worker versioning

Every deployed worker/build needs an application build ID.

Cache names MUST include build/version identity.

Old caches are cleaned during activation only after the new worker is known compatible.

Do not cache forever under generic names such as `inkstead-cache`.

---

## 18. Safe Service Worker activation

Do not use unconditional `skipWaiting()` for ordinary updates.

Preferred flow:

```text
new worker installed
  -> remains waiting
  -> running app detects update
  -> finish local write transaction
  -> verify DB/app compatibility
  -> user-safe reload point
  -> message waiting worker to activate
  -> reload
```

Security-critical updates may use a more forceful path but still must commit local content first.

---

## 19. Service Worker kill switch

Inkstead MUST provide a documented recovery path that can:

- unregister the `/journal/` worker;
- clear Inkstead application caches;
- preserve IndexedDB by default;
- reload fresh application assets.

This recovery route SHOULD live outside Service Worker scope so a broken cached journal shell does
not make recovery impossible.

Do not use broad `Clear-Site-Data` casually because it can destroy local journal storage.

---

## 20. App/schema compatibility

Track independently:

- app build version;
- Dexie schema version;
- crypto protocol version;
- sync protocol/API version;
- CRDT document schema version.

Startup must perform a compatibility check before opening the vault for writes.

---

## 21. Downgrade protection

If local data was migrated by a newer unsupported app version, an older app MUST NOT open it for
writes.

Show:

- "This local vault requires a newer Inkstead version."

Do not attempt best-effort mutation.

---

## 22. IndexedDB migration safety

Dexie schema migrations should use IndexedDB version-change transactions.

For long crypto/content migrations:

- persist migration state;
- make migration resumable;
- do not advance final version marker until migration completes;
- test crash/reload at every migration step.

---

## 23. Local crash consistency

Every user-visible "Saved on this device" state must correspond to a committed IndexedDB
transaction.

When the sync layer is added, local document ciphertext and outbox metadata MUST be committed in one
logical transaction so a saved change cannot disappear from the sync queue.

---

## 24. Local corruption

IndexedDB content is untrusted.

On corrupt/failed-to-authenticate data:

- do not overwrite automatically;
- quarantine logical record;
- attempt recovery from known-good local/server copy;
- preserve diagnostic metadata without plaintext;
- offer controlled recovery.

---

## 25. Server reachability

Do not detect "same Wi-Fi network."

Attempt the configured origin.

Server health probe SHOULD be:

- inexpensive;
- authenticated only if needed;
- free of journal metadata;
- rate limited/sane.

A failed probe changes sync state only.

---

## 26. Network state

Browser `online`/network events are hints, not proof the Inkstead server is reachable.

The sync engine must handle actual request failure.

---

## 27. Local lock without server

Vault lock/unlock must not depend on:

- server health;
- Django session;
- Redis;
- DNS after required cached application assets are available, subject to browser origin behavior.

Note: browser navigation to an HTTPS origin may still involve platform/browser origin rules; this
must be tested on target browsers rather than assumed.

---

## 28. Offline attachments

Once attachment support exists:

- encrypted local blob write must complete before local entry says attachment is safe;
- quota failure must not corrupt entry;
- unsynced attachment state is visible;
- attachment can sync later.

---

## 29. Gate 02 tests

Minimum:

1. installable PWA;
2. offline application shell start;
3. worker scope cannot intercept `/accounts/`;
4. authentication/API response not present in CacheStorage;
5. persistent storage request/result recorded;
6. denied persistence flow;
7. quota-exhaustion handling;
8. schema migration lock across two tabs;
9. writer/sync lock behavior;
10. crashed writer lock recovery;
11. privacy cover on hidden;
12. lock timeout;
13. Service Worker normal update;
14. Service Worker incompatible update refusal;
15. Service Worker kill switch preserving IndexedDB;
16. older app refuses newer DB schema;
17. browser feature-detection screen.

---

## 30. Gate 02 evidence

```text
docs/evidence/gate-02-pwa-offline.md
```

Must include:

- tested browsers/versions;
- storage persistence results;
- CacheStorage inventory;
- worker scope inspection;
- offline startup recording/results;
- multi-tab tests;
- update/kill-switch tests;
- quota failure test.

---

## 31. Blocking decisions before approval

- one active writer tab vs multi-writer tabs;
- local DB per vault vs namespaced shared DB;
- exact browser minimums;
- Web Locks fallback;
- update activation UX;
- persistent-storage warning UX;
- cache version/kill-switch route design.

---

## 32. Exit criteria

Gate 02 passes when:

- the shell is genuinely offline-startable;
- storage durability state is known;
- quota failures are safe;
- multiple tabs cannot corrupt local sequencing/migrations;
- worker scope is constrained;
- updates cannot silently mismatch data schema;
- emergency worker recovery preserves local journal data;
- no unresolved P1 PWA/local-storage gap remains.
