# Domain: Configuration

## Domain Name

Configuration

## What This Domain Governs

How the system manages, validates, and enforces its runtime configuration. This domain assesses whether configuration is externalized from code, validated before the system accepts work, and organized under a single coherent mechanism. It also covers whether the startup sequence acts as a safety gate that prevents the system from running in an invalid state.

## Why It Matters

Agentic systems have a high density of configurable parameters: model names, endpoint URLs, timeout values, retry limits, execution caps, connection strings, TTLs, and feature flags. When these values are scattered through runtime code as hardcoded literals, changing behavior requires code changes, review, and redeployment — even for values that are inherently environmental.

Configuration failures are insidious. A hardcoded timeout that worked in development silently causes production failures under load. A missing environment variable causes a crash thirty minutes into an expensive LLM execution. A duplicated configuration value across two files drifts silently until one source overrides the other in an order nobody remembers.

A rigorous configuration domain catches these problems before they reach production — at write time through externalization, at startup through validation, and at design time through a single-source-of-truth discipline.

## Principles in This Domain

| ID | Title | Severity |
|----|-------|----------|
| P2.1 | Externalized Configuration | HIGH |
| P2.2 | Configuration Validation at Startup | HIGH |
| P2.3 | Startup as a Safety Gate | HIGH |
| P2.4 | Configuration as Single Source of Truth | MEDIUM |

## Common Failure Modes

- **Hardcoded runtime values** — timeout integers, model name strings, endpoint URLs, or retry counts embedded directly in source files rather than read from configuration
- **Config model without validation** — a configuration schema or dataclass exists but is never validated at startup; the system accepts malformed values silently
- **Silent startup with missing dependencies** — the system starts successfully despite required external services being unreachable, then fails mid-execution
- **Duplicated configuration sources** — the same value appears in an environment variable, a config file, and a code constant, with no defined precedence
- **Partial externalization** — some values are externalized while others of the same category remain hardcoded, creating a false sense of completeness
- **Graceful degradation for required dependencies** — the system treats all dependencies as optional and silently degrades rather than failing fast for critical ones

## Typical Repository Indicators

Look for these when forming the system hypothesis:

- Configuration model definitions (settings classes, config schemas, typed config objects)
- Configuration file formats (YAML, TOML, JSON, .env files)
- Environment variable reading (os.environ, process.env, env parsing libraries)
- Startup or initialization modules that load and validate configuration
- Health check or readiness probe implementations
- Constants files or modules (may contain values that should be configuration)

## Common Trace Types to Inspect

- **Externalization trace** — search for hardcoded string literals, integer constants, and URL patterns in non-config source files to determine if configuration is truly externalized
- **Validation trace** — follow the configuration model from definition through loading to the point where the system begins accepting work; check whether validation occurs before that point
- **Startup gate trace** — follow the startup sequence to determine whether missing or invalid dependencies cause a halt or are silently absorbed
- **Source-of-truth trace** — identify all configuration sources and check for duplicated keys, conflicting defaults, or absent precedence rules

## Relationship to Adjacent Domains

- **Execution Boundaries (Domain 1):** Execution limits (max steps, timeouts, concurrency caps) are configuration values. P2.1 assesses whether they are externalized; P1.x principles assess whether they are enforced at runtime.
- **Scalability and Extensibility (Domain 8):** P8.4 (Configuration-Driven Behavior) depends on the configuration infrastructure assessed here. If configuration is not externalized (P2.1) or lacks a single source of truth (P2.4), configuration-driven behavior is not achievable.
- **Integration and Tool Model (Domain 4):** Tool endpoints, API keys, and connection parameters are configuration values that must be externalized (P2.1) and validated (P2.2).
- **Error Handling (Domain 6):** Configuration validation errors at startup (P2.2, P2.3) must produce structured, actionable diagnostics — the quality of those diagnostics is assessed by P6.x principles.

## Common False Positives in Assessment

- **Presence of a config file does not mean configuration is externalized** — check that the values in the config file are actually read by runtime code, not duplicated as constants elsewhere
- **A settings class does not mean validation occurs** — a typed configuration model may exist but never be instantiated or validated before the system starts work
- **Environment variable reads do not mean externalization** — if fallback defaults are hardcoded inline (e.g., `os.getenv("TIMEOUT", 30)`), the effective configuration may still be a hardcoded value in most deployments
- **Startup logging does not mean startup gating** — the system may log configuration values at startup without actually validating them or refusing to start on failure
- **A single config file does not mean single source of truth** — if code also contains constants that overlap with config file keys, there are effectively two sources
