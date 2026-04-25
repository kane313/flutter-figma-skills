# Common AI Widget Tree Misreads

Reference this when emitting issues — each entry has a short name the review output can cite.

## "Stack misread"

**Pattern:** Figma `layoutMode: NONE` with children not overlapping (contiguous y or x offsets) → AI wrongly emits `Stack`.

**Correct interpretation:** Children are in a list, not layered. Use Row or Column.

**Heuristic to detect:** compute child bounding boxes; if no overlap, flag.

## "spaceBetween with 2 children"

**Pattern:** `Row(mainAxisAlignment: MainAxisAlignment.spaceBetween, children: [left, right])`.

**Idiomatic fix:** `Row(children: [left, Spacer(), right])` — clearer intent, easier to extend.

## "padding / gap reversed"

**Pattern:** Figma `itemSpacing: 8, paddingLeft: 16` → AI emits `Padding(padding: EdgeInsets.all(8), child: Row(children: [..., SizedBox(width: 16), ...]))`.

**Correct:** Padding uses the Figma padding value; gap between children uses itemSpacing.

## "Unbounded ListView"

**Pattern:** `Column(children: [Header(), ListView(...)])` inside a full-screen page.

**Symptom:** Renders fine in scrolling page but throws "RenderFlex children have non-zero flex but incoming height constraints are unbounded" in some layouts.

**Fix:** Wrap ListView in `Expanded` if the Column is the Scaffold body; or use `shrinkWrap: true` + `physics: NeverScrollableScrollPhysics()` for short lists.

## "Nested SingleChildScrollView / ListView"

**Pattern:** `SingleChildScrollView(child: ListView(...))`.

**Why wrong:** ListView is itself a scroll view; double-scrolling is always a bug.

**Fix:** Drop the outer SingleChildScrollView; use ListView only.

## "Missing SafeArea"

**Pattern:** `Scaffold(body: Column(children: [...]))` on a page whose Figma top is at device chrome (status bar y=0).

**Symptom:** Content clipped under status bar / notch.

**Fix:** `Scaffold(body: SafeArea(child: Column(...)))`.

## "Flexible outside Flex"

**Pattern:** `Container(child: Flexible(child: Text(...)))`.

**Error at runtime:** "Incorrect use of ParentDataWidget".

**Fix:** Remove Flexible or wrap parent in Row/Column/Flex.

## "Hardcoded MediaQuery layout"

**Pattern:** `SizedBox(width: MediaQuery.of(context).size.width * 0.8, ...)` as a primary layout mechanism.

**Why warn:** Brittle to orientation change, poor reuse.

**Fix:** Use `LayoutBuilder` + constraint math, or `FractionallySizedBox(widthFactor: 0.8)`.

## "MaterialApp inside page"

**Pattern:** Page widget wraps its own `MaterialApp(...)`.

**Why wrong:** MaterialApp is app-root; nested MaterialApps silently break theme inheritance.

**Fix:** Page widget should be just a `Scaffold` or custom widget, not re-wrapped.
