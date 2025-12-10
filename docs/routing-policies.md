# Routing Policies

Routing policies define how models are selected based on budget state, semantic stage, and other conditions.

## Routing Policy Specification

Routing policies are defined in `routing.yaml`:

```yaml
routing_policies:
  - id: "default-routing"
    match:
      strand_id: "*"
    default_model: "gpt-4o-mini"
    default_fallback_model: "gpt-3.5-turbo"
    stages:
      - stage: "planning"
        default_model: "gpt-4o-mini"
        max_tokens: 2000
        trigger_downgrade_on:
          soft_threshold_exceeded: true
          remaining_budget_below: 10.0

      - stage: "synthesis"
        default_model: "gpt-4o"
        fallback_model: "gpt-4o-mini"
        max_tokens: 4000
        trigger_downgrade_on:
          soft_threshold_exceeded: true
          remaining_budget_below: 20.0
          iteration_count_above: 5
```

## Fields Reference

### Policy Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `id` | `string` | Yes | Unique identifier for this policy |
| `match` | `object` | No | Criteria for matching contexts |
| `default_model` | `string` | No | Default model when no stage matches |
| `default_fallback_model` | `string` | No | Global fallback model |
| `stages` | `list` | No | Stage-specific configurations |
| `enabled` | `boolean` | No | Whether this policy is active (default: `true`) |

### Match Criteria

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `tenant_id` | `string` | `"*"` | Tenant ID to match |
| `strand_id` | `string` | `"*"` | Strand ID to match |
| `workflow_id` | `string` | `"*"` | Workflow ID to match |

### Stage Configuration

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `stage` | `string` | Yes | Stage name to match |
| `default_model` | `string` | Yes | Model to use for this stage |
| `fallback_model` | `string` | No | Model to use when downgrading |
| `max_tokens` | `int` | No | Maximum tokens for this stage |
| `temperature` | `float` | No | Temperature setting for this stage |
| `trigger_downgrade_on` | `object` | No | Conditions that trigger downgrade |

### Downgrade Triggers

| Field | Type | Description |
|-------|------|-------------|
| `soft_threshold_exceeded` | `boolean` | Downgrade when budget soft threshold is crossed |
| `remaining_budget_below` | `float` | Downgrade when remaining budget falls below this amount |
| `iteration_count_above` | `int` | Downgrade when iteration count exceeds this number |
| `latency_above_ms` | `float` | Downgrade when average latency exceeds this (ms) |

## Model Stages

Cost Guard recognizes semantic stages within an agent loop:

| Stage | Description | Typical Use |
|-------|-------------|-------------|
| `planning` | Initial task analysis and planning | Decomposing tasks, strategy selection |
| `tool_selection` | Deciding which tools to use | Selecting APIs, functions |
| `synthesis` | Generating final responses | Summarizing, formatting output |
| `other` | Any stage not matching above | Custom stages |

### Using Stages in Your Agent

```python
# Planning phase
model_decision = guard.before_model_call(
    run_id=run_id,
    model_name="gpt-4o",
    stage="planning",  # Matches "planning" stage config
)

# Tool selection phase
model_decision = guard.before_model_call(
    run_id=run_id,
    model_name="gpt-4o",
    stage="tool_selection",  # Matches "tool_selection" stage config
)

# Final response generation
model_decision = guard.before_model_call(
    run_id=run_id,
    model_name="gpt-4o",
    stage="synthesis",  # Matches "synthesis" stage config
)
```

## Downgrade Logic

When a downgrade trigger condition is met:

1. Cost Guard checks if a `fallback_model` is configured for the stage
2. If available, the fallback model is returned instead of the default
3. The `ModelDecision` indicates `was_downgraded=True` with the reason
4. A downgrade metric event is emitted

### Trigger Evaluation

Triggers are evaluated in order. The first matching condition triggers a downgrade:

```yaml
trigger_downgrade_on:
  soft_threshold_exceeded: true      # Check 1: Budget threshold
  remaining_budget_below: 20.0       # Check 2: Absolute remaining
  iteration_count_above: 5           # Check 3: Iteration count
  latency_above_ms: 5000             # Check 4: Latency
```

## Policy Priority

When multiple routing policies match, the most specific one is selected based on match criteria:

| Criteria | Priority Score |
|----------|---------------|
| Specific `tenant_id` | +1 |
| Specific `strand_id` | +2 |
| Specific `workflow_id` | +4 |

Higher scores take precedence. Only one routing policy applies to each context.

## Examples

### Cost-Optimized Routing

Use cheaper models by default, upgrade only for synthesis:

```yaml
routing_policies:
  - id: "cost-optimized"
    match:
      strand_id: "*"
    default_model: "gpt-4o-mini"
    stages:
      - stage: "planning"
        default_model: "gpt-4o-mini"
        max_tokens: 1500

      - stage: "tool_selection"
        default_model: "gpt-4o-mini"
        max_tokens: 800

      - stage: "synthesis"
        default_model: "gpt-4o"
        fallback_model: "gpt-4o-mini"
        max_tokens: 3000
        trigger_downgrade_on:
          soft_threshold_exceeded: true
```

### Quality-First Routing

Use the best models, with graceful degradation:

```yaml
routing_policies:
  - id: "quality-first"
    match:
      strand_id: "code_generator"
    default_model: "gpt-4o"
    stages:
      - stage: "planning"
        default_model: "gpt-4o"
        fallback_model: "gpt-4o-mini"
        max_tokens: 4000
        trigger_downgrade_on:
          soft_threshold_exceeded: true
          remaining_budget_below: 50.0

      - stage: "tool_selection"
        default_model: "gpt-4o-mini"
        max_tokens: 1500

      - stage: "synthesis"
        default_model: "gpt-4o"
        fallback_model: "gpt-4o-mini"
        max_tokens: 8000
        trigger_downgrade_on:
          soft_threshold_exceeded: true
          remaining_budget_below: 30.0
```

### Iteration-Based Downgrade

Downgrade after multiple iterations to control runaway costs:

```yaml
routing_policies:
  - id: "iteration-aware"
    match:
      strand_id: "*"
    default_model: "gpt-4o-mini"
    stages:
      - stage: "synthesis"
        default_model: "gpt-4o"
        fallback_model: "gpt-4o-mini"
        max_tokens: 4000
        trigger_downgrade_on:
          iteration_count_above: 3  # Downgrade after 3 iterations
```

### Multi-Provider Routing

Use different providers for different tenants:

```yaml
routing_policies:
  # OpenAI for most tenants
  - id: "openai-default"
    match:
      strand_id: "*"
    default_model: "gpt-4o-mini"
    stages:
      - stage: "synthesis"
        default_model: "gpt-4o"
        fallback_model: "gpt-4o-mini"

  # Claude for specific tenant
  - id: "claude-tenant"
    match:
      tenant_id: "anthropic-customer"
    default_model: "claude-3.5-haiku"
    stages:
      - stage: "synthesis"
        default_model: "claude-3.5-sonnet"
        fallback_model: "claude-3.5-haiku"
        trigger_downgrade_on:
          soft_threshold_exceeded: true
```

## Integration with Budget Policies

Routing and budget policies work together:

1. **Budget policy** defines `on_soft_threshold_exceeded: "DOWNGRADE_MODEL"`
2. **Routing policy** defines which models to switch between
3. When the budget threshold is crossed, Cost Guard:
   - Sees the budget action is `DOWNGRADE_MODEL`
   - Passes `soft_threshold_exceeded=True` to routing
   - Routing returns the fallback model

Example coordination:

```yaml
# budgets.yaml
budgets:
  - id: "tenant-budget"
    scope: "tenant"
    match:
      tenant_id: "*"
    max_cost: 1000.0
    soft_thresholds: [0.7]
    on_soft_threshold_exceeded: "DOWNGRADE_MODEL"  # Triggers routing

# routing.yaml
routing_policies:
  - id: "default-routing"
    match:
      strand_id: "*"
    stages:
      - stage: "synthesis"
        default_model: "gpt-4o"
        fallback_model: "gpt-4o-mini"  # Used when budget triggers downgrade
        trigger_downgrade_on:
          soft_threshold_exceeded: true  # Responds to budget signal
```

## Handling Downgrade Decisions

```python
decision = guard.before_model_call(
    run_id=run_id,
    model_name="gpt-4o",
    stage="synthesis",
)

if decision.allowed:
    print(f"Using model: {decision.effective_model}")

    if decision.was_downgraded:
        print(f"Downgraded from gpt-4o: {decision.reason}")
        # decision.warnings contains details

    # Use decision.max_tokens if set
    response = model_client.call(
        model=decision.effective_model,
        max_tokens=decision.max_tokens,
        messages=messages,
    )
else:
    print(f"Model call rejected: {decision.reason}")
```

## Best Practices

1. **Define default policies** - Always have a catch-all routing policy
2. **Match routing to budget thresholds** - Coordinate downgrade triggers
3. **Use appropriate fallbacks** - Ensure fallback models are suitable for the task
4. **Set reasonable max_tokens** - Prevent excessive token usage per call
5. **Test downgrade paths** - Verify your agents work with fallback models
6. **Consider latency** - Cheaper models are often faster too
