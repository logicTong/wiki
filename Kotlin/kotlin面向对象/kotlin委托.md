
在 Kotlin 中，委托（Delegation）是一种通过组合（Composition）替代继承（Inheritance）的代码复用机制，核心思想是**将行为的实现委托给另一个对象**。Kotlin 支持两种委托模式：**类委托（Class Delegation）**和**属性委托（Property Delegation）**。


### **一、类委托（Class Delegation）**
类委托允许一个类（称为“委托类”）将其接口的实现**委托给另一个对象**（称为“被委托对象”），避免通过继承扩展功能时的冗余代码。


#### **1. 核心语法**
通过 `by` 关键字声明委托关系，格式为：  
`class 委托类 : 接口 by 被委托对象`  


#### **2. 示例：日志装饰器**
假设需要为一个数据存储接口添加日志功能，但不想直接修改原实现类。此时可通过类委托快速实现：

```kotlin
// 定义基础接口
interface DataStorage {
    fun save(key: String, value: String)
    fun get(key: String): String?
}

// 原始实现类（被委托对象）
class DefaultDataStorage : DataStorage {
    private val map = mutableMapOf<String, String>()
    override fun save(key: String, value: String) {
        map[key] = value
    }
    override fun get(key: String): String? = map[key]
}

// 委托类（添加日志功能）
class LoggedDataStorage(delegate: DataStorage) : DataStorage by delegate {
    // 重写 save 方法，添加日志（其他方法自动委托给 delegate）
    override fun save(key: String, value: String) {
        println("日志：正在保存 $key -> $value")
        delegate.save(key, value)  // 调用被委托对象的原始方法
    }
}

// 使用示例
fun main() {
    val original = DefaultDataStorage()
    val loggedStorage = LoggedDataStorage(original)  // 委托给原始实现
    
    loggedStorage.save("name", "Kotlin")  // 输出：日志：正在保存 name -> Kotlin
    println(loggedStorage.get("name"))    // 输出：Kotlin（自动委托 get 方法）
}
```


#### **3. 关键特性**
- **接口约束**：委托类必须实现一个接口，被委托对象需实现相同接口。
- **方法重写**：委托类可选择性重写接口中的方法（未重写的方法自动委托给被委托对象）。
- **解耦与灵活**：通过组合替代继承，避免类层级膨胀（如无需为每个功能扩展创建子类）。


### **二、属性委托（Property Delegation）**
属性委托允许将属性的 `get()` 和 `set()` 操作**委托给另一个对象**，用于抽象属性访问的通用逻辑（如延迟加载、数据验证、监听变化等）。


#### **1. 核心规则**
- 属性声明为 `val`（只读）时，委托对象需实现 `ReadOnlyProperty` 接口（含 `getValue()` 方法）。
- 属性声明为 `var`（可变）时，委托对象需实现 `MutableProperty` 接口（含 `getValue()` 和 `setValue()` 方法）。
- 通过 `by` 关键字绑定委托对象，格式为：  
  `val/var 属性名: 类型 by 委托对象`  


#### **2. 自定义属性委托**
以“延迟加载属性”为例（类似 `lazy` 但自定义逻辑）：

```kotlin
// 委托类（实现 ReadOnlyProperty 接口）
class LazyDelegate<T>(private val initializer: () -> T) {
    private var value: T? = null
    
    // 实现 getValue 方法（val 属性必须）
    operator fun getValue(thisRef: Any?, property: kotlin.reflect.KProperty<*>): T {
        if (value == null) {
            value = initializer()  // 首次访问时初始化
        }
        return value!!
    }
}

// 使用自定义委托
class Example {
    // 将属性委托给 LazyDelegate 实例
    val lazyValue: String by LazyDelegate {
        println("初始化延迟属性")
        "Hello, Kotlin"
    }
}

// 测试
fun main() {
    val example = Example()
    println(example.lazyValue)  // 输出：初始化延迟属性 → Hello, Kotlin
    println(example.lazyValue)  // 直接返回缓存值（不重复初始化）
}
```


#### **3. 标准库内置委托**
Kotlin 标准库提供了常用的属性委托实现，简化开发：

| 委托类型          | 作用                                                                 | 示例                                                                 |
|-------------------|----------------------------------------------------------------------|----------------------------------------------------------------------|
| `lazy`            | 延迟初始化（线程安全，首次访问时计算值）                              | `val data: Data by lazy { loadData() }`                              |
| `observable`      | 监听属性变化（值修改时触发回调）                                      | `var name: String by Delegates.observable("默认值") { _, old, new -> ... }` |
| `vetoable`        | 监听属性变化，并可阻止修改（返回 `false` 则拒绝修改）                  | `var age: Int by Delegates.vetoable(0) { _, old, new -> new > 0 }`    |
| `map` 委托        | 从 `Map` 中读取/写入属性（适合动态属性场景）                           | `class User(map: Map<String, Any>) { val name: String by map }`       |


#### **4. 高级用法：委托属性的参数**
委托对象可通过 `KProperty` 参数获取属性的元数据（如属性名、声明类等），实现更灵活的逻辑：

```kotlin
class DebugDelegate {
    operator fun getValue(thisRef: Any?, property: KProperty<*>): String {
        return "调试信息：属性名=${property.name}, 所属类=${thisRef?.javaClass?.simpleName}"
    }
}

class Test {
    val debugInfo: String by DebugDelegate()
}

fun main() {
    println(Test().debugInfo)  // 输出：调试信息：属性名=debugInfo, 所属类=Test
}
```


### **三、委托的设计优势**
- **代码复用**：通过组合替代继承，避免类层级爆炸（如无需为每个功能扩展创建子类）。
- **逻辑抽象**：属性委托将通用逻辑（如延迟加载、监听变化）封装，减少样板代码。
- **灵活性**：委托对象可动态替换，运行时调整行为（如切换不同的日志实现）。


### **总结**
Kotlin 的委托机制通过 `by` 关键字将行为或属性访问逻辑委托给其他对象，是实现“组合优于继承”原则的核心工具。类委托适合接口行为的复用，属性委托则擅长抽象属性访问的通用逻辑。结合标准库的内置委托（如 `lazy`、`observable`），可大幅提升代码的简洁性和可维护性。


    
    
    