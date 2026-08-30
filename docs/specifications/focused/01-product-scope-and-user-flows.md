# Focused Specification 01: Product Scope and End-to-End User Flows

**Status:** DRAFT  
**Parent:** `../master-product-and-implementation-spec.md`  
**Related gaps:** GAP-019, GAP-022, GAP-026, GAP-037, GAP-042, GAP-047  
**Implementation dependency:** This specification MUST be approved before user-facing implementation is considered stable.

---

## 1. Purpose

This specification defines what Inkstead is from the user's point of view and establishes the
end-to-end flows that every later technical specification must preserve.

It exists to prevent an architecture that is secure in isolation but unpleasant or incoherent as a
journal.

---

## 2. Product promise

Inkstead is:

- private by default;
- self-hosted;
- local-first;
- offline-capable;
- encrypted by default;
- responsive on desktop and mobile;
- designed for daily personal writing.

The default experience MUST prioritize writing over system administration.

---

## 3. V1 account model

V1 SHOULD support multiple independent user accounts on one Inkstead server.

Rules:

- public registration is disabled by default;
- accounts are administrator-provisioned unless explicitly configured otherwise;
- each account owns one independent encrypted vault;
- users do not share journals in v1;
- users cannot see other users' journal metadata;
- local browser storage is namespaced by stable user/vault identifiers;
- switching server accounts MUST lock the currently open local vault before the new account becomes active.

This gives self-hosters flexibility without introducing collaborative-encryption complexity.

---

## 4. V1 journal model

A user may have multiple journals inside one vault.

Example:

```text
Vault
  +-- Personal
  +-- Work
  +-- Gratitude
  +-- Travel
```

Journal names, entry titles, tags, mood, prompts and meaningful attachment metadata SHOULD be
encrypted.

---

## 5. Core user states

The application MUST distinguish these states explicitly:

1. **Unauthenticated / server reachable**
2. **Authenticated / no server-side vault exists**
3. **Authenticated / vault exists / no local vault material**
4. **Local vault initialized / locked**
5. **Local vault unlocked / server reachable / session valid**
6. **Local vault unlocked / server unreachable**
7. **Local vault unlocked / server reachable / session expired**
8. **Local vault version incompatible with current client**
9. **Local vault corrupt/recovery required**
10. **Storage pressure / local persistence degraded**

The UI MUST NOT collapse these into a generic "Something went wrong."

---

## 6. First-run flow

### Preconditions

- server reachable;
- user account authenticated;
- no vault exists for the account.

### Flow

```text
Sign in
  -> welcome to Inkstead
  -> create encrypted vault
  -> generate recovery material client-side
  -> require user to save/confirm recovery material
  -> create local encrypted vault
  -> upload only encrypted/wrapped vault metadata
  -> request persistent browser storage
  -> enter journal
```

### Requirements

- vault setup MUST explain that Inkstead cannot recover encrypted journal content without valid
  recovery material;
- the recovery secret MUST NOT be shown casually again without a sensitive-action confirmation;
- setup MUST verify the user captured recovery material before completion;
- journal plaintext MUST NOT exist on the server during setup.

---

## 7. Existing-vault / new-browser flow

### Preconditions

- user authenticates successfully;
- server says an encrypted vault exists;
- browser has no local vault key material.

### Flow

```text
Sign in
  -> Inkstead detects existing vault
  -> "Unlock this device"
  -> user provides recovery/unlock material
  -> client unwraps Vault Master Key
  -> client validates key against encrypted test/envelope
  -> synchronize encrypted vault state
  -> decrypt locally
  -> request persistent storage
  -> enter journal
```

A successful Django login alone MUST NOT give the browser decryption capability.

---

## 8. Daily return flow

### Local vault locked, server state irrelevant

```text
Open Inkstead
  -> privacy/lock screen
  -> local vault unlock
  -> Today/editor
```

If the server is unavailable:

- writing begins normally;
- synchronization status indicates offline;
- no modal networking error is shown.

---

## 9. Writing flow

Target flow:

```text
Open Today
  -> cursor ready
  -> type
  -> local autosave
  -> subtle "Saved on this device"
  -> background sync when available
  -> subtle "Synced"
```

No network request may block editor input.

No explicit Save button is required for normal writing.

---

## 10. Sync-state vocabulary

The UI MUST use precise language.

Recommended states:

### Saved on this device

The encrypted local transaction committed successfully.

It does not mean the server has a copy.

### Waiting to sync

Local encrypted changes exist that the server has not acknowledged.

### Syncing

A synchronization round is active.

### Synced to server

The server acknowledged all local changes known at the time.

This does not prove another replica has downloaded them.

### Offline

The server is currently unreachable. Local writing remains safe subject to browser storage
durability.

### Sign in to sync

The server is reachable but the Django session expired. Local writing remains available.

### Storage not persistent

The browser did not grant persistent storage. The UI SHOULD explain that local-only unsynced data
may be more vulnerable to browser eviction and SHOULD encourage reconnecting/backing up.

The product MUST NOT use a vague green checkmark to mean all of these different things.

---

## 11. Server outage flow

When the configured Inkstead origin is unreachable:

- the editor remains available;
- local search remains available;
- calendar/timeline remain available for local data;
- attachments already stored locally remain available;
- new local changes are queued;
- the UI displays a non-blocking offline status.

No "Retry" modal should prevent writing.

---

## 12. Expired server session flow

When the server returns 401/403 because the account session expired:

1. current local write completes;
2. pending encrypted changes remain queued;
3. sync pauses;
4. UI shows "Sign in to sync";
5. user may continue writing;
6. authentication is opened only when the user chooses to restore sync or a background-safe flow
   can do so without disrupting writing;
7. successful login resumes sync without requiring journal re-entry if the vault remains within its
   local unlock grace period.

---

## 13. Account-switch flow

A browser may eventually be used by more than one Inkstead account.

Required behavior:

```text
User A vault unlocked
  -> server account changes/logs out
  -> immediately remove User A plaintext from visible UI
  -> terminate User A Crypto Worker
  -> clear in-memory search index
  -> close User A active data handles
  -> authenticate User B
  -> User B sees only User B local vault namespace
```

An account switch MUST NOT carry open-entry state from one user into another.

---

## 14. Background/privacy flow

When the document becomes hidden:

- visible plaintext SHOULD be replaced immediately with a privacy cover;
- the vault key MAY remain active for the configured unlock grace period;
- returning within the grace period can restore the view without another prompt;
- after the lock timeout, the Crypto Worker is terminated and unlock is required.

This protects task-switcher/browser snapshots without turning every app switch into a password
prompt.

---

## 15. Create entry flow

Requirements:

- a new entry gets an opaque UUID;
- it is created locally first;
- title/body metadata remain local plaintext only while unlocked;
- entry state is encrypted before durable IndexedDB commit;
- pending sync information is created atomically with the local write.

No server-generated entry ID is required for local creation.

---

## 16. Edit entry flow

Editing MUST:

- preserve cursor position through local autosave;
- tolerate server/network failure;
- create CRDT changes;
- persist encrypted local state;
- never wait for server acknowledgement before continuing.

---

## 17. Delete and restore flow

Delete:

```text
entry
  -> move to Trash
  -> encrypted tombstone/state update
  -> sync later
```

Restore:

```text
Trash
  -> Restore
  -> encrypted state update
  -> sync later
```

Permanent deletion is a separate sensitive action.

---

## 18. Permanent deletion flow

Permanent deletion MUST:

- state exactly what will be deleted from the server;
- explain that previously synchronized data on another device cannot be remotely guaranteed erased;
- prevent a stale replica from silently resurrecting the object;
- require explicit confirmation;
- use the data-lifecycle protocol defined later.

---

## 19. Recovery flow

User-facing recovery must distinguish:

### Account recovery

Restores access to the Django account.

### Vault recovery

Restores decryption capability.

Account recovery MUST NOT imply journal recovery.

If account recovery succeeds but no vault recovery material exists, the UI must state that the
server cannot decrypt the journal.

---

## 20. Lost-device flow

Session/replica revocation:

- stops future authorized server access;
- MUST NOT claim to erase a local copy already present on the device;
- SHOULD guide the user to additional key-rotation actions when compromise is suspected.

The UI MUST distinguish "Disconnect device" from "Erase device." Inkstead cannot promise remote
device erasure.

---

## 21. Storage-pressure flow

If local storage is near quota:

- warn before large attachments are staged;
- keep text entry saving prioritized over attachments;
- report exact action the user can take;
- do not silently discard unsynced data.

If a write fails with quota exhaustion:

- the editor MUST surface that the current change could not be durably saved;
- keep recoverable in-memory content where possible;
- offer copy/export/retry actions;
- do not falsely show "Saved on this device."

---

## 22. Update flow

Application updates SHOULD be quiet when compatible.

When an update requires reload:

- local writes must be committed before reload is requested;
- the user must not lose unsynced content;
- incompatible schema/crypto clients must refuse unsafe migration/downgrade;
- emergency security updates may force a privacy-preserving reload after safe local commit.

---

## 23. Export flow

Readable export is a sensitive boundary.

```text
Export
  -> explain plaintext result
  -> reauthenticate/confirm vault
  -> client decrypts
  -> client constructs archive
  -> browser downloads archive
```

Django MUST NOT construct readable journal exports.

---

## 24. Import flow

```text
Choose import file
  -> parse as untrusted input
  -> validate schema
  -> sanitize rich content
  -> preview import summary
  -> create local encrypted records
  -> sync later
```

A malformed import must not corrupt the existing vault.

---

## 25. Metadata-safe UI rules

Journal-sensitive text MUST NOT be placed in:

- URL paths;
- query strings;
- `document.title`;
- browser notification previews by default;
- server-rendered error pages;
- CSP report payloads intentionally generated from user text;
- request headers;
- server logs.

Opaque UUIDs MAY appear in URLs if required.

Default page title SHOULD remain generic, such as:

```text
Inkstead
```

not:

```text
Inkstead - My Therapy Notes
```

---

## 26. Same-network semantics

Inkstead does not need fragile "am I on Wi-Fi X?" detection.

The client attempts synchronization against its configured origin.

If `https://inkstead.home.arpa` is only reachable on the user's private network, sync naturally
occurs only there.

If an administrator intentionally exposes the origin through VPN or externally, sync may occur
there too.

This is a deployment/reachability policy, not a journal-data rule.

---

## 27. UX security budget

Routine writing:

- zero MFA prompts;
- zero server-login prompts while offline;
- zero encryption settings;
- zero save confirmations.

Visible security is limited to:

- initial vault setup;
- vault unlock;
- recovery;
- account authentication;
- sensitive export;
- destructive deletion;
- device/session/key management.

---

## 28. Accessibility flows

All core flows MUST work:

- with keyboard only;
- with screen reader semantics;
- at increased text size;
- with reduced motion;
- on touch devices.

Security dialogs MUST remain accessible and MUST NOT rely on color alone.

---

## 29. Flow-test matrix

Before this spec is APPROVED, walk at least these scenarios on paper or prototype:

1. new self-host user with no SMTP;
2. returning user online;
3. returning user with server down;
4. session expires while typing;
5. new browser joins an existing vault;
6. second account uses the same browser profile;
7. storage persistence is denied;
8. storage quota is exhausted mid-attachment;
9. application update arrives while editing;
10. lost device is revoked;
11. user permanently deletes an entry while another replica is offline;
12. readable export is created;
13. user forgets account password but has vault recovery;
14. user has account access but lost vault recovery.

---

## 30. Blocking decisions before approval

This spec remains DRAFT until these are explicitly approved:

- multi-user independent-vault model for v1;
- exact local lock default and privacy-cover behavior;
- user-facing recovery terminology;
- sync-status wording;
- v1 import/export scope;
- v1 streak/reminder scope;
- exact permanent-deletion UX.

---

## 31. Exit criteria

This product-flow spec can be marked APPROVED when:

- every V1 flow has a defined happy path;
- every V1 flow has a network/server failure path;
- security prompts occur only at defined trust boundaries;
- metadata-safe UI rules are accepted;
- account-switch behavior is accepted;
- recovery language clearly separates account and vault recovery;
- no flow requires server availability for ordinary local writing.
