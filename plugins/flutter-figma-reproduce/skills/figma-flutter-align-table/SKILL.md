---
name: figma-flutter-align-table
description: "Scans a Figma file and a Flutter project, produces an 'align table' mapping Figma design tokens (colors, typography, spacing, radius, shadow, icons) to the project's ThemeData fields, plus a gaps list (tokens in Figma but not in the project) and a component reuse map (Figma component instances to existing project widgets). Use when starting UI reproduction work from Figma, or when auditing a Flutter project's theme coverage against a design system. Requires Figma MCP logged in and target is a Flutter project (pubspec.yaml present)."
---

# figma-flutter-align-table

## When to use

- User starts UI reproduction work from a Figma URL against an existing Flutter project
- User audits a Flutter project's theme against a Figma design system
- Invoked by `flutter-figma-reproduce` main skill as Phase 0

## Inputs

- `figma_url` (required) — Figma URL (design or make), or fileKey + nodeId
- `flutter_project_path` (optional, default: cwd) — path to Flutter project root (must contain `pubspec.yaml` with `flutter:` dep)
- `output_dir` (optional, default: `{flutter_project_path}/.figma-reproduce/<page>/`) — where to write output files

## Workflow

Read `references/scan-strategy.md` for the detailed two-sided scanning algorithm.

Read `references/output-format.md` for the exact format of the three output files.

1. **Verify prerequisites** — Figma MCP available (try `get_metadata` on input URL), Flutter project (has `pubspec.yaml`).
2. **Scan Figma side** — pull metadata → libraries → variable defs → design context for target node.
3. **Scan project side** — read `pubspec.yaml`, `lib/theme/**`, `lib/widgets/**`, detect state management.
4. **Produce align table** — match Figma tokens against project ThemeData fields. Use `templates/align-table.md.tmpl`.
5. **Produce gaps list** — Figma tokens with no project counterpart. Use `templates/gaps.md.tmpl`.
6. **Produce component map** — Figma component instances ↔ project widgets by semantic match. Use `templates/component-map.md.tmpl`.
7. **Emit summary** — in the conversation: "Align table generated at <path>. X tokens matched, Y gaps, Z component candidates. Please review."

## Outputs

- `{output_dir}/align-table.md`
- `{output_dir}/gaps.md`
- `{output_dir}/component-map.md`

## Constraints

- **Never modify the Flutter project** — this skill is read-only on the project side. Gap-filling belongs to `flutter-theme-gap-filler`.
- **Never call `use_figma`** — this skill only reads Figma via `get_*` MCP tools.
- If Figma file has no variables/styles attached, produce align table from raw hex/px values and flag it in `gaps.md` as "Figma file not tokenized".
