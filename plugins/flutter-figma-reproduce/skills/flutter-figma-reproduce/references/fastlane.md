# Mode Reference (Safe / Fast / Auto)

This file documents the three execution modes. The mode is set by the `mode` argument or auto-elevated by trigger words (see SKILL.md "Auto-mode trigger detection").

## Fast Mode (A) Rules

## Command

`/flutter-figma-reproduce <url> --mode=fast`

## What's skipped

- Phase 1 checkpoint (widget tree review still runs automatically internally, but no user pause)
- Phase 2 checkpoint (already optional in safe mode)

## What's still mandatory (same as safe mode)

- Phase 0 checkpoint — align table / gaps / component map must be user-approved; the align-table is the contract for all downstream phases
- Phase 3 checkpoint — visual diff + interaction checklist
- Phase 4 scorecard — all 4 layers must pass thresholds

## Failure handling

If Phase 4 scorecard fails in fast mode:

- Do NOT auto-fallback to safe mode
- Report detailed blockers (which layer, which likely phase)
- User decides: re-run a specific phase, accept and move on, or switch to safe mode manually next time

## When to use fast

Appropriate for:
- Experienced developers who trust the pipeline
- Re-running after targeted edits (second run — minimizes friction)
- Bulk mode: reproducing a list of similar screens after the first one was safe-mode reviewed

Inappropriate for:
- First time using this skill on a new project (run safe first to catch project-specific issues)
- Pages with complex prototype animations (Phase 3 warrants attention)
- Pages that will be production-critical (safe mode's checkpoints are cheap insurance)

## Behavioral differences table

| Aspect | Safe | Fast | Auto |
|---|---|---|---|
| Phase 0 checkpoint | Mandatory user `continue` | Mandatory user `continue` | Auto-defaults from gaps.md `[x]` marks + ScreenUtil install `y` |
| Phase 1 checkpoint | Mandatory user review | Auto-advance after internal review gate | Auto-advance |
| Phase 2 checkpoint | Optional (`/verify visual`) | Auto-advance (optional still available) | Auto-advance |
| Phase 3 checkpoint | Mandatory user review + interaction checklist | Mandatory user review + interaction checklist | Interaction checklist marked "auto-deferred"; advance |
| Phase 4 scorecard | Mandatory | Mandatory | Report written + advance (no "accept or revisit?" prompt) |
| Token layer auto-check | Every generation | Every generation | Every generation |
| Structural layer auto-check | Every generation | Every generation | Every generation |
| Visual layer check | Manual trigger OR Phase 3/4 | Manual trigger OR Phase 3/4 | Auto-attempted; skipped with note if Patrol unavailable |
| Quality threshold (scorecard) | Same | Same | Same |
| Final user pause | After every checkpoint | Phase 0/3/4 only | None — single info summary at end (no reply expected) |

Fast and auto modes are TIME optimizations, not QUALITY compromises. The same scorecard thresholds apply.

---

## Auto Mode (C) Rules

## Command

`/flutter-figma-reproduce <url> --mode=auto`

Or any prompt containing trigger phrases listed in SKILL.md "Auto-mode trigger detection" (e.g. "不用再询问", "autopilot", "yolo", "跑完为止").

## What's skipped

- All checkpoint user-pauses (Phase 0, 1, 2, 3, 4)
- All "accept or revisit" prompts
- ScreenUtil install confirmation (assumes `y`)
- Gap resolution edits (uses recommendation defaults from `gaps.md` template)
- Component reuse decisions (uses recommendation defaults from `component-map.md` template)
- Interaction checklist verification (marked `auto-deferred` in verify-report.md)

## What's still enforced

- Token / Structure layer auto-checks at every code-generating phase (same as fast/safe)
- Visual layer pixel-diff (auto-attempted; if Patrol or other infra unavailable, marked `unchecked` with reason and continue)
- Phase 4 scorecard report generation (full report written to disk)
- Stopping conditions in each `references/0N-*.md` (e.g. Figma 403/404, missing pubspec, FIGMA_TOKEN required for asset export) still abort the run with clear errors
- Skill constraints (never modify files outside `.figma-reproduce/<page>/` or `lib/pages/<target>/` without acknowledgement) — auto mode does NOT bypass these

## Mid-run downgrade

If the user interrupts with `wait` / `等等` / `pause` / `暂停` / `让我看看` / `stop` while auto mode is running, immediately:

1. Halt before the next phase advance
2. Downgrade to safe mode for the remaining phases
3. Emit the next CHECKPOINT block as if running safe from the start

## When to use auto

Appropriate for:
- Re-running after fixes when the user has already reviewed the design intent
- Bulk reproduction of a list of similar screens after the first one was safe-mode reviewed
- CI / scripted scenarios (one-shot run, accept all defaults, get report)
- Trusted second runs (`state.json` exists from prior approved run)

Inappropriate for:
- First-time use on a new Figma file (run safe to catch project-specific surprises)
- Pages with complex prototype animations (Phase 3 user review catches motion issues that scorecard can't)
- Pages with significant gaps in `gaps.md` (auto-defaults may pick "inline" when user wanted "add to theme")

## Final summary block

At the end of an auto run, emit ONE summary block (no reply expected):

```
[AUTO RUN COMPLETE]
Page: <target>
Mode: auto (triggered by: explicit | "<trigger phrase>")
Phases: 5/5 completed
Defaults taken:
  - Phase 0 gaps: <count> resolved via recommendation defaults
  - Phase 0 ScreenUtil: <installed | already present>
  - Phase 3 interactions: auto-deferred (no user verification)
Verdict: <PASS | FAIL | INCONCLUSIVE>
Files:
  - lib/pages/<target>/page.dart
  - .figma-reproduce/<target>/verify-report.md
  - .figma-reproduce/<target>/pixel-diff-*.png (if generated)
Next: review verify-report.md for any FAIL/INCONCLUSIVE layers
```
