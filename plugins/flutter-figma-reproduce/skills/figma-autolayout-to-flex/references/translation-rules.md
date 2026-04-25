# Translation Rules — Figma Auto Layout → Flutter Flex

## 1. Widget selection

| Figma `layoutMode` | Extra signal | Flutter widget |
|---|---|---|
| `HORIZONTAL` | single row, fits | `Row` |
| `HORIZONTAL` | wraps | `Wrap(direction: Axis.horizontal)` |
| `VERTICAL` | single column, fits | `Column` |
| `VERTICAL` | wraps | `Wrap(direction: Axis.vertical)` (rare) |
| `NONE` | positioned children overlap | `Stack` |
| `NONE` | single child | leaf container (no Flex) |

**Wrap detection**: Figma `layoutWrap = WRAP`.

## 2. Main axis alignment

Figma `primaryAxisAlignItems` → Flutter `mainAxisAlignment`:

| Figma | Flutter |
|---|---|
| `MIN` | `MainAxisAlignment.start` |
| `MAX` | `MainAxisAlignment.end` |
| `CENTER` | `MainAxisAlignment.center` |
| `SPACE_BETWEEN` | `MainAxisAlignment.spaceBetween` |

**Important**: If Figma uses `SPACE_BETWEEN` but only 2 children, prefer `Row(children: [..., Spacer(), ...])` — this is more idiomatic in Flutter.

## 3. Cross axis alignment

Figma `counterAxisAlignItems` → Flutter `crossAxisAlignment`:

| Figma | Flutter |
|---|---|
| `MIN` | `CrossAxisAlignment.start` |
| `MAX` | `CrossAxisAlignment.end` |
| `CENTER` | `CrossAxisAlignment.center` |
| `BASELINE` | `CrossAxisAlignment.baseline` + `textBaseline: TextBaseline.alphabetic` |
| `STRETCH` (`counterAxisAlignItems=STRETCH`) | `CrossAxisAlignment.stretch` |

## 4. Size

Figma `primaryAxisSizingMode`:
- `FIXED` → the Flex is inside a fixed-size parent (`SizedBox` wrapper may be needed)
- `AUTO` → `mainAxisSize: MainAxisSize.min`
- `FILL` → `Expanded` (if parent is Flex) or `mainAxisSize: MainAxisSize.max`

Figma `counterAxisSizingMode`:
- `FIXED` → fixed width/height
- `AUTO` → intrinsic sizing
- `FILL` → fill available (`double.infinity` or `crossAxisAlignment: stretch`)

## 5. Padding vs Gap

**Critical rule**: never map these backwards.

- Figma `paddingLeft/Right/Top/Bottom` → Flutter wrapper `Padding(padding: EdgeInsets...)` OR `Container(padding: ...)`. This is **inside the component**.
- Figma `itemSpacing` → gap **between children**. In Flutter:
  - Preferred: `SizedBox(width/height: itemSpacing)` inserted between children
  - Alternative: use the `gap` package: `Gap(itemSpacing)`
  - Do NOT map `itemSpacing` to `padding`

## 6. Absolute positioning → Stack

When `layoutMode = NONE` and children have `x, y` offsets within parent:

- Parent → `Stack`
- Each child → `Positioned(left: x, top: y, ...)` or `Align(alignment: ...)` if center/corner-aligned
- **Defensive warning**: if 2 children and they don't visually overlap, emit a warning "this might be a Row/Column misread as Stack — check with human"

## 7. Constraints

Figma `constraints = { horizontal: 'LEFT_RIGHT' }` (stretch horizontally):
- Signals responsive behavior → flag for `LayoutBuilder` at ancestor level
- `CENTER` → `Align(alignment: Alignment.center)` inside Stack
- `SCALE` → `FittedBox` with `fit: BoxFit.cover` (rare)

## 8. Scroll

If target node has content height > frame height, or explicit `overflowDirection`:
- Single-direction → `SingleChildScrollView`
- Multi-child vertical list → `ListView` or `ListView.builder`
- **Never** wrap `ListView` in `SingleChildScrollView` (unbounded height error)

## 9. Defensive warnings (must emit if applicable)

- [ ] Stack nesting > 2 layers
- [ ] Inferred Stack but children don't overlap (likely Row/Column misread)
- [ ] Scroll target with unbounded child
- [ ] Constraints present but no LayoutBuilder ancestor
- [ ] SafeArea missing at Scaffold body (top-level frame has device chrome)

## 10. Output dimensions (responsive scaling)

The translation's emitted dimensional values (padding, gap, fixed widths/heights, font sizes, radii, shadow offsets/blur) MUST use `flutter_screenutil` suffixes against the design baseline (typically 375×812):

- Horizontal extents → `.w` (e.g. `16.w`, `EdgeInsets.symmetric(horizontal: 16.w)`)
- Vertical extents → `.h` (e.g. `12.h`)
- Font size → `.sp`
- Radii / borders / square icon sizes that scale isotropically → `.r`
- Gaps in Row → `SizedBox(width: itemSpacing.w)`
- Gaps in Column → `SizedBox(height: itemSpacing.h)`
- Shadow → `Offset(0, 1.h)` (or `.w` for x), blur → `.r`

Exceptions: `0`, `flex` ratios, `MainAxisAlignment` weights — these are not pixels.

Output is "intent" — when emitting suggested code, pre-attach suffixes so the consumer doesn't accidentally drop them.
