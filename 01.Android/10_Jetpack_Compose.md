# Jetpack Compose 面试知识点

## 一、Compose 核心概念

### 1. 声明式 UI vs 命令式 UI

| 对比 | 命令式（View 体系） | 声明式（Compose） |
|------|-------------------|-----------------|
| UI 更新 | 手动调用 `setText()`、`setVisibility()` | 状态变化自动重组 |
| UI 描述 | XML + 代码 | 纯 Kotlin |
| 数据流 | 双向绑定 | 单向数据流 |
| 组件复用 | 继承 + XML include | 函数组合 |

### 2. 基本使用

```kotlin
@Composable
fun Greeting(name: String, modifier: Modifier = Modifier) {
    Column(modifier = modifier.padding(16.dp)) {
        Text(
            text = "Hello, $name!",
            style = MaterialTheme.typography.headlineMedium
        )
        Spacer(modifier = Modifier.height(8.dp))
        Button(onClick = { /* ... */ }) {
            Text("点击按钮")
        }
    }
}

// 预览
@Preview(showBackground = true)
@Composable
fun GreetingPreview() {
    Greeting("World")
}
```

---

## 二、状态管理

> **面试核心考点**：重组（Recomposition）

### 1. remember 与 mutableStateOf

```kotlin
@Composable
fun Counter() {
    // remember：在重组间保留值
    // mutableStateOf：创建可观察的状态
    var count by remember { mutableStateOf(0) }

    Column {
        Text("Count: $count")
        Button(onClick = { count++ }) {
            Text("增加")
        }
    }
}

// rememberSaveable：在配置变更后存活（类似 SavedInstanceState）
var query by rememberSaveable { mutableStateOf("") }
```

### 2. 状态提升（State Hoisting）

```kotlin
// ❌ 状态内部管理（不可测试、不可复用）
@Composable
fun BadCounter() {
    var count by remember { mutableStateOf(0) }
    Button(onClick = { count++ }) { Text("$count") }
}

// ✅ 状态提升到父组件
@Composable
fun StatelessCounter(
    count: Int,                     // 状态
    onIncrement: () -> Unit,        // 事件回调
    modifier: Modifier = Modifier
) {
    Button(onClick = onIncrement, modifier = modifier) {
        Text("Count: $count")
    }
}

@Composable
fun CounterScreen() {
    var count by remember { mutableStateOf(0) }
    StatelessCounter(count = count, onIncrement = { count++ })
}
```

### 3. 与 ViewModel 集成

```kotlin
@Composable
fun UserScreen(viewModel: UserViewModel = viewModel()) {
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()

    when (uiState) {
        is UiState.Loading -> CircularProgressIndicator()
        is UiState.Success -> UserList(users = (uiState as UiState.Success).users)
        is UiState.Error -> ErrorMessage((uiState as UiState.Error).message)
    }
}
```

---

## 三、重组（Recomposition）

> **面试高频考点**：Compose 如何知道哪些部分需要更新？

### 1. 重组触发条件

- 读取的 `State<T>` 值发生变化
- 只有**读取了该状态的 Composable** 才会被重组
- 编译器插件追踪每个 Composable 读取了哪些状态

### 2. 智能重组

```kotlin
@Composable
fun UserProfile(user: User) {
    Column {
        // 只有 user.name 变化时重组
        Text(user.name)
        // 只有 user.email 变化时重组
        Text(user.email)
        // 不依赖 user 的部分不会重组
        Text("静态文本")
    }
}
```

### 3. 稳定性标记

```kotlin
// Compose 编译器需要知道类型是否"稳定"
// 稳定类型：所有属性是 val 且类型也是稳定的

// ✅ 自动稳定
data class StableUser(val name: String, val age: Int)

// ❌ 不稳定（有 var 或集合类）
data class UnstableUser(var name: String, val tags: List<String>)

// 手动标记稳定
@Stable
class StableList(val items: List<String>)

// 或使用 @Immutable
@Immutable
data class Theme(val primaryColor: Color, val fontSize: TextUnit)
```

### 4. 性能优化

```kotlin
// 1. derivedStateOf：减少不必要的重组
val showButton by remember {
    derivedStateOf { scrollState.firstVisibleItemIndex > 0 }
}

// 2. key()：帮助 Compose 识别列表项
LazyColumn {
    items(users, key = { it.id }) { user ->
        UserItem(user)
    }
}

// 3. 使用 lambda 而非直接引用（避免重组）
// ❌ 每次重组都创建新 lambda
Button(onClick = { viewModel.submit(value) })
// ✅ 通过 rememberUpdatedState 或提升
```

---

## 四、副作用（Side Effects）

```kotlin
@Composable
fun EffectDemo() {
    // LaunchedEffect：进入组合时启动协程，key 变化时重启
    LaunchedEffect(userId) {
        viewModel.loadUser(userId)
    }

    // DisposableEffect：需要清理的副作用
    DisposableEffect(lifecycleOwner) {
        val observer = LifecycleEventObserver { _, event -> /* ... */ }
        lifecycleOwner.lifecycle.addObserver(observer)
        onDispose {
            lifecycleOwner.lifecycle.removeObserver(observer)
        }
    }

    // SideEffect：每次成功重组后执行
    SideEffect {
        analytics.trackScreen("detail")
    }

    // rememberCoroutineScope：获取与组合绑定的 CoroutineScope
    val scope = rememberCoroutineScope()
    Button(onClick = {
        scope.launch { viewModel.submit() }
    }) { Text("提交") }
}
```

| 副作用 API | 时机 | 清理 | 适用场景 |
|-----------|------|------|---------|
| `LaunchedEffect` | 进入/key变化 | 自动取消 | 加载数据、动画 |
| `DisposableEffect` | 进入/key变化 | `onDispose` | 监听器注册 |
| `SideEffect` | 每次重组 | 无 | 分析埋点 |
| `rememberCoroutineScope` | 整个生命周期 | 自动 | 事件触发的协程 |

---

## 五、Compose 与 View 互操作

```kotlin
// 在 Compose 中使用 View
@Composable
fun MapView() {
    AndroidView(
        factory = { context ->
            MapView(context).apply { onCreate(null) }
        },
        update = { mapView ->
            mapView.getMapAsync { /* 更新地图 */ }
        }
    )
}

// 在 View 中使用 Compose
class MyActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContent {
            MaterialTheme {
                MyComposeScreen()
            }
        }
    }
}

// XML 中嵌入 ComposeView
val composeView = findViewById<ComposeView>(R.id.compose_view)
composeView.setContent {
    MyComposable()
}
```

---

## 六、Navigation Compose

```kotlin
@Composable
fun NavGraph() {
    val navController = rememberNavController()

    NavHost(navController = navController, startDestination = "home") {
        composable("home") {
            HomeScreen(onNavigateToDetail = { id ->
                navController.navigate("detail/$id")
            })
        }
        composable(
            route = "detail/{userId}",
            arguments = listOf(navArgument("userId") { type = NavType.LongType })
        ) { backStackEntry ->
            val userId = backStackEntry.arguments?.getLong("userId") ?: 0
            DetailScreen(userId = userId)
        }
    }
}
```
