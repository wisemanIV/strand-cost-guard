# Budget Policies

Budget policies define spending limits and enforcement actions for your agent system.

## Budget Specification

Budgets are defined in `budgets.yaml`:

```yaml
budgets:
  - id: "tenant-production"
    scope: "tenant"
    match:
      tenant_id: "prod-tenant-001"
    period: "monthly"
    max_cost: 5000.0
    soft_thresholds: [0.7, 0.9, 0.95]
    hard_limit: true
    on_soft_threshold_exceeded: "DOWNGRADE_MODEL"
    on_hard_limit_exceeded: "REJECT_NEW_RUNS"
    max_runs_per_period: 50000
    max_concurrent_runs: 100
    constraints:
      max_iterations_per_run: 10
      max_tool_calls_per_run: 30
      max_model_tokens_per_run: 50000
```

## Fields Reference

### Core Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `id` | `string` | Yes | Unique identifier for this budget |
| `scope` | `string` | Yes | Scope level: `global`, `tenant`, `strand`, `workflow` |
| `match` | `object` | Yes | Criteria for matching this budget to contexts |
| `enabled` | `boolean` | No | Whether this budget is active (default: `true`) |

### Match Criteria

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `tenant_id` | `string` | `"*"` | Tenant ID to match (`"*"` matches all) |
| `strand_id` | `string` | `"*"` | Strand ID to match (`"*"` matches all) |
| `workflow_id` | `string` | `"*"` | Workflow ID to match (`"*"` matches all) |

### Budget Limits

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `period` | `string` | `"monthly"` | Budget period: `hourly`, `daily`, `weekly`, `monthly` |
| `max_cost` | `float` | `null` | Maximum cost in currency units for the period |
| `max_runs_per_period` | `int` | `null` | Maximum number of runs per period |
| `max_concurrent_runs` | `int` | `null` | Maximum concurrent active runs |

### Threshold Configuration

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `soft_thresholds` | `list[float]` | `[0.7, 0.9, 1.0]` | Budget utilization thresholds (0.0-1.0) |
| `hard_limit` | `boolean` | `true` | Whether to enforce hard limit at 100% |
| `on_soft_threshold_exceeded` | `string` | `"LOG_ONLY"` | Action when soft threshold crossed |
| `on_hard_limit_exceeded` | `string` | `"REJECT_NEW_RUNS"` | Action when hard limit exceeded |

### Per-Run Constraints

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `constraints.max_iterations_per_run` | `int` | Config default | Max agent loop iterations |
| `constraints.max_tool_calls_per_run` | `int` | Config default | Max tool calls per run |
| `constraints.max_model_tokens_per_run` | `int` | Config default | Max total tokens per run |
| `constraints.max_cost_per_run` | `float` | `null` | Max cost per individual run |

## Soft Thresholds

Soft thresholds define warning points as the budget is consumed. Values are **percentages expressed as decimals between 0.0 and 1.0**.

```yaml
soft_thresholds: [0.7, 0.9, 0.95]
```

With `max_cost: 1000.0`, this means:
- **0.7** = 70% = $700 - First threshold
- **0.9** = 90% = $900 - Second threshold
- **0.95** = 95% = $950 - Third threshold

When any threshold is crossed, the `on_soft_threshold_exceeded` action triggers.

### Soft Threshold Actions

| Action | Description |
|--------|-------------|
| `LOG_ONLY` | Log a warning but continue normal operation |
| `DOWNGRADE_MODEL` | Trigger model downgrade via routing policy |
| `LIMIT_CAPABILITIES` | Reduce token limits and iteration counts |
| `HALT_NEW_RUNS` | Reject new runs but allow current runs to complete |

## Hard Limits

When `hard_limit: true` and utilization reaches 100%, the `on_hard_limit_exceeded` action triggers.

### Hard Limit Actions

| Action | Description |
|--------|-------------|
| `HALT_RUN` | Stop the currently executing run immediately |
| `REJECT_NEW_RUNS` | Only reject new runs; existing runs continue |

## Budget Scopes

Budgets are defined at different scopes with increasing specificity:

```
global (lowest priority)
  └── tenant
        └── strand
              └── workflow (highest priority)
```

### Scope Definitions

| Scope | Description | Use Case |
|-------|-------------|----------|
| `global` | Applies to all tenants | Platform-wide defaults |
| `tenant` | Applies to a specific tenant | Organization limits |
| `strand` | Applies to a specific agent type | Per-agent budgets |
| `workflow` | Applies to a specific workflow | Task-specific limits |

### Priority and Matching

When multiple budgets match a context, they are all applied. More specific budgets take priority for conflicting settings.

**Priority calculation:**
1. Scope priority: `workflow` > `strand` > `tenant` > `global`
2. Match specificity: Non-wildcard matches add to priority

Example with priority scores:
```yaml
# Priority: 0 (global scope, all wildcards)
- id: "global-default"
  scope: "global"
  match:
    tenant_id: "*"

# Priority: 10 (tenant scope)
- id: "tenant-prod"
  scope: "tenant"
  match:
    tenant_id: "prod-001"

# Priority: 22 (strand scope + specific strand match)
- id: "analytics-strand"
  scope: "strand"
  match:
    tenant_id: "*"
    strand_id: "analytics"

# Priority: 34 (workflow scope + specific workflow match)
- id: "daily-report-workflow"
  scope: "workflow"
  match:
    workflow_id: "daily_report"
```

## Budget Periods

Budget periods determine when usage counters reset.

| Period | Reset Time |
|--------|------------|
| `hourly` | Start of each hour (XX:00:00) |
| `daily` | Midnight UTC (00:00:00) |
| `weekly` | Monday midnight UTC |
| `monthly` | First day of month, midnight UTC |

## Examples

### Global Default Budget

```yaml
budgets:
  - id: "global-default"
    scope: "global"
    match:
      tenant_id: "*"
    period: "monthly"
    max_cost: 10000.0
    soft_thresholds: [0.8, 0.95]
    hard_limit: true
    on_soft_threshold_exceeded: "LOG_ONLY"
    on_hard_limit_exceeded: "REJECT_NEW_RUNS"
```

### Per-Tenant Budget

```yaml
budgets:
  - id: "enterprise-tenant"
    scope: "tenant"
    match:
      tenant_id: "enterprise-001"
    period: "monthly"
    max_cost: 50000.0
    soft_thresholds: [0.7, 0.9]
    on_soft_threshold_exceeded: "DOWNGRADE_MODEL"
    max_concurrent_runs: 500

  - id: "starter-tenant"
    scope: "tenant"
    match:
      tenant_id: "starter-*"  # Wildcard prefix matching
    period: "monthly"
    max_cost: 100.0
    soft_thresholds: [0.5, 0.8]
    on_soft_threshold_exceeded: "LIMIT_CAPABILITIES"
    max_concurrent_runs: 10
```

### Cost-Sensitive Agent Budget

```yaml
budgets:
  - id: "expensive-agent"
    scope: "strand"
    match:
      strand_id: "code_generator"
    period: "daily"
    max_cost: 500.0
    soft_thresholds: [0.6, 0.8, 0.95]
    on_soft_threshold_exceeded: "DOWNGRADE_MODEL"
    on_hard_limit_exceeded: "HALT_RUN"
    constraints:
      max_iterations_per_run: 20
      max_tool_calls_per_run: 50
      max_model_tokens_per_run: 100000
```

### Development Budget (No Hard Limits)

```yaml
budgets:
  - id: "dev-environment"
    scope: "tenant"
    match:
      tenant_id: "dev-*"
    period: "daily"
    max_cost: 100.0
    soft_thresholds: [0.9]
    hard_limit: false  # Never block, just warn
    on_soft_threshold_exceeded: "LOG_ONLY"
    constraints:
      max_iterations_per_run: 100  # Allow more for debugging
```

## Querying Budget Status

```python
# Get budget summary for a context
summary = guard.get_budget_summary(
    tenant_id="prod-001",
    strand_id="analytics",
    workflow_id="report",
)

for budget_id, stats in summary.items():
    print(f"{budget_id}:")
    print(f"  Utilization: {stats['utilization']:.1%}")
    print(f"  Remaining: ${stats['remaining']:.2f}")
    print(f"  Period: {stats['period_start']} to {stats['period_end']}")
```

## Best Practices

1. **Start with global defaults** - Define a catch-all budget for safety
2. **Use soft thresholds for early warning** - Set thresholds well below 100%
3. **Match budget periods to billing cycles** - Align with your cost reporting
4. **Set per-run constraints** - Prevent runaway individual runs
5. **Use `DOWNGRADE_MODEL` for graceful degradation** - Maintain service availability
6. **Test with `hard_limit: false`** - Verify behavior before enforcing
