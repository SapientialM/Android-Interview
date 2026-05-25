# 02-Compose-UI体系

> Jetpack Compose 是 Android 的现代声明式 UI 框架。如果你有 Java 后端 + 前端（React/Vue）的经验，Compose 会非常自然。
> **核心策略**：把 Compose 理解为「用 Kotlin 写的 React」，把重组理解为 re-render。

---

## 学习目标

学完后你应该能：

- [ ] 理解声明式 UI 与命令式 UI 的区别
- [ ] 能写出基本的 Composable 函数（Text、Button、Column、Row、LazyColumn）
- [ ] 理解重组机制：什么触发重组、如何避免无效重组
- [ ] 知道状态管理的层次：remember → ViewModel + StateFlow
- [ ] 能用 LaunchedEffect 和 DisposableEffect 管理副作用
- [ ] 能回答 Compose 面试高频题

---

## 文件导航

| 文件 | 内容 | 面试价值 |
|------|------|:---:|
| [01-声明式UI思想入门](01-声明式UI思想入门.md) | 声明式 vs 命令式、Compose vs 传统 View 体系 | ⭐⭐ |
| [02-重组与状态管理](02-重组与状态管理.md) | 重组机制、remember、mutableStateOf、ViewModel+StateFlow | ⭐⭐⭐ |
| [03-副作用API详解](03-副作用API详解.md) | LaunchedEffect、DisposableEffect、SideEffect、derivedStateOf | ⭐⭐⭐ |
| [04-列表与性能优化](04-列表与性能优化.md) | LazyColumn、key、@Stable、Modifier 顺序 | ⭐⭐ |
| [05-导航与页面架构](05-导航与页面架构.md) | Navigation Compose、参数传递、Deep Link | ⭐ |

---

## 后端/前端概念对照

| 前端概念 | Compose 对应 |
|---------|------------|
| React/Vue 组件 | @Composable 函数 |
| React re-render | 重组 (Recomposition) |
| React state / Vue ref | mutableStateOf |
| React useEffect | LaunchedEffect / DisposableEffect |
| React key prop | LazyColumn 的 key 参数 |
| React 虚拟列表 | LazyColumn |
| React Context | CompositionLocal |
| React Router | Navigation Compose |
