# OpenTelemetry Metrics

Cost Guard emits OpenTelemetry-compatible metrics for cost tracking and observability.

## Setup

Cost Guard uses the global MeterProvider configured by StrandsTelemetry. Configure telemetry before creating a CostGuard instance:

```python
from strands.telemetry.config import StrandsTelemetry
from strands_costguard import CostGuard, CostGuardConfig, FilePolicySource

# Configure telemetry first
telemetry = StrandsTelemetry()
telemetry.setup_otlp_exporter(endpoint="http://localhost:4317")
telemetry.setup_meter(enable_otlp_exporter=True)

# Cost Guard uses the global MeterProvider automatically
config = CostGuardConfig(
    policy_source=FilePolicySource(path="./policies"),
    enable_metrics=True,
)
guard = CostGuard(config=config)
```

For development/testing with console output:

```python
telemetry = StrandsTelemetry()
telemetry.setup_console_exporter()
telemetry.setup_meter(enable_console_exporter=True)
```

## Metrics Reference

### Cost Metrics

| Metric Name | Type | Unit | Description |
|-------------|------|------|-------------|
| `genai.cost.total` | Counter | `{currency}` | Total cost across all operations |
| `genai.cost.model` | Counter | `{currency}` | Cost attributed to model calls |
| `genai.cost.tool` | Counter | `{currency}` | Cost attributed to tool calls |

### Token Metrics

| Metric Name | Type | Unit | Description |
|-------------|------|------|-------------|
| `genai.tokens.input` | Counter | `{token}` | Total input/prompt tokens |
| `genai.tokens.output` | Counter | `{token}` | Total output/completion tokens |

### Activity Metrics

| Metric Name | Type | Unit | Description |
|-------------|------|------|-------------|
| `genai.agent.runs` | Counter | `{run}` | Agent run events (start/end) |
| `genai.agent.iterations` | Counter | `{iteration}` | Agent loop iterations |
| `genai.agent.tool_calls` | Counter | `{call}` | Tool invocations |

### Event Metrics

| Metric Name | Type | Unit | Description |
|-------------|------|------|-------------|
| `genai.cost.downgrade_events` | Counter | `{event}` | Model downgrade events |
| `genai.cost.rejection_events` | Counter | `{event}` | Run rejection events |
| `genai.cost.halt_events` | Counter | `{event}` | Run halt events |

## Metric Attributes

All metrics include these base attributes:

| Attribute | Description |
|-----------|-------------|
| `strands.tenant_id` | Tenant/organization identifier |
| `strands.strand_id` | Agent/strand identifier |
| `strands.workflow_id` | Workflow identifier |

### Optional Attributes

| Attribute | When Present | Description |
|-----------|--------------|-------------|
| `strands.run_id` | When `include_run_id_in_metrics=True` | Unique run identifier |
| `strands.event` | Run events | Event type (`start`, `end`) |
| `strands.status` | Run end events | Final status |
| `strands.iteration_idx` | Iteration events | Iteration index |
| `strands.reason` | Rejection/halt events | Reason for event |
| `genai.model.name` | Model cost events | Model used |
| `genai.model.original` | Downgrade events | Original model |
| `genai.model.fallback` | Downgrade events | Fallback model |
| `strands.tool.name` | Tool cost events | Tool name |

### Custom Metadata

Metadata passed to `on_run_start` is included as attributes:

```python
guard.on_run_start(
    tenant_id="prod-001",
    strand_id="assistant",
    workflow_id="chat",
    run_id="run-123",
    metadata={
        "user_id": "user-abc",
        "environment": "production",
    },
)
# Metrics include:
#   strands.metadata.user_id = "user-abc"
#   strands.metadata.environment = "production"
```

## Example Queries

### Prometheus/PromQL

```promql
# Total cost by tenant (last 24h)
sum by (strands_tenant_id) (
  increase(genai_cost_total[24h])
)

# Model costs by model name
sum by (genai_model_name) (
  increase(genai_cost_model[1h])
)

# Downgrade rate by strand
sum by (strands_strand_id) (
  increase(genai_cost_downgrade_events[1h])
)

# Token usage by tenant
sum by (strands_tenant_id) (
  increase(genai_tokens_input[24h]) + increase(genai_tokens_output[24h])
)

# Rejection rate
sum(increase(genai_cost_rejection_events[1h])) /
sum(increase(genai_agent_runs{strands_event="start"}[1h]))
```

### Grafana Dashboard Panels

**Cost by Tenant:**
```json
{
  "type": "timeseries",
  "title": "Cost by Tenant",
  "targets": [{
    "expr": "sum by (strands_tenant_id) (increase(genai_cost_total[$__interval]))",
    "legendFormat": "{{strands_tenant_id}}"
  }]
}
```

**Model Usage Distribution:**
```json
{
  "type": "piechart",
  "title": "Model Cost Distribution",
  "targets": [{
    "expr": "sum by (genai_model_name) (increase(genai_cost_model[24h]))",
    "legendFormat": "{{genai_model_name}}"
  }]
}
```

## High Cardinality Considerations

By default, `run_id` is not included in metrics to avoid high cardinality issues. Each unique combination of attribute values creates a new time series.

### When to Enable Run ID

Enable `include_run_id_in_metrics=True` only if:
- You need per-run cost attribution in your metrics system
- Your backend can handle the cardinality (e.g., with aggregation)
- You have appropriate retention/downsampling policies

### Alternatives to Run ID

For per-run analysis without high cardinality:

1. **Use `get_run_cost()`** - Query cost at runtime
   ```python
   cost = guard.get_run_cost(run_id)
   ```

2. **Log to separate system** - Send run-level data to logs
   ```python
   logger.info(f"Run completed", extra={
       "run_id": run_id,
       "cost": guard.get_run_cost(run_id),
   })
   ```

3. **Use traces instead** - OpenTelemetry traces naturally support per-run data

## Disabling Metrics

To disable metrics entirely:

```python
config = CostGuardConfig(
    policy_source=source,
    enable_metrics=False,  # No metrics emitted
)
```

Or skip telemetry setup:

```python
# Without StrandsTelemetry.setup_meter(), metrics will fail silently
guard = CostGuard(config=config)  # Logs warning but continues
```

## Integration with Existing Telemetry

Cost Guard metrics integrate with your existing OpenTelemetry setup:

```python
from opentelemetry import metrics
from opentelemetry.sdk.metrics import MeterProvider
from opentelemetry.sdk.metrics.export import PeriodicExportingMetricReader
from opentelemetry.exporter.otlp.proto.grpc.metric_exporter import OTLPMetricExporter

# Your existing OTEL setup
exporter = OTLPMetricExporter(endpoint="http://collector:4317")
reader = PeriodicExportingMetricReader(exporter)
provider = MeterProvider(metric_readers=[reader])
metrics.set_meter_provider(provider)

# Cost Guard uses the global provider
guard = CostGuard(config=config)  # Metrics flow to your collector
```

## Metric Export Intervals

Metrics are exported according to your OpenTelemetry configuration. StrandsTelemetry defaults are typically:

- OTLP export: Every 60 seconds
- Console export: Every 5 seconds (for debugging)

Adjust in your StrandsTelemetry or OpenTelemetry SDK configuration.

## Troubleshooting

### No Metrics Appearing

1. Verify StrandsTelemetry is configured:
   ```python
   telemetry.setup_meter(enable_otlp_exporter=True)
   ```

2. Check `enable_metrics=True` in CostGuardConfig

3. Look for initialization warnings in logs:
   ```
   WARNING - Failed to initialize Cost Guard metrics: ...
   ```

4. Verify collector endpoint is reachable

### Missing Attributes

Attributes only appear when relevant:
- `genai.model.name` only on model cost metrics
- `strands.reason` only on rejection/halt events

### High Cardinality Warnings

If your metrics backend warns about cardinality:
1. Ensure `include_run_id_in_metrics=False`
2. Limit unique values in metadata
3. Use aggregation rules in your collector
