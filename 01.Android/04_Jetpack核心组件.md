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

### 2. LiveData 粘性事件问题

> **面试常问**：新注册的 Observer 会收到旧数据，如何解决？

```kotlin
// 问题：LiveData 默认是粘性的
// 新 Observer 注册时会立即收到最后一次 setValue 的值

// 解决方案1：使用 SingleLiveEvent（Google 示例）
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

// 解决方案2：使用 SharedFlow（推荐）
class MyViewModel : ViewModel() {
    private val _event = MutableSharedFlow<Event>()
    val event = _event.asSharedFlow()  // 无 replay，不粘性
}
```

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
```
