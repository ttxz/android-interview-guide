# Flutter 与原生混编实战

## 一、为什么需要混编？

### 1. 真实业务场景

```text
场景一：老项目渐进式迁移
  你有一个已经开发了 3 年的 Android 原生项目，老板说"Flutter 好，以后新页面都用 Flutter 写"。
  → 不可能重写整个 App，只能让 Flutter 和原生共存。

场景二：部分页面需要跨平台
  一个电商 App，商品详情页需要 iOS 和 Android 保持一致。
  → 商品详情用 Flutter，其他页面保持原生。

场景三：动态化需求
  运营页面需要快速迭代，不想每次都发版。
  → 用 Flutter 实现可热更新的运营页面，嵌入到原生 App 中。
```

### 2. 三种技术路线对比

| 路线 | 适合场景 | 代表做法 |
|------|---------|---------|
| **纯原生** | 对性能要求极高、已有成熟团队 | 传统 Android/iOS 开发 |
| **纯 Flutter** | 全新项目、小团队 | `flutter create` 从零开始 |
| **混编（Add-to-App）** | 老项目迁移、部分页面跨平台 | Flutter Module 嵌入原生工程 |

> **面试点**：大部分公司都是"混编"场景，因为很少有项目能全部推翻重来。

---

## 二、核心概念：先搞懂这些再看代码 ⭐

> 这一章是理解后面所有代码的基础。如果你直接看 API 觉得一头雾水，请先读这里。

### 1. FlutterEngine 是什么？

**一句话**：`FlutterEngine` 就是 `Flutter` 的"发动机"，负责运行 Dart 代码和渲染 UI。

**类比理解**：

```text
你已经熟悉的 WebView：
  WebView = 一个展示网页的容器
  WebView 内部有一个浏览器引擎（如 Chromium），负责解析 HTML/JS、渲染网页

Flutter 混编也是类似的：
  FlutterFragment/FlutterActivity = 一个展示 Flutter 页面的容器
  FlutterEngine = 内部的引擎，负责运行 Dart 代码、渲染 Flutter UI

┌───────────────────────────────┐
│ FlutterFragment（容器/壳子）    │ ← 相当于 WebView
│  ┌───────────────────────┐    │
│  │ FlutterEngine（引擎）   │    │ ← 相当于 Chromium 内核
│  │  ├── Dart VM          │    │    运行你的 Dart 代码
│  │  ├── Skia 渲染引擎     │    │    绘制 Flutter UI
│  │  └── Platform Channel │    │    与原生通信
│  └───────────────────────┘    │
└───────────────────────────────┘
```

**关键理解**：

- 引擎和容器是**分离**的。一个引擎可以被不同的容器使用
- 创建引擎是**重操作**（约 40-60MB 内存，1-3 秒初始化时间）
- 所以才有了「预热」的概念——提前创建好引擎

### 2. 为什么要"预热"引擎？

**类比**：冬天开车，你可以：

- **不预热**：上车直接开，发动机冷启动，前几分钟动力不足（体验差）
- **预热**：提前远程启动车辆，上车就能直接走（体验好，但费油/费电）

```text
Flutter 不预热：
  用户点击 → 创建 FlutterEngine（1-3秒）→ 运行 Dart 代码 → 渲染 UI → 用户看到页面
  体验：白屏/黑屏 1-3 秒 ❌

Flutter 预热：
  App 启动时 → 提前创建 FlutterEngine → 放到缓存里
  用户点击 → 从缓存取引擎 → 立即渲染 UI → 用户看到页面
  体验：几乎秒开 ✅

代价：预热引擎常驻内存，增加约 40-60MB 内存占用
```

### 3. FlutterFragment vs FlutterActivity：容器的两种形态

**类比**：你已经知道 Fragment 和 Activity 的区别吧？

```text
FlutterActivity = 一个完整的 Activity，整个页面都是 Flutter
  → 类比：打开一个新的全屏网页

FlutterFragment = 一个 Fragment，可以嵌入到任何 Activity 的布局中
  → 类比：在原生页面里嵌入一个 WebView

什么时候用哪个？
  ┌──────────────────────────────────────────┐
  │ 整页都是 Flutter 内容？                    │
  │  ├── 是 → FlutterActivity（更简单）       │
  │  └── 否 → FlutterFragment（更灵活）⭐     │
  │          适合：Tab 页签中嵌入、半屏显示     │
  └──────────────────────────────────────────┘
```

### 4. RenderMode：surface vs texture（为什么要关心渲染模式？）

**类比**：你可能用过 SurfaceView 和 TextureView 来播放视频吧？

```text
SurfaceView（对应 RenderMode.surface）：
  - 有自己独立的绘制层，不在 View 层级体系中
  - 优点：性能最好（GPU 直接合成）
  - 缺点：不支持透明、不支持 View 动画、在 ViewPager 中会闪烁
  - 就像一扇「玻璃窗」直接嵌在墙上，看得最清楚，但没法移动

TextureView（对应 RenderMode.texture）：
  - 渲染到纹理后当作普通 View 显示
  - 优点：支持透明、支持动画和变换、ViewPager 中正常
  - 缺点：多一次 GPU→CPU→GPU 的拷贝，性能稍差（约 5-10%）
  - 就像先拍张「照片」再贴到墙上，可以随意移动旋转

选择口诀：
  ┌───────────────────────────────┐
  │ 需要透明背景？→ 必须 texture   │
  │ 在 ViewPager 中？→ 推荐 texture │
  │ 全屏展示？→ 用 surface 性能好  │
  └───────────────────────────────┘
```

### 5. Platform Channel：原生和 Flutter 怎么"说话"？

**类比**：如果你用过 WebView 的 JavaScriptInterface（JSBridge），那 Platform Channel 就是一样的东西。

```text
JSBridge（你已知的）：
  Android ←→ JavaScript
  通过 @JavascriptInterface 注解暴露方法给 JS 调用

Platform Channel（Flutter 版的 JSBridge）：
  Android (Kotlin) ←→ Flutter (Dart)
  通过 MethodChannel 暴露方法给对方调用

对比：
  ┌────────────────────┬───────────────────────────────┐
  │ JSBridge           │ Platform Channel              │
  ├────────────────────┼───────────────────────────────┤
  │ webView.addJSI...  │ MethodChannel("通道名")        │
  │ @JavascriptIntf    │ setMethodCallHandler          │
  │ evaluateJavascript │ invokeMethod                  │
  │ 只有 MethodChannel │ 还有 EventChannel（持续推送）   │
  └────────────────────┴───────────────────────────────┘
```

### 6. 透明背景：为什么需要特殊配置？

```text
问题本质：
  Flutter 默认会创建一个"不透明的画布"来渲染 UI
  就像你买了一幅油画，画布本身是白色/黑色的，后面的墙壁被挡住了

  但如果你想做「弹窗」效果：
    → 需要画布是"透明"的
    → 只有弹窗部分有内容，其他区域能看到底层原生页面

  这需要三个层面同时配置：
  ┌──────────────────────────────────────────────┐
  │ 第 1 层：Android 窗口要透明                    │
  │   → styles.xml: windowIsTranslucent=true      │
  │   → windowBackground=transparent              │
  │                                                │
  │ 第 2 层：Flutter 容器要透明                     │
  │   → BackgroundMode.transparent（Activity 用）   │
  │   → TransparencyMode.transparent（Fragment 用） │
  │                                                │
  │ 第 3 层：渲染模式要支持透明通道                  │
  │   → RenderMode.texture（非 surface！）          │
  │                                                │
  │ 缺任何一层 = 不透明 / 黑屏 / 白屏              │
  └──────────────────────────────────────────────┘
```

---

## 三、从零开始：跑通第一个混编 Demo

> 按照这个顺序学习：先最简单的 → 再加缓存引擎 → 再加透明弹窗

### Step 1：创建 Flutter Module

```bash
# 在你的 Android 项目同级目录创建 Flutter Module
cd your_project_parent_dir
flutter create --template module flutter_module
```

```text
目录结构：
your_project_parent_dir/
├── android_app/          ← 你的原生 Android 项目
└── flutter_module/       ← 新建的 Flutter Module
    ├── lib/
    │   └── main.dart     ← Flutter 代码入口
    └── .android/         ← 自动生成的 Android 壳工程
```

### Step 2：原生项目引入 Flutter Module

```groovy
// android_app/settings.gradle
// 添加以下配置，告诉 Gradle 去哪里找 Flutter Module
setBinding(new Binding([gradle: this]))
evaluate(new File(
    settingsDir.parentFile,
    'flutter_module/.android/include_flutter.groovy'
))
```

```groovy
// android_app/app/build.gradle
dependencies {
    // 添加 Flutter 模块依赖
    implementation project(':flutter')
}
```

### Step 3：最简单的方式——FlutterActivity（5 行代码搞定）

```kotlin
// 这是最简单的混编方式：直接启动一个全屏 Flutter 页面
// 不需要预热引擎，不需要配置布局，5 行代码搞定

// 在你想跳转到 Flutter 的地方：
button.setOnClickListener {
    startActivity(
        FlutterActivity
            .withNewEngine()           // 创建一个新引擎（首次会慢 1-3 秒）
            .initialRoute("/home")     // 告诉 Flutter 打开哪个页面
            .build(this)               // 生成 Intent
    )
}
```

```xml
<!-- 别忘了在 AndroidManifest.xml 注册！ -->
<activity
    android:name="io.flutter.embedding.android.FlutterActivity"
    android:configChanges="orientation|keyboardHidden|keyboard|screenSize|locale|layoutDirection|fontScale|screenLayout|density|uiMode"
    android:hardwareAccelerated="true"
    android:windowSoftInputMode="adjustResize" />
```

```dart
// flutter_module/lib/main.dart
// Flutter 端：根据路由显示不同页面
import 'package:flutter/material.dart';

void main() => runApp(const MyApp());

class MyApp extends StatelessWidget {
  const MyApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      // 接收原生传过来的 initialRoute
      initialRoute: '/',
      routes: {
        '/': (context) => const HomePage(),
        '/home': (context) => const HomePage(),
      },
    );
  }
}

class HomePage extends StatelessWidget {
  const HomePage({super.key});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Flutter 页面')),
      body: const Center(
        child: Text('👋 这是 Flutter 渲染的页面！', style: TextStyle(fontSize: 24)),
      ),
    );
  }
}
```

> **运行效果**：点击按钮 → 白屏 1-3 秒 → Flutter 页面出现。体验不好？继续看 Step 4！

### Step 4：加上引擎预热——实现秒开

```kotlin
// ①　在 Application 中提前预热引擎
class MyApplication : Application() {
    override fun onCreate() {
        super.onCreate()

        // 创建引擎（这行代码会分配 Dart VM、加载 Flutter 资源）
        val engine = FlutterEngine(this)

        // 设置初始路由（必须在下一步之前！顺序很重要！）
        engine.navigationChannel.setInitialRoute("/home")

        // 启动 Dart 代码执行（引擎开始"预热"）
        engine.dartExecutor.executeDartEntrypoint(
            DartExecutor.DartEntrypoint.createDefault()
            // createDefault() = 执行 Dart 的 main() 函数
        )

        // 把预热好的引擎存到全局缓存中，起个名字叫 "my_engine"
        FlutterEngineCache.getInstance().put("my_engine", engine)
    }
}
```

```kotlin
// ②　使用缓存引擎打开 Flutter 页面
button.setOnClickListener {
    startActivity(
        FlutterActivity
            .withCachedEngine("my_engine")  // 从缓存取引擎（不再创建新引擎）
            // 注意：这里不能再调 .initialRoute() 了！
            // 因为引擎已经在 Application 中启动了，路由已经确定
            .build(this)
    )
}
```

> **运行效果**：点击按钮 → 立即显示 Flutter 页面，几乎无延迟！✅

### Step 5：用 FlutterFragment 嵌入原生页面

```xml
<!-- activity_host.xml -->
<!-- 在原生布局中预留一个容器给 FlutterFragment -->
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical">

    <!-- 原生内容 -->
    <TextView
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:text="👆 这是原生 TextView"
        android:padding="16dp"
        android:textSize="18sp" />

    <!-- Flutter 内容放在这里 -->
    <FrameLayout
        android:id="@+id/flutter_container"
        android:layout_width="match_parent"
        android:layout_height="0dp"
        android:layout_weight="1" />

    <!-- 原生内容 -->
    <TextView
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:text="👇 这也是原生 TextView"
        android:padding="16dp"
        android:textSize="18sp" />
</LinearLayout>
```

```kotlin
// HostActivity.kt
class HostActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_host)

        // 创建 FlutterFragment，使用缓存引擎
        val flutterFragment = FlutterFragment
            .withCachedEngine("my_engine")     // 用预热好的引擎
            .renderMode(RenderMode.surface)    // 全屏不透明，用 surface 性能最好
            .build<FlutterFragment>()          // 构建 Fragment 实例

        // 放到容器里（跟普通 Fragment 一样）
        supportFragmentManager.commit {
            replace(R.id.flutter_container, flutterFragment)
        }
    }
}
```

> **运行效果**：上面是原生 TextView，中间是 Flutter 页面，下面又是原生 TextView——混在一起了！

---

## 四、引擎管理进阶

### 1. 单引擎 vs 多引擎：什么时候需要多个引擎？

```text
单引擎（一个 FlutterEngine 供所有 Flutter 页面用）：
  ✅ 内存占用少（只有一份 ~50MB）
  ✅ 页面间天然共享数据
  ❌ 一个引擎同一时刻只能绑定一个 FlutterActivity/Fragment
  ❌ 多个 Flutter 页面需要靠 Flutter 内部路由管理，复杂

多引擎（每个 Flutter 页面一个独立引擎）：
  ✅ 每个页面完全独立，不互相影响
  ❌ 每个引擎 ~50MB，3个就是 150MB！太贵了

FlutterEngineGroup（折中方案 ⭐ 推荐）：
  ✅ 第一个引擎 ~50MB，后续每个只增加 ~180KB！
  ✅ 共享 Dart VM snapshot 和 GPU 上下文
  ✅ 每个引擎独立路由
  ✅ 这是 Google 官方推荐的多引擎方案
```

### 2. FlutterEngineGroup 完整示例

```kotlin
// Application 中创建 EngineGroup
class MyApplication : Application() {
    // EngineGroup 是引擎的"工厂"，负责创建共享资源的引擎
    lateinit var engineGroup: FlutterEngineGroup

    override fun onCreate() {
        super.onCreate()
        // 只需要创建一次，放在 Application 中
        engineGroup = FlutterEngineGroup(this)
    }

    /**
     * 创建一个新引擎
     * @param entrypoint Dart 入口函数名，默认是 "main"
     * @param initialRoute 初始路由
     *
     * 首次调用：约 50MB 内存，有初始化延迟
     * 后续调用：仅 ~180KB 增量，几乎秒创建！
     */
    fun createEngine(
        entrypoint: String = "main",
        initialRoute: String = "/"
    ): FlutterEngine {
        val dartEntrypoint = DartExecutor.DartEntrypoint(
            // findAppBundlePath() 返回 Flutter 编译产物的路径
            // 在 release 模式下是 libapp.so（AOT 编译的机器码）
            FlutterInjector.instance().flutterLoader().findAppBundlePath(),
            entrypoint
        )
        return engineGroup.createAndRunEngine(
            this, dartEntrypoint, initialRoute
        )
    }
}
```

```dart
// Flutter 端：定义多个入口点
// 默认入口（main 函数）
void main() => runApp(const MainApp());

// 自定义入口点 —— 弹窗页面
// @pragma('vm:entry-point') 这个注解非常重要！
// 它告诉 Dart 编译器："不要优化掉这个函数！"
// 因为 Dart AOT 编译器会做 tree-shaking（移除未使用的代码），
// 如果不加注解，编译器看到 dialogEntry 没被代码调用，就会删掉它
// 运行时原生代码调用这个函数就会崩溃
@pragma('vm:entry-point')
void dialogEntry() => runApp(const DialogApp());

// 自定义入口点 —— 设置页面
@pragma('vm:entry-point')
void settingsEntry() => runApp(const SettingsApp());
```

### 3. 引擎方案选择决策

```text
你的 App 中有几个 Flutter 页面？

  只有 1 个 Flutter 页面
    → 预热单引擎（FlutterEngineCache）
    → 最简单，内存开销固定 ~50MB

  有 2-3 个 Flutter 页面
    → FlutterEngineGroup
    → 首个 ~50MB + 后续每个 ~180KB = 约 50.5MB

  有很多 Flutter 页面 + 复杂的混合跳转
    → FlutterEngineGroup + flutter_boost
    → EngineGroup 管引擎，flutter_boost 管路由栈

  内存特别敏感（低端机）
    → 按需创建引擎 + 用完立即 destroy()
    → 牺牲打开速度换内存
```

---

## 五、透明背景与弹窗实战 ⭐ 面试高频

### 1. 需求还原

```text
产品经理："我们的确认弹窗和底部弹窗想用 Flutter 统一实现，
         原生页面点击按钮后弹出来，背景要半透明蒙层。"

技术难点：
  Flutter 默认渲染一个不透明的全屏画布。
  你需要让这个"画布"变透明，只显示弹窗部分。
```

### 2. 方案一：透明 FlutterActivity ⭐⭐ 最常用

**原理**：启动一个透明的 Activity，Flutter 在上面绘制弹窗 + 半透明背景。

**第一步：配置透明主题**

```xml
<!-- res/values/styles.xml -->
<style name="FlutterTransparentTheme" parent="Theme.AppCompat.NoActionBar">
    <!-- 窗口背景设为透明（不然会是白色/黑色底色） -->
    <item name="android:windowBackground">@android:color/transparent</item>

    <!-- 允许窗口半透明（这样才能看到底层 Activity） -->
    <item name="android:windowIsTranslucent">true</item>

    <!-- 去掉标题栏 -->
    <item name="android:windowNoTitle">true</item>

    <!-- 底层 Activity 上加暗色蒙层（可选，设 false 则无蒙层） -->
    <item name="android:backgroundDimEnabled">true</item>

    <!-- 蒙层透明度 0.0~1.0（越大越暗，默认 0.6） -->
    <item name="android:backgroundDimAmount">0.6</item>

    <!-- 重要！不要设为 true！否则窗口尺寸变成 wrap_content -->
    <!-- Flutter 会因为拿不到正确的屏幕尺寸而显示异常 -->
    <item name="android:windowIsFloating">false</item>
</style>
```

```xml
<!-- AndroidManifest.xml 注册 -->
<activity
    android:name=".FlutterDialogActivity"
    android:theme="@style/FlutterTransparentTheme" />
```

**第二步：创建透明 FlutterActivity**

```kotlin
class FlutterDialogActivity : FlutterActivity() {
    /**
     * 关键方法！告诉 Flutter 引擎：渲染背景要透明
     *
     * BackgroundMode.opaque（默认）→ 不透明，白色/黑色背景
     * BackgroundMode.transparent → 透明，能看到底层 Activity
     *
     * 注意：transparent 模式下会自动使用 TextureView（RenderMode.texture），
     * 因为 SurfaceView 不支持透明通道
     */
    override fun getBackgroundMode(): BackgroundMode {
        return BackgroundMode.transparent
    }

    companion object {
        fun start(context: Context) {
            context.startActivity(
                withCachedEngine("dialog_engine")
                    .backgroundMode(BackgroundMode.transparent)
                    .build(context)
            )
        }
    }
}
```

**第三步：Flutter 端弹窗页面**

```dart
class DialogPage extends StatelessWidget {
  const DialogPage({super.key});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      // 关键！Scaffold 背景设为半透明黑色
      // 这就是用户看到的"蒙层"效果
      backgroundColor: Colors.black54,
      body: Center(
        child: Container(
          width: 300,
          padding: const EdgeInsets.all(24),
          decoration: BoxDecoration(
            color: Colors.white,
            borderRadius: BorderRadius.circular(16),
          ),
          child: Column(
            mainAxisSize: MainAxisSize.min, // 弹窗大小由内容决定
            children: [
              const Text('确认操作', style: TextStyle(fontSize: 20, fontWeight: FontWeight.bold)),
              const SizedBox(height: 12),
              const Text('确定要删除这条记录吗？'),
              const SizedBox(height: 24),
              Row(
                mainAxisAlignment: MainAxisAlignment.spaceEvenly,
                children: [
                  TextButton(
                    // SystemNavigator.pop() 在混编中 = Activity.finish()
                    onPressed: () => SystemNavigator.pop(),
                    child: const Text('取消'),
                  ),
                  ElevatedButton(
                    onPressed: () {
                      // 通过 MethodChannel 把结果传回原生
                      // 然后关闭自己
                      SystemNavigator.pop();
                    },
                    child: const Text('确定'),
                  ),
                ],
              ),
            ],
          ),
        ),
      ),
    );
  }
}
```

### 3. 方案二：FlutterFragment + DialogFragment

**适用于**：Flutter 页面已经在运行，想在它上面弹窗，或者想用原生 Dialog 的容器。

```kotlin
class FlutterDialogFragment : DialogFragment() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        // STYLE_NO_FRAME 去掉 Dialog 自带的边框和背景
        setStyle(STYLE_NO_FRAME, R.style.FlutterTransparentTheme)
    }

    override fun onCreateView(
        inflater: LayoutInflater, container: ViewGroup?, savedInstanceState: Bundle?
    ): View {
        return FrameLayout(requireContext()).apply {
            id = View.generateViewId()
            post {
                val flutterFragment = FlutterFragment
                    .withCachedEngine("dialog_engine")
                    // transparencyMode = FlutterFragment 版的 BackgroundMode
                    // 作用一样：让 Flutter 渲染层变透明
                    .transparencyMode(TransparencyMode.transparent)
                    // 透明必须用 texture！surface 不支持透明通道
                    .renderMode(RenderMode.texture)
                    .build<FlutterFragment>()

                childFragmentManager.commit {
                    replace(this@apply.id, flutterFragment)
                }
            }
        }
    }
}

// 使用（跟原生 DialogFragment 一样）
FlutterDialogFragment().show(supportFragmentManager, "flutter_dialog")
```

### 4. 方案三：MethodChannel 回调（零成本方案）

**前提**：Flutter 页面已经在运行（比如首页中的某个 Tab 是 FlutterFragment）。

```kotlin
// 原生端：发消息让 Flutter 弹窗
channel.invokeMethod("showDialog", mapOf(
    "title" to "提示",
    "content" to "确认删除？"
), object : MethodChannel.Result {
    override fun success(result: Any?) {
        // result 是 Flutter 弹窗的返回值（true=确认，false=取消）
        if (result == true) deleteRecord()
    }
    override fun error(code: String, msg: String?, details: Any?) {}
    override fun notImplemented() {}
})
```

```dart
// Flutter 端：收到消息后弹窗
channel.setMethodCallHandler((call) async {
  if (call.method == 'showDialog') {
    final args = call.arguments as Map;
    // 用 Flutter 自带的 showDialog 弹出
    final result = await showDialog<bool>(
      context: navigatorKey.currentContext!,
      builder: (context) => AlertDialog(
        title: Text(args['title']),
        content: Text(args['content']),
        actions: [
          TextButton(
            onPressed: () => Navigator.pop(context, false),
            child: const Text('取消'),
          ),
          TextButton(
            onPressed: () => Navigator.pop(context, true),
            child: const Text('确定'),
          ),
        ],
      ),
    );
    return result; // 返回值自动传回原生的 Result.success()
  }
});
```

### 5. 三种弹窗方案怎么选？

```text
问自己两个问题：

Q1：当前页面有 Flutter 在运行吗？
  ├── 有 → 方案三（MethodChannel 回调）→ 零成本！
  └── 没有 ↓

Q2：弹窗是独立的还是嵌入到原生 Dialog 的？
  ├── 独立弹窗 → 方案一（透明 FlutterActivity）⭐ 最推荐
  └── 嵌入原生 Dialog → 方案二（FlutterFragment + DialogFragment）
```

| 维度 | 透明 Activity | Fragment + Dialog | MethodChannel |
|------|--------------|-------------------|---------------|
| **前提** | 无 | 无 | Flutter 页面已在运行 |
| **内存** | 新 Activity | 较低 | 零开销 |
| **复杂度** | ⭐⭐ 简单 | ⭐⭐⭐ 中等 | ⭐⭐ 简单 |
| **推荐度** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐ |

---

## 六、主题适配：让 Flutter 和原生"看起来一样"

### 1. 问题是什么？

```text
你打开一个混编 App：
  → 首页是原生的，用了 Material Design 3 主题，主色调深蓝
  → 点击某个 Tab 进入 Flutter 页面，默认 Flutter 主题，主色调紫色
  → 用户感觉："怎么换了个 App？"

需要解决：
  1. 主色调统一
  2. 暗黑模式同步
  3. 状态栏/导航栏颜色一致
```

### 2. 解决方案：通过 MethodChannel 传递主题

```kotlin
// Android 端：把原生主题信息传给 Flutter
class ThemeBridge(engine: FlutterEngine) {
    private val channel = MethodChannel(
        engine.dartExecutor.binaryMessenger,
        "com.example/theme"
    )

    fun syncTheme(isDarkMode: Boolean, primaryColor: Int) {
        channel.invokeMethod("updateTheme", mapOf(
            "isDarkMode" to isDarkMode,
            "primaryColor" to String.format("#%06X", 0xFFFFFF and primaryColor)
        ))
    }
}

// 在引擎预热后调用
val bridge = ThemeBridge(flutterEngine)
bridge.syncTheme(
    isDarkMode = isNightMode(),
    primaryColor = getThemeColor(R.attr.colorPrimary)
)
```

```dart
// Flutter 端：接收主题并应用
class ThemeManager {
  static const _channel = MethodChannel('com.example/theme');
  static final themeNotifier = ValueNotifier<ThemeData>(ThemeData.light());

  static void init() {
    _channel.setMethodCallHandler((call) async {
      if (call.method == 'updateTheme') {
        final args = call.arguments as Map;
        final isDark = args['isDarkMode'] as bool;
        final color = Color(
          int.parse(args['primaryColor'].substring(1), radix: 16) + 0xFF000000
        );
        themeNotifier.value = isDark
            ? ThemeData.dark().copyWith(colorScheme: ColorScheme.dark(primary: color))
            : ThemeData.light().copyWith(colorScheme: ColorScheme.light(primary: color));
      }
    });
  }
}

// 在 MaterialApp 中使用
ValueListenableBuilder<ThemeData>(
  valueListenable: ThemeManager.themeNotifier,
  builder: (context, theme, _) => MaterialApp(theme: theme, home: HomePage()),
)
```

### 3. 状态栏颜色冲突怎么解决？

```text
问题：Flutter 页面和原生页面的状态栏样式不一致
  例如：原生页面状态栏透明 + 白色图标
       Flutter 页面状态栏变成了蓝色 + 黑色图标

原因：Flutter 的 AppBar 会自动设置状态栏样式，覆盖了原生的设置

解决：在 Flutter 端统一控制
```

```dart
// Flutter 端：统一状态栏样式
SystemChrome.setSystemUIOverlayStyle(const SystemUiOverlayStyle(
  statusBarColor: Colors.transparent,        // 状态栏透明
  statusBarIconBrightness: Brightness.dark,  // 图标颜色（dark=黑色）
  systemNavigationBarColor: Colors.white,    // 底部导航栏
));
```

---

## 七、混合栈管理

### 1. 问题是什么？

```text
常见跳转场景：
  原生页面 A → Flutter 页面 B → 原生页面 C → Flutter 页面 D

用户按返回键：
  D ← C ← B ← A（期望）

实际可能遇到的问题：
  ❌ 在 Flutter 页面 B 按返回，直接退出了整个 App
  ❌ Flutter 页面内的路由跳转和原生页面栈混乱
  ❌ startActivityForResult 从 Flutter 页面拿不到结果
```

### 2. 简单场景：自己管理

如果混编场景简单（只有 1-2 个 Flutter 页面），自己管就够了：

```kotlin
// 统一路由管理
object HybridRouter {
    private lateinit var channel: MethodChannel

    fun init(engine: FlutterEngine) {
        channel = MethodChannel(
            engine.dartExecutor.binaryMessenger, "hybrid_router"
        )
        // 监听 Flutter 请求打开原生页面
        channel.setMethodCallHandler { call, result ->
            when (call.method) {
                "openNativePage" -> {
                    val route = call.argument<String>("route")!!
                    // 根据路由打开对应的原生页面
                    when (route) {
                        "/native/settings" -> openSettings()
                        "/native/profile" -> openProfile()
                    }
                    result.success(null)
                }
                "closePage" -> {
                    // 关闭当前 Flutter 页面（实际是 finish 宿主 Activity）
                    currentActivity?.finish()
                    result.success(null)
                }
                else -> result.notImplemented()
            }
        }
    }

    // 原生打开 Flutter 页面
    fun openFlutter(context: Context, route: String) {
        // 通过 MethodChannel 通知 Flutter 跳转路由
        channel.invokeMethod("navigate", route)
    }
}
```

### 3. 复杂场景：用 flutter_boost

如果有频繁的原生 ↔ Flutter 互跳，推荐用闲鱼的 flutter_boost：

```text
flutter_boost 解决了什么？
  ✅ 统一管理原生和 Flutter 的页面栈
  ✅ 返回键正确处理
  ✅ 页面间传参和结果返回
  ✅ 生命周期统一管理

坏处：
  ❌ 引入第三方库，增加维护成本
  ❌ 版本升级可能有 breaking changes
```

---

## 八、常见踩坑清单 ⭐

### 坑 1：首帧白屏/黑屏

```text
现象：打开 Flutter 页面时先看到白屏或黑屏，1-3 秒后才出现内容
原因：FlutterEngine 初始化 + Dart 代码编译需要时间

解决方案（按效果排序）：
┌──────────────────────────────────────┐
│ ⭐ 方案一：预热引擎                    │
│   在 Application 中提前初始化引擎      │
│   效果：几乎秒开                       │
│   代价：常驻 ~50MB 内存               │
├──────────────────────────────────────┤
│ 方案二：SplashScreen 占位图           │
│   在引擎初始化期间显示一个原生View      │
│   效果：不是秒开，但没有白屏了          │
│   代价：实现简单，需要设计闪屏图        │
├──────────────────────────────────────┤
│ 方案三：原生骨架屏                     │
│   先显示一个骨架屏动画，引擎好了再替换   │
│   效果：体验最好                       │
│   代价：实现成本最高                    │
└──────────────────────────────────────┘
```

### 坑 2：ViewPager 中 FlutterFragment 闪烁

```text
现象：ViewPager2 滑动到 FlutterFragment 时出现黑色闪烁
原因：默认 RenderMode.surface（SurfaceView 有 z-order 问题）
解决：改用 RenderMode.texture
```

```kotlin
FlutterFragment
    .withCachedEngine("my_engine")
    .renderMode(RenderMode.texture)  // 解决闪烁
    .build<FlutterFragment>()
```

### 坑 3：withCachedEngine 时 initialRoute 不生效

```text
现象：调了 .initialRoute("/detail") 但 Flutter 还是显示首页
原因：引擎在 Application 中预热时路由就确定了，后面再设置无效

解决：在预热引擎时设置路由
```

```kotlin
// ❌ 错误
FlutterFragment
    .withCachedEngine("my_engine")
    .initialRoute("/detail")  // 这行不会生效！
    .build()

// ✅ 正确：在预热时设置
engine.navigationChannel.setInitialRoute("/detail") // 必须在 executeDartEntrypoint 之前
engine.dartExecutor.executeDartEntrypoint(...)
```

### 坑 4：内存泄漏

```text
现象：反复打开关闭 Flutter 页面后内存持续增长
原因：withNewEngine() 创建的引擎在 Fragment/Activity 销毁后残留

排查：
  withNewEngine() → 引擎随容器自动销毁 ✅（正常应该不会泄漏）
  withCachedEngine() → 引擎不会自动销毁（设计如此）✅

  如果确实泄漏，检查：
    1. MethodChannel 的 Handler 是否持有了 Activity 引用
    2. Flutter 端是否有未释放的 Stream 订阅
    3. FlutterFragment 是否被 ViewModel 等长生命周期对象引用
```

### 坑 5：热重载在混编中不生效

```bash
# 混编模式下不能直接 flutter run
# 需要先运行原生 App，然后 flutter attach 连接

flutter attach -d <设备ID>
# 连接成功后，修改 Dart 代码 → 按 r 热重载 / R 热重启
```

---

## 九、面试应答模板 ⭐⭐ 背下来直接用

### Q1："介绍一下你们项目中 Flutter 混编的方案？"

```text
回答模板：

"我们项目是一个已有的原生 Android 应用，采用的是 Add-to-App 模式，
通过 Flutter Module 嵌入到原生工程中。

在容器层面，我们主要使用 FlutterFragment，因为它比 FlutterActivity 更灵活，
可以嵌入到原有的 Activity 布局中，比如 Tab 页签里的某一页就是 Flutter 写的。

在引擎管理方面，我们在 Application 中预热了 FlutterEngine 并缓存起来，
这样用户打开 Flutter 页面时可以实现秒开，避免白屏等待。
如果有多个 Flutter 页面的场景，我们使用 FlutterEngineGroup 来共享引擎资源，
第一个引擎约 50MB，后续每个只增加约 180KB。

原生和 Flutter 之间通过 MethodChannel 通信，
传递主题参数、业务数据和页面跳转指令。"
```

### Q2："FlutterFragment 和 FlutterActivity 有什么区别？"

```text
回答模板：

"本质上，FlutterActivity 是一个完整的 Activity，整个页面由 Flutter 渲染；
FlutterFragment 是一个 Fragment，可以嵌入到任何 Activity 的布局中。

打个比方，FlutterActivity 就像打开一个全屏的 WebView 页面，
而 FlutterFragment 就像在原生布局里嵌入一个 WebView 组件。

实际开发中我们更多使用 FlutterFragment，因为它更灵活——
可以和原生 View 混合排布，可以放在 ViewPager2 中，
也可以只占页面的一部分。

使用 FlutterFragment 需要注意的是：
1. 如果放在 ViewPager2 中，RenderMode 要改成 texture，否则会闪烁
2. 生命周期要注意引擎的绑定和释放
3. 使用缓存引擎时 initialRoute 需要在预热阶段就设定好"
```

### Q3："怎么实现原生调用 Flutter 弹窗？"

```text
回答模板：

"有三种方案，我们根据场景选择：

最常用的是【透明 FlutterActivity】方案——
配置 Activity 主题为透明（windowIsTranslucent + windowBackground=transparent），
然后设置 FlutterActivity 的 BackgroundMode 为 transparent。
Flutter 端 Scaffold 的背景设为半透明黑色（Colors.black54）作为蒙层，
中间放弹窗内容。这个方案实现简单、效果好。

如果 Flutter 页面已经在运行，比如某个 Tab 是 FlutterFragment，
那我们直接用 MethodChannel 通知 Flutter 端调用 showDialog()，
零额外内存开销。

关于透明背景有一个关键点：必须使用 RenderMode.texture。
因为默认的 SurfaceView 不支持透明通道，只有 TextureView 才支持。
所以透明三要素是：窗口主题透明 + BackgroundMode.transparent + texture 渲染模式。"
```

### Q4："混编中遇到过什么坑？怎么解决的？"

```text
回答模板（挑 2-3 个说）：

"说几个我们实际遇到的坑：

第一个是【首帧白屏】。刚接入时用 withNewEngine()，
用户点击后会白屏 1-2 秒。后来我们改成在 Application 中预热引擎
放到 FlutterEngineCache 里，改用 withCachedEngine()，就实现了秒开。
代价是常驻约 50MB 内存，但相比用户体验是值得的。

第二个是【initialRoute 不生效】。我们用了缓存引擎后，
发现在 FlutterFragment.withCachedEngine() 后面调 .initialRoute()
根本不起作用。排查后发现缓存引擎的路由在预热阶段就确定了，
必须在 executeDartEntrypoint() 之前调用 setInitialRoute()。

第三个是【ViewPager2 中闪烁】。把 FlutterFragment 放到 ViewPager2 后，
滑动到这一页时会出现黑色闪烁。原因是默认的 SurfaceView 有 z-order 问题，
改成 RenderMode.texture 就解决了，虽然性能稍差约 5-10%，但人眼感知不到。"
```

### Q5："Flutter 引擎的内存开销大吗？怎么优化？"

```text
回答模板：

"单个 FlutterEngine 的内存开销约 40-60MB，确实不小。
如果应用中有多个 Flutter 页面，每个都用独立引擎的话，
内存会线性增长，3个页面就要 150MB，不可接受。

Google 官方的解决方案是 FlutterEngineGroup。
它的原理是多个引擎共享同一份 Dart VM snapshot 和 GPU 上下文，
第一个引擎约 50MB，后续每个只增加约 180KB。
这样即使有 5-6 个 Flutter 页面，总内存也只有约 51MB。

另外还有一些优化策略：
1. 按需创建引擎，不用的时候及时 destroy() 释放
2. 全局只预热 1-2 个最常用页面的引擎
3. 在低端机上可以完全不预热，用 SplashScreen 过渡"
```

---

## 附：总览速查图

```text
Flutter 混编知识图谱：

混编方案
├── 容器选择
│   ├── FlutterActivity（整页 Flutter，最简单）
│   ├── FlutterFragment（嵌入原生，最灵活 ⭐）
│   └── FlutterView（极细粒度，不推荐）
│
├── 引擎管理
│   ├── withNewEngine（简单但慢）
│   ├── withCachedEngine（预热秒开 ⭐）
│   └── FlutterEngineGroup（多引擎共享 ⭐）
│
├── 透明/弹窗
│   ├── 三要素：Theme 透明 + BackgroundMode + texture
│   ├── 方案一：透明 FlutterActivity ⭐
│   ├── 方案二：FlutterFragment + DialogFragment
│   └── 方案三：MethodChannel 回调
│
├── 通信
│   ├── MethodChannel（双向方法调用）
│   ├── EventChannel（原生→Flutter 事件流）
│   └── Pigeon（类型安全代码生成）
│
├── 主题适配
│   ├── MethodChannel 传递主题参数
│   ├── 暗黑模式同步
│   └── 状态栏/导航栏统一
│
└── 常见坑
    ├── 首帧白屏 → 预热引擎
    ├── ViewPager 闪烁 → texture 模式
    ├── initialRoute 不生效 → 预热时设置
    └── 内存泄漏 → 正确管理引擎生命周期
```
