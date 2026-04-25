# Mock Data Generation

## Principles

1. **Realistic, not generic** — "张三" / "李四" over "User 1" / "User 2"
2. **Diverse shapes** — if List<Post>, include posts with short/long titles, with/without images, recent/old dates
3. **Edge cases covered** — at least one item with empty optional fields to validate null handling

## Output file

Always write to `lib/pages/<page_name>/_mock_data.dart` with `// ignore_for_file: prefer_const_constructors` at top (mock data often can't be const).

Template:

```dart
// Mock data for <page_name>. Replace with real data source in production.

import 'package:<app_package>/models/<model>.dart';

final mockPosts = [
  Post(
    id: '1',
    title: '今天天气真好',
    body: '我们一起去公园散步吧',
    author: '张三',
    createdAt: DateTime(2026, 4, 20),
    imageUrl: 'https://picsum.photos/400/300?random=1',
  ),
  Post(
    id: '2',
    title: 'A very long title that will likely wrap across two lines on a standard phone screen',
    body: 'Additional content to test multi-line rendering.',
    author: '李四',
    createdAt: DateTime(2026, 4, 18),
    imageUrl: null, // edge: no image
  ),
  // ... add 3-5 more with variety
];

final mockEmpty = <Post>[];

final mockError = Exception('网络请求失败，请检查网络连接');
```

## Per-type guidance

| Type | Good mock | Bad mock |
|---|---|---|
| `User` | 姓名混合长度，头像 picsum URL，email 用 example.com | `User('user', 'u@u.com')` |
| `List<Post>` | 5-7 条，标题长度差异大，图片有无混合 | 5 条一模一样 |
| `Order` | 涵盖几种状态（待付款/已发货/已完成） | 全 "completed" |
| `DateTime` | 相对当前日期，覆盖今天/几天前/一个月前 | 全 `DateTime(2024, 1, 1)` |

## Picsum / placeholder images

Use `picsum.photos` or local placeholder assets for image URLs. Avoid hotlinking real sites (不稳定，版权风险).

## Toggle helper

At bottom of mock_data.dart, emit a developer toggle:

```dart
/// Dev helper: switch between mock states. Pass to <Page>(initialState: ...) to preview.
enum MockMode { loading, empty, error, success }
```

This gives reviewers a one-line change to see each state.
