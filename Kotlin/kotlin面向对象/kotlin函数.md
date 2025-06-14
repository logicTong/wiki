
在 Kotlin 中，函数（Function）是组织代码逻辑的基本单元，支持多种灵活特性（如默认参数、高阶函数、扩展函数等），并深度融合了函数式编程的思想。以下是 Kotlin 函数的核心特性和使用方式：


### **1. 基本函数定义**
使用 `fun` 关键字声明函数，语法结构为：
```kotlin
fun 函数名(参数列表): 返回类型 {
    // 函数体
    return 返回值  // 若返回类型为 Unit（无返回值），可省略 return
}
```

- **示例**：
  ```kotlin
  // 计算两数之和（显式返回类型）
  fun add(a: Int, b: Int): Int {
      return a + b
  }

  // 无返回值（默认返回 Unit，可省略声明）
  fun printMessage(msg: String) {
      println(msg)  // 等价于 `fun printMessage(msg: String): Unit { ... }`
  }
  ```


### **2. 参数与返回值**
#### **(1) 默认参数**  
函数参数可设置默认值，调用时若未传递该参数则使用默认值（避免重载）。  
```kotlin
// 带默认参数的函数
fun greet(name: String = "Guest", times: Int = 1) {
    repeat(times) { println("Hello, $name!") }
}

// 调用示例
greet()                  // 输出: Hello, Guest!（调用 1 次）
greet("Alice")           // 输出: Hello, Alice!（调用 1 次）
greet("Bob", 3)          // 输出: Hello, Bob!（调用 3 次）
```

#### **(2) 命名参数**  
调用函数时可显式指定参数名，避免因参数顺序混淆导致的错误（尤其适用于多参数场景）。  
```kotlin
// 函数定义（参数顺序：width, height）
fun calculateArea(width: Int, height: Int): Int = width * height

// 常规调用（依赖参数顺序）
calculateArea(5, 10)  // 50

// 命名参数调用（可调整顺序）
calculateArea(height=10, width=5)  // 50（等价于上一行）
```

#### **(3) 可变参数（vararg）**  
使用 `vararg` 关键字声明可变数量的参数（底层为数组），通常作为最后一个参数。  
```kotlin
fun <T> joinToString(vararg elements: T, separator: String = ", "): String {
    return elements.joinToString(separator)
}

// 调用示例
joinToString(1, 2, 3, separator = "-")  // 输出: "1-2-3"
joinToString("A", "B", "C")             // 输出: "A, B, C"
```

#### **(4) 单表达式函数**  
若函数体是**单个表达式**，可省略大括号和 `return`，直接写表达式（返回类型可自动推断）。  
```kotlin
// 常规写法
fun add(a: Int, b: Int): Int {
    return a + b
}

// 单表达式简化写法（返回类型可推断，可省略）
fun add(a: Int, b: Int) = a + b  // 等价于上一行
```


### **3. 高阶函数与 Lambda**
高阶函数是指**参数为函数**或**返回值为函数**的函数，是函数式编程的核心工具。Kotlin 中常用 Lambda 表达式（匿名函数）作为高阶函数的参数。  

#### **(1) 高阶函数示例**  
```kotlin
// 定义高阶函数（参数为 (Int) -> Boolean 类型的函数）
fun filterEven(numbers: List<Int>, predicate: (Int) -> Boolean): List<Int> {
    return numbers.filter(predicate)
}

// 调用时传入 Lambda 表达式（筛选偶数）
val result = filterEven(listOf(1, 2, 3, 4)) { it % 2 == 0 }  // 输出 [2, 4]
```

#### **(2) Lambda 表达式**  
Lambda 是匿名函数的简化形式，语法为 `{ 参数列表 -> 函数体 }`，常用于简化高阶函数调用。  
- 若 Lambda 是最后一个参数，可移到括号外（称为“尾随 Lambda”）；  
- 若 Lambda 只有一个参数，可省略参数声明，用 `it` 指代（隐式参数名）。  

```kotlin
// 常规 Lambda（带参数声明）
numbers.map { num -> num * 2 }

// 简化 Lambda（使用隐式参数 it）
numbers.map { it * 2 }
```


### **4. 扩展函数（Extension Function）**  
扩展函数允许为**现有类**（包括第三方库的类）添加新方法，而无需继承或修改原类代码。  
语法：`fun 类名.扩展函数名(参数列表): 返回类型 { ... }`  

**示例：为 String 扩展“统计单词数”功能**  
```kotlin
// 扩展函数定义（为 String 类添加 countWords() 方法）
fun String.countWords(): Int {
    return if (isBlank()) 0 else split("\\s+".toRegex()).size
}

// 使用扩展函数
val text = "Hello Kotlin World"
println(text.countWords())  // 输出: 3（单词数）
```


### **5. 内联函数（Inline Function）**  
默认情况下，Lambda 会被编译为匿名类（增加内存开销）。使用 `inline` 关键字修饰函数，可将 Lambda 代码**直接插入调用处**（避免匿名类创建），提升性能（尤其适用于高频调用的高阶函数）。  

```kotlin
// 内联函数定义
inline fun <T> runWithLogging(block: () -> T): T {
    println("开始执行...")
    val result = block()  // 插入 Lambda 代码
    println("执行完成，结果: $result")
    return result
}

// 调用内联函数（Lambda 会被展开到调用处）
runWithLogging { 
    1 + 2 
}  // 输出: 开始执行... 执行完成，结果: 3
```

> **注意**：内联函数的 Lambda 参数可通过 `noinline` 禁止内联（用于需要存储 Lambda 引用的场景），或 `crossinline` 限制 Lambda 不能使用 `return` 跳转到外层。


### **6. 特殊函数**
#### **(1) 尾递归函数（Tailrec Function）**  
若函数的**最后一步操作是调用自身**（尾递归），可用 `tailrec` 修饰，编译器会优化为循环（避免栈溢出）。  

```kotlin
tailrec fun findIndex(list: List<Int>, target: Int, index: Int = 0): Int {
    if (index >= list.size) return -1
    if (list[index] == target) return index
    return findIndex(list, target, index + 1)  // 尾递归调用
}

// 调用（等价于循环查找）
val list = listOf(5, 3, 8, 2)
println(findIndex(list, 8))  // 输出: 2
```

#### **(2) 运算符重载函数**  
通过特定函数名重载 Kotlin 内置运算符（如 `+`、`-`、`==` 等），使自定义类支持运算符操作。  

```kotlin
class Point(val x: Int, val y: Int) {
    // 重载 + 运算符（函数名必须为 plus）
    operator fun plus(other: Point): Point {
        return Point(x + other.x, y + other.y)
    }
}

// 使用重载后的运算符
val p1 = Point(1, 2)
val p2 = Point(3, 4)
val p3 = p1 + p2  // 等价于 p1.plus(p2)，结果为 Point(4, 6)
```


### **总结**
Kotlin 函数的设计融合了面向对象和函数式编程的优势，通过默认参数、高阶函数、扩展函数等特性，极大提升了代码的灵活性和可维护性。熟练掌握这些特性，能帮助开发者写出更简洁、高效的 Kotlin 代码。