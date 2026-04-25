# Easing Curve Mapping: Figma → Flutter

## Direct mappings

| Figma easing | Flutter `Curves.` |
|---|---|
| LINEAR | `linear` |
| EASE_IN | `easeIn` |
| EASE_OUT | `easeOut` |
| EASE_IN_OUT | `easeInOut` |
| EASE_IN_QUAD | `easeInQuad` |
| EASE_OUT_QUAD | `easeOutQuad` |
| EASE_IN_CUBIC | `easeInCubic` |
| EASE_OUT_CUBIC | `easeOutCubic` |
| EASE_IN_OUT_CUBIC | `easeInOutCubic` |
| EASE_IN_QUART | `easeInQuart` |
| EASE_OUT_QUART | `easeOutQuart` |
| EASE_IN_OUT_QUART | `easeInOutQuart` |
| EASE_IN_EXPO | `easeInExpo` |
| EASE_OUT_EXPO | `easeOutExpo` |
| EASE_IN_OUT_EXPO | `easeInOutExpo` |
| EASE_IN_CIRC | `easeInCirc` |
| EASE_OUT_CIRC | `easeOutCirc` |
| EASE_IN_OUT_CIRC | `easeInOutCirc` |
| EASE_IN_BACK | `easeInBack` |
| EASE_OUT_BACK | `easeOutBack` |
| EASE_IN_OUT_BACK | `easeInOutBack` |

## Material-specific

Figma sometimes exports `GENTLE` / `QUICK` / `BOUNCY` presets for Material Motion. Map:

| Figma preset | Flutter |
|---|---|
| GENTLE | `Cubic(0.2, 0.0, 0.0, 1.0)` (Material standard) |
| QUICK | `Curves.fastOutSlowIn` |
| BOUNCY | `Curves.elasticOut` (careful: overshoots — use only when designer expects spring) |

## Custom cubic bezier

Figma can export a custom bezier `[x1, y1, x2, y2]`. Flutter: `Cubic(x1, y1, x2, y2)`.

```dart
const myCurve = Cubic(0.25, 0.1, 0.25, 1.0);
```

## Default fallback

If Figma omits or exports an unrecognized curve: default to `Curves.easeOutCubic` (spec default).
Emit a note: "Figma curve unrecognized (<value>), defaulted to easeOutCubic."

## Flutter Curves not in Figma

If the target project has a custom `Curves` helper (e.g. `AppCurves.snap`), the skill should NOT invent mappings to it — use only stock `Curves.` unless the user explicitly asks.
