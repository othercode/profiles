# Profiles

Engineering profiles for Claude Code — stack-specific conventions, workflows, and automation by [otherCode](https://github.com/othercode).

## What this is

A Claude Code plugin marketplace containing opinionated engineering profiles. Each profile packages the conventions, patterns, and automation for a specific technology stack.

Profiles don't teach Claude what a framework is — it already knows. Instead, they encode **your decisions**: how you structure code, what patterns to follow, what to avoid, and how to automate enforcement.

## Install

Add the marketplace:

```sh
/plugin marketplace add othercode/profiles
```

Install a profile:

```sh
/plugin install <profile-name>@othercode
```

## Update

Profiles are distributed as a mono-repo marketplace. To pull the latest versions of all profiles:

```sh
/plugin marketplace update othercode
```

> **Note:** Do not use `/plugin update <profile-name>` for individually installed profiles — that only works for plugins with external sources (separate GitHub repos). Mono-repo plugins update through the marketplace.

## Uninstall

```sh
/plugin uninstall <profile-name>@othercode
```

## Available profiles

| Profile | Stack | Install |
|---------|-------|---------|
| `laravel-engineering` | PHP, Laravel, Vue.js | `/plugin install laravel-engineering@othercode` |
| `django-engineering` | Python, Django, DRF | `/plugin install django-engineering@othercode` |
| `prompt-engineering` | Claude Code skills, agents, commands | `/plugin install prompt-engineering@othercode` |

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

```text
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

After creating the plugin structure, add an entry to `.claude-plugin/marketplace.json` and validate:

```sh
/plugin validate .
```
