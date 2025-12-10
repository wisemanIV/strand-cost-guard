# Overview

Strand Cost Guard is a cost management library for multi-agent systems built on the Strands framework. It provides budget enforcement, adaptive model routing, and OpenTelemetry-compatible metrics emission.

## Core Concepts

### Multi-Tenant Cost Attribution

Cost Guard tracks costs at multiple hierarchical levels:

```
Global
  └── Tenant (organization)
        └── Strand (agent definition)
              └── Workflow (specific task flow)
                    └── Run (single execution)
```

Each level can have its own budget limits and policies. More specific scopes take precedence over broader ones.

### Budget Enforcement

Budgets define cost limits for a given scope and time period. When limits are approached or exceeded, Cost Guard can:

- **Log warnings** - Alert operators without affecting execution
- **Downgrade models** - Switch to cheaper model alternatives
- **Limit capabilities** - Reduce token limits or iteration counts
- **Halt runs** - Stop current executions
- **Reject new runs** - Prevent new executions from starting

### Adaptive Model Routing

Cost Guard can automatically select models based on:

- Current budget utilization
- Remaining budget amount
- Iteration count within a run
- Semantic stage of the agent loop (planning, tool selection, synthesis)

This allows expensive models to be used when budget is healthy, with automatic fallback to cheaper alternatives as spending increases.

### Lifecycle Integration

Cost Guard integrates with agent runtimes through lifecycle hooks:

```
Run Start ─────────────────────────────────────────────► Run End
    │                                                        │
    ▼                                                        │
┌─ Iteration 0 ─┐   ┌─ Iteration 1 ─┐   ┌─ Iteration N ─┐   │
│               │   │               │   │               │   │
│  Model Call   │   │  Model Call   │   │  Model Call   │   │
│  Tool Call    │   │  Tool Call    │   │  Tool Call    │   │
│  Tool Call    │   │               │   │  Tool Call    │   │
└───────────────┘   └───────────────┘   └───────────────┘   │
                                                             ▼
```

At each transition point, Cost Guard evaluates policies and returns decisions about whether to proceed and how.

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                      CostGuard                              │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐  │
│  │ PolicyStore │  │BudgetTracker│  │  MetricsEmitter     │  │
│  │             │  │             │  │  (uses global OTEL) │  │
│  └──────┬──────┘  └──────┬──────┘  └──────────┬──────────┘  │
│         │                │                    │             │
│  ┌──────▼──────┐  ┌──────▼──────┐  ┌──────────▼──────────┐  │
│  │ BudgetSpecs │  │ BudgetState │  │   StrandsTelemetry  │  │
│  │ RoutingPol. │  │  RunState   │  │   (MeterProvider)   │  │
│  │ PricingTable│  │ PeriodUsage │  └─────────────────────┘  │
│  └─────────────┘  └─────────────┘                           │
└─────────────────────────────────────────────────────────────┘
```

### Components

| Component | Description |
|-----------|-------------|
| **CostGuard** | Main entry point providing lifecycle hooks |
| **PolicyStore** | Loads and caches budget/routing policies from YAML or environment |
| **BudgetTracker** | Maintains in-memory budget state and run tracking |
| **PricingTable** | Calculates costs based on model/tool pricing |
| **MetricsEmitter** | Emits OpenTelemetry metrics using global MeterProvider |
| **ModelRouter** | Optional helper for simplified model call integration |

## Design Principles

### Fail-Open by Default

When policy loading fails or metrics export errors occur, Cost Guard defaults to allowing operations to proceed. This prevents cost management from becoming a single point of failure for your agent system.

### Thread-Safe Operations

All budget state modifications are protected by locks, making Cost Guard safe to use in multi-threaded environments with concurrent agent runs.

### Low Overhead

Cost tracking happens in-memory with minimal latency impact. Metrics are batched and exported asynchronously through OpenTelemetry.

### Separation of Concerns

Cost Guard focuses solely on cost management. It integrates with but does not replace:
- **StrandsTelemetry** - For tracing and metrics infrastructure
- **Your agent runtime** - For actual execution logic
- **Your model clients** - For API communication

## When to Use Cost Guard

Cost Guard is ideal when you need to:

- **Control spending** across multiple tenants or agent types
- **Implement tiered pricing** with different limits per customer
- **Prevent runaway costs** from infinite loops or expensive operations
- **Optimize costs** by automatically using cheaper models when appropriate
- **Monitor usage** with detailed per-tenant, per-agent metrics
- **Enforce quotas** on iterations, tool calls, or tokens per run
