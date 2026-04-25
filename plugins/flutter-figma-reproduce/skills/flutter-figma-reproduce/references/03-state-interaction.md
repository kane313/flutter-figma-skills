# Phase 3: State & Interaction

## Goal
Add loading/empty/error/success states, gestures (splash, pull-to-refresh, dismissible), and animations from Figma Prototype.

## Inputs
- `lib/pages/<target>/page.dart` (from Phase 2)
- Figma node (with optional Prototype interactions)

## Orchestration

1. Read `references/failure-modes.md` section "状态/交互类"
2. Infer data type for this page from page name + Figma content hints. If ambiguous, ask user: "What's this page's data type? e.g. `List<Post>`, `UserProfile`, no data".
3. Invoke `flutter-four-states-scaffold`:
   - Generates sealed-class state model + 4 state widgets (loading/empty/error/success)
   - Generates `lib/pages/<target>/_mock_data.dart` with realistic placeholders
   - Refactors `page.dart` to use switch-on-state pattern
4. Invoke `figma-prototype-to-flutter-animation` IF Figma node has `interactions` field:
   - Produces animation snippets in `.figma-reproduce/<target>/animations/*.dart`
   - Present snippets to user; integrate accepted ones into `page.dart` (don't auto-apply all)
5. Add default interactions even if Figma didn't draw them explicitly:
   - `ElevatedButton` / `FilledButton` / `TextButton` → splash is Flutter default, nothing to do
   - Icon buttons / `InkWell` around tappables → add `InkWell` splash if missing
   - Lists → wrap in `RefreshIndicator` with stub `onRefresh` (a one-second delay that resolves)
   - List items with "swipe" affordance in Figma → wrap in `Dismissible`
6. Run Token layer check — new animation/state code must not introduce hardcoded values (animation durations in align-table if possible; otherwise 300ms standard)
7. Run `dart analyze` + `dart format`
8. **Checkpoint mandatory** (both modes):
   - Invoke `figma-flutter-pixel-diff` for a visual baseline check (success state mock)
   - Present interaction checklist from `templates/verify-report.md.tmpl` interaction section
   - Wait for user to tick each item + reply `continue` (or specify what to fix)

## Defensive rules

- All 4 states (loading/empty/error/success) must exist in code even if Figma only drew success — the scaffold handles this
- No default `Curves.linear` — use `Curves.easeOutCubic` as the baseline, or Figma-specified curve
- Default animation duration 300ms if Figma unspecified
- No default `MaterialPageRoute` for page transitions when Figma has prototype transition specified — use `PageRouteBuilder` with matching curve/duration
- Error state MUST include `onRetry` callback (even if wired to a stub)
- Lists default to `RefreshIndicator` + bottom-reach detection (load-more stub)
- `InkWell` splash on all tappable regions — buttons, cards, list items

## Stopping conditions

- `flutter-four-states-scaffold` cannot detect state management + user can't specify → fall back to `none` (plain StatefulWidget) with warning; continue the phase
- Figma has interactions but `figma-prototype-to-flutter-animation` returns "data unavailable" → skip animations with warning; do NOT block the phase (prototype is optional enrichment)
- `pixel-diff` reports > 15% difference at this phase → likely Phase 2 style issue, surface warning but continue to Phase 4 (let scorecard decide whether to revisit)
- User specifies an interaction checklist item doesn't work → return to the root cause (animation snippet? list refresh wiring?); re-generate that specific piece
