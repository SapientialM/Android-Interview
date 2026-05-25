---
layout: default
---

# Android-Interview

Android 客户端面试准备仓库。适合**零 Android 基础、有 Java 后端经验**的工程师，约 17 天从入门到面试。

## 快速开始

### 路径一：零基础入门（按顺序阅读）

[00-15天速成规划/](00-15天速成规划/) → [01-Kotlin-速成/](01-Kotlin-速成/) → [02-Compose-UI体系/](02-Compose-UI体系/) → [03-客户端架构模式/](03-客户端架构模式/)

### 路径二：面试冲刺

[05-面试八股/](05-面试八股/) → [04-项目八股/](04-项目八股/) → [06-资源索引/](06-资源索引/)

### 路径三：面试中快速查阅

直接搜索 [05-面试八股/](05-面试八股/) 中的具体题目。

## Java 后端 → Android 概念映射

| 后端 | Android | 说明 |
|------|---------|------|
| Spring Controller | Activity | 页面入口，但有生命周期 |
| @SessionScope Bean | ViewModel | 屏幕旋转时数据不丢 |
| OpenFeign | Retrofit | 动态代理 HTTP 调用 |
| Spring Security Filter | OkHttp Interceptor | 链式拦截 |
| Redis | LruCache/DiskLruCache | 三级缓存 |
| CompletableFuture | Kotlin 协程 | 异步编排 |
| Project Reactor Flux | Kotlin Flow | 响应式流 |
| gRPC | Binder IPC | 跨进程通信 |
| EventLoop + BlockingQueue | Handler+Looper+MessageQueue | 消息机制 |

## 仓库结构

{% assign dirs = site.pages | map: "dir" | uniq | sort %}
{% for d in dirs %}
{% unless d contains "assets" or d contains "tmp" %}
- [{{ d }}]({{ d }})
{% endunless %}
{% endfor %}

## 面试八股速查

| # | 专题 | 级别 |
|---|------|:---:|
| 01 | [Handler 消息机制](05-面试八股/01-Handler消息机制) | ⭐⭐⭐ |
| 02 | [Binder 跨进程通信](05-面试八股/02-Binder跨进程通信) | ⭐⭐⭐ |
| 03 | [Activity 生命周期与启动模式](05-面试八股/03-Activity生命周期与启动模式) | ⭐⭐ |
| 04 | [Compose 核心原理](05-面试八股/04-Compose核心原理) | ⭐⭐⭐ |
| 05 | [协程与 Flow](05-面试八股/05-协程与Flow) | ⭐⭐⭐ |
| 06 | [内存泄漏与 OOM](05-面试八股/06-内存泄漏与OOM) | ⭐⭐ |
| 07 | [图片加载与三级缓存](05-面试八股/07-图片加载与三级缓存) | ⭐⭐ |
| 08 | [性能优化专题](05-面试八股/08-性能优化专题) | ⭐⭐ |
| 09 | [网络与数据存储](05-面试八股/09-网络与数据存储) | ⭐ |
| 10 | [View 事件分发与绘制流程](05-面试八股/10-View事件分发与绘制流程) | ⭐ |
| 11 | [Android 系统启动与 AMS](05-面试八股/11-Android系统启动与AMS) | ⭐⭐⭐ |
| 12 | [Java 基础速览](05-面试八股/12-Java基础速览) | ⭐ |
