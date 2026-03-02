# Android 组件化与 Gradle 构建

## 一、组件化架构

### 1. 模块化 vs 组件化

| 对比 | 模块化 | 组件化 |
|------|--------|--------|
| 粒度 | 按功能分层（网络层、数据层） | 按业务拆分（订单、用户、支付） |
| 独立运行 | 不能 | 组件可独立运行调试 |
| 耦合度 | 层间依赖 | 组件间通过路由/接口通信 |

### 2. 标准分层架构

```
app (壳工程)
├── feature-home     (首页业务)
├── feature-order    (订单业务)
├── feature-user     (用户业务)
├── feature-payment  (支付业务)
├── lib-common       (公共组件：BaseActivity、工具类)
├── lib-network      (网络层)
├── lib-ui           (UI 组件库)
└── lib-base         (基础库)
```

### 3. 组件独立运行

```groovy
// gradle.properties
isModuleMode=false   // true=独立 App，false=Library

// feature-home/build.gradle.kts
if (project.property("isModuleMode").toString().toBoolean()) {
    apply(plugin = "com.android.application")
} else {
    apply(plugin = "com.android.library")
}

android {
    sourceSets {
        getByName("main") {
            if (project.property("isModuleMode").toString().toBoolean()) {
                manifest.srcFile("src/main/debug/AndroidManifest.xml")
            } else {
                manifest.srcFile("src/main/AndroidManifest.xml")
            }
        }
    }
}
```

### 4. 组件间通信

```kotlin
// 方案1：ARouter 路由
@Route(path = "/user/profile")
class UserProfileActivity : AppCompatActivity()

// 页面跳转
ARouter.getInstance()
    .build("/user/profile")
    .withLong("userId", 123)
    .navigation()

// 服务暴露
interface IUserService : IProvider {
    fun getUserInfo(): UserInfo
}

@Route(path = "/user/service")
class UserServiceImpl : IUserService {
    override fun getUserInfo(): UserInfo { /* ... */ }
}

// 获取服务
val userService = ARouter.getInstance()
    .build("/user/service")
    .navigation() as IUserService

// 方案2：接口下沉 + 依赖注入
// lib-common 中定义接口
interface UserService {
    fun getUserInfo(): UserInfo
}
// feature-user 中实现
// app 壳工程中通过 Hilt 注入
```

---

## 二、Gradle 构建基础

### 1. 构建流程

```
初始化阶段 → settings.gradle 确定参与构建的项目
    ↓
配置阶段 → 执行所有 build.gradle，生成任务依赖图
    ↓
执行阶段 → 执行需要的任务（编译、打包）
```

### 2. 依赖管理

```kotlin
// build.gradle.kts
dependencies {
    implementation("androidx.core:core-ktx:1.12.0")   // 编译+运行时
    api("com.google.gson:gson:2.10.1")                // 编译+运行时+传递
    compileOnly("javax.annotation:jsr250-api:1.0")    // 仅编译
    runtimeOnly("com.mysql:mysql-connector-j:8.0")    // 仅运行时
    testImplementation("junit:junit:4.13.2")          // 仅测试
    debugImplementation("com.squareup.leakcanary:...")  // 仅 debug

    // kapt → ksp（注解处理器，ksp 更快）
    ksp("com.google.dagger:dagger-compiler:2.48")
}
```

| 对比 | `implementation` | `api` |
|------|-----------------|-------|
| 传递性 | 不传递 | 传递给依赖者 |
| 编译隔离 | 隔离 | 暴露 |
| 编译速度 | 快（变化不影响下游） | 慢（变化波及下游） |
| 推荐 | 默认选择 | 仅在需要暴露时 |

### 3. Version Catalog（版本目录）

```toml
# gradle/libs.versions.toml
[versions]
kotlin = "1.9.22"
compose-bom = "2024.01.00"
retrofit = "2.9.0"

[libraries]
kotlin-stdlib = { module = "org.jetbrains.kotlin:kotlin-stdlib", version.ref = "kotlin" }
retrofit-core = { module = "com.squareup.retrofit2:retrofit", version.ref = "retrofit" }
retrofit-gson = { module = "com.squareup.retrofit2:converter-gson", version.ref = "retrofit" }
compose-bom = { module = "androidx.compose:compose-bom", version.ref = "compose-bom" }

[bundles]
retrofit = ["retrofit-core", "retrofit-gson"]

[plugins]
kotlin-android = { id = "org.jetbrains.kotlin.android", version.ref = "kotlin" }
```

```kotlin
// 使用
dependencies {
    implementation(libs.kotlin.stdlib)
    implementation(libs.bundles.retrofit)
    implementation(platform(libs.compose.bom))
}
```

### 4. 构建优化

```properties
# gradle.properties
org.gradle.parallel=true          # 并行编译
org.gradle.caching=true           # 构建缓存
org.gradle.daemon=true            # Gradle 守护进程
org.gradle.jvmargs=-Xmx4g        # JVM 堆内存

# Kotlin 编译
kotlin.incremental=true           # 增量编译
kotlin.code.style=official
```

---

## 三、多渠道打包

```kotlin
android {
    flavorDimensions += "env"
    productFlavors {
        create("dev") {
            dimension = "env"
            applicationIdSuffix = ".dev"
            buildConfigField("String", "BASE_URL", "\"https://dev-api.example.com\"")
        }
        create("staging") {
            dimension = "env"
            applicationIdSuffix = ".staging"
            buildConfigField("String", "BASE_URL", "\"https://staging-api.example.com\"")
        }
        create("prod") {
            dimension = "env"
            buildConfigField("String", "BASE_URL", "\"https://api.example.com\"")
        }
    }

    buildTypes {
        release {
            isMinifyEnabled = true
            proguardFiles(getDefaultProguardFile("proguard-android-optimize.txt"), "proguard-rules.pro")
            signingConfig = signingConfigs.getByName("release")
        }
    }
}
```

---

## 四、ProGuard / R8

```proguard
# 基本规则
-keep class com.example.model.** { *; }          # 保留 model 类
-keepclassmembers class ** {                      # 保留 Gson 序列化字段
    @com.google.gson.annotations.SerializedName <fields>;
}
-keep class * implements android.os.Parcelable {  # 保留 Parcelable
    public static final android.os.Parcelable$Creator *;
}

# 常见问题
# 反射调用的类被混淆 → -keep
# Gson/Moshi 解析失败 → 保留 model 类
# WebView JS 交互类 → -keepclassmembers
```

| 对比 | ProGuard | R8 |
|------|---------|-----|
| 默认 | Android 旧版 | AGP 3.4+ 默认 |
| 速度 | 慢 | 快 |
| 优化 | 基础 | 更强的优化（内联、枚举脱糖） |
| 兼容性 | 使用 ProGuard 规则 | 完全兼容 ProGuard 规则 |
