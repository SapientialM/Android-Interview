# 单向数据流 (UDF)

> UDF = Unidirectional Data Flow。核心原则：**事件向上，状态向下**。
> **后端类比**：类似 Redux / Vuex 的单向数据流，或 Spring 中 Controller 接收请求（up）→ 返回视图（down）的模式。

---

## 1. 为什么需要单向数据流？

**问题：双向绑定导致的状态混乱**

```kotlin
// ❌ 双向数据流 — 谁都可以改状态，Debug 困难
class BadViewModel {
    val state = MutableStateFlow(UiState())
    fun update() { state.value = ... }
}
class BadView {
    fun onUserAction() {
        viewModel.state.value = ...  // View 直接改 ViewModel 状态
    }
}
// 状态被谁改的？什么时候改的？完全不可追踪
```

**✅ 单向数据流 — 清晰、可追踪、可测试**

```kotlin
// 事件向上（View → ViewModel）
// 状态向下（ViewModel → View）

// ViewModel 是状态的唯一来源
class NewsViewModel : ViewModel() {
    private val _uiState = MutableStateFlow(UiState())
    val uiState: StateFlow<UiState> = _uiState.asStateFlow()

    fun onRefresh() {  // 事件处理函数
        viewModelScope.launch {
            _uiState.value = UiState.Loading
            _uiState.value = UiState.Success(repository.getNews())
        }
    }
}

// View 只发送事件 + 订阅状态
@Composable
fun NewsScreen(viewModel: NewsViewModel) {
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()  // 状态向下

    Button(onClick = viewModel::onRefresh) {  // 事件向上
        Text("Refresh")
    }
}
```

---

## 2. MVI 模式（UDF 的进阶版）

MVI = Model-View-Intent，是 UDF 的一个具体实现方案：

```
            ┌─ Intent (用户意图) ─┐
            ▼                       │
        ViewModel                   View
            │                       ▲
            └─ State (UI状态) ──────┘
```

```kotlin
// 1. 定义 Intent（用户意图）
sealed class NewsIntent {
    object LoadNews : NewsIntent()
    data class OnNewsClick(val newsId: String) : NewsIntent()
    object OnRefresh : NewsIntent()
}

// 2. ViewModel 处理所有 Intent
class NewsViewModel : ViewModel() {
    private val _uiState = MutableStateFlow(UiState())
    val uiState: StateFlow<UiState> = _uiState.asStateFlow()

    fun onIntent(intent: NewsIntent) {
        when (intent) {
            is NewsIntent.LoadNews -> loadNews()
            is NewsIntent.OnNewsClick -> navigateToDetail(intent.newsId)
            is NewsIntent.OnRefresh -> refresh()
        }
    }
}

// 3. View 只发送 Intent
@Composable
fun NewsScreen(viewModel: NewsViewModel) {
    LaunchedEffect(Unit) {
        viewModel.onIntent(NewsIntent.LoadNews)
    }
    Button(onClick = { viewModel.onIntent(NewsIntent.OnRefresh) }) {
        Text("Refresh")
    }
}
```

**MVVM vs MVI 怎么选？**

| 场景 | 推荐 |
|------|------|
| 简单页面（表单、静态信息） | MVVM（够用，小巧） |
| 复杂交互（搜索+列表+筛选） | MVI（所有事件统一管理） |
| 面试 | 知道 MVI 概念就行，项目中用 MVVM 完全 OK |

---

## 3. UDF 的好处

1. **单一数据源**：状态只在 ViewModel 中定义，View 不会私自改
2. **可测试**：ViewModel 是纯 Kotlin 类，可以直接单测
3. **可追溯**：所有状态变化都来自 ViewModel 中的事件处理函数
4. **可预测**：一个 Intent 对应一个状态变化，不会出现「不知道谁改了状态」

**后端类比**：
- 事件向上 ≈ HTTP Request 发向 Controller
- 状态向下 ≈ Controller 返回 View / JSON Response
- UDF = REST API 的无状态设计：请求的参数都包含在 Request 里，响应的数据都包含在 Response 里

---

## 面试速记

| 问题 | 回答 |
|------|------|
| 什么是 UDF？ | 单向数据流：事件向上（View→ViewModel），状态向下（ViewModel→View） |
| UDF 的好处？ | 单一数据源、可预测、可测试、可追溯状态变化 |
| MVVM vs MVI？ | MVI 是 UDF 的具体实现，多了 Intent 封装，适合复杂交互 |
