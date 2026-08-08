# Local Modifications — CHANTXU64/hermes-lcm

Purpose: index fork-only behavior so future upstream merges know which changes are active, reverted, or obsolete, and which feature docs to read.

Repository:

- Fork: https://github.com/CHANTXU64/hermes-lcm.git
- Upstream: https://github.com/stephenschoettler/hermes-lcm

This file is an index and merge guide. It is not a command to preserve every historical fork change forever.

## Merge rules for AI agents

- Match conflict files against `Files` entries before reading feature docs.
- Only open the matched `Feature docs`; do not read every file under `docs/CHANTXU64/`.
- Preserve active behavior only when current code and user decisions still support it.
- Do not revive reverted or obsolete behavior unless the user explicitly asks.
- If this index is incomplete, use git diff/log/blame evidence and report the gap.

## Active modifications

### 1. Per-model threshold-only compaction and whole-request post-compaction target

Status: active

Files:

- `README.md`
- `command.py`
- `compaction.py`
- `config.py`
- `engine.py`
- `tools.py`
- `docs/agent-config-profiles.md`
- `docs/features-overview.md`
- `docs/operator-guide.md`
- `tests/test_active_tool_stubbing.py`
- `tests/test_lcm_command.py`
- `tests/test_lcm_core.py`
- `tests/test_lcm_engine.py`
- `docs/LOCAL_MODIFICATIONS.md`
- `docs/CHANTXU64/threshold-only-compaction/README.md`

Summary:

- Adds model-specific trigger ratios, opt-in cache-stable threshold-only scheduling, and a best-effort whole-provider-request target for bounded threshold full sweeps.

What changed:

- `LCM_THRESHOLD_ONLY_COMPACTION_ENABLED=true` suppresses below-threshold Leaf summarization and DAG condensation while preserving raw ingest, deterministic cleanup, tool-result externalization/stubbing, manual force, and overflow recovery.
- `lcm.model_policies` independently overrides the effective LCM trigger and post-compaction target for matching models using longest-substring matching; both values are recalculated on model switches, omitted targets fall back globally, and omitted triggers continue through the Hermes compatibility map before the global fallback.
- Hermes `compression.model_thresholds` remains a trigger-only compatibility fallback when the matched LCM policy has no trigger field.
- `LCM_POST_COMPACTION_TARGET_RATIO` lets a threshold full sweep continue toward a whole-request token target instead of treating summary-prefix size as the entire active-context target.
- Whole-request accounting preserves fixed Hermes request overhead and a conservative proactive-recall reserve. Interim measurements avoid proactive-recall and externalization side effects; final provider-visible assembly runs once.
- Status, doctor diagnostics, telemetry, documentation, and regression tests cover invalid configuration, reachable and unreachable targets, protected content, fixed overhead, and truthful partial outcomes.

Why it matters:

- The default incremental maintenance path can repeatedly rewrite a long provider-visible prefix before the high-water mark, reducing prompt-cache stability. The fork mode delays summary-producing maintenance until the configured threshold and avoids falsely reporting a post-compaction target as reached when fixed request overhead remains above it.

Merge protection:

- Preserve when: upstream still lacks equivalent threshold-only scheduling, whole-request post-target accounting, or side-effect-free provisional measurement.
- Preserve the per-model policy path while upstream LCM lacks independent trigger and post-target overrides on initial load and model switches.
- Drop when: an upstream implementation has equivalent defaults, safety exemptions, whole-request telemetry, and regression coverage, and the local configuration has been migrated and verified.
- Ask user when: upstream changes full-sweep scheduling, request-token estimation, active assembly, proactive recall, tool externalization, or the configuration names/semantics.

Verification:

```bash
/Users/robot/.hermes/hermes-agent/venv/bin/python -m pytest -q \
  tests/test_lcm_engine.py \
  tests/test_lcm_command.py \
  tests/test_lcm_core.py \
  tests/test_active_tool_stubbing.py \
  -k 'model_policy or model_threshold or threshold_only or post_compaction_target or post_target or threshold_full_sweep or effective_assembly_trigger or live_interceptor_adopts_current_tool_stub_below_compaction_threshold'
ruff check .
/Users/robot/.hermes/hermes-agent/venv/bin/python -m compileall -q \
  compaction.py engine.py config.py command.py tools.py
git diff --check
```

Feature docs: `docs/CHANTXU64/threshold-only-compaction/README.md`

Upstream status: fork-only

### 2. Configurable `lcm_recall` disable switch

Status: active

Files:

- `config.py`
- `engine.py`
- `tools.py`
- `tests/test_tool_contracts.py`
- `docs/LOCAL_MODIFICATIONS.md`

Summary:

- Adds a default-on `LCM_RECALL_ENABLED` switch that can remove `lcm_recall` from the dynamic tool surface and reject direct or internal execution without disabling the rest of LCM.

What changed:

- `LCM_RECALL_ENABLED=false` is parsed into `LCMConfig.recall_enabled`.
- Disabled engines omit `lcm_recall` from `get_tool_schemas()` and reject engine-dispatched calls before the handler runs.
- The `lcm_recall` implementation also rejects direct/internal calls, preventing older cached schemas or indirect callers from bypassing the switch.
- The default remains enabled for upstream-compatible behavior when the environment variable is unset.

Why it matters:

- Operators can retain LCM context management and other retrieval tools while disabling the experimental cross-conversation recall surface.

Merge protection:

- Preserve when: deployments still need to disable only `lcm_recall` without removing the full context-engine toolset.
- Drop when: upstream provides an equivalent default-compatible switch that hides the schema and blocks both dispatch and direct/internal execution.
- Ask user when: upstream changes dynamic tool registration, recall dispatch, or introduces a broader retrieval-disable policy with overlapping semantics.

Verification:

```bash
/Users/robot/.hermes/hermes-agent/venv/bin/python -m pytest -q \
  tests/test_tool_contracts.py \
  tests/test_lcm_recall.py
/Users/robot/.hermes/hermes-agent/venv/bin/python -m ruff check \
  config.py engine.py tools.py tests/test_tool_contracts.py
/Users/robot/.hermes/hermes-agent/venv/bin/python -m py_compile \
  config.py engine.py tools.py tests/test_tool_contracts.py
git diff --check
```

Feature docs: none — the behavior is a small environment flag with complete merge guidance and regression coverage in this index.

Upstream status: fork-only

## Historical / reverted modifications

None.

## Current fork delta checklist

- [x] Active entries have current `Files` paths.
- [x] Each active entry has Summary, Merge protection, Verification, and Feature docs.
- [x] Reverted/obsolete entries are marked and must not be revived automatically.

## Summary statistics

- Active: 2
- Reverted: 0
- Obsolete: 0
