---
name: using-prompt-lab
description: Use when creating, running, comparing, or analyzing prompt-lab experiments, when testing prompt variants across LLM providers, when setting up LLM-as-judge evaluation, or when viewing experiment results
---

# Using Prompt-Lab

## Overview

Prompt-lab is a CLI tool for testing prompt variants across LLM providers using LLM-as-judge evaluation.

```
system.md (optional) + prompt.md + inputs.yaml → LLM → response → judge.md → score
```

Create multiple variants (v1, v2, ...) to A/B test different prompt approaches against the same inputs and judge, then compare with statistical significance.

## Experiment Structure

```
experiments/
  my-experiment/
    experiment.md       # Config: name, models, runs (required)
    judge.md            # Scoring rubric (required)
    inputs.yaml         # Shared test cases (optional)
    v1/                 # Variant (at least one required)
      prompt.md         # User message (required)
      system.md         # System message (optional)
      tools.yaml        # Tool definitions (optional)
    v2/                 # Another variant to compare
      prompt.md
```

**Fallback resolution**: `judge.md` and `inputs.yaml` are checked in the variant directory first, then the experiment directory. This enables shared test cases while allowing per-variant overrides.

## File Quick Reference

| File            | Required          | Purpose                                                 |
|-----------------|-------------------|---------------------------------------------------------|
| `experiment.md` | Yes               | YAML frontmatter: name, models, runs, hypothesis        |
| `prompt.md`     | Yes (per variant) | User message with `{{ vars }}` from inputs              |
| `system.md`     | No                | System message (persona, tool instructions)             |
| `judge.md`      | Yes               | Scoring rubric with `{{ prompt }}` and `{{ response }}` |
| `inputs.yaml`   | No                | Test cases providing template variables                 |
| `tools.yaml`    | No                | Function calling definitions                            |

For detailed file formats and judge rubric design, see [experiment-reference.md](experiment-reference.md).

## Creating Experiments

### From config file (recommended)

```bash
prompt-lab new --config spec.yaml
```

Spec format:

```yaml
name: my-experiment
description: What this tests
hypothesis: Expected outcome
models:
  - openai:gpt-4o-mini
  - anthropic:claude-sonnet-4-20250514
runs: 5
path: experiments
key_refs:
  openai: MY_CUSTOM_OPENAI_KEY

inputs:
  - id: case-1
    field_name: value1
  - id: case-2
    field_name: value2

judge:
  model: openai:gpt-4o
  score_range: [0, 5]
  temperature: 0
  chain_of_thought: true
  rubric: |
    Your rubric with {{ prompt }} and {{ response }}.

variants:
  v1:
    prompt: |
      Prompt template with {{ field_name }}.
    system: |
      Optional system prompt.
  v2:
    prompt: |
      Alternative approach with {{ field_name }}.
```

### Interactive wizard

```bash
prompt-lab new
```

### Manual creation

Create the directory structure and files by hand. Best for complex experiments.

## Running Experiments

```bash
# Run all variants in an experiment
prompt-lab run experiments/my-experiment

# Run a single variant
prompt-lab run experiments/my-experiment/v1

# Run specific model only
prompt-lab run experiments/my-experiment/v1 --model openai:gpt-4o-mini

# Skip cache (fresh API calls)
prompt-lab run experiments/my-experiment --no-cache

# Hide progress bar
prompt-lab run experiments/my-experiment -q

# Custom API key env var (format: provider:ENV_VAR)
prompt-lab run experiments/my-experiment -k openai:MY_OPENAI_KEY
```

### What happens during a run

For each `(input, run_number, model)` combination, concurrently:
1. Provider renders `prompt.md` + `system.md` with input variables via Jinja2
2. LLM generates a response (with optional tool calls)
3. Judge LLM scores the response using `judge.md` rubric
4. Result saved as JSON under `variant/results/{timestamp}/responses/`

Cache is **automatically disabled** when `runs > 1` to ensure independent responses.

## Viewing Results

### Results table

```bash
prompt-lab results experiments/my-experiment/v1
```

Shows per-input scores with mean, 95% confidence interval, and score range.

```bash
# View a specific historical run
prompt-lab results experiments/my-experiment/v1 --run 2026-01-25T19-30-00
```

### Detailed responses with judge reasoning

```bash
# All responses
prompt-lab show experiments/my-experiment/v1

# Filter by input
prompt-lab show experiments/my-experiment/v1 --input alice

# Filter by model
prompt-lab show experiments/my-experiment/v1 --model openai:gpt-4o-mini

# Combine filters
prompt-lab show experiments/my-experiment/v1 --input alice --model openai:gpt-4o-mini

# Specific historical run
prompt-lab show experiments/my-experiment/v1 --run 2026-01-25T19-30-00
```

### Results storage

```
variant/results/{timestamp}/
  run.yaml                              # Run metadata (duration, models, counts)
  stats.yaml                            # Per-input stats (mean, CI, stddev, scores)
  responses/
    {input_id}_run{N}_{provider}-{model}.json   # Individual result
```

Each response JSON contains: `input_id`, `model`, `run_number`, `cached`, `latency_ms`, `input_tokens`, `output_tokens`, `response` (content + tool_calls), `judge` (score + reasoning).

## Comparing Variants

```bash
prompt-lab compare experiments/my-experiment
```

Shows comparison table across all variants:
- Mean score per variant with 95% confidence intervals
- Average latency
- Total runs
- **Statistical significance** via Welch's t-test (p-value)
- Experiment hypothesis

Tells you whether v1 is actually better than v2, or if the difference is just noise.

## Cleaning Up

```bash
# Clean single variant results
prompt-lab clean experiments/my-experiment/v1

# Clean all variants in an experiment
prompt-lab clean experiments/my-experiment

# Skip confirmation
prompt-lab clean experiments/my-experiment --yes
```

## Cache Management

```bash
prompt-lab cache clear
```

Cache stores LLM responses to avoid redundant API calls during development. Automatically disabled when `runs > 1`.

## Experiment Design Tips

- **Variant strategy**: Each variant tests a different prompt approach (zero-shot vs few-shot, formal vs casual, structured vs freeform, with/without CoT)
- **Statistical reliability**: Use `runs: 5+` for meaningful confidence intervals
- **Judge selection**: Use a different model family as judge to reduce self-enhancement bias, or multi-judge with `models:` (plural) in judge.md
- **Model format**: Always `provider:model` (e.g., `openai:gpt-4o-mini`, `anthropic:claude-sonnet-4-20250514`)
- **Supported providers**: `openai:*`, `anthropic:*`

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Missing `{{ prompt }}` / `{{ response }}` in judge | Required for judge to see what it's evaluating |
| Template variable not in inputs.yaml | All `{{ var }}` in prompts must have matching input fields |
| Model without provider prefix | Use `openai:gpt-4o-mini`, not `gpt-4o-mini` |
| `runs: 1` for statistical comparison | Use `runs: 5+` for confidence intervals |
| Vague rubric ("rate 0-5") | Use concrete criteria with point values. See [experiment-reference.md](experiment-reference.md) |
| Same model as judge and subject | Use multi-judge or different model to reduce bias |
| No `judge.md` anywhere | Must exist in variant or experiment directory |
| No `prompt.md` in variant dir | Every variant needs a `prompt.md` |
