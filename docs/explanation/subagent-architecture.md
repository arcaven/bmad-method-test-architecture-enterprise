---
title: Subagent Architecture
description: How TEA uses subagents and agent teams across workflows
---

# Subagents and Agent Teams in TEA

This guide explains how TEA orchestrates work when a workflow can split into
independent worker steps.

## Scope

This applies to these workflows:

- `automate`
- `atdd`
- `test-review`
- `nfr-assess`
- `framework`
- `ci`
- `test-design`
- `trace`

It does not apply to `teach-me-testing`.

---

## Core Model

TEA orchestration has three parts:

1. Resolve execution mode (`tea_execution_mode` + optional runtime probe)
2. Dispatch independent worker steps (when the workflow has them)
3. Aggregate worker outputs into one deterministic final artifact

Workers are isolated and exchange data through structured outputs that the
aggregation step validates.

---

## Execution Modes

TEA supports four modes:

- `auto`
- `agent-team`
- `subagent`
- `sequential`

### What Each Mode Means

- `auto`: Choose the best supported mode at runtime.
- `agent-team`: Prefer team/delegation orchestration when runtime supports it.
- `subagent`: Prefer isolated worker orchestration when runtime supports it.
- `sequential`: Run worker steps one-by-one.

### Fallback Behavior

When `tea_capability_probe: true`, TEA can fallback safely:

- `auto` falls back in order: `agent-team` -> `subagent` -> `sequential`
- explicit `agent-team` or `subagent` falls back to next supported mode
- `sequential` always stays sequential

When `tea_capability_probe: false`, TEA honors the requested mode strictly and
fails if runtime cannot execute it.

### Runtime Scheduling

In `agent-team` and `subagent` modes, runtime decides concurrency and timing.
TEA does not impose its own parallel worker limit.

---

## Verbal Override Rules

During a run, explicit user phrasing can override config for that run only.

Supported normalized terms:

- `agent team` or `agent teams` -> `agent-team`
- `subagent` or `subagents` -> `subagent`
- `sequential` -> `sequential`
- `auto` -> `auto`

Resolution precedence:

1. Explicit run-level request (if present)
2. `tea_execution_mode` in config
3. Runtime fallback (when probing is enabled)

---

## Workflow Coverage Map

### `automate`

- Worker split: API + E2E/backend test generation workers
- Aggregation: merges generated tests, fixtures, and summary stats
- Mode effect: changes orchestration style only, not output contract

### `atdd`

- Worker split: failing API + failing E2E test generation workers
- Aggregation: validates red-phase output and merges artifacts
- Mode effect: changes orchestration style only, not red-phase requirements

### `test-review`

- Worker split: quality-dimension evaluations (determinism, isolation,
  maintainability, performance)
- Aggregation: computes combined quality score/report
- Mode effect: changes orchestration style only, not scoring schema

### `nfr-assess`

- Worker split: security, performance, reliability, scalability assessments
- Aggregation: computes overall risk, compliance summary, priority actions
- Mode effect: changes orchestration style only, not report schema

### `framework`

- Worker split: scaffold work units (structure/config, fixtures, samples)
- Aggregation: consolidates generated framework setup outputs
- Mode effect: changes orchestration style only

### `ci`

- Worker split: orchestration-capable mode resolution for pipeline generation
- Aggregation: deterministic single pipeline artifact
- Mode effect: mostly impacts orchestration policy; final pipeline contract is
  unchanged

### `test-design`

- Worker split: orchestration-capable mode resolution for output generation
- Aggregation: deterministic design artifact output
- Mode effect: orchestration policy only; output schema unchanged

### `trace`

- Worker split: phase/work-unit separation with dependency ordering
- Aggregation: merges gap analysis + coverage/gate data
- Mode effect: orchestration policy only; final decision/report contract
  unchanged

---

## Design Guarantees

TEA maintains these guarantees across all modes:

- Same output schema for a given workflow
- Same validation and aggregation rules
- Same deterministic fallback semantics
- Same failure behavior for missing/invalid worker outputs

Mode selection changes orchestration behavior, not artifact contracts.

---

## Practical Guidance

Recommended defaults:

```yaml
tea_execution_mode: 'auto'
tea_capability_probe: true
```

Use `sequential` when you need strict single-threaded execution or debugging
clarity.

Use explicit `agent-team` or `subagent` only when you intentionally want that
mode and understand runtime support in your environment.

---

## Troubleshooting Signals

Common causes of orchestration confusion:

- Explicit run-level override text was provided and took precedence over config
- Runtime did not support requested mode and fallback changed final mode
- Probe disabled (`tea_capability_probe: false`) with unsupported explicit mode

Check resolved mode logs in the workflow execution report to confirm what mode
actually ran.
