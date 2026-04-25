# Senior Developer Implicit Checklist

AI reads this in Phase 0-1 and quietly applies it. These are the things a senior Flutter dev notices in seconds that a junior dev misses entirely.

## A. 设计稿打开第一眼（30 秒内）

- [ ] Top frame 尺寸（390×844？375×667？1440×900？）→ 决定响应式策略（本 skill 不强制，但记录在 config）
- [ ] Auto Layout 开了哪些层、没开哪些层 → 决定 Flex 还是 Stack
- [ ] 是否引用 Library？variables/styles 是否全挂 tokens → 决定对齐表覆盖率
- [ ] 有没有 Prototype 连线 → 暗示交互/动画（Phase 3）
- [ ] 蓝色菱形（组件）哪些来自 library、哪些是 local → 决定复用 vs 新建

## B. 拆组件时

- [ ] 命名带 `/` 的是 variants 家族（如 `Button/Primary/Large`）→ 明确复用信号
- [ ] 重复出现 >2 次的 pattern → 抽共享组件
- [ ] 同视觉不同语义（"两个按钮长一样但一个叫 Primary 一个叫 CTA"）→ 问设计师 or 按语义拆
- [ ] Figma Auto Layout 的 padding（组件内）vs gap（组件间）不要映射反

## C. 样式细节（高频漏点）

- [ ] 字体在 `pubspec.yaml` 有吗？没有走 GoogleFonts 或提醒设计师
- [ ] 字重：`Medium` = `w500`（不是 w400）
- [ ] 行高：Figma 百分比 → Flutter `height`（倍数 = 百分比 / 100，相对字号）
- [ ] 字间距：Figma px → Flutter `letterSpacing` px（可负）
- [ ] 阴影：Figma 多层 effect → Flutter `BoxShadow` 列表
- [ ] 渐变：Figma angle → Flutter `begin/end` 坐标换算（高频翻车）

## D. 交互/动效（Figma 暗示 → Flutter 映射）

- [ ] variants 切换 interaction → `AnimatedContainer` / implicit animations
- [ ] Smart Animate → `Hero` / `AnimatedPositioned`
- [ ] 页面级过渡 → `PageRouteBuilder` + `transitionsBuilder`
- [ ] 按钮 `Press` variant → `InkWell` splash；没 variant 但是按钮 → 默认加 splash
- [ ] 列表项 "Delete" 标注 → `Dismissible`

## E. 数据/状态暗示

- [ ] 骨架屏页面 → loading 态（没画也要产）
- [ ] 空状态插画 → empty 态（没画默认"暂无数据"）
- [ ] 错误页 → error 态（没画默认 + 重试按钮）
- [ ] 长文本 placeholder（`Lorem ipsum`/`xxxx`）→ 标记假数据、提醒替换
