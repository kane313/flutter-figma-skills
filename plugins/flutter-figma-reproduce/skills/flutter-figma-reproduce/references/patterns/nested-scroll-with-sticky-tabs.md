# Pattern: Banner + Sticky Tabs + Swipeable Content

Common high-frequency layout: a tall hero (banner / cover / profile) at the top, a horizontal tab/category strip below it, and tab-switchable content (grid / list / waterfall) under the tabs. As the user scrolls up, the banner scrolls away and the tab strip pins to the top of the screen ("吸顶"). When the inner content is back at its top, pulling down further re-expands the banner.

## When to detect & apply this pattern

Trigger on Figma when ALL of:

1. Page contains a top hero region (banner / large image / Y < ~300)
2. Below the hero, a horizontal row of pills / tabs / category labels (Y in 250..350 range, height ~30-50)
3. Below the tabs, repeating card / grid content
4. **AND** the user describes any of: "切 tab"、"分类切换"、"左右滑动"、"上滑吸顶"、"sticky tabs"、"swipe between tabs"、"tab + content"

Without (4) — when the user only says "static layout" — use the simpler `Stack + Positioned` layout.

## Anti-patterns to AVOID (will compile but break behavior)

- ✗ `Stack` + `Positioned(top: scrollOffset)` manually moving widgets on scroll → janky, no fling physics, hit-test gaps
- ✗ `SingleChildScrollView` containing the banner + a `ListView`/`GridView` → "unbounded height" assertion or nested scroll conflict
- ✗ `CustomScrollView` with `SliverAppBar` but body is `Column(TabBar + TabBarView)` → TabBarView needs bounded height, breaks with sliver scroll
- ✗ Putting the tab strip in a normal `SliverToBoxAdapter` (not `pinned`) → strip scrolls away with banner, no 吸顶
- ✗ `SliverAppBar(pinned: true)` without `SliverOverlapAbsorber` + matching `SliverOverlapInjector` in the body → scroll position desyncs between outer and inner, content jumps
- ✗ Forgetting `MediaQuery.padding.top` in the pinned tab strip → tabs get covered by the device status bar after collapse
- ✗ Each tab content as a plain `GridView` (no `CustomScrollView` wrapping) → `SliverOverlapInjector` can't be added, scroll syncing breaks
- ✗ Using `SliverAppBar(pinned: true, flexibleSpace: banner, bottom: stripPreferredSize)` as the header sliver. Three cascading failures: (a) `expandedHeight=bannerH` silently crops the banner because `expandedHeight` includes `bottom.height + toolbarHeight`; (b) the "fix" `expandedHeight = bannerH + stripH + statusBarInset` correctly preserves banner height BUT pushes the entire layout down ~58px, costing the user one full row of below-fold content (the design intent is for the strip to sit AT or slightly OVERLAP the banner's bottom edge, not strictly below it); (c) `bottom` renders strictly below the banner — you can't reproduce a Figma design where the strip Y overlaps the banner's max-Y by 1-3px. **Use `SliverToBoxAdapter(banner) + SliverPersistentHeader(pinned, delegate=strip)` as separate header slivers instead** — see the "Why this structure" subsection below.

## Canonical structure

```
NestedScrollView
├─ headerSliverBuilder (returns LIST of slivers ABOVE the inner body)
│  ├─ SliverToBoxAdapter                                        ← non-pinned banner; scrolls away naturally
│  │  └─ SizedBox(height: bannerDesignHeight)
│  │     └─ Image.asset(banner, fit: BoxFit.cover, alignment: Alignment.topCenter)
│  └─ SliverOverlapAbsorber                                     ← MANDATORY: only the PINNED strip needs absorbing
│     └─ SliverPersistentHeader
│        ├─ pinned: true                                         ← strip stays at top after banner scrolls past
│        └─ delegate: _CategoriesHeaderDelegate
│           ├─ minExtent = maxExtent = stripH + statusBarInset   ← strip + bottom-of-system-status-bar padding
│           └─ build: ColoredBox(pageBg) → Padding(top: inset) → strip widget
└─ body: TabBarView
   └─ controller: <shared TabController>
      for each tab:
        ├─ key: PageStorageKey('cat_<id>')                       ← preserves per-tab scroll position
        └─ CategoryTabContent (Stateful + AutomaticKeepAliveClientMixin) ← keeps state when swiped away
           └─ CustomScrollView
              ├─ SliverOverlapInjector                           ← MANDATORY: handle = sliverOverlapAbsorberHandleFor(ctx)
              └─ SliverGrid / SliverList (the actual content)
```

### Why this structure (and NOT `SliverAppBar` + `bottom`)

The naive instinct is `SliverAppBar(pinned: true, expandedHeight: bannerH, flexibleSpace: banner, bottom: PreferredSize(strip))`. **Don't do this.** Three failure modes:

1. `SliverAppBar.expandedHeight` includes `toolbarHeight + bottom.height` — your banner silently shrinks to `expandedHeight - bottom.height`, cropping the bottom of the design (CTA buttons, decorations).
2. The "fix" of `expandedHeight = bannerH + stripH + statusBarInset` correctly preserves banner height BUT adds `stripH + statusBarInset` to the total page header height — the strip no longer overlays the banner's bottom edge as designed in Figma; the page is pushed down ~58px and the user sees one fewer row of content.
3. `SliverAppBar.bottom` always renders at the bottom of the AppBar, so when fully expanded it appears just below the banner — fine. But the natural Figma layout often has the strip overlapping the banner's bottom 1-3px (designer puts strip Y near the banner's max-Y). With `bottom`, you can't reproduce that overlap; the strip is always strictly below the banner.

The two-sliver approach (`SliverToBoxAdapter` + `SliverPersistentHeader(pinned)`) avoids all three by letting each region own exactly its design dimensions.

### Why each part exists (so AI can adapt the pattern, not copy blindly)

| Component | Why |
|---|---|
| `NestedScrollView` | Coordinates outer scroll (banner/header area) with inner scrolls (per-tab content). |
| `SliverToBoxAdapter` for banner | Renders the banner at its EXACT design height. No `bottom`-style cropping. As user scrolls up, the banner naturally scrolls off (no `pinned`). When user scrolls back to the top of the inner content and continues to drag down, NestedScrollView's outer scroll takes the gesture and the banner slides back into view. |
| `Image(alignment: Alignment.topCenter)` | If/when the SliverToBoxAdapter's height is constrained smaller than the image's natural ratio, top-anchored alignment ensures the banner's top stays put while the bottom is the part to clip — matches the intuition that "banner scrolls up". |
| `SliverOverlapAbsorber/Injector` pair | Tells the inner `CustomScrollView` "the outer pinned header occupies N pixels right now"; without this, the inner scroll's first child renders at y=0 in the inner viewport, but the outer pinned strip overlaps it → visible content is hidden behind the strip. |
| `SliverPersistentHeader(pinned: true)` for strip | Stays glued to the top of the viewport once banner has fully scrolled past. The delegate's `minExtent` controls the pinned height (== strip + statusBarInset). |
| `minExtent == maxExtent` in delegate | Strip is fixed-height (no shrink-on-scroll); we just want it to either fully show or fully scroll into the pinned position. |
| Status bar padding INSIDE the delegate's `build` | The strip pins to the very top of the viewport when the user has scrolled past the banner. Without `Padding(top: MediaQuery.padding.top)`, the strip starts at y=0 (under the device status bar icons). With it, the strip sits below the status bar. The `minExtent`/`maxExtent` must include this same inset so the layout reserves the right space. |
| `TabController` shared between strip + body | Strip click → `controller.animateTo(i)`; body swipe → `controller.index` updates; strip listens to controller and re-paints the selected pill. |
| `PageStorageKey` per tab | Without it, swiping back to a tab resets its scroll to top. PageStorage saves scroll offset per key. |
| `AutomaticKeepAliveClientMixin` | Prevents Flutter from disposing off-screen tabs. Without it, tabs lose their state on swipe-away. |

### "Pull down to reveal banner" behavior — comes for free

`NestedScrollView` + `SliverAppBar(pinned: true)` automatically routes scroll gestures:
- When inner CustomScrollView is at offset 0 AND user drags down → outer position takes the gesture → SliverAppBar expands → banner reappears.
- Once user is mid-content, drags do NOT re-expand the banner (would be jarring).

No extra code needed. **If user wants explicit pull-to-refresh on top, wrap a tab content in `RefreshIndicator(onRefresh: ...)`** — that's a separate gesture from the overscroll re-expand.

## Minimal Dart skeleton

```dart
class _ScrollShell extends StatelessWidget {
  const _ScrollShell({required this.tabController, required this.tabs, required this.bannerAsset});

  final TabController tabController;
  final List<TabItem> tabs;
  final String bannerAsset;

  static const double _bannerH = 278;   // Figma banner design height
  static const double _stripH = 30;     // Figma tab-strip design height

  @override
  Widget build(BuildContext context) {
    final statusBarInset = MediaQuery.of(context).padding.top;
    return NestedScrollView(
      headerSliverBuilder: (ctx, _) => [
        // Banner — non-pinned, owns exactly its design height. Scrolls off
        // naturally as user swipes up. Slides back when the inner scroll
        // is at top and user drags down (NestedScrollView default behavior).
        SliverToBoxAdapter(
          child: SizedBox(
            height: _bannerH,
            child: Image.asset(
              bannerAsset,
              fit: BoxFit.cover,
              alignment: Alignment.topCenter,
            ),
          ),
        ),
        // Sticky tab strip — pinned via SliverPersistentHeader. Wrapped in
        // SliverOverlapAbsorber so the inner CustomScrollView knows how
        // much vertical space is reserved for it.
        SliverOverlapAbsorber(
          handle: NestedScrollView.sliverOverlapAbsorberHandleFor(ctx),
          sliver: SliverPersistentHeader(
            pinned: true,
            delegate: _StripHeaderDelegate(
              tabController: tabController,
              tabs: tabs,
              statusBarInset: statusBarInset,
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

class _StripHeaderDelegate extends SliverPersistentHeaderDelegate {
  _StripHeaderDelegate({
    required this.tabController,
    required this.tabs,
    required this.statusBarInset,
  });

  final TabController tabController;
  final List<TabItem> tabs;
  final double statusBarInset;

  static const double _stripH = 30;

  @override
  double get minExtent => _stripH + statusBarInset;
  @override
  double get maxExtent => _stripH + statusBarInset;

  @override
  Widget build(BuildContext context, double shrinkOffset, bool overlapsContent) {
    return ColoredBox(
      color: pageBg,
      child: Padding(
        padding: EdgeInsets.only(top: statusBarInset, left: 15, right: 15),
        child: SizedBox(
          height: _stripH,
          child: _Strip(controller: tabController, tabs: tabs),
        ),
      ),
    );
  }

  @override
  bool shouldRebuild(covariant _StripHeaderDelegate old) =>
      old.tabController != tabController || old.statusBarInset != statusBarInset;
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

class _Strip extends StatelessWidget {
  const _Strip({required this.controller, required this.tabs});
  final TabController controller;
  final List<TabItem> tabs;

  @override
  Widget build(BuildContext context) {
    return ListView.separated(
      scrollDirection: Axis.horizontal,
      padding: EdgeInsets.zero,
      itemCount: tabs.length,
      separatorBuilder: (_, _) => const SizedBox(width: 12),
      itemBuilder: (_, i) => _Pill(
        label: tabs[i].label,
        selected: i == controller.index,
        onTap: () => controller.animateTo(i),
      ),
    );
  }
}
```

The `TabController` must be created in the parent's `initState` with `vsync: this` (parent uses `TickerProviderStateMixin`). Add a listener that calls `setState` (guarded by `if (!controller.indexIsChanging)`) so the strip rebuilds with the new selection when the user swipes.

## Variants worth knowing

- **Multi-section sticky** (multiple sticky strips at different scroll depths) — use multiple `SliverPersistentHeader(pinned: true)` instead of `SliverAppBar`. Less Material-y, more flexible.
- **Snap-to-tab on collapse** — set `floating: true, snap: true` on the SliverAppBar; when partially collapsed, it snaps to fully expanded or fully collapsed. Good for messy mid-state UX.
- **Header parallax** — wrap `flexibleSpace.background` in a transform that uses `_ScrollController.offset` for a subtle parallax effect. Optional.
- **Pull-to-refresh** — wrap each tab's `CustomScrollView` (NOT the outer NestedScrollView) in `RefreshIndicator(onRefresh: ...)`. Works with the inner scroll.

## Real-case validation

This pattern was implemented for the `home` page in this skill's own dev process (Figma node `1:768`, 萌宝相机 首页). Real-device validation on Android (DCO AL00) confirmed:

- Vertical scroll up → banner collapses → categories pin at top with proper status bar inset ✓
- Horizontal swipe on body → tab switches, strip pill highlight follows ✓
- Tap on strip pill → body animates to that tab ✓
- Pull down at top of inner scroll → banner re-expands ✓
- Per-tab scroll position preserved across swipes (PageStorageKey + KeepAlive) ✓

See `<flutter-project>/lib/pages/home/home_page.dart` in any project that ran this skill on the home page for a reference implementation.
