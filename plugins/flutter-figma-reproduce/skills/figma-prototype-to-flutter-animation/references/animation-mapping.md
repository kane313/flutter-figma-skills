# Animation Mapping: Figma Prototype → Flutter

## Variants switching (inside one screen)

**Figma:** Component with multiple variants, `TRIGGER_ON_CLICK` + `CHANGE_TO`.

**Flutter:**

```dart
AnimatedContainer(
  duration: Duration(milliseconds: <figma_duration>),
  curve: <curve_from_curve_mapping>,
  decoration: BoxDecoration(
    color: _isActive ? Theme.of(context).colorScheme.primary : Theme.of(context).colorScheme.surface,
    borderRadius: BorderRadius.circular(_isActive ? 16 : 8),
  ),
  child: ...,
)
```

For swapping entire child subtrees (e.g. tab bar content):

```dart
AnimatedSwitcher(
  duration: Duration(milliseconds: <figma_duration>),
  child: _selectedTab == 0 ? const TabOne() : const TabTwo(),
)
```

## Smart Animate across navigation

**Figma:** `TRIGGER_ON_CLICK` + `NAVIGATE` + `SMART_ANIMATE` where a shared element transforms across screens.

**Flutter:** `Hero` widget pair.

```dart
// source screen
Hero(tag: 'avatar-${user.id}', child: CircleAvatar(...))
// destination screen
Hero(tag: 'avatar-${user.id}', child: CircleAvatar(radius: 60, ...))
```

Hero's default curve is `Curves.fastOutSlowIn` (~250ms). To match Figma duration/curve exactly, configure `PageRouteBuilder.transitionDuration` to match and supply a custom `createRectTween` via `HeroController`.

## Page transitions (no shared element)

**Figma:** Page-level interaction with `INSTANT` / `DISSOLVE` / `SLIDE_*` transitions.

| Figma transition | Flutter |
|---|---|
| INSTANT | `PageRouteBuilder` with `transitionDuration: Duration.zero` |
| DISSOLVE | `FadeTransition` |
| SLIDE_FROM_RIGHT | `SlideTransition(position: Tween(begin: Offset(1,0), end: Offset.zero).animate(animation))` |
| SLIDE_FROM_BOTTOM | same with `Offset(0,1)` |
| PUSH_FROM_RIGHT | platform-default (`MaterialPageRoute` on Android, `CupertinoPageRoute` on iOS) |

Emit using `PageRouteBuilder` for custom transitions:

```dart
PageRouteBuilder(
  transitionDuration: Duration(milliseconds: <figma_duration>),
  pageBuilder: (_, __, ___) => const Destination(),
  transitionsBuilder: (_, animation, __, child) => SlideTransition(
    position: Tween<Offset>(begin: Offset(1, 0), end: Offset.zero)
      .chain(CurveTween(curve: <curve>))
      .animate(animation),
    child: child,
  ),
)
```

If project has `go_router`, wrap in a `CustomTransitionPage`.

## Gesture-driven (drag, swipe)

Figma's `TRIGGER_ON_DRAG` → Flutter `Dismissible` (swipe to dismiss) or `GestureDetector` + `AnimationController`.

## Implicit animations — default duration

If Figma doesn't specify duration: default to **250ms** (Material Design standard).
If Figma doesn't specify curve: default to **`Curves.easeOutCubic`** (spec default).

## Animation overriding

If the target widget is wrapped in a parent `AnimatedSwitcher` or similar, don't add another layer — warn the user and let them decide.
