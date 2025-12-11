在Jetpack Compose中，状态管理是构建可组合、响应式UI的核心。Compose的状态管理遵循**单向数据流**原则，强调状态的单一数据源和不可变性（或可观察性），主要分为**本地状态**和**共享状态**两大类，以下是详细解析：

### 一、本地状态（Local State）
本地状态是仅在单个Composable函数或其直接子组件中使用的状态，通常使用`remember`和`mutableStateOf`来创建，是最基础的状态管理方式。

#### 1. 核心API：`mutableStateOf` & `remember`
- **`mutableStateOf`**：创建一个可观察的状态对象（`MutableState<T>`），当状态值变化时，Compose会自动重组使用该状态的UI部分。
- **`remember`**：在Compose的重组过程中保留状态对象，避免每次重组都重新创建状态（若没有`remember`，状态会在重组时被重置）。

**示例：计数器（本地状态）**
```kotlin
@Composable
fun Counter() {
    // 本地状态：使用remember+mutableStateOf创建
    val count by remember { mutableStateOf(0) }
    
    Column(
        modifier = Modifier.padding(16.dp),
        horizontalAlignment = Alignment.CenterHorizontally
    ) {
        Text(text = "Count: $count", fontSize = 24.sp)
        Spacer(modifier = Modifier.height(16.dp))
        Button(onClick = { count++ }) {
            Text("Increment")
        }
    }
}
```
- 这里`count`通过**委托属性**（`by`）简化了对`MutableState`的`value`属性的访问（等价于`val countState = remember { mutableStateOf(0) }`，然后`countState.value++`）。
- 当点击按钮修改`count`时，Compose会重组`Text`和`Button`所在的UI区域（仅依赖该状态的部分）。

#### 2. `rememberSaveable`：持久化本地状态
`remember`仅在Composable的**重组**中保留状态，但当Composable被**销毁并重建**（如屏幕旋转、Activity重建）时，状态会丢失。`rememberSaveable`可以将状态序列化并保存到Bundle中，实现状态的持久化。

**示例：持久化计数器**
```kotlin
@Composable
fun PersistentCounter() {
    // 持久化状态：屏幕旋转后仍保留数值
    val count by rememberSaveable { mutableStateOf(0) }
    
    // 其余代码与Counter相同...
}
```
- `rememberSaveable`支持基本数据类型（Int、String、Boolean等）、Parcelable、Serializable对象，若需自定义类型，可通过`Saver`实现序列化。

### 二、共享状态（Shared State）
当多个Composable组件（甚至跨页面）需要共享状态时，本地状态无法满足需求，此时需要**提升状态**（State Hoisting）或使用更高级的状态管理方案。

#### 1. 状态提升（State Hoisting）
**核心思想**：将状态从子Composable提升到共同的父Composable中，子组件通过回调函数接收状态更新逻辑，实现状态的共享和单向数据流。

**示例：父子组件共享状态**
```kotlin
// 子组件：无状态（Stateless），仅接收状态和回调
@Composable
fun CounterDisplay(count: Int, onIncrement: () -> Unit) {
    Column(
        modifier = Modifier.padding(16.dp),
        horizontalAlignment = Alignment.CenterHorizontally
    ) {
        Text(text = "Count: $count", fontSize = 24.sp)
        Spacer(modifier = Modifier.height(16.dp))
        Button(onClick = onIncrement) {
            Text("Increment")
        }
    }
}

// 父组件：持有状态（Stateful），管理状态并传递给子组件
@Composable
fun ParentCounter() {
    val count by remember { mutableStateOf(0) }
    CounterDisplay(count = count, onIncrement = { count++ })
}
```
- 子组件`CounterDisplay`是**无状态Composable**（纯UI组件），仅负责展示和转发用户操作，状态由父组件管理，符合“单一数据源”原则。
- 状态提升的规则：
  - 状态应提升到所有需要访问它的Composable的最小共同父组件中。
  - 状态的修改逻辑（事件）应与状态本身一起提升。

#### 2. 使用`ViewModel`管理共享状态
当状态需要跨Composable、跨生命周期（如Activity/Fragment）共享，或需要处理业务逻辑（如数据请求、数据转换）时，推荐使用**ViewModel**（Jetpack架构组件），ViewModel的生命周期独立于UI组件，可避免配置变更导致的状态丢失，且便于单元测试。

**步骤：**
1. 添加ViewModel依赖（在`build.gradle`中）：
```gradle
dependencies {
    implementation "androidx.lifecycle:lifecycle-viewmodel-compose:2.6.2" // Compose整合ViewModel的依赖
    implementation "androidx.lifecycle:lifecycle-viewmodel-ktx:2.6.2" // 基础ViewModel依赖
}
```
2. 创建ViewModel类，持有可观察状态（通常使用`MutableState`或`LiveData`/`Flow`）：
```kotlin
import androidx.lifecycle.ViewModel
import androidx.lifecycle.viewModelScope
import kotlinx.coroutines.delay
import kotlinx.coroutines.flow.MutableStateFlow
import kotlinx.coroutines.flow.StateFlow
import kotlinx.coroutines.launch

class CounterViewModel : ViewModel() {
    // 使用StateFlow（冷流）管理可观察状态，支持协程
    private val _count = MutableStateFlow(0)
    val count: StateFlow<Int> = _count // 对外暴露不可变的StateFlow
    
    fun increment() {
        _count.value++
    }
    
    // 模拟异步操作（如网络请求）
    fun incrementDelayed() {
        viewModelScope.launch { // ViewModel的协程作用域，自动跟随ViewModel生命周期取消
            delay(1000)
            _count.value++
        }
    }
}
```
3. 在Compose中获取ViewModel并观察状态：
```kotlin
@Composable
fun ViewModelCounter() {
    // 获取ViewModel实例（Compose的viewModel()函数自动管理ViewModel的生命周期）
    val counterViewModel: CounterViewModel = viewModel()
    // 收集StateFlow的状态（collectAsStateWithLifecycle：在Lifecycle处于STARTED状态时收集，优化性能）
    val count by counterViewModel.count.collectAsStateWithLifecycle(initialValue = 0)
    
    Column(
        modifier = Modifier.padding(16.dp),
        horizontalAlignment = Alignment.CenterHorizontally
    ) {
        Text(text = "Count: $count", fontSize = 24.sp)
        Spacer(modifier = Modifier.height(16.dp))
        Button(onClick = { counterViewModel.increment() }) {
            Text("Increment")
        }
        Spacer(modifier = Modifier.height(8.dp))
        Button(onClick = { counterViewModel.incrementDelayed() }) {
            Text("Increment Delayed (1s)")
        }
    }
}
```
- **`collectAsStateWithLifecycle`**：Compose的扩展函数，用于将`Flow`/`StateFlow`转换为Compose的`State`，并根据Lifecycle的状态（如APP退到后台时）暂停收集，减少资源消耗。
- ViewModel的优势：
  - 状态与UI解耦，业务逻辑集中在ViewModel中，便于测试。
  - 支持协程（`viewModelScope`），处理异步操作更简洁。
  - 配置变更（如屏幕旋转）后，ViewModel实例会被复用，状态不会丢失。

#### 3. 其他共享状态方案
- **`StateFlow`/`SharedFlow`**：基于协程的数据流，适合处理异步状态和多订阅者场景，是Compose中推荐的数据流方案（替代`LiveData`）。
- **`rememberCoroutineScope`**：用于在Composable中创建协程作用域，处理UI相关的异步操作（如动画、短暂的网络请求），但作用域生命周期与Composable的重组相关，不适合长期运行的任务（长期任务推荐ViewModel的`viewModelScope`）。
- **第三方状态管理库**：如**Jetpack Compose Navigation**（配合ViewModel管理页面间状态）、**Redux**（如`redux-kotlin`）、**MobX**（如`mobx-compose`）、**Compose Destinations**等，适用于复杂应用的全局状态管理，但简单应用通常无需引入第三方库，原生方案已足够。

### 三、状态管理的最佳实践
1. **最小化状态**：仅保留必要的状态，避免冗余状态（如可通过计算得到的派生数据不应作为独立状态）。
   ```kotlin
   // 反例：冗余状态（fullName可由firstName和lastName计算得到）
   val firstName by remember { mutableStateOf("") }
   val lastName by remember { mutableStateOf("") }
   val fullName by remember { mutableStateOf("") } // 冗余！
   
   // 正例：派生数据通过计算得到
   val firstName by remember { mutableStateOf("") }
   val lastName by remember { mutableStateOf("") }
   val fullName = "$firstName $lastName".trim() // 无状态，仅计算
   ```
2. **不可变状态优先**：使用不可变数据类型（如`val`、`List`→`ImmutableList`、`Map`→`ImmutableMap`），状态更新时创建新对象，避免可变对象导致的意外副作用（Compose的重组依赖状态的引用变化，不可变对象更易追踪变化）。
3. **单一数据源**：任何状态应只有一个来源，避免多个地方修改同一状态导致的数据不一致。
4. **分离UI和业务逻辑**：UI层（Composable）仅负责展示和用户交互，业务逻辑（数据请求、状态转换、业务规则）应放在ViewModel或Repository中，便于复用和测试。
5. **使用合适的状态容器**：
   - 本地短期状态：`remember` + `mutableStateOf`。
   - 本地持久化状态：`rememberSaveable`。
   - 跨组件/跨生命周期状态：`ViewModel` + `StateFlow`/`MutableState`。
   - 全局应用状态：`ViewModel`（Application级） + `StateFlow`/`SharedFlow`，或第三方状态管理库。

### 四、Compose状态的底层原理
Compose的状态管理基于**组合期（Composition）**和**重组（Recomposition）**：
- **组合期**：Composable函数首次执行，构建UI树（Composition），此时`remember`会缓存状态对象。
- **重组**：当`MutableState`/`StateFlow`等可观察状态变化时，Compose会标记依赖该状态的Composable节点，并重新执行这些节点的函数，更新UI树（仅更新变化的部分，实现高效渲染）。
- **状态订阅**：Compose通过**Composer**对象追踪状态的订阅关系，当状态变化时，仅通知订阅了该状态的Composable进行重组，避免全量刷新。

总结：Compose的状态管理以“简单优先”为原则，本地状态用`remember`/`rememberSaveable`，共享状态用状态提升+ViewModel，复杂场景可结合数据流（StateFlow）或第三方库，始终遵循单向数据流和单一数据源原则，可有效降低状态管理的复杂度。