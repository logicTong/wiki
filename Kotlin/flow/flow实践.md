以下是一个在 Android 中使用 Kotlin Flow 的完整示例，展示如何从数据层到 UI 层实现响应式数据流：

### 1. 添加依赖 (app/build.gradle)
```gradle
dependencies {
    implementation "org.jetbrains.kotlinx:kotlinx-coroutines-android:1.6.4"
    implementation "androidx.lifecycle:lifecycle-viewmodel-ktx:2.5.1"
    implementation "androidx.lifecycle:lifecycle-runtime-ktx:2.5.1"
    implementation "androidx.activity:activity-ktx:1.6.1"
}
```

### 2. 创建数据层 (DataRepository.kt)
```kotlin
import kotlinx.coroutines.delay
import kotlinx.coroutines.flow.Flow
import kotlinx.coroutines.flow.flow

class DataRepository {
    // 模拟网络请求
    fun fetchData(): Flow<Resource<String>> = flow {
        emit(Resource.Loading)
        delay(2000) // 模拟网络延迟
        
        try {
            val data = "Flow Data: ${System.currentTimeMillis()}"
            emit(Resource.Success(data))
        } catch (e: Exception) {
            emit(Resource.Error("Network error"))
        }
    }

    // 模拟实时更新
    fun getTickerData(): Flow<Int> = flow {
        var count = 0
        while (true) {
            delay(1000)
            emit(count++)
        }
    }
}

sealed class Resource<T>(val data: T? = null, val message: String? = null) {
    class Success<T>(data: T) : Resource<T>(data)
    class Error<T>(message: String, data: T? = null) : Resource<T>(data, message)
    object Loading : Resource<Nothing>()
}
```

### 3. 创建 ViewModel (MainViewModel.kt)
```kotlin
import androidx.lifecycle.ViewModel
import androidx.lifecycle.viewModelScope
import kotlinx.coroutines.flow.*
import kotlinx.coroutines.launch

class MainViewModel : ViewModel() {
    private val repository = DataRepository()
    
    // StateFlow 保存 UI 状态
    private val _uiState = MutableStateFlow<Resource<String>?>(null)
    val uiState: StateFlow<Resource<String>?> = _uiState.asStateFlow()
    
    // SharedFlow 用于一次性事件（如显示 Toast）
    private val _toastEvents = MutableSharedFlow<String>()
    val toastEvents: SharedFlow<String> = _toastEvents.asSharedFlow()
    
    // 实时计数器
    val tickerFlow = repository.getTickerData()
        .stateIn(
            scope = viewModelScope,
            started = SharingStarted.WhileSubscribed(5000),
            initialValue = 0
        )

    fun fetchData() {
        viewModelScope.launch {
            repository.fetchData()
                .catch { e ->
                    _toastEvents.emit("Error: ${e.message}")
                }
                .collect { resource ->
                    _uiState.value = resource
                    
                    // 成功时显示通知
                    if (resource is Resource.Success) {
                        _toastEvents.emit("数据加载成功!")
                    }
                }
        }
    }
}
```

### 4. 创建 Activity (MainActivity.kt)
```kotlin
import android.os.Bundle
import android.widget.Button
import android.widget.TextView
import androidx.activity.viewModels
import androidx.appcompat.app.AppCompatActivity
import androidx.lifecycle.lifecycleScope
import kotlinx.coroutines.flow.collectLatest
import kotlinx.coroutines.launch

class MainActivity : AppCompatActivity() {
    private val viewModel by viewModels<MainViewModel>()

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        val tvData = findViewById<TextView>(R.id.tvData)
        val tvCounter = findViewById<TextView>(R.id.tvCounter)
        val btnFetch = findViewById<Button>(R.id.btnFetch)

        // 收集 UI 状态
        lifecycleScope.launch {
            viewModel.uiState.collect { resource ->
                when (resource) {
                    is Resource.Loading -> tvData.text = "加载中..."
                    is Resource.Success -> tvData.text = resource.data
                    is Resource.Error -> tvData.text = "错误: ${resource.message}"
                    else -> Unit
                }
            }
        }

        // 收集一次性通知
        lifecycleScope.launch {
            viewModel.toastEvents.collect { message ->
                // 实际项目中替换为 Toast.makeText(this, message, Toast.LENGTH_SHORT).show()
                tvData.append("\n[通知] $message") 
            }
        }

        // 收集实时计数器
        lifecycleScope.launch {
            viewModel.tickerFlow.collectLatest { count ->
                tvCounter.text = "计数器: $count"
            }
        }

        btnFetch.setOnClickListener {
            viewModel.fetchData()
        }
    }
}
```

### 5. 布局文件 (res/layout/activity_main.xml)
```xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical"
    android:padding="16dp"
    android:gravity="center">
    
    <TextView
        android:id="@+id/tvCounter"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="计数器: 0"
        android:textSize="18sp"/>
    
    <Button
        android:id="@+id/btnFetch"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="加载数据"
        android:layout_marginTop="24dp"/>
    
    <TextView
        android:id="@+id/tvData"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:text="点击按钮加载数据"
        android:textSize="20sp"
        android:layout_marginTop="32dp"
        android:gravity="center"/>
    
</LinearLayout>
```

### 示例功能说明：
1. **数据加载**：点击按钮触发网络请求，显示加载状态/成功数据/错误信息
2. **实时计数器**：每秒自动更新的计数器
3. **一次性事件**：使用 SharedFlow 处理 Toast 通知
4. **状态管理**：使用 StateFlow 维护 UI 状态
5. **生命周期感知**：自动取消订阅避免内存泄漏

### 关键点解析：
- `StateFlow`: 用于需要持续观察的状态（如计数器值）
- `SharedFlow`: 用于事件通知（如 Toast/Snackbar）
- `stateIn`: 将冷流转换为热流，支持配置订阅策略
- `collectLatest`: 只处理最新值，忽略中间值
- `WhileSubscribed`: ViewModel 中的推荐启动策略，当所有收集者停止时自动取消

运行此应用，您将看到：
1. 每秒自动更新的计数器
2. 点击按钮加载数据，显示加载状态
3. 2秒后显示数据或错误信息
4. 成功加载时显示通知消息