# Scorecard Checker

Automatic algorithm for the Token and Structure layers of the Phase 4 scorecard (also triggered in Phase 1/2 as incremental checks).

## Token Layer

### Goal
Every "value" in the generated Dart code must trace back to the align-table. Violations are hardcoded values that should have been tokens.

### Algorithm

1. Read `.figma-reproduce/<target>/align-table.md` — extract allowed values into sets:
   - `colors: {Color(0xFF...), ColorScheme.primary, ...}` (both literal and token ref forms)
   - `sizes: {0, 4, 8, 12, 16, 24, 32, ...}` (align-table spacing + common zero/gap values)
   - `radii: {BorderRadius.circular(4), BorderRadius.circular(8), ...}`
   - `shadows: {BoxShadow(offset: Offset(0,1), blur: 2, ...), ...}`
   - font weights / sizes / families

2. Scan target Dart files (`lib/pages/<target>/*.dart`):
   - Literal `Color(0x...)` → error unless in allowed set
   - Literal `BoxShadow(...)` → error unless in allowed set
   - **Bare numeric dimensions** without `.w / .h / .sp / .r` suffix in `EdgeInsets.*`, `SizedBox(width:|height:)`, `BorderRadius.circular`, `Border.all(width:)`, `fontSize:`, `Offset(...)`, `blurRadius:`, `spreadRadius:` → error
   - Hardcoded `EdgeInsets.all(<N>.w)` / similar (with suffix) → warn if `<N>` not in align-table spacing scale (compare the scaled value, not the suffix)
   - Literal font family string → error unless in allowed font families

3. **Exception**: values inside `_mock_data.dart` are exempt (they're data, not UI tokens).

4. **Exception**: placeholder greys (`Color(0xFFE5E5E5)`) in `_skeleton.dart` are allowed during Phase 1.

5. **Exception**: bare `0`, `flex` / `MainAxisAlignment` weight ratios, and divider widths `≤ 1` (e.g. `Divider(thickness: 1)` → emit warning instead of error suggesting `1.h`).

### Output format

```
Token layer: <PASS|FAIL>
Violations:
- lib/pages/X/page.dart:42 — `Color(0xFFABCDEF)` not in align table (closest match: `colorScheme.tertiary` = `Color(0xFFABCDEE)`, off by 1)
- lib/pages/X/page.dart:58 — `EdgeInsets.all(14)` not in align table spacing (nearest: 12 or 16)
```

Each violation includes:
- File + line reference
- The offending literal value
- Suggested replacement (closest token by numeric distance for colors/sizes, or "create new token" if too far)

## Structure Layer

### Goal
Widget tree selection matches Figma node layout intent. Delegated to `flutter-widget-tree-review` skill.

### Algorithm

1. Call `flutter-widget-tree-review` with `dart_file` + `figma_node`
2. Read returned report
3. Report PASS iff all items are PASS or WARN; FAIL iff any item is FAIL

### Output

Direct pass-through from widget-tree-review. The scorecard aggregates multi-file results by taking the worst verdict.

## Visual Layer

Delegated entirely to `figma-flutter-pixel-diff`. Orchestrator passes through threshold (default 5%). The scorecard records:

- Actual diff rate
- Whether the run was stable (triple-capture consistency)
- Heatmap path
- Top-3 diff regions

## Interaction Layer

Human-verified checklist; not algorithmic. Orchestrator presents the checklist from `templates/verify-report.md.tmpl` and waits for user tick.

## Implementation notes for the orchestrator

- Token layer scan: use `grep -n` via Bash for literal hex colors and numeric patterns; then in-Claude comparison against the parsed align-table
- Don't write a Dart AST parser — regex + heuristics is good enough and faster
- Cache the align-table parse in memory during a single Phase 4 run; don't re-read per file
- Report is written to `.figma-reproduce/<target>/verify-report.md` via template substitution, not logged to conversation (conversation gets a summary)
