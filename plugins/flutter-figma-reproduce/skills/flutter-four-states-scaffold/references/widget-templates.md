# Widget Templates

All four templates are private (`_Name`) by default, co-located with the page.

## Loading

```dart
class _Loading extends StatelessWidget {
  const _Loading();
  @override
  Widget build(BuildContext context) {
    return const Center(child: CircularProgressIndicator.adaptive());
  }
}
```

For lists, prefer shimmer skeleton (requires `shimmer` package) over spinner:

```dart
// if shimmer available:
ListView.builder(
  itemCount: 6,
  itemBuilder: (_, __) => const _ShimmerCard(),
);
```

Detect `shimmer` in pubspec; fall back to spinner if not present.

## Empty

```dart
class _Empty extends StatelessWidget {
  const _Empty();
  @override
  Widget build(BuildContext context) {
    return Center(
      child: Column(
        mainAxisSize: MainAxisSize.min,
        children: [
          Icon(Icons.inbox_outlined, size: 64, color: Theme.of(context).colorScheme.outline),
          const SizedBox(height: 12),
          Text('暂无数据', style: Theme.of(context).textTheme.bodyLarge),
        ],
      ),
    );
  }
}
```

Localize "暂无数据" via `AppLocalizations.of(context).emptyData` if i18n initialized.

## Error

```dart
class _Error extends StatelessWidget {
  const _Error({required this.message, this.onRetry});
  final String message;
  final VoidCallback? onRetry;

  @override
  Widget build(BuildContext context) {
    return Center(
      child: Column(
        mainAxisSize: MainAxisSize.min,
        children: [
          Icon(Icons.error_outline, size: 48, color: Theme.of(context).colorScheme.error),
          const SizedBox(height: 12),
          Text(message, style: Theme.of(context).textTheme.bodyMedium, textAlign: TextAlign.center),
          if (onRetry != null) ...[
            const SizedBox(height: 16),
            FilledButton(onPressed: onRetry, child: const Text('重试')),
          ],
        ],
      ),
    );
  }
}
```

## Success

Stub — the caller implements this based on Figma's success state design. The scaffold only emits:

```dart
class _Success extends StatelessWidget {
  const _Success({required this.data});
  final Object data; // replace with real data type

  @override
  Widget build(BuildContext context) {
    // TODO: implement based on Figma design
    return Center(child: Text('Success: ${data.toString()}'));
  }
}
```

Emit a clear TODO comment — the scaffold does NOT try to generate the real UI.

## List refresh / load-more

If data type is a List, wrap success state in `RefreshIndicator` and expose a `loadMore` callback:

```dart
RefreshIndicator(
  onRefresh: onRefresh,
  child: ListView.builder(
    itemCount: data.length + 1,
    itemBuilder: (_, i) {
      if (i == data.length) return _LoadMoreIndicator(onTrigger: loadMore);
      return _Item(data[i]);
    },
  ),
)
```
