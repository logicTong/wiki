在 Kotlin 中，`require()` 是一个 **预定义的顶层函数**，用于在代码中进行 **参数校验** 或 **前置条件检查**。它的核心作用是：**确保某个条件必须为 `true`，否则抛出 `IllegalArgumentException` 异常**，常用于验证函数参数的合法性。


### 一、基本语法
```kotlin
require(condition: Boolean)
require(condition: Boolean, lazyMessage: () -> Any)
```

- **参数**：
  - `condition`：必须满足的条件（布尔值）。如果为 `false`，则触发异常。
  - `lazyMessage`（可选）：异常信息的延迟计算 lambda（仅在条件为 `false` 时执行），用于描述错误原因。
- **返回值**：无（`Unit`）。
- **异常**：当 `condition` 为 `false` 时，抛出 `IllegalArgumentException`（参数不合法异常）。


### 二、使用场景：参数合法性校验
`require()` 最常见的用途是在函数开头验证输入参数是否符合预期，确保函数在“合法的前提”下执行。

#### 示例 1：基础用法
```kotlin
fun divide(a: Int, b: Int): Int {
    // 检查除数不能为 0，否则抛出异常
    require(b != 0) { "除数 b 不能为 0" }
    return a / b
}

fun main() {
    divide(10, 2) // 正常执行，返回 5
    divide(10, 0) // 触发异常：IllegalArgumentException: 除数 b 不能为 0
}
```

#### 示例 2：验证参数范围
```kotlin
fun createUser(age: Int) {
    // 检查年龄必须为正数
    require(age > 0) { "年龄必须大于 0，实际值：$age" }
    println("创建年龄为 $age 的用户")
}

createUser(20) // 正常执行
createUser(-5) // 异常：IllegalArgumentException: 年龄必须大于 0，实际值：-5
```


### 三、与其他校验函数的区别
Kotlin 中还有几个类似的校验函数，注意区分使用场景：

| 函数         | 触发条件                  | 抛出异常类型               | 典型用途                     |
|--------------|---------------------------|----------------------------|------------------------------|
| `require()`  | 条件为 `false`            | `IllegalArgumentException` | 验证函数参数合法性（外部输入） |
| `check()`    | 条件为 `false`            | `IllegalStateException`    | 验证函数内部状态合法性（内部逻辑） |
| `assert()`   | 条件为 `false`（仅调试模式） | 无（或 `AssertionError`）   | 调试阶段的断言检查（可选开启） |

#### 对比示例：
```kotlin
fun processData(data: List<String>?, status: Int) {
    // 1. 验证外部输入参数（data 不能为 null）
    require(data != null) { "输入数据不能为 null" }

    // 2. 验证内部状态（status 必须是预期值）
    check(status == 1) { "当前状态不允许处理数据，状态码：$status" }

    // 3. 调试断言（仅开发阶段生效，如数据不为空）
    assert(data.isNotEmpty()) { "数据列表为空（调试信息）" }
}
```


### 四、为什么用 `require()`？
1. **代码可读性**：明确表达“此条件是函数执行的前提”，比手动 `if + throw` 更简洁。  
   例如，替代以下冗余写法：
   ```kotlin
   // 不推荐：手动判断+抛异常
   if (b == 0) {
       throw IllegalArgumentException("除数不能为 0")
   }
   ```

2. **异常类型规范**：统一抛出 `IllegalArgumentException`，符合 Kotlin 异常处理的最佳实践（参数问题用此异常）。

3. **延迟计算消息**：`lazyMessage` 是 lambda 表达式，仅在条件为 `false` 时才执行，避免不必要的字符串拼接开销（尤其在高性能场景）。


### 五、注意事项
- `require()` 抛出的异常属于 **未受检异常（unchecked exception）**，无需在函数签名中用 `throws` 声明，但调用者需自行处理可能的异常。
- 不要滥用 `require()`：仅用于“必须满足否则函数无法执行”的场景，非关键条件可使用普通 `if` 判断并返回默认值。


总结：`require()` 是 Kotlin 中用于参数校验的“语法糖”，通过简洁的方式确保函数输入合法，提升代码的健壮性和可读性。