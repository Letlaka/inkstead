# Focused Specification 03: Authentication, Sessions and Browser Security

**Status:** DRAFT  
**Parent:** `../master-product-and-implementation-spec.md`  
**Gate:** gate-01-security-auth  
**Requires:** gate-00-foundation PASS  
**Related gaps:** GAP-003, GAP-010, GAP-018, GAP-019, GAP-021, GAP-022, GAP-023, GAP-026, GAP-027, GAP-028, GAP-034, GAP-039, GAP-042, GAP-049

---

## 1. Purpose

This specification defines the server authentication boundary and browser security baseline before
encrypted journal content exists.

The objective is to use Django, Cookiecutter Django, and django-allauth correctly rather than
building parallel identity/security infrastructure.

---

## 2. Security boundary

Django authentication proves server account access.

It does not prove possession of journal decryption material.

The journal-vault unlock protocol is specified separately.

---

## 3. Default registration policy

Production default:

```text
public registration: OFF
username login: ON
email dependency: optional
social login: OFF
remote identity provider: OFF
```

An administrator may later enable supported account features explicitly.

The default installation MUST work without public Internet access.

---

## 4. First-account bootstrap

The default secure bootstrap path SHOULD be:

```bash
uv run python manage.py createsuperuser
```

or the equivalent command inside the production container.

Inkstead MUST NOT expose an unauthenticated first-run web route that silently becomes administrator
without a separate threat-model review.

Documentation should make bootstrap straightforward enough that self-hosting does not require
editing the database manually.

---

## 5. Email policy

The default private deployment SHOULD NOT require mandatory email verification.

If SMTP is absent:

- signup is already disabled;
- administrator-created accounts can log in;
- password recovery is an administrator/server operation;
- journal-vault recovery remains independent.

If SMTP is configured, allauth's built-in email management/recovery can be enabled without changing
the vault encryption model.

The current generated Brevo coupling must be reviewed. A provider-neutral default is preferred for
the final Inkstead distribution.

---

## 6. allauth usage

Use allauth for:

- account login/logout;
- password change/reset where configured;
- reauthentication;
- MFA;
- WebAuthn;
- passkeys;
- recovery codes;
- session tracking;
- account enumeration controls;
- authentication rate limiting.

Do not implement competing versions.

---

## 7. MFA policy

Supported:

- TOTP;
- recovery codes;
- WebAuthn credentials;
- passkey login.

Recommended UX:

- passkey/WebAuthn encouraged;
- TOTP fully supported;
- recovery codes mandatory when MFA is enabled unless allauth behavior provides an equivalent safe
  recovery route.

`MFA_RECOVERY_CODES_SHOW_ONCE` SHOULD be evaluated and likely enabled.

---

## 8. Encrypt allauth MFA secrets at rest

allauth documents `MFA_ADAPTER.encrypt()` and `decrypt()` hooks specifically for protecting TOTP
secrets.

Inkstead MUST use those hooks so TOTP seeds are not stored as database plaintext.

Requirements:

- use a maintained authenticated-encryption implementation;
- server-side MFA field encryption key is independent of journal keys;
- key material is not committed to Git;
- rotation supports at least current + previous server field key during migration;
- failure to decrypt a stored MFA secret fails closed and is auditable without logging the secret.

This protects account-security metadata even though the journal itself uses client-side encryption.

---

## 9. Django admin

Django admin MUST be forced through allauth's secure admin login.

Production:

```text
DJANGO_ADMIN_FORCE_ALLAUTH=True
```

or equivalent Cookiecutter integration.

Tests MUST demonstrate:

- unauthenticated admin access redirects through allauth;
- configured MFA is enforced;
- allauth authentication rate limiting applies;
- no alternate default AdminSite login bypass remains.

---

## 10. allauth proxy trust and rate limiting

allauth rate limiting depends on correct client-IP extraction.

Production path:

```text
client -> Traefik -> Django
```

The deployment MUST explicitly configure trusted proxy behavior.

Do not blindly trust arbitrary `X-Forwarded-For`.

Gate 01 must prove the chosen allauth trusted-proxy configuration using integration tests with:

- direct spoofed forwarded headers;
- one legitimate Traefik hop;
- repeated failed login attempts.

---

## 11. User sessions

Enable `allauth.usersessions`.

The user SHOULD be able to:

- view current authenticated sessions;
- see generic device/browser information;
- terminate other sessions.

Privacy:

- do not display precise geolocation inferred from IP;
- do not store journal metadata;
- activity tracking is optional and must be justified.

---

## 12. Session lifetime

Server session lifetime and local vault lifetime are independent.

A long-lived trusted server session MAY coexist with a locally locked vault.

A locally unlocked vault MAY temporarily coexist with an expired server session.

The UI must not confuse these.

---

## 13. Sensitive-action reauthentication

Use allauth/Django reauthentication for server-sensitive actions where appropriate:

- changing account password;
- changing MFA;
- deleting account;
- revoking sessions.

Vault-sensitive actions additionally require vault possession/confirmation as defined in the crypto
spec.

---

## 14. CSRF strategy for the PWA

The PWA uses same-origin session authentication.

CSRF protection remains enabled.

Because Cookiecutter sets the CSRF cookie HttpOnly, JavaScript SHOULD obtain a masked CSRF token
through a server-rendered bootstrap element or dedicated same-origin endpoint when online.

Requirements:

- do not store CSRF secrets in IndexedDB as durable authentication material;
- after a long offline period/session refresh, fetch a fresh usable token before mutation requests;
- a CSRF refresh failure pauses synchronization but does not block local writing;
- tests cover token/session expiry and recovery.

---

## 15. DRF authentication policy

Journal APIs use:

- SessionAuthentication;
- IsAuthenticated default;
- CSRF for unsafe methods.

Generated TokenAuthentication SHOULD be removed from Inkstead journal API settings unless an ADR
introduces a real external-client requirement.

If retained for some non-journal endpoint, the scope must be explicit.

---

## 16. API authorization baseline

Every API model query MUST scope objects to the authenticated owner.

Object identifiers are not authorization.

Tests MUST attempt horizontal access:

- another user's journal UUID;
- another user's replica UUID;
- another user's blob UUID;
- another user's change/document UUID.

Expected result is non-disclosure, normally 404 or 403 according to chosen API policy.

---

## 17. Browser threat boundary

Inkstead can protect against compromised server storage and network transport.

It cannot fully protect an unlocked journal from:

- malicious browser extensions with page access;
- OS-level malware;
- compromised browser binaries;
- screen capture;
- keylogging/input-method compromise;
- an actively malicious application server serving altered JavaScript.

These assumptions MUST be documented.

Do not market around them.

---

## 18. Security headers

Baseline SHOULD include:

- Content-Security-Policy;
- Strict-Transport-Security;
- X-Content-Type-Options: nosniff;
- Referrer-Policy: no-referrer or equally restrictive approved policy;
- frame denial via CSP `frame-ancestors` and/or X-Frame-Options;
- Permissions-Policy;
- Secure cookies;
- HttpOnly session/CSRF cookies;
- SameSite appropriate to same-origin operation;
- Cross-Origin-Opener-Policy where compatible.

Header policy must be tested against allauth/WebAuthn flows.

---

## 19. CSP and WebAssembly

The selected client stack is likely to use WebAssembly for Automerge and/or libsodium.

A CSP containing only:

```text
script-src 'self'
```

can block WebAssembly compilation.

Before Gate 01 passes, choose one of:

### Option A - permit WebAssembly narrowly

Use:

```text
script-src 'self' 'wasm-unsafe-eval'
```

while keeping ordinary `unsafe-eval` prohibited.

### Option B - select non-WASM production builds

Only if compatibility, performance, and security are demonstrated across target browsers.

The decision MUST be tested in Chromium, Firefox, and WebKit before approval.

---

## 20. CSP nonces and caching

If Django CSP nonces are used:

- nonce-bearing HTML MUST NOT be cached in a way that reuses an old nonce;
- PWA shell design must avoid requiring dynamic nonce-bearing inline script where possible;
- external self-hosted hashed bundles are preferred.

The journal Service Worker must not cache dynamic authentication pages.

---

## 21. Trusted Types

Trusted Types SHOULD be enabled where browser/library compatibility allows.

The project should define one minimal sanitization policy rather than many ad hoc policies.

If a target browser lacks enforcement support, CSP + React escaping + DOMPurify remain required.

---

## 22. Browser privacy leakage controls

Journal-sensitive text MUST NOT enter:

- document title;
- URL/query parameters;
- referrer;
- notification preview by default;
- server-side analytics (none by default);
- error-monitoring payloads;
- CSP report bodies generated from user-controlled URLs where avoidable.

Links in journal content must use safe protocols.

External links SHOULD use `rel="noopener noreferrer"`.

---

## 23. Account switching

When authenticated account identity changes:

- current vault UI is immediately hidden;
- current Crypto Worker terminates;
- decrypted search state is discarded;
- local database namespace switches only after lock;
- no pending sync request from the old account may be sent under the new session.

This is a mandatory cross-account isolation test.

---

## 24. Session/replica revocation semantics

Ending an allauth session invalidates server access.

Revoking an Inkstead replica invalidates future synchronization authorization/state.

Neither action can guarantee erasure of:

- plaintext already viewed;
- encrypted local database already copied;
- keys already extracted from a compromised running browser.

The UI MUST not call this remote wipe.

Key compromise remediation belongs to the crypto spec.

---

## 25. Privacy cover

When the page becomes hidden, Inkstead SHOULD immediately obscure/remove journal plaintext from the
visible DOM.

The unlock key may remain in the Crypto Worker until the configured lock timeout.

Events to evaluate:

- `visibilitychange`;
- `pagehide`;
- back-forward cache restoration;
- PWA app switching.

This behavior must be tested on target browsers because operating-system screenshots cannot be
fully controlled by web applications.

---

## 26. Cookie settings

Production MUST retain Cookiecutter secure cookie behavior.

Review at minimum:

- `SESSION_COOKIE_SECURE=True`;
- `SESSION_COOKIE_HTTPONLY=True`;
- `CSRF_COOKIE_SECURE=True`;
- `CSRF_COOKIE_HTTPONLY=True`;
- `SameSite` values;
- secure cookie names;
- domain/path scoping.

Do not broaden cookie domains unnecessarily.

---

## 27. Password hashing

Retain Cookiecutter's Argon2-first Django password hasher order unless current Django/allauth
guidance requires change.

Account password hashing and journal KDF are distinct systems.

---

## 28. Login rate-limit UX

Rate limiting should:

- remain active by default;
- return accessible 429 behavior;
- avoid leaking whether an account exists;
- never disable local vault access merely because server login is rate-limited.

---

## 29. Error handling

Authentication/security errors MUST NOT log:

- passwords;
- OTP values;
- recovery codes;
- WebAuthn secrets;
- session cookies;
- journal keys.

User-facing errors SHOULD identify the action required without revealing account existence to
unauthenticated attackers.

---

## 30. Gate 01 required tests

At minimum:

1. public registration disabled by default;
2. no-SMTP administrator-created account can log in;
3. admin forced through allauth;
4. TOTP works;
5. recovery codes work;
6. WebAuthn/passkey supported flow works on selected browser set;
7. TOTP seed is not plaintext in database;
8. forwarded-IP spoof does not bypass rate limiting;
9. valid proxy path records correct client IP behavior;
10. account enumeration behavior reviewed;
11. same-origin CSRF passes;
12. forged/missing CSRF fails;
13. expired session pauses sync only;
14. account switch locks old vault context;
15. security headers present;
16. CSP permits required WASM but not ordinary eval;
17. external script injection blocked;
18. cross-user object authorization test fails safely.

---

## 31. Evidence

Gate 01 evidence:

```text
docs/evidence/gate-01-security-auth.md
```

Must include:

- settings diff summary;
- allauth version;
- security-header capture;
- CSP test results;
- rate-limit proxy tests;
- MFA database-at-rest inspection;
- cross-user authorization tests;
- manual passkey/TOTP flows.

---

## 32. Blocking decisions before approval

- generic SMTP vs retaining optional Brevo dependency;
- exact allauth email-verification settings;
- exact session duration/trust-browser policy;
- exact server-side MFA-secret encryption implementation;
- trusted-proxy setting values for generated Traefik;
- CSP WASM strategy;
- Trusted Types enforcement strategy;
- exact CSRF refresh/bootstrap design;
- whether USERSESSIONS_TRACK_ACTIVITY is enabled.

---

## 33. Exit criteria

Gate 01 may pass only when:

- authentication works without Internet email;
- admin cannot bypass allauth;
- MFA secrets are encrypted at rest;
- rate limiting is proven behind the actual proxy chain;
- CSP/security headers are enforced and compatible with required runtime;
- CSRF/session recovery behavior is proven;
- cross-account and cross-object access tests pass;
- no unresolved P1 authentication/browser-security gap remains.
