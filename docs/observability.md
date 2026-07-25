# AOCL Observability (v0.1)

AOCL is designed so you can *see the system think* — layer by layer — without guessing what happened.

The key words “MUST”, “MUST NOT”, “REQUIRED”, “SHALL”, “SHALL NOT”, “SHOULD”, “SHOULD NOT”, “RECOMMENDED”, “MAY”, and “OPTIONAL” in this document are to be interpreted as described in [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119).

This document defines:
- the minimum observability outputs an AOCL orchestrator SHOULD emit
- how to represent “layer activation” using AEE envelopes
- RECOMMENDED log formats for grepability and dashboards

For AOCL ↔ AEE emission rules, see: `docs/aee-binding.md`.

---

## 1. Observability Goals

An AOCL run should answer, quickly:

1. **What happened?**  
   Which layers ran, which were bypassed, which branch was selected

2. **Why did it happen?**  
   Explicit decisions (short reasons, codes) per layer

3. **What changed?**  
   Context deltas (patches) per layer, plus digests for integrity

4. **What did it do?**  
   Delegations to agents/tools and their results

5. **Where did time go?**  
   Timing per layer, plus overall run duration

---

## 2. Minimum Trace Events (Recommended)

AOCL SHOULD emit AEE `event` envelopes for:

- `aocl.stack.select` — which stack/branch was chosen (and why)
- `aocl.layer.enter` — layer started
- `aocl.layer.decision` — layer made a decision (route/policy/plan/etc.)
- `aocl.context.patch` — layer changed context (delta + digests)
- `aocl.layer.exit` — layer completed (timing)
- `aocl.control.bypass` — bypass requested / allowed / denied
- `aocl.control.branch` — branch taken / rejected
- `aocl.verify.result` — verification summary
- `aocl.run.summary` — end-of-run summary (optional but useful)

In v0.1 you can collapse enter/decision/patch/exit into a single `aocl.layer.decision` event if you want fewer records, but the four-event model is easier to visualize.

---

## 3. Three UI Views AOCL Enables

### 3.1 Trace Timeline (APM-Style)
- Each layer is a span: start/end and duration.
- Show badges: BYPASSED, BRANCH, HALT, HITL.

**Data required:**
- `run_id`
- `layer_id`
- `timing_ms` (or started/ended ts)
- `control_flags`

### 3.2 Context Diff Viewer
- Display the context delta per layer:
  - added keys
  - modified keys
  - removed keys
- Show refs (file/mem/rag/log pointers) instead of dumping huge content.

**Data required:**
- `delta` (patch/merge delta)
- `refs[]`
- `digests.context_in`, `digests.context_out`

### 3.3 Branch Map (DAG View)
- Nodes = layers
- Edges = executed transitions
- Highlight path taken; gray out skipped branches.

**Data required:**
- `stack_id`
- `branch_to` and/or executed edges (implementation-defined)

---

## 4. NDJSON Log Convention (Recommended)

Emit a single line per AEE envelope in **NDJSON** (newline-delimited JSON). This is ideal for:
- grep
- shipping to Loki/ELK
- Kafka/NATS
- replay tooling

**Example file:**
- `logs/aocl/2026-01-26.ndjson`

Each line is the full AEE envelope JSON (unchanged), as emitted.

**Recommendation:**
- Keep envelopes small: use refs + digests + deltas
- Apply redaction rules before logging (identity layer SHOULD configure these)

---

## 5. Recommended Payload Keys for Observability Events

For `intent: aocl.layer.*` payloads:

- `run_id` (string)
- `stack_id` (string)
- `layer`:
  - `id` (string)
  - `version` (string)
- `timing_ms` (number)
- `decisions[]`:
  - `code` (string)
  - `reason` (string)
- `delta` (object; small)
- `refs` (string[])
- `digests`:
  - `context_in` (string)
  - `context_out` (string)
- `control`:
  - `halt_pipeline` (bool)
  - `branch_to` (string|null)
  - `bypass_layers` (string[])
  - `require_hitl` (bool)
  - `severity` (string)

This keeps dashboards consistent even across different implementations.

---

## 6. Example: Run Summary Event

```json
{
  "v": "1",
  "id": "01AOCL_RUN_SUMMARY_0001",
  "ts": "2026-01-26T12:00:10Z",
  "type": "event",
  "from": "agent.orchestrator",
  "to": "log.aocl",
  "intent": "aocl.run.summary",
  "corr": "01AOCL_CORR_0001",
  "reply_to": "01ORIGIN_TASK_0001",
  "trace": null,
  "priority": "normal",
  "requires": null,
  "payload": {
    "run_id": "RUN-01AOCL-0001",
    "stack_id": "default",
    "layers_executed": [
      "L0.ingress.normalize",
      "L1.identity.scope",
      "L2.route.smart",
      "L3.policy.gate",
      "L5.context.retrieve",
      "L7.delegate.execute",
      "L9.assemble.respond",
      "L10.audit.writeback"
    ],
    "layers_bypassed": [],
    "branch_taken": null,
    "timing_ms_total": 9234,
    "delegations": [
      {"to": "agent.backup_auditor", "intent": "ops.backup.status.check", "status": "ok"}
    ],
    "verification": {"status": "pass", "confidence": 0.96},
    "refs": ["log://backup-store-01/job/2026-01-26T02:00Z"],
    "notes": "Trace is complete; context digests recorded per layer."
  },
  "sig": null
}
```

---

## 7. Redaction (Minimum Guidance)

AOCL stacks SHOULD treat redaction as a first-class concern:

- Identity/scope layer SHOULD set a redaction policy in context
- Logging sinks SHOULD apply it before persistence
- Implementations MUST NOT log secrets in payloads; prefer refs that require access-controlled fetch

In v0.1, the simplest safe policy is:

- MUST NOT log raw secrets
- MUST NOT log full file contents
- SHOULD log only refs + digests + small summaries

---

## 8. Roadmap Hooks (Future Observability)

Future additions (see `ROADMAP.md`):

* OpenTelemetry span mapping
* standardized `trace` usage (trace_id/span_id)
* replay tooling (re-run a corr with stored deltas/refs)
* a reference UI panel (“see layers activate”)

