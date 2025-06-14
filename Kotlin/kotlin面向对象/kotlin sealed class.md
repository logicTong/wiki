
密封类（`sealed class`）是一种用于限制类继承的编程机制，主要作用是**严格控制某个类的子类范围**，确保只有预定义的子类可以继承它。这种特性在需要枚举有限类型场景（如状态机、事件分类）时非常有用，能提高代码的安全性和可维护性。


### 核心特点
- **封闭性**：密封类的子类必须在其声明的同一文件（Kotlin 1.1 前）或同一模块（Kotlin 1.1+）中定义，外部无法新增子类。
- **类型安全**：在 `when` 表达式中处理密封类的子类时，编译器会检查是否覆盖了所有可能的子类，无需 `else` 分支（除非显式需要）。


### 示例：Kotlin 中的密封类
以下是一个用密封类表示“支付状态”的示例，包含成功、失败、处理中三种状态：

```java
sealed class PaymentState {
    // 成功状态（携带支付金额）
    data class Success(val amount: Double) : PaymentState()
    // 失败状态（携带错误信息）
    data class Failure(val error: String) : PaymentState()
    // 处理中状态（无额外数据）
    object Processing : PaymentState()
}

// 使用示例：根据支付状态输出信息
fun handlePaymentState(state: PaymentState) {
    when (state) {
        is PaymentState.Success -> println("支付成功，金额：${state.amount} 元")
        is PaymentState.Failure -> println("支付失败，原因：${state.error}")
        PaymentState.Processing -> println("支付处理中...")
    }
}

// 测试调用
fun main() {
    val successState = PaymentState.Success(99.9)
    val failureState = PaymentState.Failure("余额不足")
    val processingState = PaymentState.Processing

    handlePaymentState(successState)  // 输出：支付成功，金额：99.9 元
    handlePaymentState(failureState)  // 输出：支付失败，原因：余额不足
    handlePaymentState(processingState)  // 输出：支付处理中...
}
```
    



### 关键说明
- **子类形式**：密封类的子类可以是 `data class`（携带数据）、`object`（单例）或普通类。
- **类型检查**：通过 `is` 关键字判断具体子类类型，`when` 表达式会强制覆盖所有可能的子类（否则编译器报错）。
- **应用场景**：常见于状态管理（如页面加载状态：`Loading`/`Success`/`Error`）、事件分类（如用户操作：`Click`/`Swipe`/`LongPress`）等需要有限类型的场景。

