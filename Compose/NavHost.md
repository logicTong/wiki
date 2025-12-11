在 Jetpack Compose 中，`NavHost` 是导航组件（Navigation Compose）的核心，用于管理多个可组合项（Composable）之间的导航，实现页面跳转、参数传递、返回栈管理等功能。以下是 **完整的使用步骤 + 核心示例 + 关键知识点**，帮助你快速掌握 `NavHost` 的用法。

### 一、前置准备：添加依赖
首先需要在 `build.gradle`（Module 级别）中添加 Navigation Compose 依赖（确保版本与你的 Compose 版本兼容）：
```gradle
dependencies {
    // 核心导航依赖（推荐使用最新稳定版）
    implementation "androidx.navigation:navigation-compose:2.7.7"
    
    // 若需深度链接/动画等扩展功能，可添加：
    // implementation "androidx.navigation:navigation-common-ktx:2.7.7"
}
```

### 二、核心概念
1. **NavController**：导航控制器，负责管理导航栈、触发跳转、获取参数等，是 `NavHost` 的核心依赖。
2. **NavHost**：承载导航目的地（Destination）的容器，接收 `NavController` 和起始目的地，定义所有可导航的页面。
3. **Destination**：导航目的地，即单个可组合项（如首页、详情页），通过 `composable()` 方法注册。

### 三、基础使用步骤（完整示例）
#### 步骤1：创建 NavController
通过 `rememberNavController()` 创建可记忆的 `NavController`（确保重组时不重复创建）：
```kotlin
import androidx.compose.runtime.Composable
import androidx.compose.ui.platform.LocalContext
import androidx.navigation.NavHostController
import androidx.navigation.compose.NavHost
import androidx.navigation.compose.composable
import androidx.navigation.compose.rememberNavController

// 定义路由常量（推荐抽离，避免硬编码）
object NavRoutes {
    const val Home = "home"
    const val Detail = "detail/{id}" // 带参数的路由（{参数名} 声明参数）
    const val DetailWithOptional = "detail_optional?id={id}" // 可选参数
}

@Composable
fun MainNavigation() {
    // 1. 创建 NavController
    val navController: NavHostController = rememberNavController()
    
    // 2. 配置 NavHost
    NavHost(
        navController = navController,
        startDestination = NavRoutes.Home, // 初始显示的页面
        // 可选：添加进入/退出动画
        // enterTransition = { slideIntoContainer(AnimatedContentScope.SlideDirection.Left) },
        // exitTransition = { slideOutOfContainer(AnimatedContentScope.SlideDirection.Left) }
    ) {
        // 3. 注册目的地（Composable）
        // 首页
        composable(route = NavRoutes.Home) {
            HomeScreen(
                onNavigateToDetail = { id ->
                    // 跳转到详情页（传递参数）
                    navController.navigate(NavRoutes.Detail.replace("{id}", id.toString()))
                }
            )
        }
        
        // 详情页（带必选参数）
        composable(route = NavRoutes.Detail) { backStackEntry ->
            // 获取路由参数
            val id = backStackEntry.arguments?.getString("id") ?: "默认ID"
            DetailScreen(
                id = id,
                onBackClick = {
                    // 返回上一页（NavController 自动管理返回栈）
                    navController.popBackStack()
                },
                onNavigateToHome = {
                    // 跳转到首页并清空返回栈（避免返回详情页）
                    navController.navigate(NavRoutes.Home) {
                        popUpTo(NavRoutes.Home) { inclusive = true }
                    }
                }
            )
        }
        
        // 可选：带可选参数的页面
        composable(route = NavRoutes.DetailWithOptional) { backStackEntry ->
            val id = backStackEntry.arguments?.getString("id") ?: "可选参数默认值"
            OptionalParamScreen(id = id)
        }
    }
}
```

#### 步骤2：定义页面 Composable（HomeScreen/DetailScreen）
```kotlin
import androidx.compose.foundation.layout.Arrangement
import androidx.compose.foundation.layout.Column
import androidx.compose.foundation.layout.Spacer
import androidx.compose.foundation.layout.fillMaxSize
import androidx.compose.foundation.layout.height
import androidx.compose.foundation.layout.padding
import androidx.compose.material3.Button
import androidx.compose.material3.Text
import androidx.compose.runtime.Composable
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.unit.dp

// 首页
@Composable
fun HomeScreen(onNavigateToDetail: (Int) -> Unit) {
    Column(
        modifier = Modifier.fillMaxSize().padding(20.dp),
        horizontalAlignment = Alignment.CenterHorizontally,
        verticalArrangement = Arrangement.Center
    ) {
        Text(text = "首页")
        Spacer(modifier = Modifier.height(20.dp))
        Button(onClick = { onNavigateToDetail(1001) }) {
            Text("跳转到详情页（参数：1001）")
        }
    }
}

// 详情页
@Composable
fun DetailScreen(id: String, onBackClick: () -> Unit, onNavigateToHome: () -> Unit) {
    Column(
        modifier = Modifier.fillMaxSize().padding(20.dp),
        horizontalAlignment = Alignment.CenterHorizontally,
        verticalArrangement = Arrangement.Center
    ) {
        Text(text = "详情页 - ID：$id")
        Spacer(modifier = Modifier.height(20.dp))
        Button(onClick = onBackClick) {
            Text("返回上一页")
        }
        Spacer(modifier = Modifier.height(10.dp))
        Button(onClick = onNavigateToHome) {
            Text("回到首页（清空返回栈）")
        }
    }
}

// 可选参数页面
@Composable
fun OptionalParamScreen(id: String) {
    Column(
        modifier = Modifier.fillMaxSize().padding(20.dp),
        horizontalAlignment = Alignment.CenterHorizontally,
        verticalArrangement = Arrangement.Center
    ) {
        Text(text = "可选参数页 - ID：$id")
    }
}
```

#### 步骤3：在 Activity 中使用 MainNavigation
```kotlin
import androidx.activity.compose.setContent
import androidx.appcompat.app.AppCompatActivity
import android.os.Bundle
import androidx.compose.material3.MaterialTheme
import androidx.compose.material3.Surface

class MainActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContent {
            MaterialTheme {
                Surface(modifier = Modifier.fillMaxSize()) {
                    MainNavigation() // 挂载导航组件
                }
            }
        }
    }
}
```

### 四、关键进阶用法
#### 1. 参数传递与解析
- **必选参数**：路由格式 `detail/{id}`，通过 `backStackEntry.arguments?.getString("id")` 获取。
- **可选参数**：路由格式 `detail_optional?id={id}`，参数可传可不传，解析方式同上（需设置默认值）。
- **多参数**：路由格式 `detail/{id}/{name}?age={age}`，解析时分别获取即可。
- **类型安全参数**（推荐）：使用 `navigation-compose` 的 `arguments` 配置参数类型，避免类型转换错误：
  ```kotlin
  composable(
      route = NavRoutes.Detail,
      arguments = listOf(
          navArgument("id") {
              type = NavType.IntType // 指定参数类型为Int
              defaultValue = 0 // 默认值（可选）
          }
      )
  ) { backStackEntry ->
      val id = backStackEntry.arguments?.getInt("id") ?: 0 // 直接获取Int类型
  }
  ```

#### 2. 导航行为控制（popUpTo/inclusive）
通过 `navController.navigate()` 的扩展配置，控制返回栈：
```kotlin
navController.navigate(NavRoutes.Home) {
    // 弹出到指定路由（不包含自身）
    popUpTo(NavRoutes.Home) {
        inclusive = true // 包含自身，清空所有返回栈
    }
    launchSingleTop = true // 避免重复创建相同页面（类似singleTop）
}
```

#### 3. 监听导航状态
通过 `NavController` 的 `currentBackStackEntryAsState()` 监听当前页面：
```kotlin
import androidx.compose.runtime.collectAsStateWithLifecycle
import androidx.navigation.currentBackStackEntryAsState

@Composable
fun MainNavigation() {
    val navController = rememberNavController()
    // 监听当前路由
    val currentRoute = navController.currentBackStackEntryAsState().value?.destination?.route
    
    NavHost(navController = navController, startDestination = NavRoutes.Home) {
        // ... 注册目的地
    }
    
    // 示例：根据当前路由显示/隐藏底部导航栏
    if (currentRoute == NavRoutes.Home) {
        BottomNavigationBar(navController)
    }
}
```

#### 4. 嵌套导航（子 NavHost）
若需嵌套导航（如首页包含多个子页面），可创建子 `NavHost`：
```kotlin
composable(route = NavRoutes.Home) {
    // 首页内部的子导航控制器
    val childNavController = rememberNavController()
    Column {
        // 首页内容
        Text("首页")
        // 子 NavHost
        NavHost(
            navController = childNavController,
            startDestination = "home_sub1"
        ) {
            composable("home_sub1") { SubScreen1() }
            composable("home_sub2") { SubScreen2() }
        }
    }
}
```

### 五、常见问题
1. **NavController 未关联 NavHost**：确保 `NavHost` 使用的 `navController` 是通过 `rememberNavController()` 创建的，且唯一对应。
2. **参数解析为 null**：检查路由格式（必选参数用 `{}`，可选参数用 `?key={value}`），或设置参数默认值。
3. **返回栈异常**：使用 `popUpTo` + `inclusive` 清理返回栈，避免重复页面。

### 总结
`NavHost` 的核心流程是：
1. 创建 `NavController` → 
2. 在 `NavHost` 中注册所有目的地（`composable()`）→ 
3. 通过 `navController.navigate()` 触发跳转，`popBackStack()` 返回 → 
4. 解析参数/控制导航行为。

遵循以上模式，可轻松实现 Compose 中页面的导航管理，适配单页面、多参数、嵌套导航等场景。