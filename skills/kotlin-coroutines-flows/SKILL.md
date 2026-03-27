---
name: kotlin-coroutines-flows
description: Android 和 KMP 的 Kotlin Coroutines 和 Flow 模式 — 结构化并发、Flow 操作符、StateFlow、错误处理和测试。
origin: ECC
---

# Kotlin Coroutines & Flows

用于 Android 和 Kotlin Multiplatform 项目中的结构化并发、基于 Flow 的响应式流和协程测试的模式。

## 何时启用

- 使用 Kotlin 协程编写异步代码
- 使用 Flow、StateFlow 或 SharedFlow 进行响应式数据处理
- 处理并发操作（并行加载、防抖、重试）
- 测试协程和 Flow
- 管理协程作用域和取消

## 结构化并发

### 作用域层次结构

```
Application
  └── viewModelScope (ViewModel)
        └── coroutineScope { } (结构化子作用域)
              ├── async { } (并发任务)
              └── async { } (并发任务)
```

始终使用结构化并发 — 永远不要用 `GlobalScope`：

```kotlin
// 坏
GlobalScope.launch { fetchData() }

// 好 — 作用域限定在 ViewModel 生命周期
viewModelScope.launch { fetchData() }

// 好 — 作用域限定在 composable 生命周期
LaunchedEffect(key) { fetchData() }
```

### 并行分解

使用 `coroutineScope` + `async` 进行并行工作：

```kotlin
suspend fun loadDashboard(): Dashboard = coroutineScope {
    val items = async { itemRepository.getRecent() }
    val stats = async { statsRepository.getToday() }
    val profile = async { userRepository.getCurrent() }
    Dashboard(
        items = items.await(),
        stats = stats.await(),
        profile = profile.await()
    )
}
```

### SupervisorScope

当子任务失败不应取消兄弟任务时使用 `supervisorScope`：

```kotlin
suspend fun syncAll() = supervisorScope {
    launch { syncItems() }       // 这里的失败不会取消 syncStats
    launch { syncStats() }
    launch { syncSettings() }
}
```

## Flow 模式

### 冷 Flow — 一次性到流的转换

```kotlin
fun observeItems(): Flow<List<Item>> = flow {
    // 每当数据库变化时重新发射
    itemDao.observeAll()
        .map { entities -> entities.map { it.toDomain() } }
        .collect { emit(it) }
}
```

### StateFlow 用于 UI 状态

```kotlin
class DashboardViewModel(
    observeProgress: ObserveUserProgressUseCase
) : ViewModel() {
    val progress: StateFlow<UserProgress> = observeProgress()
        .stateIn(
            scope = viewModelScope,
            started = SharingStarted.WhileSubscribed(5_000),
            initialValue = UserProgress.EMPTY
        )
}
```

`WhileSubscribed(5_000)` 在最后一个订阅者离开后保持上游活跃 5 秒 — 在不重启的情况下经受配置更改。

### 组合多个 Flow

```kotlin
val uiState: StateFlow<HomeState> = combine(
    itemRepository.observeItems(),
    settingsRepository.observeTheme(),
    userRepository.observeProfile()
) { items, theme, profile ->
    HomeState(items = items, theme = theme, profile = profile)
}.stateIn(viewModelScope, SharingStarted.WhileSubscribed(5_000), HomeState())
```

### Flow 操作符

```kotlin
// 搜索输入防抖
searchQuery
    .debounce(300)
    .distinctUntilChanged()
    .flatMapLatest { query -> repository.search(query) }
    .catch { emit(emptyList()) }
    .collect { results -> _state.update { it.copy(results = results) } }

// 带指数退避的重试
fun fetchWithRetry(): Flow<Data> = flow { emit(api.fetch()) }
    .retryWhen { cause, attempt ->
        if (cause is IOException && attempt < 3) {
            delay(1000L * (1 shl attempt.toInt()))
            true
        } else {
            false
        }
    }
```

### SharedFlow 用于一次性事件

```kotlin
class ItemListViewModel : ViewModel() {
    private val _effects = MutableSharedFlow<Effect>()
    val effects: SharedFlow<Effect> = _effects.asSharedFlow()

    sealed interface Effect {
        data class ShowSnackbar(val message: String) : Effect
        data class NavigateTo(val route: String) : Effect
    }

    private fun deleteItem(id: String) {
        viewModelScope.launch {
            repository.delete(id)
            _effects.emit(Effect.ShowSnackbar("Item deleted"))
        }
    }
}

// 在 Composable 中收集
LaunchedEffect(Unit) {
    viewModel.effects.collect { effect ->
        when (effect) {
            is Effect.ShowSnackbar -> snackbarHostState.showSnackbar(effect.message)
            is Effect.NavigateTo -> navController.navigate(effect.route)
        }
    }
}
```

## Dispatchers

```kotlin
// CPU 密集型工作
withContext(Dispatchers.Default) { parseJson(largePayload) }

// IO 密集型工作
withContext(Dispatchers.IO) { database.query() }

// 主线程（UI）— viewModelScope 中的默认值
withContext(Dispatchers.Main) { updateUi() }
```

在 KMP 中，使用 `Dispatchers.Default` 和 `Dispatchers.Main`（所有平台可用）。`Dispatchers.IO` 仅限 JVM/Android — 在其他平台上使用 `Dispatchers.Default` 或通过 DI 提供。

## 取消

### 协作式取消

长时间运行的循环必须检查取消：

```kotlin
suspend fun processItems(items: List<Item>) = coroutineScope {
    for (item in items) {
        ensureActive()  // 如果被取消则抛出 CancellationException
        process(item)
    }
}
```

### 使用 try/finally 清理

```kotlin
viewModelScope.launch {
    try {
        _state.update { it.copy(isLoading = true) }
        val data = repository.fetch()
        _state.update { it.copy(data = data) }
    } finally {
        _state.update { it.copy(isLoading = false) }  // 始终运行，即使取消时
    }
}
```

## 测试

### 使用 Turbine 测试 StateFlow

```kotlin
@Test
fun `search updates item list`() = runTest {
    val fakeRepository = FakeItemRepository().apply { emit(testItems) }
    val viewModel = ItemListViewModel(GetItemsUseCase(fakeRepository))

    viewModel.state.test {
        assertEquals(ItemListState(), awaitItem())  // 初始状态

        viewModel.onSearch("query")
        val loading = awaitItem()
        assertTrue(loading.isLoading)

        val loaded = awaitItem()
        assertFalse(loaded.isLoading)
        assertEquals(1, loaded.items.size)
    }
}
```

### 使用 TestDispatcher 测试

```kotlin
@Test
fun `parallel load completes correctly`() = runTest {
    val viewModel = DashboardViewModel(
        itemRepo = FakeItemRepo(),
        statsRepo = FakeStatsRepo()
    )

    viewModel.load()
    advanceUntilIdle()

    val state = viewModel.state.value
    assertNotNull(state.items)
    assertNotNull(state.stats)
}
```

### Faking Flow

```kotlin
class FakeItemRepository : ItemRepository {
    private val _items = MutableStateFlow<List<Item>>(emptyList())

    override fun observeItems(): Flow<List<Item>> = _items

    fun emit(items: List<Item>) { _items.value = items }

    override suspend fun getItemsByCategory(category: String): Result<List<Item>> {
        return Result.success(_items.value.filter { it.category == category })
    }
}
```

## 要避免的反模式

- 使用 `GlobalScope` — 泄漏协程，无结构化取消
- 在 `init {}` 中收集 Flow 而没有作用域 — 使用 `viewModelScope.launch`
- 使用带有可变集合的 `MutableStateFlow` — 始终使用不可变副本：`_state.update { it.copy(list = it.list + newItem) }`
- 捕获 `CancellationException` — 让它传播以正确取消
- 使用 `flowOn(Dispatchers.Main)` 来收集 — 收集的调度器是调用者的调度器
- 在 `@Composable` 中创建 `Flow` 而没有 `remember` — 每次重组都会重新创建 flow

## 参考

参见技能：`compose-multiplatform-patterns` 了解 Flow 的 UI 消费。
参见技能：`android-clean-architecture` 了解协程在各层中的位置。
