<!-- Last verified: 2026-05-14T18:21Z by /codebase-mirror -->

# aocl — Codebase Map

**Agent Orchestration Control Layers.** A control-layer protocol for AI agent orchestration that produces observability as a first-class output. Part of the Quox protocol family (AEE, AOCL, VOLT, WARD).

## Version & Status
- **Spec:** 0.1 (Experimental)
- **IETF Draft:** `draft-cowles-aocl-00`
- **License:** MIT

## Repository Structure

```
aocl/
├── README.md                 # Protocol overview + mental model
├── ROADMAP.md                # Version roadmap (v0.1 → v0.6)
├── LICENSE                   # MIT
├── CODEBASE_MAP.md           # This file
├── docs/
│   ├── spec.md               # Core spec: layers, context bundle, design principles
│   ├── aee-binding.md        # How AOCL emits AEE envelopes (intent namespace, corr/reply_to)
│   ├── stacks.md             # Layer taxonomy (L0-L10), stack definition format (pipeline/DAG)
│   ├── observability.md      # Trace events, NDJSON logging, UI views
│   └── examples.md           # End-to-end trace + stack variants
└── examples/
    ├── stacks/
    │   ├── realtime-alert.json        # Speed-first pipeline (6 layers)
    │   └── restricted-textonly.json   # Safety-first, no tool execution
    └── traces/
        └── backup-check.ndjson        # Full AEE trace (14 envelopes)
```

## Core Concepts

### Layer Pipeline
AOCL processes events through ordered layers:
```
Ingress → Identity → Router → Policy → Plan → Context → Rewrite → Delegate → Verify → Assemble → Audit
```

### Canonical Layers (L0-L10)
| Layer | ID | Purpose |
|-------|-----|---------|
| L0 | `ingress.normalize` | Parse incoming events, create run IDs |
| L1 | `identity.scope` | Apply identity, permissions, redaction rules |
| L2 | `route.smart` | Deterministic fast-path (cached answers, known commands) |
| L3 | `policy.gate` | Safety/compliance checks, tool restrictions, HITL |
| L4 | `plan.decompose` | Convert intent to structured objectives |
| L5 | `context.retrieve` | Retrieve memory/RAG/files |
| L6 | `shape.rewrite` | Rewrite into operational form |
| L7 | `delegate.execute` | Delegate to agents/tools |
| L8 | `verify.check` | Verification, evals, evidence requirements |
| L9 | `assemble.respond` | Assemble final response |
| L10 | `audit.writeback` | Persist trace, memory writeback |

### Context Bundle Partitions
| Partition | Contents |
|-----------|----------|
| C0 Event | Source, timestamp, channel, correlation IDs |
| C1 Identity | User/org, roles, permissions, secret scope |
| C2 Task | Goal, constraints, definition of done |
| C3 Memory | RAG hits, file refs, summaries |
| C4 Policy | Safety rules, allowed tools, model restrictions |
| C5 Execution | Budgets, timeouts, concurrency, tool registry |
| C6 Audit | Log level, evidence requirements, compliance checkpoints |

### AEE Binding
AOCL emits audit via AEE envelopes using `aocl.*` intents:
- `aocl.stack.select` — stack chosen
- `aocl.layer.enter/exit/decision` — layer activity
- `aocl.context.patch` — context deltas
- `aocl.control.branch/bypass` — control flow changes
- `aocl.verify.result` — verification outcome
- `aocl.run.summary` — end-of-run summary

#### QuoxFlow Intent Vocabulary (R5.low)
Extended intents for QuoxFlow integration:

| Namespace | Intents | Purpose |
|-----------|---------|---------|
| `quox.budget.*` | `exhausted`, `warning` | Pre-execution budget checks |
| `quox.policy.*` | `rejected`, `gate_error` | Policy layer denials/errors |
| `agentic.skill.*` | `retrieved`, `drafted`, `refined`, `promoted`, `draft_rejected_already_exists` | Crystalliser/skill-promotion loop |
| `quox.approval.*` | `cancelled_workflow_deactivated` | Approval lifecycle events |

### Stack Modes
- **Pipeline:** ordered list of layers (`mode: "pipeline"`)
- **DAG:** nodes + edges with `when` conditions for branching (`mode: "dag"`)

## Example Stacks

### realtime-alert.json
```
L0 → L1 → L3 → L7 → L9 → L10
```
Speed-first: 15s timeout, 16 parallel actions, skips planning/context.

### restricted-textonly.json
```
L0 → L1 → L3 → L6 → L9 → L10
```
Safety-first: tool execution disabled, network access disabled.

## Design Principles

1. **Layered control** — work flows through ordered layers with clear responsibilities
2. **Branchable, bypassable** — alternate paths and skipping allowed, but auditable
3. **Delta-first context** — layers emit patches and refs, not huge objects
4. **Audit-first** — every run answers: what happened, why, what changed, who approved
5. **Runtime-agnostic** — AOCL standardizes control flow, not execution

## Observability Features

- **Trace Timeline** — APM-style spans per layer with timing
- **Context Diff Viewer** — delta per layer (added/modified/removed keys)
- **Branch Map** — DAG view showing path taken vs skipped branches
- **NDJSON logging** — one AEE envelope per line for grep/ELK/Kafka

## Related Protocols

| Protocol | Role | Repo |
|----------|------|------|
| AEE | Envelope format + causality | github.com/quoxai/aee |
| AOCL | Orchestration control layers | *(this repo)* |
| VOLT | Verifiable evidence ledger | github.com/quoxai/volt |
| WARD | Content-free hash-chain witnessing | github.com/quoxai/ward |

**How they connect:**
- **AEE → AOCL**: AOCL processes AEE envelopes and emits `aocl.*` events for audit
- **AOCL → VOLT**: Policy decisions become VOLT evidence events (tamper-evident)
- **WARD witnesses AOCL**: Content-free receipts of decisions for external verification

## Roadmap Summary
- **v0.1** (current): Core spec, layer contract, AEE binding
- **v0.2**: JSON schemas + validators
- **v0.3**: Transport bindings (HTTP, WS, NATS, Kafka)
- **v0.4**: Control-plane hardening (bypass policy, HITL)
- **v0.5**: OpenTelemetry mapping, replay tooling
- **v0.6**: Reference implementations (Python, TypeScript)
