<!-- Last verified: 2026-08-12 by /codebase-mirror -->

# AOCL: Codebase Map

**Agent Orchestration Control Layers (AOCL)** is a control-layer protocol for AI agent orchestration that produces observability as a first-class output. It standardizes how an orchestrator processes an incoming event through ordered layers (pipeline or DAG). Part of the Quox protocol family (AEE, AOCL, VOLT, WARD). IETF Internet-Draft: `draft-cowles-aocl-00`. **Specification only, no runtime code**: there are no entry points, package manifests, API routes, or tests in this repo.

## Metrics
| Metric | Count |
|--------|-------|
| Protocol Version | **0.1** (Experimental) |
| IETF Draft | `draft-cowles-aocl-00` |
| Spec docs (Markdown) | 5 (`docs/`, 876 lines total) |
| Example files | 3 (2 stack JSONs + 1 NDJSON trace) |
| Schema files | 0 (JSON schemas deferred to v0.2) |
| License | MIT |

## Directory Structure
```
aocl/
├── README.md                         # Protocol overview, AEE/VOLT/WARD relationships, SDK pointer
├── ROADMAP.md                        # v0.1 → v0.6 planned evolution
├── LICENSE                           # MIT License
├── CODEBASE_MAP.md                   # This file
├── docs/
│   ├── spec.md                       # Core spec: terminology, design principles, context bundle (C0–C6)
│   ├── stacks.md                     # Layer taxonomy (L0–L10), pipeline/DAG stack formats, bypass rules
│   ├── aee-binding.md                # AOCL ↔ AEE integration, aocl.* intents, QuoxFlow extensions
│   ├── observability.md              # Trace events, NDJSON logging, 3 UI views, redaction
│   └── examples.md                   # End-to-end trace walkthrough + copy-paste stack variants
└── examples/
    ├── stacks/
    │   ├── realtime-alert.json       # Speed-first stack (6 layers, 15s timeout, bypass_policy)
    │   └── restricted-textonly.json  # Safety-first stack (tools/network disabled via policy_overrides)
    └── traces/
        └── backup-check.ndjson       # 14-envelope end-to-end AEE trace (ops.backup.status.check)
```

## Authoritative Files
| File | Purpose |
|------|---------|
| `docs/spec.md` | Core spec: layer contract, context bundle partitions (C0–C6), 4 design principles |
| `docs/stacks.md` | Layer taxonomy (L0–L10), stack formats (pipeline/DAG), bypass rules, intentional omissions |
| `docs/aee-binding.md` | How AOCL emits `aocl.*` AEE envelopes; corr/reply_to conventions; QuoxFlow intent vocabulary (section 10) |
| `docs/observability.md` | Minimum trace events, NDJSON log convention, 3 UI views (timeline, context diff, branch map), redaction guidance |
| `docs/examples.md` | End-to-end trace patterns + copy-paste stack variants + suggested repo layout |
| `examples/stacks/*.json` | Runnable stack definitions (realtime-alert, restricted-textonly) |
| `examples/traces/backup-check.ndjson` | 14-envelope reference trace |

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

Bypass rule: implementations SHOULD NOT allow bypass of L1 (identity) and L3 (policy); any allowed bypass MUST emit `aocl.control.bypass`.

## Context Bundle Partitions (C0–C6)
| Partition | Contents |
|-----------|----------|
| C0 Event | Source, timestamp, channel, correlation IDs, attachments |
| C1 Identity & Scope | User/org, roles, permissions, secret scope, redaction rules |
| C2 Task | Goal, constraints, definition of done, priority |
| C3 Memory/Knowledge | Retrieved notes, RAG hits, file refs, summaries |
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

Delegation uses plain AEE `task`/`result` envelopes (no special intent namespace), corr-linked to the run.

### QuoxFlow Extensions (aee-binding.md section 10)
| Namespace | Intents | Trigger |
|-----------|---------|---------|
| `quox.budget.*` | `exhausted`, `warning` | Pre-execution budget check fails / approaches threshold |
| `quox.policy.*` | `rejected`, `gate_error` | Policy layer denies execution or errors |
| `agentic.skill.*` | `retrieved`, `drafted`, `refined`, `promoted`, `draft_rejected_already_exists` | Agentic executor crystalliser/skill-promotion loop |
| `quox.approval.*` | `cancelled_workflow_deactivated` | Pending approval cancelled (workflow deactivated) |

Defined in `quoxflow/src/core/envelope.ts` under the `Intents` constant.

## Example Stacks
| Stack ID | Flow | Timeout | Parallel | Notes |
|----------|------|---------|----------|-------|
| `realtime-alert` | L0→L1→L3→L7→L9→L10 | 15s | 16 | Speed-first; skips plan/context/verify; bypass_policy never bypasses L1/L3 |
| `restricted-textonly` | L0→L1→L3→L6→L9→L10 | 30s | 0 | Safety-first; `policy_overrides` disables tool execution and network access |

## Stack Modes
| Mode | Description |
|------|-------------|
| `pipeline` | Ordered list of layers executed sequentially |
| `dag` | Nodes + edges with `when` string conditions for branching; `BR.*` branch nodes may be sub-stacks |

## Reference SDK

**`@quox/aocl`** (TypeScript/Node, zero runtime dependencies) at **github.com/quoxai/aocl-sdk** (private until launch). Provides: L0–L10 taxonomy as data, corr-scoped layer tracer emitting `aocl.*` envelopes, stack loading/validation, trace normalizer, candidate v0.2 JSON Schemas, `aocl-trace` CLI. Policy evaluation deliberately excluded (tracer records, never decides).

## Related Protocols
| Protocol | Role | Relationship to AOCL |
|----------|------|----------------------|
| **AEE** | Envelope format + causality | AOCL processes AEE envelopes, emits `aocl.*` audit events |
| **VOLT** | Verifiable evidence ledger | AOCL policy decisions become VOLT evidence events (`context.aocl_policy_id`, `context.aocl_decision_id`) |
| **WARD** | Content-free hash-chain witnessing | WARD produces receipts of AOCL decisions without storing content |

## Invariants
| Check | Status | Details |
|-------|--------|---------|
| Version present in spec | pass | `0.1` in README table and docs/spec.md H1 |
| RFC 2119 keywords used | pass | spec.md and stacks.md carry the RFC 2119 boilerplate |
| Schema validity | defer | JSON schemas deferred to v0.2 per ROADMAP |
| Example stacks present | pass | 2 stack JSONs in examples/stacks/ |
| Example trace present | pass | 1 NDJSON trace (14 lines) in examples/traces/ |
| No internal hostnames in examples | pass | Scrubbed in commit a8c5514; re-verified 2026-08-12 (grep clean) |

## Roadmap Summary
| Version | Focus |
|---------|-------|
| v0.1 (current) | Minimal viable protocol: layer contract, pipeline/DAG stacks, AEE binding |
| v0.2 | JSON schemas + validators (TS/Python), digest canonicalization |
| v0.3 | Transport bindings (HTTP, WebSocket, NATS, Kafka) |
| v0.4 | Control-plane hardening (bypass policy schema, branch selection standard, HITL handshake) |
| v0.5 | Observability upgrades (OpenTelemetry mapping, replay tooling, reference UI) |
| v0.6 | Reference implementations (Python/TS runner skeletons, LangGraph/AutoGen/CrewAI adapters) |
