# Output Format Spec

## align-table.md

Six sections: Colors / Typography / Spacing / Radius / Shadow / Icons. Each is a table:

| Figma token | Figma value | Project ref | Project value | Match type | Confidence |
|---|---|---|---|---|---|
| `primary/500` | `#3B82F6` | `colorScheme.primary` | `Color(0xFF3B82F6)` | exact | high |
| `text/body` | `16px / 500` | `textTheme.bodyLarge` | `fontSize: 16, fontWeight: w500` | name | medium |

Every row must reference concrete Figma variable names and concrete Dart identifiers (file:line if possible).

## gaps.md

Two sections: "Tokens in Figma but not in project" and "Tokens in project but not in Figma" (latter for information only, usually empty in greenfield projects).

Each entry:

```
### <token name>
- Figma ref: `primary/400`
- Figma value: `#60A5FA`
- Suggested resolution: [ ] add to project theme as `colorScheme.primaryContainer` / [ ] inline in page (temporary)
- Reason: <why this gap exists, if inferable>
```

## component-map.md

One entry per Figma component instance found in target node:

```
### <Figma component name>
- Figma componentId: `1:234`
- Instances found in target: 3
- Project candidate: `lib/widgets/app_button.dart:12` (PrimaryButton) — name similarity: medium
- Action: [ ] reuse PrimaryButton with props {variant: primary} / [ ] create new widget / [ ] inline
```

If no project candidate: "Project candidate: none — must create or inline".
