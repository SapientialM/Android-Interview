# Java 基础速览

> 级别：⭐ 基础（Android 面试中的 Java 题，你有后端基础只需速览差异点）。
> 目标：15 分钟过一遍 Java 高频题，确保不会被基础题绊倒。

---

## 高频题速览

### 1. HashMap 原理

- **1.7**：数组 + 链表，头插法，扩容时可能死循环（多线程）
- **1.8**：数组 + 链表 + 红黑树（链表 > 8 转红黑树，< 6 转回链表），尾插法
- **put 流程**：hash → index → 判断是否冲突 → 无冲突直接放 → 有冲突判断是链表还是树 → 放入 → 检查扩容
- **扩容**：默认容量 16，负载因子 0.75，扩容到 2 倍

### 2. ConcurrentHashMap 原理

- **1.7**：Segment（分段锁，默认 16 个 Segment，并发度 16）
- **1.8**：CAS + synchronized（只锁链表头/树根节点），无锁读
- **与 Hashtable 的区别**：Hashtable 锁整个表，CHM 分段 + CAS

### 3. synchronized vs Lock

- **synchronized**：JVM 内置，自动释放，升级锁（偏向→轻量级→重量级）
- **Lock（ReentrantLock）**：API 层面，必须手动 unlock，支持公平锁、可中断、tryLock
- **选哪个**：简单场景用 synchronized（JVM 优化好），需要高级特性用 Lock

### 4. volatile 与 JMM

- **volatile**：保证**可见性**（修改立即刷新到主内存）和**有序性**（禁止指令重排），不保证原子性
- **JMM**：主内存 + 工作内存模型，8 种原子操作（lock/unlock/read/write 等）
- **典型用途**：单例的 double-checked locking（volatile 防止重排）

### 5. 线程池

```java
new ThreadPoolExecutor(
    corePoolSize,      // 核心线程数
    maximumPoolSize,   // 最大线程数
    keepAliveTime,     // 空闲线程存活时间
    unit,              // 时间单位
    workQueue,         // 任务队列
    handler            // 拒绝策略（Abort/CallerRuns/Discard/DiscardOldest）
);
```

| 线程池类型 | 特点 |
|-----------|------|
| FixedThreadPool | 固定核心线程，无界队列（可能 OOM） |
| CachedThreadPool | 弹性线程，60s 回收（适合短任务） |
| ScheduledThreadPool | 定时/周期执行 |
| SingleThreadExecutor | 单线程顺序执行 |

### 6. GC 算法与收集器

- **判断存活**：引用计数（Python）、GCRoot 可达性分析（Java）
- **GC 算法**：标记-清除、标记-复制（年轻代）、标记-整理（老年代）
- **G1 收集器**（Android 9+ ART 默认）：Region 分块，可预测的停顿时间
- **Android ART GC**：Concurrent Mark-Sweep（前台）、Compacting GC（后台）

### 7. 类加载机制

- **双亲委派**：Application CL → Extension CL → Bootstrap CL
- **Android 的类加载器**：BootClassLoader → PathClassLoader（加载 APK） → DexClassLoader（加载外部 dex/jar）
- **热修复原理**：把修复后的 class.dex 放在 PathClassLoader 的 dexElements 前面

---

## Android 面试 vs 后端面试 Java 题的差异

| 维度 | 后端面试 | Android 面试 |
|------|---------|------------|
| HashMap | 追问红黑树实现细节 | 追问 ArrayMap/SparseArray（Android 优化） |
| 并发 | 追问 AQS 源码 | 追问 Handler/RxJava/协程 |
| GC | 追问 ZGC/Shenandoah | 追问 ART vs Dalvik 的区别、GC 对 UI 卡顿的影响 |
| 线程池 | 追问 ForkJoinPool 原理 | 追问 IntentService/AsyncTask（现在已被协程替代） |
| 类加载 | 追问 OSGI 模块化 | 追问热修复/插件化的类加载机制 |
