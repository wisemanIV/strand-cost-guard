# Getting Started

This guide walks you through installing and configuring Strand Cost Guard for your first project.

## Installation

```bash
pip install strands-costguard
```

### Dependencies

Cost Guard requires:
- Python 3.10+
- `opentelemetry-api` and `opentelemetry-sdk`
- `pyyaml` for configuration loading
- `strands` framework (for telemetry integration)

## Quick Start

### 1. Set Up Telemetry

Cost Guard uses the global MeterProvider from StrandsTelemetry. Configure it first:

```python
from strands.telemetry.config import StrandsTelemetry

# Configure telemetry at application startup
telemetry = StrandsTelemetry()
telemetry.setup_otlp_exporter(endpoint="http://localhost:4317")
telemetry.setup_meter(enable_otlp_exporter=True)
```

For local development without an OTEL collector:

```python
telemetry = StrandsTelemetry()
telemetry.setup_console_exporter()
telemetry.setup_meter(enable_console_exporter=True)
```

### 2. Create Policy Files

Create a `policies/` directory with your configuration files:

**policies/budgets.yaml**
```yaml
budgets:
  - id: "default-budget"
    scope: "tenant"
    match:
      tenant_id: "*"
    period: "monthly"
    max_cost: 1000.0
    soft_thresholds: [0.7, 0.9]
    hard_limit: true
    on_soft_threshold_exceeded: "LOG_ONLY"
    on_hard_limit_exceeded: "REJECT_NEW_RUNS"
```

**policies/routing.yaml**
```yaml
routing_policies:
  - id: "default-routing"
    match:
      strand_id: "*"
    default_model: "gpt-4o-mini"
    stages:
      - stage: "synthesis"
        default_model: "gpt-4o"
        fallback_model: "gpt-4o-mini"
        trigger_downgrade_on:
          soft_threshold_exceeded: true
```

**policies/pricing.yaml**
```yaml
pricing:
  currency: "USD"
  models:
    "gpt-4o":
      input_per_1k: 2.50
      output_per_1k: 10.00
    "gpt-4o-mini":
      input_per_1k: 0.15
      output_per_1k: 0.60
```

### 3. Initialize Cost Guard

```python
from strands_costguard import CostGuard, CostGuardConfig, FilePolicySource

config = CostGuardConfig(
    policy_source=FilePolicySource(path="./policies"),
    enable_budget_enforcement=True,
    enable_routing=True,
    enable_metrics=True,
)

guard = CostGuard(config=config)
```

### 4. Integrate with Your Agent

```python
from strands_costguard import ModelUsage, ToolUsage, IterationUsage

# Start a run
admission = guard.on_run_start(
    tenant_id="my-tenant",
    strand_id="my-agent",
    workflow_id="my-workflow",
    run_id="run-123",
)

if not admission.allowed:
    print(f"Run rejected: {admission.reason}")
    exit(1)

# Agent loop
for iteration in range(10):
    # Check iteration
    iter_decision = guard.before_iteration(run_id="run-123", iteration_idx=iteration)
    if not iter_decision.allowed:
        break

    # Before model call
    model_decision = guard.before_model_call(
        run_id="run-123",
        model_name="gpt-4o",
        stage="synthesis",
    )

    # Use effective_model (may be downgraded)
    model = model_decision.effective_model

    # ... make actual model call ...

    # After model call
    guard.after_model_call(
        run_id="run-123",
        usage=ModelUsage.from_response(
            model_name=model,
            prompt_tokens=500,
            completion_tokens=200,
        ),
    )

    # After iteration
    guard.after_iteration(
        run_id="run-123",
        iteration_idx=iteration,
        usage=IterationUsage(iteration_idx=iteration),
    )

# End run
guard.on_run_end(run_id="run-123", status="completed")

# Cleanup
guard.shutdown()
```

## Using the ModelRouter Helper

For simpler integration, use the `ModelRouter` class:

```python
from strands_costguard import CostGuard, ModelRouter

guard = CostGuard(config=config)
router = ModelRouter(cost_guard=guard)

# Before making a model call
context = router.before_call(
    run_id="run-123",
    stage="planning",
    messages=[{"role": "user", "content": "Hello"}],
)

if context.allowed:
    # Make the call with the effective model
    response = your_model_client.call(
        model=context.effective_model,
        messages=messages,
        max_tokens=context.max_tokens,
    )

    # Record the usage
    router.after_call(run_id="run-123", response=response)
```

## Environment Variable Configuration

For simple single-tenant setups, configure via environment variables:

```bash
export COST_GUARD_MAX_COST=1000.0
export COST_GUARD_PERIOD=monthly
export COST_GUARD_DEFAULT_MODEL=gpt-4o-mini
export COST_GUARD_FALLBACK_MODEL=gpt-3.5-turbo
```

```python
from strands_costguard import CostGuard, CostGuardConfig
from strands_costguard.policies.store import EnvPolicySource

config = CostGuardConfig(
    policy_source=EnvPolicySource(),
)

guard = CostGuard(config=config)
```

## Next Steps

- [Configuration Reference](./configuration.md) - All configuration options
- [Budget Policies](./budget-policies.md) - Define spending limits
- [Routing Policies](./routing-policies.md) - Configure model selection
- [Lifecycle Hooks](./lifecycle-hooks.md) - Full integration guide
