<!-- Last regenerated: 2026-04-13T12:00Z by codebase-mirror scan -->

# AOCL (Agent Orchestration Control Layers) — Codebase Map

Control-layer protocol for AI agent orchestration. Standardizes how an orchestrator processes incoming events through ordered layers (routing, policy gating, context retrieval, delegation, verification). Produces observability as first-class output via AEE envelopes.

| Field | Value |
|-------|-------|
| Version | 0.1 (Experimental) |
| Status | IETF Internet-Draft |
| IETF Draft | [`draft-cowles-aocl-00`](https://datatracker.ietf.org/doc/draft-cowles-aocl/) |
| License | MIT |

## Related Protocols (Quox Protocol Family)

| Protocol | Role | Repo |
|----------|------|------|
| **AEE** | Envelope format + causality | [github.com/quoxai/aee](https://github.com/quoxai/aee) |
| **AOCL** | Orchestration control layers | *(this repo)* |
| **VOLT** | Verifiable evidence ledger | [github.com/quoxai/volt](https://github.com/quoxai/volt) |
| **WARD** | Content-free hash-chain witnessing | [github.com/quoxai/ward](https://github.com/quoxai/ward) |

## Directory Structure

```
aocl/
├── README.md               # Protocol overview, mental model, quick example
├── ROADMAP.md              # Planned extensions (v0.2–v0.6)
├── LICENSE                 # MIT
├── docs/
│   ├── spec.md             # Core spec: terminology, design principles, context bundle
│   ├── stacks.md           # Layer taxonomy, stack definition format (pipeline/DAG)
│   ├── observability.md    # Trace events, NDJSON logging, UI views
│   ├── aee-binding.md      # How AOCL emits AEE envelopes (intents, payloads)
│   └── examples.md         # End-to-end trace + stack variants
└── examples/
    ├── stacks/
    │   ├── realtime-alert.json       # Speed-first pipeline (6 layers)
    │   └── restricted-textonly.json  # Safety-first pipeline (no tool exec)
    └── traces/
        └── backup-check.ndjson       # 14-event end-to-end AEE trace
```

## Canonical Layer Taxonomy (L0–L10)

| Layer | ID | Purpose |
|-------|----|---------|
| L0 | `L0.ingress.normalize` | Normalize incoming events, create run IDs |
| L1 | `L1.identity.scope` | Apply identity, permissions, redaction rules |
| L2 | `L2.route.smart` | Deterministic fast-path router (pattern match, cache) |
| L3 | `L3.policy.gate` | Safety/compliance checks, tool restrictions, HITL |
| L4 | `L4.plan.decompose` | Convert intent to structured objectives |
| L5 | `L5.context.retrieve` | Retrieve memory/files/RAG context |
| L6 | `L6.shape.rewrite` | Rewrite into operational form (AEE tasks, tool plans) |
| L7 | `L7.delegate.execute` | Delegate to agents/tools |
| L8 | `L8.verify.check` | Verification/evals/consistency checks |
| L9 | `L9.assemble.respond` | Assemble final response, apply formatting |
| L10 | `L10.audit.writeback` | Persist trace summaries, memory writeback |

## Context Bundle Partitions (Recommended)

| Partition | Purpose |
|-----------|---------|
| C0 Event | Source, timestamp, channel, correlation IDs |
| C1 Identity & Scope | User/org, roles, permissions, redaction rules |
| C2 Task | Goal, constraints, definition of done |
| C3 Memory/Knowledge | RAG hits, file refs, summaries |
| C4 Policy | Safety rules, allowed tools, model restrictions |
| C5 Execution | Budgets, timeouts, concurrency, run_id |
| C6 Audit | Log level, required evidence, checkpoints |

## Stack Modes

| Mode | Description |
|------|-------------|
| `pipeline` | Ordered list of layers, executed sequentially |
| `dag` | Nodes + edges with `when` conditions for branching |

## AOCL Intent Namespace (`aocl.*`)

| Intent | Type | Purpose |
|--------|------|---------|
| `aocl.stack.select` | event | Which stack/branch was chosen |
| `aocl.layer.enter` | event | Layer started |
| `aocl.layer.decision` | event | Layer decision (route/policy/plan) |
| `aocl.layer.exit` | event | Layer completed |
| `aocl.context.patch` | event | Context delta applied |
| `aocl.control.bypass` | event | Bypass requested/allowed/denied |
| `aocl.control.branch` | event | Branch taken/rejected |
| `aocl.verify.result` | event | Verification summary |
| `aocl.run.summary` | event | End-of-run summary |

## Example Stacks

### `realtime-alert` (Speed-First)
- **Layers:** L0 → L1 → L3 → L7 → L9 → L10
- **Skips:** L2 (router), L4 (plan), L5 (context), L6 (shape), L8 (verify)
- **Timeout:** 15s, 16 parallel actions

### `restricted-textonly` (Safety-First)
- **Layers:** L0 → L1 → L3 → L6 → L9 → L10
- **Disables:** Tool execution, network access
- **Timeout:** 30s, 0 parallel actions

## Key Concepts

| Concept | Description |
|---------|-------------|
| **Layer Contract** | Each layer produces: decision summary, context delta, optional actions, control flags |
| **Delta-First** | Layers emit context patches + refs/digests, not full context blobs |
| **Bypass Audit** | All bypasses MUST emit `aocl.control.bypass` events |
| **AEE Binding** | AOCL uses AEE envelopes (`event`, `task`, `result`) without changing AEE spec |
| **Correlation** | All envelopes share `corr` from originating request |

## Roadmap Summary

| Version | Focus |
|---------|-------|
| v0.1 | Minimal viable protocol (current) |
| v0.2 | JSON schemas + validators |
| v0.3 | Transport bindings (HTTP, WS, NATS, Kafka) |
| v0.4 | Control-plane hardening (bypass policy, HITL) |
| v0.5 | OpenTelemetry mapping, replay tooling, reference UI |
| v0.6 | Reference implementations (Python, TypeScript) |

## Metrics

| Metric | Count |
|--------|-------|
| Spec Documents | 5 |
| Canonical Layers | 11 (L0–L10) |
| Context Partitions | 7 (C0–C6) |
| Example Stacks | 2 |
| Example Traces | 1 (14 events) |
| AOCL Intents | 9 |

## Invariants

| Check | Status | Details |
|-------|--------|---------|
| spec-complete | ✓ pass | 11 layers documented in `docs/stacks.md` |
| aee-binding | ✓ pass | 9 intents defined in `docs/aee-binding.md` |
| examples-valid | ✓ pass | 2 stacks + 1 trace in `examples/` |
| roadmap-defined | ✓ pass | v0.1–v0.6 planned in `ROADMAP.md` |
