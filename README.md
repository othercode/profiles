# profiles

Engineering profiles for Claude Code — stack-specific conventions, workflows, and automation by [otherCode](https://github.com/othercode).

## What this is

A Claude Code plugin marketplace containing opinionated engineering profiles. Each profile packages the conventions, patterns, and automation for a specific technology stack.

Profiles don't teach Claude what a framework is — it already knows. Instead, they encode **your decisions**: how you structure code, what patterns to follow, what to avoid, and how to automate enforcement.

## Install

Add the marketplace:

```
/plugin marketplace add othercode/profiles
```

Install a profile:

```
/plugin install <profile-name>@othercode
```

## Per-project setup

Add this to your project's `.claude/settings.json` to auto-suggest profiles for anyone who clones the repo:

```json
{
  "extraKnownMarketplaces": {
    "othercode": {
      "source": { "source": "github", "repo": "othercode/profiles" }
    }
  },
  "enabledPlugins": {
    "<profile-name>@othercode": true
  }
}
```

## Philosophy

Each profile is self-contained. No shared base, no cross-profile dependencies. The same concept (e.g. commenting style) may exist in multiple profiles with stack-specific rules and examples — that's intentional.

A profile provides three layers:

- **Persona** (hook) — sets the engineering mindset on startup
- **Skills** (auto-triggered) — specific conventions with code examples, activated by context
- **Commands** (user-invoked) — multi-step workflows triggered explicitly

## Creating a new profile

```
plugins/<name>/
├── .claude-plugin/
│   └── plugin.json
├── commands/
├── agents/
├── skills/
│   └── <skill-name>/
│       └── SKILL.md
└── hooks/
    └── hooks.json
```

Skills follow the naming convention `<verb>-<target>` (e.g. `writing-tests`, `commenting-code`, `implementing-domain-events`).

Skill descriptions must be trigger-only — describe **when** to activate, never summarize what the skill contains.