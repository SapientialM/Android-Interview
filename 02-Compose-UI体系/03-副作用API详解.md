# 副作用API详解

> 面试重点 ⭐⭐⭐。「副作用」是 Compose 面试的高频词，必须清楚每个 API 的用途。
> **本质理解**：副作用 = Composable 函数之外的事情（网络请求、数据库读写、注册监听器）。

---

## 1. 什么是副作用？

**副作用** = 在 Composable 函数执行过程中，做了**不纯粹**的事情（与 UI 渲染无关的操作）。

```kotlin
@Composable
fun BadExample() {
    // ❌ 副作用 — 每次重组都会执行！
    api.fetchNews()              // 网络请求（严重！）
    val data = readFromDisk()    // 磁盘读取

    // ✅ 纯 UI — 只描述界面
    Text("Hello")
}
```

**为什么副作用不能直接写在 Composable 里？**
- Composable 会重组（被重新执行多次），副作用会重复触发
- Composable 执行顺序不确定，副作用可能在非预期时机执行

**后端类比**：就像你不应该在 Thymeleaf 模板的渲染函数里直接调 `@Service` 方法。查询数据应该在 Controller 里完成，模板只负责展示。

---

## 2. 副作用 API 速查表

| API | 用途 | 生命周期 | 后端类比 |
|-----|------|---------|---------|
| `LaunchedEffect(key)` | 在 Composable 中启动协程 | key 变化或离开 Composition 时取消 | `@PostConstruct` 里启动异步任务 |
| `DisposableEffect(key)` | 注册/注销资源 | key 变化或离开 Composition 时 dispose | `@PostConstruct` 注册 + `@PreDestroy` 注销 |
| `SideEffect` | 每次重组后执行 | 每次重组后 | — |
| `rememberCoroutineScope()` | 获取脱离 Composable 上下文的协程作用域 | 调用者的生命周期 | 手动管理的线程池 |
| `derivedStateOf` | 派生状态（减少重组） | Composition 生命周期 | SQL 视图 / 物化视图 |
| `produceState` | 非 Compose 状态 → Compose State | Composition 生命周期 | 适配器模式 |

---

## 3. LaunchedEffect — 最常用的副作用 API

```kotlin
@Composable
fun NewsScreen(viewModel: NewsViewModel = hiltViewModel()) {
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()

    // key = Unit → 只在 Composition 时执行一次
    LaunchedEffect(Unit) {
        viewModel.loadNews()
    }

    // key = searchQuery → searchQuery 变化时重新执行，取消上一个
    var searchQuery by remember { mutableStateOf("") }
    LaunchedEffect(searchQuery) {
        viewModel.search(searchQuery)
    }

    when (uiState) {
        // ...
    }
}
```

**工作流程**：
1. `LaunchedEffect(Unit)` — 进入 Composition 时启动协程执行 block
2. 当 key 变化时 → 取消旧协程 → 用新 key 重新启动
3. 离开 Composition 时 → 自动取消协程（不需要手动 cancel）

**典型用法**：页面初始化加载数据、监听某个 ID 变化加载详情。

---

## 4. DisposableEffect — 资源注册 / 注销

```kotlin
@Composable
fun LocationTracker() {
    val context = LocalContext.current

    DisposableEffect(Unit) {
        val listener = LocationListener { location ->
            // 处理位置更新
        }
        locationManager.requestLocationUpdates(listener)  // 注册
        onDispose {
            locationManager.removeUpdates(listener)        // 注销
        }
    }
}
```

**工作流程**：
1. 进入 Composition → 执行 block 中 `onDispose` 之前的代码（注册）
2. key 变化 → 先调用 `onDispose`（注销旧资源）→ 再执行 block（注册新资源）
3. 离开 Composition → 调用 `onDispose`（注销）

**后端类比**：`@PostConstruct` 中注册监听器，`@PreDestroy` 中注销。

---

## 5. SideEffect — 每次重组后执行

```kotlin
@Composable
fun AnalyticsTracker(screenName: String) {
    SideEffect {
        // 每次重组后执行（与 Compose 渲染到屏幕同步）
        FirebaseAnalytics.logScreenView(screenName)
    }
}
```

**与 LaunchedEffect 的区别**：SideEffect 不是协程，是同步回调。每次重组都执行。

---

## 6. rememberCoroutineScope — 在回调中启动协程

```kotlin
@Composable
fun SaveButton() {
    val scope = rememberCoroutineScope()

    Button(onClick = {
        // onClick 不是 Composable 上下文，不能直接 launch
        scope.launch {
            // 现在可以在协程中执行了
            saveData()
        }
    }) {
        Text("Save")
    }
}
```

**为什么需要它？** `onClick` 回调中不能直接调用 `launch`（它不是 Composable 也没关联 Scope）。`rememberCoroutineScope()` 获取一个绑定到当前 Composition 的协程作用域。

---

## 7. derivedStateOf — 派生状态

```kotlin
@Composable
fun TodoStats(tasks: List<Task>) {
    // ❌ 每次重组都重新计算
    val doneCount = tasks.count { it.done }

    // ✅ 只有 tasks 变化时才重新计算，纯重组不重算
    val doneCount by remember {
        derivedStateOf { tasks.count { it.done } }
    }

    Text("已完成: $doneCount")
}
```

**什么时候用**：计算成本高，且输入不经常变化。类似 React 的 `useMemo`。

---

## 8. 副作用 API 决策树

```
这个操作是副作用吗？
  ├─ 不是 → 直接写在 Composable 里
  └─ 是的 ↓
      执行时机是？
        ├─ 页面初始化 / key 变化时 → LaunchedEffect(key)
        ├─ 需要注册 + 注销配对 → DisposableEffect(key)
        ├─ 每次重组后、与渲染同步 → SideEffect
        ├─ 用户交互回调中 → rememberCoroutineScope()
        └─ 从其他状态派生值 → derivedStateOf
```

---

## 面试速记

| 问题 | 回答 |
|------|------|
| 为什么网络请求不能直接写在 Composable 里？ | 因为重组会导致重复执行，造成多次请求、状态混乱 |
| LaunchedEffect 的 key 有什么用？ | key 变化时自动取消旧协程并重新启动 |
| LaunchedEffect vs DisposableEffect | LaunchedEffect 启动协程，DisposableEffect 注册/注销资源（非协程） |
| SideEffect 什么时候用？ | 每次重组后必须执行的操作（如埋点），是同步回调 |
