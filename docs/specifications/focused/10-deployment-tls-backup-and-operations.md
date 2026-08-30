# Focused Specification 10: Deployment, TLS, Backup and Operations

**Status:** DRAFT  
**Parent:** `../master-product-and-implementation-spec.md`  
**Gate:** gate-08-operations  
**Requires:** gate-07-data-lifecycle PASS  
**Related gaps:** GAP-025, GAP-028, GAP-031, GAP-045, GAP-049, GAP-050, GAP-054, GAP-056, GAP-057, GAP-061

---

## 1. Purpose

This specification defines how Inkstead is safely self-hosted, maintained, backed up, restored and
updated.

A privacy-preserving application is not secure if its operational deployment casually exposes
PostgreSQL, loses the TLS private key, or produces backups nobody has ever restored.

---

## 2. Default deployment posture

Inkstead's default production profile is private-network self-hosting.

Conceptual network:

```text
LAN / trusted VPN
       |
       v
Traefik :443
       |
       +------ Django/Gunicorn
       |
       +------ Nginx private ciphertext blobs

private Docker network:
  PostgreSQL
  Redis
```

Only the reverse proxy should bind a user-facing production port.

---

## 3. Same-network behavior

Inkstead does not attempt to identify a Wi-Fi SSID or subnet from browser code.

The PWA syncs whenever its configured origin is reachable.

If the default origin is reachable only on the home/private network, synchronization naturally
occurs only there.

VPN access may intentionally extend reachability without changing the application protocol.

---

## 4. HTTPS requirement

Production HTTP-only operation is unsupported.

HTTPS is required for:

- session confidentiality;
- Service Workers;
- WebAuthn/passkeys;
- secure cookies;
- protection against LAN script injection;
- expected PWA secure-context APIs.

Production startup/configuration SHOULD fail closed or emit a blocking health error when configured
in an unsafe HTTP mode.

---

## 5. Stable hostname

Recommended private default:

```text
inkstead.home.arpa
```

Administrators may use another stable name they control.

The chosen WebAuthn relying-party identity depends on hostname/origin. Hostname changes after
passkey registration require an explicit migration plan.

---

## 6. Certificate modes

Inkstead should support at least:

### Mode A - administrator-provided certificate

For users with existing DNS/PKI.

### Mode B - private CA

For fully private Internet-independent networks.

### Mode C - public ACME

Optional for administrators who intentionally use a public DNS name and Internet connectivity.

Public ACME is not a core dependency.

---

## 7. Private CA candidate

A maintained private-CA product such as Smallstep `step-ca` is a candidate, not yet a mandated
dependency.

Selection criteria:

- active maintenance;
- ACME support;
- clear client trust bootstrap;
- backup/recovery procedure;
- certificate rotation;
- no mandatory hosted service.

A simpler administrator-managed private CA may be sufficient for single-host deployments.

The focused review must choose the lowest-complexity secure default.

---

## 8. Certificate/CA recovery

Backups must identify which TLS material is required to restore the same trusted origin.

If the private CA is lost:

- clients may no longer trust renewed server certificates;
- WebAuthn origin/RP identity remains the hostname but transport trust must be restored.

Document CA root backup and private-key handling.

Do not bundle private CA keys into ordinary public release artifacts.

---

## 9. Traefik trust boundary

Django trusts only the headers expected from the configured Traefik hop.

Required review:

- `SECURE_PROXY_SSL_HEADER`;
- forwarded host behavior;
- client IP header handling;
- allauth trusted proxy count/header;
- direct backend exposure.

Clients must not be able to connect directly to Django on a LAN port and spoof proxy headers.

---

## 10. PostgreSQL exposure

Production PostgreSQL:

- no LAN host-port binding by default;
- Docker-private network only;
- unique strong application credential;
- least privilege consistent with Django migrations/runtime;
- backup/admin credentials separated if feasible.

Do not use PostgreSQL superuser for normal Django runtime.

---

## 11. Redis exposure

Production Redis:

- no LAN host-port binding;
- Docker-private network only;
- not treated as durable journal storage;
- no journal plaintext;
- authentication/config hardening reviewed.

A lost Redis cache must not lose journal state.

---

## 12. Container runtime hardening

Production SHOULD evaluate and apply:

- non-root runtime user;
- `no-new-privileges`;
- dropped capabilities;
- read-only root filesystem where compatible;
- explicit writable volumes;
- tmpfs for temporary files where appropriate;
- resource limits;
- health checks;
- restart policy;
- no Docker socket mounted into application containers.

Cookiecutter's generated non-root Django runtime MUST be preserved.

---

## 13. Image/version pinning

Production images must use explicit versions.

Mature releases SHOULD consider digest pinning for base/runtime images.

Dependency/image upgrade PRs should show:

- previous version;
- new version;
- security relevance;
- migration impact;
- test evidence.

---

## 14. Secrets

Secrets MUST NOT be:

- committed to Git;
- baked into Docker images;
- printed in CI logs;
- included in SBOM/public artifacts.

Required secrets may include:

- Django secret key;
- PostgreSQL credentials;
- server-side MFA-field encryption keys;
- optional SMTP credentials;
- TLS/CA private keys;
- backup repository credentials.

Cookiecutter env files may remain the initial mechanism if protected with strict permissions.

Mounted secret files/Docker-compatible secrets SHOULD be evaluated for production.

---

## 15. Server-side MFA field key backup

The key used to encrypt allauth MFA secrets is operationally critical.

Losing it may invalidate TOTP configuration.

It MUST:

- be backed up securely;
- be rotatable;
- remain separate from journal client-side keys;
- never be stored in PostgreSQL alongside the encrypted MFA fields.

---

## 16. Logging

Production logging:

- structured enough for diagnosis;
- no journal plaintext;
- no auth secrets;
- no cookies;
- no recovery material;
- no ciphertext bodies by default.

Reverse proxy access logs SHOULD avoid query strings containing sensitive data; Inkstead itself
should not put sensitive data in URLs.

Retention is administrator-configurable.

---

## 17. Health checks

Use maintained Django health-check support where it adds value.

Health should cover:

- Django process;
- database connectivity;
- Redis connectivity;
- blob storage writable/readable;
- migration status if safe.

Public unauthenticated health output is minimal.

Detailed diagnostics are admin-only or local.

---

## 18. Recovery Kit operational requirement

The Emergency Recovery Kit is user cryptographic recovery material, not merely another server
backup file.

Operations documentation MUST tell the administrator/user to keep valid portable recovery material
outside the Inkstead server.

A server backup and Recovery Kit solve different failures:

- server backup restores service/account/ciphertext state;
- Recovery Kit restores vault root-key access when server key envelopes are missing.

A stable release must test a scenario where the server key envelope is unavailable but the Recovery
Kit restores vault access.

---

## 19. Backup layers

Back up:

1. PostgreSQL;
2. encrypted blob storage;
3. critical server secrets/config;
4. TLS/private CA material where needed;
5. deployment manifests/config;
6. integrity metadata.

Journal plaintext is not needed for server backup.

---

## 20. Reuse Cookiecutter PostgreSQL tooling

Use the generated PostgreSQL backup/restore tooling as the foundation.

Do not replace it with another Django package merely for convenience.

Add orchestration around it only where Inkstead needs whole-system consistency.

---

## 21. Backup archive encryption

Even though journal records are client-encrypted, server backups contain:

- usernames/account metadata;
- hashed passwords;
- MFA encrypted fields;
- session/security metadata;
- network/timestamp metadata;
- server secrets if included.

Therefore backup repositories/archives themselves MUST be access-controlled and SHOULD be encrypted.

---

## 22. Backup-tool candidate

`restic` is a strong candidate for whole-system encrypted backup orchestration because it is open
source, encrypts backup repositories, supports local and remote targets, supports integrity
checking, and remains actively maintained.

It is not selected until focused review confirms operational fit.

Inkstead MUST NOT implement its own backup encryption format.

---

## 23. Backup consistency

PostgreSQL and blob backup must not produce a state where the database references a blob that the
backup can never restore.

Options to evaluate:

### Immutable blob + delayed deletion

- completed blob files are immutable;
- physical deletion is delayed beyond backup capture window;
- database dump taken before/while blobs remain present.

### Short backup coordination lock

- pause server-side sync finalization/purge;
- take DB dump;
- capture blob state;
- resume;
- clients continue writing locally and retry sync.

Choose the least disruptive design that passes destructive restore testing.

---

## 24. Backup schedule

Default recommendation to refine:

- daily backups;
- weekly retained generation;
- monthly retained generation.

Exact retention remains deployment policy.

Inkstead should expose backup health, not force one storage target.

---

## 25. Backup destination

Supported philosophy:

- local second disk;
- NAS;
- removable disk;
- administrator-controlled restic target;
- optional S3-compatible target.

No mandatory vendor cloud.

---

## 26. Backup key/password

Backup encryption credentials must not live only inside the backup being protected.

Provide recovery guidance.

A backup with an irretrievably lost repository password is not a recovery strategy.

---

## 27. Restore procedure

A supported restore must rebuild:

- database;
- encrypted blobs;
- server secrets;
- required TLS/configuration;
- application version/migrations as documented.

After restore:

- server starts;
- health passes;
- existing client detects restore generation/sequence state;
- clients reconcile without silent data loss;
- a current client may republish newer VaultPassphraseEnvelope/public crypto metadata missing from
  the backup;
- if no current client survives, the Emergency Recovery Kit must provide the independent recovery
  route.

The procedure must explicitly test a backup taken before a later vault-passphrase-envelope update.

---

## 28. Restore generation

Supported restore tooling SHOULD mark a new server restore generation.

This is used by sync clients to distinguish ordinary reconnect from a server restored to older
history.

If a restored database sequence is lower than an existing client's last-seen sequence, the client
must enter reconciliation even if restore generation was not updated.

---

## 29. Destructive restore test

Gate 08 requires an actual destructive test:

```text
create/sync data
  -> backup
  -> destroy database/blob volumes in test environment
  -> restore
  -> start server
  -> existing client reconnects
  -> new client recovers vault
  -> verify journal/attachments
```

A successful backup command alone does not pass.

---

## 30. Backup plaintext canary

The journal plaintext canary must be searched in:

- DB dump;
- blob backup;
- backup-tool metadata where inspectable.

It must not appear.

---

## 31. Updates

Server update workflow:

1. verify backup/restore health;
2. pull/build pinned release;
3. run pre-deployment checks;
4. apply database migrations;
5. verify health;
6. retain rollback plan consistent with data migrations.

Do not roll application code backward across irreversible DB/crypto migrations.

---

## 32. Security updates

Critical security patches may bypass normal cadence but not verification.

Emergency process still runs:

- targeted tests;
- security checks;
- migration compatibility;
- local write safety.

---

## 33. Django transaction boundaries

Cookiecutter enables PostgreSQL `ATOMIC_REQUESTS=True`.

That is useful for ordinary application writes but must be reviewed for endpoints that should not
hold an open database transaction across long work.

Candidates for explicit non-atomic/read-only handling include:

- large encrypted blob upload/download paths;
- health checks;
- streaming responses;
- other endpoints whose database authorization lookup can finish before long byte transfer.

Use Django's supported `non_atomic_requests`/explicit transaction boundaries only where measured
and safe.

Do not globally disable Cookiecutter's transaction behavior merely for optimization.

---

## 39. Resource limits / DoS

Production SHOULD set practical:

- request body limit;
- upload limit;
- container memory limit;
- process/file-descriptor limits where appropriate;
- database connection limits.

Self-hosted/private does not mean accidental infinite resource use is harmless.

---

## 34. Admin exposure

Admin endpoint:

- HTTPS only;
- allauth MFA;
- rate limited;
- private network by default.

Optional additional LAN/VPN allowlisting MAY be documented.

Do not rely on a hidden/random URL as the security control.

---

## 35. Gate 08 tests

Minimum:

1. only intended ports exposed;
2. direct PostgreSQL/Redis LAN connection unavailable;
3. Django runtime non-root;
4. no-new-privileges/capability policy tested where enabled;
5. HTTPS required;
6. WebAuthn works under production hostname;
7. spoofed proxy headers cannot bypass client-IP logic;
8. secrets absent from image/history/logs;
9. DB backup succeeds;
10. blob backup succeeds;
11. backup archive encrypted/access-controlled;
12. destructive restore succeeds;
13. existing client detects older restore state;
14. new client can recover vault from restored server;
15. plaintext canary absent from backup;
16. total-loss Recovery Kit path succeeds with server key envelope removed;
17. older-backup key metadata is repaired from current replica/Recovery Kit;
18. append-only sync history storage use is included in capacity/backup measurements;
19. long blob transfer does not hold an unnecessary ATOMIC_REQUESTS database transaction.

---

## 36. Gate 08 evidence

```text
docs/evidence/gate-08-operations.md
```

Include:

- port/network inventory;
- container user/capabilities;
- TLS certificate chain;
- secrets handling review;
- backup command/output;
- restore transcript;
- client reconciliation result;
- canary search.

---

## 37. Blocking decisions before approval

- private CA solution;
- backup repository tool;
- backup consistency strategy;
- default retention;
- secrets file vs Docker-secret support;
- database role split;
- read-only filesystem feasibility;
- resource limits;
- restore-generation implementation.

---

## 38. Exit criteria

Gate 08 passes only when:

- deployment is HTTPS and private-by-default;
- internal data services are not exposed;
- server secrets are managed/backupable;
- full-system backup is protected;
- destructive restore succeeds;
- restored clients reconcile safely;
- no unresolved P1 operational gap remains.
