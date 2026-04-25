# flutter-figma-skills

> 🌐 **English** · [中文](README.zh-CN.md)

Claude Code marketplace for **Flutter UI reproduction from Figma** — a 5-phase orchestrated pipeline that takes a Figma node URL and produces a high-fidelity Flutter page in your project, with per-phase scorecard verification on a real Android / iOS device.

## What's in this marketplace

| Plugin | Description |
|---|---|
| **`flutter-figma-reproduce`** | Main orchestrator: align design tokens → build skeleton → fill styles → wire states & interactions → verify via layered scorecard. Includes 8 helper sub-skills (asset-export, autolayout-to-flex, four-states-scaffold, prototype-to-animation, theme-gap-filler, widget-tree-review, pixel-diff, align-table). |

## Install

In Claude Code:

```bash
# 1. Register the marketplace (one-time)
/plugin marketplace add <owner>/flutter-figma-skills        # GitHub
# OR
/plugin marketplace add /local/path/to/flutter-figma-skills # local dir

# 2. Install the plugin
/plugin install flutter-figma-reproduce@flutter-figma-skills

# 3. Activate
/reload-plugins
```

Verify with `/help` — you should see the `flutter-figma-reproduce` skill plus 8 helper skills under the same plugin namespace.

## Prerequisites (one-time per developer machine)

| Dependency | Used for | Install |
|---|---|---|
| Claude Code (latest, with `/plugin` command) | Loading the plugin | Update Claude Code |
| Figma MCP server logged in | Reading Figma node metadata + screenshots | Configure via `/mcp` in Claude Code |
| `FIGMA_TOKEN` env var | Exporting raster/SVG assets via Figma REST API | Generate at Figma → Settings → Personal access tokens |
| `cwebp` on PATH | PNG → WebP transcoding for asset optimization | macOS: `brew install webp` |
| Flutter SDK + a Flutter project (`pubspec.yaml` with `flutter:` dep) | Build/analyze/install pipeline | `flutter doctor` should pass |
| Android device or emulator (recommended) | Phase 4 real-device pixel diff against Figma | `adb devices` should list the target |

## Quickstart

In any Flutter project directory, ask Claude Code:

```
还原这个页面：<figma-url>
```

or

```
implement this Figma design: <figma-url>
```

The skill auto-triggers when the message contains `figma.com/design/...` AND your CWD has a `pubspec.yaml` with a `flutter:` dependency.

### Modes

| Mode | When to use | How to invoke |
|---|---|---|
| `safe` (default) | First-time use on a new design | Just send the URL — checkpoints between every phase |
| `fast` | Trust the pipeline, skip mid-phase pauses | `--mode=fast` argument |
| `auto` | One-shot run, no pauses, single info summary at end | `--mode=auto` OR include trigger phrases like "不用询问" / "yolo" / "autopilot" / "跑完为止" |

Example:

```
implement this design: https://www.figma.com/design/.../page?node-id=1-870  --mode=auto
```

## Pipeline overview

The orchestrator runs 5 phases per page reproduction:

| Phase | Output | Checkpoint (safe / fast / auto) |
|---|---|---|
| 0 Align | `align-table.md` + `gaps.md` + `component-map.md` in `.figma-reproduce/<page>/` | mandatory / mandatory / auto-defaults |
| 1 Skeleton | `_skeleton.dart` (structure only, no styling) | mandatory / auto / auto |
| 2 Style | `<page>_page.dart` with tokens + assets exported to `assets/` | optional / optional / auto |
| 3 State & Interaction | sealed-class state machine + `_mock_data.dart` | mandatory / mandatory / auto-defaults |
| 4 Verify | `verify-report.md` + pixel-diff side-by-side + heatmap | mandatory / mandatory / auto-report |

Per-page artifacts are saved to `<flutter-project>/.figma-reproduce/<page>/` so you can re-run later with the diff strategy (only changed nodes get regenerated).

## Defensive rules baked in (selection)

- Status bar style = explicit `SystemUiOverlayStyle(statusBarColor: Colors.transparent, ...)` literal — never the `.dark` / `.light` preset (which carries a 25% black scrim on Android)
- Full-bleed backgrounds (header gradient/image) are lifted out of `Center+SizedBox(width:N)` content constraints — bleed to device width
- Chinese `Text` inside fixed-width containers wrapped in `Flexible+FittedBox(scaleDown)+softWrap:false` — handles PingFang SC vs Android Noto kerning drift
- Asset content pre-flight (auto mode mandatory): every Figma node flagged for export gets metadata-subtree inspected; if `children` includes `text` nodes, the asset bakes labels and text duplication is averted at design time
- Skeleton-era artifacts must be deleted before Phase 2 styling — no carrying placeholder Positioned panels into production

Full failure-mode catalog: `plugins/flutter-figma-reproduce/skills/flutter-figma-reproduce/references/failure-modes.md`

## Repo layout

```
flutter-figma-skills/
├── .claude-plugin/marketplace.json     # registry entry — required
├── README.md                           # this file
└── plugins/
    └── flutter-figma-reproduce/
        ├── LICENSE
        ├── README.md                   # plugin-level docs
        └── skills/                     # 9 skills
            ├── flutter-figma-reproduce/        ← orchestrator (start here)
            ├── figma-flutter-align-table/
            ├── figma-autolayout-to-flex/
            ├── figma-asset-export/
            ├── figma-prototype-to-flutter-animation/
            ├── figma-flutter-pixel-diff/
            ├── flutter-four-states-scaffold/
            ├── flutter-theme-gap-filler/
            └── flutter-widget-tree-review/
```

## Updating

When the marketplace owner pushes a new version:

```bash
/plugin marketplace update flutter-figma-skills
/plugin upgrade flutter-figma-reproduce@flutter-figma-skills
/reload-plugins
```

## Troubleshooting

- **"Figma MCP not responding"** — check `/mcp` status in Claude Code; re-authenticate if needed
- **"figma-asset-export: 401 / 403"** — `FIGMA_TOKEN` not set or expired; regenerate at Figma settings
- **"cwebp: command not found"** — `brew install webp` (Mac) / `apt install webp` (Linux)
- **No Android device for pixel diff** — Phase 4 visual layer falls back to "unchecked" with a note; structure / token / interaction layers still verified
- **Wrong target page detected** — pass explicit `target=<dir-name>` argument to override the auto-snake_case from Figma frame name

## License

See `plugins/flutter-figma-reproduce/LICENSE`.
