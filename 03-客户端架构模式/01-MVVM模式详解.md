# MVVM 模式详解

> 面试重点 ⭐⭐⭐。MVVM 是 Android 开发现代架构的事实标准。你的项目答辩中架构设计部分会重点问到。

---

## 1. Android 架构演进

```
MVC → MVP → MVVM → MVI
(2012)  (2015)  (2018)  (2021)
```

| 模式 | 核心思想 | 主要问题 |
|------|---------|---------|
| MVC | Controller 处理逻辑，View 展示 | Activity 既是 Controller 又是 View，臃肿 |
| MVP | Presenter 中介，View 被动更新 | 需要写大量接口，样板代码多 |
| **MVVM** | **ViewModel 持有状态，View 自动观察** | **当前主流，也是你用到的** |
| MVI | 单向数据流 + 意图封装 | 适合复杂交互，但初始成本高 |

---

## 2. MVVM 三层职责

```
┌─────────────────────────────────────────────────┐
│  View (Activity / Composable)                    │
│  → 只负责：UI渲染 + 事件传递                     │
│  → 不负责：业务逻辑、数据获取、状态计算           │
├─────────────────────────────────────────────────┤
│  ViewModel                                      │
│  → 持有 UI 状态 (StateFlow)                     │
│  → 处理业务逻辑（调用 Repository）              │
│  → 监听生命周期（viewModelScope）               │
│  → 不持有 View/Context 引用！                   │
├─────────────────────────────────────────────────┤
│  Model (Repository + DataSource)                │
│  → Repository：聚合数据源，决定从哪里取         │
│  → DataSource：Retrofit API、Room DAO          │
└─────────────────────────────────────────────────┘
```

**后端类比**：

| MVVM 层 | Spring 三层架构对应 |
|---------|-------------------|
| View (Compose) | Thymeleaf 模板 / REST 响应序列化 |
| ViewModel | Controller + 部分 Service 逻辑 |
| Repository | Service 层（聚合多个 DAO） |
| DataSource | DAO 层（Retrofit ≈ 远程 DAO，Room ≈ 本地 DAO） |

---

## 3. ViewModel 生命周期

```kotlin
class NewsViewModel : ViewModel() {
    // ViewModel 的关键特性：
    // 1. 存活时间 > Activity（屏幕旋转不会销毁）
    // 2. Activity finish() 时才清理（onCleared）
    // 3. 不与 View/Context 保持直接引用

    override fun onCleared() {
        super.onCleared()
        // 清理资源（但 viewModelScope 会自动取消协程）
    }
}
```

**ViewModel 存活原理**：

```
Activity 启动
  → ViewModelProvider 创建 ViewModel
    → ViewModel 存入 ViewModelStore（关联 Activity 的 ViewModelStoreOwner）
      → 屏幕旋转：Activity 销毁重建，ViewModelStore 保留
        → 新 Activity 从 ViewModelStore 取回同一个 ViewModel
      → Activity finish()：ViewModelStore.clear() → ViewModel.onCleared()
```

**后端类比**：ViewModel ≈ `@SessionScope` Bean。Session 有效期内 Bean 一直存在，会话结束才销毁。屏幕旋转 ≈ 同一个 HTTP Session 的多次请求。

---

## 4. ViewModel + StateFlow + Compose 的标准配方

```kotlin
// 1. 定义 UiState（用 sealed class 保证分支完整）
sealed class NewsUiState {
    object Loading : NewsUiState()
    data class Success(val news: List<NewsItem>) : NewsUiState()
    data class Error(val message: String) : NewsUiState()
    object Empty : NewsUiState()
}

// 2. ViewModel 持有 MutableStateFlow（内部可变）
class NewsViewModel @Inject constructor(
    private val repository: NewsRepository
) : ViewModel() {

    private val _uiState = MutableStateFlow<NewsUiState>(NewsUiState.Loading)
    val uiState: StateFlow<NewsUiState> = _uiState.asStateFlow()

    fun loadNews() {
        viewModelScope.launch {
            _uiState.value = NewsUiState.Loading
            try {
                val news = repository.getNews()
                _uiState.value = if (news.isEmpty()) {
                    NewsUiState.Empty
                } else {
                    NewsUiState.Success(news)
                }
            } catch (e: Exception) {
                _uiState.value = NewsUiState.Error(e.message ?: "未知错误")
            }
        }
    }
}

// 3. Composable 订阅 StateFlow
@Composable
fun NewsScreen(viewModel: NewsViewModel = hiltViewModel()) {
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()

    when (uiState) {
        is NewsUiState.Loading -> LoadingView()
        is NewsUiState.Success -> NewsList((uiState as NewsUiState.Success).news)
        is NewsUiState.Error -> ErrorView((uiState as NewsUiState.Error).message)
        is NewsUiState.Empty -> EmptyView()
    }
}
```

---

## 5. MVVM 常见错误

| 错误 | 为什么错 | 正确做法 |
|------|---------|---------|
| ViewModel 持有 Context | 屏幕旋转时泄漏 | 用 `AndroidViewModel`（持有 Application Context）或传参数 |
| 在 Composable 中调网络请求 | 重组导致重复请求 | 放在 ViewModel + LaunchedEffect |
| ViewModel 直接操作 View | 破坏 MVVM 单向依赖 | ViewModel 只暴露 State，UI 自己决定怎么画 |
| 一个 ViewModel 做所有事 | 臃肿难维护 | 按页面/功能拆分 ViewModel |

---

## 6. 面试追问题目

**Q: ViewModel 能替代 onSaveInstanceState 吗？**
A: 不能。ViewModel 只在 Activity 重建（如旋转）时存活，但在**进程被杀死**时不存活。onSaveInstanceState 处理的是进程死亡场景（系统回收内存后重建）。实际项目两者配合使用：ViewModel 存业务数据，SavedStateHandle 存最小必需参数。

**Q: ViewModel 和 AndroidViewModel 的区别？**
A: `ViewModel` 不持有任何 Android 引用。`AndroidViewModel` 持有 Application Context（但不是 Activity Context！所以不会泄漏）。如果你需要访问资源文件、SharedPreferences 等可以用 AndroidViewModel。

---

## 面试速记

| 考点 | 回答 |
|------|------|
| MVVM 各层职责 | View（UI+事件）→ ViewModel（状态+逻辑）→ Repository（数据聚合） |
| ViewModel 存活原理 | 存在 ViewModelStore 中，屏幕旋转重建 Activity 后取回 |
| 为什么 ViewModel 不持有 View 引用 | 防止屏幕旋转时内存泄漏 |
| sealed class 做 UiState | 编译器检查分支覆盖，保证每种状态都有 UI |
