# Flutter 与 Dart 基础知识

## 一、Dart 语言核心

### 1. Dart 类型系统

```dart
// Dart 是强类型语言，支持类型推断
var name = '张三';           // 推断为 String
String title = '工程师';     // 显式指定
dynamic anything = 42;       // 动态类型（运行时检查）
Object obj = 'hello';        // 所有类型的基类

// late：延迟初始化
late String description;

// final vs const
final now = DateTime.now();   // 运行时常量
const pi = 3.14159;           // 编译时常量
const list = [1, 2, 3];       // 编译时常量集合（不可变）
```

### 2. 空安全（Sound Null Safety）

```dart
String? nullable;              // 可空
String nonNull = 'hello';     // 非空

// 空安全操作符
int? length = nullable?.length;  // 安全调用
String value = nullable ?? '默认值'; // 空合并
String forced = nullable!;        // 非空断言（危险）

// late 非空延迟初始化
late final Database database;
```

### 3. 异步编程

> **面试常问**：`Future` vs `Stream`，`async/await` 的原理

```dart
// Future：单次异步结果
Future<String> fetchData() async {
  final response = await http.get(Uri.parse('https://api.example.com'));
  return response.body;
}

// Stream：多次异步事件
Stream<int> countStream(int max) async* {
  for (int i = 0; i < max; i++) {
    await Future.delayed(Duration(seconds: 1));
    yield i;  // 逐个产出值
  }
}

// 监听 Stream
countStream(5).listen((value) {
  print(value);
});

// Future 并发
final results = await Future.wait([
  fetchUsers(),
  fetchPosts(),
  fetchComments(),
]);

// Completer：手动控制 Future
final completer = Completer<String>();
// 某处完成
completer.complete('done');
// 使用
final result = await completer.future;
```

### 4. Isolate（隔离）

> **面试常问**：Dart 的并发模型

```dart
// Dart 是单线程模型（Event Loop）
// Isolate 是独立的执行线程，有自己的内存

// 使用 compute（Flutter 提供的简便方法）
final result = await compute(heavyComputation, inputData);

// 手动创建 Isolate
Future<void> spawnIsolate() async {
  final receivePort = ReceivePort();
  await Isolate.spawn(isolateEntry, receivePort.sendPort);

  receivePort.listen((message) {
    print('收到: $message');
  });
}

void isolateEntry(SendPort sendPort) {
  sendPort.send('Hello from isolate');
}
```

**Event Loop 模型**：

```
微任务队列 (Microtask Queue) → 优先级高
事件队列 (Event Queue)       → IO、Timer、手势等

执行顺序：
1. 同步代码
2. 所有微任务
3. 一个事件任务
4. 回到步骤 2
```

### 5. 扩展方法与 Mixin

```dart
// 扩展方法
extension StringExtension on String {
  bool get isEmail => RegExp(r'^[\w-.]+@[\w-]+\.\w+$').hasMatch(this);
  String capitalize() => '${this[0].toUpperCase()}${substring(1)}';
}

// Mixin（代码复用，无构造函数）
mixin Logging {
  void log(String message) => print('[LOG] $message');
}

mixin Caching<T> {
  final _cache = <String, T>{};
  T? getCache(String key) => _cache[key];
  void setCache(String key, T value) => _cache[key] = value;
}

class ApiService with Logging, Caching<String> {
  Future<String> fetchData(String url) async {
    final cached = getCache(url);
    if (cached != null) return cached;

    log('Fetching: $url');
    final data = await http.get(Uri.parse(url));
    setCache(url, data.body);
    return data.body;
  }
}
```

---

## 二、Flutter Widget 体系

### 1. Widget 三棵树

> **面试核心考点**

```
Widget Tree → 描述 UI 的配置（不可变，轻量级）
Element Tree → Widget 的实例化，桥接 Widget 和 RenderObject
RenderObject Tree → 负责实际的布局和绘制
```

| 树 | 职责 | 特点 |
|---|------|------|
| Widget | UI 描述/配置 | 不可变、可频繁重建 |
| Element | 生命周期管理 | 可复用、Diff 比较 |
| RenderObject | 布局和绘制 | 重量级、尽量复用 |

### 2. StatelessWidget vs StatefulWidget

```dart
// 无状态 Widget
class GreetingCard extends StatelessWidget {
  final String name;
  const GreetingCard({Key? key, required this.name}) : super(key: key);

  @override
  Widget build(BuildContext context) {
    return Text('Hello, $name!');
  }
}

// 有状态 Widget
class Counter extends StatefulWidget {
  const Counter({Key? key}) : super(key: key);

  @override
  State<Counter> createState() => _CounterState();
}

class _CounterState extends State<Counter> {
  int _count = 0;

  @override
  Widget build(BuildContext context) {
    return Column(
      children: [
        Text('Count: $_count'),
        ElevatedButton(
          onPressed: () => setState(() => _count++),
          child: const Text('增加'),
        ),
      ],
    );
  }
}
```

### 3. Widget 生命周期

```dart
class MyWidget extends StatefulWidget {
  @override
  State<MyWidget> createState() => _MyWidgetState();
}

class _MyWidgetState extends State<MyWidget> {
  @override
  void initState() {
    super.initState();
    // 初始化（只调用一次）
  }

  @override
  void didChangeDependencies() {
    super.didChangeDependencies();
    // InheritedWidget 变化时调用
  }

  @override
  void didUpdateWidget(MyWidget oldWidget) {
    super.didUpdateWidget(oldWidget);
    // 父 Widget 重建且配置变化时调用
  }

  @override
  Widget build(BuildContext context) {
    // 构建 UI（可多次调用）
    return Container();
  }

  @override
  void deactivate() {
    super.deactivate();
    // 从树中暂时移除
  }

  @override
  void dispose() {
    // 永久移除，释放资源
    super.dispose();
  }
}
```

### 4. Key 的作用

> **面试常问**：Key 有什么用？什么时候需要使用？

```dart
// Key 帮助 Flutter 识别哪些 Widget 发生了变化
// 在列表中重排序、增删时必须使用

// ValueKey：基于值判断
ListView(
  children: items.map((item) =>
    ListTile(key: ValueKey(item.id), title: Text(item.name))
  ).toList(),
);

// ObjectKey：基于对象引用
ObjectKey(userObject)

// UniqueKey：保证唯一性
UniqueKey()

// GlobalKey：跨 Widget 树访问 State
final formKey = GlobalKey<FormState>();
Form(key: formKey, child: /* ... */);
formKey.currentState?.validate();
```

---

## 三、常用布局与组件

### 1. 布局 Widget

```dart
// Row / Column（Flex 布局）
Row(
  mainAxisAlignment: MainAxisAlignment.spaceBetween,
  crossAxisAlignment: CrossAxisAlignment.center,
  children: [/* ... */],
)

// Stack（层叠布局）
Stack(
  children: [
    Image.asset('bg.png'),
    Positioned(
      bottom: 16, right: 16,
      child: FloatingActionButton(onPressed: () {}),
    ),
  ],
)

// ListView（滚动列表）
ListView.builder(
  itemCount: items.length,
  itemBuilder: (context, index) => ListTile(title: Text(items[index])),
)

// GridView
GridView.builder(
  gridDelegate: SliverGridDelegateWithFixedCrossAxisCount(
    crossAxisCount: 2,
    crossAxisSpacing: 8,
    mainAxisSpacing: 8,
  ),
  itemBuilder: (context, index) => Card(child: /* ... */),
)

// CustomScrollView + Sliver
CustomScrollView(
  slivers: [
    SliverAppBar(expandedHeight: 200, flexibleSpace: /* ... */),
    SliverList(delegate: SliverChildBuilderDelegate(/* ... */)),
    SliverGrid(/* ... */),
  ],
)
```

---

## 四、状态管理

> **面试高频考点**

### 1. 原生方案

#### InheritedWidget

```dart
class ThemeProvider extends InheritedWidget {
  final ThemeData theme;

  const ThemeProvider({
    required this.theme,
    required Widget child,
  }) : super(child: child);

  static ThemeProvider of(BuildContext context) {
    return context.dependOnInheritedWidgetOfExactType<ThemeProvider>()!;
  }

  @override
  bool updateShouldNotify(ThemeProvider oldWidget) {
    return theme != oldWidget.theme;
  }
}
```

### 2. Provider（推荐入门方案）

```dart
// ChangeNotifier
class CartModel extends ChangeNotifier {
  final List<Item> _items = [];
  List<Item> get items => UnmodifiableListView(_items);

  void add(Item item) {
    _items.add(item);
    notifyListeners();
  }
}

// 提供
ChangeNotifierProvider(
  create: (context) => CartModel(),
  child: MyApp(),
)

// 消费
Consumer<CartModel>(
  builder: (context, cart, child) {
    return Text('${cart.items.length} items');
  },
)

// 或使用 context.watch / context.read
final cart = context.watch<CartModel>();  // 订阅变化
final cart = context.read<CartModel>();   // 一次性读取
```

### 3. Riverpod

```dart
// 定义 Provider
final counterProvider = StateNotifierProvider<CounterNotifier, int>((ref) {
  return CounterNotifier();
});

class CounterNotifier extends StateNotifier<int> {
  CounterNotifier() : super(0);
  void increment() => state++;
}

// 使用
class CounterWidget extends ConsumerWidget {
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final count = ref.watch(counterProvider);
    return Text('$count');
  }
}

// 异步
final userProvider = FutureProvider<User>((ref) async {
  return ref.read(apiProvider).fetchUser();
});
```

### 4. BLoC（Business Logic Component）

```dart
// Event
abstract class LoginEvent {}
class LoginSubmitted extends LoginEvent {
  final String email, password;
  LoginSubmitted(this.email, this.password);
}

// State
abstract class LoginState {}
class LoginInitial extends LoginState {}
class LoginLoading extends LoginState {}
class LoginSuccess extends LoginState {}
class LoginFailure extends LoginState {
  final String error;
  LoginFailure(this.error);
}

// BLoC
class LoginBloc extends Bloc<LoginEvent, LoginState> {
  LoginBloc() : super(LoginInitial()) {
    on<LoginSubmitted>((event, emit) async {
      emit(LoginLoading());
      try {
        await authRepository.login(event.email, event.password);
        emit(LoginSuccess());
      } catch (e) {
        emit(LoginFailure(e.toString()));
      }
    });
  }
}

// 使用
BlocBuilder<LoginBloc, LoginState>(
  builder: (context, state) {
    if (state is LoginLoading) return CircularProgressIndicator();
    if (state is LoginSuccess) return Text('登录成功');
    return LoginForm();
  },
)
```

### 5. 状态管理方案对比

| 方案 | 复杂度 | 适用场景 | 特点 |
|------|--------|---------|------|
| `setState` | 低 | 局部状态 | 简单直接 |
| `InheritedWidget` | 中 | 主题、语言 | 原生方案 |
| `Provider` | 低-中 | 中小项目 | 官方推荐 |
| `Riverpod` | 中 | 中大项目 | Provider 升级版 |
| `BLoC` | 高 | 大型项目 | 严格单向数据流 |
| `GetX` | 低 | 快速开发 | 功能全但争议多 |

---

## 五、路由与导航

### 1. Navigator 1.0

```dart
// 命名路由
MaterialApp(
  routes: {
    '/': (context) => HomeScreen(),
    '/detail': (context) => DetailScreen(),
  },
);

Navigator.pushNamed(context, '/detail', arguments: {'id': 123});

// 参数接收
final args = ModalRoute.of(context)!.settings.arguments as Map;
```

### 2. Navigator 2.0 / go_router

```dart
final router = GoRouter(
  routes: [
    GoRoute(
      path: '/',
      builder: (context, state) => HomeScreen(),
      routes: [
        GoRoute(
          path: 'detail/:id',
          builder: (context, state) {
            final id = state.pathParameters['id'];
            return DetailScreen(id: id!);
          },
        ),
      ],
    ),
  ],
);

// 导航
context.go('/detail/123');
context.push('/detail/123');  // 可返回
```

---

## 六、平台通道（Platform Channel）

> **面试常问**：Flutter 如何与原生交互？

### 1. MethodChannel

```dart
// Flutter 端
class NativeBridge {
  static const _channel = MethodChannel('com.example/native');

  static Future<String> getBatteryLevel() async {
    final result = await _channel.invokeMethod<int>('getBatteryLevel');
    return '$result%';
  }

  static void listenNativeCalls() {
    _channel.setMethodCallHandler((call) async {
      switch (call.method) {
        case 'onDataReceived':
          return handleData(call.arguments);
        default:
          throw MissingPluginException();
      }
    });
  }
}
```

```kotlin
// Android 端（Kotlin）
class MainActivity : FlutterActivity() {
    override fun configureFlutterEngine(flutterEngine: FlutterEngine) {
        MethodChannel(flutterEngine.dartExecutor.binaryMessenger, "com.example/native")
            .setMethodCallHandler { call, result ->
                when (call.method) {
                    "getBatteryLevel" -> {
                        val level = getBatteryLevel()
                        result.success(level)
                    }
                    else -> result.notImplemented()
                }
            }
    }
}
```

### 2. EventChannel（持续通信）

```dart
// Flutter 端
const eventChannel = EventChannel('com.example/sensor');
eventChannel.receiveBroadcastStream().listen((data) {
  print('传感器数据: $data');
});
```

### 3. 通信方式对比

| 类型 | 方向 | 用途 |
|------|------|------|
| `MethodChannel` | 双向 | 方法调用（一次性请求/响应） |
| `EventChannel` | 原生→Flutter | 持续事件流（传感器、定位） |
| `BasicMessageChannel` | 双向 | 自定义编解码器的消息传递 |

---

## 七、Flutter 性能优化

### 1. 构建优化

```dart
// 1. 使用 const 构造函数
const Text('不变的文本');
const SizedBox(height: 16);

// 2. 拆分 Widget，缩小重建范围
// ❌ 整个页面在一个 build 方法中
// ✅ 拆分为多个小 Widget

// 3. 使用 RepaintBoundary 隔离重绘区域
RepaintBoundary(
  child: AnimatedWidget(),  // 频繁更新的部分
)

// 4. ListView.builder 而非 ListView（children:[...])
// builder 按需构建，children 全部构建
```

### 2. 调试工具

- **Flutter DevTools**：性能分析、Widget Inspector
- **Flutter Inspector**：Widget 树、渲染信息
- **Performance Overlay**：监控帧率

```dart
MaterialApp(
  showPerformanceOverlay: true,
)
```
