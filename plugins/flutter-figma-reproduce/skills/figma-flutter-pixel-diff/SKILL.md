---
name: figma-flutter-pixel-diff
description: "Runs a Flutter app via the Patrol integration test framework, captures a screenshot of a target page with native-layer stability (animation wait, font preload), pulls the corresponding Figma screenshot via MCP, computes a pixel-wise diff (overall difference rate + heatmap + region-level breakdown), and suggests likely causes (spacing, color, font, missing element). Use for visual verification during UI reproduction, or to audit an existing page against its Figma design. Requires Patrol installed in the Flutter project (patrol_cli + patrol dev dep) and Figma MCP logged in."
---

# figma-flutter-pixel-diff

## When to use

- Visual verification during UI reproduction (Phase 2/3/4 in `flutter-figma-reproduce`)
- Standalone audit: "does this Flutter page match the Figma design?"

## Inputs

- `flutter_project_path` (required) — Flutter project root; app must be runnable
- `target_route` (required) — route name / deep link to drive the app to the page under test
- `figma_url` (required) — Figma URL of the corresponding design node
- `diff_threshold` (optional, default: 0.05) — difference rate below which verdict is "pass"
- `viewport` (optional, default: phone 390x844) — output screenshot size; should match Figma frame size

## Workflow

Read `references/patrol-bridge.md` for driver details (Patrol setup, test scaffold template, stability knobs).

1. **Preflight checks**
   - `patrol_cli` available on PATH (`patrol --version`)
   - `patrol` listed in project `dev_dependencies` (check `pubspec.yaml`)
   - `integration_test` also listed in `dev_dependencies` (Patrol depends on it)
   - Figma MCP responds to `get_metadata`
   - Flutter project has `pubspec.yaml` with `flutter:` dep
2. **Capture Flutter screenshot**
   - Scaffold a Patrol integration test at `integration_test/_pixel_diff_capture.dart` (generated, not checked in) that launches the app, navigates to `target_route`, waits for animations to settle (see bridge doc), and calls Patrol's native `takeScreenshot` at `viewport` size
   - Run `patrol test -t integration_test/_pixel_diff_capture.dart`; collect the PNG from Patrol's output directory
3. **Capture Figma screenshot**
   - Invoke `mcp__figma__get_screenshot` with fileKey + nodeId
   - Normalize to same viewport size
4. **Compute diff**
   - Pixel-wise RGB delta per pixel
   - Aggregate to overall diff rate (diff pixels / total)
   - Produce heatmap (red = high delta, green = low)
   - Region-level breakdown: divide image into 6x10 grid, report per-cell diff rate
5. **Suggest causes**
   - Top-3 regions by diff rate → check: is it a text region? color region? image? → emit specific suggestion (e.g. "region (3,2) 15% diff, likely font rendering — check letterSpacing")
6. **Verdict**
   - `pass` if overall diff rate < threshold
   - `fail` with cause suggestions otherwise
7. **Output report**
   - Save to `{flutter_project_path}/.figma-reproduce/<page>/pixel-diff-{timestamp}.md` with embedded images (heatmap as base64 or file ref)

## Output format

```
# Pixel Diff Report

**Verdict:** pass / fail
**Overall diff rate:** 3.2% (threshold: 5%)
**Flutter screenshot:** path/to/flutter.png
**Figma screenshot:** path/to/figma.png
**Heatmap:** path/to/heatmap.png

## Region breakdown

| Region (col, row) | Diff rate | Likely cause |
|---|---|---|
| (3, 2) | 15% | Text region — check font/letterSpacing |
| (4, 5) | 8% | Card region — check shadow / border |
| ... | ... | ... |

## Suggestions

1. ...
2. ...
```

## Constraints

- **Never modify source code** — diagnostic only, fixes belong to upstream skills
- **Stabilize screenshots**: wait for animation, force-render fonts before capture (see bridge doc). If app can't reach steady state, report "unstable" instead of giving a wrong number.
