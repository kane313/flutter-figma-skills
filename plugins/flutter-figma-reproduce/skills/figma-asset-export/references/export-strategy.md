# Figma Export Strategy

## Critical rule: never pull asset bytes through Claude's image channel

Asset download MUST go through Bash + curl (Figma REST API). Do NOT use `mcp__figma__get_screenshot` to retrieve PNG/SVG bytes — that call surfaces the file as an Anthropic image content block, which fails for:

- SVG (Anthropic image input doesn't accept SVG) → "Could not process image"
- Images > 5MB → size limit
- Uncommon encodings → "Could not process image"

`get_screenshot` is fine for *visual reference* ("show me what the node looks like") but not for *byte retrieval*. Always use REST for the actual download.

## Identifying exportable assets

First enumerate what needs to be exported — this step is metadata-only and safe:

In the design context returned by `mcp__figma__get_design_context`:

| Node signal | Asset type | Export format |
|---|---|---|
| `fills: [{ type: IMAGE, imageRef: '<hash>' }]` | Raster image | PNG @2x via REST → WebP |
| `type: INSTANCE`, `componentId` matches a known icon library | Icon | SVG via REST |
| `type: VECTOR` with exportSettings | Vector illustration | SVG via REST |
| `type: RECTANGLE` with gradient fill only | Not an asset | Skip (render in Flutter via gradient) |

Collect the list of node IDs and desired formats in memory — do NOT call `get_screenshot` on each during enumeration.

## Preflight: FIGMA_TOKEN

Before any download, verify the token:

```bash
test -n "$FIGMA_TOKEN" && echo "token present" || echo "MISSING — abort"
```

If missing, emit: "FIGMA_TOKEN env var not set. Get a Personal Access Token from https://www.figma.com/settings (Account → Personal access tokens), then `export FIGMA_TOKEN=<token>`." Abort the skill run — don't try workarounds.

Verify scope works:

```bash
curl -sS -H "X-Figma-Token: $FIGMA_TOKEN" "https://api.figma.com/v1/me" | head -c 200
```

Expected: JSON with your user info, not an `{"status":403,...}`.

## Pulling raster images

Batched request (up to ~100 nodes per call — Figma's recommended limit):

```bash
curl -sS -H "X-Figma-Token: $FIGMA_TOKEN" \
  "https://api.figma.com/v1/images/${FILE_KEY}?ids=${NODE_IDS_COMMA}&format=png&scale=2" \
  > response.json
```

Response shape:

```json
{
  "err": null,
  "images": {
    "1:23": "https://s3-figma-exports.../abc.png?...",
    "1:45": "https://s3-figma-exports.../def.png?..."
  }
}
```

Each value is a signed S3 URL valid for ~30 minutes. Download each:

```bash
for node_id in $(jq -r '.images | keys[]' response.json); do
  url=$(jq -r ".images[\"$node_id\"]" response.json)
  curl -sS "$url" -o "tmp/${node_id//:/_}.png"
done
```

Name the output file from the Figma node `name` attribute (from `get_design_context`), sanitized:
- lowercase
- spaces / `-` / `/` → `_`
- drop non-ASCII-printable chars; if name becomes empty, fall back to `asset_<nodeId>`

## Pulling SVG icons

Same endpoint, `format=svg`:

```bash
curl -sS -H "X-Figma-Token: $FIGMA_TOKEN" \
  "https://api.figma.com/v1/images/${FILE_KEY}?ids=${ICON_IDS_COMMA}&format=svg" \
  > svg_response.json

for node_id in $(jq -r '.images | keys[]' svg_response.json); do
  url=$(jq -r ".images[\"$node_id\"]" svg_response.json)
  curl -sS "$url" -o "tmp/icons/${node_id//:/_}.svg"
done
```

Post-process optional: strip Figma's `<desc>` and unused id attributes via `sed` or `svgo` for smaller files.

## Component-source export (icon size consistency)

The same icon component appearing at multiple sizes in the design (e.g. 24×24 in a nav bar, 16×16 inline) MUST export as ONE asset at the component's **source size**, not at each instance's render size. Without this rule, the same conceptual icon ends up as multiple slightly-different bitmaps — the exact "same icon, different sizes" inconsistency POC v1 hit.

### Algorithm

1. While enumerating exportable assets (the table above), group nodes by `componentId` (Figma component primary key)
2. For each group, find the SOURCE component definition:
   - Resolve the component via `mcp__figma__get_design_context` on the component itself (NOT the instance)
   - Extract its declared `width × height` (e.g. `24 × 24`)
3. Export ONE asset per `componentId` at the source dimensions
4. Build `.figma-reproduce/<page>/dedup-map.json`:

```json
{
  "1:234": {
    "asset_path": "assets/icons/heart.svg",
    "source_size": [24, 24],
    "instances": [
      {"node_id": "5:678", "render_size": [24, 24]},
      {"node_id": "5:679", "render_size": [16, 16]}
    ]
  }
}
```

5. Emit it alongside the assets so Phase 2 knows which asset to reuse at which render size

### Render-time sizing (consumer responsibility, NOT asset responsibility)

Display size at each instance is controlled in CODE via `flutter_screenutil` suffixes (Phase 2):

```dart
// 24×24 nav icon
SvgPicture.asset('assets/icons/heart.svg', width: 24.w, height: 24.w)

// 16×16 inline icon — SAME asset, different render size
SvgPicture.asset('assets/icons/heart.svg', width: 16.w, height: 16.w)
```

Phase 2 reads `dedup-map.json` to learn that both instances point to one file.

### Edge case: nodes without componentId

For inlined `VECTOR` nodes (no componentId):

1. Compute SHA-1 of the exported SVG bytes (geometry equality)
2. If two inlined vectors have identical SHA → treat as one asset, reuse the same file
3. If they differ even slightly (anti-aliasing, rounding) → export both with disambiguating filenames (`<base>-1`, `<base>-2`) and emit a warning: "two visually similar but non-identical inlined vectors found — recommend componentizing in Figma so they share a source"

### Edge case: raster images (image fills)

Raster images use `imageRef` (not componentId). Same rule applies: dedup by `imageRef`, export ONCE at source resolution (2x scale). Different instances render at different sizes via `Image.asset(width: ...w, height: ...h)`.

## Handling errors

- `err` non-null in JSON response → log and skip those IDs
- 403 from Figma → token lacks file access; abort with clear message
- 404 on signed URL → signed URL expired, re-request
- Network timeout → retry once with exponential backoff; on second failure, skip the asset and continue

## Deduplication

After download, compute SHA-1 of each file:

```bash
sha1sum tmp/*.png | sort | uniq -w40 -d
```

Reuse first occurrence, skip duplicates, emit note "asset at node X is identical to asset at node Y, reusing <path>".

## Summary: what Claude does vs what Bash does

- **Claude**: enumerate assets from `get_design_context`, build the ID list, shell out to Bash for downloads, then for `cwebp`, then for `pubspec.yaml` patching, then emit summary
- **Bash**: curl, jq, cwebp, sha1sum, mv, sed
- **Never**: `get_screenshot` for bytes, image content blocks passing through Claude
