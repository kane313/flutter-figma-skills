---
name: figma-autolayout-to-flex
description: Translates a Figma node's Auto Layout configuration into Flutter widget tree intent — choosing between Row / Column / Stack / Wrap, mapping main/cross axis alignment, distinguishing Figma padding (inside component) vs gap (between children), and flagging constraints that require LayoutBuilder. Use when converting a Figma design node to Flutter widget structure, or when auditing an existing widget tree against a Figma layout. Input is typically a Figma nodeId; output is a structured translation suggestion (not raw code — that's the consumer's job).
---

# figma-autolayout-to-flex

## When to use

- Invoked by `flutter-figma-reproduce` main skill in Phase 1 (Skeleton)
- Standalone: "translate this Figma frame's layout to Flutter widget tree intent"

## Inputs

- `figma_node` (required) — Figma nodeId, or the JSON from `get_design_context`

## Workflow

Read `references/translation-rules.md` for the complete rule set.

Read `references/test-cases.md` for examples — use these as reference for ambiguous cases.

1. Fetch node data via `mcp__figma__get_design_context` (if not already provided).
2. For the target node and each descendant with `layoutMode`, apply the translation rules.
3. Produce a **translation tree** — for each node, emit:
   - Flutter widget: one of `Row` / `Column` / `Stack` / `Wrap` / `Flex` / leaf
   - `mainAxisAlignment`, `crossAxisAlignment`
   - `mainAxisSize`
   - Padding: `EdgeInsets.fromLTRB(...)` or `EdgeInsets.symmetric(...)`
   - Gap: `SizedBox` / `Spacer` / via package `gap`
   - Child list (recurse)
4. Emit defensive warnings:
   - Stack nesting > 2
   - Inferred ScrollView but unbounded child
   - Constraints present → flag for `LayoutBuilder`

## Output format

A markdown document (or returned inline) with sections per subtree, showing the translation decision and reasoning:

```
## Node "HeroCard" (layoutMode=VERTICAL)

- Widget: `Column`
- mainAxisAlignment: `MainAxisAlignment.start` (Figma primaryAxisAlignItems=MIN)
- crossAxisAlignment: `CrossAxisAlignment.stretch` (Figma counterAxisAlignItems=STRETCH)
- mainAxisSize: `MainAxisSize.min` (Figma counterAxisSizingMode=AUTO)
- Padding: `EdgeInsets.all(16)` (Figma paddingLeft/Right/Top/Bottom=16)
- Gap between children: `SizedBox(height: 8)` (Figma itemSpacing=8)
- Children: [Text, Row[Icon, Text], Button]
```

## Constraints

- **No code emission** — output is translation *intent*, not compilable Dart. The consumer (main skill / human dev) writes the actual widget tree.
- **Never use `use_figma`** — read-only.
