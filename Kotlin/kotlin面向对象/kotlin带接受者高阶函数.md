
在 Kotlin 中，**带接收者的高阶函数**（Function with Receiver）是一种特殊的函数类型，其定义形式为 `T.() -> R`。它允许函数在调用时**以某个对象（接收者）为上下文**，直接访问该对象的成员（属性、方法），类似扩展函数的调用体验。这种设计在构建 DSL（领域特定语言）或对象配置场景中非常常见。


### 一、核心概念：接收者（Receiver）
带接收者的函数类型 `T.() -> R` 表示：  
- 该函数需要绑定一个类型为 `T` 的接收者对象（通过 `.` 符号关联）。  
- 函数内部可以直接访问接收者 `T` 的成员（无需显式通过变量名引用）。  


### 二、与普通函数类型的对比
对比普通函数类型 `(T) -> R` 和带接收者的函数类型 `T.() -> R`：

| 类型                | 调用方式                  | 函数体内访问接收者成员       | 典型场景                 |
|---------------------|---------------------------|------------------------------|--------------------------|
| `(T) -> R`          | `function(receiver)`      | 通过参数名（如 `t.property`）| 通用数据转换             |
| `T.() -> R`         | `receiver.function()`     | 直接访问（如 `property`）     | DSL 构建、对象配置       |


### 三、典型用法示例
#### 1. 对象配置（构建器模式）
带接收者的高阶函数能让对象的属性配置代码更接近自然语言，类似“给对象下达指令”。

**示例：用带接收者的函数配置用户对象**  
```kotlin
// 定义用户类
data class User(
    var name: String = "",
    var age: Int = 0,
    var isPremium: Boolean = false
)

// 定义高阶函数：接收者是 User，无返回值（Unit）
fun configureUser(block: User.() -> Unit): User {
    val user = User()
    user.block()  // 调用带接收者的函数（以 user 为上下文）
    return user
}

// 使用示例
fun main() {
    val user = configureUser {
        // 直接访问 User 的属性（无需 user.name，而是直接 name）
        name = "Alice"
        age = 28
        isPremium = true
    }
    println(user)  // 输出: User(name=Alice, age=28, isPremium=true)
}
```


#### 2. DSL（领域特定语言）
Kotlin 的 DSL 能力很大程度上依赖带接收者的高阶函数，例如 `buildString`、`apply` 等标准库函数，或 Gradle 脚本的配置语法。

**示例：自定义 HTML 构建 DSL**  
```kotlin
// 定义 HTML 标签类
class Html {
    private val children = mutableListOf<String>()
    
    fun p(text: String) {
        children.add("<p>$text</p>")
    }
    
    fun toHtml(): String {
        return "<html>${children.joinToString()}</html>"
    }
}

// 定义高阶函数：接收者是 Html，返回 Html
fun html(block: Html.() -> Unit): Html {
    val html = Html()
    html.block()  // 以 Html 对象为接收者执行 block
    return html
}

// 使用 DSL 构建 HTML
fun main() {
    val page = html {
        p("Hello, Kotlin!")
        p("DSL 真灵活~")
    }
    println(page.toHtml()) 
    // 输出: <html><p>Hello, Kotlin!</p><p>DSL 真灵活~</p></html>
}
```


#### 3. 标准库中的接收者函数
Kotlin 标准库的作用域函数（如 `apply`、`run`）大量使用了带接收者的高阶函数：  
```kotlin
// apply 函数的定义（简化版）
fun <T> T.apply(block: T.() -> Unit): T {
    block()  // 以当前对象（T的实例）为接收者执行 block
    return this
}

// 使用示例：用 apply 配置对象
val list = mutableListOf<Int>().apply {
    add(1)
    add(2)
    add(3)
}
println(list)  // 输出: [1, 2, 3]
```


### 四、关键细节
- **接收者的作用域**：函数体内可以直接访问接收者的成员（包括 `private` 成员，只要函数定义在接收者类的作用域内）。  
- **显式访问接收者**：若需要区分局部变量和接收者成员，可用 `this` 显式引用接收者（如 `this.name`）。  
- **可空接收者**：支持接收者为可空类型（如 `T?.() -> R`），此时函数体内需处理 `this` 可能为 `null` 的情况。  


带接收者的高阶函数通过**将逻辑与对象上下文绑定**，让代码更符合人类语言习惯，尤其在需要“声明式配置”或“领域特定语法”时，能显著提升代码的可读性和简洁性。