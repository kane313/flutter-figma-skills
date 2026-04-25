# Safe pubspec.yaml Patching

## Principle

Never regenerate pubspec.yaml from scratch. Parse, mutate, re-emit preserving comments and ordering where possible.

## Tooling

Use a YAML-preserving library, not string concat:

- Dart: `package:yaml_edit` (preserves comments + formatting)
- CLI alternative: `yq -i '.flutter.assets += ["assets/images/foo.webp"]' pubspec.yaml`

Prefer `yaml_edit` when operating from within a Flutter project:

```dart
import 'package:yaml_edit/yaml_edit.dart';

final doc = YamlEditor(await File('pubspec.yaml').readAsString());
doc.update(['flutter', 'assets'], [
  ...doc.parseAt(['flutter', 'assets']).value,
  'assets/images/new_asset.webp',
]);
await File('pubspec.yaml').writeAsString(doc.toString());
```

## Section bootstrap

If `flutter.assets:` does not exist yet:

```yaml
flutter:
  uses-material-design: true
  assets:
    - assets/images/new_asset.webp
```

Insert the `assets:` list under `flutter:`. If `flutter:` itself is missing, emit error — that's a broken pubspec, not our job.

## Deduplication

Before inserting, check if the exact path is already in the list. If yes, skip — don't duplicate.

## Directory shorthand

Flutter supports `assets/images/` (trailing slash) to include all files in a directory. If the skill is adding > 5 files to the same directory, consolidate:

```yaml
flutter:
  assets:
    - assets/images/
    - assets/images/2.0x/
```

This is an optimization — only apply when safe (directory doesn't mix exported assets with user-added ones).

## Safety

1. Before writing, make a backup: `cp pubspec.yaml pubspec.yaml.bak`
2. After writing, run `flutter pub get` once to verify the file is still valid
3. If `pub get` fails, restore from backup and emit error

## Formatting

Leave a blank line before and after the `assets:` section. If the project has other conventions (indent width, quoting style), yaml_edit usually preserves them; if you see drift, log it and ask the user whether to reformat.
