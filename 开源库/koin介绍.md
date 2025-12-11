# Kotlin Koin 全面指南

Koin 是一款为 Kotlin 开发者设计的**轻量级依赖注入（DI）框架**，基于 Kotlin 语言特性（如 lambda、扩展函数）构建，无需代码生成、注解处理或反射，旨在简化应用的依赖管理，提升代码的可测试性和可维护性。


## 一、Koin 核心优势
相比 Dagger（另一款主流 DI 框架），Koin 的核心优势在于**简洁、易用、无侵入性**，具体体现在：
- **无代码生成/反射**：不依赖注解处理器（如 `@Module`/`@Inject`），编译速度更快，调试更简单。
- **Kotlin 原生**：充分利用 Kotlin 语法特性，API 简洁直观，学习成本低。
- **轻量级**：体积小（核心库仅几十 KB），无额外依赖，集成成本低。
- **多平台支持**：支持 JVM（Android、Spring Boot）、iOS、Web（Kotlin/JS）等 Kotlin 多平台场景。
- **易于测试**：可快速替换测试环境中的依赖，无需复杂的测试规则配置。


## 二、Koin 核心概念
要使用 Koin，需先理解其 4 个核心概念，它们构成了依赖注入的基础流程：

| 概念         | 作用                                                                 | 关键 API                  |
|--------------|----------------------------------------------------------------------|---------------------------|
| **Module**   | 定义依赖的“容器”，集中声明需要注入的对象（依赖）如何创建。             | `module { ... }`          |
| **Component** | 依赖的“消费者”，即需要从 Koin 中获取依赖的对象（如 Activity、ViewModel）。 | `by inject()` / `get()`   |
| **Scope**    | 控制依赖的生命周期（如“单例”“页面级”“请求级”），避免内存泄漏。         | `single` / `factory` / `scoped` |
| **Qualifier** | 当同一类型有多个实例时，用于“区分标识”（如“用户数据库”“日志数据库”）。 | `named("xxx")`            |


## 三、Koin 基础使用流程（以 Android 为例）
以下以 Android 项目为例，演示 Koin 的完整集成与使用步骤（JVM 后端、多平台项目流程类似，仅初始化入口不同）。


### 1. 集成依赖
首先在 `build.gradle`（Module 级）中添加 Koin 核心库及 Android 专用库：
```gradle
dependencies {
    // Koin 核心库（JVM 基础）
    implementation "io.insert-koin:koin-core:3.5.3"
    // Android 专用库（支持 Activity/Fragment/ViewModel）
    implementation "io.insert-koin:koin-android:3.5.3"
    // ViewModel 支持（需配合 AndroidX ViewModel）
    implementation "io.insert-koin:koin-androidx-viewmodel:3.5.3"
    // 测试支持（可选）
    testImplementation "io.insert-koin:koin-test:3.5.3"
}
```

> 注：Koin 版本需与项目的 Kotlin 版本兼容，最新版本可参考 [Koin 官方文档](https://insert-koin.io/docs/quickstart/android)。


### 2. 定义 Module（声明依赖）
通过 `module { ... }` 块声明依赖的创建规则，例如定义一个“用户仓库”和“网络服务”的依赖：

```kotlin
// 1. 先定义依赖的类（业务逻辑）
class UserRepository(private val api: ApiService) {
    // 业务方法：获取用户列表
    suspend fun getUsers() = api.getUsers()
}

class ApiService {
    // 网络请求方法（模拟）
    suspend fun getUsers(): List<String> = listOf("Alice", "Bob")
}

// 2. 定义 Koin Module（声明依赖如何创建）
val appModule = module {
    // ① 单例依赖：整个应用生命周期内只有一个实例
    single { ApiService() } // 无依赖，直接创建

    // ② 单例依赖：依赖 ApiService（通过 get() 获取已声明的依赖）
    single { UserRepository(get()) } 
}
```

#### 常见依赖声明方式
- `single { ... }`：单例模式，全局唯一实例。
- `factory { ... }`：工厂模式，每次获取依赖时创建新实例（适合“一次性”对象，如网络请求参数）。
- `scoped { ... }`：作用域模式，仅在指定作用域内有效（如 Android 的“页面级”作用域，页面销毁后依赖释放）。


### 3. 初始化 Koin（注入依赖容器）
在应用启动时初始化 Koin，将定义的 Module 传入，让 Koin 管理依赖。  
Android 中通常在 `Application` 类中初始化：

```kotlin
class MyApp : Application() {
    override fun onCreate() {
        super.onCreate()
        // 初始化 Koin
        startKoin {
            // 日志级别（DEBUG 用于调试，上线可改为 NONE）
            androidLogger(Level.DEBUG)
            // 绑定 Android 上下文（用于获取 Context 依赖）
            androidContext(this@MyApp)
            // 加载定义的 Module
            modules(appModule)
        }
    }
}
```

> 注意：需在 `AndroidManifest.xml` 中注册 `MyApp` 类，否则初始化不生效。


### 4. 注入依赖（Component 消费依赖）
在需要依赖的地方（如 Activity、ViewModel），通过 `by inject()` 或 `get()` 从 Koin 中获取依赖。

#### 示例 1：在 Activity 中注入依赖
```kotlin
class MainActivity : AppCompatActivity() {
    // 方式 1：懒加载注入（推荐，首次使用时才获取依赖）
    private val userRepo: UserRepository by inject()

    // 方式 2：立即获取（不推荐，可能提前初始化导致资源浪费）
    // private val userRepo = get<UserRepository>()

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        // 使用依赖调用业务逻辑
        lifecycleScope.launch {
            val users = userRepo.getUsers()
            Log.d("MainActivity", "用户列表：$users") // 输出：[Alice, Bob]
        }
    }
}
```

#### 示例 2：在 ViewModel 中注入依赖
Koin 提供了 `viewModel` 扩展，支持 ViewModel 的依赖注入：
```kotlin
// 1. 定义 ViewModel（依赖 UserRepository）
class UserViewModel(private val userRepo: UserRepository) : ViewModel() {
    private val _users = MutableLiveData<List<String>>()
    val users: LiveData<List<String>> = _users

    // 加载用户列表
    fun loadUsers() {
        viewModelScope.launch {
            _users.value = userRepo.getUsers()
        }
    }
}

// 2. 在 Module 中声明 ViewModel 依赖
val appModule = module {
    // ... 其他依赖（ApiService、UserRepository）...

    // 声明 ViewModel（Koin 会自动管理 ViewModel 生命周期）
    viewModel { UserViewModel(get()) }
}

// 3. 在 Activity 中获取 ViewModel（无需手动传依赖）
class MainActivity : AppCompatActivity() {
    // 注入 ViewModel（Koin 扩展方法）
    private val userViewModel: UserViewModel by viewModel()

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        // 观察数据
        userViewModel.users.observe(this) { users ->
            Log.d("MainActivity", "ViewModel 中的用户列表：$users")
        }

        // 调用 ViewModel 方法
        userViewModel.loadUsers()
    }
}
```


## 四、Koin 高级特性
### 1. 用 Qualifier 区分同类型依赖
当同一类型有多个实例时（如“本地数据库”和“远程数据库”），用 `named("xxx")` 标识：

```kotlin
// 1. 定义同类型的两个依赖
val dbModule = module {
    // 本地数据库（标识为 "local_db"）
    single(named("local_db")) { Database("Local") }
    // 远程数据库（标识为 "remote_db"）
    single(named("remote_db")) { Database("Remote") }
}

// 2. 注入时指定 Qualifier
class DataManager(
    // 注入本地数据库
    @Named("local_db") private val localDb: Database,
    // 注入远程数据库
    @Named("remote_db") private val remoteDb: Database
) {
    fun printDbNames() {
        Log.d("DataManager", "本地 DB：${localDb.name}，远程 DB：${remoteDb.name}")
    }
}

// 3. 在 Module 中声明 DataManager（需指定 Qualifier）
val appModule = module {
    includes(dbModule) // 引入 dbModule
    single { DataManager(get(named("local_db")), get(named("remote_db"))) }
}
```


### 2. 作用域（Scope）管理生命周期
Koin 的 Scope 用于控制依赖的生命周期，避免内存泄漏。例如 Android 中“页面级”Scope：

```kotlin
// 1. 定义 Scope（标识为 "activity_scope"）
val activityScope = module {
    // 作用域内的依赖（页面销毁后释放）
    scoped { ActivityData() }
}

// 2. 在 Activity 中创建并绑定 Scope
class MainActivity : AppCompatActivity() {
    private lateinit var activityScope: Scope

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        // 创建 Scope 并绑定 Activity 生命周期
        activityScope = getKoin().createScope("main_activity_scope", named("activity_scope"))

        // 从 Scope 中获取依赖（仅在该 Activity 内有效）
        val activityData: ActivityData by activityScope.inject()
    }

    override fun onDestroy() {
        super.onDestroy()
        // 销毁 Scope（释放依赖）
        activityScope.close()
    }
}
```


### 3. 测试时替换依赖
Koin 简化了测试流程，可在测试中替换真实依赖为 Mock 实例：

```kotlin
// 测试类（使用 JUnit + MockK）
class UserRepositoryTest : KoinTest { // 实现 KoinTest 以使用 Koin API
    @Test
    fun `getUsers 应返回正确数据`() {
        // 1. 启动测试用的 Koin（替换真实 ApiService 为 Mock）
        startKoin {
            modules(module {
                // Mock ApiService（返回测试数据）
                single { mock<ApiService> { coEvery { getUsers() } returns listOf("Test User") } }
                single { UserRepository(get()) }
            })
        }

        // 2. 获取依赖并测试
        val userRepo: UserRepository by inject()
        runTest {
            val users = userRepo.getUsers()
            assertEquals(1, users.size)
            assertEquals("Test User", users[0])
        }

        // 3. 停止 Koin（避免影响其他测试）
        stopKoin()
    }
}
```


## 五、Koin 适用场景与注意事项
### 适用场景
- **中小型项目**：无需复杂的 DI 配置，快速上手。
- **Kotlin 多平台项目**：统一 JVM/Android/iOS 的依赖管理。
- **注重编译速度**：无代码生成，编译效率高于 Dagger。

### 注意事项
- **大型项目**：若依赖关系极其复杂（如数百个 Module），Dagger 的编译时校验可能更有优势。
- **避免过度依赖**：仅对“跨组件复用”“需要测试替换”的对象使用 DI（如 Repository、ApiService），简单工具类无需注入。
- **生命周期管理**：非单例依赖（如 `factory`/`scoped`）需注意避免内存泄漏（尤其是 Android 中）。


## 六、参考资源
- [Koin 官方文档](https://insert-koin.io/)：最新 API 与多平台使用指南。
- [Koin GitHub](https://github.com/InsertKoinIO/koin)：源码与示例项目。
- [Android 官方 Koin 教程](https://developer.android.com/jetpack/androidx/releases/koin)：结合 Jetpack 组件的使用示例。