`LaunchedEffect` 是 Jetpack Compose 中用于**在组合函数内启动协程**的核心组件，专为“组合生命周期绑定的异步操作”设计（如网络请求、延迟任务、数据流收集），是 Compose 中处理协程的首选方式，避免了手动管理协程生命周期的风险。

### 一、核心特性
1. **生命周期绑定**：协程会在 `LaunchedEffect` 进入组合时启动，退出组合时自动取消（如组件被销毁、重组后不再执行）；
2. **触发条件可控**：通过 `key` 参数控制协程是否重新执行（仅当 `key` 变化时，重启协程）；
3. **挂起函数支持**：内部可直接调用挂起函数（如 `delay`、网络请求、Flow 收集）；
4. **无返回值**：仅用于执行副作用（不直接参与 UI 渲染）。

### 二、基本用法
#### 1. 最简示例（启动协程执行异步操作）
```kotlin
import androidx.compose.foundation.layout.Column
import androidx.compose.material.Button
import androidx.compose.material.Text
import androidx.compose.runtime.Composable
import androidx.compose.runtime.getValue
import androidx.compose.runtime.mutableStateOf
import androidx.compose.runtime.remember
import androidx.compose.runtime.setValue
import androidx.compose.ui.platform.LocalContext
import androidx.lifecycle.viewmodel.compose.viewModel
import kotlinx.coroutines.delay
import androidx.compose.runtime.LaunchedEffect

@Composable
fun BasicLaunchedEffect() {
    // 状态：控制UI展示
    var count by remember { mutableStateOf(0) }
    val context = LocalContext.current

    // LaunchedEffect：进入组合时启动协程，退出组合时取消
    LaunchedEffect(key1 = Unit) { // key为Unit：仅在首次组合时执行一次
        delay(2000) // 模拟异步操作（如网络请求）
        count = 10 // 更新状态，触发UI重组
        Toast.makeText(context, "异步操作完成", Toast.LENGTH_SHORT).show()
    }

    // UI展示
    Column {
        Text(text = "计数：$count")
    }
}
```
**核心逻辑**：
- 组件首次渲染时，`LaunchedEffect` 启动协程，延迟 2 秒后更新 `count`；
- 若组件被销毁（如退出页面），协程会自动取消，避免内存泄漏；
- `key1 = Unit` 表示“仅执行一次”（Unit 永远不变）。

#### 2. 核心参数说明
| 参数 | 作用 | 示例 |
|------|------|------|
| `key1`/`key2`/`key3` | 触发协程重启的关键值（任意类型）：<br>1. 首次组合时必执行；<br>2. 重组时若 `key` 变化，先取消旧协程，再启动新协程；<br>3. 传 `Unit` 表示仅执行一次 | `key1 = count`、`key1 = listOf(id, name)` |
| `block` | 协程执行体（可调用挂起函数） | `{ delay(1000); loadData() }` |

### 三、核心场景实战
#### 1. 依赖参数变化重启协程（动态 key）
当 `key` 变化时，`LaunchedEffect` 会取消旧协程并启动新协程，适用于“参数变化后重新执行异步操作”的场景（如根据 ID 加载数据）：
```kotlin
@Composable
fun LaunchedEffectWithDynamicKey(itemId: Int) {
    var data by remember { mutableStateOf("") }

    // key为itemId：itemId变化时，重新加载数据
    LaunchedEffect(key1 = itemId) {
        // 模拟根据ID请求数据（挂起函数）
        data = loadDataById(itemId)
    }

    Text(text = "加载的数据：$data")
}

// 模拟挂起函数（网络请求/数据库查询）
suspend fun loadDataById(id: Int): String {
    delay(1000)
    return "数据ID：$id"
}
```
**逻辑**：
- 首次调用 `LaunchedEffectWithDynamicKey(1)` 时，加载 ID=1 的数据；
- 若 `itemId` 变为 2，旧协程被取消，启动新协程加载 ID=2 的数据。

#### 2. 收集 Flow 数据流（生命周期绑定）
`LaunchedEffect` 是 Compose 中收集 Flow 的标准方式（替代 `collectAsState`，适用于需要手动处理收集逻辑的场景）：
```kotlin
import kotlinx.coroutines.flow.flow
import kotlinx.coroutines.flow.collect

// ViewModel中定义Flow
class DataViewModel : ViewModel() {
    val dataFlow = flow {
        repeat(3) {
            delay(1000)
            emit("实时数据 $it")
        }
    }
}

@Composable
fun CollectFlowInLaunchedEffect(viewModel: DataViewModel = viewModel()) {
    var flowData by remember { mutableStateOf("") }

    // 收集Flow：组件销毁时自动取消收集
    LaunchedEffect(key1 = viewModel.dataFlow) {
        viewModel.dataFlow.collect { value ->
            flowData = value // 更新UI状态
        }
    }

    Text(text = "Flow数据：$flowData")
}
```
**优势**：无需手动调用 `flow.collect().cancel()`，协程取消时 Flow 收集自动停止。

#### 3. 延迟操作/防抖（如输入框搜索）
结合 `delay` 实现防抖，避免频繁触发异步操作：
```kotlin
@Composable
fun DebounceWithLaunchedEffect() {
    var searchText by remember { mutableStateOf("") }
    var searchResult by remember { mutableStateOf("") }
    val scope = rememberCoroutineScope() // 仅用于对比，推荐用LaunchedEffect

    TextField(
        value = searchText,
        onValueChange = { newText ->
            searchText = newText
        },
        placeholder = { Text("请输入搜索内容") }
    )

    // 搜索文本变化时，延迟500ms执行搜索（防抖）
    LaunchedEffect(key1 = searchText) {
        delay(500) // 防抖：500ms内文本不变才执行
        if (searchText.isNotEmpty()) {
            searchResult = searchData(searchText) // 模拟搜索
        } else {
            searchResult = ""
        }
    }

    Text(text = "搜索结果：$searchResult")
}

suspend fun searchData(text: String): String {
    delay(1000)
    return "搜索「$text」的结果"
}
```

#### 4. 配合 SideEffect/DisposableEffect 处理生命周期
`LaunchedEffect` 专注于协程，若需处理非协程的生命周期副作用（如注册/取消监听），可配合 `DisposableEffect`：
```kotlin
import androidx.compose.runtime.DisposableEffect

@Composable
fun LifecycleSideEffectDemo() {
    // 1. LaunchedEffect：协程异步操作
    LaunchedEffect(key1 = Unit) {
        println("协程启动")
        delay(1000)
        println("协程执行完成")
    }

    // 2. DisposableEffect：非协程生命周期操作
    DisposableEffect(key1 = Unit) {
        // 进入组合时执行（如注册监听）
        println("注册监听")
        
        // 退出组合时执行（如取消监听）
        onDispose {
            println("取消监听")
        }
    }
}
```

### 四、关键注意事项
#### 1. 与 rememberCoroutineScope 的区别
| 特性 | `LaunchedEffect` | `rememberCoroutineScope` |
|------|------------------|--------------------------|
| 生命周期 | 自动绑定组合生命周期（自动取消） | 手动管理（需在 `onDispose`/按钮点击中启动） |
| 触发时机 | 进入组合/key变化时自动启动 | 手动调用 `launch` 才启动 |
| 适用场景 | 自动执行的异步操作（如初始化、数据加载） | 手动触发的异步操作（如按钮点击、下拉刷新） |

示例对比（按钮点击触发协程）：
```kotlin
@Composable
fun CoroutineScopeVsLaunchedEffect() {
    val scope = rememberCoroutineScope() // 手动协程作用域
    var text by remember { mutableStateOf("") }

    // 手动触发（按钮点击）
    Button(onClick = {
        scope.launch {
            delay(1000)
            text = "手动协程执行完成"
        }
    }) {
        Text("执行手动协程")
    }

    // 自动触发（首次组合）
    LaunchedEffect(key1 = Unit) {
        delay(1000)
        text = "LaunchedEffect执行完成"
    }

    Text(text = text)
}
```

#### 2. 常见错误与解决方案
| 错误场景 | 原因 | 解决方案 |
|----------|------|----------|
| 协程重复执行 | `key` 设置不当（如每次重组生成新对象，如 `key1 = mutableListOf()`） | 确保 `key` 是稳定值（如基本类型、`remember` 包裹的对象） |
| 内存泄漏 | 协程中持有长生命周期对象（如 Activity）且未取消 | 依赖 `LaunchedEffect` 自动取消特性，避免手动持有上下文 |
| 挂起函数阻塞UI | 在 `LaunchedEffect` 中执行耗时同步操作 | 耗时操作需封装为挂起函数（如 `withContext(Dispatchers.IO)`） |

#### 3. 进阶：多 key 触发
可通过 `key1`/`key2`/`key3` 传入多个关键值，**任意一个 key 变化**都会重启协程：
```kotlin
LaunchedEffect(key1 = userId, key2 = pageNum) {
    // userId 或 pageNum 变化时，重新加载分页数据
    loadPageData(userId, pageNum)
}
```
若 key 超过 3 个，可封装为列表/数据类（需确保 `equals` 生效）：
```kotlin
val keys = remember { listOf(userId, pageNum, keyword) }
LaunchedEffect(key1 = keys) {
    loadData(keys)
}
```

### 五、总结
`LaunchedEffect` 是 Compose 中处理“组合生命周期绑定的协程操作”的核心：
1. 核心价值：**自动管理协程生命周期**（启动/取消），避免内存泄漏；
2. 核心参数：`key` 控制协程是否重启，`block` 执行异步操作；
3. 核心场景：数据加载、Flow 收集、延迟/防抖操作、初始化异步逻辑；
4. 避坑点：确保 `key` 是稳定值，区分“自动执行”（`LaunchedEffect`）和“手动执行”（`rememberCoroutineScope`）。

掌握 `LaunchedEffect` 是 Compose 异步编程的基础，结合 `State`/`Flow` 可实现高效、安全的异步 UI 交互。