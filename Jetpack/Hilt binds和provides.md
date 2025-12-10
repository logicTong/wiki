Hilt 中的 `@Provides` 和 `@Binds` 都是用于**向依赖注入容器提供实例**的核心注解，但设计初衷、使用场景和实现方式差异显著——核心区别是：`@Binds` 更轻量（仅绑定接口与实现），`@Provides` 更灵活（可自定义实例创建逻辑）。

### 一、核心差异表
| 维度                | `@Binds`                                  | `@Provides`                              |
|---------------------|-------------------------------------------|------------------------------------------|
| 作用                | 绑定**接口/抽象类**与**具体实现类**        | 自定义实例创建逻辑（new 对象、传参、配置）|
| 方法要求            | 抽象方法（无方法体），参数=实现类，返回=接口 | 具体方法（有方法体），返回=提供的实例     |
| 依赖要求            | 实现类必须是可被 Hilt 注入的（有构造注入） | 可自由创建实例（无需构造注入）|
| 性能                | 编译期生成代码，无额外开销（更高效）| 运行期执行方法逻辑，开销略高             |
| 使用场景            | 接口与实现类的简单绑定                    | 复杂实例创建（如带参数、单例、第三方库） |

### 二、`@Binds`：接口与实现的轻量绑定
#### 1. 适用场景
当你需要告诉 Hilt：「某个接口/抽象类的实例，应该用它的某个具体实现类来提供」，且该实现类**已有构造函数注入**（`@Inject` 标注构造函数）。

#### 2. 用法规则
- 必须定义在 `@Module` + `@InstallIn` 标注的**抽象模块类**中；
- 方法必须是**抽象方法**（无方法体）；
- 方法参数：接口的具体实现类（必须可被 Hilt 注入）；
- 方法返回值：接口/抽象类类型。

#### 3. 示例
```kotlin
// 步骤1：定义接口
interface UserRepository {
    fun getUser(): String
}

// 步骤2：实现类（构造函数注入，让Hilt能创建它）
class UserRepositoryImpl @Inject constructor() : UserRepository {
    override fun getUser() = "默认用户"
}

// 步骤3：抽象模块，用@Binds绑定接口与实现
@Module
@InstallIn(SingletonComponent::class) // 绑定到单例组件
abstract class RepositoryModule {
    // 抽象方法：参数=实现类，返回=接口
    @Binds
    abstract fun bindUserRepository(impl: UserRepositoryImpl): UserRepository
}

// 步骤4：使用（Hilt自动注入接口实例）
class UserViewModel @Inject constructor(
    private val repository: UserRepository // 实际注入的是UserRepositoryImpl
)
```

### 三、`@Provides`：自定义实例创建
#### 1. 适用场景
当实例创建逻辑复杂（无法仅通过构造注入），比如：
- 实例需要传入自定义参数（如上下文、配置项）；
- 实例是第三方库的类（无法修改构造函数加 `@Inject`）；
- 需要控制实例作用域（如单例）、懒加载或多实例；
- 实例创建需要额外逻辑（如初始化、判空）。

#### 2. 用法规则
- 定义在 `@Module` + `@InstallIn` 标注的**普通模块类**（非抽象）中；
- 方法必须是**具体方法**（有方法体），返回值=要提供的实例类型；
- 方法参数：可依赖其他已注入的实例（Hilt 自动传参）；
- 可搭配 `@Singleton`/`@ActivityScoped` 等注解指定作用域。

#### 3. 示例
##### 场景1：创建带参数的实例（第三方库）
```kotlin
// 第三方库类（无法修改构造函数）
class RetrofitClient(val baseUrl: String) {
    fun getApi(): ApiService = Retrofit.Builder()
        .baseUrl(baseUrl)
        .build()
        .create(ApiService::class.java)
}

// Hilt模块：用@Provides提供RetrofitClient实例
@Module
@InstallIn(SingletonComponent::class)
object NetworkModule { // 普通对象类（非抽象）
    // 提供基础URL（可复用）
    @Provides
    fun provideBaseUrl(): String = "https://api.example.com/"

    // 依赖baseUrl，创建RetrofitClient
    @Provides
    @Singleton // 单例作用域
    fun provideRetrofitClient(baseUrl: String): RetrofitClient {
        return RetrofitClient(baseUrl)
    }

    // 基于RetrofitClient提供ApiService
    @Provides
    fun provideApiService(client: RetrofitClient): ApiService {
        return client.getApi()
    }
}

// 使用
class ApiViewModel @Inject constructor(
    private val apiService: ApiService // Hilt自动注入
)
```

##### 场景2：创建需要上下文的实例
```kotlin
@Module
@InstallIn(ActivityComponent::class) // 绑定到Activity组件
object ContextModule {
    // 提供Activity上下文（Hilt已内置ApplicationContext/Activity的注入）
    @Provides
    @ActivityScoped
    fun provideToastHelper(activity: Activity): ToastHelper {
        return ToastHelper(activity)
    }
}

class ToastHelper(private val activity: Activity) {
    fun showToast(msg: String) {
        Toast.makeText(activity, msg, Toast.LENGTH_SHORT).show()
    }
}
```

### 四、关键使用原则
#### 1. 优先用 `@Binds`，其次用 `@Provides`
`@Binds` 是编译期绑定，无运行期方法调用开销，且代码更简洁——只要满足「接口+可注入的实现类」，就优先用 `@Binds`。

#### 2. `@Binds` 无法替代 `@Provides` 的场景
- 实例创建需要自定义参数（如 Retrofit 的 baseUrl）；
- 实例是第三方库的类（无法添加构造注入）；
- 需要对实例做额外初始化（如设置配置、判空）；
- 需要创建不同配置的同类型实例（如多 baseUrl 的 Retrofit）。

#### 3. 作用域注解的使用
两者都可搭配作用域注解（如 `@Singleton`），但注意：
- `@Binds` 的作用域与实现类的作用域一致（若实现类已有 `@Singleton`，方法上可省略）；
- `@Provides` 需显式标注作用域，否则每次注入都会创建新实例。

### 五、常见错误与避坑
1. **`@Binds` 方法非抽象**：`@Binds` 必须定义在抽象模块的抽象方法中，否则编译报错；
2. **`@Binds` 实现类无构造注入**：若实现类没有 `@Inject` 构造函数，Hilt 无法创建它，需改用 `@Provides`；
3. **`@Provides` 模块类设为抽象**：`@Provides` 方法是具体方法，模块类不能是抽象类（可设为 `object` 或普通类）；
4. **重复绑定**：同一接口被 `@Binds`/`@Provides` 多次绑定，需用 `@Qualifier` 限定符区分（如 `@Named("Debug")`）。

### 六、总结
| 选择                | 判定条件                                  |
|---------------------|-------------------------------------------|
| `@Binds`            | 接口 + 可构造注入的实现类 + 无自定义创建逻辑 |
| `@Provides`         | 复杂创建逻辑 / 第三方库 / 自定义参数       |

简单来说：`@Binds` 是“绑定器”，只做接口与实现的映射；`@Provides` 是“工厂方法”，负责实例的创建与配置。日常开发中，优先用 `@Binds` 优化性能，复杂场景用 `@Provides` 兜底。