# snaps-confdb-skill

An AI skill for GitHub Copilot that assists developers building snaps with [confdb](https://snapcraft.io/docs/confdb) support.

## What is Confdb?

Confdb (configuration database) is a snapd feature that allows snaps to share structured, validated configuration data. It follows a **custodian–observer** pattern:

- **Custodian snaps** write configuration to a central store, validated against a JSON schema
- **Observer snaps** read that configuration via controlled views
- Access is governed by interface permissions, providing isolation and type safety

Confdb is suited to multi-snap systems where a single source of truth for configuration is needed. For single-snap configuration, standard `snap config` remains the appropriate choice.

## What is This Skill For?

This skill provides GitHub Copilot with domain knowledge to help you:

- Design confdb schemas and views
- Implement custodian and observer snap hooks correctly
- Avoid common pitfalls such as deadlocks from confdb access inside connect hooks
- Write unit and integration tests for confdb-enabled snaps
- Migrate existing file-based or legacy configuration to confdb
- Troubleshoot connection, permission, and initialisation issues

The skill is backed by [CONFDB_MIGRATION_GUIDE.md](confdb/CONFDB_MIGRATION_GUIDE.md), which contains patterns and examples drawn from real-world snap migrations.

## Installation

### VS Code with GitHub Copilot

Skills are loaded from `~/.copilot/skills/`. Each skill lives in its own subdirectory containing a `SKILL.md` file.

**Option 1 — Clone directly into the skills directory:**

```bash
mkdir -p ~/.copilot/skills
git clone https://github.com/canonical/snaps-confdb-skill ~/.copilot/skills/confdb
```

**Option 2 — Copy the skill folder from an existing clone:**

```bash
mkdir -p ~/.copilot/skills
cp -r /path/to/snaps-confdb-skill/confdb ~/.copilot/skills/confdb
```

After installation the directory structure should look like:

```
~/.copilot/skills/
└── confdb/
    ├── SKILL.md
    └── CONFDB_MIGRATION_GUIDE.md
```

Restart VS Code (or reload the Copilot extension) to pick up the new skill.

### Other IDEs

Any IDE that supports GitHub Copilot and loads skills from `~/.copilot/skills/` follows the same steps above. Refer to your IDE's Copilot extension documentation for the exact skills directory path if it differs.

## Usage

Once installed, the skill is available automatically in Copilot Chat when working on snap projects. You can invoke it explicitly by referencing confdb topics, for example:

- *"Help me design a confdb schema for sharing network configuration between snaps"*
- *"Write a configure hook that validates and writes to confdb"*
- *"Why is my observer snap getting a deadlock when connecting the interface?"*
- *"How do I write integration tests for a confdb custodian snap?"*

## Contents

| File | Description |
|------|-------------|
| `confdb/SKILL.md` | Skill definition loaded by Copilot |
| `confdb/CONFDB_MIGRATION_GUIDE.md` | Full patterns, examples, and troubleshooting reference |

## Further Reading

- [Snapd Confdb Documentation](https://snapcraft.io/docs/confdb)
- [Snap Interface Management](https://snapcraft.io/docs/interface-management)
- [Supported Snap Hooks](https://snapcraft.io/docs/supported-snap-hooks)
