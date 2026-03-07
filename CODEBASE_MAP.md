<!-- Last verified: 2026-03-06 by /codebase-mirror -->

# AOCL (Agent Orchestration Control Layers) — Codebase Map

## Metrics
| Metric | Count |
|--------|-------|
| Spec Files | 6 markdown + 2 JSON + 1 NDJSON |
| IETF Status | Submitted (draft-cowles-aocl-00) |
| Layers | 10 |

## Key Specs
- **Model:** OSI-style layered control pipeline for orchestrator event processing
- **Layers:** Identity/Scope → Smart Router → Policy Gate → Plan → Context → Rewrite → Delegate → Verify → Assemble → Audit
- **Integration:** Processes incoming AEE envelopes, emits `aocl.*` AEE envelopes per layer
- **Each layer:** Emits decision + context delta + optional actions

## Files
| File | Purpose |
|------|---------|
| README.md | Protocol overview |
| docs/spec.md | Core concepts and layer contract |
| docs/aee-binding.md | AEE envelope emission |
| docs/stacks.md | Stack definition format |
| docs/observability.md | Tracing and logging |
| docs/examples.md | End-to-end traces |
| examples/stacks/*.json | Stack definitions (2) |
| examples/traces/*.ndjson | Sample traces (1) |
