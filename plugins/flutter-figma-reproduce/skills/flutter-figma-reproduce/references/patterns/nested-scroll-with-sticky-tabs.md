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

```
NestedScrollView
├─ headerSliverBuilder
│  └─ SliverOverlapAbsorber                                 ← MANDATORY: handle = sliverOverlapAbsorberHandleFor(ctx)
│     └─ SliverAppBar
│        ├─ pinned: true                                     ← AppBar (with toolbar 0 + bottom strip) stays pinned at top after collapse
│        ├─ primary: true                                    ← auto-applies status-bar inset above toolbar; works with extendBodyBehindAppBar
│        ├─ toolbarHeight: 0                                 ← we don't want a Material toolbar — bottom IS our sticky strip
│        ├─ expandedHeight: statusBarInset + bannerH + stripH   ← TOTAL AppBar height when fully expanded
│        ├─ flexibleSpace: FlexibleSpaceBar
│        │   └─ background: Padding(top: statusBarInset, child: Image.asset(banner, fit: cover, alignment: topCenter))
│        │      ↑ status-bar padding here so banner content doesn't render under status-bar icons
│        ├─ bottom: PreferredSize
│        │   └─ preferredSize.height = stripH                ← strip lives here (always at AppBar's bottom edge → naturally pinned)
│        │   └─ child: ColoredBox(pageBg) → strip widget    ← NO status-bar padding needed here (handled by primary:true)
│        └─ backgroundColor: pageBg, elevation: 0, surfaceTintColor: Colors.transparent
└─ body: TabBarView
   └─ controller: <shared TabController>
      for each tab:
        ├─ key: PageStorageKey('cat_<id>')                    ← preserves per-tab scroll position
        └─ CategoryTabContent (Stateful + AutomaticKeepAliveClientMixin) ← keeps state when swiped away
           └─ CustomScrollView
              ├─ SliverOverlapInjector                       ← MANDATORY: handle = sliverOverlapAbsorberHandleFor(ctx)
              └─ SliverGrid / SliverList (the actual content)
```

### Why this structure works (collapse animation = no jumps)

`SliverAppBar` natively interpolates between `expandedHeight` and `collapsedHeight` (= `toolbarHeight + bottom.height + statusBarInset` when `primary: true`) as the user scrolls. The strip in `bottom` rides at the bottom of the AppBar throughout — it never "jumps", just smoothly slides from `bannerEnd` (expanded) to `statusBarInset` (collapsed). This is what makes the sticky transition feel native.

### Why each part exists

| Component | Why |
|---|---|
| `NestedScrollView` | Coordinates outer scroll (AppBar collapse) with inner scrolls (per-tab content). |
| `SliverOverlapAbsorber/Injector` pair | Tells the inner `CustomScrollView` "the outer pinned AppBar reserves N pixels at the top right now"; without this, the inner scroll's first child renders at y=0 in the inner viewport and gets hidden behind the pinned strip. |
| `SliverAppBar.pinned: true` | Keeps the AppBar (toolbar + bottom) visible at the top after collapse — that IS the sticky behavior. |
| `SliverAppBar.primary: true` | Auto-applies device-status-bar inset above the toolbar. With `toolbarHeight: 0` the inset still applies — total collapsed height = `statusBarInset + 0 + bottom.height`. |
| `toolbarHeight: 0` | We don't want a Material toolbar bar; the strip is the only visible bottom-of-AppBar element. |
| `expandedHeight = statusBarInset + bannerH + stripH` | **Total** AppBar height = top inset + banner + strip. Without `statusBarInset` here, banner gets cropped (`expandedHeight` includes everything). |
| `flexibleSpace.background = Padding(top: statusBarInset, child: banner)` | Banner image is rendered inside flexibleSpace, which extends from y=0 to y=(expandedHeight - bottom.height) = (statusBarInset + bannerH). The Padding shifts the banner image down by statusBarInset so its content doesn't render UNDER the status bar icons. (The status bar overlay area shows the banner image's top edge through transparency.) |
| `FlexibleSpaceBar.collapseMode: pin` | Banner stays pinned at its top while the AppBar collapses around it (vs `parallax` which scrolls banner at half-rate). For typical hero banners, `pin` looks correct. |
| `bottom: PreferredSize(stripH, ...)` | Strip is always at the bottom edge of the AppBar. When AppBar is fully expanded, strip sits right under banner (no gap). When fully collapsed, strip is right under the status bar inset — pinned. |
| `TabController` shared between strip + body | Strip click → `controller.animateTo(i)`; body swipe → `controller.index` updates; strip listens to controller and re-paints the selected pill. |
| `PageStorageKey` per tab | Without it, swiping back to a tab resets its scroll to top. PageStorage saves scroll offset per key. |
| `AutomaticKeepAliveClientMixin` | Prevents Flutter from disposing off-screen tabs. Without it, tabs lose their state on swipe-away. |

## Minimal Dart skeleton

```dart
class _ScrollShell extends StatelessWidget {
  const _ScrollShell({required this.tabController, required this.tabs, required this.bannerAsset});

  final TabController tabController;
  final List<TabItem> tabs;
  final String bannerAsset;

  static const double _bannerH = 278;
  static const double _stripH = 30;

  @override
  Widget build(BuildContext context) {
    final statusBarInset = MediaQuery.of(context).padding.top;
    final expandedHeight = statusBarInset + _bannerH + _stripH;
    return NestedScrollView(
      headerSliverBuilder: (ctx, _) => [
        SliverOverlapAbsorber(
          handle: NestedScrollView.sliverOverlapAbsorberHandleFor(ctx),
          sliver: SliverAppBar(
            pinned: true,
            primary: true,
            toolbarHeight: 0,
            expandedHeight: expandedHeight,
            backgroundColor: pageBg,
            surfaceTintColor: Colors.transparent,
            elevation: 0,
            scrolledUnderElevation: 0,
            automaticallyImplyLeading: false,
            flexibleSpace: FlexibleSpaceBar(
              collapseMode: CollapseMode.pin,
              background: Padding(
                padding: EdgeInsets.only(top: statusBarInset),
                child: Image.asset(
                  bannerAsset,
                  fit: BoxFit.cover,
                  alignment: Alignment.topCenter,
                ),
              ),
            ),
            bottom: PreferredSize(
              preferredSize: const Size.fromHeight(_stripH),
              child: ColoredBox(
                color: pageBg,
                child: SizedBox(
                  height: _stripH,
                  child: Padding(
                    padding: const EdgeInsets.symmetric(horizontal: 15),
                    child: _Strip(controller: tabController, tabs: tabs),
                  ),
                ),
              ),
            ),
          ),
        ),
      ],
      body: TabBarView(
        controller: tabController,
        children: [
          for (final t in tabs)
            _TabContent(key: PageStorageKey('tab_${t.id}'), tab: t),
        ],
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
- ✗ `SliverAppBar` without `primary: true` → status-bar inset not applied; pinned strip gets covered by status-bar icons
- ✗ `expandedHeight = bannerH` (forgetting status bar + strip in the formula) → banner cropped because `expandedHeight` is the AppBar's TOTAL height (banner is `expandedHeight − bottom.height − statusBarInset`)
- ✗ `expandedHeight = bannerH + stripH` (forgetting status bar inset) → banner cropped from the bottom by `statusBarInset` (≈28dp on Android)
- ✗ `flexibleSpace.background: Image.asset(banner)` WITHOUT `Padding(top: statusBarInset)` → banner content (text, key visuals near top) gets rendered under the device status bar icons; iOS Figma designs reserve a 44px status-bar zone but on Android (~28px) the offset is wrong
- ✗ Each tab content as a plain `GridView` (no `CustomScrollView` wrapping) → `SliverOverlapInjector` can't be added, scroll syncing breaks
- ✗ Forgetting `BoxFit.cover, alignment: Alignment.topCenter` on the banner image → as AppBar collapses, the banner shrinks; without `topCenter` alignment, the bottom of the banner stays visible while the top scrolls off — opposite of the natural intuition.
- ✗ Loading `banner.webp` / card image at @1x source resolution (e.g. `cwebp -resize 375 278`) → on @2x or @3x DPR devices the image is upscaled and looks blurry. **Always store WebP at @2x or @3x source resolution; let Flutter scale down for display.** Source res check: `python3 -c "from PIL import Image; print(Image.open('asset.webp').size)"` should be ≥ 2× the design size.

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
