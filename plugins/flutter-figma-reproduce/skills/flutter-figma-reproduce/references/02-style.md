# Phase 2: Style

## Goal
Fill colors, typography, spacing, shadows, borders on the skeleton, and export image/icon assets. Merge `_skeleton.dart` → `page.dart`.

## Inputs
- `lib/pages/<target>/_skeleton.dart` (from Phase 1)
- `.figma-reproduce/<target>/align-table.md`
- Figma node (for asset enumeration)

## Orchestration

1. Read `references/failure-modes.md` sections "样式类" and "资源类"
2. **Skeleton-artifact audit (mandatory before styling)**: walk the Phase 1 `_skeleton.dart` widget tree and tag every widget against the Figma node tree. For each widget:
   - Maps to a Figma node? → keep, will be styled in step 3
   - Does NOT map to any Figma node? → it's a skeleton-era guess (e.g. "background panel under content", "spacer to fill gap"); **delete it**, do not "style it later". Common offenders: extra `Positioned` solid-color panels, redundant `SizedBox` spacers, `Container` placeholders.
   - Save the audit list to `.figma-reproduce/<target>/phase2-cleanup.md` so the user can review what was removed.
3. **Assets first**: if target Figma node contains `fills: IMAGE` or icon instances:
   - Invoke `figma-asset-export` — downloads assets via REST API + patches `pubspec.yaml`
   - Verify `flutter pub get` succeeds after
   - If export fails, emit warning but continue with placeholder greys; user can re-run asset export later
   - **For each asset (SVG or PNG/WebP) that will be reused (whether newly exported or already in the project): inspect contents before classifying**. The inspection differs by file type:
     - **SVG**:
       ```bash
       head -c 800 assets/icons/<name>.svg                # peek structure
       grep -cE '<path|<rect|<circle' assets/icons/<name>.svg  # element count
       ```
       - 1-2 paths, simple geometry → pure icon → wrap in styling Container, render at 24-34px inside cell
       - 3+ paths including a rounded-rect outer + inner shapes → **whole component instance** (card + icon, badge + text, etc.) → render at native size, do NOT wrap in additional decoration
       - Contains `<foreignObject>` / `<filter>` → flutter_svg drops these; reproduce blur via Flutter `BackdropFilter` if needed
     - **PNG / WebP exported from a Figma node**: the file is a baked rasterization of whatever the Figma node contained. Inspect the node's metadata subtree BEFORE export:
       ```
       mcp__figma__get_metadata(fileKey, nodeId)  # enumerate children
       ```
       - If the node is a component instance with `<text>` children (e.g. icon + label), the exported PNG will INCLUDE the text baked as pixels. Adding a Flutter `Text` widget for the same label will produce duplicates.
       - **Mitigation**: either (a) export only the icon child node ID (drop the parent), OR (b) skip the Flutter `Text` widget. Decide upfront, do not let the duplicate ship.
       - Common pattern: Figma `产品logo展示` / `第三方登录` / `tab item` style components bundle icon + label. ALWAYS check children before exporting at parent level.
   - Record each asset's classification in `align-table.md` under "Assets" so Phase 2 styling code knows whether to wrap or render-direct (and whether the Flutter side adds a Text label or relies on the baked one).
4. Read `_skeleton.dart`; apply styles from align-table — **all dimensional values use `flutter_screenutil` suffixes** (`.w / .h / .sp / .r`) against `design_baseline` (default 375×812):
   - Colors → `Theme.of(context).colorScheme.X` OR align-table ref
   - Typography → `Theme.of(context).textTheme.X` with `fontSize: <N>.sp`
   - Spacing → `EdgeInsets.symmetric(horizontal: 16.w, vertical: 12.h)`, values drawn from align-table spacing scale
   - Sized boxes / fixed widths / heights → `SizedBox(width: 24.w, height: 24.h)` (use `24.r` for square dimensions that scale isotropically — icons, square avatars)
   - Shadows → `BoxShadow(offset: Offset(0, 1.h), blurRadius: 4.r, spreadRadius: 0, color: ...)` — all numeric components scaled
   - Borders → `Border.all(width: 1.w, ...)` or `BorderRadius.circular(8.r)` from align-table radii
4. Handle images/icons (assets already exported by step 2):
   - `Image.asset('assets/images/<name>.webp')`
   - `SvgPicture.asset('assets/icons/<name>.svg')` (requires `flutter_svg` in pubspec — add if missing)
   - If still missing at time of style pass, keep `ColoredBox(Color(0xFFE5E5E5))` placeholder + emit warning
5. Localize user-visible text: all strings → `AppLocalizations.of(context).<key>`. If i18n not initialized, bootstrap (see section below)
6. Run Token layer check (`tools/scorecard-checker.md`) — no hardcoded values allowed this phase; non-align-table colors are failures
7. Move `_skeleton.dart` → `page.dart`; delete `_skeleton.dart`
8. Run `dart format lib/pages/<target>/` + `dart analyze lib/pages/<target>/`
9. **Checkpoint**: optional. Do NOT pause unless user types `/verify visual`. In that case invoke `figma-flutter-pixel-diff` for a spot check, then advance.

## i18n bootstrap

If project doesn't have `AppLocalizations` (no `lib/l10n/` AND `flutter_localizations` not in pubspec):

1. Add to `pubspec.yaml` `dependencies`:
   - `flutter_localizations: { sdk: flutter }`
   - `intl: ^0.19.0`
2. Add to `pubspec.yaml` `flutter:` section: `generate: true`
3. Create `l10n.yaml` at project root (arb dir, output locale, template arb file)
4. Create `lib/l10n/app_en.arb` and `lib/l10n/app_zh.arb` with string keys derived from the current page (keys named like `pageName_element_purpose`)
5. Run `flutter gen-l10n`
6. Wire `AppLocalizations` into `MaterialApp.localizationsDelegates` and `supportedLocales` in `lib/main.dart`
7. Emit warning: "i18n initialized with placeholder arb entries. Translate before shipping."

## Defensive rules

- **Preserve immersive defaults from Phase 1 skeleton** — never set `AppBar.backgroundColor` to an opaque color unless Phase 1 explicitly opted into the non-immersive override. Transparent + `extendBodyBehindAppBar: true` is the default contract.
- **`systemOverlayStyle` must be aligned to the actual AppBar / top-region color after styling** — if Phase 2 changes the page's effective top color (e.g. fills it with a brand color while keeping it transparent), recompute luminance and update `systemOverlayStyle` to match.
- **`statusBarColor` must be explicit `Colors.transparent`, NOT a Flutter preset (`.dark` / `.light`)** — the presets carry a 25% black scrim on Android that becomes a visible dark wash over light headers. Always emit the full `SystemUiOverlayStyle(statusBarColor: Colors.transparent, statusBarIconBrightness: ..., statusBarBrightness: ...)` literal.
- **Full-bleed backgrounds (header image, top gradient, page-wide color) MUST live OUTSIDE any `Center+SizedBox(width:N)` content constraint** — see `01-skeleton.md` "Full-bleed background layering rule". On wider devices, in-constraint backgrounds expose page-bg side strips.
- **Chinese `Text` inside fixed-width containers (≤ 100w) MUST be wrapped in `FittedBox(fit: BoxFit.scaleDown) + Text(maxLines:1, softWrap:false)`** (or `Flexible+FittedBox` when inside a Flex). Figma's PingFang SC / Yuanti SC are narrower than Android's Noto Sans CJK SC fallback; cross-font kerning drift causes overflow / wrap on real Android. Real iOS PingFang fits without scaling.
- Never use `Color(0x...)` literal — must reference theme or align-table token
- **All dimensional values must use `.w / .h / .sp / .r` suffixes** — bare numbers in `EdgeInsets`, `SizedBox`, `BorderRadius`, `fontSize`, `Offset` are violations. Exceptions: `0` (zero), `flex` / `MainAxisAlignment` weight ratios (not pixels), divider widths ≤ 1. Token-layer scorecard catches these (see `tools/scorecard-checker.md`).
- Font fallback: if designated font not in pubspec, use `GoogleFonts.<name>()` or explicit fallback stack; never silently swap to default
- `BoxShadow` must have offset + blur + spread + color all specified
- Image format preference: WebP; unspecified placeholder → `Color(0xFFE5E5E5)` grey
- **SVG asset usage rule**: render at native size only if the SVG contains the full component (per Phase 0 inspection); render scaled-down (24-34px) inside a styled wrapper Container only if the SVG is a pure icon. Never wrap a "full component" SVG in additional decoration — produces double-card visual with shrunken inner content.
- No hardcoded Chinese (or any non-ASCII UI string) — must go through `AppLocalizations`
- Font weight `Medium` maps to `FontWeight.w500` (not w400)
- Line-height: Figma percentage → Flutter `height` (multiplier, not pixel)
- Letter-spacing: Figma px → Flutter `letterSpacing` px (may be negative)

## Stopping conditions

- `figma-asset-export` fails after retry → ABORT Phase 2, request user fix FIGMA_TOKEN / cwebp / network
- Token layer check has > 5 unresolvable violations → abort, ask user to augment align-table
- `dart analyze` post-format fails → abort, restore `_skeleton.dart` from git (if available) or prior in-memory snapshot
