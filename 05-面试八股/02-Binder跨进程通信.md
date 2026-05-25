# Binder 跨进程通信

> 级别：⭐⭐⭐ 必考（初中级开发也会问）。这是 Android 最核心、最独特的机制。

---

## 面试官怎么问

- 「Binder 是什么？为什么 Android 要用 Binder？」
- 「Binder 的通信流程是怎样的？」
- 「AIDL 中的 in, out, inout 有什么区别？」
- 「Binder 与 Linux 其他 IPC（管道、共享内存、Socket）的区别？」

---

## 回答框架（三层递进）

### 第一层：应用层（AIDL）

跨进程通信的接口定义语言：

```kotlin
// 定义 AIDL 接口
interface IMyService {
    String getData(int id);  // in 参数
}

// Client 调用
val service = IMyService.Stub.asInterface(binder)
val data = service.getData(123)  // 看起来像本地方法调用
```

- `in`：数据从 client 传向 server（默认）
- `out`：数据从 server 传回 client
- `inout`：双向传递（两次拷贝，开销最大）

### 第二层：Framework 层（ServiceManager + Binder 驱动）

```
┌─────────┐   1.注册     ┌──────────────┐   2.查询     ┌─────────┐
│ Server  │ ──────────→  │ ServiceManager│ ←────────── │ Client  │
│ (AMS等) │              │ (Binder 0号)  │             │         │
└────┬─────┘             └──────────────┘            └────┬─────┘
     │                                                     │
     └─────────── 3. Binder 驱动 (binder_ioctl) ────────────┘
                        └─ 路由消息、管理线程池
```

1. Server 通过 ServiceManager 注册自己
2. Client 向 ServiceManager 查询要的服务
3. Binder 驱动负责消息路由、内存映射、线程管理
4. Client 和 Server 直接通过 Binder 驱动通信（不走 ServiceManager）

### 第三层：内核层（mmap 一次拷贝）

```
传统 IPC（管道/共享内存）：
  发送方用户空间 → 内核缓冲区 → 接收方用户空间（两次拷贝）

Binder：
  接收方通过 mmap 映射一块内核缓冲区到用户空间
  发送方数据写入内核缓冲区（一次拷贝）
  接收方直接从映射区域读取（不需要再拷贝）
  
  本质：mmap 让内核和用户空间共享同一块物理内存
```

---

## 为什么 Android 选 Binder？

| 对比维度 | 管道/消息队列 | 共享内存 | Socket | **Binder** |
|---------|:---:|:---:|:---:|:---:|
| 拷贝次数 | 2 | 0 | 2 | **1** (mmap) |
| C/S 架构 | ❌ | ❌ | ✅ | **✅** |
| 安全性（UID/PID校验） | ❌ | ❌ | ❌ | **✅** |
| 并发管理 | ❌ | ❌ | ❌ | **✅（线程池管理）** |

**后端类比**：

| Android | 后端概念 |
|---------|---------|
| Binder | 增强版 RPC 框架（gRPC、Thrift） |
| ServiceManager | 服务注册中心（Nacos、Consul、Eureka） |
| mmap 一次拷贝 | sendFile 零拷贝（Kafka 用它优化性能） |
| AIDL | gRPC 的 protobuf IDL（接口定义语言） |

---

## Binder 面试的「黄金回答」

> 「Binder 是 Android 的跨进程通信框架，基于 C/S 架构。Server 通过 ServiceManager 注册，Client 通过 ServiceManager 查到 Server 后直接通过 Binder 驱动通信。Binder 相比 Linux 传统 IPC 有三个优势：一是 mmap 只需要一次内存拷贝，性能更好；二是在内核层校验了 UID/PID，更安全；三是自动管理了 Binder 线程池，让服务端可以处理并发请求。从应用层看，我们通过 AIDL 定义接口，框架自动生成 Proxy/Stub，调用远程方法就像调本地方法一样。」

---

## 延伸追问

1. **AIDL 的 in/out/inout 区别？**
   → in：数据只从 client→server（默认）；out：数据只从 server→client；inout：双向（开销最大）。

2. **ServiceManager 为什么是 Binder 0 号？**
   → SM 是 Binder 机制的入口。地址硬编码为 0，所有进程都知道它的地址。类似 DNS 的根服务器和 `127.0.0.1`。

3. **Parcel 和 Serializable 的区别？**
   → Parcel 是 Android 自定义的序列化（更高效，但只用于 IPC）。Serializable 是 Java 标准的（反射，开销大，跨平台）。IPC 必须用 Parcelable。

4. **Binder 会竞争吗？多个 Client 同时请求怎么办？**
   → Binder 驱动为每个 Server 维护了线程池（默认 16 个线程）。多个 Client 请求并发处理，类似 Web 服务器的线程池模型。

5. **说说 Binder 的死亡通知机制？**
   → Client 可以注册 DeathRecipient。Server 进程意外死亡时，Binder 驱动会回调 Client 的 binderDied() 方法。用于清理资源和重新绑定服务。
