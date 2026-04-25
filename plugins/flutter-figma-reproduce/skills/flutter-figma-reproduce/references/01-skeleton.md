# Phase 1: Skeleton

## Goal
Build the widget tree structure (no styling) that matches the Figma node's layout intent. Output: `lib/pages/<target>/_skeleton.dart`.

## Inputs
- `.figma-reproduce/<target>/align-table.md` — for allowed values and placeholder exceptions
- Figma node (fetched via `get_design_context`)

## Orchestration

1. Read `references/failure-modes.md` section "结构类" — 5 known structural pitfalls to actively avoid
2. Invoke `figma-autolayout-to-flex` on target Figma node. Output saved to `.figma-reproduce/<target>/autolayout-<nodeId>.md`
3. Load `templates/skeleton-prompt.md.tmpl` — fill `{{AUTOLAYOUT_OUTPUT}}` + `{{ALIGN_TABLE_SUMMARY}}` + write to `.figma-reproduce/<target>/skeleton-prompt.md`
4. Generate `lib/pages/<target>/_skeleton.dart` following the prompt:
   - Only structural widgets: Scaffold, Row, Column, Stack, Wrap, ListView, Padding, SizedBox
   - Leaf nodes as placeholders:
     - Text → `Text('<placeholder>')`
     - Image → `ColoredBox(color: Color(0xFFE5E5E5))`
     - Icon → `Icon(Icons.circle, color: Color(0xFFE5E5E5))`
     - Button → `Container(child: Text('<label>'))` (no styling, just structure)
   - No color literals (except placeholder grey)
   - No custom fonts, no shadows, no border-radius (deferred to Phase 2)
5. Run Token layer check (see `tools/scorecard-checker.md`). Placeholder grey + SizedBox magic numbers are exempt during Phase 1.
6. Run `flutter-widget-tree-review` structural review against Figma node. On FAIL verdict, regenerate up to 2 times — feeding back the review's issues into the generation prompt. Third failure → abort.
7. Run `dart analyze lib/pages/<target>/_skeleton.dart` — must pass before advancing
8. **Checkpoint** (safe mode): emit CHECKPOINT — "Phase 1 skeleton ready at `lib/pages/<target>/_skeleton.dart`. Review widget tree STRUCTURE only (not styling). Reply `continue` to proceed, or describe what to fix."
9. **Fast mode**: auto-advance without user pause (widget-tree-review still gated the advance internally)

## Immersive defaults (mandatory for every Scaffold)

Every page's root Scaffold MUST be immersive — content extends behind the status bar, status bar text adapts to background luminance.

Default template:

```dart
Scaffold(
  extendBodyBehindAppBar: true,
  appBar: AppBar(
    backgroundColor: Colors.transparent,
    elevation: 0,
    scrolledUnderElevation: 0,
    systemOverlayStyle: <STATUS_BAR_STYLE>,  // see inference rule below
  ),
  body: SafeArea(
    top: false,    // content extends behind status bar — top SafeArea would create a gap
    bottom: true,
    child: ...,
  ),
)
```

For pages without an AppBar, wrap with `AnnotatedRegion<SystemUiOverlayStyle>`:

```dart
AnnotatedRegion<SystemUiOverlayStyle>(
  value: <STATUS_BAR_STYLE>,
  child: Scaffold(
    body: Stack(
      children: [
        // FULL-BLEED background (image/gradient/color) — MUST be the outermost
        // Stack layer, NOT inside any Center/SizedBox(width:N) constraint.
        // Otherwise on devices wider than N, page-bg shows as side strips.
        Positioned(top: 0, left: 0, right: 0, height: ..., child: <bg>),
        // Centered design-width content
        Center(child: SizedBox(width: 375, child: <page-content-stack>)),
      ],
    ),
  ),
)
```

### Full-bleed background layering rule (mandatory)

When the page uses `Center+SizedBox(width:N)` to keep content at the design viewport (375 / 390) on wider devices, ANY full-bleed visual MUST be lifted out of that constraint:

- ✓ Lifted (full device width): page background color, header gradient/image, status-bar tint
- ✗ Inside SizedBox (capped at N): page content widgets, cards, text, icons

If unclear: ask "would the designer want this to bleed to screen edge or stay within the design width?". Backgrounds bleed; content stays.

### Inferring `<STATUS_BAR_STYLE>`

Sample the Figma top 44px region:

1. From `get_design_context`, find node(s) with fills intersecting `y ∈ [0, 44]`
2. Compute average RGB → HSL luminance using BT.601 weights `(0.299r + 0.587g + 0.114b) / 255`
3. Map → emit an EXPLICIT struct literal, NEVER the preset `SystemUiOverlayStyle.dark` / `.light`:
   - luminance > 0.5 → light background → dark icons/text:
     ```dart
     const SystemUiOverlayStyle(
       statusBarColor: Colors.transparent,
       statusBarIconBrightness: Brightness.dark,
       statusBarBrightness: Brightness.light,
     )
     ```
   - luminance ≤ 0.5 → dark background → light icons/text:
     ```dart
     const SystemUiOverlayStyle(
       statusBarColor: Colors.transparent,
       statusBarIconBrightness: Brightness.light,
       statusBarBrightness: Brightness.dark,
     )
     ```

**Why no preset:** Flutter's `SystemUiOverlayStyle.dark` preset sets `statusBarColor: Color(0x40000000)` (25% black scrim) on Android. On a page that already provides its own light header (gradient/image), this scrim renders as a visible dark wash over the status bar — the opposite of what an immersive page wants. Always emit `Colors.transparent` explicitly.

If top region has a gradient or image, sample the top 10% of pixels by mean. If gradient direction makes luminance ambiguous (mixed), default to dark icons/text variant and emit a note in the report.

The `<STATUS_BAR_STYLE>` value is captured in the `templates/skeleton-prompt.md.tmpl`'s `{{STATUS_BAR_STYLE}}` placeholder so the AI generating skeleton receives it directly.

### Override path

If Figma design clearly shows a NON-immersive page (opaque AppBar, status-bar-color fill present in the design itself), override the default:

- Drop `extendBodyBehindAppBar: true`
- Set `AppBar.backgroundColor` to the Figma value (token from align table)
- `SafeArea(top: true, bottom: true)`
- Still set `systemOverlayStyle` based on the AppBar color luminance

Emit a Phase 1 checkpoint note: "Detected non-immersive AppBar in Figma — using opaque header instead of default immersive template."

## Defensive rules (AI must enforce during generation)

- Stack nesting ≤ 2 layers
- No `SingleChildScrollView` wrapping `ListView`
- `SafeArea` wraps `Scaffold.body` when Figma top frame has device chrome (y starts at 0)
- `Flexible` / `Expanded` only inside Row/Column/Flex ancestors
- `LayoutBuilder` at ancestor level when Figma has `constraints` annotations on any node
- MaterialApp NEVER nested inside a page widget — page widget should be just Scaffold or custom widget
- No `MediaQuery.of(context).size` as primary layout mechanism — prefer `LayoutBuilder` or `FractionallySizedBox`

## Stopping conditions

- `figma-autolayout-to-flex` returns empty / error → abort, report "layout translation failed — inspect Figma node for unusual structure"
- `flutter-widget-tree-review` fails 2 consecutive regenerations → abort, ask user to intervene (tree inherently ambiguous)
- `dart analyze` fails with syntax error after regen → abort, ask user to intervene
