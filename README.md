# AOCL Protocol -- Agent Orchestration Control Layers

| | |
|---|---|
| **Version** | 0.1 (Experimental) |
| **Status** | IETF Internet-Draft ([`draft-cowles-aocl-00`](https://datatracker.ietf.org/doc/draft-cowles-aocl/)) |
| **License** | MIT |

AOCL is a control-layer protocol for AI agent orchestration that produces observability as a first-class output.

It standardizes *how an orchestrator processes an incoming event* (user message, alert, webhook, cron tick) by passing it through ordered layers such as:

- Smart routing
- Policy and safety gating
- Context/memory retrieval
- Rewriting/structuring
- Delegation to agents/tools
- Verification
- Response assembly

**AOCL is not a runtime.**
It does not prescribe how to execute tools, schedule workers, or build agents. It standardizes the *control flow* and the *audit trail*.

---

## Relationship to AEE (Agent Envelope Exchange)

AOCL is designed to run alongside **AEE**.

- **AEE** = the minimal envelope for agent↔agent / human↔agent messages (identity + intent + causality).
- **AOCL** = the orchestrator’s layered control pipeline that decides what to do.

**Key idea:** AOCL makes layers auditable by emitting **AEE envelopes for layer activity**, not just for agent hops.

**AEE repo:** https://github.com/quoxai/aee

---

## Mental Model

Think of AOCL like an **OSI-style layer stack**, but for agent orchestration:

```
Ingress → Identity/Scope → Smart Router → Policy Gate → Plan → Context → Rewrite → Delegate → Verify → Assemble → Audit
```

AOCL supports:

- **Pipelines** (simple ordered layers)
- **Graphs** (branching and conditional paths)
- **Bypass** (skip layers under explicit control + audit)
- **Parallel** (optional: run some layers concurrently)

---

## AOCL Outputs (Audit-First)

Each layer produces:

- A decision summary (what happened and why)
- A context delta (what changed)
- Optional actions (delegate work, call tools, ask for HITL approval)

These are emitted as **AEE `event`/`stream` envelopes** using `aocl.*` intents.  
See: `docs/aee-binding.md`

---

## Quick Example (High Level)

1. A human sends an AEE `task` to the orchestrator: `ops.backup.status.check`
2. AOCL runs the request through layers:
   - L2 smart router decides whether to fast-path
   - L3 policy gate applies safety/tool constraints
   - L5 context layer retrieves memory/files
   - L7 delegate layer assigns a worker agent
3. AOCL emits AEE `event` envelopes for each layer activation/decision
4. A worker returns an AEE `result`
5. AOCL assembles the final response (and logs the trace)

---

## Documentation

- `docs/spec.md` -- Core AOCL concepts and layer contract
- `docs/aee-binding.md` -- How AOCL uses AEE without changing AEE
- `docs/stacks.md` -- Default layer taxonomy and stack definition format
- `docs/observability.md` -- Tracing, logging, and “see layers activate”
- `docs/examples.md` -- End-to-end trace and stack variant examples
- `ROADMAP.md` -- Planned extensions and future work

---

## Related Protocols

AOCL is part of the **Quox protocol family** -- four complementary specs for agentic systems:

| Protocol | Role | Repo |
|----------|------|------|
| **AEE** | Envelope format + causality | [AEE](https://github.com/quoxai/aee) |
| **AOCL** | Orchestration control layers | *(this repo)* |
| **VOLT** | Verifiable evidence ledger + tamper-evident traces | [VOLT](https://github.com/quoxai/volt) |
| **WARD** | Content-free hash-chain witnessing + external anchoring | [WARD](https://github.com/quoxai/ward) |

**How they connect:**
- **AEE → AOCL**: AOCL processes incoming AEE envelopes through its layer stack and emits `aocl.*` AEE envelopes for audit.
- **AOCL → VOLT**: Every AOCL policy decision (`allow`, `deny`, `hitl_required`) becomes a VOLT evidence event with `context.aocl_policy_id` and `context.aocl_decision_id`, creating a tamper-evident record of control decisions.
- **VOLT proves AOCL**: VOLT's hash-chained event ledger guarantees that approval sequences (e.g., "HITL required → approved → tool executed") haven't been tampered with after the fact.
- **WARD witnesses AOCL**: WARD produces content-free receipts of AOCL decisions (completions, rejections, bypasses) -- proving decisions happened without storing their content. Signed tips can be published to external sinks for independent verification.

Each protocol is independently useful. Together they provide **observable, controllable, provable, witnessed** agent operations.

## Status

**Experimental (v0.1)**
AOCL is intended to stay small: stable core semantics, with extensibility through stack definitions and intent schemas.

## License

This specification is released under the [MIT License](https://opensource.org/licenses/MIT).
