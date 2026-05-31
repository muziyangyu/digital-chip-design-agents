---
name: rtl-design
description: >
  SystemVerilog RTL design — module planning, coding standards enforcement, lint
  checking, CDC/RDC analysis, and synthesis readiness verification. Use when
  writing, reviewing, or debugging RTL for ASIC or FPGA targets, or when
  checking an existing RTL package for synthesis readiness.
version: 1.0.0
author: chuanseng-ng
license: MIT
allowed-tools: Read, Write, Bash
---

# Skill: RTL Design (SystemVerilog)

## Invocation

When this skill is loaded and a user presents an RTL design task, **do not
execute stages directly**. Immediately spawn the
`digital-chip-design-agents:rtl-design-orchestrator` agent and pass the full
user request and any available context to it. The orchestrator enforces the stage
sequence, loop-back rules, and sign-off criteria defined below.

Use the domain rules in this file only when the orchestrator reads this skill
mid-flow for stage-specific guidance, or when the user asks a targeted reference
question rather than requesting a full flow execution.

## Pre-run Context

Before executing or advising on **any** stage, read the following files if they exist:

1. `memory/rtl-design/knowledge.md` — known failure patterns, successful tool flags, PDK/tool quirks.
   Incorporate its guidance into every stage decision. If absent, proceed without it.
2. `memory/rtl-design/run_state.md` — current run identity (`run_id`, `design_name`, `tool`,
   `last_stage`). Use this to resume correctly after interruption. If absent, a new run
   is starting; the orchestrator will create this file before the first stage.

This pre-run read applies whether this skill is loaded by a user or called by the
orchestrator mid-flow. It ensures the fix database is consulted before any diagnosis step.

## Purpose
Guide RTL development from module hierarchy planning through lint-clean,
CDC-clean, synthesis-ready RTL. Enforces industry-standard SystemVerilog
coding practices and produces a signed-off RTL package ready for simulation
and synthesis handoff.

---

## Supported EDA Tools

### Open-Source
- **Verilator** (`verilator --lint-only`) — fast lint and simulation
- **Slang** (`slang`) — modern, standards-compliant SV parser and elaborator
- **Surelog** (`surelog`) — SystemVerilog pre-processor and front-end for Yosys
- **sv2v** (`sv2v`) — SystemVerilog-to-Verilog converter
- **Icarus Verilog** (`iverilog`) — Verilog/SV simulator for quick sanity checks

### Proprietary
- **Synopsys SpyGlass** (`spyglass`) — lint, CDC, RDC, and clock-domain analysis
- **Cadence JasperGold CDC** (`jg`) — formal CDC verification
- **Siemens Questa CDC** (`vsim`) — CDC analysis and sign-off

---

## Stage: module_planning

### Domain Rules
1. Top-down decomposition: start with top-level module, recurse to leaf cells
2. Each module: single clear responsibility (single responsibility principle)
3. Define all port lists before coding (direction, width, type)
4. Identify all clock domains per module; mark CDC crossings explicitly
5. Identify all reset domains; mark synchronous vs asynchronous
6. Parameterise widths and depths wherever possible
7. No logic in top-level integration modules — wiring only
8. Separate datapath and control into distinct sub-modules

### Output Required
- Module hierarchy tree
- Module descriptor (name, purpose, clock domain, ports, sub-modules) per module
- Interface/port list document

---

## Stage: rtl_coding

### Domain Rules — General
1. Always use `logic` type (not wire/reg distinction)
2. All ports: explicitly typed and directioned
3. `default_nettype none` at top of every file
4. No latches: all always_comb blocks must have complete case and assignment coverage
5. No blocking assignments (=) in always_ff blocks
6. No non-blocking assignments (<=) in always_comb blocks
7. One always block per register or coherent register group
8. Reset all registers explicitly; synchronous reset preferred for ASIC

### Domain Rules — Naming Conventions
- Clocks:       `clk_[domain]`
- Resets:       `rst_n_[domain]` (active-low) or `rst_[domain]`
- Active-low:   `signal_n` suffix
- Registered:   `signal_q` suffix
- Next-state:   `signal_d` suffix
- Parameters:   `UPPER_SNAKE_CASE`
- Modules/Signals: `lower_snake_case`

### Domain Rules — Synthesis Safety
1. No delays (#) in RTL — simulation only
2. No initial blocks in ASIC RTL
3. Use `unique case` with explicit don't-cares instead of casez/casex
4. Flag any net with fanout > `design_state.constraints.timing.fanout_max` (default: 32) for buffering intent review
5. No combinational loops — will cause synthesis errors
6. Pipeline registers: clearly marked with `_q` suffix at each stage

### Domain Rules — CDC
1. Two-FF synchroniser for every single-bit CDC crossing
2. Async FIFO for multi-bit CDC data paths
3. Gray-coded pointers for async FIFO crossing
4. Never sample asynchronous data directly in synchronous logic

### Domain Rules — Power Intent (Clock Gating)
Apply these rules for every clock domain. Read `clock_power_budget` from the architecture
hand-off package. **For orchestrated Architecture → RTL runs, the `clock_power_budget` table
is a required handoff contract; if missing, treat as a handoff violation and abort with a
clear error directing the user to notify upstream packaging.** For non-orchestrated or local
RTL-only runs, classify domains using toggle-count estimates from Verilator simulation as
a fallback.

1. **High gating opportunity domains** (α < `design_state.constraints.power.activity_factors.default` (default: 0.15) from architecture, or toggle rate below that threshold from Verilator): insert an ICG cell (`CLKGATETST_X*` or technology-equivalent) at the outermost clock enable boundary. Do not rely on synthesis to infer clock gates — explicit ICG insertion at RTL is required.
2. **Moderate gating opportunity domains** (`activity_factors.default` ≤ α < `activity_factors.high` (defaults: 0.15–0.40)): insert ICG at the sub-block level for any register file or datapath wider than 32 bits.
3. **Always-on domains** (α ≥ `activity_factors.high` (default: 0.40), or documented as always-on in architecture hand-off):
   no ICG required; add a `/* always-on: <reason> */` comment at the clock port declaration.
4. ICG enable signal: must be registered (setup-timing safe); combinational enable
   is a lint error.
5. ICG cells: use only library-approved cells (`CLKGATETST_*` for testability with
   scan-enable override); do not use behavioural `if (enable) clk_gated = clk` constructs.
6. After inserting ICGs, measure `clock_gating_coverage`:
   `coverage = (register bits behind an ICG) / (total register bits in domain) × 100%`
   Report this metric in the `rtl_signoff` output.

### Supported Tools for Power Intent

| Tool | Type | Use |
|------|------|-----|
| Verilator | Open-source | Toggle coverage → activity factor for gating classification |
| SpyGlass (Synopsys) | Proprietary | RTL power lint, missing ICG detection |
| VC Static (Synopsys) | Proprietary | Power-intent rule checking |
| Questa PowerPro (Siemens) | Proprietary | Formal power analysis |

### Output Required
- RTL source files (.sv) per module
- SVA assertion files per module
- Inline comments on all non-obvious logic
- `clock_gating_coverage` metric per domain (appended to sign-off record)

---

## Stage: lint_check

### Domain Rules
1. ERROR level (must fix): latches, incomplete sensitivity lists,
   undriven outputs, multiply-driven signals, X-propagation sources
2. WARNING level (review): unused ports, truncated assignments,
   bit-width mismatches, constant conditions
3. All waivers: must include signal name, rule ID, justification, approver
4. No ERROR-level waivers without architect approval
5. All waivers logged in `lint_waivers.csv`

### QoR Metrics to Evaluate
- ERROR count: must be 0 before proceeding
- WARNING count: review all; waive with documented justification
- All RTL files checked (not just top-level)

### Output Required
- Lint report (per file, per rule)
- Waiver file
- Clean lint summary

---

## Stage: cdc_rdc_analysis

### CDC Rules
1. Every CDC crossing: approved synchroniser primitive
2. Single-bit control: 2-FF synchroniser minimum
3. Multi-bit data: async FIFO or handshake protocol
4. Pulse crossings: pulse stretcher + synchroniser
5. Zero CDC violations (unwaived) before proceeding

### RDC Rules
1. All reset domains explicitly defined in constraints
2. Reset de-assertion: synchronous to receiving clock domain
3. No combinational logic between reset sources
4. Retention registers: correct UPF annotation

### QoR Metrics to Evaluate
- CDC violations (unwaived): 0
- RDC violations (unwaived): 0
- All clock domains verified in tool constraints

### Output Required
- CDC/RDC report
- Synchroniser instance list
- Waiver file

---

## Stage: synth_check

### Domain Rules
1. Run synthesis at target frequency with typical corner
2. Check for unmapped cells (technology library gaps)
3. Identify critical paths — report to architect if WNS < −0.5 ns
4. Check area vs microarch estimate (< 120% acceptable)
5. Check for multi-driven nets or unresolved X
6. Flag high-fanout nets needing buffering strategy
7. Verify all clock definitions synthesise correctly

### QoR Metrics to Evaluate
- WNS at target frequency: > −0.5 ns acceptable at this stage (sign-off target: `design_state.constraints.timing.wns_ns_target`, default: 0)
- Area: < 120% of microarch estimate
- No unmapped cells
- No multi-driven nets

### Output Required
- Synthesis area report
- Timing report (critical paths)
- Recommendations for RTL fixes if needed

---

## Stage: rtl_signoff

### Sign-off Checklist
- [ ] All modules from planning implemented
- [ ] Lint: 0 errors, all warnings reviewed
- [ ] CDC: 0 unwaived violations
- [ ] RDC: 0 unwaived violations
- [ ] Synthesis check: WNS within acceptable range
- [ ] All ports connected in integration
- [ ] SVA assertions in place for key properties
- [ ] Code review completed
- [ ] File list and compile order documented
- [ ] ICG cells inserted for all high/moderate gating opportunity domains
- [ ] Always-on domains annotated with `/* always-on: <reason> */`
- [ ] `clock_gating_coverage` ≥ `design_state.constraints.power.gating_coverage_pct_min`% for high-opportunity domains (default: 60%); reported in sign-off record

### Output Required
- RTL file package (all .sv files)
- File list (filelist.f)
- Compile order document
- Assertion library (.sva files)
- RTL sign-off record

---

## Constraint Validation

See `plugins/meta/skills/pipeline-orchestration/SKILL.md` §Constraints Schema for the authoritative schema and stage-entry validation rule.

**Required at entry (`module_planning`) — hard-fail if missing:**
- `constraints.clock.clk_mhz` — target frequency for synth_check timing evaluation

**Optional (schema defaults apply when absent):**
- `constraints.timing.fanout_max` (default: 32) — high-fanout threshold
- `constraints.timing.wns_ns_target` (default: 0) — WNS sign-off target
- `constraints.power.gating_coverage_pct_min` (default: 60%) — ICG coverage target
- `constraints.power.activity_factors` (defaults: `{default: 0.15, high: 0.40}`) — domain classification

---

## Memory

### Write on stage completion
After each stage completes (regardless of whether an orchestrator session is active),
write or overwrite one JSON record in `memory/rtl-design/experiences.jsonl` keyed by
`run_id`. This ensures data is persisted even if the flow is interrupted or called
without full orchestrator context.

Use `run_id` = `rtl-design_<YYYYMMDD>_<HHMMSS>` (set once at flow start; reuse on each
stage update). Set `signoff_achieved: false` until the final sign-off stage completes.
### Run state (write before first stage, update after each stage)
Write `memory/rtl-design/run_state.md` as the **first action** before launching any tool:
```markdown
run_id:      rtl-design_<YYYYMMDD>_<HHMMSS>
design_name: <design>
tool:        <primary tool>
start_time:  <ISO-8601>
last_stage:  null
```
Update `last_stage` to the completed stage name only after each stage finishes successfully. This file lets wakeup-loop prompts
and resumed sessions identify the correct run without relying on in-memory state.
Create the file and parent directories if they do not exist.

### Optional: claude-mem index
If `mcp__plugin_ecc_memory__add_observations` is available in this session, emit each
applied fix as an observation to entity `chip-design-rtl-design-fixes` after writing to
`experiences.jsonl`. Skip silently if the tool is absent — JSONL is the canonical record.
