# 协程与 Flow

> 级别：⭐⭐⭐ 高频。Kotlin 面试必考题。

---

## 面试官怎么问

- 「协程和线程的区别？」
- 「suspend 函数做了什么？」
- 「launch 和 async 的区别？」
- 「StateFlow 和 SharedFlow 区别？什么时候用哪个？」
- 「Flow 的背压怎么处理？」

---

## 回答框架

### 1. 协程 vs 线程

| 对比维度 | 线程 | 协程 |
|---------|------|------|
| 调度者 | OS 内核（抢占式） | 用户态（协作式） |
| 创建开销 | 大（~1MB 栈空间） | 小（~几 KB） |
| 上下文切换 | 系统调用，慢 | 函数调用级别，快 |
| 挂起 | 阻塞（占用线程） | 挂起（不占线程，可复用） |
| 数量 | 有限（几千个） | 大量（几十万个没问题） |

**后端类比**：协程 ≈ Java 21 虚拟线程 (Project Loom)。都是用户态轻量级并发单元，挂起时不阻塞底层线程。

### 2. suspend 原理

`suspend` 函数编译后会多一个 `Continuation` 参数（CPS 转换）。挂起 = 保存当前状态 + 让出线程，恢复 = 读取状态 + 继续执行。

```kotlin
suspend fun fetchData(): String {
    delay(1000)  // 挂起 1 秒，不阻塞线程
    return "data"
}

// 编译后的伪代码
fun fetchData(continuation: Continuation<String>): Any? {
    // ...
    if (continuation.label == 0) {
        continuation.label = 1
        delay(1000, continuation)  // 挂起，让出线程
        return COROUTINE_SUSPENDED
    }
    // 恢复时继续执行
    return "data"
}
```

### 3. launch vs async

```kotlin
// launch — 无返回值（Job）
val job = viewModelScope.launch {
    doSomething()
}
// 后端类比：CompletableFuture.runAsync()

// async — 有返回值（Deferred<T>）
val deferred = viewModelScope.async {
    fetchData()  // 返回结果
}
val result = deferred.await()
// 后端类比：CompletableFuture.supplyAsync().get()
```

### 4. 调度器

| 调度器 | 线程池 | 用途 |
|-------|-------|------|
| Dispatchers.Main | 主线程 | UI 更新 |
| Dispatchers.IO | 弹性线程池（64线程上限） | 网络、文件、DB |
| Dispatchers.Default | CPU 核数个线程 | 计算密集型 |
| Dispatchers.Unconfined | 无限制 | 极少用 |

### 5. StateFlow vs SharedFlow

| 对比 | StateFlow | SharedFlow |
|------|-----------|-----------|
| 类型 | 热流（有状态） | 热流（无状态/事件） |
| 初始值 | **必须提供** | 不需要 |
| 去重 | 自动去重 | 不去重 |
| 保留历史 | 只保留最新一个值 | replay 参数控制 |
| 用途 | **UI 状态** | **一次性事件** |
| 后端类比 | BehaviorSubject | PublishSubject |

```kotlin
// StateFlow — 用途：持续变化的状态
private val _uiState = MutableStateFlow<UiState>(UiState.Loading)
val uiState: StateFlow<UiState> = _uiState.asStateFlow()

// SharedFlow — 用途：一次性事件（SnackBar、导航）
private val _events = MutableSharedFlow<UiEvent>()
val events = _events.asSharedFlow()
// emit(event) — 事件不保留，无订阅者时丢失
```

### 6. 背压处理

当生产者速度 > 消费者速度时：

```kotlin
flow {
    repeat(1000) { emit(it) }
}
    .buffer(10)          // 缓存 10 个，生产者不阻塞
    .conflate()          // 只保留最新值，丢弃中间值
    .collectLatest { }   // 新值到来取消旧的 collect
```

---

## 后端类比总结

| Kotlin 协程/Flow | Java 后端 |
|-----------------|----------|
| 协程 | 虚拟线程 (Loom) / CompletableFuture |
| suspend | 阻塞 IO 的透明异步（不占用线程） |
| Dispatchsers.IO | CachedThreadPool |
| Flow (冷流) | Project Reactor Flux |
| StateFlow | BehaviorSubject / Sinks.replay().latest() |
| SharedFlow | PublishSubject / EventBus |
| viewModelScope | @SessionScope 管理的线程池 |

---

## 延伸追问

1. **withContext 和 flowOn 的区别？**
   → withContext 切换单个协程的执行线程。flowOn 切换 Flow 上游的 emit 线程。

2. **SupervisorJob 解决了什么问题？**
   → 子协程异常不影响兄弟协程。viewModelScope 默认使用 SupervisorJob，所以新闻列表加载失败不会影响 Banner 加载。

3. **collect 和 collectLatest 的区别？**
   → collect 按顺序处理每个值。collectLatest 新值到来时如果上一个处理还没完成就取消它。适合搜索：用户快速输入时只处理最新搜索词。

4. **Flow 和 LiveData 的区别？**
   → Flow 是协程原生的，支持操作符（map/filter/combine），Kotlin 多平台可用。LiveData 是 Android 特化的，只支持基本的数据绑定，不能在纯 Kotlin 代码中测试。新项目推荐用 StateFlow 替代 LiveData。
