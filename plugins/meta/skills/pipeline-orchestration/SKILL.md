---
name: pipeline-orchestration
description: >
  Cross-domain loop orchestration for the chip design pipeline. Provides the
  fix_request protocol, iteration-cap logic, escalation templates, and dispatch
  patterns for routing verification/formal failures to the RTL orchestrator and
  back. Use when driving the closed-loop verification↔RTL feedback cycle.
version: 1.0.0
author: chuanseng-ng
license: MIT
allowed-tools: Read, Write, Bash
---

# Skill: Pipeline Orchestration

## Invocation

- **If invoked by a user** presenting a pipeline loop task: immediately spawn the
  `digital-chip-design-agents:pipeline-orchestrator` agent and pass the full user
  request and any available context. Do not execute stages directly.
- **If invoked inside another orchestrator**: read `design_state.json`, summarise open
  `fix_requests[]`, and return — do not spawn subagents (anti-recursion rule).

## Purpose

This skill provides the closed-loop verification↔RTL feedback protocol. When a DUT bug
is found during simulation or formal verification, it must be communicated to the RTL
orchestrator in a machine-actionable way and the pipeline must iterate until the bug is
fixed or the iteration limit is reached.

The protocol has three participants:

| Participant | Role |
|---|---|
| **verification-orchestrator** / **formal-orchestrator** | Detects the bug; writes a `fix_request` entry to `design_state.fix_requests[]` with `status=open`; terminates with `decision=escalate`. |
| **rtl-design-orchestrator** | Reads the open `fix_request`; sets `status=claimed`; fixes the RTL; sets `status=fixed` with `rtl_response`; terminates. |
| **pipeline-orchestrator** | Detects open entries; assigns a `pipeline_session_id`; dispatches RTL then re-verification in sequence; enforces a configurable cap (default 3, via `pipeline_config.max_cross_domain_iterations`); scopes divergence checks to the current session; archives resolved entries on signoff; escalates via `pending_approval` if cap exceeded. |

## Domain Rules

### fix_request Schema (authoritative)

All entries in `design_state.fix_requests[]` must conform to this schema:

```json
{
  "id": "fr_<YYYYMMDD>_<HHMMSS>_<seq>",
  "created_at": "<ISO-8601>",
  "updated_at": "<ISO-8601>",
  "created_by": "verification-orchestrator | formal-orchestrator",
  "failure_class": "functional | protocol | coverage_gap | formal_cex",
  "test_name": "<directed test or property name>",
  "property_or_assertion": "<assertion id or null>",
  "seed": 0,
  "waveform_path": "<path or null>",
  "log_path": "<path or null>",
  "suspected_rtl": {
    "module": "<module name>",
    "signal": "<signal or null>",
    "file": "<rtl/path.sv or null>",
    "line_range": [0, 0]
  },
  "summary": "<one-line bug description>",
  "expected_behavior": "<spec excerpt or null>",
  "observed_behavior": "<observed RTL behaviour>",
  "session_id": "<pipeline_session_id or null>",
  "status": "open | claimed | fixed | abandoned",
  "rtl_response": null,
  "history": []
}
```

`rtl_response` (populated by rtl-design-orchestrator on close):
```json
{
  "fixed_at": "<ISO-8601>",
  "diff_summary": "<one-paragraph description of changes>",
  "files_changed": ["rtl/path.sv"],
  "commit_ref": null
}
```

`fix_request.history[]` — one entry per state transition:
```json
{
  "timestamp": "<ISO-8601>",
  "agent": "<agent name>",
  "from_status": "<previous status>",
  "to_status": "<new status>",
  "note": "<optional one-liner>"
}
```

### Ownership rules

- `rtl-design-orchestrator` owns the `open→claimed` and `claimed→fixed|abandoned` transitions.
- Only the `rtl-design-orchestrator` sets `status=claimed→fixed` or `claimed→abandoned`.
- Only the `pipeline-orchestrator` sets `cross_domain_iteration_count`, `pipeline_session_id`, `pipeline_config`, `pending_approval`, and moves resolved entries to `archive_fix_requests[]`.
- All agents may append to `fix_request.history[]` but must not overwrite each other's entries.

### Iteration cap

`cross_domain_iteration_count` in `design_state.json` tracks the total number of
verification↔RTL dispatch cycles for the current pipeline session. The cap is controlled by
`pipeline_config.max_cross_domain_iterations` (default: 3 if absent). When the count exceeds
the cap, the pipeline-orchestrator writes a `pending_approval` entry and exits:

```json
{
  "pending_approval": {
    "reason": "fix_request loop exceeded 3 cross-domain iterations",
    "fix_request_id": "<id>",
    "last_summary": "<last rtl_response.diff_summary>",
    "requires_user": true
  }
}
```

The user must review the escalation, manually fix the RTL or adjust the testbench, then
clear `pending_approval` (set to `null`) and reset `cross_domain_iteration_count` to 0
before invoking the pipeline-orchestrator again. Optionally increase
`pipeline_config.max_cross_domain_iterations` if more iterations are warranted.

### Pipeline session fields

Three additional top-level fields in `design_state.json` are managed exclusively by the `pipeline-orchestrator`:

- **`pipeline_session_id`** (`"ps_<YYYYMMDD>_<HHMMSS>"` or `null`): identifies the active pipeline run. Set on entry, cleared (set to null) on successful signoff. Scopes divergence checks and archival to the current session — entries from prior sessions are ignored by the divergence check.
- **`pipeline_config`** (`object`): user-tunable pipeline settings. Written with defaults on first run; never overwritten if already present.
  - `max_cross_domain_iterations` (integer, default 3): the iteration cap for the feedback loop.
- **`archive_fix_requests[]`**: resolved entries from completed pipeline sessions, moved here by the pipeline-orchestrator on successful signoff. Same schema as `fix_requests[]`. Never written by domain orchestrators.

### format_version

`design_state.json` uses `format_version: "1.1"` when `fix_requests[]` or
`cross_domain_iteration_count` are present. All orchestrators must:
- Preserve `"1.1"` if already set (do not downgrade to `"1.0"`).
- Set `"1.1"` when first writing a `fix_request`.
- Treat missing `fix_requests` or `cross_domain_iteration_count` as `[]` / `0` respectively.

### Dispatch pattern (pipeline-orchestrator)

Sequential dispatch — never parallel:
1. RTL orchestrator (fix the bug) — block until complete.
2. Verification or formal orchestrator (validate the fix) — block until complete.

Spawn form for the Agent/Task tool:
- RTL: `subagent_type: chip-design-rtl:rtl-design-orchestrator`
- Verification: `subagent_type: chip-design-verification:verification-orchestrator`
- Formal: `subagent_type: chip-design-formal:formal-orchestrator`

Always pass the `fix_request.id` in the subagent prompt so the child can locate its work item without scanning the whole array.

### V2 extension points (not wired in V1)

- Architecture↔RTL refinement loop: `architecture.refinement_needed=true` could trigger
  an arch re-run. The `fix_request` schema is intentionally producer-agnostic; only
  `created_by` would need a new value (`architecture-orchestrator`).
- Formal property-bug routing: `failure_class=formal_cex` with `suspected_owner=formal`
  would route to the formal orchestrator instead of RTL. Not implemented in V1.
- **LEC unmatched-points loop**: `lec_run: unmatched points` in `formal-orchestrator.md` is intentionally **not** connected to the fix_request protocol in V1. LEC failures are netlist↔RTL mismatches introduced at synthesis — the correct consumer is `synthesis-orchestrator`, not `rtl-design-orchestrator`. Deferred to V2.

## QoR Metrics

- `cross_domain_iteration_count` — number of RTL↔verify dispatch cycles (target: ≤ 2 for clean designs)
- `time_to_signoff` — wall-clock time from first open `fix_request` to `verification_status.signoff=true`
- `escalation_rate` — fraction of design sessions that hit the 3-iteration cap (target: < 10%)
- `fix_request_abandonment_rate` — fraction of `fix_requests` that reach `status=abandoned` (target: 0%)

## Output Required

- `design_state.json` with all `fix_requests[]` entries at terminal status (`fixed` or `abandoned`)
- `design_state.json` with `cross_domain_iteration_count` updated
- `memory/meta/experiences.jsonl` entry for this run
- A console summary to the user: which fix_requests were processed, how many iterations, and the outcome (converged / escalated / no open requests)
