# Testing Commands With Subagents

**Load this reference when:** creating or editing commands, before deployment, to verify invocation works correctly and arguments are handled properly.

## Overview

**Testing commands is just TDD applied to workflow automation.**

You define expected behavior (invocation scenarios), watch them fail (command not found or wrong behavior), write the command (markdown file), watch tests pass (correct execution), and refactor (improve argument handling).

**Core principle:** If you didn't verify invocation and argument handling, you don't know if the command works in all scenarios.

**Two testing dimensions:**

1. **Invocation testing**: Does the command execute when called?
2. **Argument testing**: Does the command handle $ARGUMENTS correctly?

## When to Use

Test commands that:

- Take arguments ($ARGUMENTS, $1, $2)
- Reference skills (verify skill is loaded)
- Have tool restrictions (verify restrictions work)
- Include bash execution (!`command`)

Don't test:

- Trivial one-line commands with no arguments
- Commands that are pure prompts with no dynamic content

## TDD Mapping for Command Testing

| TDD Phase        | Command Testing                   | What You Do                                      |
|------------------|-----------------------------------|--------------------------------------------------|
| **RED**          | Define expected behavior          | Document what invocation should do               |
| **Verify RED**   | Verify command doesn't exist      | Run `/project:command` - should fail             |
| **GREEN**        | Write command                     | Create markdown file                             |
| **Verify GREEN** | Test invocation                   | Run command, verify expected behavior            |
| **REFACTOR**     | Improve                           | Handle edge cases, add skill references          |
| **Stay GREEN**   | Re-verify                         | Test all scenarios still work                    |

## RED Phase: Define Expected Behavior

**Goal:** Document what the command should do before writing it.

### Define Invocation Scenarios

```markdown
Command: /project:fix-issue

Scenario 1: With argument
  Input: /project:fix-issue 123
  Expected: Claude fixes GitHub issue #123

Scenario 2: Without argument
  Input: /project:fix-issue
  Expected: Claude asks which issue to fix (not crash)

Scenario 3: Multiple arguments
  Input: /project:fix-issue 123 high
  Expected: Claude fixes issue #123 with high priority
```text

### Verify Command Doesn't Exist

```markdown

1. Run: /project:fix-issue 123
2. Expected: "Command not found" or similar error
3. Document: Baseline confirmed - command doesn't exist

```text

## GREEN Phase: Write Minimal Command

Write command that passes all defined scenarios.

### Basic Command

```markdown

# .claude/commands/fix-issue.md

---
description: Fix a GitHub issue
argument-hint: [issue-number] [priority]
---

Fix GitHub issue: $ARGUMENTS

If no issue number provided, ask which issue to fix.
```text

### Test Each Scenario

```markdown
Test 1: /project:fix-issue 123
Result: ✅ Claude attempts to fix issue 123

Test 2: /project:fix-issue
Result: ✅ Claude asks which issue to fix

Test 3: /project:fix-issue 123 high
Result: ✅ Claude fixes with high priority
```text

## VERIFY GREEN: Argument Testing

### $ARGUMENTS Scenarios

| Invocation | $ARGUMENTS Value | Expected Behavior |
|------------|------------------|-------------------|
| `/cmd` | empty | Handle gracefully |
| `/cmd foo` | "foo" | Process single arg |
| `/cmd foo bar` | "foo bar" | Process full string |
| `/cmd "foo bar"` | "foo bar" | Handle quoted args |

### Positional Argument Scenarios ($1, $2)

| Invocation | $1 | $2 | Expected |
|------------|----|----|----------|
| `/cmd` | empty | empty | Handle missing |
| `/cmd foo` | "foo" | empty | Handle partial |
| `/cmd foo bar` | "foo" | "bar" | Process both |

### Test Protocol

```markdown
For each scenario:

1. Invoke command with test input
2. Observe actual behavior
3. Compare to expected behavior
4. Document any discrepancies
5. Fix command if needed

```text

## REFACTOR Phase: Improve Edge Cases

### Handle Missing Arguments

<Before>

```markdown
Fix issue #$1 with priority $2.
```text
Problem: Fails when arguments missing.
</Before>

<After>

```markdown
Fix issue #$1 with priority $2.

If no issue number provided, list open issues:
!`gh issue list --limit 5`

If no priority provided, default to "medium".
```text
</After>

### Add Skill References

<Before>

```markdown
Create a commit for staged changes.
```text
Problem: Inconsistent commit format.
</Before>

<After>

```markdown
Create a commit for staged changes.

**REQUIRED:** Follow the `git-workflow` skill for commit message format.
```text
</After>

### Improve Discoverability

<Before>

```markdown
Review code.
```text
Problem: Not discoverable in `/help`.
</Before>

<After>

```markdown
---
description: Review code for quality, security, and performance issues
argument-hint: [file-or-directory]
---

Review code in $ARGUMENTS.
```text
</After>

## Testing Skill References

When commands reference skills, verify the skill is loaded:

### Test Protocol

```markdown
Command: /project:commit (references git-workflow skill)

Test 1: Skill available
  Setup: git-workflow skill exists
  Invoke: /project:commit
  Expected: Commit follows skill conventions
  Verify: Check commit message format

Test 2: Skill missing
  Setup: Remove git-workflow skill temporarily
  Invoke: /project:commit
  Expected: Command still works (degraded)
  Verify: Note any warnings about missing skill
```text

## Testing Tool Restrictions

Verify `allowed-tools` actually restricts access:

### Test Protocol

```markdown
Command config:
---
allowed-tools: Read, Grep, Glob
---

Test 1: Allowed tool
  Ask command to read a file
  Expected: Works correctly

Test 2: Disallowed tool
  Ask command to edit a file
  Expected: Cannot use Edit tool
  Verify: Error or refusal, not silent failure
```text

## Testing Bash Execution

Verify !`command` syntax works:

### Test Protocol

```markdown
Command with:

- Branch: !`git branch --show-current`

Test:
  Invoke command
  Expected: Output includes actual branch name
  Verify: Not literal "!`git branch...`" text
```text

## Common Invocation Failures

| Failure | Cause | Fix |
|---------|-------|-----|
| "Command not found" | Wrong file location | Move to .claude/commands/ |
| Arguments not substituted | Typo in $ARGUMENTS | Check spelling |
| Bash not executed | Wrong syntax | Use !`command` not `!command` |
| Skill not loaded | Wrong skill name | Check skill exists |
| Tool restriction ignored | Frontmatter syntax | Check allowed-tools format |

## Testing Checklist

**RED Phase:**

- [ ] Documented expected invocation scenarios
- [ ] Documented expected argument handling
- [ ] Verified command doesn't exist (baseline)

**GREEN Phase:**

- [ ] Created command file in correct location
- [ ] Added description and argument-hint
- [ ] Tested invocation without arguments
- [ ] Tested invocation with single argument
- [ ] Tested invocation with multiple arguments

**REFACTOR Phase:**

- [ ] Handle missing arguments gracefully
- [ ] Add skill references if needed
- [ ] Add tool restrictions if needed
- [ ] Add bash context if helpful
- [ ] Verify `/help` shows command

**Integration:**

- [ ] Test skill references work
- [ ] Test tool restrictions work
- [ ] Test bash execution works
- [ ] Test file references work

## Quick Reference

| Test Type | What to Verify |
|-----------|----------------|
| **Invocation** | Command executes when called |
| **Arguments** | $ARGUMENTS, $1, $2 substituted correctly |
| **Missing args** | Graceful handling, not crash |
| **Skills** | Referenced skills are loaded |
| **Tools** | Restrictions enforced |
| **Bash** | !`command` output injected |
| **Files** | @path content injected |
| **Discovery** | Appears in `/help` |

## The Bottom Line

**Test commands before deploying.**

Two dimensions:

1. **Invocation**: Does `/project:command` work?
2. **Arguments**: Does `$ARGUMENTS` handle all cases?

Simple testing protocol:

1. Define scenarios (with args, without args, edge cases)
2. Write command
3. Test each scenario
4. Fix failures
5. Verify in `/help`

If you wouldn't deploy code without tests, don't deploy commands without testing invocation.
