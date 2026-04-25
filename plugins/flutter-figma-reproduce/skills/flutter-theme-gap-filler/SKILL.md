---
name: flutter-theme-gap-filler
description: "Reads a gaps.md file produced by figma-flutter-align-table and writes the user-approved token gaps (colors, text styles, spacing, radius, shadows) into the Flutter project's lib/theme/ files. Handles ColorScheme extensions, TextTheme additions, and custom theme extensions via ThemeExtension. Use after the Phase 0 align-table checkpoint when the user has decided which gaps to backfill into the project theme. Requires a Flutter project with existing ThemeData definition."
---

# flutter-theme-gap-filler

## When to use

- Invoked by `flutter-figma-reproduce` main skill after Phase 0 checkpoint approves gap backfill
- Standalone: "take this gaps.md and add the approved tokens to my theme"

## Inputs

- `gaps_file` (required) — path to gaps.md (usually `.figma-reproduce/<page>/gaps.md`)
- `flutter_project_path` (optional, default: cwd)
- `theme_file` (optional, auto-detected) — target theme file, usually `lib/theme/app_theme.dart` or first dart file under `lib/theme/` containing `ThemeData`

## Workflow

Read `references/theme-anatomy.md` for Flutter ThemeData structure and field mapping.
Read `references/insertion-strategy.md` for safe code insertion patterns.

1. **Parse gaps.md**: extract entries where the user checked `[x] add to project theme`. Skip entries marked `[x] inline in page`.
2. **Classify each approved gap** by token category:
   - Color → `ColorScheme` extension or `ThemeExtension<AppColors>`
   - Typography → `TextTheme` additions
   - Spacing / radius / shadow → `ThemeExtension<AppSpacing>` (spec recommends extensions over constants)
3. **Find insertion points** in the theme file. Use the insertion strategy rules.
4. **Generate + insert code**. Run `dart format` on the result.
5. **Verify**: run `dart analyze` on the edited file; if any error, revert and report.
6. **Emit summary**: list added tokens + the ThemeExtension class name(s) + example usage snippet.

## Outputs

- Modified `lib/theme/*.dart` files
- Summary in conversation, including example usage:
  ```
  Before: Color(0xFF3B82F6)
  After:  Theme.of(context).colorScheme.primaryContainer
  ```

## Constraints

- **Always format + analyze** — never commit malformed Dart.
- **Use `ThemeExtension<T>`** for tokens that don't have a standard Material slot (custom spacing, brand colors beyond ColorScheme). Don't pollute ColorScheme with arbitrary names.
- **Never rename existing tokens** — only add. Renaming is out of scope.
- **Don't touch non-theme files** — resist the urge to "fix" usages in page files; that's the caller's job.
