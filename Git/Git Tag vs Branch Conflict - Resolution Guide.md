# Git Tag vs Branch Conflict - Resolution Guide

## Problem

When both a **tag** and a **branch** exist with the same name in a Git repository, Git may prioritize pulling from the **tag** instead of the **branch** when using ambiguous commands.

### Example Scenario

In our repository, we have:
- **Tag**: `dev/v2.3.04-hotfix`
- **Remote Branch**: `origin/dev/v2.3.04-hotfix`
- **Local Branch**: `dev/v2.3.04-hotfix-branch`

When running:
```powershell
git pull origin dev/v2.3.04-hotfix
```

Git may pull from the **tag** instead of the **branch**, which is problematic because:
- Tags are immutable snapshots (fixed points in history)
- Branches are meant for ongoing development and receive updates
- Pulling from a tag doesn't get the latest changes from the branch

---

## Solution

### Option 1: Explicitly Specify the Branch (Recommended)

Use the full ref path to explicitly pull from the branch:

```powershell
# Pull from the branch, not the tag
git pull origin refs/heads/dev/v2.3.04-hotfix
```

### Option 2: Set Up Branch Tracking

Configure your local branch to track the remote branch, then use simple `git pull`:

```powershell
# Set up tracking (one-time setup)
git branch --set-upstream-to=origin/dev/v2.3.04-hotfix dev/v2.3.04-hotfix-branch

# Then just use:
git pull
```

### Option 3: Fetch and Merge Explicitly

```powershell
# Fetch from the branch
git fetch origin refs/heads/dev/v2.3.04-hotfix

# Merge the fetched changes
git merge FETCH_HEAD
```

---

## Verification Commands

### Check What Refs Exist on Remote

```powershell
# List all remote refs
git ls-remote origin

# Filter for specific name
git ls-remote origin | findstr "dev/v2.3.04-hotfix"
```

### Check Current Branch Status

```powershell
# See current commit and all refs pointing to it
git show HEAD

# See which branch you're on
git branch

# See tracking information
git branch -vv
```

### Dry Run Before Pulling

```powershell
# Test what would be fetched from branch
git fetch origin refs/heads/dev/v2.3.04-hotfix --dry-run --verbose

# Test what would be fetched from tag
git fetch origin refs/tags/dev/v2.3.04-hotfix --dry-run --verbose
```

---

## Understanding Git Ref Types

| Ref Type | Path Format | Purpose | Mutable |
|----------|-------------|---------|---------|
| **Branch** | `refs/heads/[name]` | Ongoing development | ✅ Yes |
| **Remote Branch** | `refs/remotes/origin/[name]` | Tracking remote branches | ✅ Yes |
| **Tag** | `refs/tags/[name]` | Mark release points | ❌ No |

---

## Best Practices

1. **Avoid naming conflicts**: Don't create tags and branches with the same name
2. **Use explicit refs**: When in doubt, use the full ref path (`refs/heads/` or `refs/tags/`)
3. **Set up tracking**: Configure branch tracking for regular workflows
4. **Use descriptive names**: 
   - Tags: `v2.3.04-hotfix` or `release/v2.3.04-hotfix`
   - Branches: `dev/v2.3.04-hotfix-branch` or `hotfix/v2.3.04`

---

## Current Repository Status

```
Commit: 92b0de0573e0f43911506dde30a8ef945258f9ae

References pointing to this commit:
- HEAD -> dev/v2.3.04-hotfix-branch (current local branch)
- origin/dev/v2.3.04-hotfix (remote branch)
- dev/v2.3.04-hotfix (tag)
```

All three references currently point to the same commit, but the **branch** may receive future updates while the **tag** remains fixed.

---

## Quick Reference

```powershell
# ✅ CORRECT: Pull from branch
git pull origin refs/heads/dev/v2.3.04-hotfix

# ❌ AMBIGUOUS: May pull from tag
git pull origin dev/v2.3.04-hotfix

# ✅ CORRECT: Fetch specific tag
git fetch origin refs/tags/dev/v2.3.04-hotfix

# ✅ CORRECT: View tag
git show refs/tags/dev/v2.3.04-hotfix
```
