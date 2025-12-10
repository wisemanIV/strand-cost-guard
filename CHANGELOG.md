# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

## [0.1.0] - 2025-01-10

### Added
- Initial release of strands-costguard
- Budget enforcement at tenant, strand, workflow, and run levels
- Configurable budget periods: hourly, daily, weekly, monthly
- Soft thresholds with configurable actions (LOG_ONLY, DOWNGRADE_MODEL, LIMIT_CAPABILITIES, HALT_NEW_RUNS)
- Hard limit enforcement with HALT_RUN and REJECT_NEW_RUNS actions
- Adaptive model routing based on budget utilization, remaining budget, iteration count, and latency
- Stage-based routing configuration (planning, tool_selection, synthesis)
- Model pricing table with support for input/output/cached/reasoning tokens
- Tool pricing with per-call and per-byte cost options
- OpenTelemetry metrics emission via StrandsTelemetry integration
- YAML-based policy configuration (budgets.yaml, routing.yaml, pricing.yaml)
- Environment variable configuration for simple single-tenant setups
- Optional Valkey/Redis persistence for budget state (`strands-costguard[valkey]`)
- Comprehensive documentation

### Notes
- **0.x.y versions**: The API is not yet stable. Minor version bumps (0.x.0) may contain breaking changes. Patch versions (0.0.x) are backwards-compatible bug fixes.
- **1.0.0**: Will be released when the API is considered stable.

---

## Versioning Policy

This project follows [Semantic Versioning 2.0.0](https://semver.org/):

- **MAJOR** version (X.0.0): Incompatible API changes
- **MINOR** version (0.X.0): New functionality in a backwards-compatible manner
- **PATCH** version (0.0.X): Backwards-compatible bug fixes

### Pre-1.0 Versions

While the version is below 1.0.0:
- The API should be considered unstable
- Minor version changes (0.X.0) may include breaking changes
- Patch version changes (0.0.X) are backwards-compatible

### What Constitutes a Breaking Change

- Removing or renaming public classes, methods, or functions
- Changing method signatures (required parameters, return types)
- Changing configuration file schema in incompatible ways
- Changing metric names or attribute keys
- Changing Valkey key structure

### What Does NOT Constitute a Breaking Change

- Adding new optional parameters with defaults
- Adding new classes, methods, or functions
- Adding new configuration options with defaults
- Adding new metrics
- Bug fixes that correct behavior to match documentation
- Performance improvements
- Documentation updates
