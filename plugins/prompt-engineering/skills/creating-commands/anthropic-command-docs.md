<!-- Last reviewed: 2026-02-16 | Source: https://code.claude.com/docs/en/skills -->
<!-- Commands have been merged into skills. This file retains command-specific guidance. -->
<!-- If this file is more than 90 days stale, verify content against the source URL before relying on it -->

# Slash Commands

> Create custom slash commands to automate repetitive workflows in Claude Code.

**Note (2026-02):** Custom slash commands have been merged into skills. A file at `.claude/commands/review.md` and a skill at `.claude/skills/review/SKILL.md` both create `/review` and work the same way. Your existing `.claude/commands/` files keep working. Skills add optional features: a directory for supporting files, frontmatter to control invocation, and Claude can load them automatically when relevant.

Slash commands provide quick access to stored prompts and procedures. Built-in commands like `/help`, `/clear`, and `/compact` are always available. You can create custom commands for your specific workflows.

## Built-in Commands

| Command | Description |
|---------|-------------|
| `/help` | Display all available commands including custom ones |
| `/clear` | Clear conversation history and start fresh |
| `/compact` | Compress conversation history to save context |
| `/config` | Configure Claude Code settings interactively |
| `/allowed-tools` | Set up tool permissions |
| `/hooks` | Configure automation hooks |
| `/mcp` | Manage Model Context Protocol servers |
| `/agents` | Create and manage subagents |
| `/vim` | Enable vim-style editing |
| `/terminal-setup` | Install keyboard shortcuts |
| `/context` | Check context usage and skill loading |

## Creating Custom Commands

Custom commands are markdown files stored in designated directories.

### File Locations

| Scope | Location | Invocation |
|-------|----------|------------|
| **Project** | `.claude/commands/name.md` | `/project:name` |
| **Project (categorized)** | `.claude/commands/category/name.md` | `/project:category:name` |
| **User** | `~/.claude/commands/name.md` | `/user:name` |

Project commands are version-controlled and shared with team members. User commands are personal and available across all projects.

### Basic Command

Create `.claude/commands/review.md`:

```markdown
Review the code for quality issues, security vulnerabilities, and performance problems.
Provide specific, actionable feedback organized by priority.
```text

Invoke with: `/project:review`

### Command with Frontmatter

Create `.claude/commands/security-scan.md`:

```markdown
---
allowed-tools: Read, Grep, Glob
description: Scan codebase for security vulnerabilities
model: claude-sonnet-4-5-20250929
argument-hint: [directory]
---

Scan $ARGUMENTS for security vulnerabilities including:

- SQL injection risks
- XSS vulnerabilities
- Exposed credentials
- Insecure configurations

Report findings with severity levels and remediation steps.
```text

Invoke with: `/project:security-scan src/`

## Frontmatter Fields

| Field | Type | Description |
|-------|------|-------------|
| `description` | string | Shown in `/help` output for discoverability |
| `allowed-tools` | string | Comma-separated list of tools command can use |
| `model` | string | Model override (`haiku`, `sonnet`, `opus`, or full model ID) |
| `argument-hint` | string | Shows expected arguments in help (e.g., `[file] [options]`) |
| `disable-model-invocation` | boolean | Set `true` to prevent Claude from auto-triggering (skills only) |
| `user-invocable` | boolean | Set `false` to hide from `/` menu (skills only) |
| `context` | string | Set to `fork` to run in a forked subagent context (skills only) |
| `agent` | string | Subagent type when `context: fork` is set (skills only) |
| `hooks` | object | Lifecycle hooks scoped to this skill (skills only) |

### Tool Restriction Syntax

```yaml

# Allow specific tools

allowed-tools: Read, Grep, Glob, LS

# Allow bash with specific command prefixes

allowed-tools: Bash(git add:*), Bash(git commit:*), Bash(npm:*)

# Combine tools and restricted bash

allowed-tools: Read, Edit, Bash(git:*), Bash(npm test:*)
```text

## Dynamic Content

### Arguments ($ARGUMENTS)

Capture everything after the command name:

```markdown
Fix the GitHub issue: $ARGUMENTS

1. Fetch issue details
2. Understand the problem
3. Implement the fix
4. Write tests

```text

Usage: `/project:fix-issue 123` → "$ARGUMENTS" becomes "123"

### Positional Arguments ($1, $2, $3)

For commands with multiple parameters:

```markdown
---
argument-hint: [issue-number] [priority]
---

Fix issue #$1 with priority level: $2
```text

Usage: `/project:fix 123 high` → "$1" is "123", "$2" is "high"

### Indexed Arguments ($ARGUMENTS[N])

Access specific arguments by 0-based index:

```markdown
Migrate the $ARGUMENTS[0] component from $ARGUMENTS[1] to $ARGUMENTS[2].
```text

Or use the `$N` shorthand (equivalent):

```markdown
Migrate the $0 component from $1 to $2.
```text

Usage: `/project:migrate SearchBar React Vue` → `$0`="SearchBar", `$1`="React", `$2`="Vue"

### Session ID (${CLAUDE_SESSION_ID})

Access the current session ID for logging or correlation:

```markdown
Log activity to logs/${CLAUDE_SESSION_ID}.log:

$ARGUMENTS
```text

### Bash Execution (!`command`)

Include live command output:

```markdown

## Current State

- Branch: !`git branch --show-current`
- Status: !`git status --short`
- Changes: !`git diff --stat`

## Task

Based on the above context, create an appropriate commit message.
```text

Bash commands execute when the command is invoked, and their output is injected into the prompt.

### File References (@path)

Include file contents:

```markdown
Review the configuration files:

- Package config: @package.json
- TypeScript config: @tsconfig.json
- ESLint config: @.eslintrc.js

Check for inconsistencies between these configurations.
```text

File contents are read and injected when the command is invoked.

## Organization

### Directory Structure

Organize commands in subdirectories:

```text
.claude/commands/
├── git/
│   ├── commit.md      → /project:git:commit
│   ├── pr.md          → /project:git:pr
│   └── rebase.md      → /project:git:rebase
├── test/
│   ├── unit.md        → /project:test:unit
│   └── e2e.md         → /project:test:e2e
└── review.md          → /project:review
```text

### Discoverability

Run `/help` to see all available commands including custom ones.

Create a help command listing your project's commands:

```markdown

# .claude/commands/commands.md

---
description: List all project commands
---

## Available Commands

- `/project:git:commit` - Create conventional commit
- `/project:git:pr [title]` - Create pull request
- `/project:test:unit [pattern]` - Run unit tests
- `/project:review [file]` - Code review

```text

## Context Budget

Command descriptions are loaded into context for discoverability. If you have many commands, they may exceed the character budget (default 15,000 characters).

Run `/context` to check for warnings about excluded commands.

To reduce context usage:

- Keep descriptions concise
- Remove unused commands
- Use subdirectories to organize (only top-level descriptions load)

## Commands and Skills

Custom slash commands have been merged with skills. Both approaches work:

| Approach | Location | Result |
|----------|----------|--------|
| Command | `.claude/commands/review.md` | Creates `/project:review` |
| Skill | `.claude/skills/review/SKILL.md` | Creates `/review` |

Skills add optional features:

- Directory for supporting files
- Frontmatter to control invocation (manual vs auto)
- Claude can load them automatically when relevant

Your existing `.claude/commands/` files continue working unchanged.

## Examples

### Git Commit Command

```markdown

# .claude/commands/git/commit.md

---
allowed-tools: Bash(git:*)
description: Create a conventional commit
---

## Current Changes

!`git status --short`

## Diff

!`git diff --cached`

## Task

Create a commit with a conventional commit message:

- type(scope): description
- Types: feat, fix, docs, style, refactor, test, chore
- Keep subject line under 72 characters

```text

### Code Review Command

```markdown

# .claude/commands/review.md

---
allowed-tools: Read, Grep, Glob
description: Comprehensive code review
argument-hint: [file-or-directory]
---

## Target

Review: $ARGUMENTS

If no target specified, review staged changes:
!`git diff --cached --name-only`

## Review Checklist

1. **Correctness**: Does the code do what it's supposed to?
2. **Security**: Any vulnerabilities (injection, XSS, auth issues)?
3. **Performance**: Any obvious inefficiencies?
4. **Readability**: Is the code clear and well-documented?
5. **Testing**: Are there adequate tests?

Provide specific feedback with file:line references.
```text

### Test Runner Command

```markdown

# .claude/commands/test/run.md

---
allowed-tools: Bash, Read
description: Run tests with pattern matching
argument-hint: [test-pattern]
---

Run tests matching: $ARGUMENTS

1. Detect test framework (Jest, pytest, etc.)
2. Run matching tests
3. If failures occur:
   - Analyze error messages
   - Suggest fixes
   - Offer to implement fixes
```text

### Dependency Check Command

```markdown

# .claude/commands/deps.md

---
allowed-tools: Bash(npm:*), Read
description: Check and update dependencies
---

## Current Dependencies

@package.json

## Outdated Packages

!`npm outdated`

## Task

1. List packages that need updates
2. Categorize by risk (major/minor/patch)
3. Identify potential breaking changes
4. Recommend update order

```text

## SDK Usage

Custom commands are available through the Claude Agent SDK:

```typescript
import { query } from "@anthropic-ai/claude-agent-sdk";

// Use a custom command
for await (const message of query({
  prompt: "/project:review src/auth/",
  options: { maxTurns: 3 }
})) {
  if (message.type === "assistant") {
    console.log(message.message);
  }
}

// List available commands
for await (const message of query({
  prompt: "Hello",
  options: { maxTurns: 1 }
})) {
  if (message.type === "system" && message.subtype === "init") {
    console.log("Commands:", message.slash_commands);
  }
}
```text

## Best Practices

### DO:

- Add `description` for discoverability in `/help`
- Add `argument-hint` when command takes arguments
- Handle missing arguments gracefully
- Use tool restrictions for safety
- Organize in subdirectories for large command sets
- Version control project commands

### DON'T:

- Put extensive methodology in commands (use skills instead)
- Hardcode paths or values that should be arguments
- Create commands without descriptions
- Assume arguments will always be provided
- Use overly broad tool permissions

## See Also

- [Skills documentation](/en/docs/claude-code/skills) - Auto-discovered capabilities
- [Agents documentation](/en/docs/claude-code/agents) - Specialized workers
- [Hooks documentation](/en/docs/claude-code/hooks) - Automation triggers
- [SDK slash commands](/en/docs/agent-sdk/slash-commands) - Programmatic usage
