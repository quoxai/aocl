<!-- Last verified: 2026-03-27 by /codebase-mirror -->

# AOCL (Agent Orchestration Control Layers) — Codebase Map

## Spec Status
| Field | Value |
|-------|-------|
| Version | 0.1 |
| IETF Draft | draft-cowles-aocl-00 |
| License | MIT |

## Architecture
Control-layer pipeline: smart routing, policy gating, context retrieval, delegation, verification, response assembly. Emits `aocl.*` AEE envelopes for audit.

## Key Files
| File | Purpose |
|------|---------|
| `README.md` | Protocol overview (121 lines) |
| `ROADMAP.md` | Planned extensions |
| `docs/spec.md` | Core concepts and layer contract |
| `docs/aee-binding.md` | AEE integration (aocl.* intents) |
| `docs/stacks.md` | Default layer taxonomy |
| `docs/observability.md` | Tracing and layer activation |
| `docs/examples.md` | End-to-end traces |
| `examples/stacks/` | 2 stack configs (realtime-alert, restricted-textonly) |
