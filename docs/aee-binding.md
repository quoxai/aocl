# AOCL ↔ AEE Binding (v0.1)

AOCL (Agent Orchestration Control Layers) is designed to work alongside AEE (Agent Envelope Exchange).

- **AEE** standardizes the message envelope for agent↔agent / human↔agent exchange (identity + intent + causality).
- **AOCL** standardizes the layered control pipeline an orchestrator runs to decide what happens.

**This binding defines how AOCL uses AEE without changing the AEE envelope.**

**AEE repo:** https://github.com/quoxai/aee

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119).

---

## 1. Design Goals

1. **No AEE changes required**  
   AOCL operates entirely via `intent` conventions and `payload` schemas.

2. **Layer activity is auditable**  
   AOCL emits AEE envelopes to record layer activation, decisions, context deltas, and control-flow changes.

3. **Keep messages small**  
   AOCL avoids shipping giant context blobs in AEE payloads. Prefer refs + digests + patches.

4. **Causality stays computable**  
   All AOCL-produced AEE envelopes share the same `corr` as the originating work item, and use `reply_to` consistently.

---

## 2. Envelope Types Used

AOCL uses existing AEE `type` categories:

- `task` — when AOCL delegates work to an agent/worker
- `result` — when an agent/worker returns work
- `event` — when AOCL emits an audit record (layer enter/exit/decision)
- `stream` — optional: for incremental traces or long-running layer output
- `error` — when a layer or delegated work fails in a structured way

AOCL does not require any new AEE `type` values.

---

## 3. AOCL Intent Namespace

AOCL defines an intent namespace for layer-level audit and control:

### Core Intents (Recommended)
- `aocl.stack.select`
- `aocl.layer.enter`
- `aocl.layer.exit`
- `aocl.layer.decision`
- `aocl.context.patch`
- `aocl.control.branch`
- `aocl.control.bypass`
- `aocl.verify.result`
- `aocl.run.summary`

### Delegation Intents
AOCL does not impose intent names for business tasks. It delegates using the caller’s intent (e.g., `ops.backup.status.check`) or a derived intent convention.

---

## 4. `corr` and `reply_to` Conventions (Important)

### 4.1 Correlation (`corr`)
All envelopes generated during processing of a single work item MUST share the same `corr` value as the originating request.

This allows grep-able traces across:
- layer events
- agent tasks/results
- verification
- final response

### 4.2 Reply chaining (`reply_to`)
AOCL supports two valid strategies. Pick one and be consistent.

#### Strategy A — Root-linked (simpler)
- All AOCL layer events set `reply_to` to the **originating request envelope**.
- All delegated tasks set `reply_to` to the **originating request** (or to the latest control decision, if preferred).

Pros:
- simplest trace
- easy to query by `corr` + root `id`

Cons:
- less like span trees; you infer ordering from timestamps

#### Strategy B — Span-linked (more detailed)
- Each layer event `reply_to` points to the *previous layer event* (forming a strict chain).
- Delegated tasks `reply_to` points to the layer decision event that emitted the task.

Pros:
- explicit causal chain (span-like)
- perfect replay ordering

Cons:
- more envelopes to manage
- strict dependency on event emission order

**Recommendation (v0.1):** Strategy A for minimalism. Strategy B is a later upgrade if you want APM-like traces.

---

## 5. Payload Patterns (Keep It Small)

AOCL layer envelopes SHOULD NOT contain the full context bundle.

Instead, payloads SHOULD include:
- `layer_id`, `layer_version`
- `decisions[]` (short reasons)
- `context_delta` (small patch or merge delta)
- `context_refs[]` (pointers to large data)
- `context_digests` (for replay / integrity)
- `control_flags` (branch/bypass/halt/HITL)
- `timing_ms` (optional, for observability)

### Recommended Payload Keys
- `run_id` — orchestration run identifier (optional but useful)
- `layer` — `{ "id": "...", "version": "..." }`
- `decisions` — list of `{code, reason}`
- `delta` — patch/merge delta
- `refs` — array of string refs
- `digests` — map of digest names to digest values
- `control` — control flags

---

## 6. Example: AOCL Layer Decision as an AEE `event`

```json
{
  "v": "1",
  "id": "01AOCL_LAYER_EVENT_0001",
  "ts": "2026-01-26T12:00:01Z",
  "type": "event",
  "from": "agent.orchestrator",
  "to": "log.aocl",
  "intent": "aocl.layer.decision",
  "corr": "01AOCL_CORR_0001",
  "reply_to": "01ORIGIN_TASK_0001",
  "trace": null,
  "priority": "normal",
  "requires": null,
  "payload": {
    "run_id": "RUN-01AOCL-0001",
    "layer": {"id": "L3.policy.gate", "version": "0.1"},
    "decisions": [
      {"code": "POLICY_ALLOW", "reason": "No restricted content; tools allowed: read-only"}
    ],
    "delta": {
      "C4.Policy.allowed_tools": ["tool.readonly.*"],
      "C5.Execution.mode": "restricted-readonly"
    },
    "refs": [],
    "digests": {
      "context_in": "sha256:...",
      "context_out": "sha256:..."
    },
    "control": {"halt_pipeline": false}
  },
  "sig": null
}
```

**Notes:**

- `to` is a logical sink (`log.aocl`) — it may map to a file, Kafka topic, NATS subject, etc.
- The AOCL meaning is in the `intent` + `payload`, while the AEE envelope stays stable.

---

## 7. Example: AOCL Delegation as AEE `task`

When a layer decides to delegate work to a worker agent, it emits an AEE `task` envelope:

```json
{
  "v": "1",
  "id": "01AOCL_DELEGATE_TASK_0001",
  "ts": "2026-01-26T12:00:03Z",
  "type": "task",
  "from": "agent.orchestrator",
  "to": "agent.backup_auditor",
  "intent": "ops.backup.status.check",
  "corr": "01AOCL_CORR_0001",
  "reply_to": "01ORIGIN_TASK_0001",
  "trace": null,
  "priority": "high",
  "requires": {"timeout_ms": 30000, "evidence": true},
  "payload": {
    "cluster_ref": "inv://clusters/node.lan",
    "window": "24h",
    "context_refs": ["mem://run/RUN-01AOCL-0001/context-snapshot"]
  },
  "sig": null
}
```

---

## 8. Bypass and Branch Decisions MUST Be Explicit

If bypassing layers is requested or applied, AOCL MUST emit an AEE `event` capturing:

- Who requested bypass (identity context)
- Which layers were skipped
- Why it was allowed/denied
- Which policy permitted it

**Recommended intent:**

- `aocl.control.bypass` with payload `{requested_by, layers, allowed, reason}`

**Same for branching:**

- `aocl.control.branch` with payload `{from, to, reason}`

---

## 9. What This Binding Does NOT Define (Yet)

- Transport bindings (HTTP, WS, NATS, Kafka)
- Signature (`sig`) format and verification rules
- Canonical JSON serialization rules for digests
- A global registry for AOCL layer IDs

---

## 10. QuoxFlow Intent Vocabulary (R5.low)

QuoxFlow extends the AOCL intent namespace with the following intents. All are emitted as AEE `event` envelopes.

### Budget / Cost Control (`quox.budget.*`)

| Intent | Trigger | Key Payload Fields |
|--------|---------|-------------------|
| `quox.budget.exhausted` | Pre-execution budget check fails | `executionId`, `workflowId`, `scope`, `scopeId`, `orgId`, `exceeded: string[]`, `reason: string` |
| `quox.budget.warning` | Usage approaches a budget threshold | `executionId`, `workflowId`, `scope`, `scopeId`, `orgId`, `exceeded: string[]`, `reason: string` |

### Policy Gates (`quox.policy.*`)

| Intent | Trigger | Key Payload Fields |
|--------|---------|-------------------|
| `quox.policy.rejected` | Policy layer denies execution (POLICY_DENIED) | `executionId`, `workflowId`, `policyId`, `policyName`, `reason: string` |
| `quox.policy.gate_error` | Unexpected error inside the policy gate (non-deny) | `executionId`, `workflowId`, `error: string` |

### Agentic Skill Loop (`agentic.skill.*`)

Emitted by the agentic executor during the crystalliser / skill-promotion loop (STUDY-REV-2 P9).

| Intent | Trigger | Key Payload Fields |
|--------|---------|-------------------|
| `agentic.skill.retrieved` | Matching skill found in library | `executionId`, `workflowId`, `skillId`, `skillName` |
| `agentic.skill.drafted` | New skill candidate drafted | `executionId`, `workflowId`, `skillId`, `draftHash` |
| `agentic.skill.refined` | Skill candidate iterated/refined | `executionId`, `workflowId`, `skillId`, `iteration: number` |
| `agentic.skill.promoted` | Skill candidate promoted to library | `executionId`, `workflowId`, `skillId`, `skillName` |
| `agentic.skill.draft_rejected_already_exists` | Draft rejected because skill already in library | `executionId`, `workflowId`, `existingSkillId` |

### Approval Lifecycle (`quox.approval.*`)

| Intent | Trigger | Key Payload Fields |
|--------|---------|-------------------|
| `quox.approval.cancelled_workflow_deactivated` | Pending approval cancelled because its workflow was deactivated | `approvalId`, `workflowId`, `executionId` |

---

All intents in the `quox.*` and `agentic.*` namespaces are defined in `quoxflow/src/core/envelope.ts` under the `Intents` constant.

These can be added in `ROADMAP.md` once the v0.1 core is stable.
