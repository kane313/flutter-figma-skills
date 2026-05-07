# Common AI Widget Tree Misreads

Reference this when emitting issues — each entry has a short name the review output can cite.

## "Stack misread"

**Pattern:** Figma `layoutMode: NONE` with children not overlapping (contiguous y or x offsets) → AI wrongly emits `Stack`.

**Correct interpretation:** Children are in a list, not layered. Use Row or Column.

**Heuristic to detect:** compute child bounding boxes; if no overlap, flag.

## "spaceBetween with 2 children"

**Pattern:** `Row(mainAxisAlignment: MainAxisAlignment.spaceBetween, children: [left, right])`.

**Idiomatic fix:** `Row(children: [left, Spacer(), right])` — clearer intent, easier to extend.

## "padding / gap reversed"

**Pattern:** Figma `itemSpacing: 8, paddingLeft: 16` → AI emits `Padding(padding: EdgeInsets.all(8), child: Row(children: [..., SizedBox(width: 16), ...]))`.

**Correct:** Padding uses the Figma padding value; gap between children uses itemSpacing.

## "Unbounded ListView"

**Pattern:** `Column(children: [Header(), ListView(...)])` inside a full-screen page.

**Symptom:** Renders fine in scrolling page but throws "RenderFlex children have non-zero flex but incoming height constraints are unbounded" in some layouts.

**Fix:** Wrap ListView in `Expanded` if the Column is the Scaffold body; or use `shrinkWrap: true` + `physics: NeverScrollableScrollPhysics()` for short lists.

## "Nested SingleChildScrollView / ListView"

**Pattern:** `SingleChildScrollView(child: ListView(...))`.

**Why wrong:** ListView is itself a scroll view; double-scrolling is always a bug.

**Fix:** Drop the outer SingleChildScrollView; use ListView only.

## "Missing SafeArea"

**Pattern:** `Scaffold(body: Column(children: [...]))` on a page whose Figma top is at device chrome (status bar y=0).

**Symptom:** Content clipped under status bar / notch.

**Fix:** `Scaffold(body: SafeArea(child: Column(...)))`.

## "Flexible outside Flex"

**Pattern:** `Container(child: Flexible(child: Text(...)))`.

**Error at runtime:** "Incorrect use of ParentDataWidget".

**Fix:** Remove Flexible or wrap parent in Row/Column/Flex.

## "Hardcoded MediaQuery layout"

**Pattern:** `SizedBox(width: MediaQuery.of(context).size.width * 0.8, ...)` as a primary layout mechanism.

**Why warn:** Brittle to orientation change, poor reuse.

**Fix:** Use `LayoutBuilder` + constraint math, or `FractionallySizedBox(widthFactor: 0.8)`.

## "MaterialApp inside page"

**Pattern:** Page widget wraps its own `MaterialApp(...)`.

**Why wrong:** MaterialApp is app-root; nested MaterialApps silently break theme inheritance.

**Fix:** Page widget should be just a `Scaffold` or custom widget, not re-wrapped.

## "Text wrap inflates fixed-height cell"

**Pattern:** A grid / list cell with FIXED height (`SliverGridDelegateWithFixedCrossAxisCount.mainAxisExtent`, `childAspectRatio`, an explicit `SizedBox(height: ...)`, or any other rigid parent constraint) contains a `Column` whose last child is a plain `Text` label. When the text's natural width exceeds the cell's available width, Flutter silently wraps it to 2+ lines, the Column's intrinsic height grows past the cell budget, and you get `RenderFlex overflowed by N pixels on the bottom`.

**Why this is the dominant cause of "tiny vertical overflow" (5–15 px) in Figma-reproduced grid UIs:**
- CJK labels are deceptively wide: each Chinese / Japanese / Korean character renders at ~1em. A 5-char label like `老照片修复` at `11.sp` ≈ 55 px wide, and a typical 4-column phone grid leaves ~50–55 px per cell. The single px tipping point silently flips a 1-line label into a 2-line label.
- The Figma source frequently has `textAutoResize: HEIGHT` (height grows with content) inside an Auto Layout that has `primaryAxisSizingMode: FIXED`. AI codegen tends to copy the FIXED parent dimension verbatim while leaving the Text unconstrained, so the contradiction only blows up at runtime — not in the screenshot the AI was checking against.
- Same root cause across grids, horizontal chip rows with `IntrinsicHeight`, BottomNavigationBar items, segmented controls, badge-with-counter buttons, etc.

**Heuristic to detect during review:**
1. Walk to every `GridView*` / fixed-height `SizedBox` / fixed-`childAspectRatio` parent.
2. For each such parent, find descendant `Text` widgets that are NOT already constrained by `maxLines: 1` + (`softWrap: false` OR a `FittedBox(scaleDown)` wrapper OR `overflow: TextOverflow.ellipsis`).
3. Cross-reference Figma: if the Figma text node has `textAutoResize: HEIGHT` or `WIDTH_AND_HEIGHT` while the parent frame is FIXED → flag.
4. Pay special attention when the project ships CJK strings (check `pubspec.yaml` for `intl` / `flutter_localizations`, or the codebase for any Chinese / Japanese / Korean string literal). CJK labels are the worst offenders.

**Fix — default (preserves uniform font size across all cells, REQUIRED for grids):**

```dart
Text(
  label,
  style: AppTextStyles.gridLabel,
  textAlign: TextAlign.center,
  maxLines: 1,
  softWrap: false,
  overflow: TextOverflow.visible,
)
```

Why this combo:
- `maxLines: 1` + `softWrap: false` — guarantees exactly one line of vertical real estate, so the Column never gets pushed past its parent budget. This alone fixes the overflow.
- `overflow: TextOverflow.visible` — when natural text width slightly exceeds the cell, glyphs render past the cell's right/left edges into the inter-cell gap. For grids with non-zero `crossAxisSpacing` (the typical case), this is visually invisible — a 5-char CJK label needs ~2 px of bleed at each end on a 4-column phone grid, and the gap is already 8–36 px. **No font scaling, no ellipsis, no font inconsistency between neighboring cells.**

**Critical: do NOT use `FittedBox(BoxFit.scaleDown)` in grids / repeating-cell layouts.** It scales glyphs only on the cells whose label exceeds width, leaving you with visually smaller text on `老照片修复` while neighboring `卡通头像` / `换发型` render at full size — the inconsistency is immediately obvious to users and breaks design fidelity. `FittedBox(scaleDown)` is acceptable ONLY when the cell is solitary (no adjacent siblings to compare font size against), e.g. a single hero badge above a button.

**Acceptable alternatives in priority order (when `overflow: visible` causes too much horizontal bleed — rare):**
1. **Reduce `crossAxisSpacing` / horizontal padding** to widen each cell. Recompute: `cellWidth = (parentWidth - 2 × hPadding - (n-1) × spacing) / n`. Aim for `cellWidth ≥ longestLabelChars × fontSize × 1.05`. This keeps font size and design proportions; only the gap shrinks slightly.
2. **Increase `mainAxisExtent` / `childAspectRatio` to allow 2 lines** — only when the Figma actually intends a 2-line label (check `textAutoResize: HEIGHT` and the design's drawn height). Otherwise it inflates the design.
3. **`maxLines: 1, overflow: TextOverflow.ellipsis`** — when the design explicitly accepts truncation. Visually unacceptable for short CJK labels (`老照片修...` looks broken); use only for English copy or genuinely long strings.

**Anti-patterns (do NOT "fix" this way):**
- `FittedBox(BoxFit.scaleDown)` in grids / lists / any layout with sibling cells — see above; breaks visual consistency between cells.
- `mainAxisSize: MainAxisSize.max` on the Column — doesn't change content height, just hides the warning location.
- Wrapping the entire cell in `SingleChildScrollView` — turns a layout bug into a UX bug (cell becomes scrollable).
- Removing the Text widget's `style.height` globally without checking other usages — mutates unrelated screens.
- Reducing `style.fontSize` for the offending label only — same problem as FittedBox: visual inconsistency.

## "Stack default-clips negative-Positioned children"

**Pattern:** A `Stack` contains a `Positioned(top: <negative>, ...)` or `Positioned(bottom: <negative>, ...)` (or `left/right` negative) child intended to visually overhang the Stack's bounding box — typical for avatar overlays dipping below a thumbnail, badges floating above a card, ribbon corners, etc. The visual is broken: only the part of the Positioned child INSIDE the Stack's box renders; the overhanging portion is invisible. The user reports "the avatar is half cut off" / "the badge isn't showing fully" / "the spacing below the image looks too large".

**Why this is silent at codegen time:**
- Flutter's `Stack` defaults to `clipBehavior: Clip.hardEdge`. Anything painting outside the Stack's layout box gets clipped.
- AI translating Figma to Flutter copies the `Positioned` offsets verbatim from Auto Layout absolute positions — including negative offsets — but doesn't think about clipping, because Figma has no equivalent concept (Figma frames don't clip overflowing children unless `clipsContent: true`, and even then it's per-frame, not the default).
- The codegen often correctly adds a sibling `SizedBox(height: overhang)` below the Stack to make room for the dipping child, then sets the parent budget large enough to fit. The math works out, but the visible result is still broken because the Positioned child itself is clipped.
- Static screenshot comparison (e.g. pixel-diff) can also miss this because the avatar-clipping pattern produces a symmetric "half avatar visible" result that may render correctly in low-fidelity captures or be obscured by content.

**Heuristic to detect during review:**
1. For every `Stack` in the file, list its children.
2. Flag any `Positioned` child with at least one negative numeric offset (`top: -22.h`, `bottom: -22.h`, `left: -10.w`, `right: -8.r`, etc.) UNLESS the enclosing Stack has `clipBehavior: Clip.none`.
3. Same flag for `Transform.translate` children with negative `offset` that push them outside Stack bounds.
4. Cross-reference Figma: if the Figma node has a child whose absolute bounding box extends outside the parent frame's bounding box, the Flutter Stack reproducing it MUST use `Clip.none`.

**Fix:**

```dart
Stack(
  clipBehavior: Clip.none,
  children: [
    // ... base content ...
    Positioned(
      left: 5.w,
      bottom: -22.h,  // overhangs below Stack — now renders fully
      child: Avatar(...),
    ),
  ],
)
```

**Important: also verify the parent's height budget covers the overhang region.** Setting `Clip.none` lets the child PAINT outside Stack bounds, but Stack's LAYOUT size doesn't grow — the parent must still leave physical space for the overhang. Typically: a sibling `SizedBox(height: overhang + visualGap)` between the Stack and the next child, AND the outermost fixed-height container budget must include that spacer. See "Outer container height under-budgeted for child stack" for the budget arithmetic.

**Acceptable alternatives (rare):**
- Use `OverflowBox` to allow the child to render outside its parent constraints — useful when you want the layout to behave as if the child had zero size at the overhang location. More flexible than Stack but easy to misuse.
- Reorder the layout to avoid overhang — e.g. wrap the image in a `Container` with explicit padding equal to the overhang, then place the avatar with a non-negative `Positioned`. Works but distorts the design's coordinate system.

**Anti-patterns:**
- Wrapping the entire Stack in `ClipRect(clipBehavior: Clip.none)` — `ClipRect` is a clipping widget; `Clip.none` on it disables clipping but the inner Stack still clips on its own. Fix the Stack directly.
- Inflating the Stack's size by adding an invisible `SizedBox` child to extend its bounds — works visually but corrupts the Stack's reported size for downstream layout calculations.
- Replacing Stack with `Column + Container(transform: ...)` — overcomplicated; the Positioned + Clip.none idiom is exactly what Stack is for.

**Symptoms users report (translation guide):**
- "image not showing fully" / "图片没展示全" — usually means the overlay (avatar/badge) is partially clipped.
- "spacing between image and text is too large" / "图文间距过大" — the layout reserves space for the overhang, but with the overhang clipped, that reserved area appears empty.
- "the icon is cut off at the corner" — corner badge with negative left/top/right offset.

These two symptoms appearing together (clipped overlay + apparent over-spacing below) is a near-certain fingerprint of this pattern.

## "Outer container height under-budgeted for child stack"

**Pattern:** A horizontal `ListView` / `Wrap` / fixed `SizedBox(height: ...)` wraps a card layout whose `Column` children sum to MORE than the parent budget — even when each child renders correctly at its design size and the text is single-line. Reported overflow is small-to-medium (5–25 px), and unlike "Text wrap inflates fixed-height cell", the overflow persists even with `maxLines: 1` + `overflow: visible` on every Text. The math simply doesn't add up: image height + spacers + caption line-box height > outer height.

**Why this slips past Phase-1 codegen:**
- AI reproducing Figma copies the outer frame's height verbatim from the design (`SizedBox(height: 161.h)` because the Figma frame is 161 tall) AND copies each child's height verbatim — but in Figma the frame's height is computed from Auto Layout summing children, not asserted. When the child rules in Flutter produce slightly more pixels (font-leading distribution, line-height multipliers translating to taller line boxes than the design's `lineHeight` asserted), the parent's verbatim-copied height becomes too small.
- Especially common when one child is an image overlay using `Stack` + `Positioned(bottom: -22.h)` for an avatar that visually dips below the image. The codegen often translates that overhang into a 28.h spacer (= 22 overhang + 6 gap) below the Stack, but Stack itself doesn't grow to accommodate the negative-positioned avatar — the parent budget must absorb both the Stack height AND the spacer AND the caption line-box, which the design's intrinsic-sum frame already implicitly did.
- The caption's `TextStyle.height` (line-height multiplier) is the silent culprit: a Figma node with `fontSize: 11, lineHeight: 22` translates to `height: 22/11 = 2.0`. Flutter renders this as a 22-px line box for a single-line caption — twice the glyph height. AI codegen rarely accounts for this when summing children to verify budget.

**Heuristic to detect during review:**
1. Find every `SizedBox(height: ...)` / `Container(height: ...)` / horizontal `ListView` wrapping a `Column` of stacked content.
2. Sum the declared heights of each Column child: image heights, `SizedBox(height: ...)` spacers, and for each Text child compute `fontSize × style.height × maxLines` (the line-box height).
3. Compare to the parent's height budget. If `sum > budget`, flag — the layout is structurally infeasible regardless of how the Text is configured.
4. Pay special attention when one child is a `Stack` with a `Positioned(bottom: <negative>)` child — the Stack's intrinsic height is just the non-positioned child, so the visual overhang must be paid for explicitly via spacers or by extending the parent budget.
5. Pay special attention when a Text uses a TextStyle with `height` ≥ 1.6 — the line box is much taller than the visible glyph, and the difference is easily overlooked.

**Fix — extend the parent budget to match the actual content sum:**

The fix is structural, not text-related. DO NOT confuse this with "Text wrap inflates fixed-height cell" — `overflow: visible` on the Text won't help here because the overflow comes from the line BOX height, not from horizontal wrapping.

```dart
// BEFORE — outer budget structurally too small
SizedBox(
  height: hasAvatar ? 161.h : 158.h,  // = image(136) + spacer(28/8) + caption-box(?) — under-budgeted
  child: ListView.builder(...),
)

// AFTER — outer budget = sum of declared child heights + small slack
SizedBox(
  height: hasAvatar ? 173.h : 170.h,  // image(136) + spacer(28/8) + caption(22) + ~6 slack
  child: ListView.builder(...),
)
```

Compute the new budget as: `imageHeight + spacers + (fontSize × textStyle.height × maxLines) + ~5h slack`. The slack absorbs sub-pixel rounding and OS-level font padding; without it, devices with slight scaling differences will still bleed.

**Acceptable alternatives in priority order:**
1. **Bump the parent height** (preferred, above) — preserves all design proportions and font sizes; only the section gets a bit taller, which in a vertically-scrolling page is visually invisible.
2. **Reduce the caption's line-height multiplier** (`style.height` from 2.0 → 1.4) when the design clearly used `lineHeight` only for vertical spacing, not for an intentional line box. The visible glyph stays at the design's `fontSize` — uniform across the app. Only do this if the TextStyle isn't shared with screens that need the original line box.
3. **Reduce a non-critical spacer** by the overflow amount — only when the spacer was decorative slack, not load-bearing (e.g. avatar overhang spacer is load-bearing; never shrink it).
4. **Shrink the image height** — last resort; visibly changes the design.

**Anti-patterns (do NOT "fix" this way):**
- Wrapping the Column in `SingleChildScrollView` — turns the card into a vertically scrollable cell, which is broken UX.
- Setting `mainAxisSize: MainAxisSize.min` on the Column inside a fixed-height parent — the parent still gives a tight constraint; this only reorders where the overflow appears, not whether it overflows.
- `Expanded(child: Text(...))` to consume remaining space — distorts the caption's line-box and clips visible glyphs when budget is too tight.
- Reducing `fontSize` on the caption — see "Text wrap inflates fixed-height cell" anti-patterns; visual inconsistency.

**How to distinguish this from "Text wrap inflates fixed-height cell":**
- If the Text already has `maxLines: 1, softWrap: false, overflow: visible` AND overflow still happens → THIS pattern.
- If multiple sibling cells overflow with similar but different short captions (e.g. all 3-5 char CJK) → THIS pattern (it's not text-width-dependent).
- If the overflow amount is roughly constant across cells regardless of caption text → THIS pattern.
- If only the cell with the longest caption overflows → "Text wrap inflates fixed-height cell".

**Diagnosis protocol — back-compute the screenutil factor before sizing the fix:**

A common mistake when fixing this pattern: the parent's height is conditional (`hasAvatar ? 161.h : 158.h`), the overflow report only shows physical px (e.g. `BoxConstraints(w=107.6, h=162.9)`), and the AI assumes "162.9 is close to 161 × 1.012, must be the avatar branch" and bumps that branch by the overflow amount. Wrong — `162.9` could equally be `158 × 1.031` (no-avatar branch on a different device), and the avatar branch may need a much larger bump. Two devices with different ScreenUtil factors will land in different branches of the conditional.

Procedure to size the fix correctly:

1. **Pin the screenutil factor** by back-computing from a value you KNOW: take a fixed-width child like `SizedBox(width: 104.w)` whose physical width is reported (`w=107.6`) → `wFactor = 107.6 / 104 = 1.0346`. Assume `hFactor ≈ wFactor` unless ScreenUtil's `splitScreenMode` or `minTextAdapt` is configured otherwise (it's `wFactor` in many Flutter projects' default config).
2. **Identify which conditional branch** is reporting by computing `reportedH / hFactor` against EVERY possible logical value. For `h=162.9, hFactor≈1.031`: `162.9 / 1.031 = 158.0` → no-avatar branch (NOT 161 / avatar). Check both candidates; don't assume.
3. **Compute target budget per branch independently**: `budget = sum_of_child_heights + (fontSize × style.height × maxLines) + 6 logical px slack`. Each branch may need a different bump magnitude — don't apply a uniform delta.
4. **Re-verify after the fix** by simulating the math for the OTHER branch — fixing the reported branch only, while leaving the other under-budgeted, just defers the overflow until that branch is exercised.

Example mis-fix from a real Figma reproduction:
- Original: `hasAvatar ? 161.h : 158.h`. Report: `h=162.9, overflow 8.6`.
- Wrong inference: "162.9 ≈ 161, so avatar branch overflows by 8.6, bump avatar to 173.h". Bumped both branches by 12 to "be safe".
- Reality: `162.9 / 1.031 = 158` → it was the no-avatar branch. The avatar branch was actually under-budgeted by ~25 logical (136+28+22=186 vs 161). Bumping it to 173 still left −13 underbudget; the next test surfaced an `overflow by 14 px` on the avatar case.
- Correct fix: `hasAvatar ? 192.h : 172.h` (each branch sized to its own content sum + slack).

**Where this most often hits in Figma-reproduced UIs:**
- Home page icon grid (4-column, square icons + label)
- Bottom sheet "more features" / "all categories" picker grid
- Tab bars with text + icon stacks under a fixed-height AppBar
- Story / avatar rails with CJK names below circular avatars
- Bottom nav items with labels under icons (these usually use `BottomNavigationBar` which handles it, but custom ones don't)
