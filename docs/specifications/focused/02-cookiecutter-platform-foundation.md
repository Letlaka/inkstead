# Focused Specification 02: Cookiecutter Platform Foundation

**Status:** DRAFT  
**Parent:** `../master-product-and-implementation-spec.md`  
**Gate:** gate-00-foundation  
**Related gaps:** GAP-024, GAP-039, GAP-040  
**Prerequisites:** Focused Specification 01 reviewed sufficiently to confirm product assumptions.

---

## 1. Purpose

This specification defines the exact generated Cookiecutter Django baseline that Inkstead will
preserve and the minimal foundation changes permitted before feature work.

The goal is to start from a known-good upstream project rather than immediately replacing the
framework's decisions.

---

## 2. Current repository facts

At the time this focused spec was created:

- default branch: `main`;
- generated baseline commit: `40ff76298f0b5048b0bc83dfce611262c6b3f737`;
- Django: 6.0.8;
- Python: 3.14;
- project package: `inkstead`;
- timezone: `Africa/Johannesburg`;
- DRF enabled;
- Webpack enabled;
- WhiteNoise enabled;
- Docker enabled;
- PostgreSQL configuration generated;
- Redis configuration generated;
- django-allauth MFA extras installed;
- `uv.lock` committed;
- frontend `package-lock.json` is not currently present;
- generated production frontend Docker stage uses `npm install`;
- generated DRF configuration currently includes both SessionAuthentication and TokenAuthentication;
- generated mail dependency currently includes Brevo through django-anymail.

These facts are baseline state, not necessarily final Inkstead policy.

---

## 3. Upstream preservation rule

Gate 00 MUST NOT redesign:

- custom user model;
- settings split;
- environment loading;
- PostgreSQL configuration;
- Docker project layout;
- Gunicorn launch;
- Traefik topology;
- WhiteNoise integration;
- DRF router/schema integration;
- allauth core account flow;
- pytest/Ruff/mypy/pre-commit foundations.

Changes are permitted only where Inkstead requirements require extension or secure configuration.

---

## 4. Upstream record

Create:

```text
docs/upstream-cookiecutter.md
```

It MUST record:

- generation date;
- source repository;
- source tag/release if known;
- source commit SHA if known;
- Cookiecutter responses;
- generated baseline commit;
- intentional deviations introduced after generation.

If the exact source SHA cannot be reconstructed, record that fact and establish the generated
baseline commit as the historical reference point.

---

## 5. Python dependency policy

Use `uv` and `uv.lock` as generated.

Requirements:

- production installs MUST use the lockfile;
- dependency updates occur in dedicated PRs or clearly scoped feature PRs;
- security updates may be expedited;
- direct dependencies MUST have a reason;
- abandoned packages MUST not become foundational;
- package compatibility with current Django/Python must be checked before addition.

---

## 6. Frontend dependency baseline

Before adding React or other frontend packages:

1. create and commit `package-lock.json`;
2. establish deterministic installation using `npm ci`;
3. update the production Docker client-builder to copy the lockfile and use `npm ci`;
4. verify the resulting bundle is reproducible enough for CI/build review;
5. preserve an explicit supported Node version.

The generated `npm install` behavior is acceptable as upstream baseline but SHOULD NOT remain the
production build strategy once Inkstead adds security-sensitive frontend dependencies.

---

## 7. Frontend package version policy

`package.json` MAY use semver ranges for developer ergonomics, but production builds MUST resolve
through committed `package-lock.json`.

No production build may generate a fresh dependency graph from ranges alone.

---

## 8. Frontend testing stack decision

Gate 00 must select a maintained testing baseline for:

- TypeScript type checking;
- component/unit tests;
- DOM behavior;
- security-sensitive utility tests.

Recommended evaluation set:

- TypeScript `tsc --noEmit`;
- ESLint;
- React Testing Library;
- Jest or Vitest after compatibility review with the Webpack project.

The final choice must minimize duplicated build tooling.

Playwright is reserved for browser/E2E flows and is specified separately.

---

## 9. DRF baseline

Generated DRF currently enables:

- SessionAuthentication;
- TokenAuthentication.

Inkstead's web PWA target intends same-origin session authentication.

Gate 00 records the generated state only.

Gate 01/Sync implementation MUST remove or disable TokenAuthentication for Inkstead journal APIs
unless an approved ADR introduces an external client requiring it.

---

## 10. Email baseline

The generated project currently includes Brevo-specific Anymail support.

Inkstead core operation must remain Internet-independent.

Gate 00 MUST record the dependency but SHOULD NOT silently replace it.

Focused Specification 03 decides:

- whether email is optional;
- whether the default becomes generic SMTP;
- whether allauth verification/recovery is disabled in default private deployment;
- how an administrator opts into SMTP later.

---

## 11. Static/frontend architecture

Keep Cookiecutter's Webpack integration as the starting frontend pipeline.

React/TypeScript must integrate through that pipeline rather than introducing Vite as a second
application build system without an ADR.

WhiteNoise remains responsible for static assets as generated.

---

## 12. Docker baseline

Preserve:

- multi-stage frontend/Python production build pattern;
- unprivileged Django runtime user;
- Compose separation of services;
- private PostgreSQL/Redis service networking;
- production Traefik entry point.

Gate 00 MUST verify what ports are actually exposed by generated Compose before security claims are
made.

---

## 13. Database baseline

PostgreSQL remains the only supported server database for v1.

Do not introduce SQLite as a server fallback.

SQLite/IndexedDB discussions apply to the client/offline layer only.

---

## 14. Redis baseline

Redis remains available for:

- Django cache;
- allauth rate limiting;
- transient operational state.

Gate 00 MUST verify no production Redis port is exposed to the LAN.

---

## 15. Baseline quality commands

Gate 00 evidence MUST include successful execution of the repository-appropriate equivalents of:

```bash
uv run python manage.py check
uv run pytest
uv run ruff check .
uv run mypy inkstead
uv run pre-commit run --all-files
```

If commands differ because of generated project configuration, evidence records the exact command.

---

## 16. Docker verification

Gate 00 MUST include:

- local Compose build;
- local startup;
- Django-to-PostgreSQL connectivity;
- Django-to-Redis connectivity;
- production image build;
- confirmation production Django runs non-root.

No feature implementation should be debugged on top of an unverified container baseline.

---

## 17. Gate-verifier implementation

Gate 00 MUST add the repository-local verifier described by Focused Specification 00.

Target:

```bash
uv run python scripts/verify_stage_gate.py --target gate-01-security-auth
```

The verifier itself must have tests.

It is process infrastructure, not optional documentation.

---

## 18. Initial evidence format

Create:

```text
docs/evidence/gate-00-foundation.md
```

Suggested metadata:

```yaml
gate: gate-00-foundation
status: PASS
commit: <sha>
verified_at: <ISO-8601>
```

Followed by commands and outputs summarized in Markdown.

---

## 19. Supply-chain baseline

Gate 00 SHOULD establish:

- Python lockfile integrity;
- frontend lockfile integrity;
- no committed secrets;
- dependency license review baseline;
- ability to produce an SBOM later.

Digest-pinning container base images may be introduced in operations hardening rather than blocking
the first foundation gate, but image tags MUST be explicit.

---

## 20. No feature creep in Gate 00

Gate 00 MUST NOT implement:

- vault encryption;
- journal models;
- sync;
- editor;
- attachments;
- search.

It establishes a trustworthy platform only.

---

## 21. Blocking decisions before approval

- exact frontend unit test runner;
- exact Node lockfile/install workflow;
- how to record the unknown/known Cookiecutter source SHA;
- whether generated GitHub Actions remain enabled, are supplemented by self-hosted CI, or both;
- whether Brevo remains temporarily until Gate 01.

---

## 22. Exit criteria

Gate 00 passes only when:

- baseline is documented;
- repository checks pass;
- Docker baseline passes;
- package-lock exists;
- deterministic frontend install is defined;
- gate-verifier exists and is tested;
- upstream tracking document exists;
- no application feature work has contaminated baseline verification.
