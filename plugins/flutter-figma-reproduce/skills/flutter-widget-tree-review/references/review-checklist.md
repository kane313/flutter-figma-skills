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

## Accessibility (warnings only)

- [ ] **Semantic labels on non-text interactive widgets** — buttons with only icons should have `Semantics(label: ...)`
- [ ] **Tappable regions ≥ 44px** minimum touch target

## Each item emits:

- PASS: meets expectation
- WARN: minor deviation, suggestion provided
- FAIL: structural issue that will break layout or render incorrectly
