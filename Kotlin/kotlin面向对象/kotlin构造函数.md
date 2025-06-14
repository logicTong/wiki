在 Kotlin 中，类的构造方法（Constructor）用于初始化类的实例，支持**主构造函数（Primary Constructor）**和**次构造函数（Secondary Constructor）**两种形式。主构造函数是类的主要构造方式，简洁且功能强大；次构造函数作为补充，提供灵活的额外构造逻辑。


### **一、主构造函数（Primary Constructor）**
主构造函数是类的“默认构造函数”，直接在类名后声明，参数可用于初始化类的属性或参与初始化逻辑。


#### **1. 基本声明**
主构造函数使用 `class 类名(参数列表)` 的形式声明，参数可通过 `val` 或 `var` 直接声明为类的属性（避免手动赋值）。  

```kotlin
// 主构造函数声明参数，并直接作为类的属性（val 只读，var 可变）
class Person(val name: String, var age: Int) {
    // 初始化块（init block）：主构造函数的代码逻辑写在这里
    init {
        println("初始化 Person：name=$name, age=$age")
    }
}

// 使用主构造函数创建实例
val person = Person("Alice", 25)  // 输出：初始化 Person：name=Alice, age=25
println(person.name)  // 输出：Alice（直接访问主构造函数声明的属性）
```


#### **2. 初始化块（Init Block）**
主构造函数本身不能包含代码（仅声明参数），所有初始化逻辑需放在 `init` 块中。多个 `init` 块按声明顺序执行。  

```kotlin
class User(var id: Int, val username: String) {
    init {
        println("第一个初始化块：id=$id")
    }
    
    // 第二个 init 块（按顺序执行）
    init {
        println("第二个初始化块：username=$username")
    }
}

// 创建实例时，init 块按顺序执行
val user = User(1, "Bob")
// 输出：
// 第一个初始化块：id=1
// 第二个初始化块：username=Bob
```


#### **3. 构造函数参数作为属性**
主构造函数的参数若声明为 `val` 或 `var`，会自动成为类的属性；若未声明（仅参数类型），则只是构造时的临时变量，不保留为类的属性。  

```kotlin
class Book(
    val title: String,  // 声明为 val，成为类的只读属性
    var price: Double,  // 声明为 var，成为类的可变属性
    isbn: String        // 未声明为 val/var，仅作为构造参数（类中无法访问）
) {
    fun printInfo() {
        println("书名：$title，价格：$price")
        // println(isbn)  ❌ 编译错误：isbn 不是类的属性
    }
}

val book = Book("Kotlin 实战", 99.9, "123456")
book.printInfo()  // 输出：书名：Kotlin 实战，价格：99.9
```


### **二、次构造函数（Secondary Constructor）**
次构造函数是类的辅助构造方式，用于提供额外的初始化逻辑（如默认值、参数重载等）。次构造函数需用 `constructor` 关键字声明，且**必须直接或间接调用主构造函数**（通过 `this()`）。


#### **1. 基本声明与主构造函数调用**
```kotlin
class Animal(val species: String) {
    // 主构造函数：species 是必传参数
    
    // 次构造函数 1：无参数，调用主构造函数（species 默认值为 "Unknown"）
    constructor() : this("Unknown") {
        println("创建了默认物种的动物")
    }
    
    // 次构造函数 2：传入 species 和 age，调用主构造函数
    constructor(species: String, age: Int) : this(species) {
        println("动物物种：$species，年龄：$age")
    }
}

// 使用次构造函数创建实例
val animal1 = Animal()  // 输出：创建了默认物种的动物（species="Unknown"）
val animal2 = Animal("Cat", 3)  // 输出：动物物种：Cat，年龄：3
```


#### **2. 无主构造函数的情况**
若类**没有主构造函数**（例如因使用 `abstract` 或注解导致），次构造函数可直接调用父类构造函数（无需通过 `this()`）。  

```kotlin
// 父类（无主构造函数，仅有次构造函数）
open class Vehicle {
    constructor(brand: String) {
        println("车辆品牌：$brand")
    }
}

// 子类（无主构造函数，次构造函数直接调用父类构造函数）
class Car : Vehicle {
    constructor(brand: String, model: String) : super(brand) {
        println("车型：$model")
    }
}

// 创建实例
val car = Car("Tesla", "Model 3")
// 输出：
// 车辆品牌：Tesla
// 车型：Model 3
```


### **三、继承中的构造函数处理**
Kotlin 中，子类必须在构造时调用父类的构造函数（主或次）。若父类有主构造函数，子类的主构造函数需在类声明时直接调用；若父类无主构造函数，子类需通过次构造函数调用。


#### **示例：子类调用父类主构造函数**
```kotlin
// 父类（有主构造函数）
open class Person(val name: String, var age: Int) {
    init {
        println("Person 初始化：$name")
    }
}

// 子类（主构造函数调用父类主构造函数）
class Student(name: String, age: Int, val grade: Int) : Person(name, age) {
    init {
        println("Student 初始化：$name，年级：$grade")
    }
}

// 创建 Student 实例
val student = Student("Alice", 18, 3)
// 输出：
// Person 初始化：Alice
// Student 初始化：Alice，年级：3
```


### **四、关键注意事项**
1. **主构造函数优先**：类实例化时，主构造函数的参数和 `init` 块会优先执行，次构造函数通过 `this()` 调用主构造函数后，再执行自身逻辑。  
2. **参数作用域**：主构造函数的参数仅在 `init` 块和属性声明中有效；次构造函数的参数仅在其函数体内有效。  
3. **与 Java 的差异**：Kotlin 的主构造函数更简洁（无需显式方法名），且参数可直接声明为类属性（Java 需手动赋值）。  


### **总结**
Kotlin 的构造方法通过主构造函数（简洁初始化）和次构造函数（灵活扩展）的组合，提供了比 Java 更简洁、更灵活的类初始化机制。主构造函数适合大部分场景，次构造函数用于补充特殊需求，结合 `init` 块和属性声明，能高效完成类的初始化逻辑。


    
    
    