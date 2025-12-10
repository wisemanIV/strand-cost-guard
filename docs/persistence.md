# Budget Persistence

By default, Cost Guard stores budget state in memory. When the application restarts, all budget counters reset to zero. For production deployments, you can enable persistence using Valkey (Redis-compatible).

## Installation

Install with the valkey extra:

```bash
pip install strands-costguard[valkey]
```

Or add to your requirements:

```
strands-costguard[valkey]>=0.1.0
```

## Configuration

```python
import valkey
from strands_costguard import CostGuard, CostGuardConfig, FilePolicySource
from strands_costguard.persistence import ValkeyBudgetStore

# Create Valkey client
client = valkey.Valkey(
    host="localhost",
    port=6379,
    db=0,
    # password="your-password",  # If authentication required
)

# Create budget store
store = ValkeyBudgetStore(client)

# Pass to CostGuard config
config = CostGuardConfig(
    policy_source=FilePolicySource(path="./policies"),
    budget_store=store,
)

guard = CostGuard(config=config)
```

## How It Works

When a `budget_store` is configured:

1. **On startup**: Budget state is loaded from Valkey instead of starting at zero
2. **During runs**: Cost updates are persisted atomically to Valkey
3. **On period expiry**: State is automatically reset when the budget period ends
4. **Concurrent safety**: Uses optimistic locking for multi-instance deployments

### Key Structure

Budget state is stored with keys following this pattern:

```
strands_costguard:budget:{scope_key}
```

Where `scope_key` depends on the budget scope:
- Global: `global:{budget_id}`
- Tenant: `tenant:{tenant_id}:{budget_id}`
- Strand: `strand:{tenant_id}:{strand_id}:{budget_id}`
- Workflow: `workflow:{tenant_id}:{strand_id}:{workflow_id}:{budget_id}`

### Data Format

Each key stores a JSON object:

```json
{
  "budget_id": "monthly-limit",
  "scope_key": "tenant:prod-001:monthly-limit",
  "period_start": "2025-01-01T00:00:00",
  "period_end": "2025-02-01T00:00:00",
  "total_cost": 1234.56,
  "total_runs": 5000,
  "total_input_tokens": 10000000,
  "total_output_tokens": 2000000,
  "total_iterations": 15000,
  "total_tool_calls": 8000,
  "model_costs": {"gpt-4o": 1000.0, "gpt-4o-mini": 234.56},
  "tool_costs": {"web_search": 50.0},
  "concurrent_run_ids": ["run-123", "run-456"]
}
```

### Automatic Expiry

Keys are set to expire at `period_end`, so stale budget data is automatically cleaned up by Valkey.

## Cloud Deployments

### Amazon ElastiCache

```python
client = valkey.Valkey(
    host="your-cluster.xxxxx.cache.amazonaws.com",
    port=6379,
    ssl=True,
)
store = ValkeyBudgetStore(client)
```

### Amazon ElastiCache Serverless

```python
client = valkey.Valkey(
    host="your-serverless-endpoint.amazonaws.com",
    port=6379,
    ssl=True,
)
store = ValkeyBudgetStore(client)
```

### Upstash

```python
client = valkey.Valkey(
    host="your-instance.upstash.io",
    port=6379,
    password="your-password",
    ssl=True,
)
store = ValkeyBudgetStore(client)
```

## Multi-Instance Deployments

The Valkey store supports concurrent updates from multiple application instances. Cost increments use optimistic locking (WATCH/MULTI/EXEC) to ensure atomic updates.

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│ Instance 1  │     │ Instance 2  │     │ Instance 3  │
│ CostGuard   │     │ CostGuard   │     │ CostGuard   │
└──────┬──────┘     └──────┬──────┘     └──────┬──────┘
       │                   │                   │
       └───────────────────┼───────────────────┘
                           │
                    ┌──────▼──────┐
                    │   Valkey    │
                    │   Cluster   │
                    └─────────────┘
```

All instances share the same budget state, ensuring consistent enforcement across your deployment.

## Fallback Behavior

If Valkey is unavailable:
- Cost Guard continues operating with in-memory state
- Budget enforcement still works within each instance
- State is not shared between instances
- A warning is logged

This fail-open behavior prevents Valkey issues from breaking your agent system.

## Custom Key Prefix

To use a different key prefix (e.g., for multiple environments sharing a Valkey instance):

```python
store = ValkeyBudgetStore(
    client=client,
    key_prefix="myapp:prod:budget:",  # Custom prefix
)
```

## Querying Budget State

You can query budget state directly from Valkey for monitoring:

```python
# List all budgets for a tenant
keys = store.list_budgets("tenant:prod-001:*")

# Get specific budget state
state = store.get("tenant:prod-001:monthly-limit")
if state:
    print(f"Total cost: ${state.total_cost:.2f}")
    print(f"Total runs: {state.total_runs}")
```

## Migration from In-Memory

When adding persistence to an existing deployment:

1. Install the valkey extra
2. Configure the ValkeyBudgetStore
3. Budget counters start fresh in Valkey
4. Historical in-memory state is not migrated

If you need to preserve existing state, you can manually seed Valkey before switching:

```python
# One-time migration script
store = ValkeyBudgetStore(client)

from strands_costguard.persistence.valkey_store import BudgetStateData
from datetime import datetime

# Seed with existing data
state = BudgetStateData(
    budget_id="monthly-limit",
    scope_key="tenant:prod-001:monthly-limit",
    period_start="2025-01-01T00:00:00",
    period_end="2025-02-01T00:00:00",
    total_cost=5000.0,  # Your existing total
    total_runs=10000,
    # ... other fields
)
store.set("tenant:prod-001:monthly-limit", state)
```
