# Agent Memory System

This directory holds persistent, file-based memory for the digital chip design orchestrators.
Agents read it at session start, write a run-state file before the first stage, and upsert
an experience record after each stage completes — no new infrastructure required.

## Two-Tier Design

### Tier 1 — `experiences.jsonl`
JSONL file with per-stage upsert/overwrite by run_id. One record per orchestrator run,
updated as stages complete. Machine-parseable; grows over time; never edited manually.

### Tier 2 — `knowledge.md`
Human- and agent-readable distilled summary. Seeded with known failure patterns, successful
tool flags, and PDK/tool quirks. Intended to be periodically updated by a memory-keeper skill
(see `FUTURE_WORK.md`) as experience records accumulate.

## Experience Record Schema

```json
{
  "run_id": "<domain>_<YYYYMMDD>_<HHMMSS>",
  "timestamp": "<ISO-8601>",
  "domain": "<domain>",
  "design_name": "<from state>",
  "pdk": "<from state if known, else null>",
  "tool_used": "<primary tool>",
  "stages_completed": ["<stage>", "..."],
  "loop_backs": {"<stage>": "<count>", "..."},
  "key_metrics": { "<domain-specific fields — see table below>" },
  "issues_encountered": ["<description>", "..."],
  "fixes_applied": ["<description>", "..."],
  "signoff_achieved": true,
  "notes": "<free-text observations>"
}
```

## Domain key_metrics Fields

| Domain       | key_metrics fields                                                    |
|--------------|-----------------------------------------------------------------------|
| architecture | `selected_arch`, `estimated_mhz`, `estimated_area_um2`              |
| compiler     | `isa_tests_passed`, `abi_compliant`, `regression_pass_rate`          |
| dft          | `scan_coverage_pct`, `atpg_fault_coverage_pct`                       |
| firmware     | `build_pass`, `flash_size_kb`, `bsp_tests_passed`                    |
| formal       | `proved`, `failed`, `unknown`                                         |
| fpga         | `lut_count`, `fmax_mhz`, `timing_met`                                |
| hls          | `latency_cycles`, `dsp_count`, `ii_achieved`                         |
| pd           | `wns_ns`, `drc_violations`, `lvs_errors`, `gds_area_um2`            |
| rtl-design   | `lint_errors`, `cdc_violations`, `synth_check_pass`                  |
| soc          | `ip_blocks_integrated`, `simulation_pass`, `memory_map_conflicts`    |
| sta          | `setup_wns_ns`, `hold_wns_ns`, `tns_ns`, `failing_paths`            |
| synthesis    | `wns_ns`, `cells`, `area_um2`, `lec_unmatched`                       |
| verification | `functional_coverage_pct`, `regression_failures`, `assertions_triggered` |

## Directory Layout

```
memory/
├── README.md                    ← this file
├── designs/                     ← per-design metric history (future use)
│   └── .gitkeep
├── architecture/
│   ├── knowledge.md             ← Tier 2: seeded domain knowledge
│   ├── experiences.jsonl        ← Tier 1: created on first run
│   └── run_state.md             ← active run identity (created at session start)
├── compiler/
├── dft/
├── firmware/
├── formal/
├── fpga/
├── hls/
├── pd/
├── rtl-design/
├── soc/
├── sta/
├── synthesis/
└── verification/
```

## Design State

`design_state.json` is a cross-orchestrator shared file written to the working directory.
It persists spec, interfaces, constraints, and per-domain outputs across all 14 orchestrator
boundaries. Every orchestrator reads it at session start (after `knowledge.md`) and performs
an atomic read-modify-write at session end (alongside `experiences.jsonl`).

Key top-level fields:
- `spec` — raw and structured product specification (written by architecture)
- `interfaces` — AXI/protocol interface list (written by architecture)
- `constraints` — shared timing, area, and power targets (written by architecture)
- `architecture`, `rtl`, `synthesis`, `sta`, `pd`, ... — per-domain signoff state
- `history[]` — append-only execution trace; one entry per orchestrator run

Atomic write protocol: read → modify → write `design_state.tmp` → rename to `design_state.json`.
This matches the `experiences.jsonl` upsert pattern and prevents partial writes.

## How Orchestrators Use This

**Session start**: Read `memory/<domain>/knowledge.md` and `memory/<domain>/run_state.md`
before the first stage. `knowledge.md` provides known failure patterns and tool flags.
`run_state.md` (if present) identifies an interrupted run to resume.

**Before first stage**: Write `memory/<domain>/run_state.md` with `run_id`, `design_name`,
`tool`, `start_time`, and `last_stage`. Update `last_stage` after each stage completes.

**Per stage**: Upsert (create-or-replace by `run_id`) one JSON line in
`memory/<domain>/experiences.jsonl` with `signoff_achieved: false` and the metrics
available so far. On final sign-off, set `signoff_achieved: true`. Do not append a second
line for the same `run_id` — overwrite the existing line.

**Optional — claude-mem index**: If `mcp__plugin_ecc_memory__add_observations` is available
in the session, also emit each applied fix as an observation to entity
`chip-design-<domain>-fixes`. Skip silently if the tool is absent — JSONL is the canonical
record; claude-mem is a supplemental cross-session search index only.
