# Insertion Strategy

## Locating the theme file

1. Check `lib/theme/app_theme.dart` (conventional)
2. Else scan `lib/theme/**/*.dart` for files containing `ThemeData(` constructor
3. Else scan `lib/**/*.dart` for the same
4. If multiple candidates, pick the one referenced from `lib/main.dart` (via `MaterialApp(theme: ...)`)

If none found, emit error: "no ThemeData definition located — this skill requires an existing theme, it doesn't bootstrap from scratch."

## Inserting into ColorScheme

Find the `colorScheme:` field in `ThemeData(...)`. It's typically:

```dart
ThemeData(
  colorScheme: ColorScheme.fromSeed(seedColor: Color(0xFF3B82F6)),
  // or explicit:
  colorScheme: ColorScheme.light(
    primary: Color(0xFF3B82F6),
    // ...
  ),
)
```

For `ColorScheme.fromSeed`, don't modify — add a `.copyWith(...)` chain:

```dart
colorScheme: ColorScheme.fromSeed(seedColor: Color(0xFF3B82F6)).copyWith(
  primaryContainer: Color(0xFF60A5FA),  // ← added
),
```

For explicit `ColorScheme.light(...)`, insert the new field inline, respecting existing formatting.

## Inserting a new ThemeExtension

1. Find the last `}` of the theme file's top level
2. Insert a new `class AppSpacing extends ThemeExtension<AppSpacing>` above it
3. Find the `ThemeData(` constructor; insert `extensions: const [AppSpacing(...)],` if not present, or append to the existing list

## Preserving comments

Use `dart format` after insertion, not before. The formatter preserves line comments but may reflow. If the theme file has block comments / documentation, anchor insertions AFTER the comment block, not inside.

## Idempotency

Before inserting, search for the target token name. If already present (user added it manually between runs), skip with a note.

## Rollback

If `dart analyze` fails after insertion:

1. Read the error message
2. If it's a syntax error in the skill-added code → revert the hunk
3. If it's a pre-existing error → report but don't revert
4. If unclear → revert the whole file using `git restore <file>` (only if under git); otherwise restore from pre-edit snapshot the skill kept in memory
