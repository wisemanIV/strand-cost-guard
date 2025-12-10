# API Reference

Complete API documentation for Strand Cost Guard.

## Core Classes

### CostGuard

Main entry point for budget enforcement and cost tracking.

```python
from strands_costguard import CostGuard, CostGuardConfig, FilePolicySource

config = CostGuardConfig(
    policy_source=FilePolicySource(path="./policies"),
)
guard = CostGuard(config=config)
```

#### Lifecycle Methods

##### on_run_start

```python
def on_run_start(
    self,
    tenant_id: str,
    strand_id: str,
    workflow_id: str,
    run_id: str,
    metadata: Optional[dict[str, str]] = None,
) -> AdmissionDecision
```

Start a new run and check budget admission.

**Parameters:**
- `tenant_id` - Organization/tenant identifier
- `strand_id` - Agent/strand identifier
- `workflow_id` - Workflow/task identifier
- `run_id` - Unique run identifier
- `metadata` - Optional attributes for metrics

**Returns:** `AdmissionDecision`

---

##### on_run_end

```python
def on_run_end(self, run_id: str, status: str) -> None
```

End a run and record final metrics.

**Parameters:**
- `run_id` - The run identifier
- `status` - Final status (e.g., "completed", "failed")

---

##### before_iteration

```python
def before_iteration(
    self,
    run_id: str,
    iteration_idx: int,
    context: Optional[dict[str, Any]] = None,
) -> IterationDecision
```

Check if an iteration can proceed.

**Parameters:**
- `run_id` - The run identifier
- `iteration_idx` - Current iteration index (0-based)
- `context` - Optional context data

**Returns:** `IterationDecision`

---

##### after_iteration

```python
def after_iteration(
    self,
    run_id: str,
    iteration_idx: int,
    usage: IterationUsage,
) -> None
```

Record metrics for a completed iteration.

**Parameters:**
- `run_id` - The run identifier
- `iteration_idx` - Completed iteration index
- `usage` - Usage metrics for the iteration

---

##### before_model_call

```python
def before_model_call(
    self,
    run_id: str,
    model_name: str,
    stage: str = "other",
    prompt_tokens_estimate: int = 0,
) -> ModelDecision
```

Get model selection decision before a call.

**Parameters:**
- `run_id` - The run identifier
- `model_name` - Requested model name
- `stage` - Semantic stage ("planning", "tool_selection", "synthesis", "other")
- `prompt_tokens_estimate` - Estimated input tokens

**Returns:** `ModelDecision`

---

##### after_model_call

```python
def after_model_call(
    self,
    run_id: str,
    usage: ModelUsage,
) -> None
```

Record usage from a completed model call.

**Parameters:**
- `run_id` - The run identifier
- `usage` - Usage metrics from the call

---

##### before_tool_call

```python
def before_tool_call(
    self,
    run_id: str,
    tool_name: str,
) -> ToolDecision
```

Check if a tool call can proceed.

**Parameters:**
- `run_id` - The run identifier
- `tool_name` - Name of the tool

**Returns:** `ToolDecision`

---

##### after_tool_call

```python
def after_tool_call(
    self,
    run_id: str,
    tool_name: str,
    cost_metadata: ToolUsage,
) -> None
```

Record usage from a completed tool call.

**Parameters:**
- `run_id` - The run identifier
- `tool_name` - Name of the tool
- `cost_metadata` - Usage metrics from the call

---

#### Query Methods

##### get_run_cost

```python
def get_run_cost(self, run_id: str) -> Optional[float]
```

Get current total cost for a run.

**Returns:** Cost in currency units, or `None` if run not found.

---

##### get_budget_summary

```python
def get_budget_summary(
    self,
    tenant_id: str,
    strand_id: str,
    workflow_id: str,
) -> dict[str, dict]
```

Get budget usage summary for a context.

**Returns:** Dictionary mapping budget IDs to usage statistics.

---

##### shutdown

```python
def shutdown(self) -> None
```

Clean up resources. Note: Metrics flushing is handled by StrandsTelemetry.

---

### CostGuardConfig

Configuration for CostGuard.

```python
@dataclass
class CostGuardConfig:
    policy_source: PolicySource                    # Required
    failure_mode: FailureMode = FailureMode.FAIL_OPEN
    policy_refresh_interval_seconds: int = 300
    enable_budget_enforcement: bool = True
    enable_routing: bool = True
    enable_metrics: bool = True
    include_run_id_in_metrics: bool = False
    currency: str = "USD"
    default_max_iterations_per_run: int = 50
    default_max_tool_calls_per_run: int = 100
    default_max_tokens_per_run: int = 100000
    log_level: str = "INFO"
```

---

## Decision Classes

### AdmissionDecision

Decision for run admission.

```python
@dataclass
class AdmissionDecision:
    allowed: bool
    reason: Optional[str] = None
    action: DecisionAction = DecisionAction.ALLOW
    remaining_budget: Optional[float] = None
    budget_utilization: Optional[float] = None
    warnings: list[str] = field(default_factory=list)
```

**Class Methods:**
- `admit(remaining_budget, budget_utilization, warnings)` - Create allowing decision
- `reject(reason)` - Create rejecting decision

---

### IterationDecision

Decision for iteration continuation.

```python
@dataclass
class IterationDecision:
    allowed: bool
    reason: Optional[str] = None
    action: DecisionAction = DecisionAction.ALLOW
    action_overrides: ActionOverrides = field(default_factory=ActionOverrides)
    remaining_iterations: Optional[int] = None
    remaining_budget: Optional[float] = None
    warnings: list[str] = field(default_factory=list)
```

**Class Methods:**
- `proceed(remaining_iterations, remaining_budget, warnings)` - Create proceeding decision
- `halt(reason)` - Create halting decision

---

### ModelDecision

Decision for model call, including routing.

```python
@dataclass
class ModelDecision:
    allowed: bool
    reason: Optional[str] = None
    action: DecisionAction = DecisionAction.ALLOW
    action_overrides: ActionOverrides = field(default_factory=ActionOverrides)
    effective_model: Optional[str] = None
    max_tokens: Optional[int] = None
    remaining_tokens: Optional[int] = None
    remaining_budget: Optional[float] = None
    was_downgraded: bool = False
    warnings: list[str] = field(default_factory=list)
```

**Class Methods:**
- `allow(effective_model, max_tokens, remaining_budget, warnings)` - Create allowing decision
- `downgrade(original_model, fallback_model, reason, max_tokens)` - Create downgrade decision
- `reject(reason)` - Create rejecting decision

---

### ToolDecision

Decision for tool call.

```python
@dataclass
class ToolDecision:
    allowed: bool
    reason: Optional[str] = None
    action: DecisionAction = DecisionAction.ALLOW
    action_overrides: ActionOverrides = field(default_factory=ActionOverrides)
    remaining_tool_calls: Optional[int] = None
    remaining_budget: Optional[float] = None
    warnings: list[str] = field(default_factory=list)
```

**Class Methods:**
- `allow(remaining_tool_calls, remaining_budget, warnings)` - Create allowing decision
- `reject(reason)` - Create rejecting decision

---

### ActionOverrides

Runtime behavior modifications.

```python
@dataclass
class ActionOverrides:
    model_name: Optional[str] = None
    max_tokens_remaining: Optional[int] = None
    force_terminate_run: bool = False
    skip_tool_call: bool = False
    fallback_response: Optional[str] = None
```

---

### DecisionAction

Enum of possible actions.

```python
class DecisionAction(str, Enum):
    ALLOW = "allow"
    REJECT = "reject"
    DOWNGRADE = "downgrade"
    LIMIT = "limit"
    HALT = "halt"
    LOG_ONLY = "log_only"
```

---

## Usage Classes

### ModelUsage

Usage metrics from a model call.

```python
@dataclass
class ModelUsage:
    model_name: str
    prompt_tokens: int
    completion_tokens: int
    total_tokens: int
    cost: float = 0.0
    latency_ms: Optional[float] = None
    cached_tokens: int = 0
    reasoning_tokens: int = 0
```

**Class Methods:**

```python
@classmethod
def from_response(
    cls,
    model_name: str,
    prompt_tokens: int,
    completion_tokens: int,
    cost: float = 0.0,
    latency_ms: Optional[float] = None,
    cached_tokens: int = 0,
    reasoning_tokens: int = 0,
) -> ModelUsage
```

---

### ToolUsage

Usage metrics from a tool call.

```python
@dataclass
class ToolUsage:
    tool_name: str
    cost: float = 0.0
    latency_ms: Optional[float] = None
    input_size_bytes: int = 0
    output_size_bytes: int = 0
    success: bool = True
    error_type: Optional[str] = None
    metadata: dict[str, str] = field(default_factory=dict)
```

---

### IterationUsage

Aggregated usage for an iteration.

```python
@dataclass
class IterationUsage:
    iteration_idx: int
    model_calls: list[ModelUsage] = field(default_factory=list)
    tool_calls: list[ToolUsage] = field(default_factory=list)
    total_cost: float = 0.0
    total_input_tokens: int = 0
    total_output_tokens: int = 0
```

**Methods:**
- `add_model_usage(usage: ModelUsage)` - Add model call usage
- `add_tool_usage(usage: ToolUsage)` - Add tool call usage

**Properties:**
- `total_tokens` - Sum of input and output tokens
- `num_model_calls` - Count of model calls
- `num_tool_calls` - Count of tool calls

---

## Policy Classes

### BudgetSpec

Budget specification from YAML.

```python
@dataclass
class BudgetSpec:
    id: str
    scope: BudgetScope
    match: BudgetMatch
    period: BudgetPeriod = BudgetPeriod.MONTHLY
    max_cost: Optional[float] = None
    soft_thresholds: list[float] = field(default_factory=lambda: [0.7, 0.9, 1.0])
    hard_limit: bool = True
    on_soft_threshold_exceeded: ThresholdAction = ThresholdAction.LOG_ONLY
    on_hard_limit_exceeded: HardLimitAction = HardLimitAction.REJECT_NEW_RUNS
    max_runs_per_period: Optional[int] = None
    max_concurrent_runs: Optional[int] = None
    constraints: BudgetConstraints = field(default_factory=BudgetConstraints)
    enabled: bool = True
```

**Class Methods:**
- `from_dict(data: dict)` - Create from dictionary

**Methods:**
- `get_priority()` - Get priority score for merging
- `matches_context(tenant_id, strand_id, workflow_id)` - Check if budget applies
- `get_current_threshold_action(utilization)` - Get action for current utilization
- `is_hard_limit_exceeded(utilization)` - Check if hard limit exceeded

---

### BudgetScope

```python
class BudgetScope(str, Enum):
    GLOBAL = "global"
    TENANT = "tenant"
    STRAND = "strand"
    WORKFLOW = "workflow"
```

---

### BudgetPeriod

```python
class BudgetPeriod(str, Enum):
    HOURLY = "hourly"
    DAILY = "daily"
    WEEKLY = "weekly"
    MONTHLY = "monthly"
```

---

### ThresholdAction

```python
class ThresholdAction(str, Enum):
    LOG_ONLY = "LOG_ONLY"
    DOWNGRADE_MODEL = "DOWNGRADE_MODEL"
    LIMIT_CAPABILITIES = "LIMIT_CAPABILITIES"
    HALT_NEW_RUNS = "HALT_NEW_RUNS"
```

---

### HardLimitAction

```python
class HardLimitAction(str, Enum):
    HALT_RUN = "HALT_RUN"
    REJECT_NEW_RUNS = "REJECT_NEW_RUNS"
```

---

### RoutingPolicy

Routing policy from YAML.

```python
@dataclass
class RoutingPolicy:
    id: str
    match: dict[str, str] = field(default_factory=dict)
    stages: list[StageConfig] = field(default_factory=list)
    default_model: str = "gpt-4o-mini"
    default_fallback_model: Optional[str] = None
    enabled: bool = True
```

**Class Methods:**
- `from_dict(data: dict)` - Create from dictionary

**Methods:**
- `matches_context(tenant_id, strand_id, workflow_id)` - Check if policy applies
- `get_stage_config(stage)` - Get config for a stage
- `get_model_for_stage(stage, soft_threshold_exceeded, remaining_budget, iteration_count, avg_latency_ms)` - Get effective model

---

### StageConfig

Configuration for a model call stage.

```python
@dataclass
class StageConfig:
    stage: str
    default_model: str
    fallback_model: Optional[str] = None
    max_tokens: Optional[int] = None
    temperature: Optional[float] = None
    trigger_downgrade_on: DowngradeTrigger = field(default_factory=DowngradeTrigger)
```

**Methods:**
- `get_effective_model(soft_threshold_exceeded, remaining_budget, iteration_count, avg_latency_ms)` - Get model to use

---

### DowngradeTrigger

Conditions for model downgrade.

```python
@dataclass
class DowngradeTrigger:
    soft_threshold_exceeded: bool = False
    remaining_budget_below: Optional[float] = None
    iteration_count_above: Optional[int] = None
    latency_above_ms: Optional[float] = None
```

**Methods:**
- `should_downgrade(soft_threshold_exceeded, remaining_budget, iteration_count, avg_latency_ms)` - Check if should downgrade

---

## Policy Sources

### FilePolicySource

Load policies from YAML files.

```python
@dataclass
class FilePolicySource:
    path: str
    budgets_file: str = "budgets.yaml"
    routing_file: str = "routing.yaml"
    pricing_file: str = "pricing.yaml"
```

**Methods:**
- `load_budgets()` - Load budget specifications
- `load_routing_policies()` - Load routing policies
- `load_pricing()` - Load pricing table

---

### EnvPolicySource

Load policies from environment variables.

```python
@dataclass
class EnvPolicySource:
    prefix: str = "COST_GUARD_"
```

**Methods:**
- `load_budgets()` - Load budget from env vars
- `load_routing_policies()` - Load routing from env vars
- `load_pricing()` - Returns empty dict (not supported)

---

### PolicyStore

Central policy store with caching.

```python
@dataclass
class PolicyStore:
    source: FilePolicySource | EnvPolicySource
    refresh_interval_seconds: int = 300
```

**Methods:**
- `refresh()` - Reload policies from source
- `get_budgets_for_context(tenant_id, strand_id, workflow_id)` - Get matching budgets
- `get_effective_budget(tenant_id, strand_id, workflow_id, scope)` - Get highest priority budget
- `get_routing_policy(tenant_id, strand_id, workflow_id)` - Get matching routing policy
- `get_pricing()` - Get pricing table

**Properties:**
- `budgets` - All loaded budgets
- `routing_policies` - All loaded routing policies

---

## Pricing Classes

### PricingTable

Central pricing table.

```python
@dataclass
class PricingTable:
    currency: str = "USD"
    models: dict[str, ModelPricing] = field(default_factory=dict)
    tools: dict[str, ToolPricing] = field(default_factory=dict)
    fallback_input_per_1k: float = 1.0
    fallback_output_per_1k: float = 3.0
```

**Class Methods:**
- `from_dict(data: dict)` - Create from dictionary

**Methods:**
- `get_model_pricing(model_name)` - Get pricing for a model
- `get_tool_pricing(tool_name)` - Get pricing for a tool
- `calculate_model_cost(model_name, prompt_tokens, completion_tokens, cached_tokens, reasoning_tokens)` - Calculate model cost
- `calculate_tool_cost(tool_name, input_size_bytes, output_size_bytes)` - Calculate tool cost
- `estimate_model_cost(model_name, estimated_input_tokens, estimated_output_tokens)` - Estimate cost

---

### ModelPricing

Pricing for a single model.

```python
@dataclass
class ModelPricing:
    model_name: str
    input_per_1k: float
    output_per_1k: float
    currency: str = "USD"
    cached_input_per_1k: Optional[float] = None
    reasoning_per_1k: Optional[float] = None
```

**Methods:**
- `calculate_cost(prompt_tokens, completion_tokens, cached_tokens, reasoning_tokens)` - Calculate total cost
- `estimate_cost(estimated_tokens, is_input)` - Estimate cost

---

### ToolPricing

Pricing for a single tool.

```python
@dataclass
class ToolPricing:
    tool_name: str
    cost_per_call: float = 0.0
    cost_per_input_byte: float = 0.0
    cost_per_output_byte: float = 0.0
    currency: str = "USD"
```

**Methods:**
- `calculate_cost(input_size_bytes, output_size_bytes)` - Calculate total cost

---

## Router Classes

### ModelRouter

Helper for model call integration.

```python
@dataclass
class ModelRouter:
    cost_guard: Any  # CostGuard
    config: RouterConfig = field(default_factory=RouterConfig)
```

**Methods:**

```python
def before_call(
    self,
    run_id: str,
    stage: str,
    messages: list[dict[str, Any]],
    requested_model: Optional[str] = None,
) -> ModelCallContext
```

Prepare for a model call.

```python
def after_call(
    self,
    run_id: str,
    response: dict[str, Any],
    model_name: Optional[str] = None,
) -> ModelUsage
```

Record usage after a call.

```python
def call(
    self,
    run_id: str,
    stage: str,
    messages: list[dict[str, Any]],
    client: ModelClient,
    requested_model: Optional[str] = None,
    **kwargs: Any,
) -> tuple[dict[str, Any], ModelUsage]
```

Make a complete model call through the router.

---

### ModelCallContext

Context returned by `before_call`.

```python
@dataclass
class ModelCallContext:
    run_id: str
    stage: str
    requested_model: str
    effective_model: str
    max_tokens: Optional[int]
    allowed: bool
    was_downgraded: bool
    reason: Optional[str]
    warnings: list[str]
    prompt_tokens_estimate: int
```

**Methods:**
- `to_dict()` - Convert to dictionary

---

### RouterConfig

Configuration for ModelRouter.

```python
@dataclass
class RouterConfig:
    retry_on_rate_limit: bool = True
    max_retries: int = 3
    retry_delay_seconds: float = 1.0
    track_latency: bool = True
```

---

## Entity Classes

### RunContext

Immutable context for a run.

```python
@dataclass(frozen=True)
class RunContext:
    tenant_id: str
    strand_id: str
    workflow_id: str
    run_id: str
    metadata: dict[str, str] = field(default_factory=dict)
    started_at: datetime = field(default_factory=datetime.utcnow)
```

**Class Methods:**
- `create(tenant_id, strand_id, workflow_id, run_id, metadata)` - Create new context

**Methods:**
- `to_attributes()` - Convert to OpenTelemetry attributes

---

### RunState

Mutable state for a run.

```python
@dataclass
class RunState:
    context: RunContext
    current_iteration: int = 0
    total_cost: float = 0.0
    total_input_tokens: int = 0
    total_output_tokens: int = 0
    total_tool_calls: int = 0
    model_costs: dict[str, float] = field(default_factory=dict)
    tool_costs: dict[str, float] = field(default_factory=dict)
    status: str = "running"
    ended_at: Optional[datetime] = None
```

**Methods:**
- `add_model_cost(model_name, cost, input_tokens, output_tokens)` - Add model cost
- `add_tool_cost(tool_name, cost)` - Add tool cost
- `increment_iteration()` - Increment and return iteration
- `end(status)` - Mark run as ended
