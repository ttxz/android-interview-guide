# Android Fragment 生命周期与通信

## 一、Fragment 生命周期

### 1. 完整生命周期回调

```
onAttach() → onCreate() → onCreateView() → onViewCreated()
→ onStart() → onResume() → [运行中]
→ onPause() → onStop() → onDestroyView() → onDestroy() → onDetach()
```

| 回调 | 说明 |
|------|------|
| `onAttach()` | 关联到 Activity |
| `onCreate()` | 初始化（不涉及 UI），接收 `savedInstanceState` |
| `onCreateView()` | 创建视图（inflate layout） |
| `onViewCreated()` | 视图创建完成，初始化 UI 组件 |
| `onStart()` | Fragment 可见 |
| `onResume()` | Fragment 可交互 |
| `onPause()` | Fragment 部分遮挡 |
| `onStop()` | Fragment 不可见 |
| `onDestroyView()` | 视图销毁（但 Fragment 实例可能存在） |
| `onDestroy()` | Fragment 销毁 |
| `onDetach()` | 从 Activity 分离 |

### 2. Fragment 与 Activity 生命周期关系

```
Activity.onCreate()    → Fragment.onAttach() → onCreate() → onCreateView() → onViewCreated()
Activity.onStart()     → Fragment.onStart()
Activity.onResume()    → Fragment.onResume()
Activity.onPause()     → Fragment.onPause()
Activity.onStop()      → Fragment.onStop()
Activity.onDestroy()   → Fragment.onDestroyView() → onDestroy() → onDetach()
```

### 3. 面试常问：View 生命周期 vs Fragment 生命周期

> **关键**：Fragment 的 View 可能被销毁而 Fragment 实例仍然存在（如 ViewPager 中：`onDestroyView()` 后 Fragment 仍在）

```kotlin
class MyFragment : Fragment(R.layout.fragment_my) {

    // ❌ 内存泄漏！Fragment 比 View 长寿
    // private val binding = FragmentMyBinding.bind(view)

    // ✅ 正确做法
    private var _binding: FragmentMyBinding? = null
    private val binding get() = _binding!!

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)
        _binding = FragmentMyBinding.bind(view)
        // 使用 viewLifecycleOwner 而非 this
        viewLifecycleOwner.lifecycleScope.launch {
            viewModel.uiState.collect { /* ... */ }
        }
    }

    override fun onDestroyView() {
        super.onDestroyView()
        _binding = null  // 释放 binding 引用
    }
}
```

---

## 二、Fragment 事务

### 1. 基本操作

```kotlin
supportFragmentManager.commit {
    setReorderingAllowed(true)  // 优化事务执行顺序
    replace(R.id.container, MyFragment())
    addToBackStack("tag")       // 加入返回栈
}

// 各种操作
add(containerId, fragment)       // 添加
remove(fragment)                 // 移除
replace(containerId, fragment)   // 替换（= remove + add）
show(fragment)                   // 显示
hide(fragment)                   // 隐藏
attach(fragment)                 // 重新关联（重新创建视图）
detach(fragment)                 // 分离（销毁视图，保留实例）
```

### 2. add vs replace

| 操作 | `add` | `replace` |
|------|-------|-----------|
| 旧 Fragment | 保留在容器中 | 从容器中移除 |
| 生命周期 | 旧 Fragment 不变 | 旧 Fragment → `onDestroyView()` |
| 叠加 | 覆盖在上方 | 只保留新 Fragment |
| 返回栈 | 移除新 Fragment，旧的可见 | 移除新 Fragment，重建旧 Fragment 视图 |

### 3. commit 方法对比

| 方法 | 特点 |
|------|------|
| `commit()` | 异步执行，不能在 `onSaveInstanceState` 后调用 |
| `commitAllowingStateLoss()` | 异步，允许状态丢失（谨慎使用） |
| `commitNow()` | 同步执行，不支持 `addToBackStack` |
| `commitNowAllowingStateLoss()` | 同步，允许状态丢失 |

---

## 三、Fragment 间通信

> **面试高频考点**

### 1. Fragment Result API（推荐，AndroidX）

```kotlin
// Fragment A（接收结果）
setFragmentResultListener("requestKey") { _, bundle ->
    val result = bundle.getString("resultKey")
}

// Fragment B（发送结果）
setFragmentResult("requestKey", bundleOf("resultKey" to "Hello"))

// 父子关系的通信
// 子 Fragment
parentFragmentManager.setFragmentResult("childResult", bundleOf("data" to value))

// 父 Fragment
childFragmentManager.setFragmentResultListener("childResult") { _, bundle -> }
```

### 2. 共享 ViewModel

```kotlin
// 共享的 ViewModel
class SharedViewModel : ViewModel() {
    private val _selectedItem = MutableLiveData<Item>()
    val selectedItem: LiveData<Item> = _selectedItem

    fun selectItem(item: Item) {
        _selectedItem.value = item
    }
}

// Fragment A
class ListFragment : Fragment() {
    // activityViewModels：Activity 范围共享
    private val sharedVM: SharedViewModel by activityViewModels()

    fun onItemClicked(item: Item) {
        sharedVM.selectItem(item)
    }
}

// Fragment B
class DetailFragment : Fragment() {
    private val sharedVM: SharedViewModel by activityViewModels()

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        sharedVM.selectedItem.observe(viewLifecycleOwner) { item ->
            updateUI(item)
        }
    }
}
```

### 3. 通信方式对比

| 方式 | 优点 | 缺点 | 适用场景 |
|------|------|------|---------|
| Fragment Result API | 解耦、类型安全 | 仅支持 Bundle 数据 | 简单数据传递 |
| 共享 ViewModel | 生命周期感知、响应式 | 需设计好共享范围 | 复杂数据共享 |
| Interface（旧方式） | 直观 | 耦合度高 | 遗留代码 |
| Navigation Args | Safe Args 类型安全 | 仅用于导航 | 页面间传参 |

---

## 四、ViewPager2 + Fragment

### 面试常问：Fragment 在 ViewPager2 中的生命周期

```kotlin
// FragmentStateAdapter（推荐）
class MyPagerAdapter(
    fragmentActivity: FragmentActivity
) : FragmentStateAdapter(fragmentActivity) {

    override fun getItemCount() = 3

    override fun createFragment(position: Int) = when (position) {
        0 -> HomeFragment()
        1 -> SearchFragment()
        2 -> ProfileFragment()
        else -> throw IllegalStateException()
    }
}

// 设置离屏页面数量
viewPager.offscreenPageLimit = 1  // 默认不预加载
```

**生命周期变化**：

- 当前页：全生命周期
- 相邻页（在 offscreenPageLimit 内）：创建到 `onStart()`
- 超出范围的页面：`onDestroyView()` → `onDestroy()`

### 懒加载方案

```kotlin
class LazyFragment : Fragment() {
    private var isDataLoaded = false

    override fun onResume() {
        super.onResume()
        if (!isDataLoaded) {
            loadData()
            isDataLoaded = true
        }
    }
}
```
