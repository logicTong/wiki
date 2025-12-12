### Compose 中 SavedStateHandle 全解析
`SavedStateHandle` 是 Android Jetpack 提供的**状态持久化工具**，核心作用是：在 ViewModel 中保存「需要跨进程/跨配置变更（如屏幕旋转、语言切换）、甚至进程重启后恢复的状态」，常与 Compose 结合实现“状态不丢失”的 UI 开发。

> 核心优势：
> 1. 自动处理配置变更（屏幕旋转、分屏）时的状态保存/恢复；
> 2. 支持进程被杀后（如系统回收内存）恢复关键状态；
> 3. 与 ViewModel 深度集成，符合 Compose “单向数据流”设计思想。

---

## 一、核心原理与适用场景
### 1. 核心原理
- `SavedStateHandle` 本质是一个「键值对容器」，内部通过 `Bundle` 实现状态的序列化/反序列化；
- 绑定到 ViewModel 生命周期：ViewModel 由 `ViewModelProvider` 创建时，可传入 `SavedStateHandle`，其状态会随 `SavedStateRegistry` 自动保存（配置变更/进程重启）；
- 支持监听状态变化：提供 `getStateFlow` 方法，返回与键绑定的 `StateFlow`，天然适配 Compose 订阅。

### 2. 适用场景
| 场景                          | 是否适合用 SavedStateHandle | 原因                                                                 |
|-------------------------------|-----------------------------|----------------------------------------------------------------------|
| 屏幕旋转后保留表单输入内容    | ✅ 是                       | 配置变更，需持久化轻量状态                                           |
| 进程重启后恢复分页列表的页码  | ✅ 是                       | 进程被杀后，需恢复关键业务状态                                       |
| 临时的 UI 状态（如按钮是否点击）| ❌ 否                       | 无需跨进程/配置变更保存，用 Compose `mutableStateOf` 即可             |
| 大量数据（如列表数据）| ❌ 否                       | 序列化成本高，建议用数据库/SP 存储                                   |

---

## 二、基础使用步骤（ViewModel + SavedStateHandle + Compose）
### 步骤1：依赖配置（确保引入）
```gradle
// ViewModel + SavedState 核心依赖
implementation "androidx.lifecycle:lifecycle-viewmodel-savedstate:2.7.0"
// Compose 与 ViewModel 集成
implementation "androidx.lifecycle:lifecycle-viewmodel-compose:2.7.0"
```

### 步骤2：ViewModel 中注入 SavedStateHandle
通过「构造函数注入」获取 `SavedStateHandle`，并封装状态的读写逻辑：
```kotlin
import androidx.lifecycle.SavedStateHandle
import androidx.lifecycle.ViewModel
import androidx.lifecycle.viewModelScope
import kotlinx.coroutines.delay
import kotlinx.coroutines.flow.StateFlow
import kotlinx.coroutines.flow.asStateFlow
import kotlinx.coroutines.flow.getStateFlow
import kotlinx.coroutines.launch

// 定义状态键（常量管理，避免硬编码）
private const val KEY_COUNTER = "counter"
private const val KEY_USER_INPUT = "user_input"

class SavedStateViewModel(
    private val savedStateHandle: SavedStateHandle // 构造函数注入（由系统自动提供）
) : ViewModel() {
    // 1. 基础类型状态：通过 getStateFlow 绑定 SavedStateHandle，返回 StateFlow
    // 第一个参数：键；第二个参数：默认值（无保存状态时使用）
    val counter: StateFlow<Int> = savedStateHandle.getStateFlow(KEY_COUNTER, 0)

    // 2. 字符串状态（如表单输入）
    val userInput: StateFlow<String> = savedStateHandle.getStateFlow(KEY_USER_INPUT, "")

    // 3. 修改状态（通过 SavedStateHandle 更新，自动持久化）
    fun incrementCounter() {
        // 获取当前值 → 修改 → 保存回 SavedStateHandle
        val current = savedStateHandle[KEY_COUNTER] ?: 0
        savedStateHandle[KEY_COUNTER] = current + 1
    }

    fun updateUserInput(input: String) {
        savedStateHandle[KEY_USER_INPUT] = input
    }

    // 示例：异步更新状态（仍会被持久化）
    fun simulateAsyncUpdate() {
        viewModelScope.launch {
            delay(1000)
            savedStateHandle[KEY_COUNTER] = 100
        }
    }
}
```

### 步骤3：Compose 中使用 ViewModel + 收集状态
通过 `viewModel()` 方法自动注入带 `SavedStateHandle` 的 ViewModel，收集状态并更新 UI：
```kotlin
import androidx.compose.foundation.layout.Column
import androidx.compose.foundation.layout.Spacer
import androidx.compose.foundation.layout.height
import androidx.compose.foundation.layout.padding
import androidx.compose.material3.Button
import androidx.compose.material3.OutlinedTextField
import androidx.compose.material3.Text
import androidx.compose.runtime.collectAsStateWithLifecycle
import androidx.compose.ui.Modifier
import androidx.compose.ui.unit.dp
import androidx.lifecycle.viewmodel.compose.viewModel

@Composable
fun SavedStateDemoScreen(
    // 自动创建带 SavedStateHandle 的 ViewModel
    viewModel: SavedStateViewModel = viewModel()
) {
    // 收集状态（生命周期感知，自动重组）
    val counter by viewModel.counter.collectAsStateWithLifecycle()
    val userInput by viewModel.userInput.collectAsStateWithLifecycle()

    Column(modifier = Modifier.padding(16.dp)) {
        // 展示计数（屏幕旋转后仍保留）
        Text(text = "计数：$counter", fontSize = 20.sp)
        
        Spacer(modifier = Modifier.height(16.dp))
        
        // 表单输入（屏幕旋转后不丢失）
        OutlinedTextField(
            value = userInput,
            onValueChange = { viewModel.updateUserInput(it) },
            label = { Text("输入框（持久化）") }
        )
        
        Spacer(modifier = Modifier.height(16.dp))
        
        // 触发状态修改
        Button(onClick = { viewModel.incrementCounter() }) {
            Text("计数+1")
        }
        
        Button(onClick = { viewModel.simulateAsyncUpdate() }) {
            Text("异步更新计数为100")
        }
    }
}
```

---

## 三、核心 API 详解
### 1. `SavedStateHandle` 常用方法
| 方法                          | 作用                                                                 | 示例                                  |
|-------------------------------|----------------------------------------------------------------------|---------------------------------------|
| `getStateFlow(key, initial)`  | 返回与键绑定的 `StateFlow`，状态变化自动持久化                       | `getStateFlow("counter", 0)`          |
| `get(key)` / `operator []`    | 获取指定键的状态值（可空）| `savedStateHandle["counter"]`         |
| `set(key, value)` / `operator []=` | 设置状态值（自动持久化）| `savedStateHandle["counter"] = 10`    |
| `remove(key)`                 | 移除指定键的状态                                                     | `savedStateHandle.remove("counter")`  |
| `contains(key)`               | 判断是否包含指定键                                                   | `savedStateHandle.contains("counter")`|
| `keys`                        | 获取所有已保存的键集合                                               | `savedStateHandle.keys`               |

### 2. 关键注意事项：可序列化限制
`SavedStateHandle` 内部通过 `Bundle` 序列化状态，因此**仅支持 Bundle 可序列化的类型**：
✅ 支持的类型：`Int`/`Long`/`String`/`Boolean`/`Float`/`Double`、`ArrayList`、`Bundle` 等；
❌ 不支持的类型：自定义数据类（需手动序列化）、`List`（需转 `ArrayList`）、`Flow`/`StateFlow` 等。

#### 解决：自定义数据类的持久化
若需保存自定义数据类，需先序列化为 `String`（如 JSON），再存入 `SavedStateHandle`：
```kotlin
import com.google.gson.Gson

// 自定义数据类
data class User(val id: Int, val name: String)

class CustomDataViewModel(savedStateHandle: SavedStateHandle) : ViewModel() {
    private val gson = Gson()
    private const val KEY_USER = "user"

    // 封装自定义数据类的读写
    var user: User?
        get() {
            val json = savedStateHandle[KEY_USER] ?: return null
            return gson.fromJson(json, User::class.java)
        }
        set(value) {
            val json = value?.let { gson.toJson(it) }
            savedStateHandle[KEY_USER] = json
        }

    // 转换为 StateFlow 供 Compose 订阅
    val userState: StateFlow<User?> = savedStateHandle.getStateFlow(KEY_USER, null)
}
```

---

## 四、进阶用法
### 1. 与 `stateIn`/`combine` 结合
将 `SavedStateHandle` 的状态与其他 Flow 组合，实现更复杂的状态联动：
```kotlin
class CombinedViewModel(savedStateHandle: SavedStateHandle) : ViewModel() {
    private val counter = savedStateHandle.getStateFlow(KEY_COUNTER, 0)
    private val isDarkMode = savedStateHandle.getStateFlow("dark_mode", false)

    // 组合状态并转换为 StateFlow
    val uiState = combine(counter, isDarkMode) { count, dark ->
        "计数：$count，暗黑模式：${if (dark) "开启" else "关闭"}"
    }.stateIn(
        scope = viewModelScope,
        started = androidx.lifecycle.processing.SharingStarted.WhileSubscribed(),
        initialValue = "初始状态"
    )

    fun toggleDarkMode() {
        val current = savedStateHandle["dark_mode"] ?: false
        savedStateHandle["dark_mode"] = !current
    }
}
```

### 2. 手动控制状态保存/恢复（特殊场景）
若需手动触发状态保存（如页面退后台时），可通过 `SavedStateRegistry`，但日常开发无需手动操作（系统自动处理）：
```kotlin
import androidx.compose.runtime.Composable
import androidx.lifecycle.SavedStateRegistryOwner
import androidx.lifecycle.viewmodel.compose.LocalViewModelStoreOwner

@Composable
fun ManualSaveStateDemo() {
    val owner = LocalViewModelStoreOwner.current as? SavedStateRegistryOwner
    owner?.savedStateRegistry?.performSave(Bundle()) // 手动保存状态
}
```

---

## 五、常见问题与避坑指南
### 1. 状态恢复失败？
- 原因1：状态类型不可序列化（如自定义数据类未处理）；
- 原因2：ViewModel 未通过 `viewModel()` 或 `ViewModelProvider` 创建（导致 `SavedStateHandle` 未注入）；
- 解决：确保类型可序列化，且 ViewModel 由系统托管创建。

### 2. 进程重启后状态仍丢失？
- 原因：`SavedStateHandle` 仅保存到「临时存储」（如 Bundle），进程彻底被杀且未触发保存时会丢失；
- 解决：核心数据（如用户登录态）需结合 `DataStore`/`SharedPreferences` 持久化，`SavedStateHandle` 仅用于临时状态。

### 3. 与 Compose `mutableStateOf` 的区别？
| 特性                | SavedStateHandle       | Compose mutableStateOf |
|---------------------|------------------------|------------------------|
| 配置变更恢复        | ✅ 支持                 | ❌ 不支持（重组后重置） |
| 进程重启恢复        | ✅ 支持（部分场景）| ❌ 不支持               |
| 适用范围            | 跨生命周期的关键状态    | 组件内临时 UI 状态      |
| 集成方式            | 需通过 ViewModel       | 直接在 Compose 中使用  |

---

## 六、最佳实践总结
1. **职责划分**：
   - 临时 UI 状态（如按钮选中、弹窗显示）：用 Compose `mutableStateOf`；
   - 跨配置变更/进程重启的状态（如表单输入、分页页码）：用 `SavedStateHandle`；
   - 大量/核心业务数据（如用户信息、列表数据）：用 `DataStore`/Room 持久化。

2. **编码规范**：
   - 用常量管理状态键（避免硬编码）；
   - 封装状态读写逻辑（ViewModel 内暴露 `StateFlow`，不直接暴露 `SavedStateHandle`）；
   - 避免保存大体积数据（优先用数据库）。

3. **Compose 集成**：
   - 始终用 `collectAsStateWithLifecycle` 收集 `SavedStateHandle` 导出的 `StateFlow`；
   - ViewModel 通过 `viewModel()` 自动注入，无需手动创建 `SavedStateHandle`。

`SavedStateHandle` 是 Compose + ViewModel 开发中实现“状态持久化”的核心工具，重点解决「配置变更导致状态丢失」的问题，合理使用可大幅提升用户体验。