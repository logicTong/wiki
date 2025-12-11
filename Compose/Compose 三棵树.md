### Compose 绘制组合树：原理与实现详解
Compose 的「组合树（Composition Tree）」是其 UI 渲染体系的核心基础，本质是**描述 UI 结构的不可变节点树**，承载了 UI 的声明式定义、依赖关系和渲染指令。理解组合树的绘制原理，需从「组合（Composition）- 布局（Layout）- 绘制（Drawing）」全流程拆解，同时厘清组合树与其他核心树（布局树、绘制树）的关系。

#### 一、核心概念铺垫
在讲解绘制原理前，先明确 Compose 中与组合树相关的关键术语：
| 术语                | 核心作用                                                                 |
|---------------------|--------------------------------------------------------------------------|
| 组合树（Composition Tree） | 由 `@Composable` 函数执行生成的不可变节点树，描述 UI 结构、状态依赖、参数等 |
| 布局树（Layout Node Tree） | 从组合树衍生的可测量/可布局节点树，承载尺寸、位置计算逻辑                 |
| 绘制树（Draw Node Tree）   | 从布局树衍生的绘制指令树，最终被 GPU 执行渲染                             |
| 重组（Recomposition）     | 状态变化时，仅重新执行变化的 `@Composable` 函数，更新组合树的局部节点     |
| 组合（Composition）       | 首次执行 `@Composable` 函数，生成组合树的过程                             |

#### 二、组合树的生成（组合阶段）
组合树的绘制流程，第一步是「生成组合树」，这是后续所有渲染的基础。

##### 1. 组合的触发条件
- **首次绘制**：Compose 根节点（如 `setContent`）执行，触发根 `@Composable` 函数执行，生成初始组合树。
- **状态变化**：`State`/`MutableState` 等可观察状态更新，触发依赖该状态的 `@Composable` 函数重组，更新组合树局部节点。

##### 2. 组合树节点的核心结构
组合树的每个节点（`Composer.Node`）包含以下关键信息：
- **UI 类型**：如 `Text`/`Box`/`Button` 等组件类型；
- **参数**：组件的入参（如 `Text` 的 `text`、`color`）；
- **状态依赖**：该节点依赖的 `State` 对象（用于重组追踪）；
- **子节点引用**：指向子组合节点的指针；
- **布局/绘制指令**：关联到该节点的布局规则（如 `Modifier.size`）、绘制指令（如 `Modifier.drawBehind`）。

##### 3. 组合的核心逻辑（简化版）
```kotlin
// 伪代码：Composer 执行 Composable 函数生成组合树
class Composer {
    private val compositionTree = mutableListOf<CompositionNode>()
    private var currentNode: CompositionNode? = null

    fun compose(composable: @Composable () -> Unit) {
        // 1. 执行 Composable 函数，遍历 UI 结构
        composable()
        // 2. 构建/更新组合树节点
        buildOrUpdateNode()
        // 3. 标记需要更新的节点（用于后续布局/绘制）
        markDirtyNodes()
    }

    // 为每个 Composable 组件创建/更新节点
    fun emitNode(componentType: String, params: Map<String, Any>, children: () -> Unit) {
        val newNode = CompositionNode(componentType, params)
        currentNode?.addChild(newNode)
        val prevNode = currentNode
        currentNode = newNode
        // 递归处理子组件
        children()
        currentNode = prevNode
        compositionTree.add(newNode)
    }
}

// 示例：调用 Composer 生成组合树
val composer = Composer()
composer.compose {
    Box(modifier = Modifier.size(100.dp)) {
        Text(text = "Hello Compose")
    }
}
```
**核心特点**：
- 组合树是「不可变」的：重组时不会修改原有节点，而是生成新节点替换变化的部分；
- 「跳过重组」优化：未依赖变化状态的 `@Composable` 函数会被跳过，避免无效计算。

#### 三、从组合树到布局树（布局阶段）
组合树仅描述 UI 结构，无法直接绘制——需先转换为「布局树」，完成尺寸和位置计算，这是绘制的前提。

##### 1. 布局树的生成逻辑
Compose 遍历组合树，为每个包含 `Modifier`（布局相关，如 `size`/`padding`/`layout`）的组合节点，创建对应的 `LayoutNode`：
- 布局树节点包含「测量（measure）」和「放置（place）」逻辑；
- 组合树的父子关系，映射为布局树的父子节点（父节点负责约束子节点的尺寸）。

##### 2. 布局计算流程（Measure & Place）
1. **测量阶段**：从根布局节点开始，递归调用子节点的 `measure` 方法：
   - 父节点传递尺寸约束（如最大宽度/高度）；
   - 子节点根据自身 `Modifier`（如 `size(100.dp)`）计算自身尺寸，返回给父节点。
2. **放置阶段**：父节点根据子节点的测量结果，调用 `place` 方法确定子节点的坐标（相对于父节点的偏移）。

**示例**：`Box` 包含 `Text` 的布局计算
```kotlin
// 伪代码：LayoutNode 的 measure 逻辑
class LayoutNode {
    fun measure(constraints: Constraints): MeasuredResult {
        // 1. 应用自身 Modifier 的测量逻辑（如 padding 会缩小可用尺寸）
        val adjustedConstraints = applyModifierMeasure(constraints)
        // 2. 测量子节点
        val childResult = child.measure(adjustedConstraints)
        // 3. 计算自身尺寸（如 Box 默认为子节点尺寸）
        val selfSize = Size(childResult.width, childResult.height)
        // 4. 放置子节点（相对于 Box 的偏移）
        placeChild(childResult, offset = Offset(0f, 0f))
        return MeasuredResult(selfSize)
    }
}
```

#### 四、从布局树到绘制树（绘制阶段）
布局树确定了每个 UI 元素的尺寸和位置后，Compose 会生成「绘制树」，最终将绘制指令提交给 GPU 渲染。

##### 1. 绘制树的核心：绘制指令（DrawCommand）
绘制树的每个节点对应一组「绘制指令」，例如：
- 绘制文本：`DrawTextCommand`（包含文本内容、字体、颜色、坐标）；
- 绘制形状：`DrawShapeCommand`（包含矩形/圆形、颜色、坐标）；
- 绘制背景：`DrawBackgroundCommand`（包含颜色、尺寸）。

这些指令由组合树中的 `Modifier`（如 `background`/`drawBehind`）或组件自身（如 `Text` 内置的文本绘制）生成。

##### 2. 绘制流程的核心步骤
1. **绘制指令收集**：Compose 遍历布局树，从叶子节点到根节点（「逆序」，保证绘制层级：父节点不会覆盖子节点），收集所有绘制指令；
2. **指令排序**：根据 `zIndex`、绘制顺序（如 `drawBehind` 先于内容绘制，`drawOnTop` 后于内容绘制）排序；
3. **提交到渲染线程**：收集的绘制指令被封装为 `Frame`，通过 `ComposeView` 传递给 Android 系统的 `Canvas`（或 OpenGL/Vulkan）；
4. **GPU 渲染**：系统将绘制指令转换为 GPU 可执行的命令，最终在屏幕上渲染像素。

**简化伪代码**：
```kotlin
// 绘制指令收集
fun collectDrawCommands(layoutNode: LayoutNode, commands: MutableList<DrawCommand>) {
    // 1. 先收集子节点的绘制指令（逆序，子节点先绘制，父节点后绘制）
    layoutNode.children.forEach { collectDrawCommands(it, commands) }
    // 2. 收集当前节点的绘制指令（如背景、边框）
    val backgroundCommand = DrawBackgroundCommand(
        color = layoutNode.params["backgroundColor"],
        size = layoutNode.measuredSize,
        position = layoutNode.position
    )
    commands.add(backgroundCommand)
    // 3. 收集组件自身的绘制指令（如 Text 的文字）
    val contentCommand = layoutNode.createContentDrawCommand()
    commands.add(contentCommand)
}

// 提交绘制指令
val drawCommands = mutableListOf<DrawCommand>()
collectDrawCommands(rootLayoutNode, drawCommands)
canvas.render(drawCommands) // 最终绘制到 Canvas
```

#### 五、组合树绘制的核心优化
Compose 之所以高效，核心是对组合树全流程的优化，避免无效绘制：

##### 1. 重组范围最小化
- 通过「状态依赖追踪」，仅重组依赖变化状态的 `@Composable` 函数，组合树仅更新局部节点；
- `remember` 缓存非状态数据，避免重组时重复计算。

##### 2. 布局跳过（Layout Skip）
- 若组合树更新未涉及布局参数（如仅修改文字颜色，未修改尺寸），布局树会跳过测量/放置，直接复用原有尺寸和位置；
- 通过 `Modifier.layoutId` 或 `stable` 类型标记，减少布局重计算。

##### 3. 绘制跳过（Draw Skip）
- 若布局树未变化，且绘制参数（如颜色、文本内容）未变化，绘制树会跳过指令收集，直接复用原有绘制指令；
- 硬件加速：绘制指令直接提交给 GPU，避免 CPU 频繁操作像素。

##### 4. 重组作用域隔离（`rememberCoroutineScope`/`LaunchedEffect`）
- 副作用与组合树解耦，避免副作用触发不必要的重组。

#### 六、关键总结：组合树绘制的完整链路
```
状态变化/首次绘制
    ↓
组合阶段：执行 @Composable 函数 → 生成/更新组合树（描述 UI 结构）
    ↓
布局阶段：遍历组合树 → 生成布局树 → 执行 Measure/Place → 确定尺寸/位置
    ↓
绘制阶段：遍历布局树 → 收集绘制指令 → 排序后提交到 Canvas/GPU
    ↓
屏幕渲染
```

#### 七、常见误区
1. 「组合树 = 布局树 = 绘制树」：三者是不同阶段的产物，组合树描述结构，布局树描述尺寸，绘制树描述渲染指令；
2. 「重组 = 重绘」：重组仅更新组合树，若未涉及布局/绘制参数，不会触发重绘；
3. 「Composable 函数是 UI 组件」：`@Composable` 函数是「组合函数」，仅负责生成组合树节点，并非直接渲染的组件。

通过以上流程，Compose 实现了「声明式 UI」的核心目标：开发者只需描述 UI 应该是什么样（组合树），框架自动处理布局和绘制的细节，同时通过分层优化保证性能。