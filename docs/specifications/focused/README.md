# Inkstead Focused Specifications

The master specification remains at:

```text
docs/specifications/master-product-and-implementation-spec.md
```

It is intentionally not modified during this decomposition.

These focused documents exist so Inkstead can review and prove one architectural layer at a time.

## Review sequence

1. [00 - Stage-Gate Framework](00-stage-gate-framework.md)
2. [Gap Register](gap-register.md)
3. [01 - Product Scope and User Flows](01-product-scope-and-user-flows.md)
4. [01A - Security, Privacy Threat Model and Data Classification](01a-security-privacy-threat-model.md)
5. [02 - Cookiecutter Platform Foundation](02-cookiecutter-platform-foundation.md)
6. [03 - Authentication, Sessions and Browser Security](03-authentication-session-and-browser-security.md)
7. [04 - PWA, Local Storage, Offline and Update Lifecycle](04-pwa-local-storage-offline-and-update-lifecycle.md)
8. [05 - Cryptography, Key Management and Recovery](05-cryptography-key-management-and-recovery.md)
9. [06 - Journal Domain, Editor and Metadata](06-journal-domain-editor-and-metadata.md)
10. [07 - CRDT, Sync, Replica and Compaction Protocol](07-sync-crdt-replica-and-compaction-protocol.md)
11. [08 - Attachments, Media and File Security](08-attachments-media-and-file-security.md)
12. [09 - Search, Export, Import, Deletion and Data Lifecycle](09-search-export-import-deletion-and-data-lifecycle.md)
13. [10 - Deployment, TLS, Backup and Operations](10-deployment-tls-backup-and-operations.md)
14. [11 - Testing, Security Assurance and Release Gates](11-testing-security-assurance-and-release-gates.md)

Machine-readable prerequisites:

```text
stage-gates.yaml
```

## Important process rule

All focused specs start as DRAFT.

A later spec may be discussed while an earlier one is being refined, but later implementation MUST
NOT begin until its prerequisite gate is VERIFIED.

Each gate produces evidence under:

```text
docs/evidence/
```

This is how Inkstead prevents assumptions from silently propagating into later architecture.
