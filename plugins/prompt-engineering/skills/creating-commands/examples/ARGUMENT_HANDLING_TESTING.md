# Testing Command Argument Handling

A worked example showing TDD for command argument handling.

## Goal

Create a `/project:fix-issue` command that:

- Takes an issue number as argument
- Optionally takes priority level
- Handles missing arguments gracefully

## Test Scenarios

### Scenario 1: Single Argument

```text
Input: /project:fix-issue 123
Expected: Fix GitHub issue #123
```text

### Scenario 2: Two Arguments
```text
Input: /project:fix-issue 123 high
Expected: Fix GitHub issue #123 with high priority
```text

### Scenario 3: No Arguments
```text
Input: /project:fix-issue
Expected: List recent issues and ask which to fix (not crash)
```text

### Scenario 4: Extra Arguments
```text
Input: /project:fix-issue 123 high urgent
Expected: Handle gracefully (ignore extra or include in context)
```text

---

## Iteration 1: Minimal Command

### Command (v1)

```markdown

# .claude/commands/fix-issue.md

Fix GitHub issue #$ARGUMENTS
```text

### Test Results

| Scenario | Result | Notes |
|----------|--------|-------|
| 1: Single arg | ✅ Pass | "Fix GitHub issue #123" |
| 2: Two args | ⚠️ Partial | "Fix GitHub issue #123 high" (awkward) |
| 3: No args | ❌ Fail | "Fix GitHub issue #" (broken) |
| 4: Extra args | ⚠️ Partial | All args concatenated |

### Problems Identified

1. No handling for missing arguments
2. Two-argument case is grammatically awkward
3. No description for `/help`

---

## Iteration 2: Add Argument Handling

### Command (v2)

```markdown

# .claude/commands/fix-issue.md

---
description: Fix a GitHub issue by number
argument-hint: [issue-number]
---

Fix GitHub issue #$ARGUMENTS

If no issue number provided, list recent open issues:
!`gh issue list --state open --limit 5 2>/dev/null || echo "Could not list issues"`

Then ask which issue to fix.
```text

### Test Results

| Scenario | Result | Notes |
|----------|--------|-------|
| 1: Single arg | ✅ Pass | Works correctly |
| 2: Two args | ⚠️ Partial | Priority not used well |
| 3: No args | ✅ Pass | Lists issues and asks |
| 4: Extra args | ⚠️ Partial | Still concatenated |

### Problems Identified

1. Two-argument case needs positional variables
2. Should use $1, $2 for structured arguments

---

## Iteration 3: Positional Arguments

### Command (v3)

```markdown

# .claude/commands/fix-issue.md

---
description: Fix a GitHub issue with optional priority
argument-hint: [issue-number] [priority]
---

## Task

Fix GitHub issue #$1

Priority: $2 (default: medium if not specified)

## Process

1. Fetch issue details from GitHub
2. Understand the problem described
3. Implement the fix
4. Write or update tests
5. Create a commit referencing the issue

## If No Issue Number

If no issue number provided, show recent issues:
!`gh issue list --state open --limit 5 2>/dev/null || echo "Run 'gh auth login' to enable issue listing"`

Then ask which issue to fix.
```text

### Test Results

| Scenario | Result | Notes |
|----------|--------|-------|
| 1: Single arg | ✅ Pass | $1=123, $2=empty → uses default |
| 2: Two args | ✅ Pass | $1=123, $2=high → uses priority |
| 3: No args | ✅ Pass | Lists issues, asks user |
| 4: Extra args | ✅ Pass | Extra args ignored (acceptable) |

### All Scenarios Pass ✅

---

## Final Command

```markdown

# .claude/commands/fix-issue.md

---
description: Fix a GitHub issue with optional priority
argument-hint: [issue-number] [priority]
---

## Task

Fix GitHub issue #$1

Priority: $2 (default: medium if not specified)

## Context

Current branch: !`git branch --show-current`
Uncommitted changes: !`git status --short | head -5`

## Process

1. Fetch issue details from GitHub
2. Understand the problem described
3. Implement the fix
4. Write or update tests
5. Create a commit referencing the issue

**REQUIRED:** Follow `git-workflow` skill for commit message format.

## If No Issue Number

If no issue number provided, show recent issues:
!`gh issue list --state open --limit 5 2>/dev/null || echo "GitHub CLI not configured"`

Then ask which issue to fix.
```text

---

## Key Lessons

### 1. Start Simple, Add Handling

Don't try to handle all cases in v1. Start with the happy path, then add edge case handling.

### 2. Use $1, $2 for Structured Arguments

When command has multiple distinct parameters, use positional variables instead of $ARGUMENTS.

```markdown

# $ARGUMENTS - single blob

Fix $ARGUMENTS  # "123 high" as one string

# $1, $2 - structured

Fix #$1 with priority $2  # $1="123", $2="high"
```text

### 3. Always Handle Missing Arguments

Never assume arguments will be provided:

```markdown

# Bad - crashes without argument

Fix issue #$1

# Good - graceful fallback

Fix issue #$1

If no issue number provided, [fallback behavior]
```text

### 4. Use Bash for Dynamic Defaults

When arguments are missing, use !`command` to provide useful context:

```markdown
If no file specified, review recent changes:
!`git diff --name-only HEAD~3`
```text

### 5. Reference Skills for Methodology

Keep commands thin, reference skills for detailed guidance:

```markdown
**REQUIRED:** Follow `git-workflow` skill for commit format.
```text

---

## Argument Pattern Quick Reference

### Single Optional Argument

```markdown
---
argument-hint: [target]
---
Process $ARGUMENTS

If no target provided, use current directory.
```text

### Two Required Arguments

```markdown
---
argument-hint: [source] [destination]
---
Copy from $1 to $2

Both source and destination are required.
```text

### Required + Optional

```markdown
---
argument-hint: [file] [options]
---
Process $1 with options: $2

File is required. Options default to standard settings.
```text

### Variadic (All Remaining)

```markdown
---
argument-hint: [files...]
---
Process files: $ARGUMENTS

Handles any number of files.
```text

---

## Testing Protocol Summary

For each command:

1. **Define scenarios** with expected behavior
2. **Write minimal command** (v1)
3. **Test all scenarios** - document failures
4. **Fix failures** (v2, v3, ...)
5. **Re-test** until all pass
6. **Add refinements** (skill refs, bash context)
7. **Verify in `/help`**
