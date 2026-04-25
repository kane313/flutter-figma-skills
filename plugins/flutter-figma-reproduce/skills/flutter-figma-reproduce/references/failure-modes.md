# AI Common Failure Modes (Flutter + Figma)

AI scans this at the start of every phase to avoid repeating known mistakes.

## 结构类

- ✗ Auto Layout 纵向 → `Column + mainAxisAlignment.spaceBetween`（实际应是 `Spacer()` 或固定 gap）
- ✗ 两个相邻 frame 塞 Stack（看着像叠层，其实是 Row/Column）
- ✗ `SingleChildScrollView` 嵌 `ListView` → unbounded height 报错
- ✗ `Flexible/Expanded` 用在非 Flex 父节点里
- ✗ 忘 `SafeArea`
- ✗ 全屏 full-bleed 背景（header image / 顶部渐变 / 全屏底色）放进 `Center+SizedBox(width:N)` 约束里 → 在比 N 宽的设备上两侧露出 page bg。**正解：full-bleed 背景必须 lift 到外层 Scaffold body Stack 的最底层，与居中 N 宽的内容 Stack 平级**
- ✗ Phase 1 skeleton 里加的"占位背景面板"（如 `Positioned(top:378, h:640, ColoredBox(white))` 这种"猜测的 page-level white panel"）在 Phase 2 没被审计删除 → 错误地遮盖了 page bg 或与其他 card 叠加。**Phase 2 必须把 skeleton 里的每个 widget 与 Figma metadata 一一比对，凡 Figma 没有对应节点的 widget 一律删除**

## 样式类

- ✗ `Color(0xFF...)` 硬编码绕过对齐表
- ✗ 字重 `Medium` 映射成 `w400`
- ✗ 行高直接写百分比
- ✗ 阴影只给 blur 漏 offset
- ✗ `BorderRadius.all` / `.only` 混用没察觉不对称
- ✗ 默认字体而非指定字体
- ✗ 跨字体的中文 `Text` 直接写在固定宽度容器里（如 78w 的 tile / 47w 的 badge），没有缩放守卫 → Figma 用 PingFang SC / Yuanti SC 等紧密中文字体能塞下，Android 回退到 Noto Sans CJK SC 笔画更宽 → 单行换行或 `RenderFlex overflow`。**正解：受限宽度内的中文 `Text` 默认包 `FittedBox(fit: BoxFit.scaleDown)`（或 `Flexible+FittedBox` 在 Flex 子节点里），并显式 `maxLines:1, softWrap:false`。真机 PingFang 不缩放，回退字体下缩 1-3% 视觉无感**
- ✗ Android `SystemUiOverlayStyle.dark` 预设直接用 → Android 上 statusBarColor 是 25% 黑（preset 内置 readability scrim），覆盖在浅色背景上看起来像半透明黑色蒙层。**正解：显式 `SystemUiOverlayStyle(statusBarColor: Colors.transparent, statusBarIconBrightness: Brightness.dark, statusBarBrightness: Brightness.light)`，不要依赖 `.dark` / `.light` 预设**

## 资源类

- ✗ PNG 而非 WebP
- ✗ 忘更新 `pubspec.yaml` assets 列表
- ✗ `Icons.xxx` 代替设计稿定制 icon（长得像但不是）
- ✗ 用 `mcp__figma__get_screenshot` 拉资源字节（SVG/大图会触发 Anthropic 400 "Could not process image"）→ 必须走 Bash + curl + Figma REST API
- ✗ **从文件名推断 SVG 内容**：把 `card_*.svg` / `tile_*.svg` / 任何包含装饰外框的 SVG 当作"纯图标"使用，外面再套一层装饰 Container（`BackdropFilter + 白底 + border`） → 双层卡片 / 图标视觉显著缩小。Figma 导出的 SVG 经常是**整个 component instance**（外框 + 内容一体），不是纯图标。**正解：Phase 0 align-table 阶段，对每个要复用的 SVG 用 `head -c 800 <svg>` 或 `grep -c '<path' <svg>` 检查实际内容；判定是 "纯图标 → 套装饰容器 + 渲染 28-34px" 还是 "完整卡片 → 直接渲染原生尺寸 + 不套外框"**
- ✗ **从节点名推断 PNG / WebP 内容**：把 Figma 命名为 `产品logo展示` / `第三方登录` / `tab item` 等的 component instance 整个导出为 PNG → 导出图里**烘焙**了内部子节点的 Text 标签（如 "萌宝相机" / "游客登录"）。Flutter 端再加 `Text` widget 显示同样的文字 → 文字渲染两次（一次在位图里，一次在 Flutter 文本层）。**正解：导出前必须用 `mcp__figma__get_metadata` 看节点子树。若 children 包含 `text` 节点 → 改导只导图标子节点（如 `I1:876;652:1376` 而不是父节点 `1:876`），或在 Flutter 里跳过对应的 Text widget。Auto mode 也必须执行此检查（不能因为不暂停用户就跳过机械 pre-flight）**
- ✗ Figma 用了 SVG `<foreignObject>` / `<filter>` / `<feGaussianBlur>` 做模糊背景 → flutter_svg 不支持，会报 `unhandled element` warning 但仍渲染其余 path。视觉差异：模糊层丢失。**正解：用 Flutter `BackdropFilter(ImageFilter.blur(...))` 在 Flutter 层补回模糊，SVG 只负责前景 path**

## 状态/交互类

- ✗ 只做 success 态
- ✗ 按钮无 splash
- ✗ 列表不支持下拉刷新
- ✗ 动画默认 linear 而非 easeOutCubic
- ✗ 页面转场默认 `MaterialPageRoute` 不看 Figma prototype
- ✗ 错误态没有重试按钮

## 工程化类

- ✗ 硬编码中文/英文字符串，绕过 `AppLocalizations`
- ✗ 在页面 widget 里嵌 `MaterialApp`（app-root 应该只有一个）
- ✗ 用 `MediaQuery.of(context).size.width * 0.8` 做主要布局（应 `LayoutBuilder` 或 `FractionallySizedBox`）
- ✗ Scaffold 忘记 `extendBodyBehindAppBar: true` + `AppBar.backgroundColor: Colors.transparent` → 内容没延伸到状态栏底下，沉浸式效果丢失
- ✗ `systemOverlayStyle` 不根据背景明暗调整 → 浅色背景配浅色状态栏文字读不清，反之亦然
- ✗ 全屏沉浸式页面用 `SafeArea(top: true)` → 状态栏底下出现白边；沉浸式页面应 `SafeArea(top: false, bottom: true)`
