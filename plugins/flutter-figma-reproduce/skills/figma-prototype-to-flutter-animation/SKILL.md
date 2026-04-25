---
name: figma-prototype-to-flutter-animation
description: "Reads a Figma node's Prototype configuration (interactions, variants, Smart Animate, page transitions) and emits the corresponding Flutter animation code suggestions — mapping variant switches to AnimatedContainer/AnimatedSwitcher, Smart Animate to Hero/AnimatedPositioned, page-level transitions to PageRouteBuilder, and Figma easing curves to Flutter Curves. Use when implementing interactions and motion from a Figma design. Requires Figma MCP logged in and the target node having prototype connections."
---

# figma-prototype-to-flutter-animation

## When to use

- Invoked by `flutter-figma-reproduce` main skill in Phase 3 (State & Interaction) when Figma prototype links exist
- Standalone: "turn this Figma prototype's animations into Flutter code"

## Inputs

- `figma_node` (required) — Figma nodeId or URL; must have prototype interactions
- `target_widget_file` (optional) — Dart file the animation should integrate into; if omitted, produces standalone snippets

## Workflow

Read `references/animation-mapping.md` for interaction → widget mapping.
Read `references/curve-mapping.md` for easing curve translation.

1. **Fetch Figma prototype data** via `mcp__figma__get_design_context` (includes `interactions` and `transitions` fields).
2. **Enumerate interactions** in the target node and descendants:
   - `TRIGGER_ON_CLICK` + `NAVIGATE` → page navigation with transition
   - `TRIGGER_ON_CLICK` + `CHANGE_TO` + variant → local state change with AnimatedContainer
   - `SMART_ANIMATE` on navigate → Hero or AnimatedPositioned
3. **Map each to Flutter widget pattern** using the mapping table.
4. **Translate easing**: Figma `EASE_IN_OUT_CUBIC` → Flutter `Curves.easeInOutCubic`, etc.
5. **Emit code suggestions** — don't modify source directly; provide snippets for the user to integrate.

## Output format

Produce a markdown report. Each interaction gets a numbered section with: the Figma source description (trigger, action, duration, curve), the Flutter pattern name, a cross-ref to `animation-mapping.md`, and the snippet as a standalone fenced `dart` block. Keep the report one level flat (no code blocks inside code blocks) so the SKILL.md itself never nests fences.

Example entry (prose):

> **Interaction 1 — Tap CTA Button → Confirm Screen.** Figma: TRIGGER_ON_CLICK + NAVIGATE + SMART_ANIMATE, 300ms, EASE_OUT_CUBIC. Flutter pattern: `PageRouteBuilder` with `FadeTransition`. See animation-mapping.md § "Smart Animate across pages". Snippet saved to `.figma-reproduce/<page>/animations/interaction-1.dart`.

## Constraints

- **No source modification** — only suggestions. Write snippets to separate files under `.figma-reproduce/<page>/animations/` so the user can copy them in deliberately.
- **Respect project conventions** — if project uses go_router, emit `context.push(...)` instead of `Navigator.push`; detect from pubspec.
- **Fail gracefully** — if Figma has prototype but MCP response omits interactions field, emit "prototype data unavailable, check Figma MCP version" and exit cleanly.
