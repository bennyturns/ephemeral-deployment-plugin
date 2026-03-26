---
name: deploy-to-ephemeral
description: Deploy a GitHub project to ephemeral namespace. Clones repo, parses README for deployment commands, executes deployment. Uses namespace from reserve-ephemeral or --namespace flag. Starts debug session on failure.
user-invocable: true
argument-hint: <github-repo-url> [--namespace <ns>]
allowed-tools:
  - Bash
  - Read
  - AskUserQuestion
---

# /deploy-to-ephemeral — Deploy to Ephemeral Namespace

Deploys a GitHub project to an ephemeral OpenShift namespace with intelligent README parsing.

**Arguments**: `$ARGUMENTS` (required: `<repo-url>`, optional: `--namespace <ns>`)

---

## Overview

Automates deployment workflow:
1. Validates namespace availability
2. Clones GitHub repository
3. Parses README for deployment commands
4. Executes deployment sequentially
5. Shows success summary or initiates debug session

**Requires**: Namespace from `/reserve-ephemeral` or `--namespace` flag

---

## Phase 1: Parse Arguments

### Extract Repository URL and Namespace

```bash
ARGS="$ARGUMENTS"

# Extract repo URL (first non-flag argument)
REPO_URL=$(echo "$ARGS" | sed 's/--namespace[[:space:]]*[^[:space:]]*//' | xargs | awk '{print $1}')

# Extract namespace if provided
if [[ "$ARGS" == *"--namespace"* ]]; then
    NAMESPACE=$(echo "$ARGS" | grep -oP '(?<=--namespace\s)\S+')
else
    # Try environment variable
    NAMESPACE="$EPHEMERAL_NAMESPACE"

    # Try file if env var not set
    if [ -z "$NAMESPACE" ] && [ -f ~/.claude/temp/current-ephemeral-namespace ]; then
        NAMESPACE=$(cat ~/.claude/temp/current-ephemeral-namespace)
    fi
fi

# Validate we have both repo and namespace
if [ -z "$REPO_URL" ]; then
    echo "Usage: /deploy-to-ephemeral <github-repo-url> [--namespace <ns>]"
    echo ""
    echo "Example:"
    echo "  /deploy-to-ephemeral https://github.com/my-org/my-app"
    echo ""
    echo "First reserve a namespace:"
    echo "  /reserve-ephemeral"
    exit 1
fi

if [ -z "$NAMESPACE" ]; then
    echo "❌ No namespace available"
    echo ""
    echo "Please reserve a namespace first:"
    echo "  /reserve-ephemeral"
    echo ""
    echo "Or specify one:"
    echo "  /deploy-to-ephemeral <repo-url> --namespace <namespace>"
    exit 1
fi

# Validate URL format
if [[ ! "$REPO_URL" =~ ^(https?://|git@|file://) ]]; then
    echo "❌ Invalid repository URL format"
    echo ""
    echo "Expected formats:"
    echo "  - https://github.com/org/repo"
    echo "  - git@github.com:org/repo.git"
    echo "  - file:///path/to/repo"
    exit 1
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

# Check 2: Namespace exists and is accessible
if ! oc get project "$NAMESPACE" &>/dev/null; then
    echo "❌ Namespace '$NAMESPACE' not found or not accessible"
    echo ""
    echo "Check namespace:"
    echo "  bonfire namespace list"
    echo ""
    echo "Reserve new namespace:"
    echo "  /reserve-ephemeral"
    exit 1
fi

echo "✅ Namespace: $NAMESPACE (accessible)"

# Check 3: git
if ! command -v git &>/dev/null; then
    echo "❌ git not found"
    echo ""
    echo "Install git package for your system"
    exit 1
fi

echo "✅ git: $(git --version)"

# Check 4: helm (optional - warning only)
if ! command -v helm &>/dev/null; then
    echo "⚠️  helm not found - helm deployments will fail"
else
    echo "✅ helm: installed"
fi

echo ""
echo "All prerequisites validated ✓"
echo ""
```

---

## Phase 2: Clone Repository

```bash
echo "================================================"
echo "Cloning repository..."
echo "================================================"
echo ""

# Create temporary directory
CLONE_DIR=$(mktemp -d)

echo "Repository: $REPO_URL"
echo "Clone directory: $CLONE_DIR"
echo ""

# Clone repository
GIT_OUTPUT=$(git clone "$REPO_URL" "$CLONE_DIR" 2>&1)
GIT_EXIT=$?

if [ $GIT_EXIT -ne 0 ]; then
    echo "❌ Failed to clone repository"
    echo ""
    echo "Error output:"
    echo "$GIT_OUTPUT"
    echo ""

    # Provide specific troubleshooting
    if echo "$GIT_OUTPUT" | grep -qi "authentication failed\|permission denied"; then
        echo "Authentication issue detected:"
        if [[ "$REPO_URL" =~ ^git@ ]]; then
            echo "  - SSH URL detected. Check SSH key: ssh -T git@github.com"
        else
            echo "  - HTTPS URL detected. Check credentials or use SSH URL"
        fi
    elif echo "$GIT_OUTPUT" | grep -qi "repository not found"; then
        echo "Repository not found:"
        echo "  - Verify the URL is correct"
        echo "  - Check you have access to the repository"
    else
        echo "Network or other issue:"
        echo "  - Check VPN/network connection"
        echo "  - Verify git configuration"
    fi

    exit 1
fi

echo "✅ Repository cloned successfully"
echo ""

# Change to clone directory
cd "$CLONE_DIR"
```

---

## Phase 3: Parse README for Deployment Commands

```bash
echo "================================================"
echo "Parsing README for deployment instructions..."
echo "================================================"
echo ""

# Find README file (case-insensitive)
README_FILE=""
for name in README.md readme.md README Readme.md; do
    if [ -f "$name" ]; then
        README_FILE="$name"
        break
    fi
done

if [ -z "$README_FILE" ]; then
    echo "⚠️  No README.md found"
    echo ""
    echo "Searching for common deployment directories..."

    # Fallback: Look for deployment directories
    DEPLOY_DIRS=()
    for dir in deploy k8s manifests openshift .openshift helm charts; do
        if [ -d "$dir" ]; then
            DEPLOY_DIRS+=("$dir")
        fi
    done

    if [ ${#DEPLOY_DIRS[@]} -gt 0 ]; then
        echo "Found deployment directories: ${DEPLOY_DIRS[*]}"
        echo ""
        echo "Suggested commands:"
        for dir in "${DEPLOY_DIRS[@]}"; do
            echo "  oc apply -f $dir/"
        done
        echo ""

        # Use AskUserQuestion to get commands
        echo "No README found. Please provide deployment commands manually."
        exit 1
    else
        # Check for static web content even without README
        HTML_FILES=$(find . -maxdepth 2 -name "*.html" -type f 2>/dev/null)
        if [ -n "$HTML_FILES" ]; then
            echo "No README or deploy directories, but found static HTML content."
            echo "Will deploy as static site using nginx s2i."
            echo ""
            # Skip README parsing, go straight to static site deployment
            README_FILE=""
        else
            echo "No deployment directories or static content found."
            echo "Please provide deployment commands manually."
            exit 1
        fi
    fi
fi

if [ -n "$README_FILE" ]; then
    echo "Found: $README_FILE"
    echo ""
fi
```

If a README was found, use the `Read` tool to read it:

```
Read the README file at $CLONE_DIR/$README_FILE
```

Then extract deployment commands:

```bash
# Create temp file for commands
COMMANDS_FILE=$(mktemp)

if [ -n "$README_FILE" ]; then
    # Extract deployment commands from README
    # Look for oc apply, helm install, kubectl apply commands

    echo "Extracting deployment commands..."
    echo ""

    # Search for deployment commands in code blocks
    # Pattern 1: oc apply -f <file>
    # Pattern 2: oc process -f <file> | oc apply -f -
    # Pattern 3: helm install <name> <chart>
    # Pattern 4: kubectl apply -f <file>

    grep -E '^\s*(oc|kubectl)\s+apply\s+-f' "$README_FILE" | sed 's/^[[:space:]]*//;s/[[:space:]]*$//' >> "$COMMANDS_FILE"
    grep -E '^\s*oc\s+process.*\|.*oc\s+apply' "$README_FILE" | sed 's/^[[:space:]]*//;s/[[:space:]]*$//' >> "$COMMANDS_FILE"
    grep -E '^\s*helm\s+(install|upgrade)' "$README_FILE" | sed 's/^[[:space:]]*//;s/[[:space:]]*$//' >> "$COMMANDS_FILE"
fi

if [ ! -s "$COMMANDS_FILE" ]; then
    if [ -n "$README_FILE" ]; then
        echo "⚠️  No deployment commands found in README"
        echo ""
    fi

    # Fall back to directory search
    DEPLOY_DIRS=()
    for dir in deploy k8s manifests openshift .openshift; do
        if [ -d "$dir" ]; then
            DEPLOY_DIRS+=("$dir")
        fi
    done

    if [ ${#DEPLOY_DIRS[@]} -gt 0 ]; then
        echo "Found deployment directories: ${DEPLOY_DIRS[*]}"
        echo ""
        echo "Auto-generating commands:"
        for dir in "${DEPLOY_DIRS[@]}"; do
            echo "oc apply -f $dir/" >> "$COMMANDS_FILE"
        done
    else
        # Check for static web content (HTML files)
        HTML_FILES=$(find . -maxdepth 2 -name "*.html" -type f 2>/dev/null)
        if [ -n "$HTML_FILES" ]; then
            echo "Detected static web content (HTML files):"
            echo "$HTML_FILES" | head -10
            echo ""

            # Derive app name from repo directory name
            APP_NAME=$(basename "$REPO_URL" .git | tr '[:upper:]' '[:lower:]' | sed 's/[^a-z0-9-]/-/g')
            echo "Deploying as static site using nginx s2i..."
            echo "App name: $APP_NAME"
            echo ""

            echo "oc new-app registry.access.redhat.com/ubi9/nginx-122~. --name=$APP_NAME" >> "$COMMANDS_FILE"
            echo "sleep 5" >> "$COMMANDS_FILE"
            echo "oc wait --for=condition=available deployment/$APP_NAME --timeout=120s" >> "$COMMANDS_FILE"
            echo "oc create route edge $APP_NAME --service=$APP_NAME --port=8080-tcp" >> "$COMMANDS_FILE"
        else
            echo "No deployment commands, directories, or static content found."
            echo "Please provide deployment commands manually."
            exit 1
        fi
    fi
fi

echo "Found deployment command(s):"
cat "$COMMANDS_FILE" | while read cmd; do
    echo "  → $cmd"
done
echo ""
```

---

## Phase 4: Execute Deployment

```bash
echo "================================================"
echo "Executing deployment..."
echo "================================================"
echo ""

# Set namespace context
echo "Setting namespace context: $NAMESPACE"
oc project "$NAMESPACE"

if [ $? -ne 0 ]; then
    echo "❌ Failed to set namespace context"
    exit 1
fi

echo ""

# Execute each command sequentially
DEPLOY_SUCCESS=true
FAILED_COMMAND=""
FAILED_OUTPUT=""

while IFS= read -r cmd; do
    echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
    echo "Running: $cmd"
    echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
    echo ""

    # Execute command and capture output
    CMD_OUTPUT=$(eval "$cmd" 2>&1)
    CMD_EXIT=$?

    echo "$CMD_OUTPUT"
    echo ""

    if [ $CMD_EXIT -ne 0 ]; then
        # STOP on first failure
        DEPLOY_SUCCESS=false
        FAILED_COMMAND="$cmd"
        FAILED_OUTPUT="$CMD_OUTPUT"
        break
    else
        echo "✅ Command completed successfully"
        echo ""
    fi
done < "$COMMANDS_FILE"
```

---

## Phase 5: Handle Results

### On Success

```bash
if [ "$DEPLOY_SUCCESS" = true ]; then
    echo "================================================"
    echo "✅ DEPLOYMENT SUCCESSFUL"
    echo "================================================"
    echo ""

    echo "Namespace: $NAMESPACE"
    echo "Repository: $CLONE_DIR"
    echo ""

    echo "Deployed resources:"
    oc get all -n "$NAMESPACE" 2>/dev/null | head -30

    echo ""
    echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
    echo "Next steps:"
    echo "  - View pods: oc get pods -n $NAMESPACE"
    echo "  - View services: oc get svc -n $NAMESPACE"
    echo "  - View routes: oc get routes -n $NAMESPACE"
    echo "  - Check logs: oc logs -l app=<name> -n $NAMESPACE"
    echo ""
    echo "Namespace management:"
    echo "  - Extend: bonfire namespace extend $NAMESPACE"
    echo "  - Release: /release-ephemeral"
    echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
    echo ""

    echo "Clone directory: $CLONE_DIR"
    echo "(Kept for reference. Delete manually when done.)"
    echo ""
fi
```

### On Failure (Debug Session)

```bash
if [ "$DEPLOY_SUCCESS" = false ]; then
    echo "================================================"
    echo "❌ DEPLOYMENT FAILED"
    echo "================================================"
    echo ""

    echo "Namespace: $NAMESPACE"
    echo "Failed Command: $FAILED_COMMAND"
    echo ""

    echo "Error Output:"
    echo "────────────────────────────────────────────────"
    echo "$FAILED_OUTPUT"
    echo "────────────────────────────────────────────────"
    echo ""

    echo "Repository: $CLONE_DIR"
    echo ""

    echo "================================================"
    echo "Debug Session Initiated"
    echo "================================================"
    echo ""
    echo "The namespace has been preserved for debugging."
    echo ""

    echo "Suggested debugging steps:"
    echo ""
    echo "1. Check pod status:"
    echo "   oc get pods -n $NAMESPACE"
    echo ""
    echo "2. View pod logs:"
    echo "   oc logs <pod-name> -n $NAMESPACE"
    echo "   oc logs -l app=<name> -n $NAMESPACE"
    echo ""
    echo "3. Describe resources:"
    echo "   oc describe pod <pod-name> -n $NAMESPACE"
    echo ""
    echo "4. Check events:"
    echo "   oc get events -n $NAMESPACE --sort-by='.lastTimestamp'"
    echo ""
    echo "5. Check resource definitions:"
    echo "   oc get <resource-type> <name> -n $NAMESPACE -o yaml"
    echo ""

    echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
    echo "Manual investigation:"
    echo "  - Switch to namespace: oc project $NAMESPACE"
    echo "  - Navigate to repo: cd $CLONE_DIR"
    echo "  - Extend if needed: bonfire namespace extend $NAMESPACE"
    echo "  - Release when done: /release-ephemeral"
    echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
    echo ""

    # Use AskUserQuestion to offer interactive debugging
    exit 1
fi
```

Then use `AskUserQuestion`:

```
The deployment has failed. I've preserved the namespace and repository for debugging.

Would you like me to:
1. Run diagnostic commands (check pods, logs, events)?
2. Analyze the error and suggest fixes?
3. Help you investigate manually?
4. Retry deployment with modifications?

What would you like to do?
```

---

## Usage Examples

### Basic Deployment

```bash
# After /reserve-ephemeral
/deploy-to-ephemeral https://github.com/my-org/my-app
```

### Specific Namespace

```bash
/deploy-to-ephemeral https://github.com/my-org/my-app --namespace ephemeral-abc123
```

### Private Repository (SSH)

```bash
/deploy-to-ephemeral git@github.com:my-org/private-app.git
```

### Local Repository

```bash
/deploy-to-ephemeral file:///home/user/my-app
```

---

## Notes

- Requires namespace from `/reserve-ephemeral` or `--namespace` flag
- Stops on first command failure (prevents cascading errors)
- Preserves namespace and clone directory on failure for debugging
- All commands execute in repository directory (relative paths work)
- Multi-line command support (backslash continuation)
- Temporary directory kept on failure, suggest cleanup on success
