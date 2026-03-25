---
name: reserve-ephemeral
description: Reserve an ephemeral OpenShift namespace using bonfire. Provisions from minimal pool with 4h duration and stores namespace for use with deploy-to-ephemeral.
user-invocable: true
argument-hint: [--pool <pool>] [--duration <time>]
allowed-tools:
  - Bash
---

# /reserve-ephemeral — Reserve Ephemeral Namespace

Provisions an ephemeral OpenShift namespace using bonfire and stores it for deployment workflows.

**Arguments**: `$ARGUMENTS` (optional: `--pool <pool>` `--duration <time>`)

---

## Overview

Reserves an ephemeral namespace from bonfire and stores it in `$EPHEMERAL_NAMESPACE` environment variable for use by `/deploy-to-ephemeral`.

**Default Configuration**:
- Pool: `minimal`
- Duration: `4h`
- Timeout: `600` seconds

---

## Workflow

### Parse Arguments

```bash
# Default values
POOL="minimal"
DURATION="4h"
TIMEOUT="600"

# Parse optional arguments
ARGS="$ARGUMENTS"

if [[ "$ARGS" == *"--pool"* ]]; then
    POOL=$(echo "$ARGS" | grep -oP '(?<=--pool\s)\w+')
fi

if [[ "$ARGS" == *"--duration"* ]]; then
    DURATION=$(echo "$ARGS" | grep -oP '(?<=--duration\s)\S+')
fi
```

### Validate Prerequisites

```bash
echo "================================================"
echo "Validating Prerequisites..."
echo "================================================"
echo ""

# Check 1: OpenShift authentication
if ! oc whoami &>/dev/null; then
    echo "❌ Not logged into OpenShift cluster"
    echo ""
    echo "Please run: oc login <cluster-url>"
    exit 1
fi

echo "✅ OpenShift: Logged in as $(oc whoami)"

# Check 2: bonfire CLI
if ! command -v bonfire &>/dev/null; then
    echo "❌ bonfire CLI not found"
    echo ""
    echo "Install with: pip install crc-bonfire"
    exit 1
fi

echo "✅ bonfire: installed"
echo ""
```

### Reserve Namespace

```bash
echo "================================================"
echo "Reserving ephemeral namespace..."
echo "================================================"
echo ""

echo "Configuration:"
echo "  Pool: $POOL"
echo "  Duration: $DURATION"
echo "  Timeout: ${TIMEOUT}s"
echo ""

# Reserve namespace with --force to skip interactive prompt
BONFIRE_OUTPUT=$(bonfire namespace reserve --pool "$POOL" --timeout "$TIMEOUT" -d "$DURATION" --force 2>&1)
BONFIRE_EXIT=$?

if [ $BONFIRE_EXIT -ne 0 ]; then
    echo "❌ Failed to reserve namespace"
    echo ""
    echo "Error output:"
    echo "$BONFIRE_OUTPUT"
    echo ""
    echo "Troubleshooting:"
    echo "  - Check available pools: bonfire pool list"
    echo "  - Check available namespaces: bonfire namespace list"
    echo "  - Verify bonfire configuration: bonfire config get-current"
    exit 1
fi

# Parse namespace name from output
# bonfire outputs the namespace name as the last line
NAMESPACE=$(echo "$BONFIRE_OUTPUT" | grep -E '^ephemeral-' | tail -1)

if [ -z "$NAMESPACE" ]; then
    echo "❌ Could not parse namespace from bonfire output"
    echo ""
    echo "Output was:"
    echo "$BONFIRE_OUTPUT"
    exit 1
fi

echo "$BONFIRE_OUTPUT"
echo ""
```

### Store Namespace

```bash
echo "================================================"
echo "✅ NAMESPACE RESERVED"
echo "================================================"
echo ""

echo "Namespace: $NAMESPACE"
echo "Pool: $POOL"
echo "Duration: $DURATION"
echo ""

# Get detailed namespace info
echo "Fetching namespace details..."
echo ""
bonfire namespace describe "$NAMESPACE" 2>&1
echo ""

# Store namespace in environment variable
export EPHEMERAL_NAMESPACE="$NAMESPACE"

# Also write to file for persistence across sessions
mkdir -p ~/.claude/temp
echo "$NAMESPACE" > ~/.claude/temp/current-ephemeral-namespace

echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
echo "Namespace stored in:"
echo "  - Environment variable: \$EPHEMERAL_NAMESPACE"
echo "  - File: ~/.claude/temp/current-ephemeral-namespace"
echo ""
echo "Use with deployment:"
echo "  /deploy-to-ephemeral <repo-url>"
echo ""
echo "Manage namespace:"
echo "  - Extend: bonfire namespace extend $NAMESPACE"
echo "  - Release: /release-ephemeral"
echo "  - Describe: bonfire namespace describe $NAMESPACE"
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
echo ""
```

---

## Usage Examples

### Default (minimal pool, 4h)

```bash
/reserve-ephemeral
```

### Custom pool

```bash
/reserve-ephemeral --pool default
```

### Custom duration

```bash
/reserve-ephemeral --duration 8h
```

### Both custom

```bash
/reserve-ephemeral --pool default --duration 2h
```

---

## Error Handling

### Not logged into cluster

**Error**: OpenShift authentication check fails

**Response**:
```
❌ Not logged into OpenShift cluster

Please run: oc login <cluster-url>
```

### bonfire not found

**Error**: bonfire command not available

**Response**:
```
❌ bonfire CLI not found

Install with: pip install crc-bonfire
```

### No namespaces available

**Error**: bonfire reservation fails

**Response**:
```
❌ Failed to reserve namespace

Troubleshooting:
  - Check available pools: bonfire pool list
  - Check available namespaces: bonfire namespace list
  - Verify bonfire configuration: bonfire config get-current
```

---

## Notes

- Uses `--force` flag to skip interactive prompts (required for automation)
- Namespace stored in both env var and file for flexibility
- File persists across Claude sessions but not shell sessions
- Environment variable works within current shell session
- Default 4h duration balances testing time vs resource conservation
