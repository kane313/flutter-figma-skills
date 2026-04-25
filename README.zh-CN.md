# flutter-figma-skills

> 🌐 [English](README.md) · **中文**

Claude Code 的 marketplace，用于把 Figma 设计稿**高保真还原成 Flutter 页面**。一条 5 阶段流水线 + 分层评分卡 + Android 真机像素 diff，全程可在 `safe / fast / auto` 三种模式间切换。

## 这个 marketplace 里有什么

| Plugin | 说明 |
|---|---|
| **`flutter-figma-reproduce`** | 主编排器：对齐设计 token → 搭骨架 → 填样式 → 接状态/交互 → 用分层评分卡验证。内含 8 个辅助 sub-skill（asset-export / autolayout-to-flex / four-states-scaffold / prototype-to-animation / theme-gap-filler / widget-tree-review / pixel-diff / align-table）|

## 安装

在 Claude Code 里：

```bash
# 1. 注册 marketplace（一次性）
/plugin marketplace add kane313/flutter-figma-skills        # 走 GitHub
# 或
/plugin marketplace add /local/path/to/flutter-figma-skills # 走本地目录

# 2. 安装 plugin
/plugin install flutter-figma-reproduce@flutter-figma-skills

# 3. 激活
/reload-plugins
```

用 `/help` 验证 — 应该能看到 `flutter-figma-reproduce` 主 skill 加 8 个 helper skill 出现在同一个 plugin 命名空间下。

## 前置依赖（每台开发机一次性配置）

| 依赖 | 用途 | 安装 |
|---|---|---|
| Claude Code（最新版，含 `/plugin` 命令）| 加载 plugin | 升级 Claude Code |
| Figma MCP server 已登录 | 读 Figma 节点 metadata + 截图 | 在 Claude Code 里 `/mcp` 配置 |
| `FIGMA_TOKEN` 环境变量 | 通过 Figma REST API 导出图片/SVG 资源 | Figma → Settings → Personal access tokens 生成 |
| `cwebp` 在 PATH | PNG → WebP 转码（资源体积优化）| macOS：`brew install webp` |
| Flutter SDK + 一个 Flutter 项目（`pubspec.yaml` 含 `flutter:` dep）| build / analyze / install 流水线 | `flutter doctor` 通过 |
| Android 真机或模拟器（推荐）| Phase 4 真机像素 diff vs Figma | `adb devices` 能看到目标设备 |

## 快速上手

`cd` 到任意 Flutter 项目根目录，跟 Claude Code 说：

```
还原这个页面：<figma-url>
```

或者英文：

```
implement this Figma design: <figma-url>
```

当消息里含 `figma.com/design/...` URL **且**当前 CWD 有 `pubspec.yaml`（含 `flutter:` 依赖）时，skill 会自动触发。

### 三种模式

| 模式 | 用什么时候 | 怎么调 |
|---|---|---|
| `safe`（默认）| 第一次还原一个新设计 | 直接发 URL — 每个 phase 之间都会暂停等你确认 |
| `fast` | 你已经信任流水线，想跳过中间 phase 的暂停 | 加 `--mode=fast` 参数 |
| `auto` | 一气呵成，全程不暂停，结尾给一个总结块 | 加 `--mode=auto` **或**消息里带触发词 `不用询问` / `yolo` / `autopilot` / `跑完为止` / `连续执行` 等 |

例子：

```
implement this design: https://www.figma.com/design/.../page?node-id=1-870  --mode=auto
```

或纯中文触发：

```
还原这个页面 https://www.figma.com/design/.../page?node-id=1-870  不用再问我了
```

## 5 阶段流水线总览

每次还原一个页面，编排器跑这 5 步：

| Phase | 输出 | Checkpoint（safe / fast / auto）|
|---|---|---|
| 0 对齐 | `align-table.md` + `gaps.md` + `component-map.md` 写到 `.figma-reproduce/<page>/` | 必须 / 必须 / 自动取默认 |
| 1 骨架 | `_skeleton.dart`（仅结构，无样式）| 必须 / 自动 / 自动 |
| 2 样式 | `<page>_page.dart` 含 token + 资源导出到 `assets/` | 可选 / 可选 / 自动 |
| 3 状态与交互 | sealed-class 状态机 + `_mock_data.dart` | 必须 / 必须 / 自动取默认 |
| 4 验证 | `verify-report.md` + 像素 diff 三联图 + 热力图 | 必须 / 必须 / 自动出报告 |

每页的中间产物保留在 `<flutter-project>/.figma-reproduce/<page>/`，下次再跑可以走差异策略（只重新生成有变化的 Figma 节点）。

## 内置防御规则（节选）

这个 skill 把多次实战踩到的坑沉淀成了硬规则，每个 phase 起手 AI 都会读 `failure-modes.md`。摘几条最常被违反的：

- **状态栏样式必须显式 `SystemUiOverlayStyle(statusBarColor: Colors.transparent, ...)` 字面量** — 不要用 `.dark` / `.light` 预设（预设在 Android 上带 25% 黑色 scrim，会在浅色 header 上看起来像半透明黑蒙层）
- **Full-bleed 背景必须 lift 到 `Center+SizedBox(width:N)` 之外** — 在比 N 宽的设备上贯通设备全宽
- **固定宽度容器里的中文 `Text` 必须包 `Flexible+FittedBox(scaleDown)+softWrap:false`** — Figma 的 PingFang SC / Yuanti SC 比 Android 回退字体 Noto Sans CJK SC 字距紧，跨字体能差 1-3px → 否则换行 / overflow
- **资源 pre-flight 检查（auto 模式强制）**：每个准备 export 的 Figma 节点先用 `mcp__figma__get_metadata` 查子节点；若 `children` 含 `text` 节点 → 资源会烘焙文字，要么改导子图标节点 ID，要么 Flutter 端跳过对应 Text widget。**这条就是上次 login 页"萌宝相机"重复显示 bug 的产物**
- **Skeleton 残留审计**：Phase 2 必须把 Phase 1 加的占位 widget（无对应 Figma 节点）逐一删除，不能"保留下来晚点 style 它"
- **SVG 资源内容分类**：full-component SVG（带圆角矩形外框 + 内部图标）按原生尺寸渲染，不要套额外的装饰 Container，否则会出现"卡中卡"+ 内部图标缩水

完整失败模式目录：`plugins/flutter-figma-reproduce/skills/flutter-figma-reproduce/references/failure-modes.md`

## 仓库布局

```
flutter-figma-skills/
├── .claude-plugin/marketplace.json     # 注册表入口（必须）
├── README.md                           # 英文 README
├── README.zh-CN.md                     # 本文档
└── plugins/
    └── flutter-figma-reproduce/
        ├── LICENSE
        ├── README.md                   # plugin 级文档
        └── skills/                     # 9 个 skill
            ├── flutter-figma-reproduce/        ← 编排器（从这开始读）
            ├── figma-flutter-align-table/
            ├── figma-autolayout-to-flex/
            ├── figma-asset-export/
            ├── figma-prototype-to-flutter-animation/
            ├── figma-flutter-pixel-diff/
            ├── flutter-four-states-scaffold/
            ├── flutter-theme-gap-filler/
            └── flutter-widget-tree-review/
```

## 升级到新版

marketplace owner push 新版本后：

```bash
/plugin marketplace update flutter-figma-skills
/plugin upgrade flutter-figma-reproduce@flutter-figma-skills
/reload-plugins
```

## 排查问题

- **"Figma MCP not responding"** — 在 Claude Code 里 `/mcp` 看状态；过期就重新认证
- **"figma-asset-export: 401 / 403"** — `FIGMA_TOKEN` 没设或过期；去 Figma settings 重新生成
- **"cwebp: command not found"** — `brew install webp`（Mac）/ `apt install webp`（Linux）
- **没有 Android 设备做像素 diff** — Phase 4 视觉层会 fallback 到 `unchecked` 并加注释；其余 token / 结构 / 交互层照常验证
- **目标页面目录名不对** — 传 `target=<dir-name>` 参数覆盖默认的"Figma frame 名 → snake_case"
- **`不用询问` 之类触发词没生效** — 检查触发词是否在你**首次发起 skill 的那条消息**里；中途加触发词不会改变模式（auto 模式中途要降级走 `等等` / `pause` / `暂停`）

## 实战案例（来自本仓库的开发过程）

| 页面 | Figma frame | 模式 | 真机 diff 率 | 主要发现 |
|---|---|---|---|---|
| `mine_unactivated`（我的-未开通）| 1:1074 | safe | 31.27% | 修复了 5 个真机才暴露的 bug：状态栏黑蒙、header 不贯通、文本换行、冗余白板、玻璃卡片图标缩水 |
| `login`（登录）| 1:870 | auto | 68.51% | auto 模式中途暴露 1 个资源烘焙文字 bug → 倒推出 "asset pre-flight" 强制规则 |
| `hot_plays`（热门玩法）| 1:428 | auto | 73.84% | pre-flight 规则生效，零中途 bug。剩余 diff 全为 Android 状态栏 vs iOS 状态栏的 17px 高度差 + 字体差 |

每次实战中发现的失败模式都已经回写到 `failure-modes.md` / `01-skeleton.md` / `02-style.md` / SKILL.md，下次同类 bug 不会再踩。

## License

详见 `plugins/flutter-figma-reproduce/LICENSE`。
