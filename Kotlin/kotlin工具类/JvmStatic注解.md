在 Kotlin 中，`@JvmStatic` 和 `@JvmOverloads` 是两个用于与 Java 交互的注解，主要解决 Kotlin 特性在 Java 中调用时的兼容性问题，各自作用如下：


### 1. `@JvmStatic`
用于将 Kotlin 中的函数或属性暴露为 Java 中的**静态成员**，方便在 Java 中直接通过类名调用（无需实例化对象）。

#### 适用场景：
- **伴生对象（companion object）中的函数**：  
  Kotlin 伴生对象的函数默认在 Java 中是通过 `ClassName.Companion` 调用的，添加 `@JvmStatic` 后可直接通过 `ClassName` 调用。
  
  ```kotlin
  class MyClass {
      companion object {
          // Java 中需通过 MyClass.Companion.foo() 调用
          fun foo() { ... }
          
          // 添加 @JvmStatic 后，Java 中可直接 MyClass.bar() 调用
          @JvmStatic
          fun bar() { ... }
      }
  }
  ```

- **顶层函数（top-level functions）**：  
  顶层函数默认会被编译到一个自动生成的 Java 类中（如 `FileNameKt`），若在伴生对象中用 `@JvmStatic` 包裹，可调整调用方式（较少见）。

- **对象声明（object）中的函数**：  
  类似伴生对象，`@JvmStatic` 可将单例对象的函数暴露为静态方法。


### 2. `@JvmOverloads`
用于让 Kotlin 的**带默认参数的函数**在 Java 中生成多个重载方法（overloads），解决 Java 不支持默认参数的问题。

#### 作用：
Kotlin 函数支持默认参数（如 `fun f(a: Int, b: String = "")`），但 Java 调用时必须显式传入所有参数。`@JvmOverloads` 会自动生成多个重载方法，省略默认参数的组合。

#### 示例：
```kotlin
class MyClass {
    // 带默认参数的函数
    @JvmOverloads
    fun printInfo(name: String, age: Int = 18, gender: String = "unknown") {
        println("$name, $age, $gender")
    }
}
```

编译后，Java 中会生成 3 个重载方法：
```java
// 对应 Kotlin 的默认参数组合
public void printInfo(String name) { ... }
public void printInfo(String name, int age) { ... }
public void printInfo(String name, int age, String gender) { ... }
```

这样在 Java 中就可以灵活调用：
```java
MyClass obj = new MyClass();
obj.printInfo("Alice"); // 自动使用 age=18, gender="unknown"
obj.printInfo("Bob", 20); // 自动使用 gender="unknown"
```


### 总结：
- `@JvmStatic`：让 Kotlin 函数/属性在 Java 中以静态成员形式存在，简化调用。
- `@JvmOverloads`：为带默认参数的 Kotlin 函数生成 Java 重载方法，适配 Java 不支持默认参数的特性。

两者常结合使用（例如在伴生对象中同时标注），优化 Kotlin 代码对 Java 的兼容性。