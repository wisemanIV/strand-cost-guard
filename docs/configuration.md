# Configuration Reference

This document details all configuration options for Strand Cost Guard.

## CostGuardConfig

The main configuration class for Cost Guard.

```python
from strands_costguard import CostGuardConfig, FilePolicySource

config = CostGuardConfig(
    policy_source=FilePolicySource(path="./policies"),
    failure_mode="fail_open",
    policy_refresh_interval_seconds=300,
    enable_budget_enforcement=True,
    enable_routing=True,
    enable_metrics=True,
    include_run_id_in_metrics=False,
    currency="USD",
    default_max_iterations_per_run=50,
    default_max_tool_calls_per_run=100,
    default_max_tokens_per_run=100000,
    log_level="INFO",
)
```

### Options

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `policy_source` | `PolicySource` | Required | Source for loading policies (file or env) |
| `failure_mode` | `str` | `"fail_open"` | How to handle policy/metrics failures |
| `policy_refresh_interval_seconds` | `int` | `300` | How often to reload policies (seconds) |
| `enable_budget_enforcement` | `bool` | `True` | Enable/disable budget limit enforcement |
| `enable_routing` | `bool` | `True` | Enable/disable adaptive model routing |
| `enable_metrics` | `bool` | `True` | Enable/disable OpenTelemetry metrics |
| `include_run_id_in_metrics` | `bool` | `False` | Include run_id in metric attributes (high cardinality warning) |
| `currency` | `str` | `"USD"` | Currency for cost calculations |
| `default_max_iterations_per_run` | `int` | `50` | Default iteration limit if not specified in budget |
| `default_max_tool_calls_per_run` | `int` | `100` | Default tool call limit if not specified in budget |
| `default_max_tokens_per_run` | `int` | `100000` | Default token limit if not specified in budget |
| `log_level` | `str` | `"INFO"` | Logging level for Cost Guard |

### Failure Modes

| Mode | Behavior |
|------|----------|
| `fail_open` | Allow operations to proceed when errors occur |
| `fail_closed` | Block operations when errors occur |

## Policy Sources

### FilePolicySource

Loads policies from YAML files in a directory.

```python
from strands_costguard.policies.store import FilePolicySource

source = FilePolicySource(
    path="./policies",
    budgets_file="budgets.yaml",
    routing_file="routing.yaml",
    pricing_file="pricing.yaml",
)
```

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `path` | `str` | Required | Directory containing policy files |
| `budgets_file` | `str` | `"budgets.yaml"` | Filename for budget policies |
| `routing_file` | `str` | `"routing.yaml"` | Filename for routing policies |
| `pricing_file` | `str` | `"pricing.yaml"` | Filename for pricing table |

### EnvPolicySource

Loads simple policies from environment variables.

```python
from strands_costguard.policies.store import EnvPolicySource

source = EnvPolicySource(prefix="COST_GUARD_")
```

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `prefix` | `str` | `"COST_GUARD_"` | Prefix for environment variable names |

#### Supported Environment Variables

| Variable | Description |
|----------|-------------|
| `{PREFIX}MAX_COST` | Maximum cost for the budget period |
| `{PREFIX}PERIOD` | Budget period (`hourly`, `daily`, `weekly`, `monthly`) |
| `{PREFIX}DEFAULT_MODEL` | Default model for routing |
| `{PREFIX}FALLBACK_MODEL` | Fallback model for routing |

## Directory Structure

Recommended project structure:

```
your-project/
├── policies/
│   ├── budgets.yaml
│   ├── routing.yaml
│   └── pricing.yaml
├── src/
│   └── your_agent.py
└── main.py
```

## Policy Refresh

Policies are cached in memory and refreshed periodically:

1. On initialization, policies are loaded from source
2. Before each policy lookup, the refresh interval is checked
3. If elapsed time exceeds `policy_refresh_interval_seconds`, policies reload
4. If reload fails, the last known good configuration is used (fail-open)

To force immediate refresh:

```python
guard._policy_store.refresh()
```

## Disabling Features

You can selectively disable Cost Guard features:

```python
# Cost tracking without enforcement
config = CostGuardConfig(
    policy_source=source,
    enable_budget_enforcement=False,  # Track costs but don't enforce limits
    enable_routing=True,
    enable_metrics=True,
)

# No routing, just budget enforcement
config = CostGuardConfig(
    policy_source=source,
    enable_budget_enforcement=True,
    enable_routing=False,  # Always use requested model
    enable_metrics=True,
)

# Metrics only (no enforcement or routing)
config = CostGuardConfig(
    policy_source=source,
    enable_budget_enforcement=False,
    enable_routing=False,
    enable_metrics=True,
)
```

## Logging Configuration

Cost Guard uses Python's standard logging:

```python
import logging

# Configure Cost Guard logging
logging.getLogger("strands_costguard").setLevel(logging.DEBUG)

# Or configure specific modules
logging.getLogger("strands_costguard.core.cost_guard").setLevel(logging.INFO)
logging.getLogger("strands_costguard.policies.store").setLevel(logging.WARNING)
```

Log messages include:
- Policy load/refresh events
- Budget threshold crossings
- Model downgrades
- Run rejections
- Errors and warnings

## High Cardinality Warning

The `include_run_id_in_metrics` option adds `strands.run_id` to all metric attributes. This can cause high cardinality in your metrics backend if you have many runs.

Only enable this if:
- You need per-run cost attribution in metrics
- Your metrics backend can handle the cardinality
- You have appropriate retention policies

For per-run costs without high cardinality metrics, use `guard.get_run_cost(run_id)` instead.
