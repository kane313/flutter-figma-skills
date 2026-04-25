# flutter-figma-reproduce

A Claude Code plugin that standardizes the **Flutter UI reproduction from Figma** workflow via a 5-phase orchestrated pipeline with layered scorecard verification.

> Install via the parent marketplace: see [../../README.md](../../README.md).

## Skills (9)

### Orchestrator

- **`flutter-figma-reproduce`** — main entry point. Drives the 5-phase pipeline, applies skill-level rules (immersive defaults, asset pre-flight, mode handling), and writes per-page artifacts to `.figma-reproduce/<page>/`.

### Helper sub-skills (invoked by the orchestrator)

- **`figma-flutter-align-table`** — scans the Figma node tree and the project theme, produces `align-table.md` (token mapping), `gaps.md` (new tokens needing decisions), and `component-map.md` (reuse opportunities).
- **`figma-autolayout-to-flex`** — translates Figma Auto Layout into Flutter widget tree intent (Row/Column/Stack/Wrap + alignment).
- **`figma-asset-export`** — downloads images / icons via Figma REST API, transcodes PNG → WebP via `cwebp`, patches `pubspec.yaml`.
- **`flutter-four-states-scaffold`** — generates a sealed-class state machine (loading / empty / error / success) plus `_mock_data.dart`.
- **`figma-prototype-to-flutter-animation`** — reads Figma Prototype interactions and emits matching Flutter animation snippets (AnimatedContainer / Hero / PageRouteBuilder + Curves).
- **`flutter-theme-gap-filler`** — writes user-approved token gaps from `gaps.md` into `lib/theme/app_theme.dart` (ColorScheme / TextTheme / ThemeExtension).
- **`flutter-widget-tree-review`** — structural review of generated widget tree against the Figma node, catches Stack-vs-Row misreads and similar pitfalls.
- **`figma-flutter-pixel-diff`** — captures the running Flutter app via Patrol or `adb screencap`, downloads the corresponding Figma screenshot via MCP, computes pixel-wise BT.601 luminance-weighted diff with heatmap.

## Pipeline

| Phase | Goal | Primary skill | Checkpoint (safe / fast / auto) |
|---|---|---|---|
| 0 Align | tokens + gaps + component map | `figma-flutter-align-table` (+ `flutter-theme-gap-filler`) | mandatory / mandatory / auto-defaults |
| 1 Skeleton | structural widget tree | `figma-autolayout-to-flex` (+ `flutter-widget-tree-review`) | mandatory / auto / auto |
| 2 Style | colors / typography / assets | `figma-asset-export` | optional / optional / auto |
| 3 State & Interaction | 4-state machine + animations + splash | `flutter-four-states-scaffold` (+ `figma-prototype-to-flutter-animation`) | mandatory / mandatory / auto-defaults |
| 4 Verify | layered scorecard | `figma-flutter-pixel-diff` | mandatory / mandatory / auto-report |

Detailed orchestration per phase: `skills/flutter-figma-reproduce/references/0N-<phase>.md`.

## Modes

- `safe` (default) — checkpoint after every mandatory phase, wait for user `continue`.
- `fast` — auto-advance through Phase 1 / 2; still pause at 0 / 3 / 4.
- `auto` — no pauses, single info summary at end. Triggered explicitly via `--mode=auto` OR by phrases like `不用询问`, `yolo`, `autopilot`, `跑完为止` in the user prompt.

Mode-specific behavior table: `skills/flutter-figma-reproduce/references/fastlane.md`.

## Per-page artifacts

The orchestrator writes everything to the consuming Flutter project:

```
<flutter-project>/
├── lib/pages/<target>/<target>_page.dart    # generated page
├── lib/pages/<target>/_mock_data.dart       # state mocks
├── assets/images/<...>.webp                 # exported images
├── assets/icons/<...>.svg                   # exported icons
└── .figma-reproduce/<target>/
    ├── config.md                            # this run's parameters
    ├── align-table.md
    ├── gaps.md
    ├── component-map.md
    ├── state.json                           # for second-run diff
    └── verify-report.md
```

## Defensive rules

The orchestrator enforces a growing catalog of rules to prevent known failure modes — see `skills/flutter-figma-reproduce/references/failure-modes.md`. Selection:

- Status bar: never use `SystemUiOverlayStyle.dark` preset; emit explicit struct with `Colors.transparent`
- Full-bleed bg: lift outside `Center+SizedBox(width:N)` — always
- Chinese text in fixed-width: wrap in `Flexible+FittedBox(scaleDown)+softWrap:false`
- Asset pre-flight: enumerate Figma node children before exporting; if any child is `text`, prefer programmatic render or export a child node ID
- Skeleton-artifact audit: Phase 2 must delete every widget that has no Figma counterpart
- SVG asset content inspection: full-component SVGs (rounded-rect outer + inner) render at native size, not wrapped in additional decoration

## License

See [LICENSE](LICENSE).
