---
name: creating-commands
description: Use when creating slash commands, editing commands, handling $ARGUMENTS, or testing command invocation before deployment
---

# Creating Commands

## Overview

**Writing commands IS Test-Driven Development applied to workflow automation.**

You write test cases (invocation scenarios), watch them fail (baseline behavior), write the command (markdown file), watch tests pass (correct execution), and refactor (improve argument handling and skill references).

**Core principle:** Commands are the interface, skills are the methodology. Commands should be thin wrappers that invoke skills for complex guidance.

**Official guidance:** For Anthropic's official command specification, see anthropic-command-docs.md

**REQUIRED:** See testing-commands-with-subagents.md for the complete testing methodology

## What is a Command?

A **command** is a manually-invoked shortcut to a stored prompt or workflow. Commands start with `/` and provide quick access to repetitive tasks.

**Commands are:** Manual triggers, workflow shortcuts, parameterized prompts

**Commands are NOT:** Auto-discovered (use skills), specialized workers (use agents), project conventions (use CLAUDE.md)

### Commands vs Skills vs Agents

| Aspect | Command | Skill | Agent |
|--------|---------|-------|-------|
| **Invocation** | Manual (`/command`) | Manual or auto | Auto or manual |
| **Purpose** | Quick workflow trigger | Reference guide/methodology | Specialized worker |
| **Has identity** | No | No | Yes ("You are...") |
| **Tool restrictions** | Yes (frontmatter) | No | Yes |
| **Model override** | Yes | No | Yes |
| **Arguments** | Yes ($ARGUMENTS) | No | Via prompt |
| **Best for** | Daily workflows | Detailed methodology | Task isolation |

### The Command-Skill Pattern

Commands reference skills for methodology:

```markdown

# .claude/commands/commit.md

---
description: Create a git commit following conventions
---

Create a commit for the staged changes.

**REQUIRED:** Follow the methodology in the `git-workflow` skill.
```text

This keeps commands thin while leveraging detailed skill guidance.

## TDD Mapping for Commands

| TDD Concept             | Command Creation                                    |
|-------------------------|-----------------------------------------------------|
| **Test case**           | Invocation scenario with arguments                  |
| **Production code**     | Command file (.md)                                  |
| **Test fails (RED)**    | Command not found or wrong behavior                 |
| **Test passes (GREEN)** | Correct execution with expected output              |
| **Refactor**            | Close loopholes in argument handling and skill compliance |
| **Write test first**    | Run baseline scenario BEFORE writing command        |
| **Watch it fail**       | Verify command doesn't exist or behaves wrong       |
| **Minimal code**        | Write command addressing specific needs             |
| **Watch it pass**       | Verify invocation produces correct result           |
| **Refactor cycle**      | Find new failure modes → fix → re-verify            |

## When to Create a Command

**Create when:**

- Task is invoked manually and repeatedly
- Task benefits from arguments/parameters
- Task is a workflow trigger (not detailed methodology)
- You want `/command` convenience

**Don't create for:**

- One-off tasks
- Auto-discovery scenarios (use skills)
- Detailed methodology documentation (use skills)
- Task isolation needs (use agents)

## Directory Structure

Commands can be placed at two levels:

```text

# Project level (recommended - version controlled)

.claude/commands/command-name.md
.claude/commands/category/command-name.md

# User level (personal commands across projects)

~/.claude/commands/command-name.md
```text

### Naming Conventions

```text

# Direct command

.claude/commands/commit.md          → /project:commit

# Categorized command

.claude/commands/git/commit.md      → /project:git:commit
.claude/commands/test/unit.md       → /project:test:unit

# User command

~/.claude/commands/standup.md       → /user:standup
```text

## Command File Structure

### Basic Command (No Frontmatter)

```markdown

# .claude/commands/review.md

Review the code in $ARGUMENTS for:

1. Code quality and readability
2. Security vulnerabilities
3. Performance issues

Provide specific, actionable feedback.
```text

Usage: `/project:review src/auth/login.ts`

### Command with Frontmatter

```markdown
---
allowed-tools: Read, Grep, Glob, Bash(git diff:*)
description: Comprehensive code review
model: claude-sonnet-4-5-20250929
argument-hint: [file-or-directory]
---

Review the code in $ARGUMENTS for quality, security, and performance.
```text

### Frontmatter Fields

| Field | Required | Description |
|-------|----------|-------------|
| `description` | No | Shown in `/help`, aids discoverability |
| `allowed-tools` | No | Restrict tools command can use |
| `model` | No | Override model for this command |
| `argument-hint` | No | Shows expected arguments in help |

## Dynamic Features

### $ARGUMENTS Placeholder

Captures everything after the command name:

```markdown
Fix the GitHub issue: $ARGUMENTS

1. Read the issue description
2. Understand the problem
3. Implement the fix
4. Write tests

```text

Usage: `/project:fix-issue 123` → "Fix the GitHub issue: 123"

**Best practice:** Structure command so it works even without arguments:

```markdown

# Good - works with or without arguments

Review the code $ARGUMENTS for quality issues.

If no specific files provided, review recently changed files:
!`git diff --name-only HEAD~1`
```text

### Positional Arguments ($1, $2, $3)

For commands with multiple parameters:

```markdown
---
argument-hint: [issue-number] [priority]
---

Fix issue #$1 with priority $2.

Priority levels:

- high: Fix immediately
- medium: Fix this sprint
- low: Add to backlog

```text

Usage: `/project:fix-issue 123 high`

### Bash Execution (!`command`)

Include live command output in the prompt:

```markdown

## Current Context

- Branch: !`git branch --show-current`
- Status: !`git status --short`
- Recent commits: !`git log --oneline -5`

## Task

Create a commit message for the staged changes.
```text

The bash commands execute when the command is invoked, injecting live output.

### File References (@path)

Include file contents:

```markdown
Review the following configuration:

- Package: @package.json
- TypeScript: @tsconfig.json
- ESLint: @.eslintrc.js

Check for inconsistencies and outdated settings.
```text

### Combining Features

```markdown
---
allowed-tools: Read, Grep, Glob, Edit, Bash(npm:*), Bash(git:*)
description: Add a new dependency with proper setup
argument-hint: [package-name]
---

## Context

- Current dependencies: @package.json
- Lock file exists: !`test -f package-lock.json && echo "yes" || echo "no"`

## Task

Add the package `$ARGUMENTS` to the project:

1. Install with npm
2. Update TypeScript config if needed
3. Add to .gitignore if it creates cache files
4. Create example usage in README

**REQUIRED:** Follow the `dependency-management` skill for version pinning conventions.
```text

## Referencing Skills

Commands should reference skills for detailed methodology:

### Pattern: Thin Command + Rich Skill

```markdown

# .claude/commands/test.md

---
description: Run tests with TDD methodology
argument-hint: [test-pattern]
---

Run tests matching: $ARGUMENTS

**REQUIRED:** Follow the `testing-conventions` skill for:

- Test file naming
- Fixture patterns
- Coverage requirements

```text

```markdown

# .claude/commands/commit.md

---
description: Create conventional commit
---

Create a commit for staged changes.

**REQUIRED METHODOLOGY:** Use the `git-workflow` skill for:

- Commit message format
- Scope conventions
- Breaking change notation

```text

### When to Reference vs Inline

**Reference a skill when:**

- Methodology is >20 lines
- Methodology is reused across commands
- Methodology needs its own testing/maintenance

**Inline when:**

- Instructions are <10 lines
- Instructions are command-specific
- No reuse anticipated

## Tool Restrictions

### Read-Only Command

```markdown
---
allowed-tools: Read, Grep, Glob, LS
description: Explore codebase without modifications
---
```text

### Git-Only Command

```markdown
---
allowed-tools: Bash(git add:*), Bash(git status:*), Bash(git commit:*), Bash(git diff:*)
description: Git operations only
---
```text

### Full Access (Default)

Omit `allowed-tools` to inherit all tools.

## Model Selection

```markdown
---
model: claude-sonnet-4-5-20250929
---
```text

| Model | Use When |
|-------|----------|
| `haiku` | Fast, simple commands (file listing, quick checks) |
| `sonnet` | Balanced (most commands) |
| `opus` | Complex reasoning (architecture, planning) |

## The Iron Law (Same as TDD)

```text
NO COMMAND WITHOUT A FAILING TEST FIRST
```text

This applies to NEW commands AND EDITS to existing commands.

Write command before testing? Delete it. Start over.
Edit command without testing? Same violation.

**No exceptions:**

- Not for "simple commands"
- Not for "just changing the description"
- Not for "minor argument updates"
- Don't keep untested changes as "reference"
- Don't "adapt" while running tests
- Delete means delete

**Test scenarios:**

- With arguments
- Without arguments
- With wrong arguments
- Edge cases

## Testing Commands

**Note:** Commands don't need pressure testing like discipline-enforcing skills. Commands are workflow triggers, not rules to follow under stress. Focus testing on:

- **Invocation**: Does `/project:command` execute?
- **Arguments**: Does `$ARGUMENTS` handle all cases (present, missing, multiple)?
- **Skill loading**: Does the referenced skill get used?

### Invocation Testing

Does the command execute correctly?

```markdown
Test: /project:commit
Expected: Creates commit with conventional message
Actual: [Run and observe]
```text

### Argument Testing

Do arguments work correctly?

```markdown
Test: /project:fix-issue 123
Expected: $ARGUMENTS = "123"

Test: /project:fix-issue 123 high
Expected: $1 = "123", $2 = "high"

Test: /project:fix-issue
Expected: Command handles missing argument gracefully
```text

### Skill Reference Testing

Does the command use referenced skills?

```markdown
Test: /project:commit (with git-workflow skill available)
Expected: Follows commit conventions from skill
Verify: Commit message format matches skill specification
```text

## Common Rationalizations for Skipping Testing

| Excuse | Reality |
|--------|---------|
| "Command is simple" | Simple commands still need argument testing |
| "It's just a prompt" | Prompts can have edge cases |
| "I'll test when problems emerge" | Problems = broken workflows. Test first. |
| "Skill handles the complexity" | Still need to verify skill is loaded |
| "Too tedious to test" | Testing is less tedious than debugging broken workflows in production. |
| "I'm confident it's good" | Overconfidence guarantees argument edge cases. Test anyway. |
| "No time to test" | Deploying untested command wastes more time fixing it later. |
| "I copied from a working command" | Your changes may break it. Test anyway. |

## Command Categories

### Workflow Commands

Multi-step processes triggered manually:

```markdown

# .claude/commands/workflows/feature.md

---
description: Complete feature development workflow
argument-hint: [feature-description]
---

Implement feature: $ARGUMENTS

1. Create feature branch
2. Write tests first (TDD)
3. Implement feature
4. Run all tests
5. Create PR

**REQUIRED:** Follow `tdd` skill for test-first methodology.
```text

### Tool Commands

Single-purpose utilities:

```markdown

# .claude/commands/tools/deps-check.md

---
allowed-tools: Bash(npm:*), Read
description: Check for outdated dependencies
---

Check for outdated dependencies:
!`npm outdated`

Summarize which packages need updates and their breaking change risk.
```text

### Context Commands

Commands that provide context:

```markdown

# .claude/commands/context/standup.md

---
description: Generate standup summary
---

## Yesterday's Work

!`git log --oneline --since="yesterday" --author="$(git config user.email)"`

## Changed Files

!`git diff --stat HEAD~5`

Summarize this as a standup update.
```text

## Common Mistakes

| Mistake | Symptom | Fix |
|---------|---------|-----|
| Fat command | >50 lines of methodology | Move to skill, reference from command |
| No argument handling | Breaks without args | Add fallback with !`git diff` or default |
| Hardcoded paths | Not reusable across projects | Use $ARGUMENTS |
| Missing description | Doesn't appear in `/help` | Add frontmatter with description |
| No skill reference | Duplicates methodology | Reference skill for detailed guidance |

## Anti-Patterns

### ❌ Fat Command (Should Be Skill)

```markdown

# BAD: 200 lines of methodology in a command

---
description: Code review
---

## Code Review Methodology

### Step 1: Understand Context

[50 lines of detailed guidance...]

### Step 2: Security Analysis

[50 lines of detailed guidance...]
```text

**Fix:** Move methodology to skill, reference from command.

### ❌ No Argument Handling

```markdown

# BAD: Breaks without argument

Review $ARGUMENTS for issues.
```text

**Fix:** Handle missing arguments gracefully:

```markdown

# GOOD

Review $ARGUMENTS for issues.

If no files specified, review staged changes:
!`git diff --cached --name-only`
```text

### ❌ Hardcoded Values

```markdown

# BAD: Hardcoded paths

Review /src/components/ for issues.
```text

**Fix:** Use arguments:

```markdown

# GOOD

Review $ARGUMENTS for issues.
Default: src/
```text

### ❌ Missing Description

```markdown

# BAD: No frontmatter

Do something with code.
```text

**Fix:** Add description for `/help` discoverability:

```markdown
---
description: Review code for quality issues
argument-hint: [file-or-directory]
---
```text

## Command Creation Checklist

**RED Phase - Write Failing Test:**

- [ ] Verify command doesn't already exist
- [ ] Document expected invocation patterns and argument handling
- [ ] Try invoking the command — confirm it fails or behaves wrong (baseline)
- [ ] Document the gap between current and expected behavior

**GREEN Phase - Write Minimal Command:**

- [ ] Create file in correct location (.claude/commands/)
- [ ] Add description in frontmatter
- [ ] Add argument-hint if command takes arguments
- [ ] Handle $ARGUMENTS gracefully (with and without)
- [ ] Reference skills for detailed methodology
- [ ] Add tool restrictions if needed
- [ ] Add model override if needed
- [ ] Add bash context (!`command`) if helpful
- [ ] Add file references (@path) if helpful
- [ ] Test invocation works
- [ ] Test arguments work

**REFACTOR Phase - Close Loopholes:**

- [ ] Verify arguments don't break with unexpected input
- [ ] Verify referenced skills are actually followed
- [ ] Verify tool restrictions prevent unintended actions
- [ ] Test edge cases (missing args, wrong args, extra args)
- [ ] Re-test until command behaves correctly in all scenarios

**Quality Checks:**

- [ ] Command is thin (methodology in skills)
- [ ] Description is clear and discoverable
- [ ] Arguments are documented (argument-hint)
- [ ] Missing arguments handled gracefully
- [ ] Skill references are correct

**Deployment:**

- [ ] Place in .claude/commands/ for project scope
- [ ] Commit to version control
- [ ] Verify appears in `/help`

## Organization Best Practices

### Recommended Structure

```text
.claude/commands/
├── workflows/           # Multi-step processes
│   ├── feature.md
│   ├── release.md
│   └── hotfix.md
├── tools/               # Single-purpose utilities
│   ├── deps-check.md
│   ├── lint-fix.md
│   └── type-check.md
├── git/                 # Git operations
│   ├── commit.md
│   ├── pr.md
│   └── rebase.md
├── test/                # Testing commands
│   ├── unit.md
│   ├── integration.md
│   └── coverage.md
└── help.md              # List all commands
```text

### Help Command

Create a master help command:

```markdown

# .claude/commands/help.md

---
description: List all available project commands
---

## Available Commands

### Workflows

- `/project:workflows:feature [description]` - Full feature development
- `/project:workflows:release [version]` - Release workflow
- `/project:workflows:hotfix [issue]` - Hotfix workflow

### Tools

- `/project:tools:deps-check` - Check outdated dependencies
- `/project:tools:lint-fix` - Auto-fix lint issues

### Git

- `/project:git:commit` - Create conventional commit
- `/project:git:pr [title]` - Create pull request

### Testing

- `/project:test:unit [pattern]` - Run unit tests
- `/project:test:coverage` - Generate coverage report

For detailed help on any command, read its file in `.claude/commands/`.
```text

## Full Command Template

```markdown
---
allowed-tools: [Tool1, Tool2, Bash(command:*)]
description: [What this command does - shown in /help]
model: [haiku|sonnet|opus]
argument-hint: [arg1] [arg2]
---

## Context

[Optional: Live context from bash or file references]

- Current branch: !`git branch --show-current`
- Config: @config/settings.json

## Task

[Main instruction using $ARGUMENTS or $1, $2]

Do something with: $ARGUMENTS

[Optional: Handle missing arguments]
If no arguments provided, use default: [default behavior]

## Methodology

**REQUIRED:** Follow the `[skill-name]` skill for:

- [Aspect 1 the skill covers]
- [Aspect 2 the skill covers]

## Output

[Optional: Expected output format]
```text

## Worked Example

For a complete walkthrough of argument handling TDD — including 4 test scenarios, 3 iterations from minimal to production command, and key lessons — see examples/ARGUMENT_HANDLING_TESTING.md

## The Bottom Line

**Commands are the interface, skills are the methodology.**

Keep commands thin - they trigger workflows and reference skills for detailed guidance. This separation gives you quick invocation (`/commit`) with rich methodology (from `git-workflow` skill).

Same TDD cycle applies: Write failing test (baseline) → Write minimal command → Close loopholes.
Same benefits: Better quality, correct invocation, bulletproof argument handling.

If you follow TDD for code, follow it for commands. It's the same discipline applied to workflow automation.
