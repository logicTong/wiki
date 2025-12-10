在 Hilt 中，`@Qualifier`（限定符）是解决**同一类型多实例注入冲突**的核心机制——当你需要为同一接口/类提供不同配置的实例（如不同 BaseUrl 的 Retrofit、不同类型的 `UserRepository`）时，自定义 `@Qualifier` 注解可以精准区分这些实例，避免注入时的歧义。

### 一、核心场景：为什么需要自定义 Qualifier？
假设你有两个 `Retrofit` 实例（一个用于用户接口，一个用于订单接口），直接用 `@Provides` 提供会导致 Hilt 无法区分：
```kotlin
// 错误：同一类型（Retrofit）被多次提供，编译报错
@Provides
fun provideUserRetrofit(): Retrofit { ... }

@Provides
fun provideOrderRetrofit(): Retrofit { ... }
```
此时需要自定义 `@Qualifier` 注解，为每个实例打上“标签”，注入时通过标签指定要使用的实例。

### 二、自定义 Qualifier 注解的完整步骤
#### 1. 定义自定义 Qualifier 注解
通过 `@Qualifier` + `@Retention` 注解组合，创建自定义限定符（推荐放在单独的注解文件中）：
```kotlin
import javax.inject.Qualifier
import kotlin.annotation.Retention

// 自定义限定符：标记“用户接口的 Retrofit”
@Qualifier
@Retention(AnnotationRetention.BINARY) // 编译期保留，优化性能
annotation class UserRetrofit

// 自定义限定符：标记“订单接口的 Retrofit”
@Qualifier
@Retention(AnnotationRetention.BINARY)
annotation class OrderRetrofit

// 拓展：带参数的 Qualifier（Kotlin 1.6+ 支持）
@Qualifier
@Retention(AnnotationRetention.BINARY)
annotation class CustomRetrofit(val baseUrl: String)
```

**关键注解说明**：
- `@Qualifier`：标记这是 Hilt 限定符注解；
- `@Retention`：指定注解保留策略，推荐用 `BINARY`（编译期保留，比 `RUNTIME` 更高效）；
- 注解命名：建议语义化（如 `UserRetrofit`/`OrderRetrofit`），避免歧义。

#### 2. 在 @Provides/@Binds 中绑定 Qualifier
为不同实例的提供方法添加自定义限定符，标记实例的“身份”：
```kotlin
@Module
@InstallIn(SingletonComponent::class)
object NetworkModule {
    // 1. 提供用户接口的 Retrofit（标记 @UserRetrofit）
    @Provides
    @UserRetrofit // 绑定自定义限定符
    @Singleton
    fun provideUserRetrofit(): Retrofit {
        return Retrofit.Builder()
            .baseUrl("https://user.example.com/")
            .addConverterFactory(GsonConverterFactory.create())
            .build()
    }

    // 2. 提供订单接口的 Retrofit（标记 @OrderRetrofit）
    @Provides
    @OrderRetrofit // 绑定自定义限定符
    @Singleton
    fun provideOrderRetrofit(): Retrofit {
        return Retrofit.Builder()
            .baseUrl("https://order.example.com/")
            .addConverterFactory(GsonConverterFactory.create())
            .build()
    }

    // 3. 基于限定符的 Retrofit 提供 ApiService
    @Provides
    @Singleton
    fun provideUserApi(@UserRetrofit retrofit: Retrofit): UserApiService {
        return retrofit.create(UserApiService::class.java)
    }

    @Provides
    @Singleton
    fun provideOrderApi(@OrderRetrofit retrofit: Retrofit): OrderApiService {
        return retrofit.create(OrderApiService::class.java)
    }
}
```

#### 3. 注入时通过 Qualifier 指定实例
在需要注入的地方，用自定义限定符标注参数，指定要使用的实例：
```kotlin
// 场景1：ViewModel 中注入
class MainViewModel @Inject constructor(
    @UserRetrofit private val userRetrofit: Retrofit, // 指定用户 Retrofit
    @OrderRetrofit private val orderRetrofit: Retrofit, // 指定订单 Retrofit
    private val userApi: UserApiService, // 自动关联 @UserRetrofit 的 Retrofit
    private val orderApi: OrderApiService // 自动关联 @OrderRetrofit 的 Retrofit
) : ViewModel() {
    // 业务逻辑...
}

// 场景2：Activity/Fragment 中注入（@AndroidEntryPoint 标注后）
@AndroidEntryPoint
class MainActivity : AppCompatActivity() {
    // 直接注入带限定符的实例
    @Inject
    @UserRetrofit
    lateinit var userRetrofit: Retrofit

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        // 使用实例...
    }
}
```

### 三、进阶用法：@Named 与自定义 Qualifier 对比
Hilt 内置了 `@Named` 限定符（基于字符串区分），但**推荐优先使用自定义 Qualifier**，而非 `@Named`：

| 特性                | 自定义 Qualifier                | @Named("xxx")                  |
|---------------------|---------------------------------|--------------------------------|
| 类型安全            | ✅ 编译期检查（写错注解会报错） | ❌ 字符串硬编码，易写错且无检查 |
| 可读性              | ✅ 语义化命名（如 @UserRetrofit）| ❌ 字符串语义模糊（如 "user_retrofit"） |
| 拓展性              | ✅ 可添加参数（如 baseUrl）| ❌ 仅支持字符串参数            |
| 代码提示            | ✅ IDE 自动提示注解名           | ❌ 需手动输入字符串，无提示    |

**反例（不推荐）**：
```kotlin
// 用 @Named 的写法（易出错）
@Provides
@Named("user_retrofit")
fun provideUserRetrofit(): Retrofit { ... }

// 注入时
@Inject
@Named("user_retrofit") // 字符串写错不会编译报错
lateinit var userRetrofit: Retrofit
```

### 四、带参数的自定义 Qualifier（高级）
Kotlin 1.6+ 支持为 Qualifier 注解添加参数，适用于动态区分实例的场景（如多环境 BaseUrl）：
```kotlin
// 定义带参数的 Qualifier
@Qualifier
@Retention(AnnotationRetention.BINARY)
annotation class EnvRetrofit(val env: String) // env: dev/test/prod

// 提供不同环境的 Retrofit
@Provides
@EnvRetrofit("dev")
fun provideDevRetrofit(): Retrofit {
    return Retrofit.Builder().baseUrl("https://dev.example.com/").build()
}

@Provides
@EnvRetrofit("prod")
fun provideProdRetrofit(): Retrofit {
    return Retrofit.Builder().baseUrl("https://prod.example.com/").build()
}

// 注入时指定参数
@Inject
@EnvRetrofit("dev")
lateinit var devRetrofit: Retrofit
```

### 五、常见避坑点
1. **Qualifier 注解未加 @Retention**：默认保留策略是 `SOURCE`（源码级），Hilt 编译期无法识别，必须显式指定 `BINARY` 或 `RUNTIME`；
2. **重复 Qualifier 绑定**：同一 Qualifier 绑定到多个同类型实例，会导致编译报错（一个 Qualifier 只能对应一个实例）；
3. **注入时遗漏 Qualifier**：同一类型有多个实例时，未标注 Qualifier 会触发“多个绑定匹配”的编译错误；
4. **Qualifier 命名冲突**：避免与系统/第三方库的 Qualifier 重名（如不要命名为 `@Singleton`）。

### 六、最佳实践
1. **语义化命名**：Qualifier 名称要体现实例的用途（如 `@UserRetrofit`/`@OrderRetrofit`），而非实现细节；
2. **集中管理**：将所有自定义 Qualifier 放在同一个文件（如 `Qualifiers.kt`），便于维护；
3. **优先自定义，禁用 @Named**：除非简单临时场景，否则避免 `@Named` 的字符串硬编码；
4. **搭配作用域**：Qualifier 仅区分实例，作用域（如 `@Singleton`）控制实例生命周期，两者需配合使用。

### 总结
自定义 `@Qualifier` 注解是 Hilt 解决“同一类型多实例注入”的标准方案，核心流程是：
1. 定义带 `@Qualifier` + `@Retention` 的自定义注解；
2. 在 `@Provides`/`@Binds` 方法上绑定该注解；
3. 注入时通过注解指定要使用的实例。

相比内置的 `@Named`，自定义 Qualifier 具备类型安全、可读性高、拓展性强的优势，是生产环境的首选方案。