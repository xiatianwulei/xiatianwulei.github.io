---
layout:     post
title:      鸿蒙 ArkUI 布局 与 CSS Flex 对照表
subtitle:   写 Less 的手感，写 ArkUI 的代码
date:       2026-10-22
author:     夏天无泪
catalog: true
tags:
    - 学习记录
    - 鸿蒙
---

# 鸿蒙 ArkUI 布局 与 CSS Flex 对照表

> 写惯了 `display: flex; justify-content: space-between;`，转过来写 ArkUI 最容易懵的就是：
> 「这个属性在鸿蒙里叫啥？」
> 这篇就是一张**翻译表**，左边 CSS/Less，右边 ArkUI（ArkTS），照着抄即可。
>
> 一句话先记住：**ArkUI 里 `Row()` / `Column()` 天生就是 flex 容器，`Flex()` 才是带换行的完整版。**

---

## 一、先搞懂三件事（不然表格也看不懂）

### 1.1 容器对应关系

| CSS / HTML | ArkUI（ArkTS） | 说明 |
|-----------|----------------|------|
| `display: flex; flex-direction: row` | `Row()` | 横向排列，**默认不换行** |
| `display: flex; flex-direction: column` | `Column()` | 纵向排列，**默认不换行** |
| `display: flex` + `flex-wrap: wrap` | `Flex({ wrap: FlexWrap.Wrap })` | 只有 `Flex` 支持换行 |
| `position: relative` 容器 + 绝对定位子元素 | `Stack()` | 层叠容器，配合 `.alignContent()` |
| `<div>` | `Column()` / `Row()` / `Stack()` | 没有 div，一切皆容器组件 |

> 注意：`Row()` 里放再多子组件也不会换行，超出的被截断/压缩。要换行必须用 `Flex({ wrap: FlexWrap.Wrap })`。

### 1.2 主轴 / 交叉轴怎么叫

CSS 里说「主轴、交叉轴」，ArkUI 里也一样，只是**属性名换了**：

| CSS | ArkUI |
|-----|-------|
| 主轴（main axis） | 主轴，`justifyContent` 控制 |
| 交叉轴（cross axis） | 交叉轴，`alignItems` 控制 |
| `flex-direction` | `FlexDirection.Row` / `Column` |

### 1.3 单位：别写 `px`

ArkUI 默认单位是 **vp（虚拟像素）**，和 CSS 的 `px` 心智模型差不多，但**写数字即可，不带单位**：

```typescript
.width(100)        // ≈ width: 100px
.width('100%')     // ≈ width: 100%
.fontSize(16)      // ≈ font-size: 16px
```

---

## 二、核心对照表（重点，直接查这里）

### 2.1 开启 Flex 容器

| CSS / Less | ArkUI | 备注 |
|-----------|-------|------|
| `display: flex;` | `Row()` / `Column()` / `Flex()` | 这三个本身就是 flex 容器，没有 `display` 属性 |
| `display: block;` | `Column()` | 纵向堆 |
| `display: inline-flex;` | 无对应 | ArkUI 没有行内概念 |
| `display: none;` | 用 `if` 条件渲染 | `if (this.show) { Text('x') }` |
| `visibility: hidden;` | `.visibility(Visibility.Hidden)` | `Visibility.None` 才会完全不占位 |

### 2.2 主轴方向 `flex-direction`

| CSS | ArkUI |
|-----|-------|
| `flex-direction: row;` | `Row()` 或 `Flex({ direction: FlexDirection.Row })` |
| `flex-direction: row-reverse;` | `Flex({ direction: FlexDirection.RowReverse })` |
| `flex-direction: column;` | `Column()` 或 `Flex({ direction: FlexDirection.Column })` |
| `flex-direction: column-reverse;` | `Flex({ direction: FlexDirection.ColumnReverse })` |

### 2.3 主轴对齐 `justify-content`（最常用）

| CSS | ArkUI | 效果 |
|-----|-------|------|
| `justify-content: flex-start;` | `.justifyContent(FlexAlign.Start)` | 靠主轴起点（默认） |
| `justify-content: center;` | `.justifyContent(FlexAlign.Center)` | 主轴居中 |
| `justify-content: flex-end;` | `.justifyContent(FlexAlign.End)` | 靠主轴终点 |
| `justify-content: space-between;` | `.justifyContent(FlexAlign.SpaceBetween)` | 两端对齐，中间等分 |
| `justify-content: space-around;` | `.justifyContent(FlexAlign.SpaceAround)` | 每个子项左右间距相等 |
| `justify-content: space-evenly;` | `.justifyContent(FlexAlign.SpaceEvenly)` | 所有间距完全相等 |

```typescript
// CSS: display:flex; flex-direction:row; justify-content:space-between;
Row() {
  Text('左边')
  Text('右边')
}
.justifyContent(FlexAlign.SpaceBetween)
.width('100%')
```

### 2.4 交叉轴对齐 `align-items`

在 `Row()`（横向容器）里，交叉轴是**垂直方向**，用 `VerticalAlign`：

| CSS | ArkUI（Row 里） | ArkUI（Column 里） |
|-----|----------------|-------------------|
| `align-items: flex-start;` | `.alignItems(VerticalAlign.Top)` | `.alignItems(HorizontalAlign.Start)` |
| `align-items: center;` | `.alignItems(VerticalAlign.Center)` | `.alignItems(HorizontalAlign.Center)` |
| `align-items: flex-end;` | `.alignItems(VerticalAlign.Bottom)` | `.alignItems(HorizontalAlign.End)` |
| `align-items: stretch;` | `ItemAlign.Stretch`（`Flex` 中） | `ItemAlign.Stretch` |

> **最容易踩的坑**：`Row` 用 `VerticalAlign.Top/Center/Bottom`，`Column` 用 `HorizontalAlign.Start/Center/End`。
> 只有 `Flex()` 容器统一用 `ItemAlign`（`Start/Center/End/Stretch/Baseline`）。

```typescript
Row() {
  Text('A')
  Text('B')
}
.alignItems(VerticalAlign.Center)   // ≈ align-items: center
```

### 2.5 单个子项对齐 `align-self`

| CSS | ArkUI |
|-----|-------|
| `align-self: center;` | `.alignSelf(ItemAlign.Center)` |
| `align-self: flex-end;` | `.alignSelf(ItemAlign.End)` |
| `align-self: stretch;` | `.alignSelf(ItemAlign.Stretch)` |
| `order: -1;` | `.displayIndex(-1)`（API 12+，用于调整显示顺序） |

### 2.6 换行与多行对齐

| CSS | ArkUI |
|-----|-------|
| `flex-wrap: nowrap;` | `Flex({ wrap: FlexWrap.NoWrap })`（默认） |
| `flex-wrap: wrap;` | `Flex({ wrap: FlexWrap.Wrap })` |
| `flex-wrap: wrap-reverse;` | `Flex({ wrap: FlexWrap.WrapReverse })` |
| `align-content: space-between;` | `Flex({ alignContent: FlexAlign.SpaceBetween })` |
| `align-content: center;` | `Flex({ alignContent: FlexAlign.Center })` |

### 2.7 弹性系数 `flex`（占比 / 撑满剩余空间）

| CSS | ArkUI | 说明 |
|-----|-------|------|
| `flex: 1;` | `.layoutWeight(1)` | **最常用**：占满父容器剩余空间 |
| `flex: 2;` | `.layoutWeight(2)` | 权重 2，按权重分配剩余空间 |
| `flex-grow: 1;` | `.flexGrow(1)` | 仅 `Flex()` 子项可用 |
| `flex-shrink: 0;` | `.flexShrink(0)` | 不允许被压缩（防止被挤扁） |
| `flex-basis: 100px;` | `.flexBasis(100)` | 主轴上的基准尺寸 |
| `flex: 1 1 0%;` | `.flexGrow(1).flexShrink(1).flexBasis(0)` | 三者组合 |

> 记忆：`Row/Column/Flex` 里想让某个子项「撑满剩下的」→ `.layoutWeight(1)`。
> 想让两个子项 1:2 分 → 分别 `.layoutWeight(1)` 和 `.layoutWeight(2)`。

```typescript
// CSS: 左侧固定，右侧撑满
Row() {
  Text('标题').width(80)
  Text('很长的内容……').layoutWeight(1)
}
.width('100%')
```

### 2.8 间距 `gap`

| CSS | ArkUI |
|-----|-------|
| `gap: 8px;` | `Row({ space: 8 })` / `Column({ space: 8 })` |
| `row-gap / column-gap` | `Flex({ space: { main: LengthMetrics.vp(8), cross: LengthMetrics.vp(8) } })`（API 14+） |
| 没 gap 时靠 `margin-right` | `.margin({ right: 8 })` | 逐个子项自己加 |

> `space` 是**构造参数**，必须写在括号里：`Column({ space: 12 })`，不是链式属性。

### 2.9 尺寸与约束

| CSS | ArkUI |
|-----|-------|
| `width: 100%;` | `.width('100%')` |
| `height: 100vh;` | `.height('100%')` |
| `width: 50%;` | `.width('50%')` |
| `min-width: 100px;` | `.constraintSize({ minWidth: 100 })` |
| `max-width: 200px;` | `.constraintSize({ maxWidth: 200 })` |
| `min-height / max-height` | `.constraintSize({ minHeight: 100, maxHeight: 200 })` |
| `aspect-ratio: 1 / 1;` | `.aspectRatio(1)` |
| `box-sizing: border-box;` | 无需设置（ArkUI 默认按边框盒计算） |
| `padding: 16px;` | `.padding(16)` |
| `padding: 8px 16px;` | `.padding({ top: 8, bottom: 8, left: 16, right: 16 })` |
| `margin: 16px;` | `.margin(16)` |
| `margin: 0 auto;` | 无 `auto`，用外层 `.alignItems(HorizontalAlign.Center)` |

### 2.10 定位与层叠

| CSS | ArkUI |
|-----|-------|
| `position: absolute; top: 10; left: 20;` | `.position({ x: 20, y: 10 })`（相对父容器左上角） |
| `position: relative; top: -5px;` | `.offset({ x: 0, y: -5 })`（相对自身原位置偏移） |
| `z-index: 10;` | `.zIndex(10)`（父容器需是 `Stack()` 或同层容器） |
| 父 `position: relative` | `Stack()` 容器 |
| `top/bottom/right/left` 贴边 | `Stack() { ... }.alignContent(Alignment.BottomEnd)` |
| `transform: translateX()` | `.translate({ x: 10 })` |
| `transform: scale(1.2)` | `.scale({ x: 1.2, y: 1.2 })` |

### 2.11 外观（写样式时顺手查）

| CSS | ArkUI |
|-----|-------|
| `background: #F5F5F5;` | `.backgroundColor('#F5F5F5')` |
| `background: linear-gradient(...)` | `.linearGradient({ angle: 90, colors: [['#FF0000', 0], ['#00FF00', 1]] })` |
| `border-radius: 8px;` | `.borderRadius(8)` |
| `border: 1px solid #EEE;` | `.border({ width: 1, color: '#EEEEEE' })` |
| `box-shadow` | `.shadow({ radius: 8, color: '#33000000', offsetX: 0, offsetY: 2 })` |
| `opacity: 0.5;` | `.opacity(0.5)` |
| `overflow: hidden;` | `.clip(true)` |
| `text-align: center;` | `.textAlign(TextAlign.Center)` |
| `white-space: nowrap; overflow: hidden; text-overflow: ellipsis;` | `.textOverflow({ overflow: TextOverflow.Ellipsis }).maxLines(1)` |
| `line-height: 24px;` | `.lineHeight(24)` |
| `cursor: pointer` / `onClick` | `.onClick(() => {})` |
| `:hover` 等伪类 | 无，需用 `@State`/`@Local` 自己维护状态切换样式 |

---

## 三、Less 复用 ↔ ArkUI 复用

写 Less 时会抽 mixin，ArkUI 里对应三件套：

| Less | ArkUI | 是否支持参数 |
|------|-------|-------------|
| `.flex-center { display:flex; align-items:center; }` 复用类 | `@Styles` 全局/组件内样式函数 | ❌ 不支持参数 |
| `.flex(@dir) { ... }` 带参 mixin | `@Extend(Text) function xxx(size: number)` | ✅ 支持参数 |
| 复用一段 HTML 结构 | `@Builder` | ✅ 支持参数，可复用任意 UI 片段 |

```less
// Less：抽一个两端对齐 + 垂直居中的 mixin
.flex-between {
  display: flex;
  justify-content: space-between;
  align-items: center;
}
.header { .flex-between(); padding: 12px; }
```

```typescript
// ArkUI：@Styles 复用一组通用属性（不能带参数）
@Styles
function flexBetween() {
  .width('100%')
  .justifyContent(FlexAlign.SpaceBetween)
  .alignItems(VerticalAlign.Center)
  .padding(12)
}

// 使用
Row() {
  Text('标题')
  Text('更多 >')
}.flexBetween()
```

```typescript
// ArkUI：@Extend 带参数（绑定到具体组件类型）
@Extend(Text)
function titleText(size: number, color: string) {
  .fontSize(size)
  .fontColor(color)
  .fontWeight(FontWeight.Bold)
}

// 使用
Text('大标题').titleText(20, '#222222')
```

> 坑点提醒：`@Styles` **只能写通用属性**（`width/height/backgroundColor/padding` 这类），
> 组件特有属性（如 `Text` 的 `fontSize`）要用 `@Extend(Text)`。

---

## 四、实战：同一张卡片，两种写法

### 4.1 需求
一个列表 item：左边头像（固定 48），右边两行文字（撑满剩余空间），最右边一个箭头。
文字区域内部：标题和副标题**两端对齐**（一个上一个下），整体垂直居中。

### 4.2 HTML + Less

```less
// Less
.item {
  display: flex;
  align-items: center;
  padding: 12px;
  gap: 12px;

  .avatar {
    width: 48px;
    height: 48px;
    border-radius: 24px;
    flex-shrink: 0;
  }

  .content {
    flex: 1;
    display: flex;
    flex-direction: column;
    justify-content: space-between;   // 两行文字撑开
    height: 48px;
  }

  .arrow {
    flex-shrink: 0;
    color: #999;
  }
}
```

```html
<div class="item">
  <img class="avatar" src="a.png" />
  <div class="content">
    <span class="title">标题</span>
    <span class="sub">副标题</span>
  </div>
  <span class="arrow">›</span>
</div>
```

### 4.3 ArkUI（ArkTS）

```typescript
Row({ space: 12 }) {
  Image($r('app.media.avatar'))
    .width(48)
    .height(48)
    .borderRadius(24)
    .flexShrink(0)

  // flex: 1 → layoutWeight(1)
  Column() {
    Text('标题').fontSize(16)
    Text('副标题').fontSize(12).fontColor('#999999')
  }
  .layoutWeight(1)
  .height(48)
  .justifyContent(FlexAlign.SpaceBetween)   // ≈ justify-content: space-between
  .alignItems(HorizontalAlign.Start)        // ≈ align-items: flex-start

  Text('›')
    .fontSize(20)
    .fontColor('#999999')
    .flexShrink(0)
}
.width('100%')
.padding(12)
.alignItems(VerticalAlign.Center)           // ≈ align-items: center
```

对照记忆：
- `gap: 12px` → `Row({ space: 12 })`
- `flex: 1` → `.layoutWeight(1)`
- `flex-shrink: 0` → `.flexShrink(0)`
- `align-items: center` → Row 里是 `.alignItems(VerticalAlign.Center)`

---

## 五、常见布局「一句话配方」

| 想要的效果 | CSS | ArkUI |
|-----------|-----|-------|
| 水平居中 | `margin: 0 auto` | 外层 `Column().alignItems(HorizontalAlign.Center)` |
| 水平垂直居中 | `display:flex; align-items:center; justify-content:center` | `Row().justifyContent(FlexAlign.Center).alignItems(VerticalAlign.Center)` |
| 左右两端对齐 | `justify-content: space-between` | `.justifyContent(FlexAlign.SpaceBetween)` |
| 三等分 | `flex: 1` × 3 | 三个子项各 `.layoutWeight(1)` |
| 左固定右自适应 | 左 `width:80px` + 右 `flex:1` | 左 `.width(80)` + 右 `.layoutWeight(1)` |
| 底部固定按钮 | `position: fixed; bottom: 0` | 外层 `Column()` + 内容区 `.layoutWeight(1)` + 按钮不设权重 |
| 撑满全屏 | `width:100%; height:100vh` | `.width('100%').height('100%')` |
| 超出一行省略 | `text-overflow: ellipsis` | `.textOverflow({ overflow: TextOverflow.Ellipsis }).maxLines(1)` |
| 流式换行标签 | `flex-wrap: wrap; gap: 8px` | `Flex({ wrap: FlexWrap.Wrap, space: { main: LengthMetrics.vp(8), cross: LengthMetrics.vp(8) } })` |

---

## 六、踩坑清单

1. **`Row()` / `Column()` 不会换行**，要换行必须换 `Flex({ wrap: FlexWrap.Wrap })`。
2. **`alignItems` 的枚举看容器类型**：`Row` → `VerticalAlign`，`Column` → `HorizontalAlign`，`Flex` → `ItemAlign`。写错直接编译报错。
3. **`space` 是构造参数**，写在 `Row({ space: 8 })` 括号里，不是 `.space(8)`。
4. **没有 `margin: 0 auto`**，居中一律靠父容器 `alignItems` / `justifyContent`。
5. **`layoutWeight` 只在 `Row/Column/Flex` 的子项里生效**，放在 `Stack()` 里无效。
6. **百分比要加引号**：`.width('100%')`，数字则不加：`.width(100)`。
7. **没有伪类**（`:hover` / `:active`），交互态要用状态变量自己切样式。
8. **默认单位是 vp 不是 px**，设计稿给 px 时按 1:1 写数字即可（vp 会自动按屏幕密度换算）。

---

## 七、速记口诀

```
display: flex        →  Row() / Column() / Flex()
flex-direction       →  FlexDirection.Row / Column
justify-content      →  .justifyContent(FlexAlign.xxx)
align-items          →  .alignItems(VerticalAlign / HorizontalAlign / ItemAlign)
align-self           →  .alignSelf(ItemAlign.xxx)
flex-wrap            →  Flex({ wrap: FlexWrap.Wrap })
gap                  →  Row({ space: 8 })
flex: 1              →  .layoutWeight(1)
width: 100%          →  .width('100%')
padding / margin     →  .padding(16) / .margin(16)
```

**一句话总结**：CSS 的 flex 属性在 ArkUI 里基本都还在，只是从「CSS 声明」变成了「链式方法」，名字从 `xxx-yyy` 变成 `xxxYyy`，值从字符串变成枚举。记住 `FlexAlign` / `ItemAlign` / `VerticalAlign` / `HorizontalAlign` 四个枚举，就能写 90% 的布局。
