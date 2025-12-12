### Compose 中使用 StateFlow 全解析
StateFlow 是 Kotlin 协程中**可观察的状态容器**（属于冷流，默认值+仅发射最新值），在 Compose 中结合 `collectAsStateWithLifecycle` 可实现「生命周期感知的状态订阅」，是 Compose 与 ViewModel/数据层通信的核心方式之一，比 LiveData 更贴合 Kotlin 协程生态。

> 核心优势：自动感知 Compose 组件的生命周期（如页面退到后台时暂停收集，前台恢复），避免内存泄漏；仅发射最新状态，减少无用重绘。

---

## 一、核心依赖（必加）
确保项目引入以下依赖（适配 Compose + 协程 + StateFlow）：
```gradle
// 1. 协程核心（必选）
implementation "org.jetbrains.kotlinx:kotlinx-coroutines-android:1.7.3"
// 2. Compose 生命周期感知收集（核心）
implementation "androidx.lifecycle:lifecycle-runtime-compose:2.7.0"
// 3. ViewModel + Compose（若用 ViewModel 托管 StateFlow）
implementation "androidx.lifecycle:lifecycle-viewmodel-compose:2.7.0"
```

---

## 二、基础使用流程（ViewModel + StateFlow + Compose）
### 步骤1：ViewModel 中定义 StateFlow
将 StateFlow 托管在 ViewModel 中（利用 ViewModel 生命周期，避免重建），通过 `MutableStateFlow` 修改状态，对外暴露只读的 `StateFlow`：
```kotlin
import androidx.lifecycle.ViewModel
import androidx.lifecycle.viewModelScope
import kotlinx.coroutines.delay
import kotlinx.coroutines.flow.MutableStateFlow
import kotlinx.coroutines.flow.StateFlow
import kotlinx.coroutines.launch

class CounterViewModel : ViewModel() {
    // 1. 定义可变 StateFlow（私有，仅内部修改）
    private val _counter = MutableStateFlow(0)
    // 2. 对外暴露只读 StateFlow（防止外部直接修改）
    val counter: StateFlow<Int> = _counter

    // 3. 业务逻辑：修改 StateFlow 的值
    fun increment() {
        _counter.value += 1 // StateFlow.value 是线程安全的
    }

    // 示例：协程中异步更新 StateFlow
    fun simulateAsyncUpdate() {
        viewModelScope.launch {
            delay(1000) // 模拟网络/数据库请求
            _counter.value = 10
        }
    }
}
```

### 步骤2：Compose 中收集 StateFlow 状态
使用 `collectAsStateWithLifecycle`（生命周期感知）将 StateFlow 转换为 Compose 可观察的 `State`，当 StateFlow 发射新值时，Compose 自动重组：
```kotlin
import androidx.compose.foundation.layout.Column
import androidx.compose.foundation.layout.Spacer
import androidx.compose.foundation.layout.height
import androidx.compose.material3.Button
import androidx.compose.material3.Text
import androidx.compose.runtime.collectAsStateWithLifecycle
import androidx.compose.ui.Modifier
import androidx.compose.ui.unit.dp
import androidx.lifecycle.viewmodel.compose.viewModel

@Composable
fun StateFlowDemoScreen(
    viewModel: CounterViewModel = viewModel() // 注入 ViewModel
) {
    // 核心：将 StateFlow 转换为 Compose State（自动订阅+生命周期感知）
    val counterState by viewModel.counter.collectAsStateWithLifecycle()

    Column {
        // 展示状态（counterState 变化时自动重组）
        Text(text = "当前计数：$counterState", fontSize = 20.sp)
        
        Spacer(modifier = Modifier.height(16.dp))
        
        // 触发状态修改
        Button(onClick = { viewModel.increment() }) {
            Text("点击加1")
        }
        
        Button(onClick = { viewModel.simulateAsyncUpdate() }) {
            Text("模拟异步更新")
        }
    }
}
```

---

## 三、关键 API 详解
### 1. `collectAsStateWithLifecycle`（推荐）
- **作用**：将 StateFlow 转换为 Compose `State`，且**感知组件生命周期**（如 Activity 退到后台时暂停收集，前台恢复）；
- **参数**：
  - `minActiveState`：最小活跃状态（默认 `Lifecycle.State.STARTED`，即前台可见时收集）；
  - `context`：协程上下文（默认用 `LocalCoroutineScope.current`）。
- **对比 `collectAsState`**：
  | 特性                | `collectAsState`       | `collectAsStateWithLifecycle` |
  |---------------------|------------------------|--------------------------------|
  | 生命周期感知        | 无（后台仍收集）       | 有（后台暂停，避免无用消耗）   |
  | 内存泄漏风险        | 有（后台持续收集）     | 无（适配生命周期）             |
  | 推荐场景            | 测试/简单场景          | 生产环境（Activity/Fragment）  |

### 2. `MutableStateFlow` 核心操作
| 操作                | 说明                                  | 示例                          |
|---------------------|---------------------------------------|-------------------------------|
| `value`             | 获取/设置当前值（线程安全）           | `_counter.value = 10`         |
| `update`            | 基于当前值修改（原子操作）            | `_counter.update { it + 1 }`  |
| `compareAndSet`     | 比较并设置（CAS 操作）                | `_counter.compareAndSet(0, 1)`|

---

## 四、进阶用法
### 1. 自定义数据类的 StateFlow
场景：管理复杂状态（如用户信息、列表数据），避免多个 StateFlow 分散：
```kotlin
// 1. 定义状态数据类
data class UiState(
    val isLoading: Boolean = false,
    val data: List<String> = emptyList(),
    val error: String? = null
)

// 2. ViewModel 中托管
class ListViewModel : ViewModel() {
    private val _uiState = MutableStateFlow(UiState(isLoading = true))
    val uiState: StateFlow<UiState> = _uiState

    // 模拟加载数据
    fun loadData() {
        viewModelScope.launch {
            // 加载中
            _uiState.value = _uiState.value.copy(isLoading = true)
            try {
                delay(1500)
                // 加载成功
                _uiState.value = _uiState.value.copy(
                    isLoading = false,
                    data = listOf("Item1", "Item2", "Item3")
                )
            } catch (e: Exception) {
                // 加载失败
                _uiState.value = _uiState.value.copy(
                    isLoading = false,
                    error = e.message
                )
            }
        }
    }
}

// 3. Compose 中使用
@Composable
fun ListScreen(viewModel: ListViewModel = viewModel()) {
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()

    when {
        uiState.isLoading -> Text("加载中...")
        uiState.error != null -> Text("错误：${uiState.error}")
        else -> {
            // 展示列表
            androidx.compose.foundation.lazy.LazyColumn {
                items(uiState.data) { item ->
                    Text(text = item, modifier = Modifier.padding(16.dp))
                }
            }
        }
    }

    Button(onClick = { viewModel.loadData() }) {
        Text("加载数据")
    }
}
```

### 2. 多个 StateFlow 组合
场景：需合并多个 StateFlow 的状态（如同时监听“用户信息”和“权限状态”），用 `combine` 操作符：
```kotlin
import kotlinx.coroutines.flow.combine

class CombinedViewModel : ViewModel() {
    private val _userName = MutableStateFlow("Alice")
    private val _isLogin = MutableStateFlow(false)
    
    // 组合两个 StateFlow
    val combinedState = combine(_userName, _isLogin) { name, isLogin ->
        "用户：$name，登录状态：${if (isLogin) "已登录" else "未登录"}"
    }.stateIn(
        scope = viewModelScope,
        started = androidx.lifecycle.processing.StartLifecycleState, // 适配生命周期
        initialValue = "加载中..."
    )
}

// Compose 中收集组合后的流
@Composable
fun CombinedScreen(viewModel: CombinedViewModel = viewModel()) {
    val combinedText by viewModel.combinedState.collectAsStateWithLifecycle()
    Text(text = combinedText)
    
    Button(onClick = { 
        viewModel._isLogin.value = true // 模拟登录
        viewModel._userName.value = "Bob"
    }) {
        Text("登录并修改用户名")
    }
}
```

### 3. 手动收集 StateFlow（不推荐，仅特殊场景）
若需自定义收集逻辑（如处理异常、手动控制生命周期），可使用 `LaunchedEffect` + `collect`：
```kotlin
@Composable
fun ManualCollectDemo(viewModel: CounterViewModel = viewModel()) {
    var counter by remember { mutableStateOf(0) }

    // LaunchedEffect：组件首次组合/key 变化时启动协程
    LaunchedEffect(Unit) {
        // 手动收集 StateFlow（无生命周期感知，需自己处理）
        viewModel.counter.collect { newCount ->
            counter = newCount
            // 自定义逻辑：如日志、埋点
            println("计数更新：$newCount")
        }
    }

    Text(text = "手动收集：$counter")
}
```

---

## 五、注意事项
### 1. 避免直接暴露 `MutableStateFlow`
- 对外仅暴露只读的 `StateFlow`（`val counter: StateFlow<Int>`），内部用 `MutableStateFlow` 修改，保证状态封装性；
- 错误示例：`val counter = MutableStateFlow(0)`（外部可直接修改 `counter.value`，破坏单一数据源）。

### 2. 生命周期感知的关键
- 务必使用 `collectAsStateWithLifecycle`（而非 `collectAsState`），否则后台仍会收集流，导致不必要的资源消耗；
- 若在非 Activity/Fragment 场景（如 Dialog），可自定义 `minActiveState`：
  ```kotlin
  val counter by viewModel.counter.collectAsStateWithLifecycle(
      minActiveState = Lifecycle.State.RESUMED // 仅前台可见时收集
  )
  ```

### 3. 初始值必须设置
StateFlow 是**有默认值的流**，创建时必须指定初始值（`MutableStateFlow(0)`），避免空值问题。

### 4. 避免频繁发射状态
- StateFlow 仅发射“与当前值不同”的新值（防抖），但仍需避免高频发射（如每秒几十次），否则导致 Compose 频繁重组；
- 优化：用 `debounce`/`distinctUntilChanged` 过滤重复值。

### 5. 与 Compose State 的区别
| 特性                | StateFlow                | Compose State（mutableStateOf） |
|---------------------|--------------------------|---------------------------------|
| 作用域              | 跨组件（ViewModel → UI） | 单组件内（仅 Compose 内部）     |
| 生命周期            | 跟随 ViewModel           | 跟随 Compose 重组               |
| 协程支持            | 原生支持异步更新         | 需配合 LaunchedEffect 异步修改  |
| 多订阅者            | 支持（多 UI 订阅同一流） | 仅当前组件可见                 |

---

## 六、典型应用场景
1. **ViewModel → UI 状态通信**：如网络请求结果、列表数据、加载状态、错误信息；
2. **跨组件状态共享**：如 App 全局状态（登录状态、主题设置），可结合 `kotlinx-coroutines-core` 的 `SharedFlow`/`StateFlow` 实现；
3. **异步数据更新**：如数据库查询结果、WebSocket 消息推送，通过 StateFlow 转发到 UI。

---

## 总结
Compose 中使用 StateFlow 的核心流程：
1. **ViewModel 中定义**：`MutableStateFlow`（内部修改） + 只读 `StateFlow`（对外暴露）；
2. **Compose 中收集**：`collectAsStateWithLifecycle` 转换为 Compose `State`，自动重组；
3. **遵循最佳实践**：封装状态、生命周期感知、避免直接暴露可变流。

这种模式符合 Compose 的“单向数据流”设计思想，是生产环境中实现“数据层 → UI 层”通信的首选方式。