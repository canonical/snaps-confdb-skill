---
name: confdb
description: How to Migrate your snaps to using ConfDB. This skill provides detailed instructions and examples for migrating your configuration database, ensuring a smooth transition with minimal disruption.
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