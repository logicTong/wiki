在Jetpack Compose中，输入框主要通过`TextField`（基础款）和`OutlinedTextField`（带轮廓款）来实现，它们都是Material Design风格的输入组件，支持输入监听、状态管理、占位符、提示文本等常用功能。以下是详细的使用示例和关键知识点：

### 1. 基础输入框（TextField）
首先需要使用`remember`和`mutableStateOf`管理输入状态（Compose的状态驱动UI核心），然后将状态绑定到`TextField`的`value`和`onValueChange`参数。

```kotlin
import androidx.compose.foundation.layout.Column
import androidx.compose.foundation.layout.padding
import androidx.compose.material3.OutlinedTextField
import androidx.compose.material3.Text
import androidx.compose.material3.TextField
import androidx.compose.runtime.Composable
import androidx.compose.runtime.getValue
import androidx.compose.runtime.mutableStateOf
import androidx.compose.runtime.remember
import androidx.compose.runtime.setValue
import androidx.compose.ui.Modifier
import androidx.compose.ui.unit.dp

@Composable
fun BasicTextFieldExample() {
    // 管理输入状态：remember用于保存重组后的状态，mutableStateOf实现状态可观察
    var inputText by remember { mutableStateOf("") }

    Column(modifier = Modifier.padding(16.dp)) {
        // 基础TextField（填充式）
        TextField(
            value = inputText,
            onValueChange = { inputText = it }, // 输入变化时更新状态
            label = { Text("基础输入框") }, // 标签（点击时浮动）
            placeholder = { Text("请输入内容") }, // 占位符（无输入时显示）
            modifier = Modifier.padding(bottom = 8.dp)
        )

        // 显示输入内容
        Text(text = "你输入的内容：$inputText")
    }
}
```

### 2. 带轮廓的输入框（OutlinedTextField）
`OutlinedTextField`是Material Design 3推荐的样式，视觉上更轻量化，用法和`TextField`几乎一致：

```kotlin
@Composable
fun OutlinedTextFieldExample() {
    var username by remember { mutableStateOf("") }
    var password by remember { mutableStateOf("") }

    Column(modifier = Modifier.padding(16.dp)) {
        // 用户名输入框
        OutlinedTextField(
            value = username,
            onValueChange = { username = it },
            label = { Text("用户名") },
            placeholder = { Text("请输入用户名") },
            modifier = Modifier.padding(bottom = 8.dp)
        )

        // 密码输入框（隐藏输入内容）
        OutlinedTextField(
            value = password,
            onValueChange = { password = it },
            label = { Text("密码") },
            placeholder = { Text("请输入密码") },
            visualTransformation = androidx.compose.ui.text.input.PasswordVisualTransformation(), // 密码隐藏
            keyboardOptions = androidx.compose.foundation.text.KeyboardOptions(
                keyboardType = androidx.compose.ui.text.input.KeyboardType.Password // 键盘类型为密码
            )
        )
    }
}
```

### 3. 关键参数说明
- **value**: 输入框的当前文本内容（必须绑定状态）。
- **onValueChange**: 文本变化时的回调，用于更新状态（核心：单向数据流）。
- **label**: 浮动标签（`Text`组件），点击输入框时会浮动到输入框上方，Material Design特色。
- **placeholder**: 占位符文本（无输入时显示，有输入时隐藏）。
- **visualTransformation**: 文本视觉转换，如密码隐藏（`PasswordVisualTransformation()`）、数字格式化等。
- **keyboardOptions**: 键盘配置，如键盘类型（`KeyboardType.Text`/`Number`/`Email`/`Password`等）、回车键图标（`imeAction`）。
- **keyboardActions**: 键盘动作回调，如监听回车键点击：
  ```kotlin
  keyboardActions = KeyboardActions(
      onDone = { /* 回车键（Done）点击时的逻辑 */ }
  )
  ```
- **enabled**: 是否禁用输入框（`Boolean`，禁用后无法输入，样式变灰）。
- **readOnly**: 是否只读（`Boolean`，只读时可选中复制，但无法编辑，样式正常）。
- **singleLine**: 是否单行输入（`Boolean`，默认`false`，多行时自动换行，单行时输入超出后横向滚动）。
- **maxLines/minLines**: 最大/最小行数（配合`singleLine = false`使用）。
- **isError**: 是否标记为错误状态（`Boolean`，为`true`时输入框边框/标签变红，通常配合`supportingText`显示错误提示）：
  ```kotlin
  var inputText by remember { mutableStateOf("") }
  val isError = inputText.length < 6

  OutlinedTextField(
      value = inputText,
      onValueChange = { inputText = it },
      label = { Text("验证码") },
      isError = isError,
      supportingText = {
          if (isError) Text("验证码至少6位") else Text("请输入验证码")
      }
  )
  ```

### 4. 自定义输入框样式
Compose支持通过`TextFieldDefaults`自定义输入框的颜色、形状、字体等样式，例如自定义轮廓输入框的颜色：

```kotlin
import androidx.compose.material3.MaterialTheme
import androidx.compose.material3.TextFieldDefaults

@Composable
fun CustomStyleTextField() {
    var inputText by remember { mutableStateOf("") }

    OutlinedTextField(
        value = inputText,
        onValueChange = { inputText = it },
        label = { Text("自定义样式输入框") },
        colors = TextFieldDefaults.outlinedTextFieldColors(
            focusedBorderColor = MaterialTheme.colorScheme.primary, // 聚焦时边框颜色
            unfocusedBorderColor = MaterialTheme.colorScheme.onSurface.copy(alpha = 0.6f), // 未聚焦时边框颜色
            focusedLabelColor = MaterialTheme.colorScheme.primary, // 聚焦时标签颜色
            cursorColor = MaterialTheme.colorScheme.primary // 光标颜色
        ),
        shape = MaterialTheme.shapes.medium // 输入框形状（圆角）
    )
}
```

### 5. 原生输入框（BasicTextField）
如果需要更高度的自定义（如完全去掉Material Design的样式、自定义绘制），可以使用`BasicTextField`，它是`TextField`和`OutlinedTextField`的底层实现，仅提供基础输入功能，样式需要完全自定义：

```kotlin
import androidx.compose.foundation.background
import androidx.compose.foundation.border
import androidx.compose.foundation.layout.padding
import androidx.compose.foundation.text.BasicTextField
import androidx.compose.material3.Text
import androidx.compose.ui.graphics.Color
import androidx.compose.ui.unit.dp

@Composable
fun BasicTextFieldCustom() {
    var inputText by remember { mutableStateOf("") }

    BasicTextField(
        value = inputText,
        onValueChange = { inputText = it },
        decorationBox = { innerTextField ->
            // 自定义装饰（标签、占位符等）
            Column {
                Text("原生输入框")
                if (inputText.isEmpty()) {
                    Text("请输入内容", color = Color.Gray)
                }
                innerTextField() // 必须调用，显示输入区域
            }
        },
        modifier = Modifier
            .padding(16.dp)
            .background(Color.White)
            .border(1.dp, Color.Gray)
            .padding(8.dp)
    )
}
```

### 核心要点
- **状态管理**：Compose的输入框必须绑定可观察状态（`mutableStateOf`/`StateFlow`/`LiveData`等），遵循“状态驱动UI”的原则。
- **Material Design 3**：Compose 1.2+推荐使用Material 3的`TextField`/`OutlinedTextField`（包路径`androidx.compose.material3`），旧版Material 2的组件在`androidx.compose.material`包下，功能类似但样式不同。
- **键盘交互**：通过`keyboardOptions`和`keyboardActions`处理键盘类型和按键事件，提升用户体验。
- **错误处理**：使用`isError`和`supportingText`配合实现友好的错误提示，符合Material Design规范。

以上是Compose输入框的常用用法和扩展，可根据实际需求组合参数或自定义样式。