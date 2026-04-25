# Test Cases — Figma Auto Layout → Flutter Translation

Each case has: Figma config (partial, relevant fields only) → expected Flutter intent.

## Case 1: Simple vertical list

**Figma:**
```
{ layoutMode: VERTICAL, itemSpacing: 8, padding: 16, children: [Text, Text, Text] }
```

**Flutter:**
```
Padding(
  padding: EdgeInsets.all(16),
  child: Column(
    mainAxisSize: MainAxisSize.min,
    crossAxisAlignment: CrossAxisAlignment.start,
    children: [
      Text(...),
      SizedBox(height: 8),
      Text(...),
      SizedBox(height: 8),
      Text(...),
    ],
  ),
)
```

## Case 2: Horizontal spaceBetween with 2 children

**Figma:**
```
{ layoutMode: HORIZONTAL, primaryAxisAlignItems: SPACE_BETWEEN, children: [Logo, Avatar] }
```

**Flutter (preferred):**
```
Row(
  children: [
    Logo(),
    Spacer(),
    Avatar(),
  ],
)
```

**Not:** `Row(mainAxisAlignment: MainAxisAlignment.spaceBetween, children: [Logo, Avatar])` — less idiomatic for 2 children.

## Case 3: Stretch column

**Figma:**
```
{ layoutMode: VERTICAL, counterAxisAlignItems: STRETCH, children: [Button, Button] }
```

**Flutter:**
```
Column(
  crossAxisAlignment: CrossAxisAlignment.stretch,
  children: [Button(), Button()],
)
```

## Case 4: Absolute positioning

**Figma:**
```
{ layoutMode: NONE, children: [
  { x: 0, y: 0, width: 390, height: 200 },  // background image
  { x: 16, y: 160, width: 200, height: 80 }, // overlay card
]}
```

**Flutter:**
```
Stack(
  children: [
    Positioned(left: 0, top: 0, child: SizedBox(width: 390, height: 200, child: Image(...))),
    Positioned(left: 16, top: 160, child: SizedBox(width: 200, height: 80, child: Card(...))),
  ],
)
```

## Case 5: Wrap

**Figma:**
```
{ layoutMode: HORIZONTAL, layoutWrap: WRAP, itemSpacing: 8, children: [Tag x 10] }
```

**Flutter:**
```
Wrap(
  spacing: 8,
  runSpacing: 8,
  children: [Tag(), Tag(), ..., Tag()],
)
```

## Case 6: Fill container (Expanded)

**Figma:**
```
{
  layoutMode: HORIZONTAL,
  children: [
    { name: "label", primaryAxisSizingMode: FILL },  // takes remaining space
    { name: "badge", primaryAxisSizingMode: AUTO },   // intrinsic
  ]
}
```

**Flutter:**
```
Row(
  children: [
    Expanded(child: Text('label')),
    Badge(),
  ],
)
```

## Case 7: Scroll

**Figma:**
```
{ layoutMode: VERTICAL, overflowDirection: VERTICAL, height: 800, children: [...100 items] }
```

**Flutter:**
```
ListView(
  children: [...],
)
// NOT: SingleChildScrollView(child: ListView(...))  — unbounded height error
```

## Case 8: Constraints → LayoutBuilder

**Figma:**
```
{
  children: [
    { name: "content", constraints: { horizontal: "LEFT_RIGHT" } }
  ]
}
```

**Flutter:**
```
LayoutBuilder(
  builder: (context, constraints) {
    return Content(); // fills available width
  },
)
```

## Case 9 (defensive): Misleading Stack

**Figma (looks like Stack):**
```
{ layoutMode: NONE, children: [
  { x: 0, y: 0, width: 100, height: 40 },  // first item
  { x: 0, y: 40, width: 100, height: 40 },  // second item directly below
]}
```

**Flutter (correct):**
```
Column(
  children: [
    SizedBox(height: 40, child: ...),
    SizedBox(height: 40, child: ...),
  ],
)
```

**Defensive warning emitted:** "Figma layoutMode=NONE but children don't visually overlap (y offsets are contiguous). Suggest Column instead of Stack. Confirm with designer if ambiguous."
