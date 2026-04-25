---
name: figma-asset-export
description: "Exports image and icon assets from a Figma node into a Flutter project's assets/ directory, converting to WebP with @2x density preference, and automatically patches pubspec.yaml's flutter.assets section. Use when implementing UI from Figma and the design contains raster images, custom icons, or image fills that need to be materialized in the Flutter project. Requires Figma MCP logged in, cwebp binary on PATH, and a Flutter project (pubspec.yaml present)."
---

# figma-asset-export

## When to use

- Invoked by `flutter-figma-reproduce` main skill in Phase 2 (Style) when image fills or icon components need materialization
- Standalone: "pull all images from this Figma frame into my Flutter project"

## Inputs

- `figma_node` (required) — Figma nodeId or URL
- `flutter_project_path` (optional, default: cwd) — Flutter project root
- `assets_root` (optional, default: `assets/images/`) — target directory under project root
- `icon_root` (optional, default: `assets/icons/`) — where to put icon/SVG assets

## Workflow

Read `references/export-strategy.md` — **critical**: it explains why asset bytes must come via Bash+curl (Figma REST API), NEVER via `get_screenshot` (that path triggers Anthropic "Could not process image" for SVG / large / exotic inputs).
Read `references/webp-conversion.md` for the cwebp command and density rules.
Read `references/pubspec-patch.md` for safe pubspec editing.

1. **Preflight**:
   - Figma MCP reachable (safe metadata call like `get_metadata`)
   - `cwebp --version`
   - `curl`, `jq` available
   - **`FIGMA_TOKEN` env var set and valid** — test with `curl -H "X-Figma-Token: $FIGMA_TOKEN" https://api.figma.com/v1/me`. If missing, abort with clear instruction to set it (see export-strategy.md).
   - Target Flutter project has `pubspec.yaml`
2. **Enumerate assets** from `mcp__figma__get_design_context` — metadata only, do NOT fetch bytes yet:
   - Nodes with `fills: [{ type: IMAGE, imageRef }]` → raster images
   - Instances referencing icon library components → SVG export candidates
3. **Download via Bash + Figma REST API** (never via `get_screenshot`):
   - Batch PNG request: `curl -H "X-Figma-Token: $FIGMA_TOKEN" https://api.figma.com/v1/images/{fileKey}?ids=...&format=png&scale=2`
   - Parse response `{images: {nodeId: signedUrl}}` with `jq`
   - `curl` each signed URL to a temp dir
   - Same pattern for SVG with `format=svg`
4. **Convert raster to WebP** via `cwebp`. Preserve filename (sans extension), add `.webp`.
5. **Move into project**: raster → `{assets_root}`, icons (SVG, no WebP conversion) → `{icon_root}`. Preserve Figma node names (sanitize to snake_case).
6. **Patch pubspec.yaml**: add new paths to `flutter.assets:` section, preserving existing entries and YAML formatting.
7. **Emit summary**: list exported paths + any skipped files + pubspec diff.

## Outputs

- Files under `{assets_root}/` and `{icon_root}/`
- Patched `pubspec.yaml`
- `.figma-reproduce/<page>/dedup-map.json` — `{componentId: {asset_path, source_size, instances}}` mapping; consumed by Phase 2 to render the same asset at multiple sizes (via `flutter_screenutil` suffixes) without duplicate exports
- Summary in conversation

## Constraints

- **NEVER call `mcp__figma__get_screenshot` to download asset bytes** — it surfaces the image as an Anthropic image content block and fails for SVG / oversized / exotic formats with "Could not process image" (400). Only Bash+curl via REST API is allowed for bytes.
- **One asset per Figma component** — use `componentId` (or `imageRef` for raster) as the dedup primary key; export ONCE at the component's source size, never at instance render size. See `references/export-strategy.md` § "Component-source export". This prevents the "same icon, different files, slightly different bitmaps" inconsistency.
- **Never overwrite** an existing asset without explicit user confirmation — compare content hash first; if different, write to `<name>-v2.webp` and flag for review.
- **SVG icons are not converted to WebP** — keep as SVG (use `flutter_svg` package for rendering).
- **pubspec.yaml must remain valid YAML** — use `references/pubspec-patch.md` strategy; never blind-write.
