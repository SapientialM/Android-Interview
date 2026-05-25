# Android-Interview

> 一个结构化、带目录的 Android 客户端面试准备仓库。
> 适合**零 Android 基础、有 Java 后端经验**的工程师，约 17 天从入门到面试。

---

## 仓库定位

- **目标人群**：Java 后端转 Android 客户端，或零基础入门 Android 开发的工程师
- **覆盖阶段**：入门 → 进阶 → 面试冲刺
- **内容策略**：面试考什么讲什么，用后端知识类比降低理解成本
- **核心理念**：不追求体系完整，追求「能讲清楚、能通过面试」

---

## 如何使用

### 路径一：零基础入门（约 10 天，按顺序阅读）

```
00-15天速成规划/  →  01-Kotlin-速成/  →  02-Compose-UI体系/  →  03-客户端架构模式/
```

先看全局规划，再分模块学习，每学完一个模块配合项目代码实践。

### 路径二：面试冲刺（约 7 天）

```
05-面试八股/  →  04-项目八股/  →  06-资源索引/（按需深入）
```

直接背高频面试题，准备项目介绍和答辩逐字稿。

### 路径三：面试中快速查阅

直接搜索 [05-面试八股/](05-面试八股/) 中的具体题目关键词。

---

## Java 后端 → Android 概念映射表

如果你是 Java 后端开发，用这张表快速建立认知锚点：

| 后端概念 | Android 对应 | 一句话说明 |
|---------|-------------|-----------|
| Spring Boot 应用 | Android App (Application + Activity) | App 是进程入口，Activity 是页面入口 |
| Spring MVC Controller | Activity / Fragment | 处理用户交互，但多了生命周期回调 |
| @Controller + Thymeleaf | Activity + Compose/Screen | 页面模板渲染 vs 声明式 UI 重组 |
| Spring Bean (IoC) | Hilt / Dagger 依赖注入 | @Autowired → @Inject, @Bean → @Provides |
| @SessionScope Bean | ViewModel | 屏幕旋转时数据不丢失 |
| Spring Security Filter Chain | OkHttp Interceptor Chain | 链式拦截，做日志/认证/缓存 |
| RestTemplate / WebClient | Retrofit + OkHttp | HTTP 客户端 |
| OpenFeign 动态代理 | Retrofit 动态代理 | 接口 + 注解 → 自动生成 HTTP 请求 |
| Redis 缓存 | LruCache / DiskLruCache | 内存缓存 + 磁盘缓存 |
| Maven / Gradle | Gradle (Kotlin DSL) | 构建工具，概念一致 |
| JVM (HotSpot) | ART (Android Runtime) | 虚拟机，Android 特化了 GC 和编译 |
| Netty EventLoop | Handler + Looper + MessageQueue | 事件循环 + 消息队列，主线程死循环不卡死 |
| CompletableFuture / 虚拟线程 | Kotlin 协程 (Coroutine) | 异步编排，但更轻量 |
| Project Reactor Flux | Kotlin Flow | 响应式流，冷流/热流概念相通 |
| gRPC / RPC 框架 | Binder IPC | 跨进程通信，Android 独有机制 |
| Spring 三层架构 | MVVM (View → ViewModel → Repository) | 分层思想一致 |

---

## 仓库结构

```
Android-Interview/
├── README.md                          ← 你在这里
│
├── 00-15天速成规划/                    ← 零基础 17 天冲刺排期
│   ├── README.md
│   ├── 第1阶段-基础搭建-Day1-5.md
│   ├── 第2阶段-面试重点突破-Day6-11.md
│   └── 第3阶段-模拟面试-Day12-17.md
│
├── 01-Kotlin-速成/                     ← Java 开发者 Kotlin 速成
│   ├── README.md
│   ├── 01-Java转Kotlin速查表.md
│   ├── 02-协程核心概念.md
│   ├── 03-Flow核心概念.md
│   └── 04-Kotlin常见陷阱.md
│
├── 02-Compose-UI体系/                  ← Jetpack Compose 核心
│   ├── README.md
│   ├── 01-声明式UI思想入门.md
│   ├── 02-重组与状态管理.md
│   ├── 03-副作用API详解.md
│   ├── 04-列表与性能优化.md
│   └── 05-导航与页面架构.md
│
├── 03-客户端架构模式/                   ← 架构设计
│   ├── README.md
│   ├── 01-MVVM模式详解.md
│   ├── 02-单向数据流-UDF.md
│   ├── 03-Hilt依赖注入.md
│   ├── 04-网络层设计-Retrofit+OkHttp.md
│   └── 05-模块化设计.md
│
├── 04-项目八股/                        ← 项目答辩准备
│   ├── README.md
│   ├── 01-项目介绍模板.md
│   ├── 02-架构图绘制模板.md
│   ├── 03-技术难点答辩模板.md
│   └── 04-常见追问FAQ.md
│
├── 05-面试八股/                        ← 高频面试题
│   ├── README.md
│   ├── 01-Handler消息机制.md
│   ├── 02-Binder跨进程通信.md
│   ├── 03-Activity生命周期与启动模式.md
│   ├── 04-Compose核心原理.md
│   ├── 05-协程与Flow.md
│   ├── 06-内存泄漏与OOM.md
│   ├── 07-图片加载与三级缓存.md
│   ├── 08-性能优化专题.md
│   ├── 09-网络与数据存储.md
│   ├── 10-View事件分发与绘制流程.md
│   ├── 11-Android系统启动与AMS.md
│   └── 12-Java基础速览.md
│
└── 06-资源索引/                        ← 原始资料 + 外部推荐
    ├── README.md
    ├── 01-原始资料库索引.md
    └── 02-外部推荐资源.md
```

---

## 声明

本仓库内容整合自多个开源项目（详见 [06-资源索引/](06-资源索引/)），经重新组织、精炼，并补充了后端类比说明，仅供学习参考。原始资料版权归原作者所有。
