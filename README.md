# Ephemeral Deployment Plugin for Claude Code

**Transform ephemeral OpenShift cluster workflows from manual multi-step processes into single-command automation.**

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![OpenShift](https://img.shields.io/badge/OpenShift-EE0000?logo=redhatopenshift&logoColor=fff)](https://www.openshift.com/)
[![Claude Code](https://img.shields.io/badge/Claude_Code-Plugin-blue)](https://github.com/anthropics/claude-code)

---

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
bonfire -h
```

#### 2. Login to OpenShift Cluster

```bash
oc login --token=<your-token> --server=https://api.<ephemeral-cluster>.openshiftapps.com:6443
```

> **Note**: Replace `<ephemeral-cluster>` with your actual ephemeral cluster domain.
> Contact your cluster administrator for the correct URL.

**Get your token**:
- Visit your cluster's OAuth token request page: `https://oauth-openshift.apps.<ephemeral-cluster>.openshiftapps.com/oauth/token/request`
- Copy the `oc login` command with your token
- Paste into terminal and execute

Verify login:
```bash
oc whoami
```

#### 3. Install Plugin

```bash
# Clone into Claude Code plugins marketplace directory
cd ~/.claude/plugins/marketplaces
git clone https://github.com/bennyturns/ephemeral-deployment-plugin.git
```

For development (edits reflected immediately):
```bash
# Symlink your workspace clone into the marketplaces directory
ln -s /path/to/your/ephemeral-deployment-plugin ~/.claude/plugins/marketplaces/ephemeral-deployment-plugin
```

#### 4. Restart Claude Code

Restart Claude Code to load the plugin. The skills will be installed to `~/.claude/skills/`.

#### 5. Verify Installation

After restarting Claude Code, verify the skills are loaded:

```bash
# In Claude Code, type / and look for:
/reserve-ephemeral
/deploy-to-ephemeral
/release-ephemeral
```

✅ **Installation complete!** You're ready to use the plugin.

---

## 🎯 Problem Statement

Developers working with ephemeral OpenShift clusters face repetitive, error-prone manual workflows:

**Current Manual Process** (15-20 minutes per deployment):
```bash
# 1. Check bonfire pools and availability
bonfire pool list
bonfire namespace list

# 2. Reserve namespace with correct parameters
bonfire namespace reserve --pool minimal --timeout 600 -d 4h --force

# 3. Wait for provisioning...
# 4. Parse output to find namespace name
# 5. Clone repository
git clone https://github.com/org/repo /tmp/some-dir
cd /tmp/some-dir

# 6. Read README, find deployment section
# 7. Manually execute each command
oc project ephemeral-xyz123
oc apply -f deploy/database.yaml
oc apply -f deploy/app.yaml
oc apply -f deploy/service.yaml

# 8. Debug failures manually
oc get pods -n ephemeral-xyz123
oc logs <pod> -n ephemeral-xyz123
oc describe pod <pod> -n ephemeral-xyz123

# 9. Remember to clean up later (often forgotten!)
bonfire namespace release ephemeral-xyz123
```

**Pain Points**:
- ❌ **Time-consuming**: 15-20 minutes of manual work per deployment
- ❌ **Error-prone**: Easy to miss steps, use wrong namespace, forget cleanup
- ❌ **Context switching**: Between terminal, README, OpenShift console
- ❌ **Namespace sprawl**: Developers forget to release, wasting resources
- ❌ **Inconsistent**: Each developer has their own workflow
- ❌ **Poor debugging**: When deployments fail, no guided troubleshooting

## ✅ Solution

**Three focused Claude Code skills that automate the entire lifecycle:**

```bash
# New Automated Process (30 seconds)
/reserve-ephemeral                                    # 1 command
/deploy-to-ephemeral https://github.com/org/repo      # 1 command
/release-ephemeral                                    # 1 command
```

**What It Does**:
1. **`/reserve-ephemeral`** - Intelligent namespace provisioning
   - Validates prerequisites (oc login, bonfire availability)
   - Reserves from optimal pool with smart defaults
   - Stores namespace context for subsequent commands
   - Provides console URLs and namespace details

2. **`/deploy-to-ephemeral`** - Smart deployment automation
   - Clones any GitHub repository (public/private, SSH/HTTPS)
   - Parses README to extract deployment commands
   - Executes commands sequentially with proper error handling
   - Auto-detects deployment patterns (oc apply, helm, kubectl)
   - **Launches interactive debug session on failure**

3. **`/release-ephemeral`** - Clean resource management
   - Shows current resources before cleanup
   - Releases bonfire reservation properly
   - Cleans up local state and temporary files
   - Prevents namespace sprawl and resource waste

## 💡 Key Benefits

### For Developers
- ⚡ **95% faster**: 30 seconds vs 15-20 minutes per deployment
- 🎯 **Zero context switching**: Everything in Claude Code
- 🔄 **Rapid iteration**: Redeploy to same namespace in seconds
- 🐛 **Guided debugging**: Auto-suggests diagnostic commands on failure
- 📚 **Self-documenting**: README parsing ensures deployments match docs

### For Engineering Teams
- 💰 **Resource efficiency**: Automated cleanup prevents namespace sprawl
- 📏 **Standardization**: Consistent workflow across all developers
- 🚀 **Faster onboarding**: New developers productive in minutes
- 📊 **Better practices**: Encourages proper README documentation
- 🔒 **Safer operations**: Built-in validation and error handling

### For Management
- ⏱️ **Time savings**: ~80 hours/year per developer (assuming 5 deploys/week)
- 💵 **Cost reduction**: Fewer wasted namespace hours through auto-cleanup
- 📈 **Higher velocity**: Faster iteration = faster feature delivery
- 🎓 **Lower barriers**: Reduced tribal knowledge requirements
- 🔍 **Visibility**: Standardized workflow enables better metrics

## 📊 Use Cases

### Daily Development Workflow
```bash
# Morning: Reserve namespace for the day
/reserve-ephemeral --duration 8h

# Deploy and test feature branch
/deploy-to-ephemeral https://github.com/my-org/my-app

# Make changes, push to GitHub, redeploy
/deploy-to-ephemeral https://github.com/my-org/my-app  # Fast! Same namespace

# Test different configurations
/deploy-to-ephemeral https://github.com/my-org/my-app  # Again!

# End of day: Clean up
/release-ephemeral
```

### Integration Testing
```bash
# Deploy multiple microservices to one namespace
/reserve-ephemeral
/deploy-to-ephemeral https://github.com/my-org/frontend
/deploy-to-ephemeral https://github.com/my-org/backend
/deploy-to-ephemeral https://github.com/my-org/database

# Test integration, then cleanup
/release-ephemeral
```

### Bug Reproduction
```bash
# Quickly deploy specific commit to test bug
/reserve-ephemeral
/deploy-to-ephemeral https://github.com/my-org/app

# Deployment fails? Auto debug session starts!
# Suggested commands, interactive help, namespace preserved
```

### Demo Preparation
```bash
# Long-running demo environment
/reserve-ephemeral --duration 12h --pool default
/deploy-to-ephemeral https://github.com/my-org/demo-app

# Extend if demo runs long
bonfire namespace extend $EPHEMERAL_NAMESPACE -d 4h
```

## 🏗️ Architecture

### Three-Skill Workflow
```
┌─────────────────────────────────────────────────────────────┐
│ Developer                                                    │
└───────────────┬─────────────────────────────────────────────┘
                │
                │ /reserve-ephemeral
                ▼
┌─────────────────────────────────────────────────────────────┐
│ Skill 1: Reserve Namespace                                   │
│                                                              │
│  ✓ Validate: oc whoami, bonfire -h                   │
│  ✓ Reserve: bonfire namespace reserve --pool minimal        │
│  ✓ Store: $EPHEMERAL_NAMESPACE + file                       │
│  ✓ Output: Namespace details, console URL                   │
└───────────────┬─────────────────────────────────────────────┘
                │
                │ Namespace: ephemeral-abc123 (stored)
                │
                │ /deploy-to-ephemeral <repo-url>
                ▼
┌─────────────────────────────────────────────────────────────┐
│ Skill 2: Deploy to Namespace                                │
│                                                              │
│  ✓ Validate: Namespace accessible                           │
│  ✓ Clone: git clone <repo> /tmp/xyz                         │
│  ✓ Parse: Extract deployment commands from README           │
│  ✓ Deploy: Execute commands sequentially                    │
│  ✓ Handle: Success summary OR debug session                 │
└───────────────┬─────────────────────────────────────────────┘
                │
                │ Success! Or Debug Session initiated
                │
                │ /release-ephemeral
                ▼
┌─────────────────────────────────────────────────────────────┐
│ Skill 3: Release Namespace                                  │
│                                                              │
│  ✓ Show: Current resources in namespace                     │
│  ✓ Confirm: User approval                                   │
│  ✓ Release: bonfire namespace release --force               │
│  ✓ Cleanup: Remove state files, unset vars                  │
└─────────────────────────────────────────────────────────────┘
```

### State Management
```
                    ┌──────────────────────────┐
                    │  /reserve-ephemeral       │
                    │  provisions namespace     │
                    └────────────┬──────────────┘
                                 │
                    ┌────────────▼──────────────┐
                    │  State Storage             │
                    │  • $EPHEMERAL_NAMESPACE    │
                    │  • ~/.claude/temp/...      │
                    └────────────┬──────────────┘
                                 │
                    ┌────────────▼──────────────┐
                    │  /deploy-to-ephemeral     │
                    │  reads stored namespace    │
                    └────────────┬──────────────┘
                                 │
                    ┌────────────▼──────────────┐
                    │  /release-ephemeral       │
                    │  cleans up everything      │
                    └────────────────────────────┘
```

## 📋 Feature Overview

### Skill Capabilities

| Feature | /reserve-ephemeral | /deploy-to-ephemeral | /release-ephemeral |
|---------|-------------------|---------------------|-------------------|
| Prerequisite validation | ✅ oc, bonfire | ✅ namespace, git | ✅ bonfire |
| Bonfire integration | ✅ Reserve with pools | ❌ | ✅ Release |
| Git operations | ❌ | ✅ Clone (SSH/HTTPS) | ❌ |
| README parsing | ❌ | ✅ Smart extraction | ❌ |
| Command execution | ❌ | ✅ Sequential | ❌ |
| Error handling | ✅ Detailed messages | ✅ Debug session | ✅ Graceful |
| State management | ✅ Store namespace | ✅ Read namespace | ✅ Clean state |
| User interaction | ⚠️ Minimal | ✅ Debug prompts | ✅ Confirmation |

### Supported Deployment Patterns

**Recognized Commands**:
- ✅ `oc apply -f <file>` - OpenShift resources
- ✅ `oc process -f <template> | oc apply -f -` - Templated deployments
- ✅ `helm install <name> <chart>` - Helm charts
- ✅ `helm upgrade --install <name> <chart>` - Helm upgrades
- ✅ `kubectl apply -f <file>` - Kubernetes resources
- ✅ `oc apply -f <directory>/` - Bulk application

**Smart Fallbacks**:
- 📁 Directory detection: `deploy/`, `k8s/`, `manifests/`, `openshift/`, `helm/`, `charts/`
- 💬 Interactive prompts: If no commands found, asks user
- 🔍 Context-aware suggestions: Based on repository structure

## 🚀 Getting Started

### Complete Walkthrough: Your First Deployment

Let's walk through deploying a sample application end-to-end.

#### Step 1: Reserve a Namespace

In Claude Code, run:
```bash
/reserve-ephemeral
```

**What happens:**
```
================================================
Validating Prerequisites...
================================================

✅ OpenShift: Logged in as your-username
✅ bonfire: installed
✅ git: git version 2.43.0

All prerequisites validated ✓

================================================
Reserving ephemeral namespace...
================================================

Configuration:
  Pool: minimal
  Duration: 4h
  Timeout: 600s

2026-03-24 10:00:00 [INFO] checking for available namespaces to reserve...
2026-03-24 10:00:01 [INFO] namespace 'ephemeral-abc123' is reserved by 'your-username' for '4h' from the minimal pool

================================================
✅ NAMESPACE RESERVED
================================================

Namespace: ephemeral-abc123
Pool: minimal
Duration: 4h

Namespace Details:
  Console URL: https://console-openshift-console.apps.<ephemeral-cluster>.openshiftapps.com/k8s/cluster/projects/ephemeral-abc123
  Project: ephemeral-abc123

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Namespace stored in:
  - Environment variable: $EPHEMERAL_NAMESPACE
  - File: ~/.claude/temp/current-ephemeral-namespace

Use with deployment:
  /deploy-to-ephemeral <repo-url>

Manage namespace:
  - Extend: bonfire namespace extend ephemeral-abc123
  - Release: /release-ephemeral
  - Describe: bonfire namespace describe ephemeral-abc123
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

**Key Points:**
- ✅ Namespace reserved: `ephemeral-abc123`
- ✅ Stored for next command (no need to copy/paste)
- ✅ Expires automatically in 4 hours
- ✅ Console URL provided for manual inspection

---

#### Step 2: Deploy Your Application

Now deploy a GitHub repository:
```bash
/deploy-to-ephemeral https://github.com/your-org/your-app
```

**What happens:**
```
================================================
Validating Prerequisites...
================================================

✅ OpenShift: Logged in as your-username
✅ Namespace: ephemeral-abc123 (accessible)
✅ git: git version 2.43.0

All prerequisites validated ✓

================================================
Cloning repository...
================================================

Repository: https://github.com/your-org/your-app
Clone directory: /tmp/tmp.xyz789

Cloning into '/tmp/tmp.xyz789'...
remote: Enumerating objects: 127, done.
remote: Counting objects: 100% (127/127), done.
remote: Compressing objects: 100% (89/89), done.
remote: Total 127 (delta 45), reused 98 (delta 32)
Receiving objects: 100% (127/127), 45.23 KiB | 1.51 MiB/s, done.
Resolving deltas: 100% (45/45), done.

✅ Repository cloned successfully

================================================
Parsing README for deployment instructions...
================================================

Found: README.md

Extracting deployment commands...

Found deployment command(s):
  → oc apply -f deploy/database.yaml
  → oc apply -f deploy/application.yaml
  → oc apply -f deploy/service.yaml

================================================
Executing deployment...
================================================

Setting namespace context: ephemeral-abc123
Now using project "ephemeral-abc123" on server "https://api.<ephemeral-cluster>.openshiftapps.com:6443".

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Running: oc apply -f deploy/database.yaml
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

deployment.apps/postgres created
service/postgres created

✅ Command completed successfully

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Running: oc apply -f deploy/application.yaml
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

deployment.apps/myapp created
configmap/myapp-config created

✅ Command completed successfully

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Running: oc apply -f deploy/service.yaml
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

service/myapp created
route.route.openshift.io/myapp created

✅ Command completed successfully

================================================
✅ DEPLOYMENT SUCCESSFUL
================================================

Namespace: ephemeral-abc123
Repository: /tmp/tmp.xyz789

Deployed resources:
NAME                          READY   STATUS    RESTARTS   AGE
pod/myapp-7d4f8c9-x5z2w       1/1     Running   0          5s
pod/postgres-8k3j2l-9mn4p     1/1     Running   0          7s

NAME               TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
service/myapp      ClusterIP   172.30.158.201   <none>        8080/TCP   5s
service/postgres   ClusterIP   172.30.45.100    <none>        5432/TCP   7s

NAME                       READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/myapp      1/1     1            1           5s
deployment.apps/postgres   1/1     1            1           7s

NAME                                  HOST/PORT
route.route.openshift.io/myapp       myapp-ephemeral-abc123.apps.<ephemeral-cluster>.openshiftapps.com

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Next steps:
  - View pods: oc get pods -n ephemeral-abc123
  - View services: oc get svc -n ephemeral-abc123
  - View routes: oc get routes -n ephemeral-abc123
  - Check logs: oc logs -l app=myapp -n ephemeral-abc123
  - Access app: https://myapp-ephemeral-abc123.apps.<ephemeral-cluster>.openshiftapps.com

Namespace management:
  - Extend: bonfire namespace extend ephemeral-abc123
  - Release: /release-ephemeral
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Clone directory: /tmp/tmp.xyz789
(Kept for reference. Delete manually when done.)
```

**Key Points:**
- ✅ Automatically found and executed 3 deployment commands
- ✅ All resources created successfully
- ✅ Route URL provided for immediate testing
- ✅ Namespace context maintained from Step 1

**Test your deployment:**
Visit the route URL in your browser or:
```bash
curl https://myapp-ephemeral-abc123.apps.<ephemeral-cluster>.openshiftapps.com
```

---

#### Step 3: Make Changes and Redeploy

Made code changes? Redeploy to the **same namespace** (fast iteration):

```bash
# Push changes to GitHub
git push origin main

# Redeploy (uses same namespace - no wait for provisioning!)
/deploy-to-ephemeral https://github.com/your-org/your-app
```

**This takes ~10 seconds** instead of 15-20 minutes because:
- No namespace provisioning wait
- No manual README parsing
- No copy/paste namespace names
- Automated command execution

---

#### Step 4: Clean Up

When done testing:
```bash
/release-ephemeral
```

**What happens:**
```
Namespace to release: ephemeral-abc123

Namespace found in bonfire:
ephemeral-abc123  true  ready  2/2  your-username  minimal  3h45m12s

Current resources in namespace:
────────────────────────────────────────────────
NAME                          READY   STATUS    RESTARTS   AGE
pod/myapp-7d4f8c9-x5z2w       1/1     Running   0          15m
pod/postgres-8k3j2l-9mn4p     1/1     Running   0          15m

NAME               TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
service/myapp      ClusterIP   172.30.158.201   <none>        8080/TCP   15m
service/postgres   ClusterIP   172.30.45.100    <none>        5432/TCP   15m
────────────────────────────────────────────────

Release namespace ephemeral-abc123?

This will:
- Delete all resources in the namespace
- Release the bonfire reservation
- Clean up local state ($EPHEMERAL_NAMESPACE)

Continue? (yes/no)

> yes

================================================
Releasing namespace...
================================================

2026-03-24 10:15:30 [INFO] releasing namespace 'ephemeral-abc123'
namespacereservation.cloud.redhat.com "bonfire-reservation-xyz" deleted

✅ Bonfire reservation released

================================================
Cleaning up local state...
================================================

✅ Removed stored namespace file
✅ Unset EPHEMERAL_NAMESPACE variable

================================================
✅ NAMESPACE RELEASED
================================================

Namespace 'ephemeral-abc123' has been released.

To reserve a new namespace:
  /reserve-ephemeral
```

**Key Points:**
- ✅ Shows what will be deleted before asking
- ✅ Requires confirmation (prevents accidents)
- ✅ Cleans up all local state
- ✅ Frees cluster resources properly

---

### 🎉 Success!

You've completed a full deployment lifecycle:
1. ✅ Reserved namespace in ~5 seconds
2. ✅ Deployed application automatically in ~15 seconds
3. ✅ Tested and iterated quickly
4. ✅ Cleaned up properly

**Total time: ~30 seconds** vs. **15-20 minutes manually**

---

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

## Known Issues and Tips

### Routes Require Edge TLS

On the ephemeral cluster, plain HTTP routes (`oc expose service`) return "Application is not available." Always create routes with edge TLS termination:

```bash
# Instead of: oc expose service/my-app
oc create route edge my-app --service=my-app --port=8080-tcp
```

The deploy skill handles this automatically.

### Static Sites (Repos Without Manifests)

If your repository has no Kubernetes manifests or deployment commands (e.g., a plain HTML project), the deploy skill will offer to serve it as a static site using an nginx s2i build (`registry.access.redhat.com/ubi9/nginx-122`).

### bonfire CLI Notes

- `bonfire -h` is not supported. Use `bonfire -h` to verify installation.
- Both `reserve` and `release` commands use `--force` to skip interactive prompts (required for automation in Claude Code).

---

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
bonfire -h
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
│  3. bonfire namespace release --force   │
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
