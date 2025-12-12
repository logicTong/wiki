### Compose Modifier 尺寸相关方法全解析
Modifier 是 Compose 中用于修饰组件布局、样式、行为的核心工具，其中**尺寸相关方法**是控制组件宽高、占比、约束的关键，可分为「固定尺寸」「填充父容器」「最小/最大约束」「自适应尺寸」四大类，以下是详细说明和使用场景：

---

## 一、核心分类与常用方法
### 1. 固定尺寸（硬编码宽/高）
直接指定组件的宽/高为固定值（如 dp、px），适用于尺寸确定的场景（如图标、按钮、卡片）。

| 方法                | 作用                                  | 示例                                  |
|---------------------|---------------------------------------|---------------------------------------|
| `size(size: Dp)`    | 同时设置宽=高（正方形）               | `Modifier.size(48.dp)`（图标大小）    |
| `size(width: Dp, height: Dp)` | 分别设置宽、高              | `Modifier.size(100.dp, 50.dp)`（按钮）|
| `width(size: Dp)`   | 仅设置宽度                            | `Modifier.width(200.dp)`              |
| `height(size: Dp)`  | 仅设置高度                            | `Modifier.height(80.dp)`              |

**示例**：
```kotlin
// 固定尺寸的按钮
Button(
    onClick = {},
    modifier = Modifier.size(120.dp, 40.dp)
) {
    Text("固定尺寸按钮")
}

// 正方形图标
Icon(
    imageVector = Icons.Default.Home,
    contentDescription = "首页",
    modifier = Modifier.size(24.dp)
)
```

### 2. 填充父容器（占比尺寸）
让组件宽/高占满父容器的全部或部分空间，适用于“撑满布局”场景（如页面背景、列表项、容器）。

| 方法                          | 作用                                  | 示例                                      |
|-------------------------------|---------------------------------------|-------------------------------------------|
| `fillMaxSize(fraction: Float = 1f)` | 宽+高占满父容器（fraction 为占比，0~1） | `Modifier.fillMaxSize()`（全屏组件）、`Modifier.fillMaxSize(0.8f)`（占80%） |
| `fillMaxWidth(fraction: Float = 1f)` | 宽度占满父容器（fraction 为占比）     | `Modifier.fillMaxWidth()`（通栏按钮）      |
| `fillMaxHeight(fraction: Float = 1f)` | 高度占满父容器（fraction 为占比）     | `Modifier.fillMaxHeight(0.5f)`（占50%高度） |

**关键说明**：
- `fraction` 默认为 1f（100%），设置 0.5f 即占父容器的 50%；
- 填充效果受父容器约束（如父容器是 wrapContent，则 fillMax 无意义）；
- 可结合 `padding` 使用，避免组件贴边：
  ```kotlin
  // 占满宽度，左右留16dp边距
  Box(
      modifier = Modifier
          .fillMaxWidth()
          .padding(horizontal = 16.dp)
          .height(60.dp)
          .background(Color.Gray)
  )
  ```

### 3. 最小/最大尺寸约束
限制组件的宽/高不能小于/大于指定值，常用于“自适应但有边界”的场景（如输入框、卡片）。

| 方法                          | 作用                                  | 示例                                      |
|-------------------------------|---------------------------------------|-------------------------------------------|
| `minWidth(size: Dp)`          | 宽度最小值（小于该值时强制为该值）    | `Modifier.minWidth(80.dp)`（按钮最小宽度） |
| `minHeight(size: Dp)`         | 高度最小值                            | `Modifier.minHeight(40.dp)`               |
| `maxWidth(size: Dp)`          | 宽度最大值（大于该值时强制为该值）    | `Modifier.maxWidth(300.dp)`（输入框最大宽度） |
| `maxHeight(size: Dp)`         | 高度最大值                            | `Modifier.maxHeight(200.dp)`              |
| `widthIn(min: Dp, max: Dp)`   | 宽度范围（同时设置最小+最大）         | `Modifier.widthIn(100.dp, 200.dp)`        |
| `heightIn(min: Dp, max: Dp)`  | 高度范围                              | `Modifier.heightIn(40.dp, 80.dp)`         |

**示例**：
```kotlin
// 输入框：宽度自适应，最小150dp，最大300dp；高度最小48dp
OutlinedTextField(
    value = "",
    onValueChange = {},
    label = { Text("输入框") },
    modifier = Modifier
        .widthIn(150.dp, 300.dp)
        .minHeight(48.dp)
)
```

### 4. 自适应尺寸（包裹内容）
让组件宽/高自适应子内容大小（默认行为，显式声明可覆盖其他约束）。

| 方法                  | 作用                                  | 示例                                      |
|-----------------------|---------------------------------------|-------------------------------------------|
| `wrapContentSize()`   | 宽+高包裹内容（取消 fillMax 等约束）  | `Modifier.wrapContentSize()`              |
| `wrapContentWidth()`  | 宽度包裹内容                          | `Modifier.wrapContentWidth()`             |
| `wrapContentHeight()` | 高度包裹内容                          | `Modifier.wrapContentHeight()`            |

**关键说明**：
- Compose 组件默认宽高为 `wrapContent`，无需显式声明；
- 仅在需要**覆盖之前的填充/固定尺寸约束**时使用（如父组件强制 fillMax，子组件需包裹内容）：
  ```kotlin
  Box(Modifier.fillMaxWidth()) { // 父容器占满宽度
      // 子组件包裹内容，居中显示
      Text(
          "自适应宽度",
          modifier = Modifier
              .wrapContentWidth()
              .align(Alignment.Center)
              .background(Color.Yellow)
      )
  }
  ```

---

## 二、高级尺寸方法
### 1. `matchParentSize()`（仅 BoxScope 可用）
让子组件大小与父 Box 完全一致（即使父 Box 是 wrapContent），区别于 `fillMaxSize`（fillMax 依赖父容器的约束，matchParentSize 直接匹配父组件尺寸）。

**示例**：
```kotlin
Box(Modifier.size(100.dp).background(Color.Gray)) { // 父 Box 固定100dp
    // 子组件匹配父 Box 大小（100dp×100dp）
    Box(
        modifier = Modifier
            .matchParentSize()
            .background(Color.Red.copy(alpha = 0.3f))
    )
}
```

### 2. `aspectRatio(ratio: Float)`
设置组件宽高比（宽度/高度），适用于“固定比例”场景（如图片、视频、卡片）。

**示例**：
```kotlin
// 宽高比 16:9 的图片容器
Box(
    modifier = Modifier
        .fillMaxWidth()
        .aspectRatio(16/9f)
        .background(Color.Black)
)
```

### 3. `requiredSize()` / `requiredWidth()` 等
强制忽略父容器的尺寸约束（慎用），例如父容器限制最大宽度为 200dp，子组件用 `requiredWidth(300.dp)` 可强制突破约束。

**示例**：
```kotlin
Box(Modifier.maxWidth(200.dp)) { // 父容器最大宽度200dp
    // 强制宽度300dp（突破父约束）
    Box(
        modifier = Modifier
            .requiredWidth(300.dp)
            .height(50.dp)
            .background(Color.Blue)
    )
}
```

---

## 三、常见组合用法
### 1. 固定宽高 + 占比
```kotlin
// 宽度占满父容器，高度固定50dp
Box(
    modifier = Modifier
        .fillMaxWidth()
        .height(50.dp)
        .background(Color.LightGray)
)
```

### 2. 最小约束 + 占比
```kotlin
// 宽度占满父容器（最小100dp），高度自适应（最小40dp）
Button(
    onClick = {},
    modifier = Modifier
        .fillMaxWidth()
        .minWidth(100.dp)
        .minHeight(40.dp)
) {
    Text("自适应按钮")
}
```

### 3. 宽高比 + 填充
```kotlin
// 宽度占满父容器，高度按1:1比例自适应（正方形）
Box(
    modifier = Modifier
        .fillMaxWidth()
        .aspectRatio(1f)
        .background(Color.Cyan)
)
```

---

## 四、注意事项
1. **优先级**：`requiredXXX` > 固定尺寸 > 最大/最小约束 > 填充/自适应；
2. **父容器约束**：子组件的尺寸最终受父容器限制（如父容器宽 200dp，子组件 `fillMaxWidth(1f)` 最多 200dp）；
3. **性能**：避免过度使用 `requiredXXX` 突破约束，可能导致布局溢出或视觉错乱；
4. **单位**：尺寸方法默认使用 `Dp`（适配不同屏幕），如需像素可使用 `sizeIn(px = ...)`（不推荐，除非特殊场景）。

---

## 总结
Modifier 尺寸方法的核心是「按需选择约束类型」：
- 固定尺寸：图标、按钮等确定尺寸的组件；
- 填充父容器：页面背景、通栏组件、容器；
- 最小/最大约束：输入框、卡片等自适应但有边界的组件；
- 自适应尺寸：文本、标签等随内容变化的组件；
- 高级方法：宽高比（图片）、matchParentSize（Box 子组件）、requiredXXX（突破约束）。

灵活组合这些方法，可满足 Compose 中几乎所有布局的尺寸需求。