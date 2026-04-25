---
name: flutter-figma-reproduce
description: "Reproduces a Flutter UI page from a Figma design with high fidelity via a 5-phase pipeline: align tokens + scan project, build skeleton widget tree, fill styles from the align table, add multi-state + interaction + animations, and verify via a layered scorecard. Orchestrates 8 helper tool skills (align-table, autolayout-to-flex, asset-export, four-states-scaffold, prototype-animation, theme-gap-filler, widget-tree-review, pixel-diff). Use when starting UI reproduction from a Figma URL, or when the user says 'reproduce this page', 'implement this design', 'build from Figma'. Supports safe mode (checkpoint per phase), fast mode (auto-chain mid-phases), and auto mode (no pauses, run end-to-end). Requires a Flutter project, Figma MCP logged in, and FIGMA_TOKEN env var for asset export."
---

# flutter-figma-reproduce

## When to use

- User provides a Figma URL + wants to implement it as a Flutter page
- Keywords: 还原 / 实现 / 写个页面 / 按设计稿做 / 按 Figma 做 / 按这个设计图做 / 对着这个做 / reproduce / implement / build
- Auto-triggers when: message contains `figma.com/design/...` OR `figma.com/make/...` URL AND current workdir has `pubspec.yaml` with `flutter:` dep

## Inputs

- `figma_url` (required) — Figma URL (design or make), or fileKey + nodeId
- `mode` (optional, default: `safe`) — `safe` for checkpoint-per-phase, `fast` for auto-chain mid-phases, `auto` for end-to-end no-pause execution. Mode can also be auto-elevated by trigger words in the user's prompt (see "Auto-mode trigger detection" below).
- `target` (optional) — page dir name under `lib/pages/`; default: Figma frame name → snake_case
- `reuse_components` (optional, default: `auto`) — scan `lib/widgets/`; pass a path to override, `none` to skip
- `design_baseline` (optional, default: `375x812`) — base resolution for `flutter_screenutil` scaling. All dimensional values in generated code use `.w / .h / .sp / .r` suffixes against this baseline. The skill auto-installs `flutter_screenutil` if missing (with user consent at Phase 0 checkpoint, OR using the default `yes` in auto mode).

## Auto-mode trigger detection

Before reading any phase reference, scan the user's invoking prompt for any of these trigger phrases (case-insensitive substring match). If ANY match is found, **upgrade the mode to `auto` regardless of the explicit `mode=` argument**. Announce the elevation: "Auto mode triggered by phrase: '<matched phrase>' — will run end-to-end without checkpoint pauses."

**Chinese triggers:**
- `不再询问` / `不要询问` / `不用再询问` / `不用问` / `别问了`
- `连续执行` / `自动执行` / `自动跑` / `自动走完` / `一口气` / `一气呵成` / `跑完为止` / `跑到底` / `一路` / `直接执行`

**English triggers:**
- `continue without asking` / `no checkpoint` / `no checkpoints` / `no prompts` / `skip prompts` / `skip checkpoints`
- `autopilot` / `auto mode` / `run to completion` / `end to end` / `all the way` / `non-stop` / `nonstop`
- `yolo` / `just do it` / `don't ask` / `dont ask`

Example: user says "还原这个页面，不用再询问我了" → auto-elevate even if no `--mode=auto` flag.

If the user later asks to "switch back to checkpoints" / "暂停一下" mid-run, downgrade to safe mode for the remaining phases.

## The 5-phase pipeline (overview)

| Phase | Name | Primary tool skill | Safe | Fast | Auto |
|---|---|---|---|---|---|
| 0 | Align | `figma-flutter-align-table` (+ `flutter-theme-gap-filler` post-checkpoint) | Mandatory | Mandatory | Auto-defaults |
| 1 | Skeleton | `figma-autolayout-to-flex` (+ auto structural review via `flutter-widget-tree-review`) | Mandatory | Auto | Auto |
| 2 | Style | `figma-asset-export` for assets; manual token binding from align table | Optional | Optional | Auto |
| 3 | State & Interaction | `flutter-four-states-scaffold` + `figma-prototype-to-flutter-animation` | Mandatory | Mandatory | Auto-defaults |
| 4 | Verify | `figma-flutter-pixel-diff` + scorecard aggregation | Mandatory | Mandatory | Auto-report |

Detailed phase orchestration: read `references/0N-<phase>.md` (one file per phase) when entering that phase. DO NOT read all phases upfront.

## Checkpoint mechanism

Safe mode (default) — at the end of each mandatory checkpoint phase:

1. Claude writes phase output files to `.figma-reproduce/<page>/`
2. Claude emits a CHECKPOINT marker message listing approval items
3. Claude waits for the user to reply one of: `continue` / `ok` / `过` / `next` (approve) OR any other content (interpreted as modification/question → respond then re-emit CHECKPOINT)
4. Only after approval, advance to next phase

Fast mode — skips checkpoint at Phase 1 and Phase 2 only. Phase 0 and Phase 3/4 checkpoints still required. On Phase 4 failure, report blockers but do NOT auto-fallback to safe mode (user decides next action).

**Auto mode override (mandatory rule)** — when mode is `auto` (whether explicit or trigger-elevated), every step in `references/0N-*.md` that says "wait for user to reply continue" / "checkpoint mandatory" / "wait for approval" is replaced by:

1. Write phase output files normally
2. Log a one-line `[AUTO] Phase N done: <key decisions taken>` to the chat (replaces the full CHECKPOINT block)
3. Apply sensible defaults for any user-decision points:
   - Phase 0 gaps.md: use the `[x]` marked recommendations (do NOT wait for user to edit)
   - Phase 0 ScreenUtil bootstrap prompt: assume `y` (install)
   - Phase 0 component-map.md: use the `[x]` marked default reuse decisions
   - Phase 3 interaction checklist: mark all items as "auto-deferred" (not verified by user) and proceed
   - Phase 4 verdict: write the report and continue; do NOT ask "accept or revisit"
4. Advance to next phase immediately

After the full run, emit ONE final summary block listing: phases completed, defaults taken, any blockers encountered, paths to all generated files. This is the only user-facing pause in auto mode (and even this is informational, not a prompt — Claude does not wait for a reply).

**Auto mode does NOT skip mechanical pre-flight checks.** Removing the user pause does NOT remove Claude's responsibility to run automated checks that prevent known failure modes. The following checks are MANDATORY in auto mode (in addition to safe/fast mode):

- **Asset content pre-flight** — before treating any Figma node as "an icon" or "a single asset", inspect the node's metadata subtree. If the node is a Figma component instance with multiple children (icon child + text child + decoration), the export will bake all children into one image — including any text labels. Re-rendering that text in Flutter creates duplicate text. Pre-flight algorithm:
  1. For every Figma node ID flagged for asset export, call `mcp__figma__get_metadata` (or read it from the already-fetched parent metadata) to enumerate children.
  2. If `children.length > 1` AND any child is a `text` / `Text` node → the asset includes a baked text label.
  3. Either (a) re-export only the icon child node (drop the parent ID), OR (b) skip the Flutter-side `Text` widget that would duplicate it.
  4. Record the decision in `.figma-reproduce/<page>/align-table.md` "Assets" section: "exported X (icon-only child of parent Y)".
  5. For SVG assets (already in project or newly exported): run `head -c 800 <svg>` + `grep -c '<path' <svg>` per `references/02-style.md` rules. Same logic — multi-path SVGs with rounded-rect outer shapes are full components, not icons.

Skipping this check in auto mode produces the failure recorded in `lessons-learned.md` of the 2026-04-25 login-page run (duplicate "萌宝相机" + duplicate "游客登录" text rendered both in the baked PNG and the Flutter Text widget). The fix cost a Phase 4 rebuild — cheap by itself but compounds when bulk-reproducing many pages.

**Mid-run mode change**: if the user interrupts an auto-mode run with content suggesting they want to review (`等等` / `wait` / `pause` / `让我看看` / `暂停`), downgrade to safe mode immediately and emit the next checkpoint as if it had been safe all along.

## Second-run handling (same Figma node)

Detected when `.figma-reproduce/<page>/state.json` exists from a prior run. Read `references/00-align.md` § "Second run" for the diff strategy (compare Figma snapshot hashes, produce change list, update targeted widgets; code regions tagged with `// @figma-reproduce:keep` are preserved).

## Session-scoped state

All intermediate artifacts live under `.figma-reproduce/<page>/` in the Flutter project:

- `config.md` — this run's parameters
- `align-table.md`, `gaps.md`, `component-map.md` — Phase 0 output (produced by `figma-flutter-align-table`)
- `autolayout-<node>.md` — Phase 1 translation output (produced by `figma-autolayout-to-flex`)
- `verify-report.md` — Phase 4 final scorecard
- `state.json` — Figma node snapshot for second-run diff

Templates for files owned by the main skill (not produced by sub-skills): `templates/config.md.tmpl`, `templates/verify-report.md.tmpl`, `templates/skeleton-prompt.md.tmpl`.

## Cross-cutting references (read as needed)

- `references/senior-knowledge.md` — what a senior dev looks at when opening Figma; consult in Phase 0/1
- `references/failure-modes.md` — AI's common mistakes in Flutter code gen; consult at every phase start
- `references/fastlane.md` — fast-mode and auto-mode rules (exact behavior differences vs safe)
- `references/patterns/` — canonical compound layouts (e.g. banner + sticky tabs + swipeable content). Consult in Phase 1 BEFORE designing layout from scratch — see `01-skeleton.md` § "Common patterns library" for triggers.
- `tools/scorecard-checker.md` — Token layer & structure layer automatic check algorithm (used by every code-generating phase)

## Constraints

- **Never skip Phase 0 checkpoint** — the align table + gaps + component map are the contract for the whole run
- **Never emit Dart code directly in this skill** — delegate to tool skills; this skill only orchestrates
- **Never modify files outside `.figma-reproduce/<page>/` or `lib/pages/<target>/`** without explicit user acknowledgement
- **Fast mode is not a "quality mode"** — same scorecard thresholds apply; only checkpoints are skipped
