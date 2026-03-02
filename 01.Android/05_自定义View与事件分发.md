# Android 自定义 View 与事件分发

## 一、View 绘制流程

> **面试核心考点**：View 的三大流程

### 1. measure（测量）

```kotlin
// MeasureSpec 三种模式
MeasureSpec.EXACTLY   // 精确值：match_parent 或具体 dp
MeasureSpec.AT_MOST   // 最大值：wrap_content
MeasureSpec.UNSPECIFIED // 不限制：ScrollView 中的子 View

// 自定义 View 测量
override fun onMeasure(widthMeasureSpec: Int, heightMeasureSpec: Int) {
    val widthMode = MeasureSpec.getMode(widthMeasureSpec)
    val widthSize = MeasureSpec.getSize(widthMeasureSpec)

    val width = when (widthMode) {
        MeasureSpec.EXACTLY -> widthSize
        MeasureSpec.AT_MOST -> min(desiredWidth, widthSize)
        else -> desiredWidth
    }

    setMeasuredDimension(width, height)
}
```

### 2. layout（布局）

```kotlin
// ViewGroup 中摆放子 View
override fun onLayout(changed: Boolean, l: Int, t: Int, r: Int, b: Int) {
    var currentTop = paddingTop
    for (i in 0 until childCount) {
        val child = getChildAt(i)
        if (child.visibility != View.GONE) {
            child.layout(
                paddingLeft,
                currentTop,
                paddingLeft + child.measuredWidth,
                currentTop + child.measuredHeight
            )
            currentTop += child.measuredHeight + spacing
        }
    }
}
```

### 3. draw（绘制）

```kotlin
// 绘制顺序
// 1. drawBackground()    → 背景
// 2. onDraw()            → 自身内容
// 3. dispatchDraw()      → 子 View
// 4. onDrawForeground()  → 前景、滚动条

override fun onDraw(canvas: Canvas) {
    super.onDraw(canvas)

    // 绘制圆形
    canvas.drawCircle(centerX, centerY, radius, paint)

    // 绘制文字
    canvas.drawText("Hello", x, y, textPaint)

    // 绘制路径
    canvas.drawPath(path, paint)

    // 保存和恢复画布状态
    canvas.save()
    canvas.translate(dx, dy)
    canvas.rotate(angle)
    // 绘制操作...
    canvas.restore()
}
```

### 4. requestLayout vs invalidate vs postInvalidate

| 方法 | 触发流程 | 线程 | 使用场景 |
|------|---------|------|---------|
| `requestLayout()` | measure → layout → draw | 主线程 | 尺寸/位置改变 |
| `invalidate()` | draw | 主线程 | 外观改变（颜色、透明度） |
| `postInvalidate()` | draw | 任意线程 | 子线程触发重绘 |

---

## 二、事件分发机制

> **面试最高频考点之一**

### 1. 事件分发三个核心方法

```kotlin
// Activity
dispatchTouchEvent(event)    // 分发事件
onTouchEvent(event)          // 处理事件

// ViewGroup
dispatchTouchEvent(event)    // 分发事件
onInterceptTouchEvent(event) // 拦截事件
onTouchEvent(event)          // 处理事件

// View
dispatchTouchEvent(event)    // 分发事件
onTouchEvent(event)          // 处理事件
```

### 2. 分发流程（U 型图）

```
Activity.dispatchTouchEvent()
    ↓
ViewGroup.dispatchTouchEvent()
    ↓
ViewGroup.onInterceptTouchEvent()  → true: 拦截，自己处理
    ↓ false                             ↓
子 View.dispatchTouchEvent()      ViewGroup.onTouchEvent()
    ↓                                   ↓ false
子 View.onTouchEvent()           Activity.onTouchEvent()
    ↓ false（不消费）
ViewGroup.onTouchEvent()
    ↓ false
Activity.onTouchEvent()
```

### 3. 关键规则

1. **DOWN 事件决定后续事件归属**：如果某个 View 消费了 DOWN 事件，后续 MOVE/UP 都给它
2. **一旦拦截，后续事件不再询问 `onInterceptTouchEvent`**
3. **子 View 可通过 `requestDisallowInterceptTouchEvent(true)` 阻止父 View 拦截**
4. **onClick 在 onTouchEvent 中的 UP 事件处理**

### 4. 滑动冲突解决

```kotlin
// 方案一：外部拦截法（推荐）
// 在父 ViewGroup 的 onInterceptTouchEvent 中判断
class ParentViewGroup : FrameLayout {
    private var lastX = 0f
    private var lastY = 0f

    override fun onInterceptTouchEvent(ev: MotionEvent): Boolean {
        when (ev.action) {
            MotionEvent.ACTION_DOWN -> {
                lastX = ev.x; lastY = ev.y
                return false  // DOWN 事件不拦截
            }
            MotionEvent.ACTION_MOVE -> {
                val dx = abs(ev.x - lastX)
                val dy = abs(ev.y - lastY)
                // 水平滑动 > 垂直滑动时拦截（如 ViewPager + RecyclerView）
                return dx > dy
            }
        }
        return super.onInterceptTouchEvent(ev)
    }
}

// 方案二：内部拦截法
// 在子 View 的 dispatchTouchEvent 中处理
class ChildView : View {
    override fun dispatchTouchEvent(ev: MotionEvent): Boolean {
        when (ev.action) {
            MotionEvent.ACTION_DOWN -> {
                parent.requestDisallowInterceptTouchEvent(true)  // 禁止父拦截
            }
            MotionEvent.ACTION_MOVE -> {
                if (shouldParentHandle()) {
                    parent.requestDisallowInterceptTouchEvent(false)  // 允许父拦截
                }
            }
        }
        return super.dispatchTouchEvent(ev)
    }
}
```

---

## 三、常见自定义 View 实现

### 1. 圆形头像

```kotlin
class CircleImageView @JvmOverloads constructor(
    context: Context, attrs: AttributeSet? = null
) : AppCompatImageView(context, attrs) {

    private val paint = Paint(Paint.ANTI_ALIAS_FLAG)
    private val borderPaint = Paint(Paint.ANTI_ALIAS_FLAG).apply {
        style = Paint.Style.STROKE
        strokeWidth = 4f
        color = Color.WHITE
    }

    override fun onDraw(canvas: Canvas) {
        val bitmap = drawableToBitmap(drawable) ?: return
        val shader = BitmapShader(bitmap, Shader.TileMode.CLAMP, Shader.TileMode.CLAMP)

        // 缩放到 View 大小
        val matrix = Matrix()
        val scale = width.toFloat() / bitmap.width
        matrix.setScale(scale, scale)
        shader.setLocalMatrix(matrix)

        paint.shader = shader
        val radius = width / 2f
        canvas.drawCircle(radius, radius, radius, paint)
        canvas.drawCircle(radius, radius, radius - 2f, borderPaint)
    }
}
```

### 2. 属性动画

```kotlin
// ValueAnimator
ValueAnimator.ofFloat(0f, 1f).apply {
    duration = 300
    interpolator = DecelerateInterpolator()
    addUpdateListener { animator ->
        view.alpha = animator.animatedValue as Float
    }
    start()
}

// ObjectAnimator
ObjectAnimator.ofFloat(view, "translationX", 0f, 100f).apply {
    duration = 500
    start()
}

// AnimatorSet
AnimatorSet().apply {
    playTogether(
        ObjectAnimator.ofFloat(view, "scaleX", 1f, 1.2f),
        ObjectAnimator.ofFloat(view, "scaleY", 1f, 1.2f),
        ObjectAnimator.ofFloat(view, "alpha", 1f, 0.8f)
    )
    duration = 200
    start()
}

// ViewPropertyAnimator（最简洁）
view.animate()
    .translationX(100f)
    .alpha(0.5f)
    .scaleX(1.2f)
    .setDuration(300)
    .setInterpolator(OvershootInterpolator())
    .withEndAction { /* 动画结束 */ }
    .start()
```
