<!-- Last verified: 2026-03-30 by /codebase-mirror -->

# AOCL (Agent Orchestration Control Layers) — Codebase Map

AOCL is a control-layer protocol for AI agent orchestration that produces observability as a first-class output. It standardizes how an orchestrator processes incoming events through ordered layers.

## Spec Status
| Field | Value |
|-------|-------|
| Version | 0.1 (Experimental) |
| IETF Draft | `draft-cowles-aocl-00` |
| License | MIT |

## Repository Structure
```
aocl/
├── README.md                 # Protocol overview + mental model
├── ROADMAP.md                # v0.1-v0.6 planned extensions
├── LICENSE                   # MIT
├── CODEBASE_MAP.md           # This file
├── docs/
│   ├── spec.md               # Core layer contract + context bundle
│   ├── aee-binding.md        # AEE integration (no AEE changes required)
│   ├── stacks.md             # Layer taxonomy + stack definition format
│   ├── observability.md      # Tracing, logging, NDJSON conventions
│   └── examples.md           # End-to-end traces + stack variants
└── examples/
    ├── stacks/
    │   ├── realtime-alert.json       # Speed-first pipeline (6 layers)
    │   └── restricted-textonly.json  # Safety-first pipeline (no tools)
    └── traces/
        └── backup-check.ndjson       # Full AEE trace example (14 events)
```

## Canonical Layer Pipeline (L0-L10)
| Layer | ID | Purpose |
|-------|-----|---------|
| L0 | `L0.ingress.normalize` | Normalize events, create run IDs |
| L1 | `L1.identity.scope` | Apply identity, permissions, redaction |
| L2 | `L2.route.smart` | Deterministic fast-path routing |
| L3 | `L3.policy.gate` | Safety/compliance checks, HITL requirements |
| L4 | `L4.plan.decompose` | Convert intent to structured objectives |
| L5 | `L5.context.retrieve` | Retrieve memory/files/RAG context |
| L6 | `L6.shape.rewrite` | Rewrite into operational form |
| L7 | `L7.delegate.execute` | Delegate to agents/tools |
| L8 | `L8.verify.check` | Verification/evals/consistency |
| L9 | `L9.assemble.respond` | Assemble final response |
| L10 | `L10.audit.writeback` | Persist trace + memory writeback |

## Context Bundle Partitions (C0-C6)
| Partition | Contents |
|-----------|----------|
| C0 Event | Source, timestamp, channel, correlation IDs |
| C1 Identity | User/org, roles, permissions, secret scope |
| C2 Task | Goal, constraints, definition of done |
| C3 Memory | Retrieved notes, RAG hits, file refs |
| C4 Policy | Safety rules, allowed tools, model restrictions |
| C5 Execution | Budgets, timeouts, concurrency, tool registry |
| C6 Audit | Log level, required evidence, checkpoints |

## AOCL Intent Namespace
Core audit intents emitted as AEE envelopes:
- `aocl.stack.select` — Stack/branch selection
- `aocl.layer.enter` / `aocl.layer.exit` — Layer lifecycle
- `aocl.layer.decision` — Layer decisions + context delta
- `aocl.context.patch` — Context modifications
- `aocl.control.branch` / `aocl.control.bypass` — Control flow changes
- `aocl.verify.result` — Verification summary
- `aocl.run.summary` — End-of-run summary

## Stack Modes
| Mode | Description |
|------|-------------|
| `pipeline` | Simple ordered list of layers |
| `dag` | Nodes + edges with `when` conditions for branching |

## Example Stacks
| Stack | Layers | Use Case |
|-------|--------|----------|
| `realtime-alert` | L0→L1→L3→L7→L9→L10 | Speed-first (15s timeout) |
| `restricted-textonly` | L0→L1→L3→L6→L9→L10 | No tool execution, text-only |

## Core Design Principles
1. **Layered control** — Work flows through ordered layers with clear responsibilities
2. **Branchable, bypassable** — Supports alternate paths and skipping layers (audited)
3. **Delta-first context** — Layers emit context deltas and refs/digests, not huge objects
4. **Audit-first** — Traces answer: what happened, why, what changed, who approved

## Related Protocols (Quox Family)
| Protocol | Role | Relationship |
|----------|------|--------------|
| AEE | Envelope format + causality | AOCL emits `aocl.*` AEE envelopes |
| VOLT | Verifiable evidence ledger | AOCL decisions become VOLT events |
| WARD | Content-free hash-chain witnessing | WARD witnesses AOCL decisions |

## Roadmap Highlights
| Version | Focus |
|---------|-------|
| v0.1 | Current — minimal viable protocol |
| v0.2 | JSON schemas + validators |
| v0.3 | Transport bindings (HTTP/WS/NATS/Kafka) |
| v0.4 | Bypass policy + HITL handshake |
| v0.5 | OpenTelemetry mapping + replay tooling |
| v0.6 | Reference implementations (TS/Python) |

## Invariants
| Check | Status |
|-------|--------|
| All 11 layers documented | ✓ |
| Example stacks valid JSON | ✓ |
| AEE binding spec complete | ✓ |
| Observability spec complete | ✓ |
