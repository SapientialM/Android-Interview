# 05-面试八股

> Android 客户端面试高频题精编。每题包含「面试官怎么问 → 回答要点 → 后端类比 → 延伸追问」。
> 内容整合自 `tmp/` 中多个资料库，按面试专题重新组织、精炼。

---

## 题目分级

| 级别 | 含义 | 阅读优先级 |
|:---:|------|------|
| ⭐⭐⭐ | 必考（大厂高频，面试极大概率问到） | **最高** |
| ⭐⭐ | 高频（大部分公司会问） | **高** |
| ⭐ | 抽查（偶尔问到，或 Compose 时代重要性降低） | 有时间再看 |

---

## 文件导航

| # | 文件 | 级别 | 核心考点 |
|---|------|:---:|------|
| 01 | [Handler消息机制](01-Handler消息机制.md) | ⭐⭐⭐ | Looper/MessageQueue/Handler、内存泄漏、同步屏障 |
| 02 | [Binder跨进程通信](02-Binder跨进程通信.md) | ⭐⭐⭐ | C/S架构、mmap、ServiceManager、AIDL |
| 03 | [Activity生命周期与启动模式](03-Activity生命周期与启动模式.md) | ⭐⭐ | 回调顺序、四种启动模式、onSaveInstanceState |
| 04 | [Compose核心原理](04-Compose核心原理.md) | ⭐⭐⭐ | 重组机制、状态管理、副作用、性能优化 |
| 05 | [协程与Flow](05-协程与Flow.md) | ⭐⭐⭐ | suspend、调度器、StateFlow vs SharedFlow、背压 |
| 06 | [内存泄漏与OOM](06-内存泄漏与OOM.md) | ⭐⭐ | 常见场景、LeakCanary原理、排查方法 |
| 07 | [图片加载与三级缓存](07-图片加载与三级缓存.md) | ⭐⭐ | LruCache/DiskLruCache/网络、Glide流程、Bitmap内存 |
| 08 | [性能优化专题](08-性能优化专题.md) | ⭐⭐ | 启动/内存/卡顿/包体积/网络/耗电 |
| 09 | [网络与数据存储](09-网络与数据存储.md) | ⭐ | Retrofit/OkHttp原理、DataStore、Room |
| 10 | [View事件分发与绘制流程](10-View事件分发与绘制流程.md) | ⭐ | dispatchTouchEvent、measure/layout/draw（Compose时代弱化） |
| 11 | [Android系统启动与AMS](11-Android系统启动与AMS.md) | ⭐⭐⭐(进阶) | init→Zygote→SystemServer、AMS、App启动流程 |
| 12 | [Java基础速览](12-Java基础速览.md) | ⭐ | Android 面试中的 Java 高频题速览 |

---

## 使用建议

1. **时间紧**：优先背 ⭐⭐⭐ 的题目（01-Handler、02-Binder、04-Compose、05-协程Flow）
2. **每个题目**：先看「面试官怎么问」→ 自己尝试回答 → 对照「回答要点」补充缺失点
3. **每天背诵**：5-10 题，睡前用「延伸追问」自测
4. **后端类比**：每个题都找到后端对应概念，用已有知识锚定新知识

## 面试当天速查清单

1. Handler：Looper.prepare → MessageQueue → Looper.loop → Handler.handleMessage
2. Binder：mmap 一次拷贝 + C/S 架构 + ServiceManager
3. Activity：A 启动 B 的回调顺序（A.onPause → B.onCreate → B.onStart → B.onResume → A.onStop）
4. Compose：重组 = State 变化触发重新执行，智能跳过，副作用放 LaunchedEffect
5. 协程：suspend = 挂起不阻塞线程，Dispatcher.Main/IO/Default
6. 内存泄漏：Handler（静态+WeakReference）、静态持有Activity、单例持有Context
7. 图片缓存：LruCache（内存）→ DiskLruCache（磁盘）→ 网络
8. MVVM 数据流：Screen → ViewModel → Repository → Retrofit
