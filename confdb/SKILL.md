---
name: confdb
description: >
  Guides migration and implementation of snapd ConfDB for custodian and observer
  snaps, including schema and view design, naming compliance, hook behavior,
  rollout strategy, assertion import/signing workflow, interface connection
  sequencing, testing patterns, and troubleshooting of connection and
  permission failures across multi-snap systems.
  WHEN: migrate snap configuration to confdb, design a confdb schema, define admin and state views, check confdb naming conventions, review snapcraft.yaml plugs for confdb, implement configure hook writes, implement change-view hooks, implement observe-view hooks, avoid connect hook deadlocks, troubleshoot confdb interface connection failures, debug no custodian snap connected errors, debug view not found errors, validate storage path version prefixes, sign and import confdb-schema assertions, write unit and integration tests for confdb hooks, plan hybrid backward-compatible content-slot rollout, review confdb migration pull requests.
license: Apache-2.0
metadata:
  author: Canonical
  version: "1.0.0"
  summary: Guidance for adopting and operating ConfDB in snaps with migration, naming, hooks, testing, and troubleshooting support.
  tags:
    - snaps
    - snapd
    - confdb
    - configuration
    - migration
---

Use the documents in this skill to help developers adopt confdb in their snaps.

## Primary References

- [Migration Playbook](./CONFDB_MIGRATION_GUIDE.md) — step-by-step migration workflow, hybrid/backward-compatibility rollout, and links to all reference docs
- [Naming Conventions](./CONFDB_NAMING_CONVENTIONS.md) — rules and compliance checklist for schema names, view names, plug names, and storage paths

## Detailed Reference Docs

- [Concepts](./CONFDB_CONCEPTS.md) — what confdb is, custodian/observer pattern, connection order
- [Getting Started](./CONFDB_GETTING_STARTED.md) — enabling the feature, signing keys, schema import
- [Schema Design](./CONFDB_SCHEMA_DESIGN.md) — schema structure, views, defaults files, versioning/evolution
- [Hook Patterns](./CONFDB_HOOKS.md) — configure hook, connect hook, reading config, pitfalls with code examples
- [Testing](./CONFDB_TESTING.md) — unit tests, integration tests, snapctl-wrapper helper
- [Troubleshooting](./CONFDB_TROUBLESHOOTING.md) — diagnosing errors, debugging commands, best practices