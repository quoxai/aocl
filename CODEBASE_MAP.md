<!-- Last verified: 2026-07-27T15:30:00Z by /codebase-mirror -->

# AOCL — Codebase Map

**Agent Orchestration Control Layers (AOCL)** — A control-layer protocol for AI agent orchestration that produces observability as a first-class output. Standardizes *how an orchestrator processes an incoming event* through ordered layers. Part of the Quox protocol family (AEE, AOCL, VOLT, WARD). IETF Internet-Draft: `draft-cowles-aocl-00`. Specification only — no runtime code.

## Metrics
| Metric | Count |
|--------|-------|
| Protocol Version | **0.1** (Experimental) |
| IETF Draft | `draft-cowles-aocl-00` |
| Schema files | 0 (JSON schemas deferred to v0.2) |
| Example files | 3 (2 stacks + 1 trace) |
| Spec docs (Markdown) | 5 (`docs/`) |
| License | MIT |

## Directory Structure
```
aocl/
├── README.md                         # Protocol overview, relationship to AEE/VOLT/WARD
├── ROADMAP.md                        # v0.1 → v0.6 planned evolution
├── LICENSE                           # MIT License
├── CODEBASE_MAP.md                   # This file — structural overview
├── docs/
│   ├── spec.md                       # Core spec: layers, context bundle, design principles
│   ├── stacks.md                     # Layer taxonomy (L0–L10), stack format (pipeline/DAG)
│   ├── aee-binding.md                # AOCL ↔ AEE integration + QuoxFlow intents
│   ├── observability.md              # Trace events, NDJSON logging, UI views
│   └── examples.md                   # End-to-end trace + stack variant patterns
└── examples/
    ├── stacks/
    │   ├── realtime-alert.json       # Speed-first stack (6 layers, 15s timeout)
    │   └── restricted-textonly.json  # Safety-first stack (tools/network disabled)
    └── traces/
        └── backup-check.ndjson       # 14-envelope end-to-end AEE trace
```

## Reference SDK

**`@quox/aocl`** (TypeScript/Node, zero runtime dependencies) at **github.com/quoxai/aocl-sdk**

The SDK provides:
- Canonical L0–L10 layer taxonomy as data
- Corr-scoped layer tracer emitting `aocl.*` AEE envelopes per `docs/aee-binding.md`
- Stack loading and validation
- Trace normalizer
- Candidate JSON Schemas for v0.2
- `aocl-trace` CLI

Policy evaluation stays out of the SDK (tracer records, never decides). SDK repo is private until launch.

## Authoritative Files
| File | Purpose |
|------|---------|
| `docs/spec.md` | Core spec: layer contract, context bundle (C0–C6), design principles |
| `docs/stacks.md` | Layer taxonomy (L0–L10), stack formats (pipeline/DAG), bypass rules |
| `docs/aee-binding.md` | How AOCL emits `aocl.*` AEE envelopes; QuoxFlow intent extensions |
| `docs/observability.md` | Trace events, NDJSON logging conventions, 3 UI views |
| `docs/examples.md` | End-to-end trace patterns + copy-paste stack variants |
| `examples/stacks/*.json` | Runnable stack definitions (realtime-alert, restricted-textonly) |
| `examples/traces/backup-check.ndjson` | 14-envelope reference trace (ops.backup.status.check) |

## Canonical Layers (L0–L10)
| Layer | ID | Responsibility |
|-------|----|----------------|
| L0 | `ingress.normalize` | Normalize incoming events, create run IDs, basic parsing |
| L1 | `identity.scope` | Apply identity, permissions, secret scope, redaction rules |
| L2 | `route.smart` | Deterministic fast-path router (pattern match, cached answers) |
| L3 | `policy.gate` | Safety/compliance checks, tool/model restrictions, HITL requirements |
| L4 | `plan.decompose` | Convert intent into structured objectives, multi-step/multi-agent |
| L5 | `context.retrieve` | Retrieve memory/files/RAG context; produce refs + bounded summaries |
| L6 | `shape.rewrite` | Rewrite/structure into operational form (AEE tasks, tool plans) |
| L7 | `delegate.execute` | Delegate to agents/tools, manage dependencies/concurrency |
| L8 | `verify.check` | Verification/evals/consistency checks; evidence requirements |
| L9 | `assemble.respond` | Assemble final response; redact; apply tone/formatting |
| L10 | `audit.writeback` | Persist trace summaries and permitted memory writeback |

## Context Bundle Partitions (C0–C6)
| Partition | Contents |
|-----------|----------|
| C0 Event | Source, timestamp, channel, correlation IDs, attachments |
| C1 Identity | User/org, roles, permissions, secret scope, redaction rules |
| C2 Task | Goal, constraints, definition of done, priority |
| C3 Memory | Retrieved notes, RAG hits, file refs, summaries |
| C4 Policy | Safety/compliance rules, allowed tools, model restrictions |
| C5 Execution | Budgets, timeouts, concurrency, tool registry, model routing |
| C6 Audit | Log level, required evidence, compliance checkpoints |

## AEE Binding Intents

### Core AOCL Intents (`aocl.*`)
| Intent | Description |
|--------|-------------|
| `aocl.stack.select` | Which stack/branch was chosen |
| `aocl.layer.enter` | Layer started |
| `aocl.layer.exit` | Layer completed (timing) |
| `aocl.layer.decision` | Layer made a decision (route/policy/plan) |
| `aocl.context.patch` | Layer changed context (delta + digests) |
| `aocl.control.branch` | Branch taken/rejected |
| `aocl.control.bypass` | Bypass requested/allowed/denied |
| `aocl.verify.result` | Verification summary |
| `aocl.run.summary` | End-of-run summary |

### QuoxFlow Extensions (Section 10 of aee-binding.md)
| Namespace | Intents | Trigger |
|-----------|---------|---------|
| `quox.budget.*` | `exhausted`, `warning` | Pre-execution budget check fails/approaches threshold |
| `quox.policy.*` | `rejected`, `gate_error` | Policy layer denies execution or errors |
| `agentic.skill.*` | `retrieved`, `drafted`, `refined`, `promoted`, `draft_rejected_already_exists` | Agentic executor crystalliser/skill-promotion loop |
| `quox.approval.*` | `cancelled_workflow_deactivated` | Pending approval cancelled (workflow deactivated) |

## Example Stacks
| Stack ID | Flow | Timeout | Parallel | Notes |
|----------|------|---------|----------|-------|
| `realtime-alert` | L0→L1→L3→L7→L9→L10 | 15s | 16 | Speed-first, skips planning/context/verify |
| `restricted-textonly` | L0→L1→L3→L6→L9→L10 | 30s | 0 | Safety-first, tools/network disabled |

## Stack Modes
| Mode | Description |
|------|-------------|
| `pipeline` | Ordered list of layers executed sequentially |
| `dag` | Nodes + edges with `when` conditions for branching |

## Related Protocols
| Protocol | Role | Relationship to AOCL |
|----------|------|----------------------|
| **AEE** | Envelope format + causality | AOCL processes AEE envelopes, emits `aocl.*` audit events |
| **VOLT** | Verifiable evidence ledger | AOCL policy decisions become VOLT evidence events |
| **WARD** | Content-free hash-chain witnessing | WARD produces receipts of AOCL decisions without content |

## Invariants
| Check | Status | Details |
|-------|--------|---------|
| Version present in spec | pass | `0.1` in README table and docs/spec.md H1 |
| RFC 2119 keywords used | pass | All spec docs use MUST/SHOULD/MAY correctly |
| Schema validity | defer | JSON schemas deferred to v0.2 per ROADMAP |
| Example stacks present | pass | 2 stack JSONs in examples/stacks/ |
| Example trace present | pass | 1 NDJSON trace (14 lines) in examples/traces/ |

## Roadmap Summary
| Version | Focus |
|---------|-------|
| v0.1 (current) | Minimal viable protocol: layer contract, pipeline/DAG stacks, AEE binding |
| v0.2 | JSON schemas + validators for layer payloads |
| v0.3 | Transport bindings (HTTP, WS, NATS, Kafka) |
| v0.4 | Control-plane hardening (bypass policy schema, HITL handshake) |
| v0.5 | Observability upgrades (OpenTelemetry mapping, replay tooling, reference UI) |
| v0.6 | Reference implementations (Python, TypeScript, framework adapters) |
