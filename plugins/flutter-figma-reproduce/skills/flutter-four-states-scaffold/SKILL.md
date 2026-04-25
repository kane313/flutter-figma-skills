---
name: flutter-four-states-scaffold
description: "Generates loading / empty / error / success state widgets plus mock data for a Flutter data-driven page. Adapts to the project's state management (detects provider / riverpod / bloc / none from pubspec.yaml) and produces a sealed-class state model, switch-based widget builder, and a _mock_data.dart file with realistic placeholder data. Use when implementing a page that loads async data — never ship only the success state. Requires a Flutter project."
---

# flutter-four-states-scaffold

## When to use

- Invoked by `flutter-figma-reproduce` main skill in Phase 3 (State & Interaction) for any page that loads async data
- Standalone: "give me a loading/empty/error/success scaffold for this data type"

## Inputs

- `data_type` (required) — Dart type name or description (e.g. `List<Post>`, `UserProfile`, "list of orders")
- `page_name` (required) — snake_case page name (determines file names)
- `flutter_project_path` (optional, default: cwd)
- `state_mgmt` (optional, auto-detect) — `provider` / `riverpod` / `bloc` / `none`; if `none`, produces a plain `StatefulWidget` with a nullable data field

## Workflow

Read `references/state-model.md` for the sealed-class state model.
Read `references/widget-templates.md` for the 4 state widget templates.
Read `references/mock-data.md` for generating realistic placeholders.

1. **Detect state management** from `pubspec.yaml` dependencies if not provided.
2. **Emit state model** — a sealed class or enum + data holder matching the detected pattern.
3. **Emit widget file** — `lib/pages/<page_name>/page.dart` with switch on state → 4 state widgets.
4. **Emit mock data** — `lib/pages/<page_name>/_mock_data.dart` with realistic data for success state + toggle helper.
5. **Emit state widget files** (optional, if page grows large) — `_loading.dart`, `_empty.dart`, `_error.dart` siblings.

## Output format

Summary in conversation listing files written plus one-line dev instructions on how to toggle between states for preview. Keep it prose, not nested code blocks.

## Constraints

- **Never omit a state** — all four are required even if Figma only drew one.
- **Match project state mgmt** — don't drag riverpod into a plain-Provider project.
- **Mock data must be realistic** — avoid `List.generate(5, (i) => 'Item $i')`; use domain-appropriate placeholders from `mock-data.md`.
- **Error state must offer retry** — error widget includes a callback.
