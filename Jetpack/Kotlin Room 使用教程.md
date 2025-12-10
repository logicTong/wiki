# Kotlin Room 使用教程（完整版）
Room 是 Google 推出的 Android 本地数据库框架，基于 SQLite 封装，解决了原生 SQLite 手写 SQL 易出错、无编译期检查、数据映射繁琐等问题，是 Jetpack 组件的核心成员，也是 Kotlin 开发中本地存储的首选方案。

## 一、核心优势
1. **编译期 SQL 检查**：SQL 语句错误会在编译阶段暴露，而非运行时崩溃；
2. **简洁的注解式 API**：无需手写 Cursor、ContentValues，通过注解实现数据映射；
3. **协程友好**：支持挂起函数（suspend），适配 Kotlin 协程异步操作；
4. **可观察数据**：结合 LiveData/Flow 实现数据变化监听，自动更新 UI；
5. **与 Jetpack 无缝集成**：兼容 ViewModel、Lifecycle 等组件。

## 二、环境配置（Android 项目）
### 1. 添加依赖（build.gradle / build.gradle.kts）
在模块级（app）的构建文件中添加以下依赖（适配最新版 AGP 和 Kotlin）：

#### Groovy 版本（build.gradle）
```gradle
plugins {
    id 'com.android.application'
    id 'org.jetbrains.kotlin.android'
    id 'kotlin-kapt' // 必须添加，用于处理 Room 注解
}

android {
    // 确保启用 Java 8 兼容
    compileOptions {
        sourceCompatibility JavaVersion.VERSION_1_8
        targetCompatibility JavaVersion.VERSION_1_8
    }
    kotlinOptions {
        jvmTarget = '1.8'
    }
}

dependencies {
    // Room 核心库
    def room_version = "2.6.1" // 建议使用最新稳定版
    implementation "androidx.room:room-runtime:$room_version"
    kapt "androidx.room:room-compiler:$room_version" // 注解处理器

    // 可选：协程支持（必须）
    implementation "androidx.room:room-ktx:$room_version"
    // 可选：LiveData 支持
    implementation "androidx.room:room-liveData:$room_version"
    // 可选：Paging 3 支持
    implementation "androidx.room:room-paging:$room_version"
}
```

#### Kotlin DSL 版本（build.gradle.kts）
```kotlin
plugins {
    id("com.android.application")
    id("org.jetbrains.kotlin.android")
    id("kotlin-kapt")
}

android {
    compileOptions {
        sourceCompatibility = JavaVersion.VERSION_1_8
        targetCompatibility = JavaVersion.VERSION_1_8
    }
    kotlinOptions {
        jvmTarget = "1.8"
    }
}

dependencies {
    val roomVersion = "2.6.1"
    implementation("androidx.room:room-runtime:$roomVersion")
    kapt("androidx.room:room-compiler:$roomVersion")
    implementation("androidx.room:room-ktx:$roomVersion") // 协程支持
    implementation("androidx.room:room-liveData:$roomVersion") // 可选
}
```

## 三、核心组件与基础用法
Room 包含 3 个核心组件：
- **Entity**：实体类，映射数据库中的表（每个字段对应表的列）；
- **Dao**（Data Access Object）：数据访问对象，定义数据库操作（增删改查）；
- **Database**：数据库抽象类，管理数据库实例、版本、关联 Entity 和 Dao。

### 步骤 1：定义 Entity（表实体）
用 `@Entity` 注解标记实体类，`@PrimaryKey` 标记主键，支持自增、复合主键等。

```kotlin
import androidx.room.Entity
import androidx.room.PrimaryKey

// 定义用户表，表名默认是类名，可通过 tableName 指定
@Entity(tableName = "user_table") 
data class User(
    // 主键自增：autoGenerate = true
    @PrimaryKey(autoGenerate = true) val id: Int = 0,
    // 列名默认是字段名，可通过 @ColumnInfo 指定
    @ColumnInfo(name = "user_name") val name: String,
    val age: Int,
    val email: String? = null // 可空字段
)
```

**关键注解说明**：
- `@Entity(tableName = "xxx")`：指定表名；
- `@PrimaryKey(autoGenerate = true)`：主键自增；
- `@ColumnInfo(name = "xxx")`：指定列名、默认值、索引等；
- `@Ignore`：忽略某个字段（不映射到表列）；
- `@Index`：为字段创建索引（优化查询性能），示例：
  ```kotlin
  @Entity(
      tableName = "user_table",
      indices = [Index(value = ["user_name"], unique = true)] // 用户名唯一索引
  )
  ```

### 步骤 2：定义 Dao（数据访问接口）
用 `@Dao` 注解标记接口/抽象类，通过注解定义增删改查操作，支持协程（`suspend`）、返回 Flow/LiveData。

```kotlin
import androidx.room.Dao
import androidx.room.Delete
import androidx.room.Insert
import androidx.room.Query
import androidx.room.Update
import kotlinx.coroutines.flow.Flow

@Dao
interface UserDao {
    // 插入单条/多条数据，onConflict 处理冲突（如主键重复）
    @Insert(onConflict = OnConflictStrategy.REPLACE) // 冲突时替换
    suspend fun insertUser(user: User)

    @Insert
    suspend fun insertUsers(vararg users: User) // 批量插入

    // 更新数据
    @Update
    suspend fun updateUser(user: User)

    // 删除数据
    @Delete
    suspend fun deleteUser(user: User)

    // 自定义查询：查询所有用户，返回 Flow（数据变化自动通知）
    @Query("SELECT * FROM user_table ORDER BY id ASC")
    fun getAllUsers(): Flow<List<User>>

    // 条件查询：根据年龄查询
    @Query("SELECT * FROM user_table WHERE age > :minAge")
    suspend fun getUsersOlderThan(minAge: Int): List<User>

    // 删除所有数据
    @Query("DELETE FROM user_table")
    suspend fun deleteAllUsers()

    // 根据主键查询单条数据
    @Query("SELECT * FROM user_table WHERE id = :userId LIMIT 1")
    suspend fun getUserById(userId: Int): User?
}
```

**关键注解说明**：
- `@Insert`/`@Update`/`@Delete`：默认支持单条/批量操作，无需写 SQL；
- `@Query`：自定义 SQL 查询（核心），支持参数占位符（`:参数名`）；
- `OnConflictStrategy`：冲突策略（REPLACE 替换、ABORT 终止、IGNORE 忽略）；
- 协程支持：所有操作可加 `suspend` 关键字，避免主线程阻塞；
- Flow/LiveData：返回 Flow 可监听数据变化（Room 自动切换到后台线程）。

### 步骤 3：定义 Database（数据库抽象类）
继承 `RoomDatabase`，用 `@Database` 注解指定版本、关联的 Entity，提供 Dao 实例的抽象方法。

```kotlin
import androidx.room.Database
import androidx.room.Room
import androidx.room.RoomDatabase
import android.content.Context

// 版本号 version = 1，后续升级需修改版本并处理迁移
@Database(
    entities = [User::class], // 关联的 Entity 列表
    version = 1,
    exportSchema = true // 建议开启，生成架构文件（便于版本迁移）
)
abstract class AppDatabase : RoomDatabase() {
    // 提供 Dao 实例（Room 自动实现）
    abstract fun userDao(): UserDao

    // 单例模式：避免重复创建数据库实例
    companion object {
        // volatile 保证多线程可见性
        @Volatile
        private var INSTANCE: AppDatabase? = null

        fun getInstance(context: Context): AppDatabase {
            // 双重校验锁，确保单例
            return INSTANCE ?: synchronized(this) {
                val instance = Room.databaseBuilder(
                    context.applicationContext, // 用 Application 上下文，避免内存泄漏
                    AppDatabase::class.java,
                    "app_database" // 数据库文件名
                )
                // 开发阶段可选：数据库升级时清空数据（生产环境禁用）
                // .fallbackToDestructiveMigration()
                .build()
                INSTANCE = instance
                instance
            }
        }
    }
}
```

**关键说明**：
- 必须继承 `RoomDatabase`，且为抽象类；
- `@Database(version = x)`：版本号，升级数据库时需递增；
- 单例模式：数据库实例创建开销大，必须保证全局唯一；
- `applicationContext`：必须使用应用上下文，避免 Activity 上下文导致内存泄漏。

## 四、数据库操作实战
### 1. 获取数据库与 Dao 实例
在 Activity/Fragment/ViewModel 中获取 Dao 实例（建议结合 ViewModel + 协程使用）：

```kotlin
// 在 Activity 中
val db = AppDatabase.getInstance(applicationContext)
val userDao = db.userDao()

// 在 ViewModel 中（推荐）
class UserViewModel(application: Application) : AndroidViewModel(application) {
    private val userDao = AppDatabase.getInstance(application).userDao()
    // 监听所有用户数据（Flow 自动切换到后台线程）
    val allUsers: Flow<List<User>> = userDao.getAllUsers()

    // 封装操作：调用 Dao 的 suspend 方法，需在协程中执行
    fun insertUser(user: User) = viewModelScope.launch {
        userDao.insertUser(user)
    }

    fun deleteUser(user: User) = viewModelScope.launch {
        userDao.deleteUser(user)
    }
}
```

### 2. 基础增删改查示例
```kotlin
// 1. 插入数据
viewModelScope.launch {
    val user = User(name = "张三", age = 25, email = "zhangsan@example.com")
    userDao.insertUser(user)
}

// 2. 查询所有用户（监听变化）
lifecycleScope.launch {
    userDao.getAllUsers().collect { users ->
        // 数据变化时自动回调，更新 UI
        recyclerView.adapter = UserAdapter(users)
    }
}

// 3. 条件查询
lifecycleScope.launch {
    val users = userDao.getUsersOlderThan(20)
    Log.d("UserQuery", "年龄大于20的用户：$users")
}

// 4. 更新数据
lifecycleScope.launch {
    val oldUser = userDao.getUserById(1)
    oldUser?.let {
        val updatedUser = it.copy(age = 26) // 不可变数据类，用 copy 更新
        userDao.updateUser(updatedUser)
    }
}

// 5. 删除数据
lifecycleScope.launch {
    val user = User(id = 1, name = "张三", age = 26)
    userDao.deleteUser(user)
}
```

## 五、进阶用法
### 1. 数据库版本升级与迁移
当修改 Entity（如新增字段、表）时，需升级数据库版本，并编写迁移脚本（避免数据丢失）。

#### 步骤 1：修改 Entity（新增字段）
```kotlin
@Entity(tableName = "user_table")
data class User(
    @PrimaryKey(autoGenerate = true) val id: Int = 0,
    @ColumnInfo(name = "user_name") val name: String,
    val age: Int,
    val email: String? = null,
    val phone: String? = null // 新增字段
)
```

#### 步骤 2：升级数据库版本（version = 2）
```kotlin
@Database(
    entities = [User::class],
    version = 2, // 版本从 1 升级到 2
    exportSchema = true
)
abstract class AppDatabase : RoomDatabase() {
    // ... 其他代码不变

    // 定义迁移脚本（从版本 1 到 2）
    private val MIGRATION_1_2 = object : Migration(1, 2) {
        override fun migrate(database: SupportSQLiteDatabase) {
            // 执行 SQL：给 user_table 新增 phone 列
            database.execSQL("ALTER TABLE user_table ADD COLUMN phone TEXT")
        }
    }

    companion object {
        @Volatile
        private var INSTANCE: AppDatabase? = null

        fun getInstance(context: Context): AppDatabase {
            return INSTANCE ?: synchronized(this) {
                val instance = Room.databaseBuilder(
                    context.applicationContext,
                    AppDatabase::class.java,
                    "app_database"
                )
                .addMigrations(MIGRATION_1_2) // 添加迁移脚本
                // .fallbackToDestructiveMigration() // 开发阶段：迁移失败则清空数据（生产禁用）
                .build()
                INSTANCE = instance
                instance
            }
        }
    }
}
```

**注意**：
- 迁移脚本需保证 SQL 语法正确，否则会导致数据库升级失败；
- 生产环境禁止使用 `fallbackToDestructiveMigration()`（会清空所有数据）；
- 复杂迁移（如拆分表、修改主键）需谨慎编写 SQL。

### 2. 关联查询（一对一/一对多）
Room 支持通过 `@Relation` 注解实现实体关联（无需手动写 JOIN SQL）。

#### 示例：一对多（用户-订单）
```kotlin
// 1. 定义订单 Entity
@Entity(tableName = "order_table")
data class Order(
    @PrimaryKey(autoGenerate = true) val orderId: Int = 0,
    val userId: Int, // 外键：关联 User 的 id
    val productName: String,
    val price: Double
)

// 2. 定义关联数据类（非 Entity）
data class UserWithOrders(
    @Embedded val user: User, // 嵌入用户实体
    @Relation(
        parentColumn = "id", // 父表（User）的主键
        entityColumn = "userId" // 子表（Order）的外键
    )
    val orders: List<Order> // 用户的所有订单
)

// 3. 在 Dao 中定义关联查询
@Dao
interface OrderDao {
    @Transaction // 事务：保证查询原子性
    @Query("SELECT * FROM user_table WHERE id = :userId")
    suspend fun getUserWithOrders(userId: Int): UserWithOrders?
}
```

### 3. 结合 Paging 3 实现分页查询
Room 支持返回 `PagingSource`，结合 Paging 3 实现高效分页加载：

```kotlin
// 在 Dao 中定义分页查询
@Dao
interface UserDao {
    @Query("SELECT * FROM user_table ORDER BY id ASC")
    fun getPagedUsers(): PagingSource<Int, User>
}

// 在 ViewModel 中配置 Paging
class UserViewModel(application: Application) : AndroidViewModel(application) {
    private val userDao = AppDatabase.getInstance(application).userDao()
    val pagedUsers = Pager(
        config = PagingConfig(pageSize = 20), // 每页 20 条
        pagingSourceFactory = { userDao.getPagedUsers() }
    ).flow
}

// 在 Activity 中观察分页数据
lifecycleScope.launch {
    viewModel.pagedUsers.collectLatest { pagingData ->
        adapter.submitData(pagingData)
    }
}
```

## 六、最佳实践与避坑点
### 1. 最佳实践
- **协程优先**：所有数据库操作必须放在协程中（用 `suspend` 或 `Flow`），禁止在主线程执行；
- **单例管理**：数据库实例必须单例，避免多实例导致锁冲突；
- **Flow 监听数据**：用 Flow 替代 LiveData（更贴合 Kotlin 协程），实现数据变化自动更新 UI；
- **合理索引**：为高频查询字段创建索引，避免全表扫描；
- **版本迁移**：提前规划表结构，升级时编写完整的迁移脚本；
- **结合 ViewModel/Lifecycle**：在 ViewModel 中封装数据库操作，利用 `viewModelScope` 管理协程生命周期。

### 2. 常见坑点
- **上下文泄漏**：必须使用 `applicationContext` 创建数据库实例，避免 Activity/Fragment 上下文泄漏；
- **主线程操作**：Room 默认禁止主线程执行数据库操作（可通过 `allowMainThreadQueries()` 关闭，但强烈不推荐）；
- **数据类不可变**：Entity 建议用 `data class`，但避免修改字段（如需更新，用 `copy` 生成新对象）；
- **SQL 语法错误**：`@Query` 中的 SQL 错误仅在编译期提示，需仔细检查；
- **迁移脚本错误**：升级版本时，迁移脚本的 SQL 需与 Entity 变更完全匹配。

## 七、调试技巧
1. **查看数据库文件**：
   - 模拟器：`data/data/包名/databases/` 目录下的 `.db` 文件；
   - 真机：需 root 或通过 Android Studio 的 `App Inspection` 查看（推荐）。
2. **App Inspection 调试**：
   - 打开 Android Studio → View → Tool Windows → App Inspection；
   - 选择运行中的应用 → 找到 Room 数据库 → 查看表数据、执行 SQL 测试。

## 总结
Room 是 Kotlin 开发 Android 本地数据库的标准方案，核心流程是：
1. 定义 Entity 映射表；
2. 定义 Dao 封装数据库操作；
3. 定义 Database 管理实例和版本；
4. 通过协程/Flow 执行异步操作。

掌握基础用法后，可结合协程、Flow、Paging 3 等实现更复杂的本地数据管理，同时注意版本迁移和性能优化，避免常见坑点。