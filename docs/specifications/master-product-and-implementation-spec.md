# Inkstead Master Product and Implementation Specification

**Document status:** Baseline architecture and implementation specification  
**Version:** 1.0  
**Date:** 30 August 2026  
**Project:** Inkstead  
**Repository:** `Letlaka/inkstead`  
**Primary platform:** Responsive web application / Progressive Web Application  
**Backend foundation:** Cookiecutter Django  
**Security assurance target:** OWASP ASVS 5.0 Level 2 plus applicable Level 3 controls  
**Default deployment model:** Fully self-hosted and Internet-independent for normal operation

---

## 1. Purpose

This document is the authoritative baseline for Inkstead product scope, architecture, security,
privacy, user experience, synchronization, deployment, testing, and implementation sequencing.

Inkstead is a privacy-first, local-first journaling application. It is intended to provide a
pleasant daily writing experience while making strong security the default state rather than an
optional configuration.

Future implementation work MUST preserve the invariants defined here. Material deviations require
an Architecture Decision Record (ADR) explaining the reason, security impact, migration impact, and
replacement design.

---

## 2. Normative language

The words **MUST**, **MUST NOT**, **SHOULD**, **SHOULD NOT**, and **MAY** are normative.

- **MUST / MUST NOT**: mandatory project requirement.
- **SHOULD / SHOULD NOT**: expected default; deviations require explicit justification.
- **MAY**: optional.

---

## 3. Product definition

Inkstead is a free and open-source, fully self-hosted, offline-first journaling Progressive Web
Application built on Cookiecutter Django.

It MUST:

- work on desktop and mobile browsers through one responsive web application;
- continue to work after initial setup when the Internet is unavailable;
- allow writing even when the self-hosted server is temporarily unreachable;
- persist journal data locally in encrypted form;
- synchronize automatically when the server becomes reachable again;
- encrypt meaningful journal content before it reaches Django;
- avoid mandatory third-party cloud services;
- avoid telemetry, advertising, and external tracking by default;
- provide a calm, journaling-first experience rather than presenting itself as a security console.

The core product promise is:

> **Your journal. Your server. Your words.**

---

## 4. Product benchmark

Inkstead combines lessons from several mature products rather than attempting to clone any one of
them.

### 4.1 Standard Notes

Use as the benchmark for:

- client-side encryption discipline;
- zero-knowledge-style server storage;
- cryptographic protocol versioning;
- security documentation;
- independent security review;
- long-term encrypted-data compatibility.

### 4.2 Notesnook

Use as the benchmark for:

- encryption that is active by default;
- encrypted metadata;
- encrypted local storage;
- offline operation;
- privacy that does not require constant user interaction;
- straightforward app locking.

### 4.3 Day One

Use as the benchmark for journaling UX:

- immediate writing;
- timeline;
- calendar;
- multiple journals;
- prompts;
- On This Day;
- favourites;
- gentle habit building;
- polished mobile/desktop interaction.

### 4.4 Joplin and TriliumNext

Use as benchmarks for:

- self-hosting;
- import/export;
- offline data ownership;
- operational deployment lessons;
- E2EE migration lessons;
- non-root/container hardening.

### 4.5 Memos

Use as a benchmark for:

- simple self-hosting;
- low-friction capture;
- product restraint;
- no unnecessary infrastructure.

### 4.6 Target position

Inkstead should aim for:

> Day One-quality journaling concepts + Standard Notes-level security thinking + Notesnook-style
> privacy usability + a simple, self-hosted Django operational model.

---

## 5. Core product principles

### 5.1 Local first

Every user-authored journal change MUST succeed locally before synchronization is considered.

Primary write path:

```text
User
  -> editor
  -> local CRDT update
  -> client encryption
  -> IndexedDB
  -> later synchronization
```

A network request MUST NOT sit in the critical path of typing or local autosave.

### 5.2 Encryption by default

Journal encryption MUST be mandatory.

There MUST NOT be an "Enable encryption" option for journal data.

### 5.3 Security by construction

Security controls MUST be active by default and SHOULD normally be invisible to the user.

### 5.4 Security without writing friction

Once a vault is unlocked, normal writing MUST NOT be interrupted by:

- network failure;
- server unavailability;
- expired synchronization session;
- background synchronization errors;
- MFA prompts;
- server maintenance.

### 5.5 Self-hosted means self-contained

Core operation MUST NOT require:

- Firebase;
- hosted authentication;
- proprietary storage services;
- Apple or Google backend services;
- a remote analytics platform;
- public cloud object storage;
- CDN JavaScript;
- third-party web fonts;
- Internet-based license validation;
- remote AI.

### 5.6 Upstream before custom infrastructure

For ordinary application concerns, implementation preference is:

1. Django core;
2. Cookiecutter Django functionality;
3. extension points in already selected packages;
4. mature maintained third-party packages;
5. custom infrastructure only where the Inkstead domain genuinely requires it.

---

## 6. Current repository baseline

The repository was generated from the current Cookiecutter Django generation available at project
creation time.

At this specification baseline the repository contains:

- Django 6.0.8;
- Python 3.14;
- PostgreSQL-oriented Cookiecutter configuration;
- Django REST Framework;
- drf-spectacular;
- django-allauth with MFA extras;
- Argon2 password hashing;
- Redis/django-redis;
- WhiteNoise;
- Webpack;
- Bootstrap 5;
- Docker Compose;
- Traefik production deployment;
- Nginx local media service in the Cookiecutter production model;
- pytest;
- Ruff;
- mypy;
- pre-commit;
- uv-based Python dependency management.

The generated Django timezone is already:

```python
TIME_ZONE = "Africa/Johannesburg"
USE_TZ = True
```

The generated project currently contains Brevo-specific Anymail support. Inkstead's target
architecture does not require a specific external mail provider. Phase 1 MUST deliberately decide
whether to retain optional provider support or reduce the default deployment to generic/local SMTP.
This change MUST NOT be performed implicitly.

The pristine/generated baseline commit MUST remain identifiable in project history.

---

## 7. Technology stack

### 7.1 Server

- Django 6.x from the pinned Cookiecutter baseline;
- PostgreSQL;
- Redis;
- Django REST Framework;
- drf-spectacular;
- django-allauth;
- django-allauth MFA;
- allauth user sessions;
- django-environ;
- django-redis;
- WhiteNoise;
- Gunicorn;
- Traefik;
- Nginx for efficient authorized private blob delivery where applicable.

### 7.2 Additional Django packages

Approved for evaluation/use:

- `django-privates` for authorized private media/blob delivery;
- `django-permissions-policy` for browser capability restrictions;
- `django-health-check` for operational readiness;
- `django-auditlog` for carefully scoped security/administrative auditing.

Packages MUST be pinned only after compatibility and maintenance status are reverified at
implementation time.

### 7.3 Packages not to add without new evidence

Do not introduce these merely to duplicate existing capability:

- django-axes;
- django-two-factor-auth;
- django-otp;
- django-csp;
- django-dbbackup;
- django-guardian;
- django-simple-history.

Reasons include overlap with django-allauth, Django 6 CSP, Cookiecutter backup tooling, Automerge,
or absence of a v1 requirement.

### 7.4 Frontend

- React;
- TypeScript;
- Cookiecutter Webpack pipeline;
- Bootstrap 5 / Sass;
- Tiptap / ProseMirror;
- Dexie / IndexedDB;
- Automerge;
- Workbox;
- libsodium-wrappers-sumo;
- Web Crypto;
- DOMPurify;
- MiniSearch.

Frontend packages MUST be pinned and kept behind lockfiles.

---

## 8. Hybrid application architecture

Inkstead MUST NOT turn every Cookiecutter Django page into a SPA.

Django remains authoritative for conventional server concerns:

```text
/accounts/
/users/
/admin/
/api/
/health/
```

The journaling PWA owns:

```text
/journal/
```

This preserves Django/allauth behavior around authentication, forms, administration, and server
security while allowing the journal itself to operate as a local-first client application.

---

## 9. High-level architecture

```text
                         PRIVATE NETWORK
+----------------------------------------------------------+
|                                                          |
|  Desktop browser                 Mobile browser          |
|  Installed PWA                   Installed PWA           |
|        |                               |                 |
|        +----------- HTTPS ------------+                 |
|                        |                                 |
|                        v                                 |
|                    Traefik                              |
|                        |                                 |
|              +---------+---------+                       |
|              |                   |                       |
|              v                   v                       |
|           Django               Nginx                     |
|           Gunicorn        encrypted private blobs        |
|              |                                           |
|        +-----+------+                                    |
|        |            |                                    |
|        v            v                                    |
|   PostgreSQL       Redis                                  |
|                                                          |
+----------------------------------------------------------+


                      BROWSER / PWA
+----------------------------------------------------------+
| React + TypeScript                                       |
|       |                                                  |
|       +-- Tiptap                                         |
|       +-- Automerge                                      |
|       +-- MiniSearch                                     |
|       |                                                  |
|       v                                                  |
|  Dedicated Crypto Worker                                 |
|       |                                                  |
|       +-- libsodium                                      |
|       +-- Web Crypto                                     |
|       |                                                  |
|       v                                                  |
|  Dexie / IndexedDB                                       |
|       +-- encrypted documents                            |
|       +-- encrypted attachments                          |
|       +-- encrypted search state                         |
|       +-- pending sync changes                           |
|       +-- key envelopes                                  |
|                                                          |
| Workbox Service Worker                                   |
|       +-- application shell/static assets only           |
+----------------------------------------------------------+
```

---

## 10. Architectural security boundary

Django authenticates, authorizes, synchronizes, stores ciphertext, performs operational management,
and provides backup infrastructure.

The browser creates, edits, encrypts, decrypts, searches, merges, and creates readable exports.

The server MUST NOT normally possess the keys required to decrypt journal contents.

Because Inkstead is web-delivered, the server that delivers application JavaScript remains part of
the trusted computing base. Client-side encryption protects strongly against stolen databases,
backups, and passive server-side data inspection, but it cannot fully protect an already unlocked
vault from a server actively compromised and intentionally serving malicious JavaScript.

Inkstead MUST document this limitation and MUST NOT make stronger claims than the architecture can
support.

---

## 11. Authentication

Authentication MUST be based on django-allauth.

Inkstead MUST NOT create custom implementations for:

- login;
- logout;
- password reset;
- password validation;
- MFA;
- passkeys;
- WebAuthn credentials;
- TOTP;
- recovery codes;
- login rate limiting;
- account enumeration protection.

### 11.1 Default offline-friendly account mode

The default self-hosted mode SHOULD use:

- username login;
- public registration disabled;
- SMTP optional;
- email recovery optional;
- no mandatory external email dependency.

The generated Cookiecutter email flow MUST be adjusted deliberately in Phase 1.

### 11.2 MFA

Supported factors SHOULD include:

- passkeys;
- WebAuthn security keys;
- TOTP;
- allauth recovery codes.

Passkeys SHOULD be recommended where supported but MUST NOT be required for basic offline-capable
self-hosting.

### 11.3 Django admin

Production admin access MUST use the allauth authentication flow.

`DJANGO_ADMIN_FORCE_ALLAUTH` MUST be enabled in production.

There MUST NOT be an alternate administrative login that bypasses allauth MFA/rate-limit behavior.

### 11.4 User sessions

Inkstead SHOULD enable `allauth.usersessions` so users can inspect and terminate authenticated
server sessions.

Session records MUST NOT include journal contents or journal-sensitive metadata.

---

## 12. Account authentication and vault encryption are separate

Django authentication answers:

> May this person use this Inkstead server?

Vault encryption answers:

> Can this browser decrypt this user's journal?

These MUST remain independent.

Changing or resetting a Django password MUST NOT directly decrypt journal data and MUST NOT require
re-encrypting every entry.

---

## 13. Cryptographic protocol

Inkstead MUST use a versioned cryptographic protocol from the first encrypted record.

Example protocol ID:

```text
INKSTEAD-CRYPTO-v1
```

Every encrypted envelope MUST identify enough information to support future migration, including:

- protocol version;
- algorithm identifier;
- key identifier where appropriate;
- nonce;
- ciphertext;
- authenticated context.

No unversioned ciphertext format is permitted.

---

## 14. Key hierarchy

Recommended v1 hierarchy:

```text
                 Recovery Secret / unlock secret
                          |
                          v
                       Argon2id
                          |
                          v
                    Wrapping Key
                          |
                          v
                 +----------------+
                 | Vault Master   |
                 | Key (random)   |
                 +-------+--------+
                         |
             +-----------+-----------+
             |           |           |
             v           v           v
       document keys   blob keys   local/index keys
```

The Vault Master Key MUST be random.

It MUST NOT simply be a password hash.

Password/recovery-secret changes SHOULD rewrap keys rather than require rewriting every journal
entry.

---

## 15. Recovery

Initial vault setup MUST create a high-entropy recovery mechanism client-side.

The recovery secret:

- MUST NOT be transmitted to Django in plaintext;
- MUST be clearly presented during setup;
- SHOULD be suitable for storage in a password manager or offline copy;
- MUST be sufficient to recover the encrypted vault when supported by the protocol.

Inkstead MUST NOT contain:

- an administrator master decryption key;
- a hidden recovery backdoor;
- a server-side emergency plaintext export facility.

Loss of all valid recovery mechanisms may make journal data unrecoverable. The UI and documentation
MUST state this plainly.

---

## 16. Cryptographic primitives

The v1 design SHOULD use well-reviewed primitives from established libraries, including:

- Argon2id for password/key derivation;
- XChaCha20-Poly1305 for authenticated encryption;
- HKDF/domain separation where appropriate;
- cryptographically secure random 256-bit keys.

Cryptographic algorithms MUST NOT be implemented manually.

Exact parameters MUST be fixed in the crypto protocol specification and MUST be versionable.

---

## 17. Per-document and attachment keys

Each journal entry SHOULD have an independent random document key.

Each attachment SHOULD have an independent random blob key.

Meaningful attachment metadata, such as original filename and caption, MUST be encrypted.

For large attachments the design SHOULD support authenticated chunked encryption so the entire file
does not need to be held in memory.

---

## 18. Authenticated encryption context

AEAD associated data SHOULD bind ciphertext to its context.

Depending on envelope type, associated data may include:

- crypto protocol version;
- journal UUID;
- document UUID;
- envelope type;
- key ID;
- replica UUID;
- change UUID;
- client sequence.

This reduces the risk of valid ciphertext being transplanted into an unintended context.

---

## 19. Crypto Worker

Long-lived vault keys MUST NOT be stored in ordinary React state.

Cryptographic operations SHOULD execute inside a dedicated Web Worker.

Conceptual API:

```text
unlock()
lock()
encrypt()
decrypt()
wrapKey()
unwrapKey()
deriveKey()
```

The Service Worker MUST NOT hold vault keys.

The Crypto Worker is defense in depth, not a claim that XSS becomes harmless.

---

## 20. XSS protection

XSS is a high-priority client threat because malicious same-origin JavaScript running while a vault
is unlocked can observe displayed plaintext or request decryption operations.

Inkstead MUST use layered defenses:

- Django 6 native CSP;
- strict React escaping behavior;
- Trusted Types where compatible;
- DOMPurify for hostile HTML;
- restrictive Tiptap schema;
- no arbitrary executable HTML;
- no remote JavaScript;
- no third-party embeds in v1;
- no plugin system in v1.

---

## 21. Content Security Policy

Django 6 native CSP MUST be used rather than adding duplicate CSP middleware.

Target policy should converge toward:

```text
default-src 'self'
script-src 'self'
style-src 'self'
font-src 'self'
connect-src 'self'
img-src 'self' data: blob:
media-src 'self' blob:
worker-src 'self'
object-src 'none'
frame-src 'none'
frame-ancestors 'none'
base-uri 'self'
form-action 'self'
manifest-src 'self'
```

Development SHOULD begin with report-only where necessary, but v1 production MUST enforce CSP.

---

## 22. Permissions Policy

Browser capabilities that Inkstead does not need MUST be denied.

Examples:

- geolocation: denied by default;
- payment: denied;
- USB: denied;
- serial: denied;
- Bluetooth: denied;
- camera: denied until an explicit photo-capture feature requires it;
- microphone: denied until an explicit audio-recording action requires it.

Permissions MUST be requested only in direct response to a user action.

---

## 23. Local persistence

Use Dexie over IndexedDB.

IndexedDB MAY contain:

- encrypted Automerge documents;
- encrypted attachment data;
- pending encrypted synchronization changes;
- replica state;
- encrypted search state;
- encrypted key envelopes;
- non-sensitive local preferences.

IndexedDB MUST NOT persist journal plaintext.

---

## 24. Service Worker

Use Workbox for PWA shell caching and offline application startup.

The Service Worker MAY cache:

- application HTML shell;
- compiled JavaScript;
- CSS;
- fonts bundled with Inkstead;
- icons;
- manifest;
- safe static assets.

The Service Worker MUST NOT cache:

- plaintext journal content;
- private decrypted attachments;
- authentication responses;
- admin responses;
- raw keys.

API responses containing synchronized ciphertext SHOULD use explicit cache controls and SHOULD NOT
be treated as application-shell cache objects.

---

## 25. Service Worker scope

The PWA Service Worker SHOULD be scoped to:

```text
/journal/
```

Authentication and administration SHOULD remain outside that scope:

```text
/accounts/
/users/
/admin/
/api/
```

---

## 26. Offline contract

After initial vault setup, the following operations MUST work while Django is unavailable:

- open the local journal;
- unlock the vault;
- create an entry;
- edit an entry;
- delete an entry locally;
- restore an entry;
- change tags;
- favourite/unfavourite;
- browse timeline;
- browse calendar;
- search locally;
- view locally cached encrypted attachments after decryption;
- stage new local attachments;
- use offline prompts;
- change local-only preferences.

An expired Django session is a synchronization problem, not a writing problem.

---

## 27. Autosave

There MUST be no manual Save requirement for ordinary entry editing.

Recommended local save path:

```text
Tiptap
  -> debounce
  -> Automerge change
  -> encrypt
  -> IndexedDB
```

UI state SHOULD distinguish:

- **Saved on this device**
- **Waiting to sync**
- **Synced**

The application MUST NOT display a server-centric "unsaved changes" warning when local encrypted
persistence has succeeded.

---

## 28. Editor

Use Tiptap/ProseMirror.

The canonical rich-text representation SHOULD be structured ProseMirror/Tiptap JSON rather than
arbitrary HTML.

Initial schema:

- paragraphs;
- headings;
- bold;
- italic;
- strike;
- blockquote;
- ordered lists;
- unordered lists;
- task lists;
- links;
- horizontal rule;
- attachment references.

No arbitrary:

- script;
- iframe;
- executable SVG;
- raw HTML embed;
- third-party interactive embed.

---

## 29. Paste and import sanitization

Untrusted HTML MUST follow:

```text
untrusted input
   -> DOMPurify
   -> restricted Tiptap parser/schema
   -> document model
```

User-provided HTML MUST NOT be assigned directly to `innerHTML`.

Importers MUST be treated as hostile-input parsers.

---

## 30. Synchronization model

Inkstead SHOULD use Automerge CRDT documents.

One journal entry SHOULD correspond to one Automerge document.

Do not store an entire journal as one giant CRDT.

This enables independent offline editing, more manageable compaction, and conflict-safe convergence.

---

## 31. Replica model

Every initialized browser/PWA installation receives a random replica UUID.

A replica represents synchronization state only. It is not an authentication credential.

Replica state may include:

- replica UUID;
- created time;
- last successful sync;
- last server cursor;
- next client sequence;
- revoked time.

Django/allauth sessions remain the authentication boundary.

---

## 32. Sync transport

Use DRF and same-origin Django session authentication.

The PWA SHOULD NOT require persistent bearer tokens.

Suggested API surface:

```text
GET    /api/v1/sync/changes/
POST   /api/v1/sync/changes/
POST   /api/v1/sync/snapshots/
POST   /api/v1/blobs/
GET    /api/v1/blobs/{uuid}/
POST   /api/v1/replicas/
DELETE /api/v1/replicas/{uuid}/
```

The exact contract MUST be documented through drf-spectacular/OpenAPI.

---

## 33. Synchronization lifecycle

```text
PWA opens
   |
   v
local journal immediately available
   |
   v
check server reachability
   |
   +-- unavailable --> continue offline
   |
   v
check Django session
   |
   +-- expired --> continue offline; request login only when sync is needed
   |
   v
pull encrypted changes after cursor
   |
   v
decrypt locally and apply to Automerge
   |
   v
push pending encrypted changes
   |
   v
upload pending encrypted blobs
   |
   v
advance synchronization cursor
```

---

## 34. Sync triggers

Synchronization SHOULD be attempted:

- at application startup;
- when returning to foreground;
- after browser `online` events;
- periodically while the app remains visible;
- shortly after meaningful edits;
- when manually requested.

Rapid edits SHOULD be batched.

Inkstead MUST NOT promise reliable synchronization while the browser/PWA has been completely
suspended by the operating system.

---

## 35. Sync reliability requirements

The synchronization protocol MUST support:

- idempotent submissions;
- duplicate change detection;
- monotonically increasing replica sequence;
- server high-water mark/cursor;
- safe retry;
- resumable synchronization;
- replay detection/rejection where applicable;
- snapshot compaction;
- corrupt envelope detection;
- explicit protocol/schema version compatibility.

Network failure at any stage MUST NOT corrupt the locally persisted journal.

---

## 36. Server data model

Django stores encrypted transport objects, not plaintext journal records.

Conceptual model:

```text
User
 |
 +-- Journal
      |
      +-- Replica
      |
      +-- SyncDocument
      |    +-- ChangeEnvelope
      |    +-- SnapshotEnvelope
      |
      +-- Blob
```

### 36.1 Journal

Server-readable fields may include:

- UUID;
- owner;
- crypto protocol version;
- wrapped master-key envelope;
- recovery envelope;
- timestamps.

The user-facing journal name SHOULD remain encrypted.

### 36.2 ChangeEnvelope

Suggested fields:

- UUID;
- journal;
- document UUID;
- replica;
- client sequence;
- server sequence;
- protocol version;
- key ID;
- ciphertext;
- ciphertext hash;
- created time.

### 36.3 SnapshotEnvelope

Suggested fields:

- journal;
- document UUID;
- through-server-sequence marker;
- protocol version;
- key ID;
- ciphertext;
- ciphertext hash;
- created time.

### 36.4 Blob

Suggested server-visible fields:

- UUID;
- journal;
- encrypted file;
- ciphertext hash;
- ciphertext size;
- created time;
- deleted time.

Original meaningful filenames SHOULD remain encrypted.

---

## 37. Attachments

V1 SHOULD support:

- JPEG;
- PNG;
- WebP;
- browser-supported audio;
- PDF;
- generic downloadable files within administrator-defined limits.

SVG and HTML MUST NOT be rendered inline by default.

Attachments MUST be encrypted before durable server storage.

Use `django-privates` or equivalent maintained Django infrastructure for authorized delivery of
ciphertext blobs, with Nginx/sendfile acceleration where appropriate.

---

## 38. Search

Search MUST remain client-side for encrypted journal data.

MiniSearch is the initial candidate.

Search SHOULD cover:

- title;
- body;
- tags;
- journal;
- date;
- mood;
- favourite state;
- attachment presence.

Potential filters:

```text
tag:
journal:
before:
after:
has:attachment
has:photo
has:audio
is:favourite
```

Search queries MUST NOT be sent to Django for ordinary encrypted journal search.

Persisted search indexes MUST be encrypted.

Plaintext indexes MAY exist only in memory while the vault is unlocked.

---

## 39. Journal locking

"Sign out" and "Lock journal" are separate actions.

### Sign out

Ends the Django/allauth server session.

### Lock journal

Destroys active local decryption state while leaving encrypted IndexedDB data intact.

Recommended default:

- lock after 15 minutes hidden/backgrounded.

Configurable choices MAY include:

- immediately;
- 5 minutes;
- 15 minutes;
- 30 minutes;
- 1 hour;
- when PWA/browser closes.

Typing MUST NOT trigger lock prompts.

---

## 40. Sensitive-action reauthentication

Normal writing MUST NOT require reauthentication.

Sensitive actions SHOULD require explicit user confirmation and, where appropriate, allauth
reauthentication/passkey verification or vault confirmation.

Examples:

- plaintext full export;
- viewing recovery material;
- changing recovery credentials;
- deleting all journal data;
- removing a trusted replica/session;
- disabling an MFA factor.

---

## 41. Journaling feature scope

### 41.1 Core writing

V1 SHOULD provide:

- multiple journals;
- entries;
- rich text;
- Markdown-style shortcuts;
- tags;
- favourites;
- mood;
- entry date;
- attachments;
- photos;
- audio where browser support permits;
- autosave;
- trash;
- restore;
- local revision history through CRDT/document history where practical.

### 41.2 Reflection

- Timeline;
- Calendar;
- On This Day;
- favourites review;
- prompts;
- templates;
- search.

### 41.3 Habit support

- optional streaks;
- optional writing goals;
- gentle reminders where browser capability permits.

Streaks MUST be optional and SHOULD NOT use punitive or shaming language.

---

## 42. Navigation

Primary navigation:

- Today;
- Timeline;
- Calendar;
- Journals;
- Search;
- Favourites.

Secondary:

- On This Day;
- Tags;
- Prompts;
- Trash;
- Settings.

---

## 43. Responsive UX

Inkstead is one responsive PWA, not separate desktop and mobile sites.

### Desktop target

Three-pane layout may be used:

```text
+--------------+--------------------+------------------------+
| Navigation   | Entry list         | Editor                 |
+--------------+--------------------+------------------------+
```

### Mobile target

Single-focus navigation should prioritize the entry/editor while retaining quick access to Today,
Calendar, Journals, and Search.

The editor MUST remain comfortable on touch screens.

---

## 44. User-experience rule for security

Security should feel like gravity: always active, rarely demanding attention.

Invisible controls include:

- encryption;
- local encryption;
- encrypted synchronization;
- TLS;
- HSTS;
- CSP;
- CSRF;
- secure cookies;
- sanitization;
- rate limiting;
- private media authorization;
- metadata minimization;
- safe logging;
- dependency validation.

Visible security interactions are reserved for real trust boundaries.

---

## 45. Accessibility

Target WCAG 2.2 AA.

Requirements include:

- complete keyboard navigation;
- semantic landmarks;
- proper labels;
- visible focus;
- sufficient contrast;
- reduced-motion support;
- scalable text;
- adequate touch targets;
- editor keyboard accessibility;
- no color-only status meaning.

Accessibility checks MUST be part of release verification.

---

## 46. Privacy defaults

Default installation MUST include no external:

- analytics;
- advertising;
- tracking pixels;
- crash-report upload;
- profiling;
- CDN scripts;
- remote fonts;
- remote AI.

Operational logs remain under the self-host administrator's control.

---

## 47. Logging

Server logs MAY contain:

- timestamps;
- request IDs;
- endpoints;
- response status;
- duration;
- infrastructure errors;
- authentication security events.

Logs MUST NOT contain:

- journal plaintext;
- entry titles;
- journal names;
- search terms;
- decrypted filenames;
- keys;
- passwords;
- session cookies;
- authorization credentials;
- request bodies containing journal ciphertext except under explicitly controlled local debugging.

Ciphertext SHOULD NOT normally be logged either.

---

## 48. Audit logging

Audit administrative/security events only.

Good audit candidates:

- account changes;
- MFA changes;
- replica/session revocation;
- administrative configuration changes;
- journal deletion request;
- recovery-security changes.

Do not audit encrypted journal payloads.

An audit trail MUST NOT become a second copy of the user's journal.

---

## 49. Health monitoring

Health checks SHOULD verify:

- Django;
- PostgreSQL;
- Redis;
- encrypted blob storage.

Public health output MUST be minimal.

Detailed diagnostics SHOULD require administrative access.

---

## 50. Deployment

Primary supported deployment is Cookiecutter Django Docker Compose.

Expected services:

```text
traefik
django
postgres
redis
nginx
```

Only Traefik SHOULD normally bind to the LAN.

PostgreSQL and Redis MUST NOT expose production host ports to the LAN.

---

## 51. HTTPS

Production HTTP-only operation is unsupported.

HTTPS is required for:

- credential protection;
- PWA secure-context behavior;
- Service Workers;
- WebAuthn/passkeys;
- protection against LAN injection.

Deployment MUST support a path that does not require public Internet access, such as:

- administrator-provided certificate;
- trusted private/local CA.

A public ACME certificate MAY be supported but MUST NOT be required.

---

## 52. Hostname

A stable deployment hostname is important because browser secure-origin and WebAuthn identity depend
on it.

`inkstead.home.arpa` is a suitable default private-network convention, but deployment
documentation SHOULD permit administrator-controlled DNS names.

Changing the relying-party hostname after WebAuthn credentials exist requires explicit migration
consideration.

---

## 53. Container hardening

Production containers SHOULD use:

- non-root users;
- minimum capabilities;
- `no-new-privileges`;
- read-only root filesystems where feasible;
- explicit writable volumes only;
- private Docker networks;
- pinned base images.

Cookiecutter's generated Django production container already runs as an unprivileged `django`
user and that behavior MUST be preserved.

---

## 54. Redis

Redis MAY be used for:

- Django caching;
- allauth/rate-limit support where appropriate;
- transient server state.

Redis MUST NOT contain plaintext journal contents.

---

## 55. Celery

Celery is not part of the initial Inkstead architecture.

Do not add a distributed task queue until a real requirement justifies its operational complexity.

Initial background/maintenance work SHOULD use:

- synchronous request work where appropriate;
- Django management commands;
- host scheduling/systemd/cron for backups and maintenance.

---

## 56. CORS and API exposure

The Inkstead PWA is same-origin.

Production MUST NOT use wildcard CORS.

DRF browser endpoints SHOULD use `SessionAuthentication` and Django CSRF protection.

Persistent DRF token authentication SHOULD be removed/disabled for journal API usage unless a later
external client requirement justifies it.

---

## 57. Backup model

Two backup concepts MUST remain distinct.

### 57.1 Operational server backup

Contains:

- PostgreSQL;
- encrypted blobs;
- encrypted key envelopes;
- configuration required to restore service;
- integrity metadata.

Journal contents remain ciphertext.

### 57.2 User-readable export

Contains:

- Markdown;
- JSON;
- attachments.

Readable exports MUST be generated client-side from an unlocked vault.

---

## 58. Backup implementation

Reuse Cookiecutter Django's PostgreSQL backup/restore tooling rather than adding duplicate database
backup infrastructure.

Backup scope MUST eventually cover:

- database;
- encrypted blob volume;
- critical deployment metadata;
- certificates/configuration where required for recovery;
- integrity manifest.

Backup targets MAY include:

- local disk;
- NAS;
- removable storage;
- user-controlled remote storage.

No Internet backup target is mandatory.

---

## 59. Backup verification

Inkstead MUST test restoration, not merely creation of backups.

The product SHOULD surface:

- last successful backup;
- last integrity check;
- last successful restore test.

A backup is not considered proven until restore has been tested.

---

## 60. Export and import

### V1 export

- Markdown archive;
- structured JSON archive;
- attachments.

Export SHOULD preserve:

- journals;
- timestamps;
- tags;
- mood;
- favourite state;
- attachment relationships.

### V1 import

- Inkstead JSON;
- Inkstead Markdown export.

Future importers MAY support Day One, Joplin, Standard Notes, and Notesnook.

Every importer MUST validate and sanitize hostile input.

---

## 61. Trash and deletion

Deletion SHOULD initially be soft.

Default retention target:

- 30 days before permanent purge.

Synchronization MUST preserve tombstones long enough for offline replicas to learn about deletion.

Permanent deletion MUST require explicit confirmation.

---

## 62. Security assurance target

Inkstead uses OWASP ASVS 5.0 as the formal application-security verification framework.

Target:

- all applicable Level 1;
- all applicable Level 2;
- applicable Level 3 controls covering cryptography, authentication, session security,
  client-side security, sensitive data, administration, and deployment.

Maintain:

```text
docs/security/asvs-5-matrix.md
```

Each requirement MUST eventually be marked:

- implemented;
- not applicable;
- accepted risk;
- planned.

---

## 63. Threat model

Inkstead MUST maintain a living threat model.

Minimum threats and mitigations:

| Threat | Primary mitigation |
| --- | --- |
| Stolen PostgreSQL database | client-side encryption |
| Stolen server disk | ciphertext-only journal data |
| Stolen server backup | ciphertext-only journal data |
| LAN interception | HTTPS |
| Credential guessing | Argon2 + allauth rate limiting + MFA |
| Phishing | WebAuthn/passkeys |
| CSRF | Django CSRF |
| SQL injection | Django ORM + validation |
| XSS | CSP + escaping + Trusted Types + sanitization |
| Browser-profile theft | encrypted IndexedDB |
| Physical access to unlocked device | vault lock |
| Sync tampering | authenticated encryption |
| Sync replay/duplicates | sequence + idempotency controls |
| Malicious attachment | validation + restricted rendering |
| Dependency compromise | lockfiles + scans + SBOM + review |
| Container compromise | non-root + least privilege |
| Logging disclosure | strict sensitive-data logging policy |
| Broken backup | restore verification |
| Admin brute force | allauth-secured admin |
| Actively compromised web server | documented web-delivery trust boundary |

---

## 64. Plaintext canary release test

Create a unique marker in a journal entry, for example:

```text
INKSTEAD_PLAINTEXT_MUST_NOT_LEAK_93C1A8
```

After synchronization and backup, automated/integration checks MUST search:

- PostgreSQL dump;
- Redis where inspectable;
- encrypted blob storage;
- Django logs;
- Nginx logs;
- Traefik logs;
- backup artifacts.

The marker MUST NOT appear.

This is a permanent release gate.

---

## 65. Offline end-to-end release test

Automated browser flow MUST cover:

```text
login
unlock
create entry
disconnect network
edit entry
restart/reopen PWA offline
unlock
verify entry
edit again
reconnect
sync
open second browser context
verify converged content
```

This is a permanent release gate.

---

## 66. Multi-replica convergence tests

Test at least:

- independent edits in different paragraphs;
- edits to the same paragraph;
- deletion versus edit;
- duplicated change upload;
- interrupted upload;
- interrupted pull;
- stale cursor;
- snapshot recovery;
- replica revocation.

Data MUST converge or surface a deterministic, recoverable conflict outcome. Silent data loss is
unacceptable.

---

## 67. Security-header test

CI/integration verification MUST cover:

- Content-Security-Policy;
- Strict-Transport-Security;
- Permissions-Policy;
- X-Content-Type-Options;
- Referrer-Policy;
- frame restrictions;
- Secure cookies;
- HttpOnly cookies;
- SameSite;
- absence of unintended external executable resources.

---

## 68. Supply-chain security

Every production release SHOULD include:

- Python vulnerability scanning;
- npm vulnerability scanning;
- container image scanning;
- secret scanning;
- static analysis;
- lockfile verification;
- SBOM generation.

Future mature releases SHOULD add:

- signed checksums;
- signed container artifacts;
- provenance attestations.

Production dependencies MUST be pinned via lockfiles.

---

## 69. Required repository security documents

Before v1, create:

```text
SECURITY.md
PRIVACY.md
CONTRIBUTING.md
docs/security/THREAT_MODEL.md
docs/security/CRYPTOGRAPHY.md
docs/security/ASVS-5-MATRIX.md
docs/security/SECURE-DEVELOPMENT.md
docs/architecture/
docs/adr/
docs/backup/
```

---

## 70. Cookiecutter upstream tracking

A generated Cookiecutter project does not automatically inherit later template improvements.

Inkstead MUST record:

- Cookiecutter Django release/commit used;
- generation date;
- generation options.

Create:

```text
docs/upstream-cookiecutter.md
```

Relevant upstream security, dependency, Docker, Django, deployment, and test improvements SHOULD be
reviewed periodically.

---

## 71. Dependency policy

Every new dependency MUST be evaluated for:

- active maintenance;
- current compatibility;
- license suitability;
- supply-chain risk;
- dependency-tree size;
- telemetry behavior;
- actual benefit over existing capability.

Dependencies MUST NOT be added merely to save a few lines of ordinary code when they introduce a
large maintenance/security surface.

---

## 72. Development workflow

Feature and architecture work SHOULD use dedicated branches and pull requests.

Each task follows:

```text
specification
 -> tests / expected behavior
 -> implementation
 -> local verification
 -> security checks
 -> review
 -> merge
```

Material architecture changes require an ADR.

The `main` branch should represent reviewed, mergeable project state.

---

# Implementation Master Plan

## Phase 0 - Preserve and verify Cookiecutter baseline

### Deliverables

- record Cookiecutter upstream version/commit;
- record generation options;
- confirm timezone `Africa/Johannesburg`;
- preserve initial generated commit;
- verify generated project locally;
- verify Docker local environment;
- verify production image build.

### Verification

- pytest;
- Ruff;
- mypy;
- pre-commit;
- Django system checks;
- Docker build.

### Exit criterion

The generated foundation is known-good before Inkstead-specific architecture changes begin.

---

## Phase 1 - Security and authentication baseline

### Deliverables

- disable public registration by default;
- decide/remove default mandatory external email dependency;
- review current Brevo coupling and replace with provider-neutral behavior if appropriate;
- configure allauth account flow;
- configure MFA;
- configure passkeys/WebAuthn;
- configure recovery codes;
- enable user-session visibility;
- force admin through allauth;
- configure allauth proxy/rate-limit trust correctly behind Traefik;
- add/enforce Django CSP;
- add Permissions Policy;
- verify security-cookie settings;
- establish local/private HTTPS deployment guidance;
- add health checks.

### Exit criterion

Authentication/session/browser-security baseline is verified before journal content exists.

---

## Phase 2 - PWA shell

### Deliverables

- React/TypeScript integration into Cookiecutter Webpack;
- `/journal/` application entry point;
- manifest;
- Workbox;
- scoped Service Worker;
- responsive app shell;
- installability;
- static offline startup.

### Critical test

Install/open the PWA, stop Django, reopen it, and confirm the cached application shell starts.

---

## Phase 3 - Encrypted local vault

### Deliverables

- Dexie schema;
- IndexedDB storage layer;
- dedicated Crypto Worker;
- random Vault Master Key;
- recovery-secret workflow;
- Argon2id wrapping;
- versioned encryption envelopes;
- local lock/unlock;
- encrypted persistence.

### Critical security test

A plaintext canary stored in an entry MUST NOT be discoverable in raw IndexedDB.

---

## Phase 4 - Core journal domain

### Deliverables

- multiple journals;
- entries;
- dates;
- tags;
- favourites;
- mood;
- trash/restore;
- local metadata;
- autosave.

Everything MUST function while Django is offline.

---

## Phase 5 - Editor

### Deliverables

- Tiptap;
- restricted schema;
- Markdown-style shortcuts;
- formatting controls;
- undo/redo;
- mobile toolbar;
- paste sanitization;
- DOMPurify;
- editor accessibility.

### Security tests

- hostile HTML;
- malformed paste;
- unsafe link schemes;
- attempted iframe/script/SVG injection.

---

## Phase 6 - CRDT layer

### Deliverables

- one Automerge document per entry;
- replica identity;
- local serialized changes;
- local snapshots;
- deterministic merge tests;
- document-version migration hooks.

### Exit criterion

Concurrent local documents converge correctly before server synchronization is implemented.

---

## Phase 7 - Django sync transport

### Deliverables

- Journal model;
- Replica model;
- SyncDocument model where useful;
- ChangeEnvelope;
- SnapshotEnvelope;
- DRF endpoints;
- cursor/high-water-mark design;
- idempotency;
- sequence validation;
- OpenAPI schema.

### Security invariant

No plaintext journal payload crosses the Django API boundary.

---

## Phase 8 - Multi-replica synchronization

### Deliverables

- pull;
- push;
- retry;
- duplicate handling;
- snapshot handling;
- interrupted-sync recovery;
- replica revocation;
- foreground automatic sync.

### Exit criterion

Two independent browser profiles can edit offline and later converge without silent data loss.

---

## Phase 9 - Encrypted attachments

### Deliverables

- local attachment staging;
- attachment encryption;
- independent blob keys;
- upload limits;
- private authorized delivery;
- Nginx/sendfile acceleration where appropriate;
- image preview;
- audio support;
- safe downloads.

---

## Phase 10 - Local search

### Deliverables

- MiniSearch integration;
- title/body/tag search;
- filters;
- encrypted persisted index;
- rebuild capability;
- incremental updates.

### Security test

Search queries MUST NOT appear in Django requests or logs.

---

## Phase 11 - Journaling experience

### Deliverables

- Today;
- Timeline;
- Calendar;
- Journals;
- Tags;
- Favourites;
- Prompts;
- On This Day;
- optional streaks;
- writing statistics.

This phase turns the secure storage engine into a pleasant journaling product.

---

## Phase 12 - Responsive UX and accessibility

### Verify on

- desktop;
- laptop;
- tablet;
- narrow mobile;
- portrait;
- landscape;
- installed PWA;
- standard browser tab.

### Browser targets

- Chromium;
- Firefox;
- Safari/WebKit.

### Accessibility

- WCAG 2.2 AA review;
- keyboard navigation;
- screen reader semantics;
- touch-target verification;
- reduced motion.

---

## Phase 13 - Export and import

### Deliverables

- client-side structured JSON export;
- Markdown export;
- attachment export;
- safe import;
- schema validation;
- sanitization;
- migration hooks.

Readable journal exports MUST NOT be produced server-side.

---

## Phase 14 - Backup and recovery

### Deliverables

- Cookiecutter PostgreSQL backup integration;
- encrypted blob backup;
- integrity manifest;
- restore instructions;
- restore verification script;
- backup health/status.

### Critical test

Destroy a test deployment, restore from backup, sync a browser, unlock, and verify expected data.

---

## Phase 15 - Operational security

### Deliverables

- audit logging;
- safe structured logs;
- hardened containers;
- private service networks;
- upload limits;
- production rate limits;
- security configuration validation;
- secret handling.

---

## Phase 16 - Security verification

### Deliverables

- ASVS Level 1/2 mapping;
- applicable Level 3 mapping;
- XSS test suite;
- CSRF verification;
- authentication attack tests;
- plaintext canary;
- raw database review;
- Redis review;
- log review;
- browser-storage review;
- dependency audit;
- container scan;
- backup leakage test.

---

## Phase 17 - V1 readiness and release

V1 is achieved when a user can:

1. deploy Inkstead on a private server;
2. access it over trusted HTTPS;
3. create a secure account;
4. configure MFA/passkey authentication;
5. initialize an encrypted vault;
6. install the PWA;
7. create/edit entries offline;
8. attach media;
9. search locally;
10. browse timeline/calendar;
11. use prompts/favourites;
12. reconnect and synchronize automatically;
13. use a second browser replica;
14. converge offline changes safely;
15. export their journal;
16. create an operational backup;
17. restore a destroyed deployment from backup;
18. demonstrate that server-side persistence never contains journal plaintext.

---

## 73. Permanent release gates

A release MUST NOT be promoted as stable if any mandatory gate fails.

Required gates:

- backend tests;
- frontend tests;
- type checks;
- lint;
- pre-commit;
- Playwright/E2E;
- offline test;
- multi-replica convergence test;
- plaintext canary test;
- security-header test;
- dependency vulnerability policy;
- container scan policy;
- backup/restore test;
- database migration consistency;
- crypto compatibility fixtures;
- ASVS assessment status appropriate to the release;
- threat-model review.

---

## 74. Performance targets

### Typing

No network operation may block keystrokes.

### Local save

Ordinary local encrypted persistence SHOULD feel effectively immediate.

### Search

Common personal-journal searches SHOULD feel instantaneous on ordinary client hardware.

### Sync

Incremental sync SHOULD transfer changes rather than entire journals.

### Memory

The PWA SHOULD lazily load/decrypt data rather than materialize an entire lifetime of journal
entries into memory on startup.

---

## 75. Data-scale target

V1 SHOULD be tested against at least:

- 50,000 entries;
- substantial tag relationships;
- several gigabytes of encrypted attachments;
- multiple browser replicas.

Scale testing MUST occur before stable v1.

---

## 76. Browser-storage quota

Inkstead MUST detect storage quota failures and provide recoverable errors.

Before staging large local attachments, the client SHOULD estimate available storage.

A failed attachment write MUST NOT discard its associated journal entry.

---

## 77. Update and migration strategy

Updates MUST NOT silently make existing encrypted data unreadable.

Database and cryptographic migrations require:

- explicit versioning;
- backward-compatibility plan;
- fixtures from previous versions;
- migration tests;
- rollback consideration.

No cryptographic migration may ship without compatibility fixtures.

---

## 78. Cryptographic fixtures

Maintain deterministic compatibility fixtures, for example:

```text
tests/fixtures/crypto/v1/
  vault-envelope.json
  entry-envelope.json
  attachment-envelope.json
  expected-plaintext.json
  corrupted-examples/
```

Fixtures allow future clients, maintainers, and auditors to independently verify protocol behavior.

---

## 79. Open-source security transparency

Before stable v1, publish:

- threat model;
- crypto protocol description;
- security assumptions;
- known limitations;
- recovery behavior;
- persistence architecture;
- hardening guidance.

Marketing MUST NOT make claims stronger than the documented and tested architecture.

---

## 80. Independent review roadmap

### Before v1

- internal security review;
- public threat model;
- ASVS assessment;
- property/integration testing;
- community cryptographic review where possible.

### After v1

Seek:

- community security review;
- targeted penetration testing;
- cryptographic design audit;
- professional assessment when funding permits.

---

## 81. Explicit non-goals for v1

Do not implement in v1:

- public journals;
- anonymous share links;
- multi-user collaborative editing;
- social networking;
- public publishing;
- plugin system;
- extension marketplace;
- remote third-party embeds;
- advertisements;
- remote cloud AI;
- federation;
- native iOS application;
- native Android application;
- WebSocket synchronization unless a later proven requirement appears.

Each of these widens the security boundary and requires a separate threat-model update.

---

## 82. Future features requiring new threat modelling

Before implementing any of the following, revise the threat model and create an ADR:

- sharing;
- collaboration;
- plugins;
- browser extensions;
- public links;
- third-party integrations;
- remote AI;
- federation;
- external API clients;
- native applications.

---

## 83. Licensing

The repository is currently generated with GPLv3.

The final open-source license SHOULD be deliberately reviewed before external contributions become
substantial.

If retaining network-service copyleft is an important project goal, AGPL-3.0-or-later may be worth
formal evaluation. License changes MUST be handled deliberately and with contributor implications
understood.

---

# Architectural invariants

The following are non-negotiable unless superseded by an approved ADR.

1. Writing never requires network connectivity.
2. Losing the server never makes already initialized local journal data unavailable.
3. An expired Django session never blocks local writing.
4. Django/allauth authentication is not reimplemented.
5. Journal encryption cannot be disabled.
6. Journal plaintext is never persistently stored by Django.
7. Journal plaintext is never persistently stored in browser CacheStorage.
8. IndexedDB journal content is encrypted at rest.
9. Search occurs locally for encrypted journal content.
10. Search queries are not sent to Django.
11. Readable exports are generated client-side.
12. Server backups contain ciphertext, not readable journal entries.
13. The Service Worker never stores vault encryption keys.
14. Long-lived vault keys do not live in ordinary React state.
15. Cryptographic envelopes are always versioned.
16. Encrypted journal objects use authenticated encryption.
17. Remote executable JavaScript is prohibited in the core product.
18. External analytics/telemetry are prohibited by default.
19. User-provided rich content is hostile input until sanitized/parsed.
20. Security controls are active by default.
21. Security interrupts users only at genuine trust boundaries.
22. Critical security invariants are verified automatically.
23. Cookiecutter/Django functionality is extended rather than needlessly replaced.
24. Custom infrastructure is reserved for Inkstead-specific needs: local-first storage, encryption,
    key management, CRDT synchronization, and encrypted search/export boundaries.

---

# Definition of Done for architecture-changing work

An architecture-changing PR is not complete unless it includes, where applicable:

- updated specification or ADR;
- threat-model impact;
- data-migration impact;
- backward-compatibility impact;
- tests;
- security acceptance criteria;
- documentation;
- explicit confirmation that affected architectural invariants still hold.

---

# Reference projects and standards

These references informed the design and SHOULD be revisited when relevant architecture is
implemented:

- Cookiecutter Django: https://github.com/cookiecutter/cookiecutter-django
- Django: https://www.djangoproject.com/
- django-allauth: https://docs.allauth.org/
- Django REST Framework: https://www.django-rest-framework.org/
- OWASP ASVS: https://owasp.org/www-project-application-security-verification-standard/
- OWASP Cheat Sheet Series: https://cheatsheetseries.owasp.org/
- Standard Notes security: https://standardnotes.com/help/security
- Notesnook security/sync: https://notesnook.com/help/
- Day One features: https://dayoneapp.com/features/
- Joplin E2EE specification: https://joplinapp.org/help/dev/spec/e2ee/
- TriliumNext: https://github.com/TriliumNext/Trilium
- Memos: https://github.com/usememos/memos
- Automerge: https://automerge.org/
- Tiptap: https://tiptap.dev/
- Dexie: https://dexie.org/
- Workbox: https://developer.chrome.com/docs/workbox/
- libsodium: https://libsodium.gitbook.io/doc/
- DOMPurify: https://github.com/cure53/DOMPurify

---

# Final project standard

Inkstead follows this engineering principle:

> Prefer boring, maintained infrastructure for ordinary application concerns and reserve custom
> engineering for the small set of problems that make Inkstead unusual: local-first operation,
> encrypted client storage, cryptographic key management, conflict-safe synchronization, and
> privacy-preserving client-side search/export.

Security is not a later hardening phase.

Offline operation is not a fallback mode.

Self-hosting is not an afterthought.

They are core product properties from the first implementation task onward.
