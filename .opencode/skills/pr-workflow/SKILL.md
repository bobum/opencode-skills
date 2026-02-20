---
name: pr-workflow
description: Standardized PR creation workflow with proper issue linking. Use when creating pull requests across any any repo.
---

# Pull Request Workflow

This skill covers the complete workflow for creating PRs with proper issue references.

## Critical: Issue Linking

> **NEVER use bare `#123` references.** GitHub uses the same numbering for issues AND PRs, so `#123` could link to a closed PR instead of the intended issue.

### Correct Syntax

```markdown
# WRONG - ambiguous, might link to a PR
Closes #123

# RIGHT - explicit repo, verifiable
Closes merciless-creations/gridiron-engine#210

# RIGHT - cross-repo reference
Closes merciless-creations/gridiron-web#45
Closes merciless-creations/gridiron-meta#12
```

### Verification Before Linking

**ALWAYS verify the reference is an issue (not a PR) before including it:**

```bash
# Verify it's an issue and check its state
gh issue view 210 --repo merciless-creations/gridiron-engine --json number,title,state,url

# If this fails with "not found" or shows a PR, DO NOT use it
# PRs are accessed via: gh pr view 210 --repo ...

# Get the full URL for the PR body
gh issue view 210 --repo merciless-creations/gridiron-engine --json url --jq '.url'
```

### What to Check

1. **It's an issue, not a PR** - `gh issue view` succeeds
2. **It's open** - `state` is `OPEN`, not `CLOSED`
3. **It's in the correct repo** - matches the work being done
4. **It's the right issue** - title matches the work

---

## PR Creation Steps

### Step 1: Verify Branch State

```bash
# Check you're on the correct feature branch
git branch --show-current

# Verify commits are ready
git log --oneline -5

# Check nothing uncommitted
git status
```

### Step 2: Push Branch

```bash
# Push with upstream tracking
git push -u origin feature/your-branch-name
```

### Step 3: Verify Issue Reference

```bash
# Check the issue exists and is open
gh issue view 210 --repo merciless-creations/gridiron-engine --json number,title,state

# Expected output:
# {
#   "number": 210,
#   "title": "DPI yardage distribution fix",
#   "state": "OPEN"
# }

# If you see "state": "CLOSED" or the command fails, DO NOT reference this issue
```

### Step 4: Create PR

```bash
gh pr create \
  --repo merciless-creations/gridiron-engine \
  --base main \
  --title "Fix DPI yardage distribution" \
  --body "$(cat <<'EOF'
## Summary

- Implemented log-normal distribution for DPI yardage sampling
- Added coverage penalty completion damping
- Added DPI yardage validation to validator

## Test Plan

- [ ] All unit tests pass
- [ ] 1000-game validation within 15% of NFL targets
- [ ] DPI fires on both complete and incomplete passes

Closes merciless-creations/gridiron-engine#210
EOF
)"
```

---

## PR Body Template

```markdown
## Summary

- [Bullet point describing main change]
- [Additional changes]

## Test Plan

- [ ] [How to verify the change works]
- [ ] [Additional test scenarios]

Closes merciless-creations/{repo}#{issue_number}
```

---

## Cross-Repo PRs

When a PR in one repo relates to issues in another:

```bash
# Verify each issue first
gh issue view 50 --repo merciless-creations/gridiron-meta --json title,state
gh issue view 265 --repo merciless-creations/gridiron --json title,state

# In PR body, use full references
Part of merciless-creations/gridiron-meta#50
Closes merciless-creations/gridiron#265
```

---

## Linking Keywords

| Keyword | Effect |
|---------|--------|
| `Closes` | Closes the issue when PR merges |
| `Fixes` | Same as Closes |
| `Resolves` | Same as Closes |
| `Part of` | Links without closing (for sub-issues of epics) |
| `Related to` | Links without closing |

**Always use `Closes` for the primary issue being addressed.**

---

## Common Mistakes

| Mistake | Problem | Fix |
|---------|---------|-----|
| `Closes #123` | Ambiguous - might link to PR #123 | Use `merciless-creations/repo#123` |
| Not verifying first | Links to wrong item | Run `gh issue view` first |
| Linking to closed issue | Confusing, no effect | Check `state` field |
| Wrong repo | Links to wrong project | Verify repo in `gh issue view` |
| Linking to PR instead of issue | Confusing, won't auto-close | Use `gh issue view` not `gh pr view` |

---

## ⚠️ PR Body Updates: Use REST API, Not `gh pr edit`

> **`gh pr edit --body` silently fails.** It returns a GraphQL "Projects Classic deprecation" warning but does NOT actually update the PR body. This is a known issue when the repo has GitHub Projects linked.

### The Problem

```bash
# DON'T DO THIS - silently fails, body is NOT updated
gh pr edit 222 --repo merciless-creations/gridiron-engine --body "new body text"
# Returns GraphQL warning but body remains unchanged
```

### The Fix: Use REST API

```bash
# DO THIS - reliably updates the PR body
gh api repos/merciless-creations/gridiron-engine/pulls/222 \
  -X PATCH \
  -f body="$(cat <<'EOF'
## Summary

- Your PR description here

Closes merciless-creations/gridiron-engine#210
EOF
)"
```

### For Long PR Bodies

Write the body to a temp file first:

```bash
cat > /tmp/pr-body.md <<'EOF'
## Summary

- Change description here

## Test Plan

- [ ] Tests pass

Closes merciless-creations/gridiron-engine#210
EOF

gh api repos/merciless-creations/gridiron-engine/pulls/222 \
  -X PATCH \
  -f body="$(cat /tmp/pr-body.md)"
```

### Why This Matters

- The `gh pr edit --body` command uses GraphQL, which conflicts with GitHub Projects Classic deprecation
- The REST API (`gh api ... -X PATCH`) bypasses this entirely and works reliably
- **Always use REST API for updating PR bodies** in merciless-creations repos

---

## Repository Reference Guide

| Repo | Full Reference |
|------|----------------|
| API backend | `merciless-creations/gridiron#N` |
| Frontend | `merciless-creations/gridiron-web#N` |
| Simulation engine | `merciless-creations/gridiron-engine#N` |
| Injury engine | `merciless-creations/gridiron-injury#N` |
| Meta/Epics | `merciless-creations/gridiron-meta#N` |

---

## Verification Script

Run this before creating any PR with issue references:

```bash
#!/bin/bash
# verify-issue.sh REPO ISSUE_NUMBER

REPO=$1
ISSUE=$2

echo "Checking merciless-creations/$REPO#$ISSUE..."

result=$(gh issue view $ISSUE --repo merciless-creations/$REPO --json number,title,state,url 2>&1)

if echo "$result" | grep -q "Could not resolve"; then
    echo "ERROR: Issue #$ISSUE not found in $REPO"
    echo "This might be a PR number, not an issue."
    exit 1
fi

state=$(echo "$result" | jq -r '.state')
title=$(echo "$result" | jq -r '.title')
url=$(echo "$result" | jq -r '.url')

if [ "$state" != "OPEN" ]; then
    echo "WARNING: Issue is $state, not OPEN"
fi

echo "Found: $title"
echo "State: $state"
echo "URL: $url"
echo ""
echo "Use in PR body: Closes merciless-creations/$REPO#$ISSUE"
```

---

## Checklist Before Creating PR

- [ ] On correct feature branch
- [ ] All changes committed
- [ ] Branch pushed to origin
- [ ] Issue verified with `gh issue view`
- [ ] Issue is OPEN (not closed)
- [ ] Issue is in the correct repo
- [ ] Using full `owner/repo#number` syntax
- [ ] PR title is clear and concise
- [ ] PR body includes Summary, Test Plan, and issue reference
