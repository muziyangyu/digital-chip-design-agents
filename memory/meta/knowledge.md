# Pipeline Orchestration — Domain Knowledge

## Cross-Domain Loop Patterns

### Common verification flap signatures
- **Off-by-one in address calculation** — frequently shows as `wrap_addr` or `incr_addr` mismatch in AXI burst tests; fix usually confined to 2–5 lines in the address-generation logic.
- **Reset-domain crossing oversight** — RTL lint passes but simulation fails on reset-active transactions; fix requires synchroniser insertion or reset qualification.
- **State-machine dead state** — directed test reaches unreachable state; fix is adding a default branch or a recovery transition.
- **Width mismatch on signed/unsigned boundary** — shows as sign-extension artefacts; fix is an explicit cast.

### Iteration-cap heuristics
- If `cross_domain_iteration_count=2` without any `status=fixed` entry, the suspected_rtl location is likely wrong. Suggest the user re-examine `waveform_path` before approving a 3rd dispatch.
- If a new fix_request has the same `summary` as a previous `status=fixed` entry, the fix did not hold — check whether the RTL file was committed before simulation ran.
- If `cross_domain_iteration_count=3` is reached within a single session, the root cause is usually a misdiagnosis in `suspected_rtl.module`. Escalation message should suggest waveform re-analysis.

### Escalation message templates

**Cap exceeded (3 iterations)**:
```
Pipeline loop exceeded 3 cross-domain iterations for fix_request <id>.
Bug summary: <summary>
Last RTL fix attempted: <rtl_response.diff_summary>
Waveform at: <waveform_path>

Action required: please review the waveform and re-identify the root cause,
then clear `pending_approval` and reset `cross_domain_iteration_count` to 0
before invoking /chip-design-meta:pipeline-orchestration again.
```

**RTL abandoned without fix**:
```
RTL orchestrator terminated without closing fix_request <id> (status stayed claimed).
Bug summary: <summary>
Possible causes: max turns exceeded during RTL coding; design too complex for auto-fix.

Action required: manually fix <suspected_rtl.file> around lines <line_range>,
update fix_request status to "fixed" with rtl_response populated,
then re-invoke /chip-design-verification:functional-verification.
```

## RTL Fix-Request Idioms

- When `failure_class=formal_cex`, the CEX trace often pinpoints the exact failing cycle. Remind the RTL orchestrator to load the trace in the sim tool before modifying RTL.
- `coverage_gap` fix_requests indicate missing RTL behaviour, not a bug per se. The RTL orchestrator should confirm whether new logic is needed or the testbench stimulus is insufficient before modifying RTL.
- Always check `suspected_rtl.line_range` — if it is `[0, 0]`, the bug location is unknown and the RTL orchestrator should run lint + simulation replay before modifying code.
