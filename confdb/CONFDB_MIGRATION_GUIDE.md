# Confdb Migration Guide

This guide provides patterns and best practices for migrating snaps to use snapd's confdb (configuration database) feature. It is based on real-world implementation experience and is not specific to any particular snap.

## Table of Contents

1. [Overview](#overview)
2. [Prerequisites](#prerequisites)
3. [Architecture Patterns](#architecture-patterns)
4. [Migration Steps](#migration-steps)
5. [Hook Patterns](#hook-patterns)
6. [Testing Strategies](#testing-strategies)
7. [Common Pitfalls](#common-pitfalls)
8. [Troubleshooting](#troubleshooting)

## Overview

### What is Confdb?

Confdb is snapd's experimental configuration database feature that allows snaps to share configuration data through a structured schema. It follows a custodian-observer pattern where:

- **Custodian snaps** write configuration to confdb
- **Observer snaps** read configuration from confdb
- Configuration is validated against a JSON schema
- Access is controlled through interface permissions

### When to Use Confdb

✅ **Good Use Cases:**
- Multiple snaps need to share configuration
- Configuration needs validation and type safety
- You want a single source of truth for config
- Observer snaps should react to configuration changes

❌ **Not Ideal For:**
- Single-snap configuration (use snap config)
- Unstructured or highly dynamic data
- Snaps that need to work without confdb support

### Core Principle: Confdb-First Architecture

**Confdb is the single source of truth for configuration.** Files are used only for:
1. **Initialization defaults** (in `defaults/` folder)
2. **Live runtime configuration** (updated from confdb when needed)

## Prerequisites

### Enable Confdb Feature

```bash
sudo snap set system experimental.confdb=true
```

### Create Signing Key

```bash
# Generate a key for signing schemas
snap create-key my-confdb-key

# List keys
snap keys
```

### Install Schema Tooling

You'll need tools to:
- Convert YAML schemas to JSON
- Sign schemas with your key
- Import signed assertions

Example tools setup:
```bash
# Create schema signing script
cat > sign-schema.sh << 'EOF'
#!/bin/sh -e
SCHEMA_FILE=$1
KEY_NAME=$2

# Convert YAML to JSON
./yaml-to-sign-json.py "$SCHEMA_FILE" > schema.json

# Sign the schema
snap sign -k "$KEY_NAME" < schema.json > schema.assert

# Import the assertion
sudo snap ack schema.assert

echo "Schema signed and acknowledged with revision:"
snap known confdb-schema | grep revision | head -1
EOF

chmod +x sign-schema.sh
```

## Architecture Patterns

### Custodian Pattern

The custodian snap manages configuration and writes to confdb.

**snapcraft.yaml:**
```yaml
name: my-config-snap
base: core22
version: '1.0'

plugs:
  my-config-control:
    interface: confdb
    account: YOUR_ACCOUNT_ID
    view: my-app/control-config

hooks:
  configure:
    plugs: [my-config-control]
```

### Observer Pattern

Observer snaps read configuration from confdb.

**snapcraft.yaml:**
```yaml
name: my-app-snap
base: core22
version: '1.0'

plugs:
  my-config-observe:
    interface: confdb
    account: YOUR_ACCOUNT_ID
    view: my-app/observe-config

hooks:
  connect-plug-my-config-observe:
    plugs: [my-config-observe]
```

### Hybrid Pattern (Backward Compatibility)

If you need to support both confdb and legacy file-based configuration:

**snapcraft.yaml:**
```yaml
slots:
  configuration-read:
    interface: content
    read:
      - $SNAP_COMMON/configuration

hooks:
  configure:
    plugs: [my-config-control]
```

**configure hook must:**
1. Validate all config values are set
2. Write to confdb
3. Update files in `$SNAP_COMMON/configuration` for content slot consumers

## Migration Steps

### Step 1: Design Your Schema

**schema.yaml:**
```yaml
storage:
  schema:
    my-app:
      values: map
      keys: str
      schema:
        config-field-1:
          type: string
        config-field-2:
          type: int
        config-field-3:
          type: string

views:
  control-config:
    rules:
      -
        request: my-app
        storage: my-app
        access: read-write
  
  observe-config:
    rules:
      -
        request: my-app
        storage: my-app
        access: read
```

**Key Decisions:**
- What fields do you need?
- What are their types? (string, int, bool, etc.)
- Who needs read-write vs read-only access?
- Do you need computed values? (compute at runtime in code)

### Step 2: Create Defaults File

Create a defaults file with placeholders for user-provided values.

**snap/local/configuration/defaults/config.yaml:**
```yaml
config-field-1: "field-1-placeholder"
config-field-2: 0
config-field-3: "field-3-placeholder"
```

### Step 3: Implement Custodian Configure Hook

The configure hook validates configuration and writes to confdb.

**snap/hooks/configure (Python):**
```python
#!/usr/bin/env python3

import os
import subprocess
import sys
import logging
import logging.handlers

# Setup logging
logger = logging.getLogger("configure-hook")
logger.setLevel(logging.INFO)

try:
    syslog_handler = logging.handlers.SysLogHandler(address='/dev/log')
    syslog_formatter = logging.Formatter('my-snap.configure: %(message)s')
    syslog_handler.setFormatter(syslog_formatter)
    logger.addHandler(syslog_handler)
except Exception:
    pass

# Defaults file path
defaults_file = os.path.join(
    os.environ.get("SNAP", ""),
    "etc/configuration/defaults/config.yaml"
)

if not os.path.exists(defaults_file):
    logger.error(f"Defaults file not found: {defaults_file}")
    sys.exit(1)

try:
    # Get snap configuration values
    def get_snap_config(key):
        """Get snap configuration value."""
        try:
            result = subprocess.run(
                ["snapctl", "get", key],
                capture_output=True,
                text=True,
                check=False
            )
            if result.returncode == 0 and result.stdout.strip():
                return result.stdout.strip()
        except Exception:
            pass
        return None
    
    # Get all required values
    field1 = get_snap_config("field-1")
    field2 = get_snap_config("field-2")
    field3 = get_snap_config("field-3")
    
    # Validate all required values are set
    missing = []
    if not field1:
        missing.append("field-1")
    if not field2:
        missing.append("field-2")
    if not field3:
        missing.append("field-3")
    
    if missing:
        logger.info(f"Configuration incomplete. Missing: {', '.join(missing)}")
        logger.info("Skipping confdb update until all values are set")
        sys.exit(0)
    
    logger.info("All configuration values present, updating confdb")
    
    # Read and update defaults
    with open(defaults_file, 'r') as f:
        config_data = f.read()
    
    config_data = config_data.replace("field-1-placeholder", field1)
    config_data = config_data.replace("field-3-placeholder", field3)
    # field-2 is numeric, handle appropriately
    
    # Write to confdb
    result = subprocess.run(
        ["snapctl", "set", "--view", ":my-config-control", f"data={config_data}"],
        capture_output=True,
        text=True
    )
    
    if result.returncode != 0:
        logger.warning(f"Failed to write to confdb: {result.stderr}")
    else:
        logger.info("Configuration updated in confdb")
    
    # If you have a content slot, update files here
    # subprocess.run([f"{os.environ.get('SNAP')}/usr/bin/update-config-files.sh"])

except Exception as e:
    logger.error(f"Error in configure hook: {e}")

# Always exit successfully
sys.exit(0)
```

### Step 4: Implement Observer Connect Hook

**snap/hooks/connect-plug-my-config-observe:**
```bash
#!/bin/sh -e

# Log connection
logger -t "my-app-snap.connect-plug" "Config interface connected"

# Trigger your app to reload config from confdb
if snapctl services my-app-snap.my-service | grep -q active; then
    snapctl restart my-app-snap.my-service
fi
```

### Step 5: Implement Configuration Reading in Observer

**Python example (confdb_utils.py):**
```python
import subprocess
import yaml
import logging

logger = logging.getLogger(__name__)

def get_config_from_confdb():
    """Read configuration from confdb view."""
    try:
        result = subprocess.run(
            ["snapctl", "get", "--view", ":my-config-observe", "data"],
            capture_output=True,
            text=True,
            check=True
        )
        
        config_data = result.stdout.strip()
        if not config_data:
            logger.warning("No configuration data found in confdb")
            return None
        
        # Parse YAML config
        config = yaml.safe_load(config_data)
        
        # Check for placeholders
        for key, value in config.items():
            if isinstance(value, str) and "placeholder" in value.lower():
                logger.warning(f"Configuration contains placeholder: {key}={value}")
                return None
        
        return config
        
    except subprocess.CalledProcessError as e:
        logger.error(f"Failed to read from confdb: {e}")
        return None
    except Exception as e:
        logger.error(f"Error parsing confdb config: {e}")
        return None

def get_config_value(key, view=":my-config-observe"):
    """Get a specific config value from confdb."""
    try:
        result = subprocess.run(
            ["snapctl", "get", "--view", view, key],
            capture_output=True,
            text=True,
            check=True
        )
        return result.stdout.strip()
    except subprocess.CalledProcessError:
        return None
```

### Step 6: Update File-Based Config (If Needed)

If you maintain a content slot for backward compatibility:

**snap/local/update-config-files.sh:**
```bash
#!/bin/sh -e

CONFIG_DIR="$SNAP_COMMON/configuration"

if [ ! -d "$CONFIG_DIR" ]; then
    logger -t my-snap "ERROR: Configuration directory not found"
    exit 1
fi

# Function to replace placeholder in all config files
replace_in_files() {
    local placeholder="$1"
    local value="$2"
    
    if [ -z "$value" ]; then
        return
    fi
    
    find "$CONFIG_DIR" -type f -exec sed -i "s#${placeholder}#${value}#g" {} \;
}

# Get values from snap config
FIELD1=$(snapctl get field-1 2>/dev/null || echo "")
FIELD3=$(snapctl get field-3 2>/dev/null || echo "")

# Update files
if [ -n "$FIELD1" ]; then
    replace_in_files "field-1-placeholder" "$FIELD1"
    logger -t my-snap "Updated field-1 in configuration files"
fi

if [ -n "$FIELD3" ]; then
    replace_in_files "field-3-placeholder" "$FIELD3"
    logger -t my-snap "Updated field-3 in configuration files"
fi

logger -t my-snap "Configuration files updated successfully"
```

## Hook Patterns

### Critical Hook Rules

1. **NEVER access confdb in connect hooks** - This causes deadlock
2. **Use change-view hooks for initialization** - Triggered on first write
3. **Configure hooks validate before writing** - Prevent partial configuration
4. **Always exit successfully from hooks** - Use logging for errors

### Hook Types and Usage

| Hook Type | When It Runs | Can Access Confdb? | Use Case |
|-----------|--------------|-------------------|----------|
| `install` | Snap installation | ❌ No | Copy files to $SNAP_COMMON |
| `configure` | `snap set` commands | ✅ Yes | Validate and write to confdb |
| `connect-plug-X` | Interface connection | ❌ No | Log connection, restart services |
| `change-view-X` | Confdb write | ✅ Yes | Initialize confdb, validate values |

### Connect Hook Pattern

```bash
#!/bin/sh -e

# Minimal logging only
logger -t "my-snap.connect-plug" "Interface connected"

# Restart service to pick up new config
if snapctl services my-snap.service | grep -q active; then
    snapctl restart my-snap.service
fi

# Always succeed
exit 0
```

### Configure Hook Pattern (with validation)

```python
#!/usr/bin/env python3

# 1. Get all required config values from snapctl
# 2. Check if ALL values are set (list missing ones)
# 3. If incomplete, log and exit successfully
# 4. If complete, update confdb
# 5. Update files if maintaining content slot
# 6. Always exit successfully
```

## Testing Strategies

### Unit Tests

**Pytest Configuration (pytest.ini):**
```ini
[pytest]
markers =
    unit: Unit tests that don't require external dependencies
    integration: Integration tests that require external resources
```

**Mock File I/O:**
```python
from unittest.mock import patch, mock_open
import pytest

@pytest.mark.unit
@patch("pathlib.Path.is_file", return_value=True)
@patch("builtins.open", new_callable=mock_open)
def test_read_config(mock_file, mock_is_file):
    # Mock yaml.safe_load separately
    with patch("yaml.safe_load", return_value={"key": "value"}):
        result = read_config_file("test.yaml")
        assert result["key"] == "value"
```

### Integration Tests

**Gate with Environment Variable:**
```python
import os
import pytest

@pytest.mark.integration
def test_confdb_integration():
    """Integration test requiring confdb setup."""
    
    # Skip unless explicitly requested
    if not os.environ.get("RUN_INTEGRATION_TESTS"):
        pytest.skip(
            "Skipping integration test. Set RUN_INTEGRATION_TESTS=1"
        )
    
    # Test with real snap installation
    # ...
```

**Snapctl Wrapper for Testing:**

Create an app to execute snapctl from tests:

```yaml
# snapcraft.yaml
apps:
  snapctl-wrapper:
    command: snap/local/snapctl-wrapper.sh
    plugs: [my-config-control]  # CRITICAL: Must include confdb plug
```

```bash
# snap/local/snapctl-wrapper.sh
#!/bin/sh
exec snapctl "$@"
```

**Usage from tests:**
```python
import subprocess
import json

def set_confdb_value(snap_name, view, key, value):
    """Set a confdb value from test."""
    subprocess.run([
        "snap", "run", f"{snap_name}.snapctl-wrapper",
        "set", "--view", view, f"{key}={value}"
    ], check=True)

def get_confdb_values(snap_name, view):
    """Get all confdb values from test."""
    result = subprocess.run([
        "snap", "run", f"{snap_name}.snapctl-wrapper",
        "get", "--view", view, "-d"
    ], capture_output=True, text=True, check=True)
    return json.loads(result.stdout)
```

## Common Pitfalls

### 1. Accessing Confdb in Connect Hooks

**❌ Wrong:**
```bash
# snap/hooks/connect-plug-my-config
snapctl set --view :my-config key=value  # DEADLOCK!
```

**✅ Correct:**
```bash
# snap/hooks/connect-plug-my-config
logger -t my-snap "Interface connected"
# Let configure hook or change-view hook handle confdb
```

### 2. Not Validating Complete Configuration

**❌ Wrong:**
```python
# Writing to confdb even if some values are missing
field1 = get_snap_config("field-1") # might be None
# Write to confdb anyway - ships broken config!
```

**✅ Correct:**
```python
# Check ALL required values first
missing = []
if not field1:
    missing.append("field-1")
if not field2:
    missing.append("field-2")

if missing:
    logger.info(f"Missing values: {', '.join(missing)}")
    sys.exit(0)  # Exit successfully, don't write to confdb
```

### 3. Forgetting to Update Content Slot Files

If you have a content slot, configure hook must update both:

**❌ Wrong:**
```python
# Only update confdb
subprocess.run(["snapctl", "set", "--view", ":control", f"data={config}"])
# Forgot to update files - content slot consumers see stale data!
```

**✅ Correct:**
```python
# Update confdb
subprocess.run(["snapctl", "set", "--view", ":control", f"data={config}"])

# Update files for content slot
subprocess.run([f"{os.environ['SNAP']}/usr/bin/update-files.sh"])
```

### 4. Not Checking for Placeholders

**❌ Wrong:**
```python
config = get_config_from_confdb()
url = config["server-url"]  # Might be "server-url-placeholder"!
requests.get(url)  # Fails with invalid URL
```

**✅ Correct:**
```python
config = get_config_from_confdb()
if config is None:
    return None

url = config.get("server-url")
if "placeholder" in url.lower():
    logger.warning("Configuration not ready (contains placeholders)")
    return None

return url
```

### 5. Using File Logging Instead of Journalctl

**❌ Wrong:**
```python
# Writing to log file
with open(f"{SNAP_COMMON}/hooks.log", "a") as f:
    f.write(f"Hook ran at {datetime.now()}\n")
```

**✅ Correct:**
```python
import logging
import logging.handlers

logger = logging.getLogger("my-snap")
logger.setLevel(logging.INFO)

try:
    syslog = logging.handlers.SysLogHandler(address='/dev/log')
    syslog.setFormatter(logging.Formatter('my-snap: %(message)s'))
    logger.addHandler(syslog)
except Exception:
    pass

logger.info("Hook ran successfully")
```

### 6. Custodian Not Connecting Both Plugs

**❌ Wrong:**
```bash
# Only connect control plug
sudo snap connect my-config:my-config-control
# Observer cannot connect yet!
```

**✅ Correct:**
```bash
# Connect both control and observe plugs
sudo snap connect my-config:my-config-control
sudo snap connect my-config:my-config-observe
# Now observers can connect
```

## Troubleshooting

### Check Confdb Status

```bash
# View confdb schema
snap known confdb-schema

# View confdb values (from custodian)
sudo snap run my-config-snap.snapctl-wrapper get --view :my-config-control -d | jq

# View confdb values (from observer)
sudo snap run my-app-snap.snapctl-wrapper get --view :my-config-observe -d | jq
```

### Check Hook Execution

```bash
# View hook logs
sudo journalctl -u snapd | grep "my-snap"

# View specific hook
sudo journalctl -u snapd | grep "my-snap.configure"
```

### Check Interface Connections

```bash
# List all connections for a snap
snap connections my-snap

# Check if confdb interface is connected
snap connections my-snap | grep confdb
```

### Debug Connection Issues

If observer cannot connect:

1. Check custodian is installed and connected:
   ```bash
   snap connections custodian-snap | grep confdb
   ```

2. Check custodian has both control AND observe plugs connected:
   ```bash
   # Both should show "connected"
   snap connections custodian-snap | grep my-config-control
   snap connections custodian-snap | grep my-config-observe
   ```

3. Check confdb has been initialized (custodian must write first):
   ```bash
   sudo snap run custodian-snap.snapctl-wrapper get --view :control -d
   # Should return config data, not empty
   ```

4. Check observer snap has correct plug configuration:
   ```bash
   snap info my-app-snap | grep plugs
   ```

### Common Error Messages

| Error | Cause | Solution |
|-------|-------|----------|
| "no custodian snap connected" | Observer connecting before custodian | Connect custodian first, ensure it has observe plug |
| "view not found" | Typo in view name or schema not imported | Check view name matches schema, verify schema imported |
| "permission denied" | Observer trying to write to read-only view | Use control view for writes, observe for reads |
| "deadlock detected" | Accessing confdb in connect hook | Move confdb access to configure or change-view hook |

## Best Practices Summary

✅ **DO:**
- Validate ALL required config values before writing to confdb
- Use journalctl logging (logger command) instead of file logging
- Exit successfully from hooks even on errors (log instead)
- Gate integration tests with environment variables
- Use mock_open without read_data in unit tests
- Keep content slots synchronized if maintaining backward compatibility
- Document placeholder patterns in defaults files

❌ **DON'T:**
- Access confdb in connect hooks (causes deadlock)
- Write partial configuration to confdb (validate completeness first)
- Hardcode placeholder values (read from defaults)
- Use file-based logging in hooks
- Run integration tests in normal unit test suite
- Forget to connect custodian's observe plug
- Store computed values in confdb (compute at runtime)

## Additional Resources

- [Snapd Confdb Documentation](https://snapcraft.io/docs/confdb)
- [Interface Reference](https://snapcraft.io/docs/interface-management)
- [Hook Reference](https://snapcraft.io/docs/supported-snap-hooks)
- Schema examples: `proxy-confdb-demo` snap

---

**Note:** This guide is based on real implementation experience migrating production snaps to confdb. Patterns may evolve as the confdb feature matures.
