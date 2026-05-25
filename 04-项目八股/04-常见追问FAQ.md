# 常见追问 FAQ

> 准备好这些追问的逐字稿，答辩时就不会「被问住」。

---

## 一、技术选型类

### Q1: 为什么用 Compose 而不是传统 XML？
声明式 UI 更契合 Kotlin 语言特性，状态驱动 UI 减少样板代码（不需要 findViewById / DataBinding）。Compose 是 Google 推荐的未来方向，API 已稳定，新项目应该选它。如果是存量项目，可能需要考虑迁移成本。

### Q2: 为什么选 Hilt 而不是 Koin？
Hilt 基于 Dagger，编译期做依赖校验（编译就发现配置错误），Koin 是运行时的。Hilt 与 Jetpack ViewModel 集成更好（@HiltViewModel）。但 Koin 的 API 更简单，小项目也可以选它。

### Q3: 为什么用协程而不是 RxJava？
协程是 Kotlin 语言级别的异步方案，学习曲线更低，结构化并发保证不泄漏。RxJava 虽然有大量操作符，但学习成本高、栈轨迹难读。新 Kotlin 项目全用协程。

### Q4: 为什么选 Coil 而不是 Glide？
Coil 是 Kotlin 协程原生的，API 更 Kotlin 化（协程 + Flow），对 Compose 支持最好（AsyncImage composable），包体积小。Glide 更成熟但偏传统的线程池模型。

---

## 二、架构设计类

### Q5: 你的 UI 状态是怎么定义的？
用 sealed class 定义所有可能的 UI 状态：Loading / Success(data) / Error(message) / Empty。ViewModel 暴露 StateFlow<UiState>，UI 层通过 when 分支渲染不同状态。好处是编译器保证不会遗漏状态。

### Q6: 为什么不用 LiveData？
StateFlow 是协程原生支持，可以配合 flowOn、combine 等操作符，在纯 Kotlin 中即可测试。LiveData 是 Android 特化的，操作符少，在非 Android 环境无法使用。而且 StateFlow 必须有初始值，避免了 LiveData 的 null 问题。

### Q7: 网络请求失败了怎么处理？
Repository 层 try-catch 捕获异常（IOException 和普通 Exception），统一封装成 sealed class Result。ViewModel 根据 Result 类型更新 UiState（Loading/Error/Success）。UI 层根据 Error 状态显示重试按钮。OkHttp 层加了日志拦截器和超时配置。

### Q8: 屏幕旋转时状态怎么保留？
业务数据放在 ViewModel 中（ViewModel 在 ViewModelStore 中存活，不受旋转影响）。临时 UI 状态（如输入框文字）用 rememberSaveable 保存（通过 SavedStateHandle 序列化到 Bundle）。

---

## 三、质量保障类

### Q9: 你写了单元测试吗？
目前项目比较小，没有写完整的单元测试。但架构设计时考虑了可测试性：ViewModel 是纯 Kotlin 类，通过注入 Repository 的 Mock 实现即可测试。如果后续继续开发，我会给 ViewModel 和 Repository 补上单测。

### Q10: 怎么保证 App 不崩溃？
统一错误处理（sealed class UiState + try-catch）。网络层有超时配置和重试机制。边界情况（空数据、弱网）有对应的 UI 状态（Empty/Error）。如果上线，会接入 Firebase Crashlytics 或 Bugly 监控崩溃。

### Q11: 灰度/AB 测试怎么做？
（适合大厂面试）通过远程配置（Firebase Remote Config 或自建配置中心）下发实验开关。不同用户看到不同 UI/功能，对比数据指标（留存、点击率、转化率）。

---

## 四、性能类

### Q12: App 启动速度怎么样？怎么优化的？
没有精确测量，但可以优化的方向：Application.onCreate 延迟初始化（非必需组件异步加载）；Splash 主题预加载（windowBackground 设品牌色实现视觉秒开）；首页数据用缓存避免白屏等待。

### Q13: 列表有 10 万条数据怎么办？
分页加载（Paging3）只加载可见页数据。ViewHolder 复用（LazyColumn 自动复用 + key 确保 Diff 正确）。图片管控（Coil size 按需加载、内存+磁盘缓存、滑动时暂停加载）。状态隔离（列表滚动状态与业务状态分离）。

### Q14: 包体积优化做过吗？
没有在项目中实践，但了解的优化方向：R8 代码混淆和资源压缩、图片转 WebP、App Bundle 按架构分发 SO、删除无用资源、动态下发非核心功能。

### Q15: 内存泄漏怎么预防？
开发阶段用 LeakCanary 自动检测。代码层面注意：单例用 ApplicationContext 而不是 Activity Context；Handler 用静态内部类 + WeakReference；在 onDestroy 中取消协程和移除回调；避免匿名内部类持有外部 Activity 引用。

---

## 你的行动清单

- [ ] 每道题写出自己的 2-3 句话版本
- [ ] 特别准备 Q5/Q7/Q8/Q13（和你项目最相关）
- [ ] 面试前 1 小时快速过一遍本页所有题目
