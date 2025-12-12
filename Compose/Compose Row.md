`Row` 是 Jetpack Compose 中用于**水平排列子组件**的核心布局容器，对标传统 Android 的 `LinearLayout(horizontal)`，支持灵活的对齐方式、间距控制、权重分配等特性，是实现横向布局的基础。以下是 `Row` 的完整用法，包括基础使用、核心参数、进阶场景和常见技巧。

### 一、核心特性
1. 水平排列：子组件默认从左到右依次排列；
2. 自适应尺寸：默认包裹子组件的宽高，也可通过 `Modifier` 强制设置宽高；
3. 灵活对齐：支持子组件的水平/垂直对齐、权重分配；
4. 溢出处理：可通过 `horizontalArrangement` 或 `Modifier.horizontalScroll` 处理子组件超出屏幕的情况；
5. 与 Compose 状态联动：子组件状态变化自动重组。

### 二、基础使用
#### 1. 最简示例（横向排列多个组件）
```kotlin
import androidx.compose.foundation.layout.Row
import androidx.compose.foundation.layout.padding
import androidx.compose.material.Text
import androidx.compose.material.Icon
import androidx.compose.material.icons.Icons
import androidx.compose.material.icons.filled.Phone
import androidx.compose.runtime.Composable
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.unit.dp

@Composable
fun BasicRow() {
    // 基础 Row：水平排列 Icon + Text
    Row(
        // 修饰符：设置内边距、宽高、背景等
        modifier = Modifier
            .padding(16.dp)
            .fillMaxWidth(), // 占满父布局宽度
        // 垂直对齐：子组件在垂直方向的对齐方式（默认 CenterVertically）
        verticalAlignment = Alignment.CenterVertically,
        // 水平排列方式：子组件在水平方向的分布（默认 Start）
        horizontalArrangement = Arrangement.Start
    ) {
        // 子组件1：Icon
        Icon(
            imageVector = Icons.Default.Phone,
            contentDescription = "Phone Icon",
            modifier = Modifier.padding(end = 8.dp) // 与右侧Text的间距
        )
        // 子组件2：Text
        Text(
            text = "13800138000",
            fontSize = 16.sp
        )
    }
}
```

#### 2. 核心参数说明
| 参数 | 作用 | 常用值 |
|------|------|--------|
| `modifier` | 布局修饰符（宽高、内边距、背景、点击事件等） | `fillMaxWidth()`、`padding()`、`background()` |
| `horizontalArrangement` | 子组件在水平方向的排列/分布方式 | `Arrangement.Start`（左对齐）、`Arrangement.Center`（居中）、`Arrangement.End`（右对齐）、`Arrangement.SpaceBetween`（两端对齐）、`Arrangement.SpaceAround`（均匀分布）、`Arrangement.SpaceEvenly`（等间距分布） |
| `verticalAlignment` | 子组件在垂直方向的对齐方式 | `Alignment.Top`（顶部对齐）、`Alignment.CenterVertically`（垂直居中）、`Alignment.Bottom`（底部对齐）、`Alignment.Baseline`（文字基线对齐） |
| `content` | 子组件集合（可放任意 Composable） | - |

### 三、核心场景示例
#### 1. 权重分配（weight）
通过 `Modifier.weight()` 实现子组件按比例分配水平空间（对标传统 Android 的 `layout_weight`），**注意：使用 `weight` 的组件会自动占满剩余空间，需避免与 `fillMaxWidth()` 冲突**。

```kotlin
@Composable
fun RowWithWeight() {
    Row(
        modifier = Modifier
            .fillMaxWidth()
            .padding(16.dp),
        verticalAlignment = Alignment.CenterVertically
    ) {
        // 占比 1/4
        Text(
            text = "左",
            modifier = Modifier
                .weight(1f)
                .background(Color.LightGray)
                .padding(8.dp)
        )
        // 占比 3/4
        Text(
            text = "右（权重3）",
            modifier = Modifier
                .weight(3f)
                .background(Color.Gray)
                .padding(8.dp)
        )
    }
}
```

#### 2. 两端对齐（SpaceBetween）
实现“左侧组件左对齐，右侧组件右对齐”的经典布局（如标题+按钮）：

```kotlin
@Composable
fun RowSpaceBetween() {
    Row(
        modifier = Modifier
            .fillMaxWidth()
            .padding(16.dp),
        horizontalArrangement = Arrangement.SpaceBetween,
        verticalAlignment = Alignment.CenterVertically
    ) {
        // 左侧文本
        Text(text = "设置", fontSize = 18.sp)
        // 右侧图标
        Icon(
            imageVector = Icons.Default.ArrowForwardIos,
            contentDescription = "Next",
            modifier = Modifier.size(20.dp)
        )
    }
}
```

#### 3. 文字基线对齐（Baseline）
当 Row 内有不同字号的文本时，使用 `Alignment.Baseline` 保证文字基线对齐（更符合视觉习惯）：

```kotlin
@Composable
fun RowBaselineAlignment() {
    Row(
        modifier = Modifier.padding(16.dp),
        verticalAlignment = Alignment.Baseline // 基线对齐
    ) {
        Text(
            text = "大号文字",
            fontSize = 20.sp,
            modifier = Modifier.padding(end = 8.dp)
        )
        Text(
            text = "小号文字",
            fontSize = 14.sp,
            color = Color.Gray
        )
    }
}
```

#### 4. 横向滚动（Row + horizontalScroll）
当 Row 内子组件总宽度超出屏幕时，添加 `Modifier.horizontalScroll` 实现横向滚动（类似横向列表）：

```kotlin
import androidx.compose.foundation.horizontalScroll
import androidx.compose.foundation.rememberScrollState

@Composable
fun ScrollableRow() {
    // 记住滚动状态（避免重组丢失）
    val scrollState = rememberScrollState()

    Row(
        modifier = Modifier
            .fillMaxWidth()
            .horizontalScroll(scrollState), // 开启横向滚动
        verticalAlignment = Alignment.CenterVertically,
        horizontalArrangement = Arrangement.spacedBy(8.dp) // 子组件间距
    ) {
        // 生成多个子组件（超出屏幕宽度）
        repeat(10) {
            Text(
                text = "标签$it",
                modifier = Modifier
                    .background(Color.LightGray)
                    .padding(8.dp)
                    .clip(RoundedCornerShape(4.dp)) // 圆角
            )
        }
    }
}
```

#### 5. 嵌套 Row/Column
Row 可嵌套 Row/Column 实现复杂布局（如“图标+文字+右侧图标”的复合布局）：

```kotlin
@Composable
fun NestedRow() {
    Row(
        modifier = Modifier
            .fillMaxWidth()
            .padding(16.dp),
        horizontalArrangement = Arrangement.SpaceBetween,
        verticalAlignment = Alignment.CenterVertically
    ) {
        // 左侧嵌套 Row：Icon + Column（两行文字）
        Row(verticalAlignment = Alignment.CenterVertically) {
            Icon(
                imageVector = Icons.Default.Person,
                contentDescription = "Avatar",
                modifier = Modifier
                    .size(40.dp)
                    .padding(end = 8.dp)
            )
            Column {
                Text(text = "张三", fontSize = 16.sp)
                Text(text = "在线", fontSize = 12.sp, color = Color.Green)
            }
        }

        // 右侧按钮
        Button(onClick = { /* 点击事件 */ }) {
            Text(text = "关注")
        }
    }
}
```

### 四、进阶技巧
#### 1. 子组件间距控制
- 方式1：`horizontalArrangement = Arrangement.spacedBy(8.dp)`（统一设置所有子组件间距）；
- 方式2：给单个子组件加 `Modifier.padding(horizontal = 8.dp)`（自定义单个间距）；
- 方式3：使用 `Spacer` 组件（灵活控制任意两个子组件之间的间距）。

```kotlin
import androidx.compose.foundation.layout.Spacer
import androidx.compose.foundation.layout.width

@Composable
fun RowWithSpacer() {
    Row(
        modifier = Modifier.fillMaxWidth().padding(16.dp),
        verticalAlignment = Alignment.CenterVertically
    ) {
        Text(text = "左侧")
        Spacer(modifier = Modifier.width(16.dp)) // 固定宽度的空白间隔
        Text(text = "中间")
        Spacer(modifier = Modifier.weight(1f)) // 占满剩余空间（推右侧组件到最右）
        Text(text = "右侧")
    }
}
```

#### 2. 权重 + 固定宽度混合使用
实现“固定宽度组件 + 权重占满剩余空间 + 固定宽度组件”的经典布局（如搜索栏）：

```kotlin
@Composable
fun SearchBarRow() {
    Row(
        modifier = Modifier
            .fillMaxWidth()
            .padding(16.dp)
            .background(Color.LightGray, RoundedCornerShape(24.dp))
            .padding(horizontal = 16.dp, vertical = 8.dp),
        verticalAlignment = Alignment.CenterVertically
    ) {
        // 固定宽度图标
        Icon(
            imageVector = Icons.Default.Search,
            contentDescription = "Search",
            modifier = Modifier.size(20.dp)
        )
        // 权重占满剩余空间的输入框
        TextField(
            value = "",
            onValueChange = {},
            modifier = Modifier
                .weight(1f)
                .padding(horizontal = 8.dp),
            placeholder = { Text(text = "请输入搜索内容") },
            // 隐藏输入框默认的边框和背景
            colors = TextFieldDefaults.textFieldColors(
                backgroundColor = Color.Transparent,
                focusedIndicatorColor = Color.Transparent,
                unfocusedIndicatorColor = Color.Transparent
            )
        )
        // 固定宽度文字
        Text(text = "搜索", color = Color.Blue)
    }
}
```

#### 3. 点击事件与可点击区域
给 Row 添加点击事件，可通过 `Modifier.clickable` 扩大点击范围：

```kotlin
@Composable
fun ClickableRow() {
    Row(
        modifier = Modifier
            .fillMaxWidth()
            .clickable { // 整行可点击
                Toast.makeText(LocalContext.current, "点击了整行", Toast.LENGTH_SHORT).show()
            }
            .padding(16.dp),
        verticalAlignment = Alignment.CenterVertically
    ) {
        Icon(Icons.Default.Email, contentDescription = "Email")
        Text(text = "邮箱：test@example.com", modifier = Modifier.padding(start = 8.dp))
    }
}
```

### 五、常见问题与解决方案
#### 1. Row 内子组件超出屏幕（无滚动）
- 原因：Row 默认包裹内容，子组件总宽度超出父布局且未开启滚动；
- 解决：添加 `Modifier.horizontalScroll(rememberScrollState())` 开启横向滚动，或调整子组件尺寸/权重。

#### 2. 权重（weight）不生效
- 原因1：Row 未设置 `fillMaxWidth()`（权重需要父布局有足够的宽度）；
- 原因2：子组件同时设置了 `fillMaxWidth()`（与权重冲突）；
- 解决：确保 Row 占满宽度，且权重组件仅用 `weight()` 不叠加 `fillMaxWidth()`。

#### 3. 垂直对齐不符合预期
- 问题：子组件高度不同时，垂直对齐混乱；
- 解决：明确设置 `verticalAlignment`（如 `CenterVertically`/`Baseline`），避免使用默认值。

#### 4. 嵌套 Row 导致布局混乱
- 问题：多层嵌套后子组件位置偏移；
- 解决：给内层 Row 设置明确的 `modifier`（如 `weight`/`width`），或使用 `Box` 辅助定位。

### 六、与 LazyRow 的区别
| 特性 | Row | LazyRow |
|------|-----|---------|
| 渲染方式 | 一次性渲染所有子组件 | 按需渲染可见子组件（性能更高） |
| 适用场景 | 少量子组件（如导航栏、搜索栏） | 大量子组件（如横向列表、标签栏） |
| 滚动支持 | 需手动添加 `horizontalScroll` | 自带滚动（无需额外修饰符） |
| 权重支持 | 原生支持 `weight()` | 不支持权重，需用 `fillParentMaxWidth()` |

> 总结：少量横向布局用 `Row`，大量横向列表用 `LazyRow`。

### 七、总结
`Row` 是 Compose 横向布局的基础，核心掌握：
1. `horizontalArrangement`（水平分布）和 `verticalAlignment`（垂直对齐）的灵活使用；
2. `Modifier.weight()` 实现空间分配；
3. `horizontalScroll` 处理溢出场景；
4. `Spacer` 控制子组件间距。

结合 `Column`（垂直布局）和 `Box`（层叠布局），可组合实现绝大多数业务场景的布局需求。