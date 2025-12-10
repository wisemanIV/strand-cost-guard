# Lifecycle Hooks

Cost Guard integrates with agent runtimes through lifecycle hooks that are called at key points during execution.

## Hook Overview

| Hook | When Called | Returns | Purpose |
|------|-------------|---------|---------|
| `on_run_start()` | Before starting a new run | `AdmissionDecision` | Approve/reject run, check budgets |
| `on_run_end()` | After a run completes | `None` | Record final costs, cleanup |
| `before_iteration()` | Before each agent loop iteration | `IterationDecision` | Check iteration limits |
| `after_iteration()` | After each iteration completes | `None` | Record iteration metrics |
| `before_model_call()` | Before each model call | `ModelDecision` | Select model, check limits |
| `after_model_call()` | After each model call | `None` | Record usage and costs |
| `before_tool_call()` | Before each tool call | `ToolDecision` | Check tool call limits |
| `after_tool_call()` | After each tool call | `None` | Record tool costs |

## Run Lifecycle

### on_run_start

Called when a new agent run begins.

```python
admission = guard.on_run_start(
    tenant_id="prod-tenant",
    strand_id="analytics",
    workflow_id="report",
    run_id="run-123",
    metadata={"user_id": "user-abc", "priority": "high"},
)
```

**Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `tenant_id` | `str` | Yes | Organization/tenant identifier |
| `strand_id` | `str` | Yes | Agent/strand identifier |
| `workflow_id` | `str` | Yes | Workflow/task identifier |
| `run_id` | `str` | Yes | Unique run identifier |
| `metadata` | `dict[str, str]` | No | Additional attributes for metrics |

**Returns:** `AdmissionDecision`

```python
@dataclass
class AdmissionDecision:
    allowed: bool                      # Whether the run can proceed
    reason: Optional[str]              # Rejection reason if not allowed
    action: DecisionAction             # ALLOW or REJECT
    remaining_budget: Optional[float]  # Remaining budget after this run
    budget_utilization: Optional[float] # Current utilization (0.0-1.0)
    warnings: list[str]                # Budget warnings
```

**Example:**

```python
admission = guard.on_run_start(
    tenant_id="prod-001",
    strand_id="assistant",
    workflow_id="chat",
    run_id=str(uuid.uuid4()),
)

if not admission.allowed:
    raise RuntimeError(f"Run rejected: {admission.reason}")

if admission.warnings:
    logger.warning(f"Budget warnings: {admission.warnings}")

print(f"Remaining budget: ${admission.remaining_budget:.2f}")
print(f"Utilization: {admission.budget_utilization:.1%}")
```

### on_run_end

Called when a run completes (success, failure, or cancellation).

```python
guard.on_run_end(run_id="run-123", status="completed")
```

**Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `run_id` | `str` | Yes | The run identifier |
| `status` | `str` | Yes | Final status (e.g., "completed", "failed", "cancelled") |

**Returns:** `None`

Always call `on_run_end`, even for failed runs:

```python
try:
    result = execute_agent_run()
    guard.on_run_end(run_id, status="completed")
except Exception as e:
    guard.on_run_end(run_id, status="failed")
    raise
```

## Iteration Lifecycle

### before_iteration

Called before each iteration of the agent loop.

```python
decision = guard.before_iteration(
    run_id="run-123",
    iteration_idx=0,
    context={"messages_count": 5},
)
```

**Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `run_id` | `str` | Yes | The run identifier |
| `iteration_idx` | `int` | Yes | Current iteration index (0-based) |
| `context` | `dict[str, Any]` | No | Optional context about the iteration |

**Returns:** `IterationDecision`

```python
@dataclass
class IterationDecision:
    allowed: bool                        # Whether to proceed
    reason: Optional[str]                # Halt reason if not allowed
    action: DecisionAction               # ALLOW or HALT
    action_overrides: ActionOverrides    # Runtime behavior modifications
    remaining_iterations: Optional[int]  # Remaining allowed iterations
    remaining_budget: Optional[float]    # Remaining budget
    warnings: list[str]                  # Budget warnings
```

**Example:**

```python
for iteration in range(max_iterations):
    decision = guard.before_iteration(run_id=run_id, iteration_idx=iteration)

    if not decision.allowed:
        logger.info(f"Run halted: {decision.reason}")
        break

    if decision.action_overrides.force_terminate_run:
        logger.warning("Forced termination requested")
        break

    # ... execute iteration ...
```

### after_iteration

Called after each iteration completes.

```python
from strands_costguard import IterationUsage

usage = IterationUsage(iteration_idx=0)
# Add model and tool usage to iteration
usage.add_model_usage(model_usage)
usage.add_tool_usage(tool_usage)

guard.after_iteration(
    run_id="run-123",
    iteration_idx=0,
    usage=usage,
)
```

**Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `run_id` | `str` | Yes | The run identifier |
| `iteration_idx` | `int` | Yes | Completed iteration index |
| `usage` | `IterationUsage` | Yes | Usage metrics for the iteration |

**Returns:** `None`

## Model Call Lifecycle

### before_model_call

Called before each model API call.

```python
decision = guard.before_model_call(
    run_id="run-123",
    model_name="gpt-4o",
    stage="synthesis",
    prompt_tokens_estimate=500,
)
```

**Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `run_id` | `str` | Yes | The run identifier |
| `model_name` | `str` | Yes | Requested model name |
| `stage` | `str` | No | Semantic stage (default: "other") |
| `prompt_tokens_estimate` | `int` | No | Estimated input tokens (default: 0) |

**Stage Values:**
- `"planning"` - Task decomposition and strategy
- `"tool_selection"` - Choosing tools/functions
- `"synthesis"` - Generating final output
- `"other"` - Any other stage

**Returns:** `ModelDecision`

```python
@dataclass
class ModelDecision:
    allowed: bool                        # Whether the call can proceed
    reason: Optional[str]                # Rejection/downgrade reason
    action: DecisionAction               # ALLOW, DOWNGRADE, or REJECT
    action_overrides: ActionOverrides    # Runtime modifications
    effective_model: Optional[str]       # Model to actually use
    max_tokens: Optional[int]            # Token limit to apply
    remaining_tokens: Optional[int]      # Remaining token budget
    remaining_budget: Optional[float]    # Remaining cost budget
    was_downgraded: bool                 # Whether model was downgraded
    warnings: list[str]                  # Cost warnings
```

**Example:**

```python
decision = guard.before_model_call(
    run_id=run_id,
    model_name="gpt-4o",
    stage="synthesis",
    prompt_tokens_estimate=estimate_tokens(messages),
)

if not decision.allowed:
    raise RuntimeError(f"Model call rejected: {decision.reason}")

# Use the effective model (may be downgraded)
response = model_client.call(
    model=decision.effective_model,
    messages=messages,
    max_tokens=decision.max_tokens,  # Apply token limit if set
)

if decision.was_downgraded:
    logger.info(f"Model downgraded: {decision.warnings}")
```

### after_model_call

Called after each model API call completes.

```python
from strands_costguard import ModelUsage

usage = ModelUsage.from_response(
    model_name="gpt-4o-mini",
    prompt_tokens=500,
    completion_tokens=200,
    latency_ms=450.0,
    cached_tokens=100,
)

guard.after_model_call(run_id="run-123", usage=usage)
```

**Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `run_id` | `str` | Yes | The run identifier |
| `usage` | `ModelUsage` | Yes | Usage metrics from the call |

**ModelUsage Fields:**

| Field | Type | Description |
|-------|------|-------------|
| `model_name` | `str` | Model that was used |
| `prompt_tokens` | `int` | Input tokens consumed |
| `completion_tokens` | `int` | Output tokens generated |
| `total_tokens` | `int` | Total tokens (computed) |
| `cost` | `float` | Cost (computed if not provided) |
| `latency_ms` | `float` | Call latency in milliseconds |
| `cached_tokens` | `int` | Tokens served from cache |
| `reasoning_tokens` | `int` | Reasoning tokens (o1 models) |

**Returns:** `None`

## Tool Call Lifecycle

### before_tool_call

Called before each tool invocation.

```python
decision = guard.before_tool_call(
    run_id="run-123",
    tool_name="web_search",
)
```

**Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `run_id` | `str` | Yes | The run identifier |
| `tool_name` | `str` | Yes | Name of the tool |

**Returns:** `ToolDecision`

```python
@dataclass
class ToolDecision:
    allowed: bool                        # Whether the call can proceed
    reason: Optional[str]                # Rejection reason
    action: DecisionAction               # ALLOW or REJECT
    action_overrides: ActionOverrides    # Runtime modifications
    remaining_tool_calls: Optional[int]  # Remaining allowed calls
    remaining_budget: Optional[float]    # Remaining cost budget
    warnings: list[str]                  # Warnings
```

**Example:**

```python
decision = guard.before_tool_call(run_id=run_id, tool_name="web_search")

if not decision.allowed:
    logger.warning(f"Tool call rejected: {decision.reason}")
    # Handle rejection - skip tool or use fallback
else:
    result = execute_tool("web_search", args)
    # ...
```

### after_tool_call

Called after each tool invocation completes.

```python
from strands_costguard import ToolUsage

usage = ToolUsage(
    tool_name="web_search",
    latency_ms=150.0,
    input_size_bytes=100,
    output_size_bytes=5000,
    success=True,
)

guard.after_tool_call(
    run_id="run-123",
    tool_name="web_search",
    cost_metadata=usage,
)
```

**Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `run_id` | `str` | Yes | The run identifier |
| `tool_name` | `str` | Yes | Name of the tool |
| `cost_metadata` | `ToolUsage` | Yes | Usage metrics from the call |

**ToolUsage Fields:**

| Field | Type | Description |
|-------|------|-------------|
| `tool_name` | `str` | Tool that was called |
| `cost` | `float` | Cost (computed if not provided) |
| `latency_ms` | `float` | Call latency in milliseconds |
| `input_size_bytes` | `int` | Input data size |
| `output_size_bytes` | `int` | Output data size |
| `success` | `bool` | Whether the call succeeded |
| `error_type` | `str` | Error type if failed |
| `metadata` | `dict` | Additional metadata |

**Returns:** `None`

## Complete Integration Example

```python
from strands_costguard import (
    CostGuard, CostGuardConfig, FilePolicySource,
    ModelUsage, ToolUsage, IterationUsage,
)
import uuid

# Initialize
guard = CostGuard(config=CostGuardConfig(
    policy_source=FilePolicySource(path="./policies"),
))

def run_agent(tenant_id: str, strand_id: str, task: str) -> dict:
    run_id = str(uuid.uuid4())

    # === RUN START ===
    admission = guard.on_run_start(
        tenant_id=tenant_id,
        strand_id=strand_id,
        workflow_id="task_execution",
        run_id=run_id,
    )

    if not admission.allowed:
        return {"status": "rejected", "reason": admission.reason}

    try:
        result = {"status": "running", "iterations": 0}

        for iteration in range(10):
            # === BEFORE ITERATION ===
            iter_decision = guard.before_iteration(
                run_id=run_id,
                iteration_idx=iteration,
            )

            if not iter_decision.allowed:
                result["status"] = "halted"
                result["reason"] = iter_decision.reason
                break

            iteration_usage = IterationUsage(iteration_idx=iteration)

            # === BEFORE MODEL CALL ===
            model_decision = guard.before_model_call(
                run_id=run_id,
                model_name="gpt-4o",
                stage="planning" if iteration == 0 else "synthesis",
            )

            if model_decision.allowed:
                # Make actual model call
                response = call_model(
                    model=model_decision.effective_model,
                    max_tokens=model_decision.max_tokens,
                )

                # === AFTER MODEL CALL ===
                model_usage = ModelUsage.from_response(
                    model_name=model_decision.effective_model,
                    prompt_tokens=response["usage"]["prompt_tokens"],
                    completion_tokens=response["usage"]["completion_tokens"],
                )
                guard.after_model_call(run_id=run_id, usage=model_usage)
                iteration_usage.add_model_usage(model_usage)

            # Process tool calls if any
            for tool_call in extract_tool_calls(response):
                # === BEFORE TOOL CALL ===
                tool_decision = guard.before_tool_call(
                    run_id=run_id,
                    tool_name=tool_call["name"],
                )

                if tool_decision.allowed:
                    tool_result = execute_tool(tool_call)

                    # === AFTER TOOL CALL ===
                    tool_usage = ToolUsage(
                        tool_name=tool_call["name"],
                        input_size_bytes=len(str(tool_call["args"])),
                        output_size_bytes=len(str(tool_result)),
                    )
                    guard.after_tool_call(
                        run_id=run_id,
                        tool_name=tool_call["name"],
                        cost_metadata=tool_usage,
                    )
                    iteration_usage.add_tool_usage(tool_usage)

            # === AFTER ITERATION ===
            guard.after_iteration(
                run_id=run_id,
                iteration_idx=iteration,
                usage=iteration_usage,
            )

            result["iterations"] = iteration + 1

            if is_task_complete(response):
                result["status"] = "completed"
                break

        return result

    finally:
        # === RUN END ===
        guard.on_run_end(run_id=run_id, status=result.get("status", "unknown"))
```

## Best Practices

1. **Always call `on_run_end`** - Use try/finally to ensure cleanup
2. **Use appropriate stages** - Help routing make better decisions
3. **Estimate tokens when possible** - Enables pre-call cost checks
4. **Check all decisions** - Handle rejections gracefully
5. **Record accurate usage** - Ensures correct cost tracking
6. **Log warnings** - Budget warnings indicate approaching limits
