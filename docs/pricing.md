# Pricing Configuration

Cost Guard calculates costs using a configurable pricing table for models and tools.

## Pricing Table Specification

Pricing is defined in `pricing.yaml`:

```yaml
pricing:
  currency: "USD"
  fallback_input_per_1k: 1.0
  fallback_output_per_1k: 3.0

  models:
    "gpt-4o":
      input_per_1k: 2.50
      output_per_1k: 10.00
      cached_input_per_1k: 1.25

    "gpt-4o-mini":
      input_per_1k: 0.15
      output_per_1k: 0.60

  tools:
    "web_search":
      cost_per_call: 0.01

    "file_upload":
      cost_per_call: 0.00
      cost_per_input_byte: 0.000001
```

## Fields Reference

### Global Settings

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `currency` | `string` | `"USD"` | Currency for all prices |
| `fallback_input_per_1k` | `float` | `1.0` | Input price for unknown models |
| `fallback_output_per_1k` | `float` | `3.0` | Output price for unknown models |

### Model Pricing

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `input_per_1k` | `float` | Yes | Cost per 1,000 input tokens |
| `output_per_1k` | `float` | Yes | Cost per 1,000 output tokens |
| `cached_input_per_1k` | `float` | No | Cost per 1,000 cached input tokens |
| `reasoning_per_1k` | `float` | No | Cost per 1,000 reasoning tokens (o1 models) |
| `currency` | `string` | No | Override global currency |

### Tool Pricing

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `cost_per_call` | `float` | `0.0` | Fixed cost per tool invocation |
| `cost_per_input_byte` | `float` | `0.0` | Cost per byte of input data |
| `cost_per_output_byte` | `float` | `0.0` | Cost per byte of output data |
| `currency` | `string` | Global | Override global currency |

## Cost Calculation

### Model Costs

Total cost for a model call:

```
cost = (standard_input_tokens / 1000) × input_per_1k
     + (cached_tokens / 1000) × cached_input_per_1k
     + (completion_tokens / 1000) × output_per_1k
     + (reasoning_tokens / 1000) × reasoning_per_1k
```

Where:
- `standard_input_tokens = prompt_tokens - cached_tokens`
- Cached and reasoning token costs only apply if pricing is configured

### Tool Costs

Total cost for a tool call:

```
cost = cost_per_call
     + (input_size_bytes × cost_per_input_byte)
     + (output_size_bytes × cost_per_output_byte)
```

## Default Model Pricing

Cost Guard includes built-in default pricing for common models:

| Model | Input/1k | Output/1k |
|-------|----------|-----------|
| gpt-4 | $30.00 | $60.00 |
| gpt-4-turbo | $10.00 | $30.00 |
| gpt-4o | $2.50 | $10.00 |
| gpt-4o-mini | $0.15 | $0.60 |
| gpt-4.1 | $5.00 | $15.00 |
| gpt-4.1-mini | $0.40 | $1.60 |
| gpt-3.5-turbo | $0.50 | $1.50 |
| claude-3-opus | $15.00 | $75.00 |
| claude-3-sonnet | $3.00 | $15.00 |
| claude-3-haiku | $0.25 | $1.25 |
| claude-3.5-sonnet | $3.00 | $15.00 |
| claude-3.5-haiku | $0.80 | $4.00 |
| gemini-1.5-pro | $3.50 | $10.50 |
| gemini-1.5-flash | $0.075 | $0.30 |
| gemini-2.0-flash | $0.10 | $0.40 |
| llama-3.1-405b | $5.00 | $15.00 |
| llama-3.1-70b | $0.90 | $0.90 |
| llama-3.1-8b | $0.20 | $0.20 |

These defaults are used when no pricing.yaml is provided or when a model isn't found in your configuration.

## Model Name Matching

Cost Guard uses flexible model name matching:

1. **Exact match** - `gpt-4o` matches `gpt-4o`
2. **Prefix match** - `gpt-4o-2024-08-06` matches `gpt-4o` pricing
3. **Fallback** - Unknown models use `fallback_input_per_1k` and `fallback_output_per_1k`

This allows version-specific model names to use base model pricing.

## Examples

### Standard Configuration

```yaml
pricing:
  currency: "USD"
  fallback_input_per_1k: 1.0
  fallback_output_per_1k: 3.0

  models:
    "gpt-4o":
      input_per_1k: 2.50
      output_per_1k: 10.00

    "gpt-4o-mini":
      input_per_1k: 0.15
      output_per_1k: 0.60

    "claude-3.5-sonnet":
      input_per_1k: 3.00
      output_per_1k: 15.00
```

### With Cached Token Pricing

```yaml
pricing:
  models:
    "gpt-4o":
      input_per_1k: 2.50
      output_per_1k: 10.00
      cached_input_per_1k: 1.25  # 50% discount for cached tokens

    "claude-3.5-sonnet":
      input_per_1k: 3.00
      output_per_1k: 15.00
      cached_input_per_1k: 0.30  # 90% discount for cached tokens
```

### With Reasoning Token Pricing (o1 Models)

```yaml
pricing:
  models:
    "o1-preview":
      input_per_1k: 15.00
      output_per_1k: 60.00
      reasoning_per_1k: 60.00  # Reasoning tokens billed separately

    "o1-mini":
      input_per_1k: 3.00
      output_per_1k: 12.00
      reasoning_per_1k: 12.00
```

### Tool Pricing

```yaml
pricing:
  tools:
    "web_search":
      cost_per_call: 0.01  # $0.01 per search

    "code_execution":
      cost_per_call: 0.005  # $0.005 per execution

    "file_upload":
      cost_per_call: 0.00
      cost_per_input_byte: 0.000001  # $1 per MB uploaded

    "image_generation":
      cost_per_call: 0.04  # $0.04 per image
```

## Programmatic Access

### Get Model Pricing

```python
from strands_costguard import PricingTable

# Load from dict
pricing = PricingTable.from_dict({
    "currency": "USD",
    "models": {
        "gpt-4o": {"input_per_1k": 2.50, "output_per_1k": 10.00}
    }
})

# Get pricing for a model
model_pricing = pricing.get_model_pricing("gpt-4o")
print(f"Input: ${model_pricing.input_per_1k}/1k tokens")
print(f"Output: ${model_pricing.output_per_1k}/1k tokens")
```

### Calculate Costs

```python
# Calculate model cost
cost = pricing.calculate_model_cost(
    model_name="gpt-4o",
    prompt_tokens=1000,
    completion_tokens=500,
    cached_tokens=200,
)
print(f"Model cost: ${cost:.4f}")

# Estimate cost before call
estimated = pricing.estimate_model_cost(
    model_name="gpt-4o",
    estimated_input_tokens=1000,
    estimated_output_tokens=500,
)
print(f"Estimated cost: ${estimated:.4f}")

# Calculate tool cost
tool_cost = pricing.calculate_tool_cost(
    tool_name="file_upload",
    input_size_bytes=1_000_000,  # 1 MB
)
print(f"Tool cost: ${tool_cost:.4f}")
```

## Updating Pricing

To update pricing at runtime:

```python
# Force policy refresh (includes pricing)
guard._policy_store.refresh()
```

Or set a short refresh interval:

```python
config = CostGuardConfig(
    policy_source=FilePolicySource(path="./policies"),
    policy_refresh_interval_seconds=60,  # Refresh every minute
)
```

## Best Practices

1. **Keep pricing up to date** - Model pricing changes frequently
2. **Set fallback prices** - Handle unknown models gracefully
3. **Use cached token pricing** - Track actual costs more accurately
4. **Configure tool costs** - Don't forget API calls that have costs
5. **Match provider pricing** - Verify against your actual API invoices
6. **Version your pricing files** - Track pricing changes over time
