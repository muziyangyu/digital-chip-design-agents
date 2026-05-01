# Future Work

Items deferred from the two-tier agent memory system implementation (see `memory/README.md`).
Track these as follow-up issues after the initial memory system ships.

## 1. Memory-Keeper Skill ✓ IMPLEMENTED

**Status:** Shipped — `plugins/infrastructure/skills/memory-keeper/`

A skill that periodically distils `memory/<domain>/experiences.jsonl`
into an updated `memory/<domain>/knowledge.md` summary. Implementation includes:

- `SKILL.md` — three-stage agent skill (load_experiences → distil_knowledge → report)
- `distill.py` — CLI helper: parses JSONL, computes issue/fix pairs, metric ranges, and
  tool-flag candidates; emits a structured JSON summary for the agent to use
- Invocable via `/chip-design-infrastructure:memory-keeper --domain <name>` or `--all`
- Threshold guard: skips domains with fewer than N records (default: 5)

## 2. Semantic Search Over Experiences

An MCP memory server that embeds experience records and allows orchestrators to query by similarity
(e.g., "what fixed WNS issues on sky130 before?") rather than full-file read.

**Prerequisite**: Experience log must be large enough to justify the infrastructure overhead —
target threshold is ~50 records per domain before semantic search adds value over keyword grep.

Implementation options:
- SQLite + sqlite-vec extension (zero-dependency, file-based)
- Chroma or Qdrant (local Docker container)
- Hosted: Pinecone, Weaviate Cloud

## 3. Cross-Design Metric Trending ✓ IMPLEMENTED

**Status:** Shipped — `tools/qor_trends.py`

A reporting utility that reads all `memory/<domain>/experiences.jsonl` files and
produces QoR trend tables and optional matplotlib charts for a named design. Use cases:

- Regression detection: flags when a metric degrades across runs (⚠ alert column)
- PDK comparison: compare WNS/area across sky130 vs GF180 for the same RTL — use `--group-by pdk` (note: tools/qor_trends.py applies `--pdk` filtering before `--group-by`, so combining `--group-by pdk` with a specific `--pdk VALUE` will collapse to a single group and defeat the comparison)
- Tool comparison: compare Yosys vs DC synthesis area for the same design — use `--group-by tool` (note: tools/qor_trends.py applies `--tool` filtering before `--group-by`, so combining `--group-by tool` with a specific `--tool VALUE` will collapse to a single group and defeat the comparison)

Usage:
```bash
# Text table for all domains where design "aes_core" appears
python3 tools/qor_trends.py --design aes_core

# WNS trend for synthesis only
python3 tools/qor_trends.py --design aes_core --domain synthesis --metric wns_ns

# Save a matplotlib chart
python3 tools/qor_trends.py --design aes_core --plot --output aes_core_qor.png
```

## 4. Infrastructure Orchestrator Memory

Track tool versions, install paths, and MCP configuration choices across setups in
`memory/infrastructure/`. Deferred because:

- Infrastructure state is environment-specific (machine A ≠ machine B)
- The value is lower than domain memory — tool version is better tracked in a lockfile
- MCP configuration is already stored in `.claude/settings.json`

Revisit if tool version mismatches cause repeated debugging across sessions.

## 5. Central "Design" State

Today each orchestrator maintains its own session-scoped state object. There is no shared
artifact that survives an orchestrator boundary, so downstream agents cannot see upstream
decisions (e.g., RTL coding cannot query the architecture trade-off rationale).

Introduce a persistent `design_state.json` written to the working directory. Every orchestrator
reads it on entry and appends its results on exit. This replaces the ad-hoc inter-orchestrator
handoff packages currently documented in `docs/MASTER_INDEX.md`.

### Minimum fields

```json
{
  "spec": { "raw": "<natural language>", "structured": {} },
  "interfaces": [{ "name": "AXI3-lite", "width": 32, "role": "subordinate" }],
  "constraints": { "clk_mhz": 500, "area_um2": null, "power_mw": null },
  "rtl": { "top_module": null, "files": [], "lint_clean": false, "cdc_clean": false },
  "verification_status": { "coverage_pct": null, "signoff": false },
  "tool_feedback": [],
  "history": []
}
```

`history[]` entries record the agent, stage, decision, reason, and constraint reference so that
the entire evolution of the design is traceable across sessions.

**Prerequisite for:** items 6, 8, 9, 11.

## 6. Continuous Verification Loop

Each orchestrator already loops within its own domain (e.g., lint fail → re-run RTL coding,
coverage gap → re-run constrained random). What is missing is a feedback path *across*
orchestrator boundaries: when the verification orchestrator finds a DUT bug, there is no
automated callback to the RTL orchestrator.

The flow changes from:

```
Architecture → RTL → Verification  (linear, no feedback)
```

To:

```
RTL → Verify → [bug found?] → RTL (with bug context) → Verify → … → pass
```

**Scope of change:**
- Verification orchestrator writes a structured "RTL fix request" to `design_state.json`
  on DUT bug detection (functional mismatch or AXI protocol violation).
- A thin meta-orchestrator (or a cross-domain loop rule) detects the pending request
  and re-invokes the RTL orchestrator from the `rtl_coding` stage, passing the bug context.
- Maximum cross-domain iterations: 3 before escalating to the user.

**Prerequisite:** item 5 (design_state.json must exist for the fix request handoff).

## 7. Agent Contract Standardization

The stage-agent output format defined in `docs/MASTER_INDEX.md` is already close to a
unified contract, but three fields are missing: confidence score, failure class, and
suggested next step. Without these, orchestrators use ad-hoc `recommendation` strings
and cannot drive retry logic programmatically.

**Target stage-agent output schema (additions in bold):**

```json
{
  "stage": "<stage_name>",
  "status": "PASS | FAIL | WARN",
  "qor": {},
  "issues": [{ "severity": "ERROR | WARN", "description": "...", "fix": "..." }],
  "recommendation": "proceed | loop_back_to:<stage> | escalate",
  "output": {},
  "confidence": 0.85,
  "failure_class": "none | invalid_rtl | verification_failure | interface_mismatch | incomplete_spec",
  "suggested_next_step": "<agent_or_stage_name>"
}
```

`confidence` (0.0–1.0) allows orchestrators to weight outputs when multiple candidates
exist or when deciding whether to proceed without human review (item 11).
`failure_class` feeds directly into the retry strategy table in item 10.
`suggested_next_step` makes orchestration logic explicit rather than embedded in prose rules.

All 14 orchestrator `.md` files must be updated to emit and consume these fields.

## 8. Constraint Awareness

Constraints currently live as prose in SKILL.md files or as file paths (SDC, LEF) in
orchestrator state. Agents embed constraint values in their rules rather than reading them
from a shared source, so a change to the clock target requires editing multiple skill files.

**Agent Behavior Changes:**

- On stage entry, read `design_state.constraints` (item 5) and fail fast if required
  constraints are absent rather than assuming defaults silently.
- Tag each design decision with the constraint it satisfies, e.g.:
  `"meets_constraint": "clk_core ≥ 500 MHz"`.
- If a constraint cannot be met, set `failure_class: interface_mismatch` or
  `incomplete_spec` (item 10) and halt rather than producing a non-compliant output.

**Affected domains:** architecture, RTL, synthesis, STA, PD — all have timing/area
targets that are currently hardcoded in skill rules.

**Prerequisite:** item 5.

## 9. Architecture Exploration Improvement

The architecture SKILL.md already mandates generating three candidates (conservative,
balanced, aggressive) with a trade-off matrix. The gap is that candidates exist only
in the session context — they are not persisted, and there is no mechanism for a
downstream failure (e.g., synthesis cannot close timing) to trigger a return to
architecture with that constraint violation as input.

**Improvements:**

- Persist the full trade-off matrix to `design_state.json` under
  `architecture.candidates[]` so downstream agents can reference the rejected options.
- Add a `refinement_needed` flag: if synthesis or PD sets
  `design_state.architecture.refinement_needed = true` with a reason, the architecture
  orchestrator re-enters at `perf_modelling` using the persisted candidates as a
  starting point rather than generating from scratch.
- Extend `memory/architecture/experiences.jsonl` schema to record
  `candidates_evaluated` count and `winning_candidate_profile` for cross-design learning.

This reduces early bad decisions without requiring perfect up-front prompts, and
leverages the memory system already in place.

**Prerequisite:** item 5.

## 10. Structured Failure Handling

All failures currently land in the same `issues[]` array with only `ERROR | WARN`
severity. Orchestrators apply hardcoded loop-back rules per stage but have no
structured way to distinguish a recoverable code error from an ambiguous spec.

**Failure class taxonomy** (maps to `failure_class` field from item 7):

| Class | Definition | Default retry strategy |
|---|---|---|
| `invalid_rtl` | Syntax, lint, or CDC errors in generated RTL | `regenerate` — re-run rtl_coding with error context |
| `verification_failure` | DUT functional bug found by simulation | `refine` — re-run rtl_coding with failing test + waveform |
| `interface_mismatch` | AXI/protocol violation or port width conflict | `refine` — re-run rtl_coding targeting the violated interface |
| `incomplete_spec` | Ambiguous or missing requirement blocks progress | `escalate` — halt and request user clarification |

**Agent behavior changes:**
- Tag every FAIL status with one of the four classes above.
- Attach `retry_strategy` to the issue: `regenerate | refine | escalate`.
- If max loop iterations are reached, the escalation message must include the failure
  class and a plain-language description of what information the user must provide to
  unblock the flow.

**Prerequisite:** item 7 (failure_class field in output contract).

## 11. Human-in-the-loop Control Points + Observability

Currently the only human interaction points are invocation and max-iteration escalation.
There are no optional approval gates between pipeline stages, and there is no structured
record of why an agent made a particular decision.

### Approval Checkpoints

Add a `require_approval` flag to configurable stage transitions in `design_state.json`.
When set, the orchestrator writes a human-readable summary to
`design_state.pending_approval` and halts. The user resumes by setting
`design_state.approved_by_human: true` (manually or via a `--approve` flag).

Suggested default checkpoint positions:
- After `arch_signoff` — before RTL coding begins
- After `rtl_signoff` — before verification or synthesis begins
- Before tape-out (PD `signoff` stage)

### Execution Trace

Each stage completion appends an entry to `design_state.history[]`:

```json
{
  "timestamp": "2026-04-18T10:00:00Z",
  "agent": "architecture-orchestrator",
  "stage": "arch_signoff",
  "decision": "proceed",
  "reason": "Balanced candidate meets timing with >20% WNS headroom.",
  "constraint_ref": "clk_core_500MHz"
}
```

This provides a clear relationship between each decision and its output, and enables
post-run audits without replaying the full agent conversation.

**Prerequisite:** item 5 (design_state.json), item 7 (confidence score informs whether
approval is required).

## 12. New Adjacent Agent Domains

The following domains are adjacent to the current 14-plugin pipeline and are not yet covered.
Each represents a distinct discipline with its own toolchain and sign-off criteria.

| Domain | Rationale | Key tools |
|--------|-----------|-----------|
| **Power Intent / UPF** | Multi-voltage, low-power design (UPF/CPF authoring, power domain verification, isolation/retention cell insertion). Mandatory for mobile/IoT chips and not covered by RTL, Synthesis, or PD agents. | Synopsys MVSIM, Cadence CPF tools, open-source: uvmf-power |
| **Silicon Validation / Debug** | Post-silicon failure analysis, ATE interface bring-up, scan dump triage, silicon characterisation. Completely unaddressed by any current agent. | Teradyne UltraFLEX, Advantest V93000, internal ATE scripts |
| **Package & Chiplet / 2.5D-3D** | Die-to-die interface definition (UCIe, HBM), bump/RDL floorplanning, package-level SI/PI analysis. Distinct from chip-level PD. | Cadence Sigrity, Synopsys 3DIC Compiler, open-source: KiCad |
| **Security / Hardware Roots-of-Trust** | Side-channel analysis (power/EM), fault injection modelling, secure boot ROM design, PUF integration. Cross-cuts RTL and firmware but warrants its own specialist agent. | ChipWhisperer, SideChannelMarvels, Synopsys DesignWare Security |
| **NoC / Interconnect Design** | Network-on-Chip topology exploration, latency/bandwidth modelling, flit-level simulation. Currently assumed inside SoC integration but deep enough for a dedicated agent. | gem5 (network mode), Noxim, open-source: OpenSoC Fabric |
| **Emulation Platform (ZeBu/Palladium)** | Hardware emulation bringup (Synopsys ZeBu, Cadence Palladium) is distinct from FPGA prototyping — different partitioning problem, transaction-based interfaces, and tool flow. | Synopsys ZeBu, Cadence Palladium, Mentor Veloce |
| **AMS Integration** | Qualifying analog IP blocks (PLLs, ADCs, LDOs, SerDes), generating behavioral/Verilog-A models for digital co-simulation, and guiding analog peripheral firmware bring-up. Scoped as an *integration* agent rather than a full analog design agent (full analog closure requires human waveform review and is not automatable end-to-end). | ngspice, Xyce, Xschem, Spectre (behavioral model generation only) |

**Recommended priority:** Power Intent/UPF and Silicon Validation fill the most immediate gaps
and connect directly to existing agents. AMS Integration adds value for SoC flows that pull in
analog IP. The remaining domains are longer-term additions.

## 13. Domain Breakdown: Sub-Agent Specialisation

Several existing agents cover broad enough scope that splitting them into focused sub-agents
would improve parallelism, reduce context window pressure, and sharpen sign-off criteria.
The central design state (item 5) is a prerequisite for clean handoffs between sub-agents.

### Proposed Splits

**Physical Design → 3 sub-agents** *(highest priority — broadest current scope)*
- `pd-floorplan`: I/O placement, macro placement, power grid planning (70–80% utilisation target)
- `pd-implementation`: Placement, CTS (skew <150 ps), routing, DRC/LVS clean
- `pd-signoff`: Antenna fix, fill insertion, GDS export, tape-out checklist

**Verification → 3 sub-agents** *(after design state is implemented)*
- `verification-tb`: UVM TB architecture, agent design, scoreboard, coverage model definition
- `verification-regression`: Test plan execution, constrained-random stimulus, coverage closure (≥95%)
- `verification-emulation`: Firmware-driven verification on FPGA/emulator prototype

**Firmware → 3 sub-agents** *(after design state is implemented)*
- `firmware-bsp`: Board support package, linker scripts, startup code, peripheral drivers
- `firmware-rtos`: RTOS porting (FreeRTOS/Zephyr), task design, IPC primitives
- `firmware-bringup`: Chip bring-up scripts, JTAG/UART debug, post-silicon smoke tests

**Hold (do not split yet):** STA and DFT — already well-scoped; fragmentation overhead
outweighs benefit at current agent count.

### Pros of Finer-Grained Breakdown

| # | Benefit |
|---|---------|
| 1 | Tighter SKILL.md focus — one clear responsibility per file, easier to maintain |
| 2 | Smaller context window per run — more tokens available for design artefacts |
| 3 | Parallel execution — independent sub-agents (e.g., `pd-floorplan` + `verification-tb`) run concurrently |
| 4 | Deeper tool integration — a dedicated agent can specialise on one tool without cluttering a broad skill |
| 5 | Cleaner per-stage sign-off gates — one well-defined done criterion per agent |

### Cons of Finer-Grained Breakdown

| # | Risk |
|---|------|
| 1 | Cross-agent state explosion — more handoffs break without central design state (item 5) |
| 2 | Orchestration overhead — parent orchestrator must sequence more children; more failure modes |
| 3 | Redundant context loading — shared context (PDK rules, timing targets) re-loaded on each cold start |
| 4 | Marketplace discoverability — 20+ entries increases cognitive load for users choosing an agent |
| 5 | Maintenance burden — more `plugin.json` manifests, SKILL.md files, and memory directories to keep in sync |

**Prerequisite for all splits:** item 5 (central design state) must be implemented first
so sub-agents can share artefacts without dropping data at boundaries.
