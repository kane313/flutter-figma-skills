# Scan Strategy (detailed)

## Figma side

Call in order:

1. `mcp__figma__get_metadata({ fileKey, nodeId })` — verify file access; extract root frame size.
2. `mcp__figma__get_libraries({ fileKey })` — enumerate attached libraries (design systems).
3. `mcp__figma__get_variable_defs({ fileKey })` — get all variables (colors, numbers, strings, booleans) and their values per mode.
4. `mcp__figma__get_design_context({ fileKey, nodeId })` — get the target node's structured representation.

Extract the following:

| Figma concept | Extraction path |
|---|---|
| Color tokens | variables with type `COLOR` |
| Typography | variables with type `FLOAT` under font-related collection + text style definitions in design context |
| Spacing scale | variables with type `FLOAT` in spacing / gap collections |
| Radius scale | variables with type `FLOAT` in radius collection |
| Shadow tokens | effect styles from design context |
| Icon / image assets | nodes with `type: INSTANCE` referencing icon components, or `fills: IMAGE` |
| Component instances | nodes with `type: INSTANCE` + `componentId` |

## Flutter project side

Scan (glob patterns):

| What | Where |
|---|---|
| ThemeData colors | `lib/theme/**/*.dart` → look for `ColorScheme`, `ThemeData`, `MaterialColor` |
| TextTheme | `lib/theme/**/*.dart` → `TextTheme`, `TextStyle` definitions |
| Spacing constants | `lib/theme/**/*.dart`, `lib/constants/**/*.dart` |
| Existing widgets | `lib/widgets/**/*.dart`, `lib/components/**/*.dart` |
| State management | `pubspec.yaml` dependencies (provider / riverpod / bloc / get) |
| Routing | `pubspec.yaml` + `lib/main.dart` / `lib/router.dart` |

## Matching algorithm

For each Figma token:

1. **Exact value match**: Figma `#FF5733` vs any `Color(0xFFFF5733)` in project theme → matched.
2. **Name similarity match**: Figma token `primary/500` vs project `colorScheme.primary` → matched with confidence note.
3. **No match**: token goes to gaps.

Record for each match: Figma ref, project ref, match type (exact / name / manual), confidence (high / medium).

## Component matching

For each Figma component instance (e.g. `Button/Primary/Large`):

1. Search `lib/widgets/**` for files/classes with similar names (`PrimaryButton`, `LargeButton`, `AppButton`).
2. Record match candidates with confidence.
3. If no candidate, mark as "needs new widget" in component-map.
