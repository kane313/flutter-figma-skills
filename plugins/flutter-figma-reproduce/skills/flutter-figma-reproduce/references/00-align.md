# Phase 0: Align

## Goal
Produce align table + gaps list + component map, then get user approval on gap resolution.

## Orchestration

1. **Preflight** — verify Figma MCP, Flutter project (`pubspec.yaml` + `flutter:` dep), optionally FIGMA_TOKEN if Phase 2 will need asset export. Also detect `flutter_screenutil` in pubspec — if missing, mark for the Phase 0 checkpoint prompt (handled in the ScreenUtil bootstrap section below).
2. **Derive run config**:
   - Resolve `target`: if omitted, fetch `get_metadata`, extract top node name, snake_case it (pinyin for Chinese).
   - Write `.figma-reproduce/<target>/config.md` from `templates/config.md.tmpl`.
3. **Invoke `figma-flutter-align-table`** with `figma_url` + `flutter_project_path`. It produces:
   - `.figma-reproduce/<target>/align-table.md`
   - `.figma-reproduce/<target>/gaps.md`
   - `.figma-reproduce/<target>/component-map.md`
4. **Check Figma file tokenization**: if align-table's "Stats" line shows `< 50%` matched AND `get_libraries` returned empty, emit a warning: "Figma file appears untokenized — align table is built from raw values; gaps will be large."
5. **Emit CHECKPOINT**: "Phase 0 align complete. X tokens matched, Y gaps, Z component candidates. Review the three files and:
   - For each gap: mark `[x] add to project theme` or `[x] inline in page`
   - For each component candidate: mark `[x] reuse <existing>` or `[x] new widget` or `[x] inline`
   - Reply `continue` when done."
6. **Wait for `continue`** (or variations listed in SKILL.md).
7. **Post-approval**: if any gap is marked "add to project theme", invoke `flutter-theme-gap-filler` with `gaps_file=.figma-reproduce/<target>/gaps.md`. Wait for it to finish. Run `dart analyze lib/theme/` to catch issues.
8. **Write state.json** snapshot (Figma node hash map) for second-run support.

## ScreenUtil bootstrap

The skill standardizes responsive sizing on `flutter_screenutil` against `design_baseline` (default `375x812`). All dimensional values across generated code use `.w / .h / .sp / .r` suffixes — Phase 2 enforces this via the Token-layer scorecard.

If `flutter_screenutil` is absent from `pubspec.yaml`:

1. At the Phase 0 CHECKPOINT, prompt the user: "flutter_screenutil not detected. Add `flutter_screenutil: ^5.9.0` to pubspec.yaml and wrap MaterialApp in ScreenUtilInit? [y/n]"
2. If `y` (default):
   - Append to pubspec.yaml dependencies
   - Run `flutter pub get`
   - Patch `lib/main.dart` — wrap the existing `MaterialApp(...)` in `ScreenUtilInit(designSize: Size(375, 812), minTextAdapt: true, splitScreenMode: true, builder: (_, child) => MaterialApp(...))`
   - Record `design_baseline=375x812` in `config.md`
3. If `n`:
   - Abort the skill run with note: "the skill requires flutter_screenutil for Phase 2 dimensional scaling; user opted out, cannot proceed"
   - User can re-run later after manual setup, OR override with `--design_baseline=<custom>` and provide their own scaling helper

If `flutter_screenutil` is already present: skip bootstrap; just record the baseline in `config.md`.

## Second run

When `.figma-reproduce/<target>/state.json` already exists:

1. Read old state.json
2. Re-fetch Figma node + compute new hash map
3. Produce a "changed node list": nodes whose hash differs
4. Read the existing generated Dart code; look for `// @figma-reproduce:keep` comment markers — any block so marked is preserved
5. Show the user the change list + what will be regenerated; checkpoint waits for approval
6. After approval, proceed to Phase 1 but with the "targeted regeneration" flag — subsequent phases operate only on changed subtrees

If the diff produces > 50% of the tree changed, fall back to full regen with a warning — targeted diff becomes more fragile than helpful at that scale.

## Bare project handling

If `lib/theme/` is absent AND no `ThemeData(` exists anywhere in `lib/`, Phase 0 enters "bare mode":

- `figma-flutter-align-table` will report nearly all tokens as gaps (expected)
- Default all gap resolutions to "add to project theme"
- `flutter-theme-gap-filler` will bootstrap `lib/theme/app_theme.dart` with a full ColorScheme + TextTheme + extensions derived from the align table
- This is deliberate: reproducing a designed page into a bare project IS a "bootstrap project theme" task

## Stopping conditions

Abort Phase 0 and emit clear error if:

- Figma MCP returns 403/404 on target node → likely file access issue
- No `pubspec.yaml` in workdir → wrong directory
- User replies anything other than approval that the skill cannot interpret after 3 clarification rounds → ask user to handle manually
