Kotlin 的协变（Covariance）和逆变（Contravariance）是解决泛型类型继承关系的核心机制，用于控制泛型类或接口在类型参数变化时的子类型关系。通过 `out`（协变）和 `in`（逆变）关键字，Kotlin 能在编译期保证类型安全，同时让代码更灵活。


### **一、为什么需要协变与逆变？**
假设我们有一个泛型类 `Box<T>`，当 `T` 是 `Int`（子类型）时，`Box<Int>` 和 `Box<Number>`（父类型）之间是否存在子类型关系？  
在 Java/Kotlin 中，**原生泛型类型本身不具备继承性**（如 `Box<Int>` 不是 `Box<Number>` 的子类型），这会导致一些场景下的不便：  
例如，无法将 `Box<Int>` 赋值给 `Box<Number>` 类型的变量，即使 `Int` 是 `Number` 的子类型。  

协变与逆变的目标就是解决这类问题，通过明确泛型类型参数的“使用方向”（输出或输入），允许安全的子类型转换。


### **二、协变（Covariance）：`out`**
协变用于声明泛型类型参数**仅作为输出**（如返回值），允许泛型类的子类型关系与类型参数的子类型关系一致。


#### **1. 核心规则**
- 用 `out` 标记类型参数（如 `class Producer<out T>`）。
- 类型参数 `T` 只能出现在**输出位置**（函数返回值、只读属性），不能出现在输入位置（函数参数、可写属性）。
- 若 `A` 是 `B` 的子类型，则 `Producer<A>` 是 `Producer<B>` 的子类型（协变关系）。


#### **2. 示例：安全的“生产者”**
假设定义一个生产 `T` 类型对象的接口 `Producer`，它只需要返回 `T`（输出）：  
```kotlin
// 协变接口（T 仅用于输出）
interface Producer<out T> {
    fun produce(): T  // T 作为返回值（输出位置）
}

// 子类型：生产 Int
class IntProducer : Producer<Int> {
    override fun produce() = 42
}

// 父类型：生产 Number（Int 是 Number 的子类型）
val numberProducer: Producer<Number> = IntProducer()  // 合法！协变允许子类型替换父类型
```
这里，`IntProducer`（`Producer<Int>`）可以安全地赋值给 `Producer<Number>`，因为它只会“生产” `Int`（`Number` 的子类型），不会要求接收 `Number`（避免类型污染）。


#### **3. 协变的限制**
`out` 标记的类型参数不能出现在输入位置（如函数参数），否则会破坏类型安全。  
**错误示例**：  
```kotlin
interface BadProducer<out T> {
    fun consume(value: T)  ❌ 编译错误！T 作为输入参数（输入位置）
}
```
若允许 `T` 作为输入参数，可能出现以下问题：  
```kotlin
val intProducer: BadProducer<Int> = ... 
val numberProducer: BadProducer<Number> = intProducer  // 假设允许协变
numberProducer.consume(3.14)  // 向 Int 生产者传入 Double（类型污染）
```
因此，`out` 类型参数只能用于输出位置，确保不会接收“父类型”数据。


### **三、逆变（Contravariance）：`in`**
逆变与协变相反，用于声明泛型类型参数**仅作为输入**（如函数参数），允许泛型类的子类型关系与类型参数的子类型关系**相反**。


#### **1. 核心规则**
- 用 `in` 标记类型参数（如 `class Consumer<in T>`）。
- 类型参数 `T` 只能出现在**输入位置**（函数参数、可写属性），不能出现在输出位置（函数返回值、只读属性）。
- 若 `A` 是 `B` 的子类型，则 `Consumer<B>` 是 `Consumer<A>` 的子类型（逆变关系）。


#### **2. 示例：灵活的“消费者”**
假设定义一个消费 `T` 类型对象的接口 `Consumer`，它只需要接收 `T`（输入）：  
```kotlin
// 逆变接口（T 仅用于输入）
interface Consumer<in T> {
    fun consume(value: T)  // T 作为参数（输入位置）
}

// 子类型：消费 Number
class NumberConsumer : Consumer<Number> {
    override fun consume(value: Number) {
        println("Consuming number: $value")
    }
}

// 父类型：消费 Int（Number 是 Int 的父类型）
val intConsumer: Consumer<Int> = NumberConsumer()  // 合法！逆变允许父类型替换子类型
```
这里，`NumberConsumer`（`Consumer<Number>`）可以赋值给 `Consumer<Int>`，因为它能处理所有 `Int`（`Number` 的子类型），不会要求传入更具体的类型（避免类型不匹配）。


#### **3. 逆变的限制**
`in` 标记的类型参数不能出现在输出位置（如函数返回值），否则会破坏类型安全。  
**错误示例**：  
```kotlin
interface BadConsumer<in T> {
    fun produce(): T  ❌ 编译错误！T 作为返回值（输出位置）
}
```
若允许 `T` 作为返回值，可能出现以下问题：  
```kotlin
val numberConsumer: BadConsumer<Number> = ... 
val intConsumer: BadConsumer<Int> = numberConsumer  // 假设允许逆变
val value: Int = intConsumer.produce()  // 从 Number 消费者获取 Int（可能返回 Double，类型错误）
```
因此，`in` 类型参数只能用于输入位置，确保不会返回“子类型”数据。


### **四、协变与逆变的实际应用**
Kotlin 标准库中大量使用了协变和逆变，典型例子包括：


#### **1. `List<out T>`（协变）**
Kotlin 的 `List` 接口定义为 `List<out T>`，表示它是协变的。这意味着 `List<Int>` 可以安全地视为 `List<Number>`（因为 `List` 仅提供只读操作，不会修改元素）：  
```kotlin
val intList: List<Int> = listOf(1, 2, 3)
val numberList: List<Number> = intList  // 合法！协变允许子类型替换父类型
```


#### **2. `Comparator<in T>`（逆变）**
`Comparator` 接口定义为 `Comparator<in T>`，表示它是逆变的。这意味着一个能比较 `Number` 的比较器可以用于比较 `Int`（因为 `Int` 是 `Number` 的子类型）：  
```kotlin
// 比较 Number 的比较器（父类型）
val numberComparator: Comparator<Number> = Comparator { a, b -> 
    a.toDouble().compareTo(b.toDouble()) 
}

// 用于比较 Int（子类型）
val intComparator: Comparator<Int> = numberComparator  // 合法！逆变允许父类型替换子类型
println(intComparator.compare(3, 5))  // 输出 -1（正确比较）
```


### **五、星投影（Star Projection）：`*`**
当泛型类型参数的具体类型未知时，可以用 `*` 表示“任意类型”，类似 Java 的 `<?>`。它是协变和逆变的简化形式：  
- `List<*>` 等价于 `List<out Any?>`（协变，只能读取 `Any?` 类型）。  
- `Consumer<*>` 等价于 `Consumer<in Nothing>`（逆变，只能接收 `Nothing` 类型，但 `Nothing` 是所有类型的子类型，实际无意义，因此星投影主要用于协变场景）。  

**示例**：  
```kotlin
fun printList(list: List<*>) {  // 等价于 List<out Any?>
    list.forEach { println(it) }  // 只能读取为 Any? 类型
}

printList(listOf(1, 2, 3))       // 合法
printList(listOf("A", "B", "C")) // 合法
```


### **六、对比 Java 的通配符**
Kotlin 的 `out`/`in` 与 Java 的通配符 `<? extends T>`/`<? super T>` 等价，但语法更简洁：  

| Kotlin       | Java               | 含义                          |
|--------------|--------------------|-----------------------------|
| `Producer<out T>` | `Producer<? extends T>` | 协变，允许子类型替换父类型（仅输出） |
| `Consumer<in T>`   | `Consumer<? super T>`   | 逆变，允许父类型替换子类型（仅输入） |


### **总结**
协变（`out`）和逆变（`in`）是 Kotlin 泛型中控制类型参数子类型关系的核心机制：  
- **协变**：类型参数仅用于输出（返回值），允许子类型泛型类替换父类型泛型类（如 `List<Int>` → `List<Number>`）。  
- **逆变**：类型参数仅用于输入（参数），允许父类型泛型类替换子类型泛型类（如 `Comparator<Number>` → `Comparator<Int>`）。  
- **星投影**：`*` 表示未知类型，用于简化泛型类型声明。  

通过合理使用 `out` 和 `in`，可以在保证类型安全的同时，大幅提升代码的复用性和灵活性。