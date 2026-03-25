---
name: release-ephemeral
description: Release/cleanup an ephemeral namespace. Releases the stored namespace from reserve-ephemeral or accepts explicit namespace argument. Cleans up bonfire reservation and local state.
user-invocable: true
argument-hint: [namespace]
allowed-tools:
  - Bash
  - AskUserQuestion
---

# /release-ephemeral — Release Ephemeral Namespace

Releases an ephemeral OpenShift namespace reservation and cleans up local state.

**Arguments**: `$ARGUMENTS` (optional: `<namespace>`)

---

## Overview

Cleans up ephemeral namespace:
1. Determines namespace (from args, env var, or file)
2. Optionally shows current resources
3. Confirms release (if interactive)
4. Releases bonfire reservation
5. Cleans up local state files

**Default**: Uses namespace from `/reserve-ephemeral`

---

## Phase 1: Determine Namespace

```bash
ARGS="$ARGUMENTS"

# Get namespace from arguments or stored location
if [ -n "$ARGS" ]; then
    # Namespace provided as argument
    NAMESPACE=$(echo "$ARGS" | xargs)
else
    # Try environment variable
    NAMESPACE="$EPHEMERAL_NAMESPACE"

    # Try file if env var not set
    if [ -z "$NAMESPACE" ] && [ -f ~/.claude/temp/current-ephemeral-namespace ]; then
        NAMESPACE=$(cat ~/.claude/temp/current-ephemeral-namespace)
    fi
fi

if [ -z "$NAMESPACE" ]; then
    echo "❌ No namespace to release"
    echo ""
    echo "Usage:"
    echo "  /release-ephemeral [namespace]"
    echo ""
    echo "Or reserve a namespace first:"
    echo "  /reserve-ephemeral"
    exit 1
fi

# Validate namespace format
if [[ ! "$NAMESPACE" =~ ^ephemeral- ]]; then
    echo "❌ Invalid namespace format: $NAMESPACE"
    echo ""
    echo "Namespace should start with 'ephemeral-'"
    exit 1
fi

echo "Namespace to release: $NAMESPACE"
echo ""
```

---

## Phase 2: Validate Prerequisites

```bash
# Check bonfire CLI
if ! command -v bonfire &>/dev/null; then
    echo "❌ bonfire CLI not found"
    echo ""
    echo "Install with: pip install crc-bonfire"
    exit 1
fi

# Check if namespace exists in bonfire
NAMESPACE_INFO=$(bonfire namespace list 2>&1 | grep "^$NAMESPACE")

if [ -z "$NAMESPACE_INFO" ]; then
    echo "⚠️  Namespace '$NAMESPACE' not found in bonfire reservations"
    echo ""
    echo "It may have already been released or expired."
    echo ""

    # Still clean up local state
    echo "Cleaning up local state..."
    rm -f ~/.claude/temp/current-ephemeral-namespace
    unset EPHEMERAL_NAMESPACE
    echo "✅ Local state cleaned"
    exit 0
fi

echo "Namespace found in bonfire:"
echo "$NAMESPACE_INFO"
echo ""
```

---

## Phase 3: Show Current Resources (Optional)

```bash
# Check if we have access to the namespace
if oc whoami &>/dev/null && oc get project "$NAMESPACE" &>/dev/null 2>&1; then
    echo "Current resources in namespace:"
    echo "────────────────────────────────────────────────"
    oc get all -n "$NAMESPACE" 2>/dev/null | head -20
    echo "────────────────────────────────────────────────"
    echo ""
else
    echo "ℹ️  Unable to check namespace resources (not logged in or no access)"
    echo ""
fi
```

---

## Phase 4: Confirm Release

Use `AskUserQuestion` for interactive confirmation:

```
Release namespace $NAMESPACE?

This will:
- Delete all resources in the namespace
- Release the bonfire reservation
- Clean up local state ($EPHEMERAL_NAMESPACE)

Continue? (yes/no)
```

If user confirms, proceed. Otherwise exit.

For non-interactive mode (future enhancement), could add `--force` flag.

---

## Phase 5: Release Namespace

```bash
echo "================================================"
echo "Releasing namespace..."
echo "================================================"
echo ""

# Release bonfire reservation
RELEASE_OUTPUT=$(bonfire namespace release "$NAMESPACE" --force 2>&1)
RELEASE_EXIT=$?

if [ $RELEASE_EXIT -ne 0 ]; then
    echo "❌ Failed to release namespace"
    echo ""
    echo "Error output:"
    echo "$RELEASE_OUTPUT"
    echo ""
    echo "Troubleshooting:"
    echo "  - Check namespace status: bonfire namespace describe $NAMESPACE"
    echo "  - Try manual release: bonfire namespace release $NAMESPACE"
    echo "  - List all namespaces: bonfire namespace list"
    exit 1
fi

echo "$RELEASE_OUTPUT"
echo ""

echo "✅ Bonfire reservation released"
echo ""
```

---

## Phase 6: Clean Up Local State

```bash
echo "================================================"
echo "Cleaning up local state..."
echo "================================================"
echo ""

# Remove stored namespace file
if [ -f ~/.claude/temp/current-ephemeral-namespace ]; then
    STORED_NAMESPACE=$(cat ~/.claude/temp/current-ephemeral-namespace)
    if [ "$STORED_NAMESPACE" = "$NAMESPACE" ]; then
        rm -f ~/.claude/temp/current-ephemeral-namespace
        echo "✅ Removed stored namespace file"
    fi
fi

# Unset environment variable (note: only affects subshell)
if [ "$EPHEMERAL_NAMESPACE" = "$NAMESPACE" ]; then
    unset EPHEMERAL_NAMESPACE
    echo "✅ Unset EPHEMERAL_NAMESPACE variable"
fi

echo ""

echo "================================================"
echo "✅ NAMESPACE RELEASED"
echo "================================================"
echo ""

echo "Namespace '$NAMESPACE' has been released."
echo ""
echo "To reserve a new namespace:"
echo "  /reserve-ephemeral"
echo ""
```

---

## Usage Examples

### Release Stored Namespace

```bash
# After /reserve-ephemeral
/release-ephemeral
```

### Release Specific Namespace

```bash
/release-ephemeral ephemeral-abc123
```

### Release and Confirm

```bash
/release-ephemeral
# Responds to confirmation prompt
```

---

## Error Handling

### No Namespace Found

**Error**: No namespace to release

**Response**:
```
❌ No namespace to release

Usage:
  /release-ephemeral [namespace]

Or reserve a namespace first:
  /reserve-ephemeral
```

### Namespace Not Found in Bonfire

**Warning**: Namespace not in bonfire reservations

**Response**:
```
⚠️  Namespace 'ephemeral-abc123' not found in bonfire reservations

It may have already been released or expired.

Cleaning up local state...
✅ Local state cleaned
```

### Release Failed

**Error**: bonfire command fails

**Response**:
```
❌ Failed to release namespace

Error output:
[error details]

Troubleshooting:
  - Check namespace status: bonfire namespace describe ephemeral-abc123
  - Try manual release: bonfire namespace release ephemeral-abc123
  - List all namespaces: bonfire namespace list
```

---

## Notes

- Safe to run multiple times (idempotent)
- Cleans up local state even if namespace already released
- Shows current resources before release (if accessible)
- Confirms before releasing (prevents accidents)
- Only unsets environment variable in skill's subshell (won't affect parent shell)
- File state (`~/.claude/temp/current-ephemeral-namespace`) is fully cleaned
