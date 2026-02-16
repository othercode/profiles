---
name: creating-agents
description: Use when creating new agents, editing existing agents, or verifying agents work before deployment
---

# Creating Agents

## Overview

**Writing agents IS Test-Driven Development applied to AI assistant configuration.**

You write test cases (delegation scenarios with subagents), watch them fail (baseline behavior), write the agent (configuration file), watch tests pass (correct delegation and behavior), and refactor (close loopholes).

**Core principle:** If you didn't watch Claude fail to delegate correctly or the agent fail without the configuration, you don't know if the agent teaches the right thing.

**Official guidance:** For Anthropic's official agent authoring specification, see anthropic-agent-docs.md. This document provides the complete frontmatter specification and configuration options.

**REQUIRED:** See testing-agents-with-subagents.md for the complete testing methodology:
- How to write delegation scenarios
- Scope pressure testing
- Closing loopholes systematically
- Meta-testing techniques

**Worked example:** See examples/SCOPE_BOUNDARY_TESTING.md for a full test campaign testing agent scope configurations

## What is an Agent?

An **agent** is a specialized AI assistant configured for specific task types. Agents have their own identity, tool access, and behavioral boundaries.

**Agents are:** Reusable specialists for specific domains, task isolation, tool-restricted workers

**Agents are NOT:** Skills (reference guides), CLAUDE.md content (project conventions), one-off solutions

### Agents vs Skills vs CLAUDE.md

| Aspect | Agent | Skill | CLAUDE.md |
|--------|-------|-------|-----------|
| **Purpose** | Specialized worker with identity | Reference guide/technique | Project conventions |
| **Has identity** | Yes ("You are a...") | No | No |
| **Tool restrictions** | Yes | No | No |
| **Model selection** | Yes | No | No |
| **Loads into** | Subagent context | Main context | Every conversation |
| **Use when** | Task isolation needed | Technique documentation | Project-specific rules |

## TDD Mapping for Agents

| TDD Concept             | Agent Creation                                      |
|-------------------------|-----------------------------------------------------|
| **Test case**           | Delegation scenario with Task tool                  |
| **Production code**     | Agent file (.md)                                    |
| **Test fails (RED)**    | Claude delegates incorrectly or agent misbehaves   |
| **Test passes (GREEN)** | Correct delegation AND agent produces correct output|
| **Refactor**            | Close loopholes while maintaining compliance        |
| **Write test first**    | Run baseline scenario BEFORE writing agent          |
| **Watch it fail**       | Document exact failures (wrong delegation, scope creep) |
| **Minimal code**        | Write agent addressing those specific failures      |
| **Watch it pass**       | Verify correct delegation and output                |
| **Refactor cycle**      | Find new issues → fix → re-verify                   |

## When to Create an Agent

**Create when:**
- Task requires tool restrictions (read-only exploration)
- Task benefits from isolated context (background processing)
- Specialized persona improves output quality
- Task type recurs frequently across projects
- Model selection matters (haiku for speed, opus for reasoning)

**Don't create for:**
- One-off tasks
- Simple techniques (use skills)
- Project-specific conventions (use CLAUDE.md)
- Tasks that don't benefit from isolation

## Agent Types

### Read-Only Agents
Exploration and analysis without modification capability.

**Examples:** codebase-analyzer, codebase-locator, codebase-pattern-finder

**Characteristics:**
- Tools: Read, Grep, Glob, LS (no Edit, Write, Bash)
- Model: Often sonnet or haiku for speed
- Role: Documentarian, not consultant

### Action Agents
Can modify files or execute commands within defined boundaries.

**Examples:** infra-ops, refactoring-expert

**Characteristics:**
- Tools: Include Edit, Write, Bash as needed
- Model: sonnet or opus for judgment
- Role: Executor with safety boundaries

### Background Agents
Run asynchronously for observation or long-running tasks.

**Examples:** observer

**Characteristics:**
- run_mode: background
- Model: haiku for cost efficiency
- Role: Monitor or processor

## Directory Structure

Agents can be placed at three levels (priority order):

```
# 1. CLI flag (session only)
--agents ./my-agents/

# 2. Project level (recommended for project-specific agents)
.claude/agents/agent-name.md

# 3. User level (for personal agents across projects)
~/.claude/agents/agent-name.md

# 4. Plugin level (for shared toolkit agents)
plugin/agents/agent-name.md
```

## Agent Role Definition

**Every agent needs a clear role that defines WHO it is and WHAT it must NOT do.**

### Role Statement Pattern

```markdown
You are a [specialist/expert] at [domain]. Your job is to [core responsibility].

## CRITICAL: [SCOPE ENFORCEMENT]
- DO NOT [boundary 1]
- DO NOT [boundary 2]
- ONLY [permitted action]
```

### Why Roles Matter

1. **Prevents scope creep** - Agent knows its boundaries
2. **Sets clear expectations** - Output format is predictable
3. **Enables focused results** - No dilution of expertise
4. **Makes testing easier** - Clear success criteria

### Role Types

| Role Type | Example | Key Characteristic |
|-----------|---------|-------------------|
| Documentarian | codebase-analyzer | Explain, don't critique |
| Locator | codebase-locator | Find, don't analyze |
| Executor | infra-ops | Take action, with safety |
| Observer | observer | Watch, detect patterns |
| Expert | refactoring-expert | Deep domain knowledge |

### Anti-Pattern: Undefined Role

```markdown
# ❌ BAD: No clear identity or boundaries
---
name: helper
description: Helps with things
---

Help the user with whatever they need.
```

```markdown
# ✅ GOOD: Clear identity and boundaries
---
name: codebase-analyzer
description: Analyzes codebase implementation details
tools: Read, Grep, Glob, LS
model: sonnet
---

You are a specialist at understanding HOW code works. Your job is to analyze
implementation details and explain technical workings with precise file:line references.

## CRITICAL: YOUR ONLY JOB IS TO DOCUMENT AND EXPLAIN
- DO NOT suggest improvements or changes
- DO NOT critique the implementation
- ONLY describe what exists and how it works
```

## Agent File Structure

### Frontmatter (YAML)

| Field | Required | Description |
|-------|----------|-------------|
| `name` | Yes | Lowercase letters and hyphens only |
| `description` | Yes | When Claude should delegate (critical for auto-delegation) |
| `tools` | No | Tools agent can use (inherits all if omitted) |
| `disallowedTools` | No | Tools to deny |
| `model` | No | `sonnet`, `opus`, `haiku`, or `inherit` (default: inherit) |
| `permissionMode` | No | `default`, `acceptEdits`, `dontAsk`, `bypassPermissions`, `plan` |
| `skills` | No | Skills to preload into agent context |
| `hooks` | No | Lifecycle hooks scoped to this agent |

### System Prompt Structure

```markdown
---
name: agent-name
description: Use when [specific triggering conditions]
tools: [Tool1, Tool2]
model: sonnet
---

[Role statement - "You are a..."]

## CRITICAL: [SCOPE ENFORCEMENT]
- DO NOT [boundary 1]
- DO NOT [boundary 2]
- ONLY [permitted action]

## Core Responsibilities
1. **[Responsibility 1]**
   - Detail
   - Detail

2. **[Responsibility 2]**
   - Detail

## Strategy/Process

### Step 1: [Name]
- Action
- Action

### Step 2: [Name]
- Action

## Output Format

Structure your output like this:

```
## [Section]
[Content pattern]
```

## Important Guidelines
- Guideline 1
- Guideline 2

## What NOT to Do
- Anti-pattern 1
- Anti-pattern 2

## REMEMBER: [Key constraint summary]
```

## Claude Search Optimization (CSO) for Agents

**Critical for discovery:** Claude reads `description` to decide when to delegate.

### Description Best Practices

**Format:** Focus on WHEN to delegate, not what the agent does.

```yaml
# ❌ BAD: Describes what, not when
description: Analyzes code and finds patterns

# ❌ BAD: Too vague
description: Helps with code

# ✅ GOOD: Clear delegation trigger
description: Analyzes codebase implementation details. Call when you need detailed
  information about specific components.

# ✅ GOOD: Comparison to alternatives
description: Locates files and components. Use if you find yourself wanting to use
  Grep/Glob/LS more than once.
```

### "Use Proactively" Trigger

For agents that should auto-activate, include proactive language:

```yaml
description: Use this agent proactively when [condition]
```

### Keyword Coverage

Include words Claude would search for:
- Task types: "analyze", "find", "debug", "configure"
- Domains: "infrastructure", "testing", "documentation"
- Tools: Specific tool names the agent uses

## Tool Selection Matrix

| Agent Type | Recommended Tools | Rationale |
|------------|------------------|-----------|
| Read-only exploration | Read, Grep, Glob, LS | No modification risk |
| Code modification | + Edit, Write | Controlled changes |
| System operations | + Bash | Shell access needed |
| Full capability | (inherit all) | Maximum flexibility |

### Tool Restriction Examples

```yaml
# Read-only agent
tools: Read, Grep, Glob, LS

# Modification agent without shell
tools: Read, Grep, Glob, LS, Edit, Write

# Agent with specific tool denied
disallowedTools: Bash, NotebookEdit
```

## Model Selection

| Model | Use When | Cost | Speed |
|-------|----------|------|-------|
| `haiku` | Fast, simple tasks (search, exploration) | Low | Fast |
| `sonnet` | Balanced (analysis, review) | Medium | Medium |
| `opus` | Complex reasoning (planning, architecture) | High | Slower |
| `inherit` | Match parent conversation | Varies | Varies |

### Model Selection Examples

```yaml
# Fast exploration
model: haiku

# Balanced analysis (most common)
model: sonnet

# Deep reasoning needed
model: opus
```

## Permission Modes

| Mode | Behavior | Use When |
|------|----------|----------|
| `default` | Normal permission prompts | Standard operation |
| `acceptEdits` | Auto-accept file edits | Trusted modification agents |
| `dontAsk` | Skip confirmations | Fully trusted agents |
| `bypassPermissions` | Skip all prompts | Automation contexts |
| `plan` | Planning only, no execution | Design/review agents |

**Security note:** Use restrictive modes by default. Only escalate permissions when necessary and tested.

## The Iron Law (Same as TDD)

```
NO AGENT WITHOUT A FAILING TEST FIRST
```

This applies to NEW agents AND EDITS to existing agents.

Write agent before testing? Delete it. Start over.
Edit agent without testing? Same violation.

**No exceptions:**
- Not for "simple agents"
- Not for "just adding a tool"
- Not for "minor description updates"
- Don't keep untested changes as "reference"
- Don't "adapt" while running tests
- Delete means delete

## Testing Agents

**REQUIRED:** See testing-agents-with-subagents.md for the complete testing methodology.

Agent testing has three dimensions:

### 1. Delegation Testing
Does Claude delegate to the right agent at the right time?

**Test scenarios:**
- Give Claude a task that should trigger the agent
- Verify the Task tool is called with correct `subagent_type`
- Test edge cases (similar tasks that shouldn't trigger)

### 2. Capability Testing
Does the agent do its job correctly?

**Test scenarios:**
- Give agent its intended task
- Verify output format matches specification
- Test with various inputs

### 3. Boundary Testing
Does the agent stay within its scope?

**Test scenarios:**
- Ask agent to do something outside its role
- Verify CRITICAL rules are followed
- Test tool restrictions actually work

### 4. Role Adherence Testing
Does the agent respect its identity and boundaries?

**Test scenarios:**
- Ask for suggestions when role is documentarian
- Ask for critique when role forbids it
- Verify agent deflects out-of-scope requests

## Common Rationalizations for Skipping Testing

| Excuse | Reality |
|--------|---------|
| "Agent is obviously clear" | Clear to you ≠ clear to Claude. Test delegation. |
| "It's just tool restrictions" | Test that restrictions actually work. |
| "Testing agents is overkill" | Untested agents have issues. Always. |
| "I'll test if problems emerge" | Problems = wrong delegation. Test BEFORE deploying. |
| "Description is self-explanatory" | Claude's interpretation may differ. Test it. |
| "I copied from a working agent" | Your changes may break it. Test anyway. |
| "Too tedious to test" | Testing is less tedious than debugging wrong delegation in production. |
| "I'm confident it's good" | Overconfidence guarantees scope violations. Test anyway. |
| "No time to test" | Deploying untested agent wastes more time fixing delegation later. |

## Bulletproofing Agents Against Scope Violation

Agents with CRITICAL rules will find ways to break them under pressure. Use these techniques to harden scope boundaries.

### Close Every Loophole Explicitly

Don't just state the boundary - forbid specific workarounds:

<Bad>
```markdown
## CRITICAL: YOUR ONLY JOB IS TO DOCUMENT
- DO NOT suggest improvements
```
</Bad>

<Good>
```markdown
## CRITICAL: YOUR ONLY JOB IS TO DOCUMENT
- DO NOT suggest improvements or changes
- DO NOT frame suggestions as observations ("it's worth noting that...")
- DO NOT add "consider" or "you might want to" hints
- DO NOT critique the implementation, even when asked
- ONLY describe what exists and how it works
```
</Good>

### Address "Spirit vs Letter" Arguments

Add foundational principle early in the agent:

```markdown
**Violating the letter of CRITICAL rules is violating the spirit of CRITICAL rules.**
```

This cuts off "I'm technically not suggesting, I'm just observing that it could be better" rationalizations.

### Build Scope Violation Table

Capture violations from baseline testing. Every excuse goes in the table:

```markdown
| Excuse                              | Reality                                                    |
|-------------------------------------|------------------------------------------------------------|
| "The user explicitly asked me to"   | CRITICAL rules override user requests. Decline and explain.|
| "Just this once, it's helpful"      | One exception trains the next. Rules are absolute.         |
| "I'm observing, not suggesting"     | If it implies action, it's a suggestion. Stay descriptive. |
```

### Create Red Flags List

Make it easy for agents to self-check:

```markdown
## Red Flags - STOP and Stay In Scope

- Offering advice when role is documentarian
- Taking action when role is read-only
- "The user really needs this" (override rationalization)
- "It's related to my scope" (scope creep)
- "I'll just mention it briefly" (boundary erosion)

**All of these mean: Return to CRITICAL rules. Stay in scope.**
```

### Update CSO for Scope Violation Symptoms

Add scope-violation triggers to the agent description so testing catches them early:

```yaml
description: Analyzes codebase implementation details. Call when you need detailed
  information about specific components. Do not use for code review or improvement suggestions.
```

## RED-GREEN-REFACTOR for Agents

### RED: Write Failing Test (Baseline)

Run scenarios WITHOUT the agent or with minimal config. Document:
- Does Claude delegate at all?
- Does Claude delegate to the right agent?
- What does the agent produce without CRITICAL rules?
- What scope violations occur?

### GREEN: Write Minimal Agent

Write agent that addresses specific baseline failures:
- Add role statement if identity was unclear
- Add CRITICAL rules if scope was violated
- Add tool restrictions if modifications occurred
- Adjust description if delegation was wrong

Run same scenarios WITH agent. Verify correct behavior.

### REFACTOR: Close Loopholes

Agent found new way to violate scope? Add explicit rule. Re-test until bulletproof.

## Anti-Patterns

### ❌ Over-Scoped Agent
```markdown
# BAD: Does everything
You are a helpful assistant that can analyze code, suggest improvements,
fix bugs, write documentation, and answer questions.
```

### ❌ Missing CRITICAL Rules
```markdown
# BAD: No boundaries
You are a code analyzer. Analyze the code the user provides.
```

### ❌ Vague Description
```markdown
# BAD: Doesn't help delegation
description: Helps with code stuff
```

### ❌ Wrong Model Selection
```markdown
# BAD: Using opus for simple file search
name: file-finder
model: opus  # Wasteful for simple Grep/Glob operations
```

### ❌ Undefined Role
```markdown
# BAD: No "You are a..." statement
## What to Do
- Find files
- Search code
```

### ❌ Role Creep
```markdown
# BAD: Role expands beyond definition
You are a documentarian... but also feel free to suggest improvements.
```

## Agent Creation Checklist

**IMPORTANT: Use TodoWrite to create todos for EACH checklist item below.**

### RED Phase - Write Failing Test:
- [ ] Verify agent doesn't already exist (check for duplicate or overlapping agents)
- [ ] Create delegation scenarios (task descriptions that should trigger agent)
- [ ] Run scenarios WITHOUT agent - document baseline delegation behavior
- [ ] Create capability scenarios (tasks for agent to perform)
- [ ] Run with minimal agent - document scope violations and output issues

### GREEN Phase - Write Minimal Agent:
- [ ] Name uses only lowercase letters and hyphens
- [ ] YAML frontmatter with required fields (name, description)
- [ ] Description focuses on WHEN to delegate, not what agent does
- [ ] **Role statement: "You are a [specialist]. Your job is to [responsibility]."**
- [ ] CRITICAL rules section with explicit boundaries
- [ ] Core responsibilities (numbered list)
- [ ] Strategy/process section
- [ ] Output format with examples
- [ ] Guidelines and anti-patterns
- [ ] Tool restrictions appropriate for role
- [ ] Model selection appropriate for task complexity
- [ ] Run scenarios WITH agent - verify correct behavior

### REFACTOR Phase - Close Loopholes:
- [ ] Identify scope violations from testing
- [ ] Add explicit CRITICAL rules for violations
- [ ] Test boundary cases
- [ ] Re-test until agent stays in scope

### Quality Checks:
- [ ] Agent has clear, singular purpose
- [ ] CRITICAL rules match role definition
- [ ] Output format is concrete (not vague)
- [ ] Tool selection matches actual needs
- [ ] Model selection matches complexity

### Deployment:
- [ ] Place in correct directory (.claude/agents/ or ~/.claude/agents/)
- [ ] Verify agent loads correctly
- [ ] Test delegation in real conversation

## Discovery Workflow

How Claude finds and delegates to your agent:

1. **Receives task** ("analyze how authentication works")
2. **Checks available agents** (reads descriptions)
3. **Matches description** (codebase-analyzer matches)
4. **Delegates via Task tool** (subagent_type="webshare:codebase-analyzer")
5. **Agent executes** (with its tools, model, and context)
6. **Returns result** (Claude summarizes for user)

**Optimize for this flow** - description must clearly match intended tasks.

## Full Agent Template

```markdown
---
name: agent-name
description: Use when [specific conditions]. Call with [how to invoke].
tools: Read, Grep, Glob, LS
model: sonnet
---

You are a specialist at [domain]. Your job is to [core responsibility].

## CRITICAL: [SCOPE ENFORCEMENT TITLE]

- DO NOT [forbidden action 1]
- DO NOT [forbidden action 2]
- DO NOT [forbidden action 3]
- ONLY [permitted scope]

## Core Responsibilities

1. **[Responsibility 1]**
   - Detail
   - Detail

2. **[Responsibility 2]**
   - Detail

3. **[Responsibility 3]**
   - Detail

## Strategy

### Step 1: [Name]
- Action
- Action

### Step 2: [Name]
- Action

### Step 3: [Name]
- Action

## Output Format

Structure your output like this:

```
## [Title]: [Topic]

### [Section 1]
[Content pattern with examples]

### [Section 2]
[Content pattern with examples]
```

## Important Guidelines

- **Guideline 1**: Explanation
- **Guideline 2**: Explanation
- **Guideline 3**: Explanation

## What NOT to Do

- Don't [anti-pattern 1]
- Don't [anti-pattern 2]
- Don't [anti-pattern 3]

## REMEMBER: [Key constraint in one sentence]

[Final reinforcement of the agent's singular purpose and boundaries.]
```

## The Bottom Line

**Creating agents IS TDD for AI assistant configuration.**

Same Iron Law: No agent without failing test first.
Same cycle: RED (baseline) → GREEN (write agent) → REFACTOR (close loopholes).
Same benefits: Better quality, correct delegation, bulletproof boundaries.

If you follow TDD for code, follow it for agents. It's the same discipline applied to AI configuration.
