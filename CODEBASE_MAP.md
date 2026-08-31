<!-- Last verified: 2026-08-11 by codebase-mirror -->

# AOCL — Codebase Map

**Agent Orchestration Control Layers (AOCL)** is a control-layer protocol for AI agent orchestration that produces observability as a first-class output. Standardizes how an orchestrator processes an incoming event through ordered layers. Part of the Quox protocol family (AEE, AOCL, VOLT, WARD). IETF Internet-Draft: `draft-cowles-aocl-00`. Specification only, no runtime code.

## Metrics
| Metric | Count |
|--------|-------|
| Protocol Version | **0.1** (Experimental) |
| IETF Draft | `draft-cowles-aocl-00` |
| Schema files | 0 (JSON schemas deferred to v0.2) |
| Example files | 3 (2 stack definitions, 1 trace) |
| Spec docs (Markdown) | 5 (`docs/`) |
| Total documentation | ~1,194 lines (spec + examples + roadmap) |
| License | Apache License 2.0 (LICENSE file) |

## Directory Structure
```
aocl/
├── README.md                         # Protocol overview, relationship to AEE/VOLT/WARD (129 lines)
├── ROADMAP.md                        # v0.1 → v0.6 planned evolution (132 lines)
├── LICENSE                           # Apache License 2.0
├── CODEBASE_MAP.md                   # This file (structural overview)
├── docs/
│   ├── spec.md                       # Core spec: layers, context bundle, design principles (78 lines)
│   ├── stacks.md                     # Layer taxonomy (L0–L10), stack format (pipeline/DAG) (197 lines)
│   ├── aee-binding.md                # AOCL ↔ AEE integration + intents (272 lines)
│   ├── observability.md              # Trace events, NDJSON logging, UI views (207 lines)
│   └── examples.md                   # End-to-end trace + stack variant patterns (122 lines)
└── examples/
    ├── stacks/
    │   ├── realtime-alert.json       # Speed-first stack (6 layers, 15s timeout) (22 lines)
    │   └── restricted-textonly.json  # Safety-first stack (tools/network disabled) (21 lines)
    └── traces/
        └── backup-check.ndjson       # Full end-to-end trace (14 AEE envelopes)
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
| File | Purpose | Lines |
|------|---------|-------|
| `docs/spec.md` | Core spec: layer contract, context bundle (C0–C6), design principles (RFC 2119) | 78 |
| `docs/aee-binding.md` | How AOCL emits `aocl.*` AEE envelopes; intents, causality | 272 |
| `docs/stacks.md` | Layer taxonomy (L0–L10), stack formats (pipeline/DAG), bypass rules | 197 |
| `docs/observability.md` | Trace events, NDJSON logging conventions, 3 UI views | 207 |
| `docs/examples.md` | End-to-end trace patterns + copy-paste stack variants | 122 |
| `examples/stacks/*.json` | Runnable stack definitions (realtime-alert, restricted-textonly) | 43 |

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

### QuoxFlow Extended Intents (Section 10)
From `docs/aee-binding.md`:

**Budget/Cost Control:**
- `quox.budget.exhausted` — Pre-execution budget check fails
- `quox.budget.warning` — Usage approaches threshold

**Policy Gates:**
- `quox.policy.rejected` — Policy layer denies execution
- `quox.policy.gate_error` — Unexpected error in policy gate

**Agentic Skill Loop:**
- `agentic.skill.retrieved` — Matching skill found in library
- `agentic.skill.drafted` — New skill candidate drafted
- `agentic.skill.refined` — Skill candidate iterated
- `agentic.skill.promoted` — Skill promoted to library
- `agentic.skill.draft_rejected_already_exists` — Draft rejected (duplicate)

**Approval Lifecycle:**
- `quox.approval.cancelled_workflow_deactivated` — Pending approval cancelled

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

## Core Design Principles

1. **Layered control** — Work flows through ordered layers with clear responsibilities
2. **Branchable, bypassable** — Alternate paths and layer skipping are auditable
3. **Delta-first context** — Emit patches + refs/digests, not huge objects
4. **Audit-first** — Produce traces answering: what happened, why, what changed, who approved bypasses

## Observability Outputs

AOCL enables three key UI views:
1. **Trace Timeline** (APM-style spans with duration, badges for BYPASSED/BRANCH/HITL)
2. **Context Diff Viewer** (per-layer deltas, refs, digests)
3. **Branch Map** (DAG view showing path taken, grayed-out skipped branches)

**Log Format:** NDJSON (newline-delimited JSON), one AEE envelope per line

## What AOCL Does NOT Define (v0.1)

- How to execute tools or schedule workers
- How to build agents or runtime implementations
- Transport layer specifics (HTTP, WebSocket, NATS, Kafka)
- Expression language for `when` conditions in DAG stacks
- Plugin registry for layer `ref` resolution
- Signature format and verification rules for stack files
- Canonical JSON serialization rules for digests

## Key Constraints

- **Bypass policy:** Implementations SHOULD NOT allow bypass of L1 (identity) or L3 (policy) layers
- **Mandatory audit:** All bypass/branch decisions MUST emit `aocl.control.*` events
- **Causality:** All envelopes for a single work item MUST share the same `corr` value
- **Small payloads:** Use refs + digests + deltas instead of full context blobs
- **Redaction:** MUST NOT log secrets; SHOULD log only refs + digests + small summaries

## Invariants
| Check | Status | Details |
|-------|--------|---------|
| Version present in spec | pass | `0.1` in README table and docs/spec.md H1 |
| RFC 2119 keywords used | pass | All spec docs use MUST/SHOULD/MAY correctly |
| Schema validity | defer | JSON schemas deferred to v0.2 per ROADMAP |
| Example stacks present | pass | 2 stack JSONs in examples/stacks/ |
| Example trace format | pass | Full NDJSON trace example in docs/examples.md and examples/traces/ |

## Roadmap Summary
| Version | Focus |
|---------|-------|
| v0.1 (current) | Minimal viable protocol: layer contract, pipeline/DAG stacks, AEE binding |
| v0.2 | JSON schemas + validators for layer payloads, canonicalization for digests |
| v0.3 | Transport bindings (HTTP, WS, NATS, Kafka) |
| v0.4 | Control-plane hardening (bypass policy schema, HITL handshake) |
| v0.5 | Observability upgrades (OpenTelemetry mapping, replay tooling, reference UI) |
| v0.6 | Reference implementations (Python, TypeScript, framework adapters) |

## Working with This Repo

This is a protocol specification repo (no executable code). Changes go through:
1. Issue discussion
2. Draft update in `docs/`
3. Example validation in `examples/`
4. IETF draft sync (when applicable)

### Reading Order (Recommended)
1. `README.md` — Overview, mental model, relationship to AEE/VOLT/WARD
2. `docs/spec.md` — Core concepts, layer contract, context bundle
3. `docs/stacks.md` — Layer taxonomy, stack definition format
4. `docs/aee-binding.md` — How AOCL emits AEE envelopes
5. `docs/observability.md` — Trace events, logging, UI views
6. `docs/examples.md` — End-to-end examples
7. `examples/` — Runnable stack definitions

## Document Details

### `README.md` (129 lines)
Entry point and protocol overview.
- Version, status, license
- What AOCL is (and isn't)
- Relationship to AEE
- Mental model (OSI-style layers)
- AOCL outputs (audit-first)
- Quick example (backup status check)
- Documentation index
- Reference SDK pointer
- Related protocols table

### `docs/spec.md` (78 lines)
The authoritative protocol specification.
- Terminology (Intent, Orchestrator, Layer, Stack, Context Bundle)
- Core Design Principles
- Context Bundle (C0–C6 partitions)
- Layer Contract (decisions, context_delta, actions, control_flags, timing, refs, digests)
- Control Flow Primitives (bypass, branch, halt_pipeline, parallel)
- Stack Execution Model (Pipeline vs DAG modes)
- Determinism and Replay
- Versioning

### `docs/aee-binding.md` (272 lines)
Defines how AOCL uses AEE without changing the AEE envelope.
- Design Goals (no AEE changes, layer activity auditable, small messages)
- Envelope Types Used (`task`, `result`, `event`, `stream`, `error`)
- AOCL Intent Namespace (`aocl.stack.select`, `aocl.layer.*`, etc.)
- Correlation (`corr`) and Reply Chaining (root-linked vs span-linked)
- Payload Schemas (layer decision event, context patch, run summary)
- QuoxFlow Extended Intents (section 10): `quox.budget.*`, `quox.policy.*`, `agentic.skill.*`, `quox.approval.*`
- Delta Encoding recommendations
- Refs and Digests patterns

### `docs/stacks.md` (197 lines)
Layer taxonomy and stack definition format.
- Recommended Layer Taxonomy (L0–L10 canonical layers)
- Minimal Stack Definition Format (pipeline mode, DAG mode)
- Branching Stacks (conditional routes)
- Bypass Policy (role-based bypass control, never-bypass rules)
- Stack Variants (realtime-alert, restricted-textonly, deep-research)

### `docs/observability.md` (207 lines)
Tracing, logging, and "see layers activate".
- Observability Goals (what/why/what changed/what did it do/where did time go)
- Minimum Trace Events (9 recommended `aocl.*` event types)
- Three UI Views (trace timeline, context diff viewer, branch map)
- NDJSON Log Convention (newline-delimited JSON for grep/Loki/ELK/Kafka/NATS)
- Recommended Payload Keys (run_id, layer, decisions, delta, refs, digests, control)
- Run Summary Event (aocl.run.summary)
- Redaction Guidance (MUST NOT log secrets, prefer refs over full content)

### `docs/examples.md` (122 lines)
End-to-end trace and stack variant examples.
- End-to-End Trace (NDJSON format with 14 AEE envelopes showing full backup check flow)
- Stack Variants (realtime-alert, restricted-textonly)
- Trace structure demonstration (layer events, delegation, results)
- Copy-paste ready examples
- Suggested repo layout

### `ROADMAP.md` (132 lines)
Protocol evolution plan (v0.1 → v0.6 and beyond).
- v0.1 (Current): Minimal viable protocol
- v0.2: Schemas & Validators
- v0.3: Transport Bindings
- v0.4: Control-Plane Hardening
- v0.5: Observability Upgrades
- v0.6: Reference Implementations
- Future ideas (parked): layer marketplaces, stack signing, conformance suite

---

*Map regenerated: 2026-08-11*
