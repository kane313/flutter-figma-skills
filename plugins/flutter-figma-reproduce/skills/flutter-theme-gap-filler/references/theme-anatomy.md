# Flutter ThemeData Anatomy

## Standard slots (prefer these)

| Token category | ThemeData slot | Accessed via |
|---|---|---|
| Primary / surface / error colors | `ColorScheme` | `Theme.of(context).colorScheme.primary` |
| Body text, headlines, labels | `TextTheme` | `Theme.of(context).textTheme.bodyLarge` |
| Button styles | `ElevatedButtonThemeData` etc. | `Theme.of(context).elevatedButtonTheme` |
| Icon color / size | `IconThemeData` | `Theme.of(context).iconTheme` |
| Divider color | `DividerThemeData` | `Theme.of(context).dividerTheme` |

Standard slots are preferred — Flutter widgets pick them up automatically.

## Custom tokens → ThemeExtension

For design tokens without a standard Material slot (custom brand colors, spacing scale, custom shadows), use `ThemeExtension<T>`:

```dart
@immutable
class AppSpacing extends ThemeExtension<AppSpacing> {
  const AppSpacing({
    required this.xs,
    required this.sm,
    required this.md,
    required this.lg,
    required this.xl,
  });

  final double xs, sm, md, lg, xl;

  @override
  AppSpacing copyWith({double? xs, double? sm, double? md, double? lg, double? xl}) =>
      AppSpacing(
        xs: xs ?? this.xs,
        sm: sm ?? this.sm,
        md: md ?? this.md,
        lg: lg ?? this.lg,
        xl: xl ?? this.xl,
      );

  @override
  AppSpacing lerp(covariant ThemeExtension<AppSpacing>? other, double t) {
    if (other is! AppSpacing) return this;
    return AppSpacing(
      xs: lerpDouble(xs, other.xs, t)!,
      sm: lerpDouble(sm, other.sm, t)!,
      md: lerpDouble(md, other.md, t)!,
      lg: lerpDouble(lg, other.lg, t)!,
      xl: lerpDouble(xl, other.xl, t)!,
    );
  }
}
```

Register in ThemeData:

```dart
ThemeData(
  extensions: const [
    AppSpacing(xs: 4, sm: 8, md: 16, lg: 24, xl: 32),
  ],
)
```

Access: `Theme.of(context).extension<AppSpacing>()!.md`.

## When ColorScheme doesn't fit

ColorScheme has fixed slot names (primary, secondary, tertiary, surface, etc.). If Figma defines `brand/warning`, `brand/success` beyond that, use `AppBrandColors extends ThemeExtension<AppBrandColors>` rather than overloading ColorScheme.

## Typography additions

If Figma defines `display-xl` that's larger than Material's `displayLarge`, either:
- Option A: copy into `textTheme.displayLarge` (lossy — replaces Material)
- Option B: add to `AppTypography extends ThemeExtension<AppTypography>` (non-destructive, recommended)

Default to Option B.
