# State Model

Preferred shape: **sealed class** (Dart 3+) representing the 4 states.

```dart
sealed class <Name>State {
  const <Name>State();
}

final class <Name>Loading extends <Name>State {
  const <Name>Loading();
}

final class <Name>Empty extends <Name>State {
  const <Name>Empty();
}

final class <Name>Error extends <Name>State {
  const <Name>Error(this.message, {this.retry});
  final String message;
  final VoidCallback? retry;
}

final class <Name>Success extends <Name>State {
  const <Name>Success(this.data);
  final <DataType> data;
}
```

Usage in build:

```dart
Widget build(BuildContext context) {
  return switch (state) {
    <Name>Loading() => const _Loading(),
    <Name>Empty() => const _Empty(),
    <Name>Error(:final message, :final retry) => _Error(message: message, onRetry: retry),
    <Name>Success(:final data) => _Success(data: data),
  };
}
```

Dart's exhaustiveness checking will catch missing states.

## Per-state-mgmt adaptations

### Riverpod (AsyncValue)

Riverpod ships `AsyncValue<T>` with `when(data:, loading:, error:)`. We extend for "empty" — a success value that semantically represents empty.

```dart
final postsProvider = FutureProvider<List<Post>>((ref) async { /* fetch */ });

Widget build(context, ref) {
  return ref.watch(postsProvider).when(
    loading: () => const _Loading(),
    error: (e, _) => _Error(message: e.toString(), onRetry: () => ref.invalidate(postsProvider)),
    data: (posts) => posts.isEmpty ? const _Empty() : _Success(data: posts),
  );
}
```

### Bloc / Cubit

Emit a sealed `State` type as above, and a Cubit that transitions states. The page uses `BlocBuilder`:

```dart
BlocBuilder<PostsCubit, PostsState>(
  builder: (context, state) {
    return switch (state) {
      PostsLoading() => const _Loading(),
      PostsEmpty() => const _Empty(),
      PostsError(:final message) => _Error(message: message),
      PostsSuccess(:final data) => _Success(data: data),
    };
  },
)
```

### Provider (ChangeNotifier)

ChangeNotifier with mutable `state` field + `notifyListeners()`. Page uses `Consumer`.

### None (plain StatefulWidget)

Plain state with nullable `data` + `loading` + `error` flags. Sealed class overkill; use enum + field:

```dart
enum _Stage { loading, empty, error, success }
_Stage _stage = _Stage.loading;
List<Post>? _data;
String? _error;
```
