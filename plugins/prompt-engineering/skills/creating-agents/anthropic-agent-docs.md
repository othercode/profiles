# Sub-agents

> Configure specialized sub-agents that Claude can delegate to for specific tasks.

Sub-agents are specialized assistants configured for specific task types. They allow Claude to delegate tasks to purpose-built agents with constrained tools, specific models, or custom behaviors.

For conceptual background on how sub-agents fit into the Claude Code architecture, see the [Claude Code overview](/en/docs/claude-code/overview).

## Why use sub-agents?

Sub-agents enable **task isolation** and **specialization**:

- **Tool restrictions**: Limit what tools an agent can use (e.g., read-only exploration)
- **Model selection**: Use faster/cheaper models for simple tasks, more capable models for complex reasoning
- **Focused context**: Agent receives only relevant instructions, not the full conversation
- **Permission boundaries**: Control what actions an agent can take without user approval

**Example use cases:**

- Code exploration agent with read-only tools
- Documentation writer with opus for quality
- Fast file finder using haiku for speed
- Infrastructure agent with controlled bash access

## Sub-agent structure

Sub-agents are defined as markdown files with YAML frontmatter:

```markdown
---
name: agent-name
description: When Claude should delegate to this agent
tools: Read, Grep, Glob
model: sonnet
---

# Agent Name

You are a specialist at [domain]. Your job is to [responsibility].

## Instructions

[Agent-specific instructions here]
```text

### Frontmatter fields

| Field | Required | Description |
|-------|----------|-------------|
| `name` | Yes | Lowercase letters and hyphens only (e.g., `code-explorer`) |
| `description` | Yes | When Claude should delegate - critical for auto-delegation |
| `tools` | No | Comma-separated list of tools agent can use. Inherits all if omitted |
| `disallowedTools` | No | Tools to explicitly deny |
| `model` | No | `sonnet`, `opus`, `haiku`, or `inherit` (default: inherit from parent) |
| `permissionMode` | No | Controls permission prompts (see below) |
| `skills` | No | Skills to preload into agent context |
| `hooks` | No | Lifecycle hooks scoped to this agent |

### Tool restrictions

Restrict agent capabilities by listing allowed tools:

```yaml

# Read-only agent

tools: Read, Grep, Glob, LS

# Modification agent without shell

tools: Read, Grep, Glob, LS, Edit, Write

# Full capability (inherits all)
# Omit tools field entirely
```text

Or deny specific tools:

```yaml
disallowedTools: Bash, NotebookEdit
```text

### Model selection

| Model | Use case | Characteristics |
|-------|----------|-----------------|
| `haiku` | Fast, simple tasks | Low cost, fast response |
| `sonnet` | Balanced tasks | Good reasoning, moderate cost |
| `opus` | Complex reasoning | Best quality, higher cost |
| `inherit` | Match parent | Uses same model as caller |

```yaml

# Fast exploration

model: haiku

# Balanced analysis

model: sonnet

# Complex planning

model: opus
```text

### Permission modes

| Mode | Behavior |
|------|----------|
| `default` | Normal permission prompts |
| `acceptEdits` | Auto-accept file edits |
| `dontAsk` | Skip confirmations |
| `bypassPermissions` | Skip all prompts |
| `plan` | Planning only, no execution |

**Security note:** Use restrictive modes by default. Only escalate when necessary.

## Sub-agent locations

Sub-agents can be placed at multiple levels (checked in order):

```text

# 1. CLI flag (session only)

--agents ./my-agents/

# 2. Project level

.claude/agents/agent-name.md

# 3. User level

~/.claude/agents/agent-name.md

# 4. Plugin level

plugin/agents/agent-name.md
```text

Higher priority locations override lower ones if names conflict.

## How delegation works

1. Claude receives a task
2. Claude reads available agent descriptions
3. If an agent matches, Claude delegates via the Task tool:
```text
   Task(subagent_type="agent-name", prompt="...")
```text
4. Agent executes with its configured tools and model
5. Agent returns result to Claude
6. Claude summarizes result for user

### Triggering delegation

The `description` field is critical for auto-delegation. Write it to answer: "When should Claude delegate to this agent?"

```yaml

# Good: Clear trigger conditions

description: Analyzes codebase implementation details. Use when you need
  detailed information about how specific components work.

# Good: Comparison to alternatives

description: Locates files and components. Use if you find yourself wanting
  to use Grep/Glob/LS more than once.

# Bad: Vague

description: Helps with code
```text

**Proactive delegation**: Include "proactively" to trigger without explicit request:

```yaml
description: Use this agent proactively when exploring unfamiliar codebases
```text

## Writing effective agent instructions

### Role statement

Start with a clear identity:

```markdown
You are a specialist at [domain]. Your job is to [core responsibility].
```text

### Scope boundaries (CRITICAL rules)

Explicitly define what the agent must NOT do:

```markdown

## CRITICAL: YOUR ONLY JOB IS TO [SCOPE]

- DO NOT suggest improvements or changes
- DO NOT critique the implementation
- DO NOT perform root cause analysis
- ONLY describe what exists and how it works

```text

### Output format

Define expected output structure:

```markdown

## Output Format

Structure your analysis like this:

## Analysis: [Component Name]

### Overview

[2-3 sentence summary]

### Entry Points

- `file.js:45` - Description

### Key Patterns

- **Pattern Name**: Where used and why

```text

### Guidelines and anti-patterns

```markdown

## Important Guidelines

- Always include file:line references
- Read files thoroughly before making statements
- Be precise about function names

## What NOT to Do

- Don't guess about implementation
- Don't skip error handling
- Don't make recommendations

```text

## Example agents

### Read-only code explorer

```markdown
---
name: code-explorer
description: Explores codebase structure and finds relevant files. Use when
  you need to understand code organization or find files by pattern.
tools: Read, Grep, Glob, LS
model: haiku
---

You are a specialist at finding code in large codebases. Your job is to
locate relevant files and explain their organization.

## CRITICAL: Read-only exploration only

- DO NOT suggest changes or improvements
- DO NOT analyze code quality
- ONLY find and describe file locations

## Output Format

## Files for [Topic]

### Implementation

- `path/file.js` - Purpose

### Tests

- `path/file.test.js` - What it tests

### Configuration

- `config/file.json` - Settings it contains

```text

### Documentation writer

```markdown
---
name: doc-writer
description: Writes technical documentation. Use when creating README files,
  API documentation, or code comments.
tools: Read, Grep, Glob, Write
model: opus
---

You are a technical writer. Your job is to create clear, accurate
documentation that helps developers understand and use code.

## CRITICAL: Documentation focus

- DO NOT modify code
- DO NOT suggest code changes
- ONLY write documentation files

## Guidelines

- Write for the target audience (developers, users, operators)
- Include working code examples
- Explain the "why" not just the "what"

```text

### Infrastructure operator

```markdown
---
name: infra-ops
description: Diagnoses and configures infrastructure. Use for server
  configuration, networking, or deployment tasks.
tools: Read, Write, Edit, Bash, Grep, Glob
model: sonnet
permissionMode: default
---

You are an infrastructure specialist. Your job is to diagnose issues and
configure systems safely.

## CRITICAL: Safety first

- DO NOT run destructive commands without confirmation
- DO NOT modify production systems without explicit approval
- ALWAYS explain what commands will do before running them
- ALWAYS provide rollback instructions

## Process

1. Diagnose: Gather information before acting
2. Explain: State what you'll do and why
3. Confirm: Get approval for risky operations
4. Execute: Run commands with error handling
5. Verify: Confirm changes worked correctly

```text

## Preloading skills

Load skills into agent context:

```yaml
skills: managing-server-lifecycle, testing-conventions
```text

The agent will have access to these skills' content during execution.

## Agent hooks

Define lifecycle hooks scoped to the agent:

```yaml
hooks:
  PreToolCall:
    - command: echo "Agent calling tool: $TOOL_NAME"
```text

See [Hooks documentation](/en/docs/claude-code/hooks) for available hooks.

## Testing agents

Before deploying, verify:

1. **Delegation**: Does Claude delegate correctly?
2. **Capability**: Does the agent do its job?
3. **Boundaries**: Does the agent stay in scope?
4. **Tool restrictions**: Do restrictions actually work?

### Testing delegation

```markdown
Give Claude a task that should trigger your agent.
Verify the Task tool is called with correct subagent_type.
```text

### Testing boundaries

```markdown
Ask agent to do something outside its role.
Verify it declines or deflects appropriately.
```text

## Best practices

### DO:

- Define clear scope boundaries
- Use appropriate model for task complexity
- Restrict tools to minimum needed
- Include concrete output examples
- Test before deploying

### DON'T:

- Create over-scoped "do everything" agents
- Use opus for simple tasks (waste)
- Use haiku for complex reasoning (quality loss)
- Skip the CRITICAL rules section
- Assume Claude will always delegate correctly

## Common patterns

### Documentarian pattern

Agent explains without critiquing. Read-only tools, clear "ONLY describe" rules.

### Executor pattern

Agent takes actions safely. Includes confirmation steps, rollback guidance.

### Observer pattern

Background agent for monitoring. Uses haiku, writes to logs/files.

### Expert pattern

Deep domain knowledge agent. Uses opus, comprehensive instructions.

## Troubleshooting

**Agent not triggered:**

- Check description clarity
- Verify agent file is in correct location
- Check for name conflicts

**Agent exceeds scope:**

- Add more explicit CRITICAL rules
- List specific anti-patterns
- Test boundary scenarios

**Wrong model being used:**

- Verify `model` field in frontmatter
- Check for `inherit` default behavior

## Next steps

- [Skills documentation](/en/docs/claude-code/skills) - Create skills agents can use
- [Hooks documentation](/en/docs/claude-code/hooks) - Add lifecycle hooks
- [Claude Code overview](/en/docs/claude-code/overview) - Understand the architecture
