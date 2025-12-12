### Kotlin 重载 `getValue`/`setValue`：委托属性的核心
`getValue` 和 `setValue` 是 Kotlin 中实现**委托属性（Delegated Properties）** 的核心函数——通过重载这两个函数，我们可以自定义属性的「读取」和「赋值」逻辑，让属性的访问/修改行为完全由我们控制（如懒加载、缓存、监听、校验等）。

> 本质：`val` 属性仅需重载 `getValue`，`var` 属性需同时重载 `getValue` 和 `setValue`；这两个函数需用 `operator` 标记，且遵循固定的参数/返回值规则。

---

## 一、核心规则
### 1. 函数签名（必遵守）
| 函数       | 适用属性 | 参数要求                                                                 | 返回值               |
|------------|----------|--------------------------------------------------------------------------|----------------------|
| `getValue` | `val`/`var` | ① 接收者（属性所属对象）：`thisRef: T` <br> ② 属性元数据：`property: KProperty<*>` | 属性的读取结果（任意类型） |
| `setValue` | `var`    | ① 接收者：`thisRef: T` <br> ② 属性元数据：`property: KProperty<*>` <br> ③ 赋值的值：`value: V` | 无返回值（`Unit`）|

- `thisRef`：类型需匹配属性所属的类（如属性在 `User` 类中，`thisRef` 应为 `User` 或其父类）；
- `KProperty<*>`：表示属性的元数据（如属性名、类型、注解等），可通过它获取属性信息；
- 泛型说明：通常用「泛型类」实现，让 `getValue`/`setValue` 适配任意类型的属性。

### 2. 实现方式
重载 `getValue`/`setValue` 有两种常见方式：
- **方式1**：定义为「泛型类的成员函数」（最常用，可复用）；
- **方式2**：定义为「扩展函数」（适配无法修改的类）。

---

## 二、基础示例：自定义委托类
### 场景：实现一个“带日志的委托属性”（读取/赋值时打印日志）
```kotlin
import kotlin.reflect.KProperty

// 步骤1：定义委托类，重载 getValue/setValue
class LoggingDelegate<T>(private var value: T) {
    // 重载 getValue：属性读取时触发
    operator fun getValue(
        thisRef: Any?, // 属性所属对象（这里适配任意类，用 Any?）
        property: KProperty<*> // 属性元数据
    ): T {
        // 自定义读取逻辑：打印日志
        println("读取属性 ${property.name}，当前值：$value")
        return value // 返回存储的实际值
    }

    // 重载 setValue：属性赋值时触发（仅 var 属性需要）
    operator fun setValue(
        thisRef: Any?,
        property: KProperty<*>,
        value: T // 要赋的新值
    ) {
        // 自定义赋值逻辑：打印日志 + 校验（示例：非空校验）
        println("为属性 ${property.name} 赋值，旧值：${this.value} → 新值：$value")
        if (value == null) {
            throw IllegalArgumentException("${property.name} 不能为 null")
        }
        this.value = value // 保存新值
    }
}

// 步骤2：使用委托属性
class User {
    // var 属性：委托给 LoggingDelegate
    var name: String by LoggingDelegate("默认名称")
    // val 属性：仅需 getValue（简化版委托类）
    val id: Int by object {
        operator fun getValue(thisRef: Any?, property: KProperty<*>): Int {
            println("读取只读属性 ${property.name}")
            return 1001 // 固定返回值
        }
    }
}

// 测试
fun main() {
    val user = User()
    // 读取属性（触发 getValue）
    println(user.name) // 输出：读取属性 name，当前值：默认名称 → 默认名称
    println(user.id)   // 输出：读取只读属性 id → 1001

    // 赋值属性（触发 setValue）
    user.name = "张三"  // 输出：为属性 name 赋值，旧值：默认名称 → 新值：张三
    println(user.name) // 输出：读取属性 name，当前值：张三 → 张三

    // 测试非空校验
    // user.name = null // 抛出异常：name 不能为 null
}
```

---

## 三、进阶示例：常用委托场景
### 1. 懒加载（Lazy）
Kotlin 内置的 `lazy` 本质就是通过重载 `getValue` 实现的，我们可以自定义简化版：
```kotlin
// 自定义懒加载委托
class LazyDelegate<T>(private val initializer: () -> T) {
    private var value: T? = null
    private var isInitialized = false

    operator fun getValue(thisRef: Any?, property: KProperty<*>): T {
        if (!isInitialized) {
            value = initializer() // 首次读取时初始化
            isInitialized = true
            println("懒加载属性 ${property.name} 初始化完成")
        }
        return value!!
    }
}

// 简化调用（扩展函数）
fun <T> lazyCustom(initializer: () -> T) = LazyDelegate(initializer)

// 使用
class AppConfig {
    // 懒加载：首次访问时才执行初始化逻辑
    val apiKey: String by lazyCustom {
        println("执行复杂的初始化逻辑...")
        "123456-ABCDEF"
    }
}

fun main() {
    val config = AppConfig()
    println("首次访问 apiKey：${config.apiKey}") // 输出初始化日志 + 123456-ABCDEF
    println("再次访问 apiKey：${config.apiKey}") // 直接返回缓存值，无初始化日志
}
```

### 2. 观察属性变化（Observable）
类似 Android 的 `LiveData`，赋值时触发回调：
```kotlin
// 可观察的委托类
class ObservableDelegate<T>(
    private var value: T,
    private val onChange: (oldValue: T, newValue: T) -> Unit
) {
    operator fun getValue(thisRef: Any?, property: KProperty<*>): T = value

    operator fun setValue(thisRef: Any?, property: KProperty<*>, newValue: T) {
        val oldValue = this.value
        if (oldValue != newValue) {
            this.value = newValue
            onChange(oldValue, newValue) // 触发变化回调
        }
    }
}

// 使用
class ViewModel {
    var count: Int by ObservableDelegate(0) { old, new ->
        println("计数从 $old 变为 $new") // 赋值时触发回调
    }
}

fun main() {
    val vm = ViewModel()
    vm.count = 1 // 输出：计数从 0 变为 1
    vm.count = 2 // 输出：计数从 1 变为 2
    vm.count = 2 // 无输出（值未变化）
}
```

### 3. 映射委托（MapDelegate）
将属性值存储在 `Map` 中（适合动态配置、JSON 解析等场景）：
```kotlin
// 读取 Map 的委托（val 属性）
class MapDelegate<T>(private val map: Map<String, Any?>) {
    operator fun getValue(thisRef: Any?, property: KProperty<*>): T {
        // 根据属性名从 Map 中取值
        return map[property.name] as T
    }
}

// 使用
data class Person(
    val name: String,
    val age: Int
) {
    // 从 Map 初始化属性（模拟 JSON 解析）
    constructor(map: Map<String, Any?>) : this(
        name = map["name"] as String,
        age = map["age"] as Int
    )

    // 或直接委托给 Map（更简洁）
    val address: String by MapDelegate(mapOf("address" to "北京市"))
}

fun main() {
    val personMap = mapOf(
        "name" to "李四",
        "age" to 25
    )
    val person = Person(personMap)
    println(person.name)    // 输出：李四
    println(person.address) // 输出：北京市
}
```

---

## 四、关键注意事项
### 1. `thisRef` 类型匹配
- 若委托仅用于特定类（如 `User`），`thisRef` 应指定为该类（如 `thisRef: User`），避免泛用；
- 若委托需适配任意类，`thisRef` 用 `Any?`（可空）。

### 2. `val` vs `var`
- `val`（只读属性）：仅需重载 `getValue`，`setValue` 无需实现（也不能实现）；
- `var`（可变属性）：必须同时重载 `getValue` 和 `setValue`。

### 3. 与 Kotlin 内置委托的关系
Kotlin 标准库中的 `lazy`、`observable`、`vetoable`、`map` 等委托，本质都是通过重载 `getValue`/`setValue` 实现的：
```kotlin
// 标准库内置委托示例
class Demo {
    // 懒加载（val）
    val lazyValue: String by lazy { "懒加载值" }
    // 可观察（var）
    var observableValue: Int by Delegates.observable(0) { _, old, new ->
        println("值变化：$old → $new")
    }
    // 可否决（赋值前校验）
    var vetoableValue: Int by Delegates.vetoable(0) { _, old, new ->
        new > old // 仅允许赋值更大的值
    }
}
```

### 4. 扩展函数形式的重载
若需为现有类扩展委托能力，可将 `getValue`/`setValue` 定义为扩展函数：
```kotlin
// 为 String 扩展 getValue（模拟“默认值”委托）
operator fun String.getValue(thisRef: Any?, property: KProperty<*>): String {
    println("读取扩展委托属性 ${property.name}，值：$this")
    return this
}

// 使用
class Test {
    val defaultName: String by "默认名称" // 委托给 String 实例
}

fun main() {
    println(Test().defaultName) // 输出：读取扩展委托属性 defaultName，值：默认名称 → 默认名称
}
```

---

## 五、典型应用场景
1. **Android 开发**：
   - ViewModel 中属性变化监听（替代 LiveData/StateFlow 简化版）；
   - Compose 中 `remember` 委托的自定义扩展；
   - SharedPreferences 委托（读写 SP 无需重复写 `putString/getString`）。
2. **通用开发**：
   - 懒加载初始化（如数据库连接、网络客户端）；
   - 配置中心动态取值（从配置文件/远程接口读取属性）；
   - 数据校验（赋值时自动校验合法性，如手机号、邮箱格式）。

---

## 总结
重载 `getValue`/`setValue` 是 Kotlin 委托属性的核心，其本质是：
- `getValue` 接管属性的「读取逻辑」；
- `setValue` 接管属性的「赋值逻辑」；
通过这两个函数，我们可以将属性的底层实现与业务代码解耦，实现可复用、可扩展的属性行为。
核心要点：
- 必须用 `operator` 标记；
- 函数签名（参数/返回值）需严格匹配规则；
- `val` 仅需 `getValue`，`var` 需两者都实现。