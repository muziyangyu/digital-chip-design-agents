# Changelog

## [Unreleased] — feat/rtl-verify-feedback-loop branch

### Added

- **Closed-loop verification↔RTL feedback** (FUTURE_WORK item 6): verification and formal orchestrators now write structured `fix_request` entries to `design_state.json` when a DUT bug or formal CEX is found, instead of suspending with prose-only output. The new `chip-design-meta` plugin (`plugins/meta/agents/pipeline-orchestrator.md`) detects open `fix_requests`, dispatches the RTL orchestrator to fix the bug, re-runs the originating verification or formal check, and loops up to 3 cross-domain iterations before escalating via `pending_approval`.
- **`fix_request` schema** (`format_version 1.1`): two new top-level fields in `design_state.json` — `fix_requests[]` (structured bug handoff with id, failure_class, suspected_rtl, waveform_path, status lifecycle) and `cross_domain_iteration_count` (integer cap enforced by the pipeline-orchestrator). Fully backward-compatible — all existing readers treat missing keys as null/zero.
- **`chip-design-meta` plugin** (`plugins/meta/`): 15th plugin in the marketplace, with `pipeline-orchestrator` agent, `pipeline-orchestration` skill (hosts authoritative fix_request schema and dispatch patterns), and persistent memory under `memory/meta/`.
- **Formal orchestrator fix_request support**: `formal-orchestrator.md` now writes `failure_class=formal_cex` fix_requests (including CEX trace path) on property counter-example, matching the verification orchestrator's protocol. Both route to the RTL orchestrator for fixing in V1.

### Changed

- **Plugin count**: 14 → 15 plugins. CI assertion in `validate.yml` and `ides/copilot/applyto-map.json` domain count updated accordingly.
- **`verification-orchestrator.md`**: loop-back rule for `directed_tests DUT bug found` and Behaviour Rule 4 updated to write structured fix_requests and exit with `decision=escalate`; Design State Read step extended to handle re-invocation context.
- **`rtl-design-orchestrator.md`**: Design State Read step extended to detect and claim open fix_requests; new Behaviour Rule 6 documents the fix_request close protocol.
- **`functional-verification/SKILL.md`**: Domain Rule 7 updated from "suspend and wait for confirmation" to "write fix_request and terminate for pipeline-orchestrator dispatch".
- **`memory/README.md`**: design_state.json schema documentation updated with `fix_requests[]`, `cross_domain_iteration_count`, and `format_version 1.1` details.
- **Divergence detection scoped to current session** (`pipeline_session_id`): the pipeline-orchestrator divergence check now compares only against `status=fixed` entries sharing the same `pipeline_session_id`, preventing false escalations when the same bug class legitimately recurs after a refactor in a later session.
- **Archival on signoff**: pipeline-orchestrator success branch moves resolved `fix_requests[]` entries into `design_state.archive_fix_requests[]` and resets session state, preventing unbounded array growth across long-running designs.
- **Configurable iteration cap** (`pipeline_config.max_cross_domain_iterations`): iteration limit lifted from hardcoded `3` to a user-tunable field in `design_state.json` (default 3 if absent). Set the field to tune per-design without editing agent files.
- **LEC unmatched-points loop intentionally deferred to V2**: `lec_run: unmatched points` in `formal-orchestrator.md` is not connected to the fix_request protocol in V1 — proper support requires `synthesis-orchestrator` as a consumer. Documented in `pipeline-orchestration/SKILL.md` V2 extension points.
- **Removed vestigial per-entry `iteration_count`** (S1): `fix_request` schema no longer includes `iteration_count` — the field was always 0 or 1 in practice because re-failures open a *new* entry. The top-level `cross_domain_iteration_count` is the sole iteration counter. Removed from `meta/SKILL.md`, both producer agent schemas, `rtl-design-orchestrator.md` Behaviour Rule 6 and Write step 5a, the fixture, and CI `REQUIRED_FIELDS`.
- **Seeded `memory/meta/experiences.jsonl`** (S7): two illustrative records added — one `converged` (single-iteration MAC unit fix) and one `escalated` (AXI DMA cap exceeded after 3 iterations).
- **Marketplace version bump** and **README reconciliation** (Cosmetic): `metadata.version` bumped to `1.3.0`; README header updated to "15 plugins · 16 skill files"; `chip-design-meta` added to the Available Plugins table.

---

## [Unreleased] — agent-scope-review branch

### Added
- **Pre-run context** (`## Pre-run Context`) section added to all 13 domain SKILL.md files:
  agents now read `knowledge.md` and `run_state.md` at every invocation point, not only
  at orchestrator session start.
- **Run-state tracking**: all 13 domain SKILL.md files and the PD orchestrator now write
  `memory/<domain>/run_state.md` as the first action before any tool invocation; `last_stage`
  is updated after each stage so wakeup-loop prompts can resume correctly.
- **Per-stage experience writes**: PD orchestrator (and all domain skills) now upsert to
  `experiences.jsonl` after each stage rather than only on session end; partial runs are
  persisted even if the session is interrupted.
- **Optional claude-mem integration**: all 13 domain skills and the memory-keeper skill now
  emit applied fixes to `mcp__plugin_ecc_memory__add_observations` when the MCP tool is
  present; guard clause skips silently when absent so JSONL remains the canonical record.
- **Clock gating opportunity analysis** added to `architecture` SKILL.md
  (`power_area_estimation` stage): classifies each clock domain by activity factor α,
  produces a `clock_power_budget` hand-off table (domain → frequency, α, est. clock power,
  gating class), and enforces a new QoR gate (≥ 70% of register bits in gateable domains).
- **Power intent / ICG insertion rules** added to `rtl-design` SKILL.md (`rtl_coding` stage):
  RTL agent reads `clock_power_budget` from architecture hand-off and inserts ICG cells for
  high/moderate gating domains; enforces `clock_gating_coverage` ≥ 60% QoR gate.
- **Architecture → RTL handoff contract** updated in `docs/MASTER_INDEX.md` to include
  `clock_power_budget` artifact.
- `memory/README.md` updated to document run_state.md, per-stage write semantics, `run_id`
  schema field, and the optional claude-mem index pattern.
- `docs/Architecture_Evaluation_Flow.md` and `docs/RTL_Design_Flow.md` updated to match
  the new clock gating analysis and ICG insertion rules added to the live SKILL.md files.
- OpenROAD MCP config (`mcp-openroad.json`) comment improved to call out the two placeholder
  values that require substitution during installation.

---

## [1.2.0] — 2026-04-14

### Added
- Multiple IDE support: GitHub Copilot, Google Gemini Code Assist, and OpenCode
- `ides/copilot/` — Copilot workspace instructions and per-domain file-glob mapping (`applyto-map.json`)
- `ides/gemini/` — preamble header injected into a generated `GEMINI.md`
- `ides/opencode/` — base OpenCode config template with all 13 chip-design modes
- `install.sh --ide <copilot|gemini|opencode|all>` flag to deploy IDE-specific config into the target project
- `install.ps1 -IDE <copilot|gemini|opencode|all>` equivalent for Windows PowerShell
- CI/CD validation extended to lint IDE config files on every PR

### Changed
- Agents and skills updated with explicit EDA tool usage annotations
- AgentShield CI step removed (no `.claude` directory present in repo)

---

## [1.1.1] — 2026-04-13

### Added
- AgentShield CI check to validate Claude agent files on every PR

### Fixed
- Issues reported after CodeRabbit review pass on the AgentShield integration

---

## [1.1.0] — 2026-04-13

### Added
- Install scripts for all OS: `install.sh` (macOS / Linux / Git Bash) and `install.ps1` (Windows PowerShell)
- `strict: true` set in `marketplace.json` to enforce exact plugin paths

### Changed
- **Breaking restructure:** all 13 agents and skills split from a shared flat directory into isolated per-plugin subdirectories (`plugins/<domain>/agents/` and `plugins/<domain>/skills/`) to eliminate file-system racing conditions when multiple plugins load concurrently
- Each plugin now has its own `.claude-plugin/plugin.json` manifest
- CI/CD updated for the new directory layout
- README updated to document the new structure and remove the prior racing-issue caveat

### Fixed
- Recursion guard added to agent and skill invocation chains
- Agents now read their skill file before executing; skills now spawn the corresponding orchestrator before executing

---

## [1.0.3] — 2026-04-12

### Fixed
- Marketplace recursive-directory bug: strengthened schema checks to enforce path typing and prevent the marketplace registry from resolving into subdirectories recursively

---

## [1.0.2] — 2026-04-12

### Fixed
- Validate CI and `plugin.json` incorrect formatting (follow-up to v1.0.1)

---

## [1.0.1] — 2026-04-12

### Fixed
- Validate CI pipeline failures on initial setup
- `plugin.json` formatting errors flagged by the CI linter
- Minor environment file corrections reported by CodeRabbit
- Removed stray `.claude` settings file from repo root

---

## [1.0.0] — 2026-04-12 — Initial Release

### Added
- 13 Claude Code marketplace plugins covering the complete digital chip design pipeline
- 13 skill files with YAML frontmatter, staged domain rules, QoR metrics, and fix guidance
- 13 orchestrator agent markdown files with stage sequences, loop-back rules, and sign-off criteria
- `.claude-plugin/plugin.json` — Claude Code plugin manifest
- `.claude-plugin/marketplace.json` — marketplace registry for all 13 plugins
- CI validation workflow (GitHub Actions) — validates every PR
- Automated release workflow with tar.gz archive generation

### Domains in v1.0.0
Architecture Evaluation · RTL Design (SystemVerilog) · Functional Verification (UVM) ·
Formal Verification (FPV/LEC) · Logic Synthesis · Design for Test (DFT) ·
Static Timing Analysis (STA) · High-Level Synthesis (HLS) · Physical Design ·
SoC IP Integration · Compiler Toolchain (LLVM) · Embedded Firmware · FPGA Emulation
