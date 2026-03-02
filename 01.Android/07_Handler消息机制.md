# Android Handler 消息机制

## 一、核心组件

> **面试最高频考点之一**：Handler/Looper/MessageQueue 的工作原理

### 1. 架构总览

```
                    ┌──────────────────────────┐
                    │        Looper            │
                    │  ┌──────────────────┐    │
 Handler ──send──→ │  │  MessageQueue     │    │
    ↑               │  │  ┌─┐ ┌─┐ ┌─┐    │    │
    │               │  │  │M│→│M│→│M│    │    │
    │               │  │  └─┘ └─┘ └─┘    │    │
    │               │  └──────────────────┘    │
    └──dispatch──── │         loop()           │
                    └──────────────────────────┘
```

| 组件 | 职责 |
|------|------|
| `Handler` | 发送消息（sendMessage）和处理消息（handleMessage） |
| `MessageQueue` | 消息队列（按时间排序的单链表） |
| `Looper` | 无限循环从 MessageQueue 取消息，分发给 Handler |
| `Message` | 消息载体（what、arg1、arg2、obj、target） |

### 2. 核心源码分析

#### Looper.prepare() 与 Looper.loop()

```java
// Looper.prepare()：创建 Looper 并存入 ThreadLocal
public static void prepare() {
    if (sThreadLocal.get() != null) {
        throw new RuntimeException("Only one Looper may be created per thread");
    }
    sThreadLocal.set(new Looper());  // 每个线程最多一个 Looper
}

// Looper.loop()：死循环取消息
public static void loop() {
    final Looper me = myLooper();
    final MessageQueue queue = me.mQueue;

    for (;;) {
        Message msg = queue.next(); // 可能阻塞（epoll）
        if (msg == null) return;    // null 表示退出

        msg.target.dispatchMessage(msg);  // 分发给 Handler
        msg.recycleUnchecked();            // 回收 Message
    }
}
```

#### MessageQueue.next()

```java
// 基于 epoll 机制的阻塞等待
Message next() {
    int nextPollTimeoutMillis = 0;
    for (;;) {
        nativePollOnce(ptr, nextPollTimeoutMillis);  // epoll_wait 阻塞
        synchronized (this) {
            final long now = SystemClock.uptimeMillis();
            Message msg = mMessages;
            if (msg != null && msg.when <= now) {
                mMessages = msg.next;
                return msg;  // 返回到期的消息
            }
            if (msg != null) {
                nextPollTimeoutMillis = (int)(msg.when - now);  // 计算等待时间
            } else {
                nextPollTimeoutMillis = -1;  // 无消息，无限等待
            }
        }
    }
}
```

#### Handler.dispatchMessage()

```java
public void dispatchMessage(Message msg) {
    if (msg.callback != null) {
        // post(Runnable) 方式
        msg.callback.run();
    } else {
        if (mCallback != null) {
            // 构造函数传入的 Callback
            if (mCallback.handleMessage(msg)) return;
        }
        // 子类重写的 handleMessage
        handleMessage(msg);
    }
}

// 分发优先级：
// 1. Message.callback (post Runnable)
// 2. Handler.mCallback (构造函数 Callback)
// 3. Handler.handleMessage (子类重写)
```

---

## 二、面试高频问题

### 1. 为什么主线程的 Looper.loop() 死循环不会 ANR？

> **标准回答**：
>
> - ANR 是指**消息处理超时**，而不是没有消息
> - Looper.loop() 在没有消息时通过 `epoll_wait` **阻塞等待**，不消耗 CPU
> - 有消息/事件到来时（触摸、绘制、生命周期），epoll 唤醒线程处理
> - ANR 发生在**某个消息处理时间过长**（如 5 秒内未响应输入事件）

### 2. Handler 导致的内存泄漏

```kotlin
// ❌ 内部类持有 Activity 引用
class MyActivity : AppCompatActivity() {
    val handler = object : Handler(Looper.getMainLooper()) {
        override fun handleMessage(msg: Message) {
            // 隐式持有 MyActivity.this → 内存泄漏
            updateUI()
        }
    }
}

// ✅ 方案1：静态内部类 + 弱引用
class MyActivity : AppCompatActivity() {
    class SafeHandler(activity: MyActivity) : Handler(Looper.getMainLooper()) {
        private val ref = WeakReference(activity)
        override fun handleMessage(msg: Message) {
            ref.get()?.updateUI()
        }
    }

    override fun onDestroy() {
        super.onDestroy()
        handler.removeCallbacksAndMessages(null)  // 清除所有消息
    }
}

// ✅ 方案2：使用 lifecycleScope（推荐）
lifecycleScope.launch {
    delay(3000)
    updateUI()  // 自动与生命周期关联
}
```

### 3. 子线程如何创建 Handler？

```kotlin
// 方案1：手动 prepare + loop
val thread = Thread {
    Looper.prepare()        // 创建 Looper
    val handler = Handler(Looper.myLooper()!!)
    Looper.loop()           // 开始循环（会阻塞在这里）
    // Looper.myLooper()!!.quit()  // 退出
}

// 方案2：HandlerThread（推荐）
val handlerThread = HandlerThread("background")
handlerThread.start()
val bgHandler = Handler(handlerThread.looper)
bgHandler.post { /* 在后台线程执行 */ }
// 退出时
handlerThread.quitSafely()
```

### 4. Message 复用机制

```kotlin
// 使用 obtain() 从对象池获取，避免频繁创建
val msg = Message.obtain()  // 从回收池获取（链表实现）
// 或
val msg = handler.obtainMessage(MSG_WHAT, arg1, arg2, obj)

// Message 处理完后自动回收到池中（Looper.loop 中调用 recycleUnchecked）
// 池大小上限：50
```

---

## 三、同步屏障（SyncBarrier）

> **面试加分项**

```java
// 同步屏障：让异步消息优先处理
// ViewRootImpl 中的绘制流程使用同步屏障确保绘制消息优先

// 发送屏障消息（target = null 的 Message）
MessageQueue.postSyncBarrier()

// 效果：
// 普通消息（同步消息）被阻挡
// 异步消息（msg.setAsynchronous(true)）正常处理

// UI 绘制流程：
// 1. postSyncBarrier() 插入屏障
// 2. Choreographer 发送异步的绘制消息
// 3. 绘制完成后 removeSyncBarrier()
```

---

## 四、IdleHandler

```kotlin
// 在消息队列空闲时执行（适用于延迟初始化）
Looper.myQueue().addIdleHandler {
    // 空闲时执行
    initNonCriticalSDK()
    false  // false = 只执行一次，true = 每次空闲都执行
}

// 实际应用：
// 1. 启动优化：延迟初始化非关键模块
// 2. GC 相关
// 3. Activity 动画结束后执行操作
```

---

## 五、ThreadLocal 原理

> **面试常问**：Looper 如何保证一个线程只有一个？

```java
// ThreadLocal：线程隔离的变量存储
// 每个 Thread 内部有 ThreadLocalMap
// key = ThreadLocal 对象，value = 存储的值

// Looper 使用 ThreadLocal 存储
static final ThreadLocal<Looper> sThreadLocal = new ThreadLocal<>();

// 所以 Looper.prepare() 只能调用一次
// Looper.myLooper() 返回当前线程的 Looper
```

**ThreadLocal 内存泄漏**：

- Entry 的 key 是 **弱引用**
- 如果 ThreadLocal 被回收，key 变 null，但 value 仍在
- 解决：用完调用 `threadLocal.remove()`
