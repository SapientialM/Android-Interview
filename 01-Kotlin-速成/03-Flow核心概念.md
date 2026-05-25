# Flow 核心概念

> 面试重点 ⭐⭐⭐。Flow 是 Kotlin 的响应式流，在 Compose 项目中用于 ViewModel 向 UI 层传递状态。
> **核心策略**：用 Java Stream / Project Reactor 类比学习。

---

## 1. Flow 是什么？

**一句话**：Flow 是 Kotlin 的**冷流**，只有在被收集（collect）时才执行，支持协程的异步数据流。

```kotlin
// 定义一个 Flow 不会立即执行
val flow = flow {
    emit(1)
    delay(1000)
    emit(2)
}

// 只有 collect 时才执行
viewModelScope.launch {
    flow.collect { value ->
        println(value)
    }
}
```

**后端类比**：

| 概念 | Project Reactor | Kotlin Flow |
|------|----------------|-------------|
| 冷流 | `Flux` / `Mono` | `Flow`（冷流） |
| 热流（有最新值） | `BehaviorSubject` / `Sinks.replay().latest()` | `StateFlow` |
| 热流（纯事件） | `PublishSubject` / `Sinks.many().multicast()` | `SharedFlow` |
| 操作符 | `map`, `filter`, `flatMap` | 同名操作符 |
| 线程切换 | `.subscribeOn(Schedulers.boundedElastic())` | `.flowOn(Dispatchers.IO)` |
| 背压 | `onBackpressureBuffer()` 等 | `buffer()`, `conflate()`, `collectLatest()` |

---

## 2. 冷流 vs 热流

```kotlin
// 冷流 — 每次 collect 重新执行
val coldFlow = flow {
    println("开始执行")
    emit(1)
    emit(2)
}
// collect 一次 → 打印 "开始执行" 一次
// collect 两次 → 打印 "开始执行" 两次
// 不 collect → 不打印 → 逻辑不执行

// 热流 (StateFlow) — 无论有没有订阅者都在运行
private val _state = MutableStateFlow(0)  // 初始值为 0
val state: StateFlow<Int> = _state        // 只读版本

_state.value = 1   // 立即更新
_state.value = 2   // 订阅者收到最新值 2
// 新订阅者会立刻收到最新值 2（不会收到 0 和 1）
```

| 对比维度 | Flow (冷流) | StateFlow (热流) |
|---------|------------|-----------------|
| 执行时机 | collect 时才执行 | 创建即可，不依赖收集者 |
| 保留历史 | 不保留 | 只保留最新值 |
| 多订阅者 | 各自独立执行 | 共享同一份数据 |
| 去重 | 不去重 | 自动去重（value 相同时不通知） |
| 主要用途 | 数据库查询、文件流 | ViewModel 暴露 UI 状态 |
| 后端类比 | Flux（冷） | BehaviorSubject（热，有最新值） |

---

## 3. StateFlow — Compose 状态管理的核心

```kotlin
// ViewModel 中
class NewsViewModel : ViewModel() {

    // 内部可变的 StateFlow
    private val _uiState = MutableStateFlow<UiState>(UiState.Loading)
    // 暴露给 UI 的只读版本
    val uiState: StateFlow<UiState> = _uiState.asStateFlow()

    fun refresh() {
        viewModelScope.launch {
            _uiState.value = UiState.Loading  // 使用 .value 而不是 emit()
            val data = repository.getNews()
            _uiState.value = UiState.Success(data)
        }
    }
}

// Compose UI 中
@Composable
fun NewsScreen(viewModel: NewsViewModel = hiltViewModel()) {
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()
    // ↑ StateFlow → Compose State，Flow 值变化时自动重组

    when (uiState) {
        is UiState.Loading -> LoadingView()
        is UiState.Success -> ContentView((uiState as UiState.Success).data)
        is UiState.Error -> ErrorView((uiState as UiState.Error).message)
        is UiState.Empty -> EmptyView()
    }
}
```

**StateFlow 优于 LiveData 的地方**：
- 是协程原生的（不是 Android 特化），纯 Kotlin 就能用
- 支持更多操作符（map, combine, filter 等）
- 初始值必须提供（LiveData 可能为 null）
- 自动去重（value 相同时不通知订阅者）

---

## 4. SharedFlow — 事件总线

```kotlin
// ViewModel 中 — 用于一次性事件（SnackBar、导航）
class NewsViewModel : ViewModel() {

    private val _events = MutableSharedFlow<UiEvent>()  // SharedFlow
    val events = _events.asSharedFlow()

    fun onItemClick(news: NewsItem) {
        viewModelScope.launch {
            _events.emit(UiEvent.NavigateToDetail(news.id))  // 事件不会被遗漏
        }
    }
}

// replay = 0: 新订阅者不会收到历史事件（默认，适合事件）
// replay = 1: 新订阅者收到最近 1 个事件
// extraBufferCapacity: 缓冲区大小（emit 太快时缓存）
```

| 对比维度 | StateFlow | SharedFlow |
|---------|----------|-----------|
| 用途 | 状态持有（持续变化的值） | 一次性事件（SnackBar、导航） |
| 初始值 | 必须提供 | 不需要 |
| 去重 | 自动去重 | 不去重 |
| 丢失事件 | 保留最新值 | 默认 replay=0，无订阅者时事件丢失 |
| 后端类比 | BehaviorSubject / State\<T\> | PublishSubject / EventBus |

**何时用哪个？** 一句话：**状态用 StateFlow，事件用 SharedFlow**。

---

## 5. Flow 常用操作符

```kotlin
// 转换类
flow.map { it * 2 }                // 一对一转换
flow.filter { it > 0 }             // 过滤
flow.take(5)                       // 只取前 5 个

// 组合类
combine(flow1, flow2) { a, b ->    // 任意一个变化就组合
    "$a and $b"
}

// 去重/节流
flow.distinctUntilChanged()        // 连续相同值只发一次
flow.debounce(300)                  // 停止发射 300ms 后才发最新值（搜索框防抖）

// 异常处理
flow.catch { e -> emit(fallback) } // 捕获上游异常并发送备用值

// 生命周期
flow.flowOn(Dispatchers.IO)        // 切换上游发射线程
flow.cancellable()                 // 允许取消

// collect 的变体（用于消费）
flow.collect { }                   // 标准收集
flow.collectLatest { }             // 新值来时取消之前的处理（适合搜索）
flow.first()                       // 只取第一个值
```

**面试常问**：`collect {}` vs `collectLatest {}` 的区别？
- `collect`：每个值都处理，按顺序等待
- `collectLatest`：新值来时如果上一个处理还没完成，取消它（适合搜索方案）

---

## 6. 背压 (Backpressure)

当生产者速度 > 消费者速度时的处理策略：

```kotlin
flow {
    repeat(1000) { emit(it) }  // 快速发 1000 个数据
}
    .buffer(10)        // 缓存 10 个，不阻塞生产者
    .conflate()        // 只保留最新值，丢弃中间值
    .collectLatest { } // 新值来时取消旧的处理
```

| 策略 | 说明 | 后端类比 |
|------|------|---------|
| 默认（无背压处理） | 生产者等消费者，慢 | — |
| `buffer(n)` | 缓冲 n 个值，生产者不阻塞 | 消息队列缓冲区 |
| `conflate()` | 只保留最新值，丢弃中间值 | Reactor 的 `onBackpressureLatest` |
| `collectLatest {}` | 新值到来取消旧的 collect | — |

---

## 7. 项目中的典型数据流

```
[网络请求] → [Repository] → [ViewModel] → [Compose UI]
   Retrofit     Flow<T>      StateFlow       collectAsState
```

```kotlin
// 完整的 Flow 链
// 1. Repository — 返回 Flow
class NewsRepository {
    suspend fun getNews(): List<NewsItem> {
        return api.getNews()
    }
}

// 2. ViewModel — Flow → StateFlow
class NewsViewModel : ViewModel() {
    private val _uiState = MutableStateFlow<UiState>(UiState.Loading)
    val uiState: StateFlow<UiState> = _uiState.asStateFlow()
}

// 3. Compose UI — StateFlow → Compose State → 重组
@Composable
fun NewsScreen(viewModel: NewsViewModel = hiltViewModel()) {
    val state by viewModel.uiState.collectAsStateWithLifecycle()
    // state 变化 → 当前 Composable 重组
}
```

---

## 面试速记

| 考点 | 回答要点 |
|------|---------|
| Flow 是什么 | 冷流，collect 时才执行，支持协程 |
| 冷流 vs 热流 | 冷流每次 collect 重新执行；热流独立于收集者 |
| StateFlow vs SharedFlow | StateFlow = 状态持有（有初始值、去重、保留最新）；SharedFlow = 事件分发（无初始值、不去重） |
| collect vs collectLatest | collectLatest 新值到来时取消之前的处理（搜索场景） |
| stateIn / shareIn | 冷流转热流：stateIn（→ StateFlow），shareIn（→ SharedFlow） |
| 与 LiveData 的区别 | StateFlow 是协程原生、支持操作符、必须有初始值、Kotlin 多平台可用 |
