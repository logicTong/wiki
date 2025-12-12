### 一、什么是 NavGraph（导航图）？
`NavGraph`（导航图）是 Jetpack Navigation 组件（包括 Compose 版 `Navigation Compose`）的核心概念，本质是**页面（Destination）的结构化集合**，用于定义：
- 哪些页面（Composable/Destination）属于同一个逻辑分组；
- 页面之间的默认跳转规则（如启动页、返回栈行为）；
- 分组内的共享配置（如动画、参数传递、ViewModel 作用域）。

简单来说，NavGraph 是对页面导航逻辑的“模块化封装”——比如把“订单流程（列表→详情→支付）”“个人中心（资料→设置→帮助）”分别封装成独立的 NavGraph，让导航逻辑更清晰、可维护。

#### 核心特性：
1. **层级化**：NavGraph 可以嵌套（比如根 Graph 包含订单 Graph、个人中心 Graph）；
2. **作用域隔离**：同一个 NavGraph 内的页面可共享 ViewModel（如订单流程的共享状态）；
3. **统一管控**：可对整个 Graph 配置统一的跳转动画、参数校验、返回行为。

### 二、NavGraph 的定义与页面导航（Compose 版）
#### 前置准备
先添加 Navigation Compose 依赖（以最新版为例）：
```kotlin
dependencies {
    // Navigation Compose 核心
    implementation "androidx.navigation:navigation-compose:2.7.7"
    // 若用 Hilt 整合 ViewModel
    implementation "androidx.hilt:hilt-navigation-compose:1.2.0"
}
```

#### 步骤 1：定义 NavGraph（核心）
通过 `navigation()` 函数定义导航图，分为**根 Graph** 和**子 Graph** 两层：
```kotlin
@Composable
fun AppNavHost(navController: NavController) {
    // 根 NavGraph：包含所有页面/子 Graph
    NavHost(
        navController = navController,
        startDestination = "root_home" // 根 Graph 的默认启动页
    ) {
        // 1. 普通 Destination（单个页面）
        composable(route = "root_home") {
            HomeScreen(navController = navController)
        }

        // 2. 子 NavGraph：封装“订单流程”（模块化）
        navigation(
            startDestination = "order_list", // 子 Graph 的默认启动页
            route = "order_graph" // 子 Graph 的唯一标识
        ) {
            // 子 Graph 内的页面（Destination）
            composable(route = "order_list") {
                OrderListScreen(navController = navController)
            }
            // 带参数的 Destination（如订单详情，传订单 ID）
            composable(
                route = "order_detail/{orderId}", // 占位符传参
                arguments = listOf(
                    navArgument("orderId") {
                        type = NavType.StringType // 参数类型
                        nullable = false // 非空
                    }
                )
            ) { backStackEntry ->
                // 获取页面参数
                val orderId = backStackEntry.arguments?.getString("orderId") ?: ""
                OrderDetailScreen(navController = navController, orderId = orderId)
            }
        }

        // 3. 另一个子 NavGraph：个人中心
        navigation(
            startDestination = "profile_info",
            route = "profile_graph"
        ) {
            composable(route = "profile_info") { ProfileInfoScreen(navController) }
            composable(route = "profile_setting") { ProfileSettingScreen(navController) }
        }
    }
}
```

#### 步骤 2：初始化 NavController 并挂载 NavHost
在 Activity 中创建 `NavController`（导航控制器），并挂载定义好的 `NavHost`：
```kotlin
class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContent {
            MyAppTheme {
                // 1. 创建 NavController（remember 保证重组不重建）
                val navController = rememberNavController()
                
                // 2. 挂载 NavHost（导航容器）
                AppNavHost(navController = navController)
            }
        }
    }
}
```

#### 步骤 3：页面导航（核心操作）
Navigation Compose 通过 `NavController.navigate()` 实现页面跳转，分以下场景：

##### 场景 1：跳转到普通页面（无参数）
```kotlin
// 在 HomeScreen 中跳转到订单列表（子 Graph 的启动页）
@Composable
fun HomeScreen(navController: NavController) {
    Button(
        onClick = {
            // 跳转路径：子 Graph 的 route + 内部 Destination（或直接子 Graph route）
            navController.navigate("order_graph/order_list") 
            // 简化：直接跳转子 Graph，会自动进入其 startDestination
            // navController.navigate("order_graph")
        }
    ) {
        Text("进入订单列表")
    }
}
```

##### 场景 2：跳转到带参数的页面（如订单详情）
```kotlin
// 在 OrderListScreen 中跳转到订单详情（传 orderId）
@Composable
fun OrderListScreen(navController: NavController) {
    // 模拟订单列表
    val orderList = listOf("order_001", "order_002", "order_003")
    
    LazyColumn {
        items(orderList) { orderId ->
            Button(
                onClick = {
                    // 拼接参数：route + / + 参数值
                    navController.navigate("order_graph/order_detail/$orderId")
                }
            ) {
                Text("查看订单 $orderId")
            }
        }
    }
}
```

##### 场景 3：返回上一页（默认行为）
Navigation Compose 自动处理返回键（物理键/系统返回），也可手动调用：
```kotlin
// 在 OrderDetailScreen 中手动返回
@Composable
fun OrderDetailScreen(navController: NavController, orderId: String) {
    Button(
        onClick = {
            // 方式 1：返回上一页（等价于按返回键）
            navController.popBackStack()
            
            // 方式 2：返回指定页面（比如从订单详情直接回到首页）
            // navController.popBackStack(route = "root_home", inclusive = false)
            // inclusive = false：保留目标页面；true：销毁目标页面
        }
    ) {
        Text("返回")
    }
    Text("订单 ID：$orderId")
}
```

##### 场景 4：清除返回栈（如登录后跳转到首页，禁止返回登录页）
```kotlin
// 登录成功后跳转首页，清除之前的返回栈
navController.navigate("root_home") {
    // 清除当前导航栈中所有页面（除了目标页面）
    popUpTo("root_home") { inclusive = true }
    // 防止重复点击导致多次入栈
    launchSingleTop = true
}
```

#### 步骤 4：NavGraph 的进阶用法（共享 ViewModel）
如前文所述，NavGraph 可作为 ViewModel 作用域，实现分组内页面共享状态：
```kotlin
// 在 OrderListScreen/OrderDetailScreen 中共享 OrderViewModel
@Composable
fun OrderListScreen(navController: NavController) {
    // 绑定到 order_graph 的 ViewModel（整个订单流程共享）
    val orderViewModel = hiltViewModel<OrderViewModel>(
        viewModelStoreOwner = navController.getBackStackEntry("order_graph")
    )
    
    // 使用共享数据（比如订单筛选条件）
    val filterType by orderViewModel.filterType.collectAsState()
}
```

### 三、核心概念总结
| 术语          | 作用                                                                 |
|---------------|----------------------------------------------------------------------|
| NavController | 导航控制器：管理导航栈、触发页面跳转/返回、获取 BackStackEntry       |
| NavHost       | 导航容器：承载所有 Destination/NavGraph，是 Composable 导航的根布局  |
| Destination   | 页面节点：对应单个 Composable 页面，是导航的最小单元                 |
| NavGraph      | 导航分组：封装一组相关的 Destination，可嵌套、可共享 ViewModel 作用域 |
| Route         | 唯一标识：每个 Destination/NavGraph 都有唯一 route，用于跳转和定位   |

### 四、关键注意事项
1. **Route 命名规范**：建议用“Graph 标识/页面标识”（如 `order_graph/order_detail`），避免冲突；
2. **参数传递**：复杂参数（如对象）建议用 `navArgument` + `Parcelable`，而非直接拼接（避免路由过长）；
3. **返回栈管理**：跳转时通过 `popUpTo`/`launchSingleTop` 避免返回栈冗余；
4. **嵌套 Graph**：子 Graph 的 route 需唯一，跳转时需指定完整路径（或直接跳转子 Graph 启动页）。