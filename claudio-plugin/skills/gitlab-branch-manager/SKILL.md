---
name: gitlab-branch-manager
description: Create and protect GitLab branches. Use when the user asks to create a branch, protect a branch, or set up branch protection rules on GitLab. Uses the glab GitLab CLI.
allowed-tools: Bash(*/gitlab-branch-manager/scripts/*.sh:*),Bash(*/tools/glab/install.sh:*),Bash(*/tools/jq/install.sh:*)
---

# GitLab Branch Manager

## Overview

Create GitLab branches and apply protection rules. Designed for release workflows but usable for any branch management scenario.

**Prerequisites:**
- `glab` command is available and authenticated
- Works with gitlab.com, GitLab Self-Managed, and GitLab Dedicated
- Optional: `jq` for JSON parsing

**Installation:**
```bash
# glab CLI (required)
../../../tools/glab/install.sh

# jq (optional, recommended)
../../../tools/jq/install.sh
```

## Core Concepts

- **Protected Branch**: A branch with restrictions on who can push, merge, force-push, and unprotect
- **Access Levels**: `0` = No access, `30` = Developer, `40` = Maintainer, `60` = Admin (self-managed only)

## Default Protection Rules

| Rule | Default | Effect |
|------|---------|--------|
| `push_access_level` | `0` (No access) | Direct push blocked for everyone |
| `merge_access_level` | `40` (Maintainer) | Only maintainers can merge MRs |
| `allow_force_push` | `false` | Force push blocked |
| `code_owner_approval_required` | `false` | No code owner approval required |

All rules are configurable via flags. Defaults enforce a strict protection posture suitable for release branches, matching the pattern used in production repositories.

Note: `unprotect_access_level` is not set by default (uses GitLab server default). Use `--unprotect-level` or `--rule unprotect_access_level=40` to set it explicitly.

Note: `push_access_level=0` already blocks direct push and deletion for everyone.

## Repo Input Format

The script accepts three input formats:

| Format | Example | Resolution |
|--------|---------|------------|
| Short name | `aipcc-claudio` | Resolved via GitLab API search |
| Full path | `redhat/rhel-ai/ci-cd/aipcc-claudio` | Used as-is |
| URL | `https://gitlab.com/redhat/rhel-ai/ci-cd/aipcc-claudio.git` | Parsed to extract project path |

## Script Reference

### `create_and_protect_branch.sh`

Creates a branch and applies protection rules in a single operation.

**Usage:**
```bash
./scripts/create_and_protect_branch.sh <repo> <branch-name> [OPTIONS]
```

**Arguments:**

| Argument | Required | Default | Description |
|----------|----------|---------|-------------|
| `<repo>` | Yes | -- | Repo name, full path, or URL |
| `<branch-name>` | Yes | -- | Name for the new branch |
| `--ref REF` | No | `main` | Source ref to branch from |
| `--push-level N` | No | `0` | Push access level |
| `--merge-level N` | No | `40` | Merge access level |
| `--unprotect-level N` | No | not set | Unprotect access level |
| `--allow-force-push` | No | `false` | Allow force push |
| `--code-owner-approval` | No | `false` | Require code owner approval |
| `--gitlab-host HOST` | No | `gitlab.com` | GitLab hostname |
| `--rule KEY=VALUE` | No | -- | Override any protection rule by key |
| `--dry-run` | No | -- | Show planned actions, no API calls |
| `--human-readable` | No | -- | Human-readable output |

**Examples:**
```bash
# Create and protect a release branch with defaults
./scripts/create_and_protect_branch.sh aipcc-claudio release-1.5

# Branch from a specific tag
./scripts/create_and_protect_branch.sh redhat/rhel-ai/ci-cd/aipcc-claudio release-1.5 --ref v1.4.0

# Custom protection levels
./scripts/create_and_protect_branch.sh aipcc-claudio release-1.5 --push-level 40 --merge-level 40

# Override any rule with --rule
./scripts/create_and_protect_branch.sh aipcc-claudio release-1.5 --rule merge_access_level=30

# Dry run - see what would happen
./scripts/create_and_protect_branch.sh aipcc-claudio release-1.5 --dry-run

# From a URL
./scripts/create_and_protect_branch.sh https://gitlab.com/redhat/rhel-ai/ci-cd/aipcc-claudio.git release-1.5
```

**Output (JSON, default):**
```json
{
  "repository": "redhat/rhel-ai/ci-cd/aipcc-claudio",
  "branch": "release-1.5",
  "ref": "main",
  "branch_created": true,
  "protection_applied": true,
  "protection_rules": {
    "push_access_level": 0,
    "merge_access_level": 40,
    "unprotect_access_level": 0,
    "allow_force_push": false,
    "code_owner_approval_required": false
  }
}
```

## Error Handling

| Scenario | Behavior |
|----------|----------|
| Branch already exists | Exit 1 with error JSON |
| Protection matches requested rules | Exit 0, info logged to stderr |
| Protection differs from requested | Exit 1 with error JSON showing current vs requested rules |
| Source ref not found | Exit 1 with error JSON |
| Insufficient permissions | Exit 1 with error JSON |
| Short name matches multiple projects | Exit 1 with error JSON listing candidates |

## Dependencies

**Required:** `glab` (install via `tools/glab/install.sh`)
**Optional:** `jq` (install via `tools/jq/install.sh`)
