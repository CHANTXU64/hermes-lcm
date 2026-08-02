# Threshold-only compaction with a whole-request post target

## Purpose

This fork feature reduces unnecessary provider-visible prefix rewrites in long, cache-sensitive conversations. It keeps ordinary summary-producing maintenance dormant below the configured high-water mark, then reuses the existing bounded threshold full sweep to compact in one publication cycle.

It also makes the optional post-compaction target describe the estimated whole provider request, including fixed Hermes request overhead, rather than only the LCM message list or summary prefix.

## Difference from upstream

Upstream can perform incremental Leaf maintenance below the global context threshold when raw backlog becomes eligible. It also has threshold full-sweep machinery, but this fork adds an opt-in scheduling mode and whole-request target semantics:

1. Below threshold, threshold-only mode permits ingest and deterministic replay maintenance but blocks Leaf summaries and DAG condensation.
2. At threshold, threshold-only mode activates the existing bounded full sweep.
3. A configured post-compaction ratio drives additional eligible Leaf and same-depth condensation work toward a whole-request token target.
4. Fixed system/tool/schema overhead supplied by Hermes remains part of every target decision.
5. Provisional accounting is side-effect-free; proactive recall and large-output externalization are reserved or measured without repeatedly executing their side effects during each pass.
6. An unreachable target is reported as partial with an explicit reason instead of being labelled completed.
7. LCM supports `lcm.model_policies` so different models can independently override both the trigger and whole-request target ratios.

Both new capabilities are disabled by default, preserving upstream behavior unless explicitly enabled.

## Files

- `compaction.py` — threshold-only gating, full-sweep control, whole-request token accounting, stop reasons, and telemetry.
- `engine.py` — bounded condensation toward the whole-request target and pure/final assembly separation.
- `config.py` — new configuration fields, environment parsing, defaults, and source warnings.
- `command.py` and `tools.py` — status and doctor observability.
- `README.md`, `docs/agent-config-profiles.md`, `docs/features-overview.md`, and `docs/operator-guide.md` — operator-facing configuration and behavior.
- `tests/test_lcm_engine.py` — scheduling, target, accounting, side-effect, failure, and safety regressions.
- `tests/test_active_tool_stubbing.py` — deterministic active-replay maintenance below threshold.
- `tests/test_lcm_command.py` and `tests/test_lcm_core.py` — command/status and configuration coverage.

## Configuration / usage

Enable threshold-only scheduling:

```bash
LCM_THRESHOLD_ONLY_COMPACTION_ENABLED=true
```

Optionally set a model-specific trigger and target in Hermes config. Matching
uses longest-substring selection and is recalculated on each model switch:

```yaml
lcm:
  model_policies:
    deepseek-v4-flash:
      context_threshold: 0.40
      post_compaction_target_ratio: 0.10
```

Optionally set a best-effort whole-request target as a fraction of the effective runtime context length:

```bash
LCM_POST_COMPACTION_TARGET_RATIO=0.35
```

The post-compaction ratio must be greater than zero and lower than the effective trigger ratio. Invalid or malformed supplied values remain disabled and are exposed through configuration-source warnings and `lcm_doctor`.

The two ratios are independent. The model policy above starts the sweep at
400K and sets a 100K estimated whole-provider-request target on a 1M window.
An omitted target falls back to its global LCM value. An omitted trigger first
checks Hermes `compression.model_thresholds`, then falls back globally.
Reaching 100K remains best effort: fixed Hermes request overhead, protected
fresh-tail content, summary size, the 12-pass bound, or the 120-second bound can
produce a truthful `partial` result instead.

For compatibility, Hermes `compression.model_thresholds` remains a
trigger-only fallback when a matched LCM policy does not define
`context_threshold`; it cannot set the post-compaction target.

Relevant safety behavior:

- Raw messages continue to be persisted below threshold.
- Manual force and overflow recovery remain available.
- Sensitive cleanup, tool-call/tool-result pairing, and configured large-output replay handling remain active.
- Full sweeps remain bounded by the existing pass and time limits.
- Fresh-tail and tool-transaction protection can make a target unreachable; telemetry then reports a truthful partial result.
- The target is best effort and does not guarantee a provider cache hit.

## Merge guidance

- Preserve when: upstream can still run summary-producing incremental maintenance below threshold, or its target excludes fixed request overhead or triggers side effects during provisional measurement.
- Drop when: upstream provides equivalent opt-in scheduling, whole-request accounting, truthful partial telemetry, one final side-effectful assembly, and equivalent regression coverage.
- Ask user when: upstream changes the threshold/full-sweep pipeline, active-context assembly, proactive recall, externalization, token estimation, or these configuration variables.

During conflicts, compare the upstream implementation against the behavioral tests before choosing either side. Do not preserve old lines mechanically if upstream has implemented an equivalent or stronger invariant.

## Verification

```bash
/Users/robot/.hermes/hermes-agent/venv/bin/python -m pytest -q \
  tests/test_lcm_engine.py \
  tests/test_lcm_command.py \
  tests/test_lcm_core.py \
  tests/test_active_tool_stubbing.py \
  -k 'threshold_only or post_compaction_target or post_target or threshold_full_sweep or effective_assembly_trigger or live_interceptor_adopts_current_tool_stub_below_compaction_threshold'

ruff check .

/Users/robot/.hermes/hermes-agent/venv/bin/python -m compileall -q \
  compaction.py engine.py config.py command.py tools.py

git diff --check
```

A future upstream merge must also rerun the fixed-overhead and repeated-side-effect regressions in `tests/test_lcm_engine.py`; passing only configuration or status tests is insufficient.

## LOCAL_MODIFICATIONS entry

Corresponding entry in `docs/LOCAL_MODIFICATIONS.md`: `Threshold-only compaction and whole-request post-compaction target`.
