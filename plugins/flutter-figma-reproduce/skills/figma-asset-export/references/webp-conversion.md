# WebP Conversion

## Prerequisites

`cwebp` binary must be on PATH. Install:
- macOS: `brew install webp`
- Ubuntu: `apt install webp`

Verify: `cwebp -version` → should print a version number.

## Conversion command

```bash
cwebp -q 85 -m 6 -mt -af input.png -o output.webp
```

Flags:
- `-q 85`: quality 85 — good balance; bump to 90 for hero images, 75 for backgrounds
- `-m 6`: compression method 6 — slower but smaller; acceptable for build-time
- `-mt`: multithread
- `-af`: auto filter

For images with transparency, cwebp preserves alpha by default. No extra flag needed.

## Density preference

Flutter's asset resolver looks for density-aware paths:
- `assets/images/foo.webp` — base
- `assets/images/2.0x/foo.webp` — @2x
- `assets/images/3.0x/foo.webp` — @3x

This skill defaults to **@2x only**:
- Export PNG from Figma at `scale: 2`
- Place output under `{assets_root}/2.0x/{name}.webp`
- Also place a 1x downscaled copy at `{assets_root}/{name}.webp` (Flutter requires base path)

Generate the 1x copy via cwebp resize:

```bash
cwebp -q 85 -resize <half_w> <half_h> input.png -o base.webp
```

Skip 1x generation only if the image is pixel-art or has text baked in (bilinear downscale would degrade); in that case the skill should emit a warning and leave the base path empty (user must handle).

## Skipping conversion

If the image is a GIF or already WebP, skip conversion, just copy. If PNG < 4KB, skip conversion (overhead not worth it).
