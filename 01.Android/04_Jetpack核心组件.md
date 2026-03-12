# Android Jetpack 核心组件

## 一、ViewModel

### 1. 核心原理

> **面试常问**：ViewModel 如何在配置变更后存活？

```kotlin
class MyViewModel(
    private val repository: UserRepository
) : ViewModel() {

    private val _users = MutableLiveData<List<User>>()
    val users: LiveData<List<User>> = _users

    fun loadUsers() {
        viewModelScope.launch {
            _users.value = repository.getUsers()
        }
    }

    override fun onCleared() {
        super.onCleared()
        // 清理资源
    }
}
```

**存活原理**：

- ViewModel 存储在 `ViewModelStore` 中
- `ViewModelStore` 通过 `NonConfigurationInstances` 在配置变更时保留
- Activity 重建后从 `ViewModelStoreOwner` 获取同一个 `ViewModelStore`

```
配置变更（横竖屏切换）：
Activity.onDestroy() → 新 Activity.onCreate()
ViewModelStore → 跨越配置变更保留 → ViewModel 实例不变
```

### 2. ViewModel 的创建方式

```kotlin
// 1. 无参构造（默认 Factory）
val viewModel: MyViewModel by viewModels()

// 2. 有参构造（自定义 Factory）
val viewModel: MyViewModel by viewModels {
    MyViewModelFactory(repository)
}

// 3. Activity 范围共享
val sharedVM: SharedViewModel by activityViewModels()

// 4. 使用 SavedStateHandle
class SearchViewModel(
    private val savedStateHandle: SavedStateHandle
) : ViewModel() {
    val query = savedStateHandle.getLiveData<String>("query")

    fun setQuery(q: String) {
        savedStateHandle["query"] = q  // 自动保存到 SavedInstanceState
    }
}
```

### 3. ViewModel vs onSaveInstanceState

| 对比 | ViewModel | onSaveInstanceState |
|------|-----------|-------------------|
| 存活范围 | 配置变更 | 配置变更 + 进程死亡 |
| 数据大小 | 无限制 | Bundle ~1MB |
| 数据类型 | 任意对象 | 可序列化数据 |
| 使用场景 | UI 状态、大数据 | 少量关键状态 |

---

## 二、LiveData

### 1. 核心特性

```kotlin
// 生命周期感知：自动管理订阅
viewModel.users.observe(viewLifecycleOwner) { users ->
    adapter.submitList(users)
}

// Transformations
val userName: LiveData<String> = Transformations.map(userLiveData) { user ->
    "${user.firstName} ${user.lastName}"
}

val userDetails: LiveData<UserDetails> = Transformations.switchMap(userId) { id ->
    repository.getUserDetails(id)  // 返回新的 LiveData
}

// MediatorLiveData：合并多个数据源
val result = MediatorLiveData<Resource<Data>>()
result.addSource(dbSource) { data ->
    result.value = Resource.success(data)
}
result.addSource(apiSource) { response ->
    result.value = Resource.fromResponse(response)
}
```

### 2. LiveData 粘性事件问题（深入）

> **面试常问**：新注册的 Observer 会收到旧数据，如何解决？

**📌 粘性事件的本质原因**：

```
LiveData 内部维护 mVersion（每次 setValue +1）
每个 Observer 维护自己的 mLastVersion（初始为 -1）
Observer 注册时：mLastVersion(-1) < mVersion(1) → 立即回调
```

```kotlin
// ❌ 问题场景：导航页面时 Toast 重复弹出
class MyViewModel : ViewModel() {
    val showToast = MutableLiveData<String>()
    fun doSomething() { showToast.value = "操作成功" }
}
// Fragment B 返回栈时重新 observe → 又收到旧的 "操作成功" 消息！

// ✅ 解决方案1：SingleLiveEvent（只能有一个 Observer）
class SingleLiveEvent<T> : MutableLiveData<T>() {
    private val mPending = AtomicBoolean(false)

    override fun observe(owner: LifecycleOwner, observer: Observer<in T>) {
        super.observe(owner) { t ->
            if (mPending.compareAndSet(true, false)) {
                observer.onChanged(t)
            }
        }
    }

    override fun setValue(t: T?) {
        mPending.set(true)
        super.setValue(t)
    }
}

// ✅ 解决方案2：SharedFlow(replay=0)（推荐，支持多个 Observer）
class MyViewModel : ViewModel() {
    // replay=0：无回放，新订阅者收不到历史事件
    private val _event = MutableSharedFlow<String>(replay = 0)
    val event = _event.asSharedFlow()

    fun doSomething() {
        viewModelScope.launch { _event.emit("操作成功") }
    }
}

// Fragment 中安全收集
viewLifecycleOwner.lifecycleScope.launch {
    viewLifecycleOwner.repeatOnLifecycle(Lifecycle.State.STARTED) {
        viewModel.event.collect { message -> Toast.makeText(context, message, Toast.LENGTH_SHORT).show() }
    }
}
```

| 方案 | 多 Observer | 线程安全 | 推荐度 |
|------|------------|---------|-------|
| `SingleLiveEvent` | ❌ 仅 1 个 | ⚠️ | 过渡方案 |
| `SharedFlow(replay=0)` | ✅ | ✅ | **推荐** |

---

## 三、Lifecycle

### 生命周期感知组件

```kotlin
// 自定义生命周期观察者
class LocationObserver(
    private val context: Context,
    private val callback: (Location) -> Unit
) : DefaultLifecycleObserver {

    override fun onStart(owner: LifecycleOwner) {
        startLocationUpdates()
    }

    override fun onStop(owner: LifecycleOwner) {
        stopLocationUpdates()
    }
}

// 使用
lifecycle.addObserver(LocationObserver(this) { location ->
    updateMap(location)
})

// repeatOnLifecycle（推荐收集 Flow 的方式）
lifecycleScope.launch {
    repeatOnLifecycle(Lifecycle.State.STARTED) {
        viewModel.uiState.collect { state ->
            updateUI(state)
        }
    }
}
```

---

## 四、Room 数据库

### 1. 核心组件

```kotlin
// Entity
@Entity(tableName = "users")
data class UserEntity(
    @PrimaryKey(autoGenerate = true) val id: Long = 0,
    @ColumnInfo(name = "user_name") val userName: String,
    val email: String,
    @Ignore val tempData: String? = null  // 不存入数据库
)

// DAO
@Dao
interface UserDao {
    @Query("SELECT * FROM users WHERE id = :userId")
    suspend fun getUserById(userId: Long): UserEntity?

    @Query("SELECT * FROM users ORDER BY user_name ASC")
    fun getAllUsers(): Flow<List<UserEntity>>  // 响应式查询

    @Insert(onConflict = OnConflictStrategy.REPLACE)
    suspend fun insertUser(user: UserEntity): Long

    @Update
    suspend fun updateUser(user: UserEntity)

    @Delete
    suspend fun deleteUser(user: UserEntity)

    @Transaction
    @Query("SELECT * FROM users WHERE id = :userId")
    suspend fun getUserWithPosts(userId: Long): UserWithPosts
}

// Database
@Database(
    entities = [UserEntity::class, PostEntity::class],
    version = 2,
    exportSchema = true
)
@TypeConverters(Converters::class)
abstract class AppDatabase : RoomDatabase() {
    abstract fun userDao(): UserDao
    abstract fun postDao(): PostDao
}

// 数据库迁移
val MIGRATION_1_2 = object : Migration(1, 2) {
    override fun migrate(database: SupportSQLiteDatabase) {
        database.execSQL("ALTER TABLE users ADD COLUMN avatar_url TEXT")
    }
}

// 创建数据库
val db = Room.databaseBuilder(context, AppDatabase::class.java, "app.db")
    .addMigrations(MIGRATION_1_2)
    .fallbackToDestructiveMigration()  // 无迁移时销毁重建（开发用）
    .build()
```

### 2. 关系查询

```kotlin
// 一对多
data class UserWithPosts(
    @Embedded val user: UserEntity,
    @Relation(
        parentColumn = "id",
        entityColumn = "user_id"
    )
    val posts: List<PostEntity>
)
```

---

## 五、Navigation 组件

```kotlin
// nav_graph.xml 中定义导航图
// Safe Args 类型安全传参
val action = HomeFragmentDirections.actionHomeToDetail(userId = 123)
findNavController().navigate(action)

// 接收参数
class DetailFragment : Fragment() {
    private val args: DetailFragmentArgs by navArgs()

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        val userId = args.userId
    }
}

// DeepLink
findNavController().navigate(
    Uri.parse("myapp://detail/123")
)
```

---

## 六、WorkManager

```kotlin
// 定义 Worker
class SyncWorker(
    context: Context,
    params: WorkerParameters
) : CoroutineWorker(context, params) {

    override suspend fun doWork(): Result {
        return try {
            syncData()
            Result.success()
        } catch (e: Exception) {
            if (runAttemptCount < 3) Result.retry()
            else Result.failure()
        }
    }
}

// 创建请求
val constraints = Constraints.Builder()
    .setRequiredNetworkType(NetworkType.CONNECTED)
    .setRequiresBatteryNotLow(true)
    .build()

val workRequest = OneTimeWorkRequestBuilder<SyncWorker>()
    .setConstraints(constraints)
    .setBackoffCriteria(BackoffPolicy.EXPONENTIAL, 10, TimeUnit.SECONDS)
    .setInputData(workDataOf("key" to "value"))
    .build()

WorkManager.getInstance(context).enqueue(workRequest)

// 周期性任务
val periodicWork = PeriodicWorkRequestBuilder<SyncWorker>(
    15, TimeUnit.MINUTES  // 最小间隔 15 分钟
).build()

// 链式任务
WorkManager.getInstance(context)
    .beginWith(downloadWork)
    .then(processWork)
    .then(uploadWork)
    .enqueue()
```

---

## 七、Hilt 依赖注入

```kotlin
// Application
@HiltAndroidApp
class MyApp : Application()

// Module
@Module
@InstallIn(SingletonComponent::class)
object AppModule {
    @Provides
    @Singleton
    fun provideRetrofit(): Retrofit {
        return Retrofit.Builder()
            .baseUrl(BASE_URL)
            .addConverterFactory(GsonConverterFactory.create())
            .build()
    }

    @Provides
    @Singleton
    fun provideApiService(retrofit: Retrofit): ApiService {
        return retrofit.create(ApiService::class.java)
    }
}

// ViewModel 注入
@HiltViewModel
class UserViewModel @Inject constructor(
    private val repository: UserRepository,
    private val savedStateHandle: SavedStateHandle
) : ViewModel()

// Activity / Fragment
@AndroidEntryPoint
class MainActivity : AppCompatActivity() {
    private val viewModel: UserViewModel by viewModels()
}

---

## 八、Paging 3（分页加载）

> **面试常问**：Paging 3 的核心组件和数据流是什么？

### 1. 核心架构

```
Repository 层                ViewModel 层            UI 层
┌──────────────┐            ┌─────────────┐        ┌──────────────┐
│  PagingSource │ ←定义分页→ │  Pager      │ →Flow→ │  PagingData  │
│  (数据来源)   │            │  (分页配置)  │        │  Adapter     │
└──────────────┘            └─────────────┘        └──────────────┘
       ↑                           ↑
  RemoteMediator           PagingConfig
  (网络+数据库)          (页大小/预加载距离)
```

### 2. PagingSource 实现

```kotlin
// 定义分页数据源
class UserPagingSource(
    private val api: ApiService
) : PagingSource<Int, User>() {

    override suspend fun load(params: LoadParams<Int>): LoadResult<Int, User> {
        val page = params.key ?: 1  // 默认从第1页开始
        return try {
            val response = api.getUsers(page = page, size = params.loadSize)
            LoadResult.Page(
                data = response.users,
                prevKey = if (page == 1) null else page - 1,  // 第一页没有上一页
                nextKey = if (response.users.isEmpty()) null else page + 1
            )
        } catch (e: Exception) {
            LoadResult.Error(e)
        }
    }

    // 刷新时从哪页开始
    override fun getRefreshKey(state: PagingState<Int, User>): Int? {
        return state.anchorPosition?.let { anchor ->
            state.closestPageToPosition(anchor)?.prevKey?.plus(1)
                ?: state.closestPageToPosition(anchor)?.nextKey?.minus(1)
        }
    }
}
```

### 3. ViewModel + UI 接入

```kotlin
// ViewModel
class UserViewModel(private val repository: UserRepository) : ViewModel() {
    val userFlow: Flow<PagingData<User>> = Pager(
        config = PagingConfig(
            pageSize = 20,
            prefetchDistance = 5,     // 距底部5条时预加载
            enablePlaceholders = false
        )
    ) {
        UserPagingSource(repository.api)
    }.flow.cachedIn(viewModelScope)  // 跨越配置变更缓存
}

// Fragment
class UserFragment : Fragment() {
    private val adapter = UserPagingAdapter()

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        recyclerView.adapter = adapter.withLoadStateFooter(
            footer = LoadStateAdapter { adapter.retry() }
        )

        viewLifecycleOwner.lifecycleScope.launch {
            viewModel.userFlow.collectLatest { pagingData ->
                adapter.submitData(pagingData)
            }
        }

        // 监听加载状态
        adapter.addLoadStateListener { loadState ->
            when (loadState.refresh) {
                is LoadState.Loading -> showLoading()
                is LoadState.Error -> showError((loadState.refresh as LoadState.Error).error)
                is LoadState.NotLoading -> hideLoading()
            }
        }
    }
}
```

### 4. RemoteMediator（网络 + Room 组合分页）

```kotlin
// 适合：先展示本地缓存，后台静默同步网络数据
@OptIn(ExperimentalPagingApi::class)
class UserRemoteMediator(
    private val db: AppDatabase,
    private val api: ApiService
) : RemoteMediator<Int, UserEntity>() {

    override suspend fun load(
        loadType: LoadType,
        state: PagingState<Int, UserEntity>
    ): MediatorResult {
        val page = when (loadType) {
            LoadType.REFRESH -> 1
            LoadType.PREPEND -> return MediatorResult.Success(endOfPaginationReached = true)
            LoadType.APPEND -> getNextPage() ?: return MediatorResult.Success(endOfPaginationReached = true)
        }
        return try {
            val response = api.getUsers(page, state.config.pageSize)
            db.withTransaction {
                if (loadType == LoadType.REFRESH) db.userDao().clearAll()
                db.userDao().insertAll(response.users.map { it.toEntity() })
            }
            MediatorResult.Success(endOfPaginationReached = response.users.isEmpty())
        } catch (e: Exception) {
            MediatorResult.Error(e)
        }
    }
}
```

> **💡 面试回答模板**：
> "Paging 3 的核心是 `PagingSource`（定义how to加载数据）、`Pager`（分页配置）、`PagingData`（封装分页数据的流）。数据流是：`PagingSource → Pager → Flow<PagingData> → PagingDataAdapter`。对于需要网络+本地缓存的场景，可以用 `RemoteMediator` 协调，Room 作为 Single Source of Truth，网络数据写入 Room 再由 PagingSource 从 Room 读取。"

---

## 九、DataStore（替代 SharedPreferences）

> **面试常问**：为什么 DataStore 要替代 SharedPreferences？

### 1. 对比核心差异

| 对比 | SharedPreferences | DataStore (Preferences) | DataStore (Proto) |
|------|------------------|------------------------|------------------|
| 线程安全 | ❌（`apply` 可能丢数据） | ✅（基于 Flow/协程） | ✅ |
| 异常处理 | ❌（静默失败） | ✅（抛出 IOException） | ✅ |
| 类型安全 | ❌（getString 可能 NPE） | ⚠️（仍用键值对） | ✅（Proto schema） |
| 数据迁移 | ✅ | ✅ | ✅ |
| 事务支持 | ❌ | ✅（原子更新） | ✅ |

### 2. Preferences DataStore 使用

```kotlin
// 创建 DataStore（Application 级别）
val Context.dataStore: DataStore<Preferences> by preferencesDataStore(name = "settings")

// 定义 Keys
object PreferenceKeys {
    val DARK_MODE = booleanPreferencesKey("dark_mode")
    val USER_NAME = stringPreferencesKey("user_name")
    val FONT_SIZE = intPreferencesKey("font_size")
}

// 读取（返回 Flow，自动监听变化）
val darkModeFlow: Flow<Boolean> = context.dataStore.data
    .catch { e ->
        if (e is IOException) emit(emptyPreferences())  // 出错时给默认值
        else throw e
    }
    .map { preferences ->
        preferences[PreferenceKeys.DARK_MODE] ?: false
    }

// 写入（挂起函数，确保原子性）
suspend fun setDarkMode(enabled: Boolean) {
    context.dataStore.edit { preferences ->
        preferences[PreferenceKeys.DARK_MODE] = enabled
    }
}
```

### 3. 在 ViewModel 中使用

```kotlin
class SettingsViewModel(private val dataStore: DataStore<Preferences>) : ViewModel() {

    val darkMode: StateFlow<Boolean> = dataStore.data
        .map { it[PreferenceKeys.DARK_MODE] ?: false }
        .stateIn(
            scope = viewModelScope,
            started = SharingStarted.WhileSubscribed(5000),
            initialValue = false
        )

    fun toggleDarkMode(enabled: Boolean) {
        viewModelScope.launch {
            dataStore.edit { it[PreferenceKeys.DARK_MODE] = enabled }
        }
    }
}
```

> **📌 SharedPreferences 的致命问题**：
> - `apply()` 将写入放到后台线程，但在 `onStop()` 等生命周期 `waitForEnd()` 时可能阻塞主线程
> - 不支持协程/Flow，需要手动注册 `OnSharedPreferenceChangeListener`
> - `getInt()` 对不存在的 key 返回默认值，容易产生隐蔽 bug
```
