# Kotlin 高阶函数与 Lambda 表达式

## 一、高阶函数基础

高阶函数是以**函数作为参数**或**返回值为函数**的函数。

### 1. 函数类型声明

```kotlin
// 函数类型语法：(参数类型列表) -> 返回类型
val onClick: () -> Unit = { println("clicked") }
val transform: (String) -> Int = { it.length }
val combine: (Int, Int) -> Int = { a, b -> a + b }

// 带接收者的函数类型（扩展函数类型）
val builderAction: StringBuilder.() -> Unit = {
    append("Hello")  // this 指向 StringBuilder
    append(" World")
}
```

### 2. 高阶函数定义

```kotlin
// 函数作为参数
fun <T> List<T>.customFilter(predicate: (T) -> Boolean): List<T> {
    val result = mutableListOf<T>()
    for (item in this) {
        if (predicate(item)) result.add(item)
    }
    return result
}

// 函数作为返回值
fun getComparator(order: String): (String, String) -> Int {
    return when (order) {
        "asc" -> { a, b -> a.compareTo(b) }
        "desc" -> { a, b -> b.compareTo(a) }
        else -> { _, _ -> 0 }
    }
}
```

---

## 二、Lambda 表达式

### 1. Lambda 语法规则

```kotlin
// 完整语法
val sum: (Int, Int) -> Int = { x: Int, y: Int -> x + y }

// 类型推断
val sum2 = { x: Int, y: Int -> x + y }

// 单参数使用 it
val doubled = listOf(1, 2, 3).map { it * 2 }

// 最后一个参数是 Lambda 时，可放到括号外
listOf(1, 2, 3).filter { it > 1 }.map { it * 2 }

// 解构声明
mapOf("a" to 1).forEach { (key, value) ->
    println("$key -> $value")
}

// 不使用的参数用 _ 代替
mapOf("a" to 1).forEach { (_, value) ->
    println(value)
}
```

### 2. Lambda 中的 return

> **面试高频考点**：Lambda 中 `return` 的行为

```kotlin
// 非局部返回：从包含 Lambda 的函数返回（仅 inline 函数）
fun findFirst(list: List<Int>): Int? {
    list.forEach {
        if (it > 5) return it  // 从 findFirst 返回
    }
    return null
}

// 局部返回：使用标签
fun printFiltered(list: List<Int>) {
    list.forEach label@{
        if (it < 0) return@label  // 仅跳过当前迭代
        println(it)
    }
}

// 等价写法：使用函数名作为隐式标签
fun printFiltered2(list: List<Int>) {
    list.forEach {
        if (it < 0) return@forEach
        println(it)
    }
}

// 匿名函数的 return 总是局部的
fun printFiltered3(list: List<Int>) {
    list.forEach(fun(value) {
        if (value < 0) return  // 仅从匿名函数返回
        println(value)
    })
}
```

---

## 三、内联函数（inline）

> **面试核心考点**：inline 函数的原理和使用场景

### 1. 为什么需要 inline？

**Lambda 的性能开销**：

- 每个 Lambda 编译为一个匿名内部类
- 如果捕获外部变量，每次调用都创建新对象
- 间接调用带来性能损失

```kotlin
// 不使用 inline
fun <T> measure(block: () -> T): T {
    val start = System.nanoTime()
    val result = block()  // 间接调用
    println("耗时: ${System.nanoTime() - start}ns")
    return result
}
// 编译后：创建 Function0 匿名类对象

// 使用 inline：Lambda 体在调用处展开
inline fun <T> measureInline(block: () -> T): T {
    val start = System.nanoTime()
    val result = block()  // 编译后直接展开代码
    println("耗时: ${System.nanoTime() - start}ns")
    return result
}
```

### 2. noinline 与 crossinline

```kotlin
// noinline：阻止某个参数被内联（当需要将其存储或传递给非内联函数时）
inline fun execute(
    inlined: () -> Unit,
    noinline stored: () -> Unit   // 不能被内联，因为需要存储
) {
    inlined()
    someList.add(stored)  // 存储到集合中
}

// crossinline：禁止非局部返回，但仍然内联
inline fun runOnUiThread(crossinline action: () -> Unit) {
    handler.post {
        action()  // 在另一个执行上下文中，不能非局部返回
    }
}
```

### 3. 具体化类型参数（reified）

> **面试常问**：如何在运行时获取泛型类型？

```kotlin
// 普通泛型：类型擦除，运行时无法获取 T 的类型
fun <T> isType(value: Any): Boolean {
    // return value is T  // 编译错误！类型擦除
    return false
}

// reified + inline：保留类型信息
inline fun <reified T> isType(value: Any): Boolean {
    return value is T  // ✅ 编译通过！
}

// 实际应用：简化 startActivity
inline fun <reified T : Activity> Context.startActivity(
    vararg pairs: Pair<String, Any?>
) {
    val intent = Intent(this, T::class.java)
    pairs.forEach { (key, value) ->
        when (value) {
            is String -> intent.putExtra(key, value)
            is Int -> intent.putExtra(key, value)
            // ...
        }
    }
    startActivity(intent)
}

// 使用
startActivity<DetailActivity>("id" to 123, "title" to "Hello")
```

---

## 四、常用高阶函数

### 1. 集合操作

```kotlin
val numbers = listOf(1, 2, 3, 4, 5, 6, 7, 8, 9, 10)

// 变换
numbers.map { it * 2 }            // [2, 4, 6, 8, 10, 12, 14, 16, 18, 20]
numbers.flatMap { listOf(it, -it) } // [1, -1, 2, -2, ...]

// 过滤
numbers.filter { it % 2 == 0 }    // [2, 4, 6, 8, 10]
numbers.filterNot { it > 5 }      // [1, 2, 3, 4, 5]
numbers.partition { it % 2 == 0 } // Pair([2,4,6,8,10], [1,3,5,7,9])

// 聚合
numbers.reduce { acc, i -> acc + i }      // 55
numbers.fold(100) { acc, i -> acc + i }   // 155
numbers.groupBy { if (it % 2 == 0) "偶数" else "奇数" }

// 查找
numbers.first { it > 5 }          // 6
numbers.firstOrNull { it > 100 }  // null
numbers.any { it > 5 }            // true
numbers.all { it > 0 }            // true
numbers.none { it < 0 }           // true

// 排列
numbers.sortedBy { -it }          // [10, 9, 8, ...]
numbers.sortedWith(compareBy { it % 3 })
```

### 2. Sequence（序列）vs List

> **面试常问**：什么时候用 Sequence 而不是 List？

```kotlin
// List：每步操作创建中间集合（急切求值）
listOf(1, 2, 3, 4, 5)
    .map { it * 2 }       // 创建新 List [2, 4, 6, 8, 10]
    .filter { it > 5 }    // 创建新 List [6, 8, 10]
    .first()               // 6

// Sequence：惰性求值，无中间集合
listOf(1, 2, 3, 4, 5)
    .asSequence()
    .map { it * 2 }       // 不执行
    .filter { it > 5 }    // 不执行
    .first()               // 才开始：1→2(×), 2→4(×), 3→6(✓) 返回

// generateSequence
val fibonacci = generateSequence(Pair(0, 1)) { Pair(it.second, it.first + it.second) }
    .map { it.first }
    .take(10)
    .toList()  // [0, 1, 1, 2, 3, 5, 8, 13, 21, 34]
```

| 对比 | `Iterable`（List） | `Sequence` |
|------|-------------------|-----------|
| 求值方式 | 急切（每步都执行） | 惰性（终端操作触发） |
| 中间集合 | 每步创建新集合 | 无中间集合 |
| 适用场景 | 小集合、简单操作 | 大集合、多步操作链 |
| 性能 | 小数据量更好 | 大数据量更好 |

---

## 五、函数引用

```kotlin
// 顶层函数引用
fun isOdd(x: Int) = x % 2 != 0
val odds = listOf(1, 2, 3).filter(::isOdd)

// 成员函数引用
class Person(val name: String, val age: Int)
val names = people.map(Person::name)         // 属性引用
val adults = people.filter { it.age >= 18 }

// 构造函数引用
val factory: (String, Int) -> Person = ::Person

// 绑定引用（Kotlin 1.1+）
val isAdult: (Person) -> Boolean = { it.age >= 18 }
val lengthOfHello = "Hello"::length  // 绑定到具体实例
```

---

## 六、DSL 构建（领域特定语言）

> **面试加分项**：展示对 Kotlin DSL 的理解

```kotlin
// 利用带接收者的 Lambda 构建 DSL
class Html {
    private val children = mutableListOf<String>()

    fun body(init: Body.() -> Unit) {
        val body = Body()
        body.init()
        children.add(body.render())
    }

    fun render() = "<html>${children.joinToString("")}</html>"
}

class Body {
    private val children = mutableListOf<String>()

    fun p(text: String) { children.add("<p>$text</p>") }
    fun h1(text: String) { children.add("<h1>$text</h1>") }

    fun render() = "<body>${children.joinToString("")}</body>"
}

fun html(init: Html.() -> Unit): Html {
    val html = Html()
    html.init()
    return html
}

// 使用
val page = html {
    body {
        h1("标题")
        p("这是一段文字")
    }
}

// Android 中的实际应用：Anko / Jetpack Compose
// Gradle Kotlin DSL
dependencies {
    implementation("androidx.core:core-ktx:1.9.0")
}
```
