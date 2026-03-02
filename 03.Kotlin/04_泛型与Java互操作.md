# Kotlin 泛型与类型系统进阶

## 一、泛型基础

### 1. 泛型类与泛型函数

```kotlin
// 泛型类
class Box<T>(val value: T)

// 泛型函数
fun <T> singletonList(item: T): List<T> = listOf(item)

// 泛型接口
interface Repository<T> {
    fun findById(id: Long): T?
    fun findAll(): List<T>
    fun save(entity: T)
}
```

### 2. 类型约束

```kotlin
// 上界约束
fun <T : Comparable<T>> sort(list: List<T>) { /* ... */ }

// 多重约束
fun <T> copyWhenGreater(list: List<T>, threshold: T): List<String>
    where T : CharSequence,
          T : Comparable<T> {
    return list.filter { it > threshold }.map { it.toString() }
}
```

---

## 二、型变（Variance）

> **面试核心考点**：`in`、`out`、`*` 的含义和使用场景

### 1. 协变（`out` — 生产者）

```kotlin
// out T：只能作为输出（返回值），不能作为输入（参数）
interface Producer<out T> {
    fun produce(): T
    // fun consume(item: T)  // ❌ 编译错误！out 位置不能作为参数
}

// 类比 Java 的 ? extends T
// List<out T> 是协变的：List<String> 是 List<Any> 的子类型
val strings: List<String> = listOf("a", "b")
val anys: List<Any> = strings  // ✅ 合法
```

### 2. 逆变（`in` — 消费者）

```kotlin
// in T：只能作为输入（参数），不能作为输出（返回值）
interface Consumer<in T> {
    fun consume(item: T)
    // fun produce(): T  // ❌ 编译错误！in 位置不能作为返回值
}

// 类比 Java 的 ? super T
// Comparable<in T> 是逆变的：Comparable<Any> 是 Comparable<String> 的子类型
val anyComparable: Comparable<Any> = Comparable { _, _ -> 0 }
val stringComparable: Comparable<String> = anyComparable  // ✅ 合法
```

### 3. PECS 原则（Producer Extends, Consumer Super）

| | Kotlin | Java | 位置 |
|---|--------|------|------|
| 协变 | `out T` | `? extends T` | 只读/生产者 |
| 逆变 | `in T` | `? super T` | 只写/消费者 |
| 不变 | `T` | `T` | 读写均可 |

### 4. 星投影（`*`）

```kotlin
// 类似 Java 的 ?
// 当你不关心具体类型时使用
fun printAll(list: List<*>) {
    list.forEach { println(it) }  // 每个元素当作 Any? 处理
}

// MutableList<*> 等价于 MutableList<out Any?>
// 可以读（Any?），不可写
```

---

## 三、类型擦除与具体化

### 1. 类型擦除

```kotlin
// JVM 运行时泛型信息被擦除
fun <T> isListOf(list: List<Any>): Boolean {
    // return list is List<T>  // ❌ 无法检查泛型参数
    return list is List<*>     // ✅ 只能检查是否是 List
}

// 类型擦除导致的问题
fun <T> List<T>.toArrayOf(): Array<T> {
    // return Array(size) { this[it] }  // ❌ 无法创建 Array<T>
    throw UnsupportedOperationException()
}
```

### 2. reified 具体化类型参数

```kotlin
// 通过 inline + reified 在运行时保留类型信息
inline fun <reified T> Gson.fromJson(json: String): T {
    return fromJson(json, T::class.java)
}

// 使用
val user: User = gson.fromJson(jsonString)

// 常见应用
inline fun <reified T : Activity> Context.startActivity() {
    startActivity(Intent(this, T::class.java))
}

inline fun <reified T> Bundle.getParcelableCompat(key: String): T? {
    return if (Build.VERSION.SDK_INT >= 33) {
        getParcelable(key, T::class.java)
    } else {
        @Suppress("DEPRECATION")
        getParcelable(key) as? T
    }
}
```

---

## 四、Kotlin 与 Java 互操作

> **面试常问**：Kotlin 和 Java 混合开发注意事项

### 1. 从 Java 调用 Kotlin

```kotlin
// Kotlin 文件
// @file:JvmName("StringUtils")  // 指定生成的 Java 类名
package com.example

fun String.isEmail(): Boolean = contains("@")

object ApiClient {
    @JvmStatic   // 生成真正的 static 方法
    fun getInstance(): ApiClient = this

    const val BASE_URL = "https://api.example.com"  // 编译为 static final
}

data class User(
    @JvmField val name: String,      // 直接暴露字段，无 getter
    val age: Int                      // 生成 getAge() 方法
) {
    @JvmOverloads                     // 生成重载方法
    fun greet(greeting: String = "Hello", suffix: String = "!") {
        println("$greeting, $name$suffix")
    }
}
```

```java
// 从 Java 调用
StringUtils.isEmail("test@test.com");
ApiClient.getInstance();
String url = ApiClient.BASE_URL;

User user = new User("John", 25);
user.greet();                    // 使用默认参数
user.greet("Hi");                // 部分默认
user.greet("Hi", "?");           // 全部指定
```

### 2. 从 Kotlin 调用 Java

```kotlin
// 平台类型：Java 返回的类型在 Kotlin 中标记为 T!（可空也可能非空）
val list: MutableList<String> = ArrayList()  // Java ArrayList

// SAM 转换：Java 的单抽象方法接口可使用 Lambda
button.setOnClickListener { view ->
    // 等价于 new View.OnClickListener() { ... }
}

// 处理 Java 的 checked exception
// Kotlin 没有 checked exception，但调用 Java 时要注意
try {
    Files.readAllLines(path)
} catch (e: IOException) {
    // 需要手动处理
}
```

### 3. 注解兼容

```kotlin
// @JvmStatic：在 companion object / object 中生成 static 方法
// @JvmField：暴露为 Java 字段而非生成 getter/setter
// @JvmOverloads：为默认参数生成重载方法
// @JvmName：指定 JVM 上的方法/类名（解决签名冲突）
// @Throws：声明 checked exception（以便 Java 调用者处理）

@Throws(IOException::class)
fun readFile(path: String): String {
    return File(path).readText()
}
```
