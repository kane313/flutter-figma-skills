# Widget Tree Review Checklist

For each widget in the tree, walk through:

## Container / Layout

- [ ] **Root widget matches Figma frame semantics**: Scaffold for screens, custom widget for components
- [ ] **SafeArea present at top-level** if the Figma frame is full-screen with device chrome (top inset > 0)
- [ ] **Material / CupertinoApp not nested inside pages** (common AI mistake — wraps page in another MaterialApp)

## Row / Column

- [ ] **Direction matches Figma `layoutMode`**: HORIZONTAL → Row, VERTICAL → Column
- [ ] **mainAxisAlignment matches Figma `primaryAxisAlignItems`**
  - MIN / MAX / CENTER / SPACE_BETWEEN → start / end / center / spaceBetween
  - Two-child SPACE_BETWEEN → prefer `[child, Spacer(), child]` over `mainAxisAlignment: spaceBetween`
- [ ] **crossAxisAlignment matches Figma `counterAxisAlignItems`**
  - STRETCH → `crossAxisAlignment: stretch`
- [ ] **mainAxisSize**: if Figma `primaryAxisSizingMode: AUTO`, expect `mainAxisSize: MainAxisSize.min`
- [ ] **Gap as SizedBox, not mainAxisAlignment hack** — see `autolayout-to-flex` rules
- [ ] **Padding vs gap not reversed**

## Stack

- [ ] **Only used when Figma children overlap** — check at least 2 children have overlapping bounds
- [ ] **Nesting depth ≤ 2** — deeper Stack usually indicates misread
- [ ] **Each child uses `Positioned` or `Align`** — bare children in Stack center by default, often unintentional
- [ ] **`clipBehavior: Clip.none` set whenever any descendant `Positioned` has a negative offset** (`top: -N`, `bottom: -N`, `left: -N`, or `right: -N`) — without this, the overhanging part of the child is silently clipped. Same applies to negative `Transform.translate` offsets. See common-misreads.md § "Stack default-clips negative-Positioned children".
- [ ] **Sibling spacer below Stack accounts for overhang** when a child has `bottom: -N` — there must be a `SizedBox(height: ≥ N + visualGap)` between the Stack and the next sibling, AND the outermost fixed-height parent must budget for it.

## Scroll

- [ ] **ListView not wrapped by SingleChildScrollView** — unbounded height error
- [ ] **ListView inside Row/Column has bounded height** — via `Expanded`, `SizedBox`, or `shrinkWrap: true`
- [ ] **shrinkWrap: true only when content is short** — large lists need `ListView.builder`

## Flex

- [ ] **Expanded / Flexible only inside Row / Column / Flex** — not inside Container/Stack/etc
- [ ] **Expanded matches Figma `primaryAxisSizingMode: FILL`**

## Responsive

- [ ] **LayoutBuilder present** if Figma has constraints annotations on any node
- [ ] **Hardcoded MediaQuery.size not used for layout decisions** — prefer LayoutBuilder or responsive widgets

## Text inside fixed-height cells

Apply this section to every `Text` whose nearest ancestor enforces a fixed height: `GridView*` with `mainAxisExtent` or `childAspectRatio`, `SizedBox(height: ...)`, `Container(height: ...)`, `IntrinsicHeight` row of equal-height cells, custom bottom-nav / tab bars, etc.

- [ ] **Text has `maxLines: 1` + `softWrap: false` + `overflow: TextOverflow.visible`** when the parent height budget assumes a single line (check Figma: if the text node's bounding box height ≈ one line of its font size × line-height, the design assumes one line). Without this, a marginally-too-wide string silently wraps to 2 lines and overflows the parent — see common-misreads.md § "Text wrap inflates fixed-height cell".
- [ ] **Sibling-cell visual consistency preserved** — for grids / lists / any layout with adjacent cells using the same TextStyle, FAIL if the fix uses `FittedBox(BoxFit.scaleDown)` per cell. That scales only overflowing labels, leaving neighboring labels rendered larger — visually inconsistent. Default to `overflow: TextOverflow.visible` instead so all cells render at the same font size.
- [ ] **Cell width budget verified against the longest label** — for icon grids and similar, multiply (longest label char count) × (font size in sp) and compare to cell width; if the ratio is > ~0.95 the bleed from `overflow: visible` may be visible at the cell edges. In that case, prefer reducing `crossAxisSpacing` / horizontal padding to widen cells (preserves uniform font), not scaling the text.
- [ ] **Figma `textAutoResize` cross-check** — if the Figma text node has `textAutoResize: HEIGHT` (auto-grows vertically) but the Flutter parent is fixed-height, the design and the implementation contradict each other. Either bump the parent height to match the design's natural multi-line allowance, OR enforce 1-line + `overflow: visible` if the screenshot shows 1 line.

## Outer container height budget arithmetic

For every fixed-height parent (`SizedBox(height:)`, horizontal `ListView` height, `Container(height:)`) wrapping a Column of stacked children, verify the parent budget can mathematically contain the children. See common-misreads.md § "Outer container height under-budgeted for child stack".

- [ ] **Sum of children ≤ parent height**. Compute: image heights + explicit `SizedBox(height:)` spacers + for every Text child `fontSize × textStyle.height × maxLines`. If the sum > parent budget, FAIL — no Text-config fix can rescue this; the parent height must grow.
- [ ] **`Stack` with `Positioned(bottom: <negative>)` has explicit spacer below**. Stack's intrinsic height = non-positioned child only; any visually-overhanging negative-Positioned child must be compensated by a sibling `SizedBox(height: overhang + gap)` AND counted in the parent budget.
- [ ] **TextStyle `height` ≥ 1.6 counted in the budget**. A `TextStyle(fontSize: 11, height: 2.0)` line box is 22 px tall, not 11. AI codegen often forgets this when verifying parent fits children. Flag any captioned card whose parent budget didn't visibly leave room for `fontSize × height` per line.
- [ ] **Slack of ≥ 4–6 logical px**. After summing children, the parent should still have a few px of slack to absorb sub-pixel rounding and OS font-padding differences. A budget that exactly equals the sum will overflow on some devices.
- [ ] **Conditional height: every branch independently sized**. When the parent uses `condition ? heightA : heightB` (e.g. `hasAvatar ? 161.h : 158.h`), verify the budget arithmetic for BOTH branches separately. A mis-fix that bumps both by the same delta is wrong when the under-budget amount differs per branch.
- [ ] **Overflow report → back-compute screenutil factor before sizing the fix**. Take a known child width (e.g. `SizedBox(width: 104.w)` reporting `w=107.6` → factor = 1.0346) and divide the reported `h` by that factor to recover the LOGICAL value. Match the logical value against EVERY conditional branch's declared height — don't guess which branch overflowed. See common-misreads.md § "Outer container height under-budgeted for child stack" → "Diagnosis protocol".

## Accessibility (warnings only)

- [ ] **Semantic labels on non-text interactive widgets** — buttons with only icons should have `Semantics(label: ...)`
- [ ] **Tappable regions ≥ 44px** minimum touch target

## Each item emits:

- PASS: meets expectation
- WARN: minor deviation, suggestion provided
- FAIL: structural issue that will break layout or render incorrectly
