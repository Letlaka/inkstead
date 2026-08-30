# Focused Specification 06: Journal Domain, Editor and Metadata Semantics

**Status:** DRAFT  
**Parent:** `../master-product-and-implementation-spec.md`  
**Gate:** gate-04-local-journal  
**Requires:** gate-03-crypto-vault PASS  
**Related gaps:** GAP-022, GAP-030, GAP-042, GAP-047, GAP-051, GAP-058

---

## 1. Purpose

This specification defines the actual journal data and writing experience once reliable encrypted
local storage exists.

No server synchronization is required to pass this stage.

---

## 2. Domain rule

The domain model MUST be usable entirely locally.

Django server models are transport/storage models, not the canonical plaintext journal domain.

---

## 3. Object identifiers

Use opaque cryptographically random identifiers for client-created journal objects.

Preferred:

- UUIDv4 or equivalent random 128-bit identifiers.

Avoid using sequential database IDs in client-visible domain objects.

Avoid UUID formats chosen primarily for timestamp ordering when that timestamp is not required,
because time-encoded identifiers create unnecessary metadata.

---

## 4. Vault

A vault belongs to one Inkstead account and contains:

- journals;
- entries;
- tags;
- prompts/templates;
- local preferences where appropriate.

No sharing/collaboration in v1.

---

## 5. Journal

Conceptual fields:

```text
journal_uuid
name
description?        optional
created_at
updated_at
archived_at?        optional
sort_order
settings
```

Meaningful fields are encrypted.

---

## 6. Entry

Conceptual fields:

```text
entry_uuid
journal_uuid
title?
body_document
entry_date
created_at_utc
modified_at_utc
creation_timezone
tags
mood?
favourite
trash_state
attachment_refs
prompt_ref?
schema_version
```

The exact CRDT representation is defined in the sync spec.

---

## 7. Entry date semantics

A journal entry has two different time concepts.

### Entry date

A user-facing calendar date such as:

```text
2026-08-30
```

It is not a UTC instant.

It determines:

- calendar grouping;
- Today;
- On This Day;
- journal chronology chosen by the user.

### Technical timestamps

Created/modified timestamps are UTC instants for informational ordering/audit-like behavior.

They MUST NOT be trusted as synchronization security sequence numbers.

---

## 8. Timezone

Store the IANA timezone associated with creation/editing where needed for faithful display.

Default application timezone is `Africa/Johannesburg` for the initial deployment, but Inkstead must
remain usable by users in other zones.

On This Day should use the entry's journal date, not reinterpret a UTC instant into a different
calendar day.

---

## 9. Client clock trust

Client timestamps are user-controlled/untrusted.

Do not use them to:

- authorize operations;
- establish replay protection;
- decide sync precedence;
- decide which change is cryptographically newest.

Server sequences/CRDT causality provide technical ordering.

---

## 10. Title

Title is optional.

Inkstead MAY derive a display title locally from the first meaningful line when no title is set.

Derived title remains plaintext only in unlocked client memory and encrypted persistence.

It MUST NOT be placed in browser `document.title`.

---

## 11. Body representation

Canonical rich text SHOULD be Tiptap/ProseMirror JSON with a versioned restricted schema.

Do not make arbitrary HTML the source of truth.

---

## 12. Editor/CRDT integration gate

Tiptap is a ProseMirror-based editor, but that does not automatically make every Tiptap extension
compatible with Automerge's ProseMirror rich-text binding.

The current `@automerge/prosemirror` project is useful and actively developed, but its published
package still describes itself as beta and requires a specific schema mapping.

Before Gate 04 implementation is approved, build a disposable prototype that proves:

1. the exact editor stack can initialize from an Automerge document;
2. every required node/mark maps through the Automerge schema adapter;
3. two independent editor instances can make concurrent rich-text changes;
4. formatting and block structure converge;
5. undo/redo remains local-user understandable;
6. serialization survives reload;
7. unsupported Tiptap extensions are identified before product code depends on them;
8. CSP/Trusted Types/style policy remains compatible.

Decision options:

### Option A - Tiptap + @automerge/prosemirror

Keep Tiptap only if the required Tiptap feature set is proven compatible.

### Option B - direct ProseMirror + @automerge/prosemirror

Prefer this if Tiptap adds schema/transaction behavior that cannot be safely mapped.

### Option C - different CRDT/editor integration

Requires an ADR and equivalent convergence/security evidence.

Product code MUST NOT manually translate arbitrary Tiptap JSON diffs into Automerge operations as a
home-grown synchronization layer.

---

## 13. Initial editor schema

Allow:

- paragraph;
- heading;
- bold;
- italic;
- strike;
- blockquote;
- ordered list;
- bullet list;
- task list;
- horizontal rule;
- safe hyperlink;
- attachment reference.

Reject/strip:

- script;
- iframe;
- arbitrary style blocks;
- inline event attributes;
- dangerous URL schemes;
- arbitrary embedded HTML;
- executable SVG.

---

## 13. Links

Allowed link schemes SHOULD be explicitly enumerated, e.g.:

- https;
- http if user deliberately inserts it, with warning policy considered;
- mailto only if accepted by privacy review.

Reject:

- javascript:;
- data: for arbitrary links;
- vbscript:;
- malformed control-character schemes.

External links use `noopener noreferrer`.

No automatic remote preview fetches in v1.

---

## 14. Paste behavior

Rich paste is untrusted input.

Pipeline:

```text
clipboard HTML
  -> DOMPurify
  -> Tiptap parser
  -> restricted schema
```

Plain text paste remains supported.

Do not fetch remote images automatically from pasted HTML.

---

## 15. Autosave

A typing change is considered locally saved only when:

- CRDT/domain update created;
- ciphertext generated;
- local IndexedDB transaction committed.

Autosave debounce should be short enough to feel instantaneous but avoid per-keystroke expensive
KDF/encryption work.

Encryption uses already-unlocked symmetric keys; Argon2id is not run on every save.

---

## 16. Entry creation

Avoid persisting empty entries merely because the editor opened.

Preferred:

- Today/editor opens empty local draft state;
- first meaningful edit creates the entry UUID and encrypted document;
- subsequent edits update the same entry.

This reduces clutter.

---

## 17. Trash

Trash state is part of encrypted domain state.

Trash UI may show:

- deletion date;
- scheduled purge date.

Permanent purge behavior is specified in the data-lifecycle spec.

---

## 18. Tags

Tags belong to the encrypted vault.

Requirements:

- user-facing spelling preserved;
- canonical matching behavior defined consistently;
- renaming does not silently create duplicate logical tags;
- tags are searchable locally;
- server does not need plaintext tag names.

Blocking decision:

- tags as independent CRDT objects vs embedded normalized metadata.

---

## 19. Mood

Mood is optional.

V1 should use a deliberately small, accessible representation rather than storing health diagnosis
claims.

Possible model:

- optional user-selected value from a configurable scale;
- optional local label.

No mood analytics are sent to server.

---

## 20. Favourites

Favourite is encrypted boolean/domain state.

It has no server plaintext column.

---

## 21. Prompts

Built-in prompts ship as static application resources and work offline.

User-created prompts are encrypted user data.

If a prompt seeds an entry, the resulting answer is a normal encrypted entry.

---

## 22. On This Day

Derived locally from encrypted/decrypted entry dates.

No server-side content analysis required.

---

## 23. Streaks

Derived locally.

Must be optional.

No punitive copy.

A streak does not affect data validity or feature access.

---

## 24. Writing statistics

Potential local-only statistics:

- entries per period;
- words/characters per period;
- days written;
- streak.

Do not upload plaintext statistics unless they can be derived from already accepted server-visible
metadata and there is an explicit product reason.

Prefer local computation.

---

## 25. Search hooks

The domain layer exposes normalized plaintext documents to the in-memory search layer only while
unlocked.

Search persistence is encrypted and specified later.

---

## 26. Metadata-safe browser behavior

Never place sensitive content in:

- URL path/query;
- document title;
- HTML meta description;
- browser history labels;
- network error path;
- analytics (none);
- log messages.

Use opaque UUID routes if route identity is needed, e.g.:

```text
/journal/entry/4b8d.../
```

The visible browser title remains generic.

---

## 27. Privacy cover integration

The editor must be able to unmount or replace sensitive DOM quickly when the app becomes hidden.

Cursor/editor state may be restored from encrypted/local state on return.

---

## 28. Local revision history

Automerge provides historical change information but user-facing revision history must be designed
carefully.

V1 MAY expose basic revision restore after CRDT behavior is proven.

Do not add a second plaintext history database.

---

## 29. Entry size

A maximum practical entry size SHOULD be defined to protect browser performance.

The limit must be high enough for real journals and low enough to prevent accidental multi-megabyte
paste from freezing sync/search.

Exact value is a blocking implementation benchmark decision.

Attachments are separate objects and do not live inline as base64.

---

## 30. Editor error recovery

If local persistence fails:

- do not discard current editor text;
- stop claiming "Saved on this device";
- preserve recoverable in-memory text;
- offer retry and safe copy/export;
- avoid navigating away silently.

---

## 31. Accessibility

Editor requirements:

- keyboard formatting;
- semantic toolbar labels;
- screen-reader-friendly state;
- no focus loss during autosave;
- touch-friendly controls;
- reduced motion;
- selection/cursor stability.

---

## 32. Mobile behavior

On narrow screens:

- editor is primary;
- formatting controls collapse sensibly;
- save/sync status remains visible but non-blocking;
- keyboard appearance does not obscure the active line;
- navigation does not discard editor state.

---

## 33. Gate 04 tests

Minimum:

1. create entry with server offline;
2. edit and reopen locally;
3. delete and restore locally;
4. wrong/corrupt ciphertext fails safely;
5. paste script/iframe is sanitized;
6. dangerous link schemes rejected;
7. no remote image fetch on rich paste;
8. document title contains no entry text;
9. URL contains no entry title/body;
10. Today/calendar uses entry_date semantics;
11. client clock changes do not alter security sequence;
12. local persistence failure preserves editor content;
13. mobile editor flow;
14. keyboard accessibility smoke test;
15. Tiptap/ProseMirror/Automerge prototype passes concurrent rich-text convergence;
16. CSP style policy works without broad unsafe-inline.

---

## 34. Gate 04 evidence

```text
docs/evidence/gate-04-local-journal.md
```

Must contain:

- domain schema version;
- Tiptap schema snapshot;
- sanitization tests;
- offline local CRUD flow;
- metadata leakage inspection;
- date/time test cases;
- accessibility results.

---

## 35. Blocking decisions before approval

- exact entry/title UX;
- entry maximum size;
- tag object model;
- mood representation;
- revision-history v1 scope;
- exact safe link schemes;
- Tiptap vs direct ProseMirror after the Automerge binding spike;
- exact supported ProseMirror schema/extensions;
- draft creation threshold;
- prompt/streak v1 scope.

---

## 36. Exit criteria

Gate 04 passes when:

- complete daily writing works locally/offline;
- autosave is crash-consistent;
- editor input is safely sanitized;
- journal-sensitive text does not leak into browser/network metadata;
- date semantics are deterministic;
- mobile/keyboard flows work;
- the chosen rich-text editor has proven CRDT binding behavior rather than assumed compatibility;
- no server code is needed for ordinary writing.
