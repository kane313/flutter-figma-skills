# Pattern: Banner + Sticky Tabs + Swipeable Content

Common high-frequency layout: a tall hero (banner / cover / profile) at the top, a horizontal tab/category strip below it, and tab-switchable content (grid / list / waterfall) under the tabs. As the user scrolls up, the banner scrolls away (smoothly collapsing) and the tab strip pins to the top of the screen ("吸顶"). When the inner content is back at its top, pulling down further re-expands the banner.

## When to detect & apply this pattern

Trigger on Figma when ALL of:

1. Page contains a top hero region (banner / large image / Y < ~300)
2. Below the hero, a horizontal row of pills / tabs / category labels (Y in 250..350 range, height ~30-50)
3. Below the tabs, repeating card / grid content
4. **AND** the user describes any of: "切 tab"、"分类切换"、"左右滑动"、"上滑吸顶"、"sticky tabs"、"swipe between tabs"、"tab + content"

Without (4) — when the user only says "static layout" — use the simpler `Stack + Positioned` layout.

## Canonical structure

Use `SliverAppBar` (not raw `SliverToBoxAdapter` + `SliverPersistentHeader`). The built-in collapse animation is what gives the smoothest "feels native" sticky-tab transition; rolling your own with two-sliver header introduces height-jump artifacts at the pin moment.

Three subtle requirements need to be satisfied simultaneously:

A. **Banner immersive** — banner extends INTO the status bar area (designer's top-44px decorative gradient is meant to peek through the device status bar)
B. **Zero gap above strip pill in expanded state** — pill flush against banner.bottom
C. **Strip pill avoids status bar in pinned state** — `MediaQuery.padding.top` of clearance above pill so device icons don't overlap

These three constraints CANNOT all be satisfied with static layout. The trick: **interpolate BOTH the strip slot height AND the pill's top padding by `p` (scroll progress 0..1)** so the slot smoothly grows from `pillH` (expanded) to `pillH + statusBarInset` (pinned), the pill's top padding grows from 0 to `statusBarInset`, AND the body content (grid) follows the slot bottom. The "extra 28px" doesn't exist in expanded state at all (slot is just `pillH`), and it's hidden under the device status bar in pinned state. Grid stays glued to pill bottom in both states — no wasted vertical space, no visible gap.

The companion to interpolated slot height: **expandedHeight is also interpolated** — `expandedHeight = bannerHeight + slotHeight` where `slotHeight = pillH + statusBarInset * p`. This means the AppBar's total height grows by `statusBarInset` as the strip pins, which automatically pushes the body (grid) down by that same amount. Looks visually as if "the strip grows out of nowhere" but it's actually "the page header grows by inset to make room for status bar protection of the pill".

```
ValueListenableBuilder<double>(_pinProgress, build: (_, p, _) {
  final extraInset = statusBarInset * p;          ← grows 0 → inset as strip pins
  final slotHeight = pillH + extraInset;          ← grows pillH → pillH+inset
  final expandedHeight = bannerHeight + slotHeight;  ← grows bannerH+pillH → bannerH+pillH+inset
  return NestedScrollView(
    headerSliverBuilder
    └─ SliverOverlapAbsorber                                  ← MANDATORY
       └─ SliverAppBar
          ├─ pinned: true                                      ← AppBar (with toolbar 0 + bottom strip) stays pinned at top
          ├─ primary: false                                    ← banner immersive (extends into status-bar area)
          ├─ toolbarHeight: 0
          ├─ expandedHeight: <interpolated>                    ← grows during pin transition
          ├─ flexibleSpace: FlexibleSpaceBar
          │   ├─ collapseMode: CollapseMode.pin
          │   └─ background: Image.asset(banner, fit: fitWidth, alignment: topCenter)  ← NOT cover; see Anti-patterns
          └─ bottom: PreferredSize
             ├─ preferredSize.height = slotHeight              ← interpolated, NOT static
             └─ child: ColoredBox(pageBg) → Padding(top: extraInset, ...) → strip
    body: TabBarView
})
   └─ controller: <shared TabController>
      for each tab:
        ├─ key: PageStorageKey('cat_<id>')                    ← preserves per-tab scroll position
        └─ CategoryTabContent (Stateful + AutomaticKeepAliveClientMixin) ← keeps state when swiped away
           └─ CustomScrollView
              ├─ SliverOverlapInjector                       ← MANDATORY: handle = sliverOverlapAbsorberHandleFor(ctx)
              └─ SliverGrid / SliverList (the actual content)
```

The parent (`StatefulWidget`) owns a `ValueNotifier<double> _pinProgress` and uses `NotificationListener<ScrollNotification>` to update it as the user scrolls:

```dart
NotificationListener<ScrollNotification>(
  onNotification: (n) {
    if (n.depth == 0) {  // outer scroll only, ignore inner per-tab scrolls
      final p = (n.metrics.pixels / bannerH).clamp(0.0, 1.0);
      if ((p - _pinProgress.value).abs() > 0.005) {
        _pinProgress.value = p;
      }
    }
    return false;
  },
  child: NestedScrollView(...),
)
```

### Why this structure works

`SliverAppBar` natively interpolates its height from `expandedHeight` down to `bottom.height` (the collapsed minimum) as the user scrolls. The strip in `bottom` rides at the AppBar's bottom edge throughout — it never "jumps", just smoothly slides upward as the AppBar collapses.

The `_pinProgress` interpolation (driven by `NotificationListener` + `ValueListenableBuilder`) handles the pill's top padding INSIDE the slot: 0 when banner is fully visible, `statusBarInset` when banner is fully collapsed. This makes the pill SLIDE smoothly within its slot, not JUMP at a discrete pin moment.

### Why each part exists

| Component | Why |
|---|---|
| `NestedScrollView` | Coordinates outer scroll (AppBar collapse) with inner scrolls (per-tab content). |
| `SliverOverlapAbsorber/Injector` pair | Tells the inner `CustomScrollView` "the outer pinned AppBar reserves N pixels at the top right now"; without this, the inner scroll's first child renders at y=0 in the inner viewport and gets hidden behind the pinned strip. |
| `SliverAppBar.pinned: true` | Keeps the AppBar (toolbar + bottom) visible at the top after collapse — that IS the sticky behavior. |
| **`SliverAppBar.primary: false`** | Banner extends into status-bar area (immersive). With `primary: true` Flutter auto-pushes the AppBar down by `statusBarInset`, leaving a `pageBg`-colored strip at top y=0..statusBarInset — looks like an unwanted "white space" above banner. We instead handle status-bar inset MANUALLY on the strip. |
| `toolbarHeight: 0` | We don't want a Material toolbar bar; the strip is the only visible bottom-of-AppBar element. |
| `expandedHeight = bannerH + stripH + statusBarInset` | **Total** AppBar height. The strip's `bottom` widget reserves `stripH + statusBarInset` (slot for pill + inset). Banner gets `expandedHeight - bottom.height = bannerH`. |
| `flexibleSpace.background = Image.asset(banner, fit: cover, alignment: topCenter)` (no Padding wrapper) | Banner image renders from y=0, including the status-bar area. Status bar icons overlay the banner's top decorative gradient. The `topCenter` alignment ensures the banner's top stays anchored as the AppBar collapses. |
| `FlexibleSpaceBar.collapseMode: pin` | Banner stays pinned at its top while the AppBar collapses around it (vs `parallax` which scrolls banner at half-rate). For typical hero banners, `pin` looks correct. |
| `bottom: PreferredSize(stripH + statusBarInset, ...)` | Strip slot is permanently `pillH + statusBarInset` tall. The pill itself is always `stripH` — the extra `statusBarInset` is dynamically positioned via the interpolated Padding (above pill in pinned, below pill in expanded). |
| `ValueListenableBuilder<double>` around the slot's content | Rebuilds the Padding's `top` value as `_pinProgress` changes, smoothly sliding the pill within the slot. |
| `NotificationListener<ScrollNotification>` (depth == 0) | Watches outer-scroll position; computes `p = pixels / bannerH` (0..1) and updates `_pinProgress`. Filtering on `n.depth == 0` is critical — inner per-tab scrolls fire at depth >= 1 and would corrupt the value. |
| `TabController` shared between strip + body | Strip click → `controller.animateTo(i)`; body swipe → `controller.index` updates; strip listens to controller and re-paints the selected pill. |
| `PageStorageKey` per tab | Without it, swiping back to a tab resets its scroll to top. PageStorage saves scroll offset per key. |
| `AutomaticKeepAliveClientMixin` | Prevents Flutter from disposing off-screen tabs. Without it, tabs lose their state on swipe-away. |

## Minimal Dart skeleton

```dart
class _ScrollShell extends StatefulWidget {
  const _ScrollShell({required this.tabController, required this.tabs, required this.bannerAsset});

  final TabController tabController;
  final List<TabItem> tabs;
  final String bannerAsset;

  @override
  State<_ScrollShell> createState() => _ScrollShellState();
}

class _ScrollShellState extends State<_ScrollShell> {
  static const double _bannerH = 278;
  static const double _stripH = 30;

  /// Smooth pin progress: 0 when banner fully visible, 1 when fully pinned.
  /// Interpolated linearly with outer scroll position over banner height.
  /// Drives the pill's top padding so it slides 0 → statusBarInset.
  final ValueNotifier<double> _pinProgress = ValueNotifier(0.0);

  @override
  void dispose() {
    _pinProgress.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    final statusBarInset = MediaQuery.of(context).padding.top;
    final expandedHeight = _bannerH + _stripH + statusBarInset;
    return NotificationListener<ScrollNotification>(
      onNotification: (n) {
        if (n.depth == 0) {
          final p = (n.metrics.pixels / _bannerH).clamp(0.0, 1.0);
          if ((p - _pinProgress.value).abs() > 0.005) {
            _pinProgress.value = p;
          }
        }
        return false;
      },
      child: ValueListenableBuilder<double>(
        valueListenable: _pinProgress,
        builder: (_, p, _) {
          // Interpolate BOTH slotHeight AND expandedHeight by p so the body
          // (grid) follows the strip down as it grows, instead of leaving a
          // wasted 28px gap between pill and grid in the expanded state.
          final extraInset = statusBarInset * p;
          final slotHeight = _stripH + extraInset;
          final expandedHeight = bannerHeight + slotHeight;
          return NestedScrollView(
            headerSliverBuilder: (ctx, _) => [
              SliverOverlapAbsorber(
                handle: NestedScrollView.sliverOverlapAbsorberHandleFor(ctx),
                sliver: SliverAppBar(
                  pinned: true,
                  primary: false,                              // immersive banner
                  toolbarHeight: 0,
                  expandedHeight: expandedHeight,
                  backgroundColor: pageBg,
                  surfaceTintColor: Colors.transparent,
                  elevation: 0,
                  scrolledUnderElevation: 0,
                  automaticallyImplyLeading: false,
                  flexibleSpace: FlexibleSpaceBar(
                    collapseMode: CollapseMode.pin,
                    background: Image.asset(                  // NOT cover, NOT Padding-wrapped
                      widget.bannerAsset,
                      fit: BoxFit.fitWidth,
                      alignment: Alignment.topCenter,
                    ),
                  ),
                  bottom: PreferredSize(
                    preferredSize: Size.fromHeight(slotHeight),
                    child: ColoredBox(
                      color: pageBg,
                      child: Padding(
                        padding: EdgeInsets.only(
                          top: extraInset,
                          left: 15, right: 15,
                        ),
                        child: SizedBox(
                          height: _stripH,
                          child: _Strip(controller: widget.tabController, tabs: widget.tabs),
                        ),
                      ),
                    ),
                  ),
                ),
              ),
            ],
            body: TabBarView(
              controller: widget.tabController,
              children: [
                for (final t in widget.tabs)
                  _TabContent(key: PageStorageKey('tab_${t.id}'), tab: t),
              ],
            ),
          );
        },
      ),
    );
  }
}

class _TabContent extends StatefulWidget {
  const _TabContent({super.key, required this.tab});
  final TabItem tab;
  @override
  State<_TabContent> createState() => _TabContentState();
}

class _TabContentState extends State<_TabContent> with AutomaticKeepAliveClientMixin {
  @override bool get wantKeepAlive => true;

  @override
  Widget build(BuildContext context) {
    super.build(context);
    return CustomScrollView(
      slivers: [
        SliverOverlapInjector(handle: NestedScrollView.sliverOverlapAbsorberHandleFor(context)),
        // ... your SliverPadding + SliverGrid / SliverList here ...
      ],
    );
  }
}
```

## Anti-patterns to AVOID (will compile but break behavior)

- ✗ `Stack` + `Positioned(top: scrollOffset)` manually moving widgets on scroll → janky, no fling physics, hit-test gaps
- ✗ `SingleChildScrollView` containing the banner + a `ListView`/`GridView` → "unbounded height" assertion or nested scroll conflict
- ✗ `CustomScrollView` with `SliverAppBar` but body is `Column(TabBar + TabBarView)` → TabBarView needs bounded height, breaks with sliver scroll
- ✗ Putting the tab strip in a normal `SliverToBoxAdapter` (not pinned in any way) → strip scrolls away with banner, no 吸顶
- ✗ `SliverAppBar(pinned: true)` without `SliverOverlapAbsorber` + matching `SliverOverlapInjector` in the body → scroll position desyncs, content jumps behind the pinned strip
- ✗ `SliverAppBar` with `primary: true` AND a banner that is supposed to be immersive → `primary: true` auto-pushes the AppBar down by `statusBarInset`, leaving a `pageBg`-colored strip at viewport y=0..statusBarInset above the banner. Looks like an unwanted whitespace at the top. **Use `primary: false`** and handle status-bar inset manually on the strip's content (interpolated padding).
- ✗ Wrapping `flexibleSpace.background` in `Padding(top: statusBarInset)` → with `primary: false`, the AppBar already starts at y=0 and the banner image gets rendered from y=0 immersively. Adding the Padding pushes the banner DOWN by inset, leaving a `pageBg`-colored strip ABOVE the banner image (the same whitespace bug as `primary: true`). **Just `Image.asset(...)` directly inside `background:` — no Padding wrapper.**
- ✗ `expandedHeight = bannerH` (forgetting status bar + strip in the formula) → banner cropped because `expandedHeight` is the AppBar's TOTAL height. Use `bannerH + stripH + statusBarInset`.
- ✗ Static `Padding(top: statusBarInset)` on the strip's pill (always-on inset) → 28px gap between banner.bottom and pill in expanded state — strip looks like it's floating above where it should be pinned to. **Use `ValueListenableBuilder<double>` driven by `NotificationListener` to interpolate the padding from 0 → statusBarInset as scroll progresses 0 → bannerHeight.**
- ✗ Interpolating only the pill's top padding while keeping the strip slot height STATIC at `pillH + statusBarInset` → pill correctly slides from top to bottom of slot, BUT the slot itself reserves `inset` of unused space below the pill in expanded state. Grid renders 28px lower than the design intends — designer-eyes notice the extra spacing between pill and first row of cards. **Interpolate BOTH `slotHeight` AND `expandedHeight` together** (`slotHeight = pillH + statusBarInset * p`, `expandedHeight = bannerH + slotHeight`) so the grid follows the slot down as it grows during the pin transition. No wasted vertical space in either state.
- ✗ Each tab content as a plain `GridView` (no `CustomScrollView` wrapping) → `SliverOverlapInjector` can't be added, scroll syncing breaks
- ✗ Forgetting `alignment: Alignment.topCenter` on the banner image → as AppBar collapses, the banner shrinks; without `topCenter` alignment, the bottom of the banner stays visible while the top scrolls off — opposite of the natural intuition.
- ✗ Using `BoxFit.cover` on the banner image with a fixed `bannerHeight` constant (e.g. 278) → on devices wider than the design width (e.g. Android at 387dp vs design 375dp), `cover` scales the image up to fill the wider container, growing the height beyond `bannerHeight` and cropping the top + bottom (`(287 − 278)/2 ≈ 4.5px each`). Visually the banner contents look "amplified" 3-5% — characters appear bigger, decorative elements overflow toward the edges, and the bottom alpha-mask region (if any) gets cropped out, killing the soft fade-to-page-bg effect designers usually bake into hero banners. **Use `BoxFit.fitWidth` and compute `bannerHeight = deviceWidth × imageRatio`** so the image renders at exactly its native aspect on every device width, preserving alpha and avoiding crop:
  ```dart
  final deviceWidth = MediaQuery.of(context).size.width;
  final bannerHeight = deviceWidth * bannerImgHeight / bannerImgWidth;
  // expandedHeight = bannerHeight + stripH + statusBarInset
  // FlexibleSpaceBar background: Image.asset(banner, fit: BoxFit.fitWidth, alignment: Alignment.topCenter)
  ```
- ✗ **Lossy WebP for hero banners or photographic cards** (`cwebp -q 90` etc.) → even at quality 90, WebP introduces visible artifacts on gradient-rich content (banners with soft pink fades) and human faces (slight smoothing/blurring). **Default to PNG for hero / photographic assets**; reserve WebP for icons or non-hero images where compression artifacts won't show. Critically, **WebP conversion can drop the alpha channel** (depending on encoder settings) — if your hero PNG has a designer-baked fade-to-transparent mask at the edges, a careless WebP conversion can erase it entirely, leaving you with a hard-edged banner that "overflows" its visual region. Test with `python3 -c "from PIL import Image; im=Image.open('asset.webp'); print(im.mode, im.getchannel('A').getextrema() if im.mode == 'RGBA' else 'NO ALPHA')"`.
- ✗ Storing PNG/WebP at @1x source resolution (e.g. `cwebp -resize 375 278` from a 750px source) → on @2x or @3x DPR devices the image is upscaled and looks blurry. **Always store image assets at @2x or @3x source resolution; let Flutter scale down for display.** Source-res check: `python3 -c "from PIL import Image; print(Image.open('asset.png').size)"` should be ≥ 2× the design size.

## Why NOT a two-sliver header (`SliverToBoxAdapter` + `SliverPersistentHeader`)?

A natural-feeling alternative is to put banner in `SliverToBoxAdapter` (non-pinned) and the strip in a separate `SliverPersistentHeader(pinned: true)`. This is APPEALING because each region owns exactly its design height with no formula to remember.

But it has the **status-bar-inset dilemma** that has no clean fix:

- The pinned strip needs `MediaQuery.padding.top` of empty space ABOVE the pill so device status-bar icons don't overlay it when pinned
- That same inset becomes a visible 28px gap between banner.bottom and pill in the expanded state — strip looks like it's floating, not "stuck under banner"

Workarounds that DON'T solve it cleanly:
- Static `Padding(top: inset)` always: visible gap in expanded state ✗
- Alignment toggle (top in expanded, bottom in pinned), fixed slot of `pillH + inset`: fixes gap above pill, but adds gap BELOW pill in expanded state (28px empty pink wash before grid's own padding) ✗
- Dynamic slot height via `ValueListenableBuilder` rebuilding the delegate with different `min/maxExtent`: at the moment of pinning, slot grows from `pillH` to `pillH + inset` — a noticeable 28px content jump ✗

`SliverAppBar`'s native interpolation between `expandedHeight` and the auto-computed collapsed height (`toolbarHeight + bottom.height + statusBarInset`) sidesteps all of this — the strip rides at the AppBar's bottom edge throughout collapse with no jumps.

(The dilemma + workarounds were debugged through 5 iterations on the home page in this skill's own dev process. Sparing future implementations the same loop is the entire reason this section exists.)

## Variants worth knowing

- **Multi-section sticky** (multiple sticky strips at different scroll depths): use multiple `SliverPersistentHeader(pinned: true)` instead of `SliverAppBar`. Less Material-y, more flexible. Each strip needs its own `SliverOverlapAbsorber`.
- **Snap-to-tab on collapse**: set `floating: true, snap: true` on the SliverAppBar; when partially collapsed, it snaps to fully expanded or fully collapsed. Good for messy mid-state UX.
- **Header parallax**: use `FlexibleSpaceBar.collapseMode: parallax` instead of `pin` — banner scrolls at half-rate as AppBar collapses. Looks fancy but can disorient on long banners.
- **Pull-to-refresh**: wrap each tab's `CustomScrollView` (NOT the outer NestedScrollView) in `RefreshIndicator(onRefresh: ...)`. Works with the inner scroll.

## Real-case validation

This pattern was implemented for the `home` page in this skill's own dev process (Figma node `1:768`, 萌宝相机 首页). After 5 iterations debugging the status-bar-inset dilemma (ultimately resolved by using `SliverAppBar` instead of two-sliver header), real-device validation on Android (DCO AL00) confirmed:

- Vertical scroll up → banner smoothly collapses → categories pin at top with status-bar inset auto-applied (no jumps) ✓
- Horizontal swipe on body → tab switches, strip pill highlight follows ✓
- Tap on strip pill → body animates to that tab ✓
- Pull down at top of inner scroll → AppBar smoothly re-expands, banner re-appears ✓
- Per-tab scroll position preserved across swipes (PageStorageKey + KeepAlive) ✓
- Banner image renders sharp (source webp stored at @2x = 750×556, not @1x) ✓

See `<flutter-project>/lib/pages/home/home_page.dart` in any project that ran this skill on the home page for a reference implementation.
