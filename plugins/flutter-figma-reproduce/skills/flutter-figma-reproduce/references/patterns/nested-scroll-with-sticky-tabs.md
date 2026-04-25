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

## Canonical structure

```
NestedScrollView
├─ headerSliverBuilder (returns slivers ABOVE the inner body)
│  └─ SliverOverlapAbsorber                          ← MANDATORY: handle = sliverOverlapAbsorberHandleFor(context)
│     └─ SliverAppBar
│        ├─ pinned: true                              ← keeps the AppBar (incl. its `bottom`) visible after collapse
│        ├─ toolbarHeight: 0                          ← we don't want a Material toolbar bar; the bottom IS our sticky strip
│        ├─ expandedHeight: <banner-design-height>    ← e.g. 278
│        ├─ flexibleSpace: FlexibleSpaceBar
│        │   └─ background: Image.asset(banner, fit: BoxFit.cover)
│        └─ bottom: PreferredSize
│           ├─ preferredSize: Size.fromHeight(<tab-strip-height> + statusBarPadding)
│           └─ child: ColoredBox(pageBg) → Padding(top: statusBarPadding) → tab strip
└─ body: TabBarView
   └─ controller: <shared TabController>
      for each tab:
        ├─ key: PageStorageKey('cat_<id>')             ← preserves per-tab scroll position
        └─ CategoryTabContent (StatefulWidget with AutomaticKeepAliveClientMixin) ← keeps state when swiped away
           └─ CustomScrollView
              ├─ SliverOverlapInjector                ← MANDATORY: handle = sliverOverlapAbsorberHandleFor(context)
              └─ SliverGrid / SliverList (the actual content)
```

### Why each part exists (so AI can adapt the pattern, not copy blindly)

| Component | Why |
|---|---|
| `NestedScrollView` | Coordinates outer scroll (banner/header area) with inner scrolls (per-tab content). |
| `SliverOverlapAbsorber/Injector` pair | Tells the inner `CustomScrollView` "the outer header is N pixels tall right now"; without this, the inner scroll's first child renders at y=0 in the inner viewport, but the outer header overlaps it → visible content is hidden behind the pinned strip. |
| `pinned: true` | The collapsed AppBar (toolbar + bottom) stays at the top of the viewport. |
| `toolbarHeight: 0` | We don't want a Material toolbar bar; only the `bottom` (our tab strip) should remain visible after collapse. |
| `expandedHeight` | Total AppBar height when fully expanded = banner + bottom. |
| `bottom: PreferredSize` | Anything in `bottom` is rendered at the BOTTOM of the AppBar regardless of scroll position. When AppBar is fully expanded, bottom sits right above the body. When fully collapsed, bottom sits at the top of the screen — that IS the 吸顶 effect. |
| Status bar padding INSIDE `bottom` | The `bottom` widget rides at AppBar's bottom edge. When the AppBar is fully expanded, the bottom sits ~278px down from the screen top — well below the status bar. But when collapsed to just the bottom strip + statusBarHeight, the bottom edge hits y = `statusBar + tabHeight`. Without inset padding, the strip starts at y=0 (under the status bar). With `Padding(top: MediaQuery.padding.top)`, the strip starts below the status bar. |
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

  static const double _bannerH = 278;
  static const double _stripH = 30;

  @override
  Widget build(BuildContext context) {
    return NestedScrollView(
      headerSliverBuilder: (ctx, _) => [
        SliverOverlapAbsorber(
          handle: NestedScrollView.sliverOverlapAbsorberHandleFor(ctx),
          sliver: SliverAppBar(
            pinned: true,
            toolbarHeight: 0,
            expandedHeight: _bannerH,
            backgroundColor: pageBg,
            surfaceTintColor: Colors.transparent,
            elevation: 0,
            scrolledUnderElevation: 0,
            automaticallyImplyLeading: false,
            flexibleSpace: FlexibleSpaceBar(
              background: Image.asset(bannerAsset, fit: BoxFit.cover),
            ),
            bottom: PreferredSize(
              preferredSize: Size.fromHeight(_stripH + MediaQuery.of(ctx).padding.top),
              child: ColoredBox(
                color: pageBg,
                child: Padding(
                  padding: EdgeInsets.only(top: MediaQuery.of(ctx).padding.top, left: 15, right: 15),
                  child: SizedBox(
                    height: _stripH,
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
