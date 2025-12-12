### Compose Box 组件详解
Box 是 Jetpack Compose 中**最基础的布局容器之一**，核心作用是「将子组件堆叠显示」（Z 轴方向），同时支持对子组件进行精确的位置对齐、大小约束和偏移控制。它类似于 Android 传统 View 体系中的 `FrameLayout`，是实现叠层布局、自定义组件位置的核心容器。

---

### 核心特性
1. **堆叠布局**：多个子组件会按声明顺序堆叠（后声明的组件在上方）；
2. **灵活对齐**：支持对子组件进行水平/垂直方向的精准对齐（如居中、居左、居右、顶部、底部等）；
3. **位置控制**：可通过 `Modifier.offset()` `Modifier.align()` 等调整子组件位置；
4. **大小自适应**：默认包裹内容（wrap content），也可通过 `Modifier.fillMaxSize()` 等强制占满父容器；
5. **无方向约束**：不同于 Row（横向）、Column（纵向），Box 没有固定的排列方向，完全靠对齐/偏移控制子组件位置。

---

### 基础用法
#### 1. 最简示例（堆叠+居中对齐）
```kotlin
@Composable
fun BasicBoxDemo() {
    // Box 默认包裹内容，子组件默认居中对齐
    Box(
        modifier = Modifier
            .size(200.dp) // 固定宽高
            .background(Color.LightGray) // 背景色，便于观察
    ) {
        // 第一个子组件（底层）
        Text(
            text = "底层文本",
            fontSize = 20.sp,
            modifier = Modifier.align(Alignment.Center) // 居中对齐
        )
        // 第二个子组件（上层，叠在底层文本上方）
        Icon(
            imageVector = Icons.Default.Star,
            contentDescription = "星星图标",
            tint = Color.Yellow,
            modifier = Modifier
                .size(40.dp)
                .align(Alignment.TopEnd) // 右上角对齐
        )
    }
}
```

#### 2. 关键参数说明
Box 的核心参数集中在函数签名中：
```kotlin
@Composable
fun Box(
    modifier: Modifier = Modifier, // 布局修饰符（大小、背景、边距等）
    contentAlignment: Alignment = Alignment.Center, // 子组件的默认对齐方式
    propagateMinConstraints: Boolean = false, // 是否将最小约束传递给子组件
    content: @Composable BoxScope.() -> Unit // 子组件内容（带 BoxScope 作用域）
)
```

---

### 核心用法场景
#### 1. 叠层布局（Z 轴堆叠）
多个子组件按声明顺序堆叠，可通过 `Modifier.zIndex()` 调整堆叠层级（数值越大越靠上）：
```kotlin
@Composable
fun StackedBoxDemo() {
    Box(Modifier.size(150.dp)) {
        // 底层：红色方块
        Box(
            modifier = Modifier
                .size(100.dp)
                .background(Color.Red)
                .align(Alignment.Center)
        )
        // 中层：绿色方块（zIndex 1）
        Box(
            modifier = Modifier
                .size(70.dp)
                .background(Color.Green)
                .align(Alignment.Center)
                .zIndex(1f) // 层级高于红色方块
        )
        // 上层：蓝色方块（zIndex 2，最顶层）
        Box(
            modifier = Modifier
                .size(40.dp)
                .background(Color.Blue)
                .align(Alignment.Center)
                .zIndex(2f)
        )
    }
}
```

#### 2. 精准对齐子组件
通过 `Alignment` 枚举或 `Modifier.align()` 实现细粒度对齐：
| 对齐方式                | 效果                     |
|-------------------------|--------------------------|
| `Alignment.Center`      | 水平+垂直居中            |
| `Alignment.TopStart`    | 左上角（LTR 布局）       |
| `Alignment.BottomEnd`   | 右下角（LTR 布局）       |
| `Alignment.CenterStart` | 垂直居中、水平靠左       |
| `Alignment.CenterEnd`   | 垂直居中、水平靠右       |

示例：
```kotlin
@Composable
fun AlignedBoxDemo() {
    Box(
        modifier = Modifier
            .fillMaxWidth() // 占满父容器宽度
            .height(120.dp)
            .background(Color.LightGray),
        contentAlignment = Alignment.Center // 全局默认居中
    ) {
        // 覆盖全局对齐，单独设置为左下角
        Text("左下角", Modifier.align(Alignment.BottomStart))
        // 覆盖全局对齐，单独设置为右上角
        Text("右上角", Modifier.align(Alignment.TopEnd))
        // 使用全局默认居中
        Text("居中")
    }
}
```

#### 3. 偏移控制（Modifier.offset）
通过 `offset` 让子组件偏离对齐位置，支持 dp 或 px 单位：
```kotlin
@Composable
fun OffsetBoxDemo() {
    Box(
        modifier = Modifier
            .size(200.dp)
            .background(Color.LightGray),
        contentAlignment = Alignment.Center
    ) {
        Text(
            text = "偏移文本",
            modifier = Modifier
                .offset(x = 30.dp, y = -20.dp) // 右移30dp，上移20dp
                .background(Color.White)
        )
    }
}
```

#### 4. 占满父容器+子组件居底
常见于“卡片底部加标签”“图片底部加文字”场景：
```kotlin
@Composable
fun BoxWithBottomContent() {
    Box(
        modifier = Modifier
            .size(200.dp)
            .background(Color.Cyan)
    ) {
        // 底部通栏文本
        Text(
            text = "底部标签",
            modifier = Modifier
                .fillMaxWidth() // 占满宽度
                .background(Color.Black.copy(alpha = 0.5f))
                .padding(8.dp)
                .align(Alignment.BottomCenter), // 底部居中
            color = Color.White
        )
    }
}
```

---

### 高级技巧
#### 1. 约束子组件大小（Modifier.matchParentSize）
让子组件大小与 Box 完全一致（即使 Box 是包裹内容），需配合 `BoxScope` 使用：
```kotlin
@Composable
fun MatchParentSizeDemo() {
    Box(Modifier.background(Color.Gray)) { // Box 包裹内容
        Text("父容器文本")
        // 子组件占满 Box 大小（覆盖父容器背景）
        Box(
            modifier = Modifier
                .matchParentSize() // 匹配 Box 大小
                .background(Color.Red.copy(alpha = 0.3f))
        )
    }
}
```

#### 2. 嵌套 Box 实现复杂布局
结合多层 Box 实现“背景+图标+文字+角标”的复合组件：
```kotlin
@Composable
fun ComplexBoxDemo() {
    Box(
        modifier = Modifier
            .size(120.dp)
            .background(Color.White)
            .border(1.dp, Color.Gray, RoundedCornerShape(8.dp))
            .padding(8.dp)
    ) {
        // 图标（居中）
        Icon(
            imageVector = Icons.Default.Mail,
            contentDescription = "邮件",
            modifier = Modifier.align(Alignment.Center),
            tint = Color.Gray
        )
        // 角标（右上角）
        Box(
            modifier = Modifier
                .size(20.dp)
                .background(Color.Red, CircleShape)
                .align(Alignment.TopEnd),
            contentAlignment = Alignment.Center
        ) {
            Text("5", color = Color.White, fontSize = 12.sp)
        }
        // 底部文字
        Text(
            "未读邮件",
            modifier = Modifier.align(Alignment.BottomCenter),
            fontSize = 12.sp
        )
    }
}
```

---

### 与其他布局容器的区别
| 布局容器 | 核心特点                  | 适用场景                     |
|----------|---------------------------|------------------------------|
| Box      | 堆叠布局、自由对齐        | 叠层UI、精准定位、复合组件   |
| Row      | 横向线性排列              | 水平列表、按钮组、导航栏     |
| Column   | 纵向线性排列              | 垂直列表、表单、页面内容     |
| ConstraintLayout | 约束布局 | 复杂多组件对齐、比例布局     |

---

### 总结
Box 是 Compose 中最灵活的基础布局容器，核心价值在于**叠层显示**和**精准对齐**。无论是简单的“图标+文字”组合，还是复杂的叠层UI（如卡片、角标、遮罩），都可以通过 Box 快速实现。掌握 Box 的 `alignment` `offset` `zIndex` `matchParentSize` 等核心修饰符，是构建自定义 Compose 组件的基础。