# Experiment File Format Reference

Detailed formats for all experiment files. See [SKILL.md](SKILL.md) for the overview and CLI commands.

## experiment.md

YAML frontmatter with markdown body. The frontmatter configures the experiment; the body is optional documentation.

```yaml
---
name: my-experiment
description: Brief description of what this experiment tests
hypothesis: Concise prompts will score higher than verbose ones
models:
  - openai:gpt-4o-mini
  - anthropic:claude-sonnet-4-20250514
runs: 5
key_refs:
  openai: MY_CUSTOM_OPENAI_KEY
---

Optional markdown content describing the experiment in detail.
```

| Field | Default | Description |
|-------|---------|-------------|
| `name` | folder name | Experiment identifier |
| `description` | `""` | Brief description |
| `hypothesis` | `""` | What you're testing (displayed in results) |
| `models` | **required** | List of `provider:model` strings |
| `runs` | `5` | Runs per input per model (for statistical analysis) |
| `key_refs` | `{}` | Map of `provider: ENV_VAR` for custom API key env vars |

## prompt.md

The user message sent to the LLM. Uses Jinja2 `{{ variable }}` syntax. Each variant has its own `prompt.md` — this is what you A/B test.

**With variables:**
```markdown
Generate 5 creative product names.

Product description: {{ description }}
Seed words: {{ seeds }}

Product names:
```

**Few-shot variant** (same experiment, different approach):
```markdown
Generate 5 creative product names.

Product description: A water bottle that keeps drinks cold for 48 hours
Seed words: arctic, chill, endure
Product names: ArcticSip, ChillVault, FrostLock, EnduraCool, PolarPour

Product description: {{ description }}
Seed words: {{ seeds }}

Product names:
```

**Static prompt** (no variables needed, no inputs.yaml required):
```markdown
Tell me a joke about programming.
```

Literal braces in prompts don't need escaping: `{"result": "value"}` works fine.

Can include optional YAML frontmatter with `models:` to override experiment-level models for this variant only.

## system.md (optional)

System message setting persona and behavior. Same `{{ variable }}` support from inputs.yaml.

```markdown
You are a helpful assistant. You can check the weather using the
get_weather tool when users ask about weather conditions.

Only use the weather tool when the user is actually asking about weather.
For other questions, just respond normally without using any tools.
```

**When to use system.md:**
- Setting a persona or role for the LLM
- Providing tool usage instructions
- Separating "what the model is" from "what the user asks"

When absent, the prompt is sent as user message with no system message.

## inputs.yaml

YAML list of test cases. Each item provides variables for `prompt.md`, `system.md`, and `judge.md` templates.

```yaml
# Weather experiment inputs - mix of should/shouldn't call tools
- id: weather-paris
  message: "What's the weather like in Paris?"
  should_call_tool: true
  expected_location: "Paris"

- id: greeting
  message: "Hello, how are you?"
  should_call_tool: false
  expected_location: ""

# Product naming inputs
- id: shoes
  description: "A pair of shoes that can fit any foot size"
  seeds: "adaptable, fit, omni-fit"
  runs: 10  # Override: more runs for this important case
```

| Field      | Default           | Description                                |
|------------|-------------------|--------------------------------------------|
| `id`       | `input-N`         | Unique identifier for results              |
| `runs`     | experiment `runs` | Override runs for this specific input      |
| All others | -                 | Available as `{{ variable }}` in templates |

**Location**: Experiment level (shared across variants) or variant level (variant-specific). Variant-level takes precedence.

**If omitted**: Runs once with empty data (useful for static prompts without variables).

## judge.md

YAML frontmatter + markdown rubric. The judge is another LLM that evaluates each response.

### Single judge

```yaml
---
model: openai:gpt-4o
score_range: [0, 5]
temperature: 0
---

You are evaluating product names for brandability and consistency.

## Scoring (start at 3, add/subtract)

### Format Consistency (-1 to +1)
- **+1**: ALL names follow identical pattern
- **0**: Minor inconsistency (4/5 match)
- **-1**: Mixed or chaotic formatting

### Brandability (-1 to +1)
- **+1**: All single CamelCase compounds (FlexiFit, OmniStep)
- **0**: Clean two-word phrases or mix of compound and multi-word
- **-1**: Names with 3+ words

## Soft Penalties
- Names including product category ("Shoes"): -0.5 per occurrence (max -1)

## Evaluation Context
**Prompt:** {{ prompt }}
**Generated Names:** {{ response }}
```

### Multi-judge (reduces self-enhancement bias)

```yaml
---
models:
  - openai:gpt-4o-mini
  - anthropic:claude-sonnet-4-20250514
aggregation: mean
score_range: [0, 5]
temperature: 0
---

Same rubric body...
```

Use `model:` (singular) for single judge, `models:` (plural) for multi-judge.

### Judge options

| Option             | Default         | Description                             |
|--------------------|-----------------|-----------------------------------------|
| `model`            | `openai:gpt-4o` | Single judge model                      |
| `models`           | -               | Multi-judge: list of models             |
| `aggregation`      | `mean`          | Multi-judge: `mean` or `median`         |
| `score_range`      | `[0, 5]`        | Min and max score                       |
| `temperature`      | `0`             | 0 = deterministic, higher = more varied |
| `chain_of_thought` | `true`          | Step-by-step reasoning before scoring   |

### Available template variables

| Variable              | Description                         |
|-----------------------|-------------------------------------|
| `{{ prompt }}`        | Rendered user prompt (**required**) |
| `{{ response }}`      | LLM's text response (**required**)  |
| `{{ system_prompt }}` | Rendered system prompt              |
| `{{ tool_calls }}`    | Tool calls made by the model        |
| Any input field       | Available from inputs.yaml          |

### Chain-of-Thought evaluation

Enabled by default. The system automatically prepends instructions for step-by-step analysis before scoring, improving alignment with human judgment. Disable with `chain_of_thought: false` for faster/cheaper evaluations.

### Judge output format

The system automatically appends a JSON output instruction. The judge must respond with:
```json
{"score": <integer in range>, "reasoning": "<explanation>"}
```

## tools.yaml (optional)

Define tools the LLM can call. For testing function-calling prompts.

```yaml
- name: get_weather
  description: Get current weather for a location
  parameters:
    type: object
    properties:
      location:
        type: string
        description: City name (e.g., "Paris", "Tokyo")
    required:
      - location

- name: search
  description: Search the web
  parameters:
    type: object
    properties:
      query:
        type: string
```

| Field         | Required | Description                |
|---------------|----------|----------------------------|
| `name`        | Yes      | Tool/function name         |
| `description` | No       | What the tool does         |
| `parameters`  | No       | JSON Schema for parameters |

Tool calls made by the model are captured in the response and available to the judge via `{{ tool_calls }}`.

## Writing Effective Rubrics

**1. Start at a baseline, add/subtract** — More predictable than absolute scoring:
```markdown
## Scoring (start at 3, add/subtract)
### Criterion A (-1 to +1)
```

**2. Use concrete criteria** — Not "is it good?" but "+2 if ALL names are CamelCase compounds":
```markdown
# Bad:  "Rate the quality"
# Good: "+2 if ALL names are single CamelCase compounds (FlexiFit, OmniStep)"
```

**3. Soft penalties for anti-patterns** — Capped to prevent runaway deductions:
```markdown
- Names including product category: -0.5 per occurrence (max -1)
```

**4. Include score guide** — Anchors the judge's expectations:
```markdown
## Score Guide
- **5**: Exceptional
- **4**: Good
- **3**: Average
- **2**: Below average
- **0-1**: Poor
```

**5. For tool-use experiments** — Include tool_calls and define expected behavior:
```markdown
**Tool calls made:** {{ tool_calls }}
- If weather question: Should call `get_weather` with correct location → 5
- If non-weather question: Should NOT call any tool → 5
```

**6. Multi-judge for fairness** — When testing GPT responses, add Claude as judge (and vice versa) to reduce self-enhancement bias.

## Config Spec Reference (for `prompt-lab new --config`)

Complete spec with all fields:

```yaml
# Required
name: experiment-name          # Will be slugified (spaces → hyphens)
models:                        # At least one required
  - openai:gpt-4o-mini
  - anthropic:claude-sonnet-4-20250514
variants:                      # At least one required
  v1:
    prompt: "Prompt text with {{ var }}"     # Required per variant
    system: "Optional system prompt"         # Optional
    description: "What this variant tests"   # Optional
    tools:                                   # Optional
      - name: tool_name
        description: What it does
        parameters: { type: object, properties: { ... } }

# Optional
description: What the experiment tests
hypothesis: Expected outcome
runs: 5                        # Default: 5
path: experiments              # Default: experiments
key_refs:                      # Custom API key env vars
  openai: MY_CUSTOM_OPENAI_KEY

inputs:                        # Default: [{ id: default }]
  - id: case-1
    field: value

judge:                         # All optional with defaults
  model: openai:gpt-4o        # Default judge model
  models: [...]                # Multi-judge (overrides model)
  aggregation: mean            # mean or median
  score_range: [0, 5]
  temperature: 0.0
  chain_of_thought: true
  rubric: |                    # Must include {{ prompt }} and {{ response }}
    Evaluation rubric text
```

### Validation rules

- `name` is slugified: spaces become hyphens
- Each model must use `provider:model` format
- Each input must have a string `id`
- Template variables `{{ var }}` in prompts must match fields from inputs
- Judge rubric (if provided) must contain `{{ prompt }}` and `{{ response }}`
- Target directory must not already exist
