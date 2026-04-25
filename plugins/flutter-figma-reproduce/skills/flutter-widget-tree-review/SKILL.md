---
name: flutter-widget-tree-review
description: "Reviews a generated Flutter widget tree against a Figma node's layout intent, catching common misreads (Stack used where Row/Column would fit, Column + spaceBetween when a Spacer is better, SingleChildScrollView wrapping a ListView, etc.). Emits pass/fail verdict plus a list of specific issues with suggested fixes. Use as the structural layer of the Phase 1 skeleton verification, or as a standalone widget-tree audit. Requires access to both the Flutter source file and the corresponding Figma node."
---

# flutter-widget-tree-review

## When to use

- Invoked by `flutter-figma-reproduce` main skill as the automatic "structural layer" check during Phase 1 (Skeleton)
- Standalone: "review this widget tree against the Figma design, flag issues"

## Inputs

- `dart_file` (required) — path to the Flutter widget file to review
- `figma_node` (required) — Figma nodeId or URL of the design node this file implements
- `strict` (optional, default: false) — if true, treat warnings as failures

## Workflow

Read `references/review-checklist.md` for the full checklist structure.
Read `references/common-misreads.md` for AI-generated code pitfalls.

1. **Read the Dart file** and identify the top-level widget tree (build method body).
2. **Fetch Figma node** via `mcp__figma__get_design_context` if not cached.
3. **Walk the widget tree** — for each `Row` / `Column` / `Stack` / `Wrap` / scroll widget, cross-reference the corresponding Figma subtree.
4. **Apply checklist** — for each item, determine pass / warn / fail.
5. **Emit report** with:
   - Overall verdict (pass / fail)
   - Per-issue entries: severity, location (file:line), description, suggested fix
   - Cross-references to `references/common-misreads.md` for known patterns

## Output format

Produce a markdown report with sections. Each issue is a list item, not a nested code block:

- Header: **Target**, **Figma**, **Verdict** (pass / fail with issue count)
- Section "Issues" with one entry per finding:
  - Severity tag (FAIL / WARN / PASS)
  - Location as `file:line`
  - One-line description + Figma property evidence
  - Cross-ref to `common-misreads.md` section if applicable
  - `Suggested fix:` as inline prose or a short `before → after` snippet on a single line

Example entry (rendered as prose, not as nested code block):

> **FAIL — Stack where Column would fit** (login.dart:42). Figma subtree has `layoutMode: NONE` but children don't overlap (y offsets are contiguous). See common-misreads.md § "Stack misread". Suggested fix: replace `Stack(children: [header, body])` with `Column(children: [header, SizedBox(height: 16), body])`.

Avoid nesting code blocks inside the report — keep fixes as inline code or short one-liners; if a multi-line snippet is essential, link to a separate `*-fix.dart` file written beside the report.

## Constraints

- **Read-only** — never modify the file; only suggest fixes.
- **Cross-reference Figma** — every issue must cite a Figma node property as evidence, not just "looks wrong".
- **Don't second-guess styling** — this is a structural review, not a pixel audit (that's pixel-diff's job).
