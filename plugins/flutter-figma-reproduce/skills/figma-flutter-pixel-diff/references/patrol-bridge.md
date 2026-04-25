# Patrol Bridge — How to drive the app for screenshot capture

Patrol is a Flutter integration-testing framework that controls the app at the native layer (Android / iOS), which makes it far more stable than pure-Dart screenshot capture for UI-regression scenarios: it can wait on real animations, preload fonts, and take screenshots through the OS rather than the Flutter engine alone.

Docs: https://patrol.leancode.co/

## Prerequisites (one-time project setup)

Add to `pubspec.yaml`:

```yaml
dev_dependencies:
  patrol: ^3.0.0
  integration_test:
    sdk: flutter
```

Then install `patrol_cli`:

```bash
dart pub global activate patrol_cli
```

Verify:

```bash
patrol --version
```

If this is the first run in the project, also run `patrol bootstrap` once to generate the platform scaffolding (`ios/` / `android/` test target hooks). Skip this step silently on subsequent runs — check for `integration_test/` existing as the signal.

## Screenshot capture test template

Generated at `integration_test/_pixel_diff_capture.dart` (this file is owned by the skill; overwrite freely, add to `.gitignore`):

```dart
import 'package:flutter_test/flutter_test.dart';
import 'package:patrol/patrol.dart';
import 'package:<app_package>/main.dart' as app;

void main() {
  patrolTest(
    'pixel-diff capture: <target_route>',
    ($) async {
      // 1. launch
      app.main();
      await $.pumpAndSettle(const Duration(seconds: 3));

      // 2. navigate (adapt to the project router)
      await $.tester.tap(find.byKey(const ValueKey('<nav_entry_key>')));
      await $.pumpAndSettle();

      // 3. preload fonts — force text to re-layout with actual glyphs
      await $.pumpAndSettle(const Duration(milliseconds: 500));

      // 4. triple-capture stability check
      final shot1 = await $.native.takeScreenshot(name: 'capture_1');
      await $.pumpAndSettle(const Duration(milliseconds: 300));
      final shot2 = await $.native.takeScreenshot(name: 'capture_2');
      await $.pumpAndSettle(const Duration(milliseconds: 300));
      final shot3 = await $.native.takeScreenshot(name: 'capture_3');

      // the diff skill picks capture_2 (middle) and verifies shot1 ≈ shot2 ≈ shot3;
      // large drift between shots → report "unstable" verdict
    },
  );
}
```

Replace `<app_package>` / `<target_route>` / `<nav_entry_key>` from the skill inputs. For deep-link navigation, use `await $.native.openApp()` followed by `await $.native.openUrl('<deep_link>')` instead of `tester.tap`.

## Run command

```bash
patrol test -t integration_test/_pixel_diff_capture.dart \
  --device <device_id> \
  --flavor <flavor_if_any>
```

Patrol writes PNGs to `<project>/build/patrol/screenshots/<timestamp>/`. The skill reads the middle capture from there.

## Stability fallbacks

Patrol is more stable than Dart-only capture but still needs these guards:

- **Triple-capture drift check**: if `shot1` vs `shot3` differ by > 1% pixel-wise, the scene has ongoing animation — report "unstable" instead of guessing
- **Font warmup**: on iOS simulator first launch, custom fonts may render as system font in the first frame. Always `pumpAndSettle(500ms)` after navigation before the first capture
- **Crash recovery**: if `patrol test` exits non-zero before capture, propagate the stderr tail into the diff report as `"Patrol test failed: <log>"`
- **Device chrome**: Patrol's `takeScreenshot` captures the full screen including status bar. Normalize by cropping `viewport.w × viewport.h` starting at `(0, status_bar_height)` before diffing

## Viewport alignment with Figma

Figma frame `390 × 844` (iPhone 13) should target a simulator with the same logical size. Use:

```bash
xcrun simctl list devices | grep "iPhone 13"
patrol test --device <iPhone13_UDID> ...
```

If the project targets multiple viewports (phone + tablet), the skill should run the capture once per viewport and produce one diff report per viewport.

## Diff algorithm

Use pixel-wise sRGB delta with luminance weighting:

```python
# Pseudocode
def diff(pixel_a, pixel_b):
    dr = abs(pixel_a.r - pixel_b.r)
    dg = abs(pixel_a.g - pixel_b.g)
    db = abs(pixel_a.b - pixel_b.b)
    # weighted: eye more sensitive to green (BT.601)
    return (dr * 0.299 + dg * 0.587 + db * 0.114) / 255

def is_different(pixel_a, pixel_b, threshold=0.03):
    return diff(pixel_a, pixel_b) > threshold
```

Per-pixel threshold 0.03 accounts for font rendering / simulator scaling noise.

## Region breakdown

Divide image into 6 columns × 10 rows (60 cells).
Per cell: cell diff rate = different pixels / total pixels.
Rank cells by diff rate, take top 3 for the report.

## Cause inference (rule-based)

For each top region, cross-reference Figma design context:

- Region contains text nodes → "text region — check font family, weight, letterSpacing, lineHeight, `height` property"
- Region is solid color (>80% pixels within 1 stdDev) → "color region — check `colorScheme` token binding"
- Region contains image/icon → "asset region — check image file, format (WebP?), resolution (@2x?)"
- Region has high-frequency content (gradient, pattern) → "may be rendering artifact — check device pixel ratio, screen density"
- Region is near edge but has diff → "likely SafeArea or status-bar cropping mismatch"

## Heatmap generation

Output PNG same size as input. Per pixel:

- delta in [0, 0.03] → green (low noise)
- delta in (0.03, 0.10] → yellow (moderate)
- delta in (0.10, 1.0] → red (high)

Save to `pixel-diff-{ts}-heatmap.png` alongside the captures.

## Cleanup

After reporting, the skill may optionally leave `integration_test/_pixel_diff_capture.dart` in place for repeatability (re-run the diff after a code change). Document this in the report so users know to delete it or add to `.gitignore`.
