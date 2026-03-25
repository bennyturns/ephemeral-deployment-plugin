# Ephemeral Deployment Plugin for Claude Code

Streamline your ephemeral OpenShift cluster workflow with intelligent automation. Reserve namespaces, deploy GitHub projects, and clean up resources—all with simple slash commands.

## Overview

This Claude Code plugin provides three focused skills for managing ephemeral OpenShift namespaces with bonfire:

1. **`/reserve-ephemeral`** - Provision ephemeral namespace
2. **`/deploy-to-ephemeral`** - Deploy GitHub projects with smart README parsing
3. **`/release-ephemeral`** - Clean up namespaces and local state

## Quick Start

```bash
# 1. Reserve namespace (once per session)
/reserve-ephemeral

# 2. Deploy your project (multiple times)
/deploy-to-ephemeral https://github.com/my-org/my-app

# 3. Clean up when done
/release-ephemeral
```

## Installation

### Prerequisites

- **Claude Code CLI** installed and configured
- **OpenShift CLI (`oc`)** - Must be logged into cluster
- **bonfire** - `pip install crc-bonfire` (v6.8.2+)
- **git** - For cloning repositories
- **Helm** (optional) - For Helm chart deployments

### Setup Steps

#### 1. Install bonfire

```bash
pip install crc-bonfire
```

Verify installation:
```bash
bonfire --version
```

#### 2. Login to OpenShift Cluster

```bash
oc login --token=<your-token> --server=https://api.crc-eph.r9lp.p1.openshiftapps.com:6443
```

**Get your token**:
- Visit: https://oauth-openshift.apps.crc-eph.r9lp.p1.openshiftapps.com/oauth/token/request
- Copy the login command with your token
- Paste into terminal

Verify login:
```bash
oc whoami
```

#### 3. Install Plugin

```bash
# Clone this repo into Claude Code plugins marketplace
cd ~/.claude/plugins/marketplaces
git clone https://github.com/bennyturns/ephemeral-deployment-plugin.git

# Or create symlink to your workspace (for development)
ln -s ~/Workspace/ephemeral-deployment-plugin ~/.claude/plugins/marketplaces/ephemeral-deployment
```

#### 4. Restart Claude Code

Restart Claude Code to load the plugin.

## Skills

### 1. `/reserve-ephemeral` - Reserve Namespace

Provisions an ephemeral namespace from bonfire and stores it for deployment workflows.

**Usage:**
```bash
/reserve-ephemeral [--pool <pool>] [--duration <time>]
```

**Examples:**
```bash
# Default (minimal pool, 4h)
/reserve-ephemeral

# Custom pool
/reserve-ephemeral --pool default

# Custom duration
/reserve-ephemeral --duration 8h

# Both custom
/reserve-ephemeral --pool default --duration 2h
```

**What it does:**
- ✅ Validates prerequisites (oc login, bonfire)
- ✅ Reserves namespace from specified pool (default: minimal)
- ✅ Stores namespace in `$EPHEMERAL_NAMESPACE` env var
- ✅ Stores namespace in `~/.claude/temp/current-ephemeral-namespace`
- ✅ Shows namespace details and console URL

**Output:**
```
================================================
✅ NAMESPACE RESERVED
================================================

Namespace: ephemeral-abc123
Pool: minimal
Duration: 4h

Namespace stored in:
  - Environment variable: $EPHEMERAL_NAMESPACE
  - File: ~/.claude/temp/current-ephemeral-namespace

Use with deployment:
  /deploy-to-ephemeral <repo-url>
```

---

### 2. `/deploy-to-ephemeral` - Deploy Project

Deploys a GitHub project to your ephemeral namespace with intelligent README parsing.

**Usage:**
```bash
/deploy-to-ephemeral <github-repo-url> [--namespace <ns>]
```

**Examples:**
```bash
# Use stored namespace
/deploy-to-ephemeral https://github.com/my-org/my-app

# Specific namespace
/deploy-to-ephemeral https://github.com/my-org/my-app --namespace ephemeral-xyz789

# Private repo (SSH)
/deploy-to-ephemeral git@github.com:my-org/private-app.git

# Local repo (testing)
/deploy-to-ephemeral file:///home/user/my-app
```

**What it does:**
- ✅ Validates namespace availability
- ✅ Clones GitHub repository
- ✅ Parses README for deployment commands
- ✅ Recognizes: `oc apply`, `helm install`, `kubectl apply`
- ✅ Executes commands sequentially
- ✅ Stops on first failure (prevents cascading errors)
- ✅ Shows deployment summary or starts debug session

**Recognized Deployment Commands:**
- `oc apply -f <file>`
- `oc process -f <template> | oc apply -f -`
- `helm install <name> <chart>`
- `helm upgrade --install <name> <chart>`
- `kubectl apply -f <file>`

**README Example:**
````markdown
## Deployment

To deploy this application to OpenShift:

```bash
oc apply -f deploy/database.yaml
oc apply -f deploy/application.yaml
oc apply -f deploy/service.yaml
```
````

**Success Output:**
```
================================================
✅ DEPLOYMENT SUCCESSFUL
================================================

Namespace: ephemeral-abc123
Repository: /tmp/tmp.xyz789

Deployed resources:
NAME                     READY   STATUS    RESTARTS   AGE
pod/myapp-7d4f8c9-x5z2w  1/1     Running   0          30s
...

Next steps:
  - View pods: oc get pods -n ephemeral-abc123
  - View routes: oc get routes -n ephemeral-abc123
```

**Failure Output (Debug Session):**
```
================================================
❌ DEPLOYMENT FAILED
================================================

Namespace: ephemeral-abc123
Failed Command: oc apply -f deploy/service.yaml

Error Output:
────────────────────────────────────────────────
Error from server (AlreadyExists): service "myapp" already exists
────────────────────────────────────────────────

Debug Session Initiated

Suggested debugging steps:
1. Check pod status: oc get pods -n ephemeral-abc123
2. View logs: oc logs <pod-name> -n ephemeral-abc123
3. Check events: oc get events -n ephemeral-abc123 --sort-by='.lastTimestamp'

What would you like to investigate?
```

---

### 3. `/release-ephemeral` - Cleanup Namespace

Releases bonfire reservation and cleans up local state.

**Usage:**
```bash
/release-ephemeral [namespace]
```

**Examples:**
```bash
# Release stored namespace
/release-ephemeral

# Release specific namespace
/release-ephemeral ephemeral-abc123
```

**What it does:**
- ✅ Shows current resources in namespace
- ✅ Confirms before releasing (interactive)
- ✅ Releases bonfire reservation
- ✅ Cleans up `$EPHEMERAL_NAMESPACE`
- ✅ Removes `~/.claude/temp/current-ephemeral-namespace`
- ✅ Idempotent (safe to run multiple times)

**Output:**
```
Current resources in namespace:
────────────────────────────────────────────────
pod/myapp-7d4f8c9-x5z2w  1/1  Running  0  5m
service/myapp            ClusterIP  ...
────────────────────────────────────────────────

Release namespace ephemeral-abc123?
Continue? (yes/no)

> yes

================================================
✅ NAMESPACE RELEASED
================================================

Namespace 'ephemeral-abc123' has been released.
```

## Workflow Examples

### Single Deployment Session

```bash
# Morning: Reserve namespace
/reserve-ephemeral

# Deploy and test your app
/deploy-to-ephemeral https://github.com/my-org/my-app

# Make code changes, push to GitHub

# Redeploy to same namespace (fast iteration)
/deploy-to-ephemeral https://github.com/my-org/my-app

# End of day: Clean up
/release-ephemeral
```

### Multiple Projects in One Namespace

```bash
# Reserve once
/reserve-ephemeral

# Deploy multiple services
/deploy-to-ephemeral https://github.com/my-org/frontend
/deploy-to-ephemeral https://github.com/my-org/backend
/deploy-to-ephemeral https://github.com/my-org/database

# Test integration
oc get all -n $EPHEMERAL_NAMESPACE

# Clean up all at once
/release-ephemeral
```

### Long-Running Test Session

```bash
# Reserve with longer duration
/reserve-ephemeral --duration 12h

# Deploy app
/deploy-to-ephemeral https://github.com/my-org/app

# Extend if needed (manual bonfire command)
bonfire namespace extend $EPHEMERAL_NAMESPACE -d 6h

# Continue testing...
```

## Configuration

### Default Values

- **Pool**: `minimal` (optimized for testing)
- **Duration**: `4h` (balances usability vs. resources)
- **Timeout**: `600s` (bonfire CLI requirement)

### Available Pools

Check available pools:
```bash
bonfire pool list
```

Common pools:
- `minimal` - Small resource allocation (default)
- `default` - Standard resources
- `prometheus` - With monitoring
- `ai-development` - AI/ML workloads

### Namespace Storage

Namespace is stored in two locations:

1. **Environment variable**: `$EPHEMERAL_NAMESPACE`
   - Works within current Claude Code session
   - Accessible in bash commands

2. **File**: `~/.claude/temp/current-ephemeral-namespace`
   - Persists across Claude Code sessions
   - Single namespace tracking

## README Format Requirements

For automatic deployment, your repository's README should include deployment commands in code blocks:

### ✅ Good Examples

**Example 1: Simple Commands**
````markdown
## Deployment

```bash
oc apply -f deploy/app.yaml
```
````

**Example 2: Multiple Steps**
````markdown
## Getting Started

Deploy to OpenShift:

```bash
oc apply -f deploy/database.yaml
oc apply -f deploy/app.yaml
oc apply -f deploy/service.yaml
```
````

**Example 3: Helm Chart**
````markdown
## Installation

```bash
helm install myapp ./charts/myapp
```
````

### Fallback: Directory Detection

If no commands found in README, the skill searches for:
- `deploy/`
- `k8s/`
- `manifests/`
- `openshift/`
- `.openshift/`
- `helm/`
- `charts/`

And suggests: `oc apply -f <directory>/`

## Troubleshooting

### "Not logged into OpenShift cluster"

**Solution:**
```bash
oc login <cluster-url>
```

### "bonfire CLI not found"

**Solution:**
```bash
pip install crc-bonfire
bonfire --version
```

### "No namespace available"

**Error**: Can't provision namespace

**Solutions:**
- Check available namespaces: `bonfire namespace list`
- Try different pool: `bonfire pool list`
- Wait for namespace to free up
- Contact cluster administrator

### "No deployment commands found"

**Error**: README doesn't have deployment section

**Solutions:**
1. Add deployment section to README
2. Create `deploy/` directory with YAML files
3. Provide commands interactively when prompted

### "Pod in CrashLoopBackOff"

**Error**: Deployment succeeds but pods fail

**Debug steps:**
```bash
# Check pod status
oc get pods -n $EPHEMERAL_NAMESPACE

# View logs
oc logs <pod-name> -n $EPHEMERAL_NAMESPACE

# Describe pod
oc describe pod <pod-name> -n $EPHEMERAL_NAMESPACE

# Check events
oc get events -n $EPHEMERAL_NAMESPACE --sort-by='.lastTimestamp'
```

**Common causes**:
- Image pull failures (ImagePullBackOff)
- Permission issues (OpenShift security constraints)
- Missing environment variables
- Resource limits exceeded

### Git Authentication Failed

**Error**: Can't clone private repository

**Solutions:**

For SSH:
```bash
ssh -T git@github.com
# Add SSH key if needed
```

For HTTPS:
```bash
# Use SSH URL instead
/deploy-to-ephemeral git@github.com:org/repo.git
```

## Advanced Usage

### Manual Namespace Management

```bash
# Describe namespace
bonfire namespace describe $EPHEMERAL_NAMESPACE

# Extend duration
bonfire namespace extend $EPHEMERAL_NAMESPACE -d 2h

# List all your namespaces
bonfire namespace list | grep $(oc whoami)

# Release specific namespace
bonfire namespace release ephemeral-abc123
```

### Deploy to Specific Namespace

```bash
# Don't use stored namespace
/deploy-to-ephemeral https://github.com/org/repo --namespace ephemeral-xyz789
```

### Multiple Simultaneous Namespaces

```bash
# Reserve first namespace
/reserve-ephemeral
NS1=$EPHEMERAL_NAMESPACE

# Reserve second (overwrites stored)
/reserve-ephemeral
NS2=$EPHEMERAL_NAMESPACE

# Deploy to specific namespaces
/deploy-to-ephemeral https://github.com/org/app1 --namespace $NS1
/deploy-to-ephemeral https://github.com/org/app2 --namespace $NS2
```

### Test Deployment Locally

```bash
# Create test repo
mkdir /tmp/test-app
cd /tmp/test-app
# ... create deployment files and README ...
git init && git add . && git commit -m "test"

# Deploy from local
/deploy-to-ephemeral file:///tmp/test-app
```

## Architecture

### State Management

```
┌─────────────────────────────────────────┐
│ /reserve-ephemeral                      │
│                                         │
│  1. bonfire namespace reserve           │
│  2. Parse namespace name                │
│  3. Store in:                           │
│     - $EPHEMERAL_NAMESPACE (env)        │
│     - ~/.claude/temp/current-... (file) │
└─────────────────────────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────┐
│ /deploy-to-ephemeral                    │
│                                         │
│  1. Read namespace (env or file)        │
│  2. Clone repository                    │
│  3. Parse README                        │
│  4. Execute commands                    │
│  5. Report success or debug             │
└─────────────────────────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────┐
│ /release-ephemeral                      │
│                                         │
│  1. Read namespace (env or file)        │
│  2. Confirm release                     │
│  3. bonfire namespace release           │
│  4. Clean up state files                │
└─────────────────────────────────────────┘
```

### Skill Dependencies

```
reserve-ephemeral (standalone)
    │
    └─> stores namespace
            │
            ├─> deploy-to-ephemeral (reads namespace)
            │       └─> can run multiple times
            │
            └─> release-ephemeral (reads namespace)
```

## Best Practices

### 1. Reserve Once, Deploy Many

```bash
# Good: One namespace, multiple deploys
/reserve-ephemeral
/deploy-to-ephemeral https://github.com/org/app
# ... make changes ...
/deploy-to-ephemeral https://github.com/org/app
```

### 2. Match Pool to Workload

```bash
# Small app
/reserve-ephemeral --pool minimal

# Production-like testing
/reserve-ephemeral --pool default

# AI/ML workloads
/reserve-ephemeral --pool ai-development
```

### 3. Clean Up After Testing

```bash
# Always release when done
/release-ephemeral

# Or extend if you need more time
bonfire namespace extend $EPHEMERAL_NAMESPACE -d 2h
```

### 4. Document Deployment in README

```markdown
## Deployment

Clear, simple deployment steps:

```bash
oc apply -f deploy/
```

Not complex scripts or manual steps.
```

## Contributing

Contributions welcome! Please:

1. Fork the repository
2. Create a feature branch
3. Test with real ephemeral cluster
4. Submit pull request

## License

MIT License - see LICENSE file

## Support

- GitHub Issues: [bennyturns/ephemeral-deployment-plugin/issues](https://github.com/bennyturns/ephemeral-deployment-plugin/issues)
- bonfire docs: `bonfire --help`
- OpenShift docs: https://docs.openshift.com

## Changelog

### v1.0.0 (2026-03-24)

Initial release with three core skills:
- `/reserve-ephemeral` - Namespace provisioning
- `/deploy-to-ephemeral` - Intelligent deployment
- `/release-ephemeral` - Cleanup automation
