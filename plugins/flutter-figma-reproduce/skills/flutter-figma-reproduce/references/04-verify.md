# Phase 4: Verify

## Goal
Run full layered scorecard. No code changes. Produces `.figma-reproduce/<target>/verify-report.md`.

## Inputs
- `lib/pages/<target>/page.dart` + co-located files
- `.figma-reproduce/<target>/align-table.md`
- Figma node

## Orchestration

1. Initialize report from `templates/verify-report.md.tmpl`
2. **Token layer**:
   - Run `tools/scorecard-checker.md` Token layer algorithm over `lib/pages/<target>/*.dart`
   - Record violations with `file:line` references
   - Status: PASS if 100% of values traceable to align-table, else FAIL
3. **Structure layer**:
   - Invoke `flutter-widget-tree-review` one final time against the Figma node
   - Status: pass-through from the tool's verdict
4. **Visual layer**:
   - Invoke `figma-flutter-pixel-diff` with `target_route` (derived from page name)
   - Status: PASS if diff rate < 5% (configurable threshold), else FAIL with heatmap + top-3 regions
   - If Patrol not set up in target project, emit clear setup instructions and SKIP this layer (mark as "unchecked"). Verdict still usable from other 3 layers.
5. **Interaction layer**:
   - Present the interaction checklist (loading/empty/error/success render via MockMode toggle, button splash, list refresh, animations play correctly, four-state UI matches Figma)
   - Wait for user to tick each item
6. Aggregate all layer statuses into overall verdict:
   - ALL layers pass → PASS
   - Any layer FAIL → FAIL
   - Any layer "unchecked" → verdict is INCONCLUSIVE with explanation
7. Map failures to likely source phase (write into report's "Failure Mapping" section):
   - Token fail → Phase 2 (style application)
   - Structure fail → Phase 1 (skeleton)
   - Visual fail + regions text-heavy → Phase 2 typography; image-heavy → Phase 2 asset export; layout-wide → Phase 1 skeleton
   - Interaction fail → Phase 3
8. Write complete `verify-report.md` to `.figma-reproduce/<target>/`
9. **Checkpoint mandatory**:
   - Present report summary (verdict + scorecard table + top blockers if any)
   - If PASS → "Accept this page as done? Reply `continue` to complete the run, or specify a Phase number (0-3) to revisit."
   - If FAIL → "Which phase to revisit? Reply with phase number (0-3), or `accept` to override verdict (not recommended — verified issues become silent regressions)."
10. On revisit: return to the specified phase; preserve outputs of unaffected phases; re-run only the target phase; return to Phase 4 when done.

## Stopping conditions

- `pixel-diff` fails Patrol preflight (no device, Patrol not installed) → mark visual layer "unchecked — Patrol unavailable", continue with other 3 layers
- `flutter-widget-tree-review` cannot parse the Dart file → syntax error from previous phase; return directly to the phase that broke it (likely Phase 2) and report
- User requests revisit more than 3 times in one run → pause and ask: "Multiple revisits suggest the Figma design / align table may have deeper issues. Should we step back and review those first?"

## What this phase does NOT do

- Does not modify code (diagnostic only)
- Does not re-run previous phases automatically (user decides on revisit target)
- Does not gate on aesthetic subjective judgment (only algorithmic/checklist layers)
- Does not retry `pixel-diff` if Patrol is flaky — if the tool reports "unstable" three times, escalate to user, don't keep retrying
