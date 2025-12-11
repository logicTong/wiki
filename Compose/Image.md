在Android开发中，`Compose Image`（`Image`组件）用于在Jetpack Compose界面中显示图片，支持多种图片源（如资源文件、网络图片、位图等），并提供了丰富的配置选项（如缩放模式、裁剪、占位符等）。以下是`Image`组件的详细使用方法：

### 一、基本使用
`Image`组件的核心构造函数为：
```kotlin
@Composable
fun Image(
    painter: Painter,
    contentDescription: String?, // 无障碍描述（必填，null需特殊说明）
    modifier: Modifier = Modifier,
    alignment: Alignment = Alignment.Center,
    contentScale: ContentScale = ContentScale.Fit,
    alpha: Float = 1.0f,
    colorFilter: ColorFilter? = null
)
```

#### 1. 加载本地资源图片
通过`painterResource()`加载`res/drawable`或`res/mipmap`目录下的图片资源：
```kotlin
import androidx.compose.foundation.Image
import androidx.compose.runtime.Composable
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.layout.ContentScale
import androidx.compose.ui.res.painterResource

@Composable
fun LocalImageDemo() {
    Image(
        painter = painterResource(id = R.drawable.ic_launcher), // 图片资源ID
        contentDescription = "应用图标", // 无障碍描述（建议非null）
        modifier = Modifier.size(100.dp), // 图片大小
        alignment = Alignment.Center, // 内容对齐方式
        contentScale = ContentScale.Crop, // 缩放模式
        alpha = 0.8f // 透明度（0~1）
    )
}
```

#### 2. 加载网络图片
Jetpack Compose本身不直接支持网络图片加载，需结合第三方库（如**Coil**、**Glide**、**Picasso**）。其中**Coil**为Compose做了专门适配，使用最广泛：

- **步骤1**：添加Coil依赖（build.gradle.kts）：
```kotlin
dependencies {
    implementation("io.coil-kt:coil-compose:2.5.0") // Coil Compose适配库
}
```

- **步骤2**：使用`rememberImagePainter()`加载网络图片：
```kotlin
import androidx.compose.foundation.Image
import androidx.compose.runtime.Composable
import androidx.compose.runtime.remember
import androidx.compose.ui.Modifier
import androidx.compose.ui.layout.ContentScale
import coil.compose.rememberImagePainter

@Composable
fun NetworkImageDemo() {
    val imageUrl = "https://example.com/image.jpg" // 网络图片URL
    
    val painter = rememberImagePainter(
        data = imageUrl,
        builder = {
            crossfade(true) // 淡入淡出效果
            placeholder(R.drawable.placeholder) // 占位图（本地资源）
            error(R.drawable.error) // 加载失败占位图
        }
    )
    
    Image(
        painter = painter,
        contentDescription = "网络图片",
        modifier = Modifier.size(200.dp),
        contentScale = ContentScale.Fit
    )
}
```

#### 3. 加载位图（Bitmap）
直接使用`BitmapPainter`加载内存中的`Bitmap`对象：
```kotlin
import android.graphics.Bitmap
import androidx.compose.foundation.Image
import androidx.compose.runtime.Composable
import androidx.compose.ui.Modifier
import androidx.compose.ui.graphics.asImageBitmap
import androidx.compose.ui.graphics.painter.BitmapPainter

@Composable
fun BitmapImageDemo(bitmap: Bitmap) {
    Image(
        painter = BitmapPainter(bitmap.asImageBitmap()), // 转换Bitmap为Painter
        contentDescription = "位图图片",
        modifier = Modifier.size(150.dp)
    )
}
```

### 二、关键参数说明
1. **contentDescription**:  
   用于无障碍服务（如屏幕阅读器）的图片描述，**建议非null**。若图片为装饰性元素（无实际意义），可传`null`并添加`Modifier.semantics { contentDescription = null }`。

2. **contentScale**:  
   图片缩放模式，常用值：
   - `ContentScale.Fit`：保持宽高比，完整显示图片（可能留空白）。
   - `ContentScale.Crop`：保持宽高比，裁剪超出部分（填满容器）。
   - `ContentScale.FillBounds`：拉伸图片至容器大小（可能变形）。
   - `ContentScale.FillWidth`/`FillHeight`：按宽度/高度填满容器，保持宽高比。
   - `ContentScale.None`：不缩放，按原图大小显示（可能超出容器）。

3. **colorFilter**:  
   为图片添加颜色滤镜，例如：
   ```kotlin
   import androidx.compose.ui.graphics.Color
   import androidx.compose.ui.graphics.ColorFilter
   import androidx.compose.ui.graphics.ColorMatrix

   Image(
       painter = painterResource(id = R.drawable.image),
       contentDescription = null,
       colorFilter = ColorFilter.tint(Color.Red) // 红色滤镜
       // 或自定义颜色矩阵（如灰度）：
       // colorFilter = ColorFilter.colorMatrix(ColorMatrix().apply { setToSaturation(0f) })
   )
   ```

4. **modifier**:  
   常用修饰符：
   - `size()`/`width()`/`height()`：设置图片大小。
   - `clip()`：裁剪图片（如圆形图片）：
     ```kotlin
     import androidx.compose.foundation.shape.CircleShape
     import androidx.compose.ui.graphics.RectangleShape
     import androidx.compose.ui.Modifier
     import androidx.compose.ui.clip

     Image(
         painter = painterResource(id = R.drawable.avatar),
         contentDescription = "头像",
         modifier = Modifier
             .size(80.dp)
             .clip(CircleShape) // 圆形裁剪
     )
     ```
   - `border()`：添加边框：
     ```kotlin
     import androidx.compose.foundation.border
     import androidx.compose.ui.graphics.Color
     import androidx.compose.ui.unit.dp

     Modifier.border(2.dp, Color.Gray, CircleShape)
     ```

### 三、高级用法
1. **圆形图片**：  
   结合`clip(CircleShape)`和`size()`实现圆形图片（如头像），示例见上文。

2. **带占位符的网络图片**：  
   Coil支持加载中/失败占位符，且可结合`rememberImagePainter()`的`builder`配置缓存、超时等：
   ```kotlin
   val painter = rememberImagePainter(
       data = imageUrl,
       builder = {
           placeholder(R.drawable.placeholder)
           error(R.drawable.error)
           fallback(R.drawable.fallback) // 数据为null时的占位图
           memoryCachePolicy(CachePolicy.ENABLED) // 启用内存缓存
           diskCachePolicy(CachePolicy.ENABLED) // 启用磁盘缓存
           timeout(30_000) // 超时时间（30秒）
       }
   )
   ```

3. **加载SVG图片**：  
   Coil支持SVG格式（需添加`coil-svg`依赖）：
   ```kotlin
   dependencies {
       implementation("io.coil-kt:coil-svg:2.5.0")
   }
   ```
   加载时自动识别SVG格式，用法与普通网络图片一致。

4. **动画图片（Gif/WebP）**：  
   Coil支持Gif和WebP动画（需添加`coil-gif`依赖）：
   ```kotlin
   dependencies {
       implementation("io.coil-kt:coil-gif:2.5.0")
   }
   ```
   加载后自动播放动画，Compose的`Image`组件会处理帧渲染。

### 四、注意事项
1. **性能优化**：  
   - 加载大图片时，建议压缩图片资源（如使用WebP格式），避免OOM。
   - 网络图片加载使用Coil的缓存策略，减少重复请求。
   - 列表中使用图片时，结合`rememberImagePainter()`（自动缓存Painter）和`LazyColumn`/`LazyRow`的复用机制，避免内存泄漏。

2. **无障碍适配**：  
   非装饰性图片必须设置`contentDescription`，确保屏幕阅读器可识别。

3. **资源命名规范**：  
   本地图片资源命名需符合Android规范（小写字母、下划线，如`ic_launcher.png`），避免中文或特殊字符。

### 总结
`Compose Image`组件的核心是`Painter`对象，不同图片源需通过对应的`Painter`实现（如`painterResource`、`BitmapPainter`、Coil的`ImagePainter`）。结合Modifier和ContentScale可灵活控制图片的显示效果，网络图片推荐使用Coil库以简化加载逻辑和处理缓存、占位符等需求。