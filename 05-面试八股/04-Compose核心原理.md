# Compose 核心原理

> 级别：⭐⭐⭐ 高频（2024+ 面试新趋势，Compose 项目必问）。
> 注意：`tmp/` 原始资料不含 Compose 内容，本章为全新编写。

---

## 面试官怎么问

- 「Compose 的重组机制是什么？和 View 的 invalidate 有什么区别？」
- 「Compose 怎么知道哪些 UI 需要更新（智能跳过）？」
- 「remember、mutableStateOf、LaunchedEffect 的区别？」
- 「Compose 怎么处理性能问题？」

---

## 回答框架

### 1. Compose 的核心公式

**UI = f(state)**

你写一个 @Composable 函数描述 UI 长什么样，框架负责在 state 变化时重新调用这个函数来刷新 UI。

```kotlin
@Composable
fun Greeting(name: String) {
    Text("Hello $name")  // name 变了就自动更新
}
```

### 2. 重组 (Recomposition) = 智能 re-render

**触发条件**：Composable 函数读取的 **State 对象** 值发生变化。

```kotlin
@Composable
fun Counter() {
    var count by remember { mutableStateOf(0) }  // State 对象

    Text("Count: $count")  // 读取 count → 建立依赖

    Button(onClick = { count++ })  // count++ → State 变化 → 触发重组
}
// count 变化 → Counter() 重新执行 → Text 内容更新
```

**智能跳过**：Compose 编译期在每个 Composable 调用前后插入相等性比较。如果参数都没变，跳过重组。类似 React 的 `shouldComponentUpdate`（但 Compose 是自动的）。

### 3. 状态管理四层

| 层级 | API | 生命周期 | 用途 |
|------|-----|---------|------|
| Composable 内 | `remember { mutableStateOf() }` | 重组间 | 临时 UI 状态（输入框文字） |
| Composable 内 | `rememberSaveable { }` | 旋转屏幕+进程死亡 | 输入内容、滚动位置 |
| ViewModel | `MutableStateFlow` | ViewModel 生命周期 | 业务数据（新闻列表） |
| 全局 | `CompositionLocal` | App 生命周期 | 主题、语言 |

### 4. 副作用 = Composable 之外的事

| API | 用途 | 时机 |
|-----|------|------|
| `LaunchedEffect(key)` | 启动协程（数据加载） | key 变化时重启，离开 Composition 时取消 |
| `DisposableEffect(key)` | 注册/注销资源 | key 变化或离开时 dispose |
| `SideEffect` | 每次重组后执行 | 每次重组后、与渲染同步 |
| `rememberCoroutineScope()` | 回调中启动协程 | 点击等非 Composable 上下文 |
| `derivedStateOf` | 派生状态（优化重组） | 输入变化时才重算 |

### 5. 性能优化

| 手段 | 原理 |
|------|------|
| `@Stable` / `@Immutable` | 告诉 Compose 这个类的相等性可依赖，减少无效重组 |
| `derivedStateOf` | 输入不变时不重算 |
| LazyColumn 的 `key` | 确保列表 Diff 正确，避免错位重组 |
| `Modifier` 顺序 | padding 在 background 之前（内边距 vs 外边距） |

---

## 后端/前端类比

| Compose | React | 后端类比 |
|---------|-------|---------|
| @Composable 函数 | React 组件 | Thymeleaf 模板片段 |
| 重组 (Recomposition) | Re-render | 模板重新渲染 |
| mutableStateOf | useState() | 带发布/订阅的变量 |
| LaunchedEffect | useEffect(async) | @PostConstruct 中启动异步 |
| LazyColumn + key | 虚拟列表 + key prop | 分页查询 |
| CompositionLocal | React Context | 全局配置 |

---

## 延伸追问

1. **Compose 和 RecyclerView 的区别？**
   → RecyclerView 需要手动写 Adapter + ViewHolder + LayoutManager。LazyColumn 只需 `items {}`，框架自动做复用和 Diff。类型安全（编译期检查），与状态管理体系集成好。

2. **什么情况下 Compose 不能跳过重组？**
   → 参数不稳定（没有 @Stable 标记的类）、频繁 new 对象（每次传新 Lambda）、列表没加 key。

3. **Compose 可以嵌入传统 View 吗？**
   → 可以。用 `AndroidView` —— 在 Compose 中嵌入传统 View。反向也可：`ComposeView` 在 XML 中嵌入 Compose。

4. **Compose 适合所有场景吗？**
   → 大部分场景都适合。但复杂的自定义 View（如地图 SDK、视频播放器）可能需要用 `AndroidView` 包装。

5. **Compose 和 Flutter 的思路像吗？**
   → 都是声明式 UI。但 Flutter 有自己的渲染引擎（Skia），Compose 也直接渲染到 Canvas，不依赖原生 View 树。两者都是 `UI = f(state)` 范式。
