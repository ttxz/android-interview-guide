# Android Activity 生命周期与启动模式

## 一、Activity 生命周期

### 1. 生命周期回调

```
onCreate() → onStart() → onResume() → [运行中]
                                          ↓
                                     onPause() → onStop() → onDestroy()
                                        ↑           ↑
                                     onResume()   onRestart() → onStart()
```

| 回调 | 调用时机 | 常见操作 |
|------|---------|---------|
| `onCreate()` | Activity 首次创建 | 初始化视图、绑定数据、恢复状态 |
| `onStart()` | Activity 变为可见 | 注册广播接收器 |
| `onResume()` | Activity 进入前台可交互 | 开始动画、获取相机资源 |
| `onPause()` | Activity 部分遮挡/失焦 | 暂停动画、保存轻量数据 |
| `onStop()` | Activity 完全不可见 | 释放重量级资源、持久化数据 |
| `onDestroy()` | Activity 被销毁 | 释放所有资源 |
| `onRestart()` | 从停止状态重新启动 | - |

### 2. 面试常问场景

**场景一：A 启动 B**

```
A.onPause() → B.onCreate() → B.onStart() → B.onResume() → A.onStop()
```

> ⚠️ A.onPause() 先于 B.onCreate()！因此 onPause() 不要做耗时操作。

**场景二：B 返回 A**

```
B.onPause() → A.onRestart() → A.onStart() → A.onResume() → B.onStop() → B.onDestroy()
```

**场景三：横竖屏切换**

- 默认：销毁 → 重建（`onDestroy() → onCreate()`）
- 配置 `android:configChanges="orientation|screenSize"` → 不重建，回调 `onConfigurationChanged()`
- 用 `ViewModel` 保存 UI 状态（不受配置变更影响）

**场景四：内存不足被杀**

```
onSaveInstanceState() → [被杀] → onCreate(savedInstanceState) / onRestoreInstanceState()
```

### 3. onSaveInstanceState 调用时机

> **面试常问**：onSaveInstanceState 什么时候调用？

- Android P 之前：在 `onStop()` **之前**
- Android P 及之后：在 `onStop()` **之后**
- 保证在 `onDestroy()` 之前
- **不适合**保存大量数据（Bundle 有大小限制 ~1MB）

```kotlin
override fun onSaveInstanceState(outState: Bundle) {
    super.onSaveInstanceState(outState)
    outState.putString("KEY_QUERY", searchQuery)
    outState.putInt("KEY_POSITION", recyclerView.scrollPosition)
}

override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    savedInstanceState?.let {
        searchQuery = it.getString("KEY_QUERY", "")
    }
}
```

---

## 二、Activity 启动模式（LaunchMode）

> **面试高频考点**

### 1. 四种启动模式

| 模式 | 特点 | 适用场景 |
|------|------|---------|
| `standard` | 默认，每次创建新实例 | 普通页面 |
| `singleTop` | 栈顶复用，调用 `onNewIntent()` | 搜索页、通知详情页 |
| `singleTask` | 栈内复用，清除其上 Activity | 首页、主页面 |
| `singleInstance` | 独占一个任务栈 | 来电界面、特殊入口 |

### 2. 详解与图示

#### standard（标准模式）

```
任务栈：[A → B → B → B]  // 每次 startActivity(B) 都创建新 B
```

#### singleTop（栈顶复用）

```
栈顶是 B：startActivity(B) → 不创建新 B，调用 B.onNewIntent()
栈顶不是 B：startActivity(B) → 创建新 B

任务栈：[A → B → C]
startActivity(C)  → [A → B → C]（C 在栈顶，复用，调用 onNewIntent）
startActivity(B)  → [A → B → C → B]（B 不在栈顶，创建新的）
```

#### singleTask（栈内复用）

```
任务栈：[A → B → C → D]
startActivity(B)  → [A → B]（B 已在栈内，移除其上的 C、D，调用 B.onNewIntent）
```

#### singleInstance（单实例）

```
Task1: [A → C]
Task2: [B]  // B 独占 Task2

startActivity(B) → 切换到 Task2 复用 B，调用 onNewIntent()
```

### 3. Intent Flags

```kotlin
// 等价于 singleTop
intent.addFlags(Intent.FLAG_ACTIVITY_SINGLE_TOP)

// 等价于 singleTask（配合 FLAG_ACTIVITY_CLEAR_TOP）
intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK or Intent.FLAG_ACTIVITY_CLEAR_TOP)

// 清除栈顶（如果目标 Activity 在栈中，销毁其上所有 Activity）
intent.addFlags(Intent.FLAG_ACTIVITY_CLEAR_TOP)

// 新建任务栈
intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK)

// 不记录到最近任务列表
intent.addFlags(Intent.FLAG_ACTIVITY_EXCLUDE_FROM_RECENTS)
```

### 4. taskAffinity

```xml
<!-- 指定 Activity 归属的任务栈 -->
<activity
    android:name=".DetailActivity"
    android:taskAffinity="com.example.detail"
    android:launchMode="singleTask" />
```

- 默认值为应用包名
- 需配合 `singleTask` 或 `FLAG_ACTIVITY_NEW_TASK` 使用
- 相同 `taskAffinity` 的 Activity 在同一个任务栈

---

## 三、Activity 间通信

### 1. Intent 传递数据

```kotlin
// 基本类型
val intent = Intent(this, DetailActivity::class.java).apply {
    putExtra("id", 123)
    putExtra("name", "John")
}

// Parcelable（推荐）
@Parcelize
data class User(val name: String, val age: Int) : Parcelable

intent.putExtra("user", user)

// Bundle
val bundle = bundleOf(
    "key1" to "value1",
    "key2" to 123
)
intent.putExtras(bundle)
```

### 2. Activity Result API（替代 startActivityForResult）

> **面试加分项**：展示对新 API 的了解

```kotlin
// 注册结果回调
val launcher = registerForActivityResult(
    ActivityResultContracts.StartActivityForResult()
) { result ->
    if (result.resultCode == RESULT_OK) {
        val data = result.data?.getStringExtra("result")
    }
}

// 启动
launcher.launch(Intent(this, EditActivity::class.java))

// 预定义 Contract
val pickImage = registerForActivityResult(ActivityResultContracts.GetContent()) { uri ->
    uri?.let { imageView.setImageURI(it) }
}
pickImage.launch("image/*")

val requestPermission = registerForActivityResult(
    ActivityResultContracts.RequestPermission()
) { isGranted ->
    if (isGranted) { /* 权限已授予 */ }
}
requestPermission.launch(Manifest.permission.CAMERA)
```

---

## 四、进程和任务管理

### 面试常问：前台/后台进程优先级

1. **前台进程**：正在交互的 Activity（最高优先级）
2. **可见进程**：可见但不可交互的 Activity
3. **服务进程**：运行 Service 的进程
4. **缓存/后台进程**：不可见的 Activity
5. **空进程**：没有活动组件（最低优先级）

> 系统内存不足时，按优先级从低到高杀进程。
