# Android RecyclerView 高级用法

## 一、RecyclerView 核心架构

### 1. 核心组件

```
RecyclerView
├── LayoutManager      → 负责布局（线性、网格、瀑布流）
├── Adapter            → 负责数据绑定
├── ViewHolder         → 负责视图缓存
├── ItemDecoration     → 负责装饰（分割线、间距）
├── ItemAnimator       → 负责动画
└── RecycledViewPool   → 负责缓存池
```

### 2. RecyclerView 缓存机制

> **面试核心考点**：四级缓存

| 级别 | 名称 | 默认大小 | 是否需要重新绑定 | 说明 |
|------|------|---------|----------------|------|
| 一级 | `mAttachedScrap` | 无限制 | 否 | 屏幕内可见的 ViewHolder |
| 一级 | `mChangedScrap` | 无限制 | 是 | 数据变化的 ViewHolder |
| 二级 | `mCachedViews` | 2 | 否 | 刚滚出屏幕的 ViewHolder |
| 三级 | `ViewCacheExtension` | 自定义 | 自定义 | 开发者自定义缓存 |
| 四级 | `RecycledViewPool` | 5/类型 | 是 | 按 viewType 分类的缓存池 |

**缓存查找流程**：

```
滑动需要新 View
  → 1. 查找 Scrap（屏幕内）
  → 2. 查找 CachedViews（position 匹配，无需 bind）
  → 3. 查找 ViewCacheExtension（自定义）
  → 4. 查找 RecycledViewPool（type 匹配，需要 bind）
  → 5. 都没找到 → createViewHolder() + bindViewHolder()
```

```kotlin
// 增加缓存大小
recyclerView.setItemViewCacheSize(10)  // 增加二级缓存

// 共享缓存池（多个 RecyclerView 共享）
val pool = RecyclerView.RecycledViewPool()
pool.setMaxRecycledViews(VIEW_TYPE_ITEM, 20)
recyclerView1.setRecycledViewPool(pool)
recyclerView2.setRecycledViewPool(pool)
```

---

## 二、Adapter 实现

### 1. ListAdapter + DiffUtil（推荐方案）

> **面试常问**：DiffUtil 的原理？

```kotlin
class UserAdapter : ListAdapter<User, UserAdapter.ViewHolder>(UserDiffCallback()) {

    class ViewHolder(val binding: ItemUserBinding) : RecyclerView.ViewHolder(binding.root) {
        fun bind(user: User) {
            binding.apply {
                nameText.text = user.name
                emailText.text = user.email
                Glide.with(avatar).load(user.avatarUrl).into(avatar)
            }
        }
    }

    override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): ViewHolder {
        val binding = ItemUserBinding.inflate(
            LayoutInflater.from(parent.context), parent, false
        )
        return ViewHolder(binding)
    }

    override fun onBindViewHolder(holder: ViewHolder, position: Int) {
        holder.bind(getItem(position))
    }
}

// DiffUtil.ItemCallback
class UserDiffCallback : DiffUtil.ItemCallback<User>() {
    // 判断是否同一个 Item（通常比较 ID）
    override fun areItemsTheSame(oldItem: User, newItem: User): Boolean {
        return oldItem.id == newItem.id
    }

    // 判断内容是否相同（决定是否需要更新）
    override fun areContentsTheSame(oldItem: User, newItem: User): Boolean {
        return oldItem == newItem
    }

    // 可选：返回变化的部分（局部更新用）
    override fun getChangePayload(oldItem: User, newItem: User): Any? {
        return if (oldItem.name != newItem.name) "name_changed" else null
    }
}

// 使用
adapter.submitList(newList)  // 自动在后台线程计算 Diff
```

**DiffUtil 核心算法**：

- 使用 **Eugene Myers 差分算法**
- 时间复杂度：`O(N + D²)`，N = 列表长度，D = 编辑距离
- `AsyncListDiffer` / `ListAdapter` 在**后台线程**计算 Diff

### 2. 多 ViewType

```kotlin
class MultiTypeAdapter : ListAdapter<ListItem, RecyclerView.ViewHolder>(DiffCallback()) {

    companion object {
        const val TYPE_HEADER = 0
        const val TYPE_CONTENT = 1
        const val TYPE_FOOTER = 2
    }

    override fun getItemViewType(position: Int) = when (getItem(position)) {
        is ListItem.Header -> TYPE_HEADER
        is ListItem.Content -> TYPE_CONTENT
        is ListItem.Footer -> TYPE_FOOTER
    }

    override fun onCreateViewHolder(parent: ViewGroup, viewType: Int) = when (viewType) {
        TYPE_HEADER -> HeaderViewHolder(/* ... */)
        TYPE_CONTENT -> ContentViewHolder(/* ... */)
        TYPE_FOOTER -> FooterViewHolder(/* ... */)
        else -> throw IllegalArgumentException("Unknown viewType: $viewType")
    }

    override fun onBindViewHolder(holder: RecyclerView.ViewHolder, position: Int) {
        when (val item = getItem(position)) {
            is ListItem.Header -> (holder as HeaderViewHolder).bind(item)
            is ListItem.Content -> (holder as ContentViewHolder).bind(item)
            is ListItem.Footer -> (holder as FooterViewHolder).bind(item)
        }
    }
}

// 密封类定义数据类型
sealed class ListItem {
    data class Header(val title: String) : ListItem()
    data class Content(val data: ContentData) : ListItem()
    data class Footer(val text: String) : ListItem()
}
```

---

## 三、性能优化

> **面试高频考点**

### 1. 核心优化策略

```kotlin
// 1. 固定大小（如果 Item 大小不影响 RecyclerView 大小）
recyclerView.setHasFixedSize(true)

// 2. 预取设置
(recyclerView.layoutManager as LinearLayoutManager).apply {
    initialPrefetchItemCount = 4  // 嵌套 RecyclerView 预取
}

// 3. 避免嵌套滚动冲突
recyclerView.isNestedScrollingEnabled = false

// 4. 减少布局层级
// 使用 ConstraintLayout 替代多层嵌套的 LinearLayout/RelativeLayout

// 5. 图片优化（在滑动时暂停加载）
recyclerView.addOnScrollListener(object : RecyclerView.OnScrollListener() {
    override fun onScrollStateChanged(recyclerView: RecyclerView, newState: Int) {
        when (newState) {
            RecyclerView.SCROLL_STATE_IDLE -> Glide.with(context).resumeRequests()
            else -> Glide.with(context).pauseRequests()
        }
    }
})
```

### 2. Payload 局部更新

```kotlin
override fun onBindViewHolder(
    holder: ViewHolder,
    position: Int,
    payloads: MutableList<Any>   // payloads 不为空时做局部更新
) {
    if (payloads.isEmpty()) {
        // 全量更新
        holder.bind(getItem(position))
    } else {
        // 局部更新（更高效）
        payloads.forEach { payload ->
            when (payload) {
                "name_changed" -> holder.updateName(getItem(position).name)
                "avatar_changed" -> holder.updateAvatar(getItem(position).avatarUrl)
            }
        }
    }
}
```

### 3. 常见性能问题及解决

| 问题 | 原因 | 解决方案 |
|------|------|---------|
| 滑动卡顿 | `onBindViewHolder` 耗时 | 减少 bind 操作，使用预加载 |
| 内存泄漏 | ViewHolder 持有外部引用 | 使用弱引用或及时释放 |
| Item 闪烁 | `notifyDataSetChanged()` | 使用 DiffUtil / 精确通知 |
| 首次加载慢 | Layout 过于复杂 | 使用 `ConstraintLayout`，减少层级 |
| 图片抖动 | ViewHolder 复用，异步加载 | 设置 placeholder + tag 校验 |

---

## 四、ItemDecoration

```kotlin
// 自定义分割线
class SpaceItemDecoration(
    private val space: Int,
    private val orientation: Int = LinearLayoutManager.VERTICAL
) : RecyclerView.ItemDecoration() {

    override fun getItemOffsets(
        outRect: Rect,
        view: View,
        parent: RecyclerView,
        state: RecyclerView.State
    ) {
        if (orientation == LinearLayoutManager.VERTICAL) {
            outRect.bottom = space
        } else {
            outRect.right = space
        }
    }

    override fun onDraw(c: Canvas, parent: RecyclerView, state: RecyclerView.State) {
        // 在 Item 下方绘制（在 Item 之前绘制）
    }

    override fun onDrawOver(c: Canvas, parent: RecyclerView, state: RecyclerView.State) {
        // 在 Item 上方绘制（在 Item 之后绘制，如悬浮吸顶效果）
    }
}
```

---

## 五、ConcatAdapter（合并适配器）

```kotlin
// 将多个 Adapter 合并为一个
val headerAdapter = HeaderAdapter()
val contentAdapter = ContentAdapter()
val footerAdapter = FooterAdapter()

recyclerView.adapter = ConcatAdapter(
    ConcatAdapter.Config.Builder()
        .setIsolateViewTypes(true)  // 隔离 viewType，避免冲突
        .build(),
    headerAdapter,
    contentAdapter,
    footerAdapter
)
```

---

## 六、ItemTouchHelper（拖拽与滑动删除）

```kotlin
val callback = object : ItemTouchHelper.SimpleCallback(
    ItemTouchHelper.UP or ItemTouchHelper.DOWN,  // 拖拽方向
    ItemTouchHelper.LEFT or ItemTouchHelper.RIGHT // 滑动方向
) {
    override fun onMove(
        recyclerView: RecyclerView,
        viewHolder: RecyclerView.ViewHolder,
        target: RecyclerView.ViewHolder
    ): Boolean {
        val fromPos = viewHolder.adapterPosition
        val toPos = target.adapterPosition
        adapter.moveItem(fromPos, toPos)
        return true
    }

    override fun onSwiped(viewHolder: RecyclerView.ViewHolder, direction: Int) {
        val position = viewHolder.adapterPosition
        adapter.removeItem(position)
    }
}

ItemTouchHelper(callback).attachToRecyclerView(recyclerView)
```
