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
  "id": "fr_<pipeline_session_id>_<YYYYMMDD>_<HHMMSS>_<seq>",
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
- Only the `pipeline-orchestrator` sets `cross_domain_iteration_count`, `pipeline_session_id`, `pipeline_config`, and moves resolved entries to `archive_fix_requests[]`.
- Domain orchestrators **may** set `pending_approval` exclusively with `type: "checkpoint"` at their own sign-off stage; `type: "escalation"` remains the sole responsibility of the `pipeline-orchestrator`.
- `approved_checkpoints[]` is written by the user (or by an orchestrator executing an explicit approval instruction) and read by all orchestrators.
- All agents may append to `fix_request.history[]` but must not overwrite each other's entries.

### Iteration cap

`cross_domain_iteration_count` in `design_state.json` tracks the total number of
verification↔RTL dispatch cycles for the current pipeline session. The cap is controlled by
`pipeline_config.max_cross_domain_iterations` (default: 3 if absent). The orchestrator treats the cap as "reaches or exceeds" (use `>= max_cross_domain_iterations`), writing a `pending_approval` entry and exiting as soon as the iteration count is equal to or greater than the cap, preventing an off-by-one extra cycle before escalation:

```json
{
  "pending_approval": {
    "type": "escalation",
    "stage": null,
    "agent": "pipeline-orchestrator",
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

Top-level fields in `design_state.json` managed by the `pipeline-orchestrator` and related infrastructure:

- **`pipeline_session_id`** (`"ps_<YYYYMMDD>_<HHMMSS>"` or `null`): identifies the active pipeline run. Set on entry, cleared (set to null) on successful signoff. Scopes divergence checks and archival to the current session — entries from prior sessions are ignored by the divergence check.
- **`pipeline_config`** (`object`): user-tunable pipeline settings. Written with defaults on first run; never overwritten if already present.
  - `max_cross_domain_iterations` (integer, default 3): the iteration cap for the feedback loop.
  - `checkpoints` (array of strings, default `[]`): stage names that require human approval before the producing orchestrator may declare sign-off. Empty array ⇒ fully autonomous (preserves today's behavior). Example default: `["arch_signoff", "rtl_signoff", "signoff"]`. Written by the user; domain orchestrators read but never overwrite this field.
- **`approved_checkpoints[]`**: list of stages the user has approved. Schema per entry: `{ "stage": "<stage name>", "approved_at": "<ISO-8601>", "approved_by": "user" }`. Written by the user (or by an orchestrator acting on an explicit approval instruction); read by all orchestrators at their sign-off gate check.
- **`archive_fix_requests[]`**: resolved entries from completed pipeline sessions, moved here by the pipeline-orchestrator on successful signoff. Same schema as `fix_requests[]`. Never written by domain orchestrators.

### format_version

`design_state.json` version tiers (each tier is a superset of the previous):
- **`"1.1"`**: `fix_requests[]` and `cross_domain_iteration_count` present.
- **`"1.2"`**: `history[]` entries carry standardized `confidence`, `failure_class`, and `suggested_next_step` fields.
- **`"1.3"`**: `pipeline_config.checkpoints`, `approved_checkpoints[]`, `pending_approval.type/stage/agent` present; per-stage `history[]` entries (one entry per completed stage, not just one terminal entry per run).

All orchestrators must:
- Upgrade to `"1.3"` if absent or currently `"1.0"`, `"1.1"`, or `"1.2"`; never downgrade.
- All prior-version requirements are subsumed by `"1.3"` — no separate upgrades required.
- Treat missing `fix_requests` or `cross_domain_iteration_count` as `[]` / `0`.
- Treat missing `confidence`, `failure_class`, or `suggested_next_step` in history entries as `null` for backward compatibility.
- Treat missing `pipeline_config.checkpoints` as `[]` (no checkpoints — fully autonomous).
- Treat missing `approved_checkpoints` as `[]`.
- Treat missing `pending_approval.type` as `"escalation"` for backward compatibility.

### Approval Checkpoints

Proactive human-in-the-loop gates at configurable stage boundaries. Orthogonal to the
failure-driven `pending_approval` (type `escalation`) set by the pipeline-orchestrator.

#### Checkpoint configuration

```json
{
  "pipeline_config": {
    "checkpoints": ["arch_signoff", "rtl_signoff", "signoff"]
  },
  "approved_checkpoints": [
    { "stage": "arch_signoff", "approved_at": "<ISO-8601>", "approved_by": "user" }
  ]
}
```

Default: `checkpoints: []` → no gates, fully autonomous (backward compatible).

#### Gate logic (applied by every domain orchestrator at its sign-off stage)

Before setting the domain's `signoff=true` and writing it to `design_state.json`:

1. **Skip the gate in fix-request-servicing mode**: if a `fix_request.id` was passed in the
   prompt (invoked by meta to repair a bug), skip the checkpoint check entirely. Checkpoints
   gate forward design progression, not the automated verify↔RTL repair loop.

2. Read `pipeline_config.checkpoints` from `design_state.json`. If the orchestrator's
   sign-off stage name appears in the list **and** does not appear in `approved_checkpoints[].stage`:
   - Atomic RMW: set `pending_approval`:
     ```json
     {
       "type": "checkpoint",
       "stage": "<sign-off stage name>",
       "agent": "<this-orchestrator>",
       "reason": "checkpoint <stage> requires human approval before proceeding",
       "fix_request_id": null,
       "last_summary": "<QoR one-liner>",
       "requires_user": true
     }
     ```
   - Append a `history[]` entry with `decision: "await_approval"`, `confidence: "high"`,
     `failure_class: "none"`, `suggested_next_step: "escalate"`, `reason: "checkpoint <stage> requires human approval"`.
   - **Do not** set `signoff=true`. Print the gate message to the user and halt.

3. On re-invocation: if the stage now appears in `approved_checkpoints[]`, clear
   `pending_approval` (atomic RMW → set to `null`), then proceed to set `signoff=true`.

#### Resume paths

The user may resume a gated orchestrator by either:
- **Manual edit**: append `{ "stage": "<stage>", "approved_at": "<ISO-8601>", "approved_by": "user" }` to `approved_checkpoints[]` and set `pending_approval=null` in `design_state.json`, then re-invoke the orchestrator.
- **Approval instruction**: invoke the orchestrator with "approve checkpoint `<stage>`" in the prompt — the orchestrator performs the `approved_checkpoints[]` append and `pending_approval` clear (atomic RMW) itself, then continues.

#### pending_approval type-awareness (pipeline-orchestrator)

The `pipeline-orchestrator`'s `detect_open_fix_requests` halts on **any** non-null
`pending_approval` (conservative — a domain checkpoint also blocks meta dispatch). It prints
a type-specific message:
- `type: "checkpoint"`: "Checkpoint `<stage>` is awaiting human approval (set by `<agent>`). Approve or skip to continue."
- `type: "escalation"`: (existing message) "Fix-request loop escalation — review required."

#### Per-stage history trace (format_version 1.3)

Every domain orchestrator appends one `history[]` entry after each internal stage completes
(PASS, FAIL, WARN), not just at session end. The last entry written is the terminal entry
read by the pipeline-orchestrator's decision table. Entry shape is unchanged (same 9-field
schema). This enables post-run audits without replaying the full conversation.

### Programmatic branching on standardized history[] fields

After a domain orchestrator completes, the pipeline-orchestrator reads the terminal
`history[]` entry to make retry/escalate decisions without string-parsing prose. Decision
table (evaluated in order):

| `confidence` | `failure_class` | `suggested_next_step` | Pipeline action |
|---|---|---|---|
| any | `resource_limit` | any | Escalate via `pending_approval` — cap exceeded |
| `low` | any | `escalate` | Escalate — result unreliable, human review required |
| `low` | any | any (not escalate) | Escalate — low confidence overrides any retry intent |
| any | `tool_error` | `retry_stage` | Re-dispatch the same orchestrator once; if still `tool_error`, escalate |
| any | `functional` \| `coverage_gap` | `escalate` | Append new `fix_request` and loop back via RTL orchestrator |
| `high` \| `medium` | `none` | `proceed` | Advance to next stage / signoff |
| any | any | `abandon` | Escalate via `pending_approval` — child reports unrecoverable, human decision required |

For any combination not in the table, apply the most conservative matching rule (prefer
`escalate` over retry). Programmatic branches must read from the history entry's structured
fields — do not re-derive intent from `reason` (free-text, for humans only).

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
