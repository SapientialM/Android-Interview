# Android 系统启动与 AMS

> 级别：⭐⭐⭐ 进阶（大厂面试区分度题）。非必考，但答出来很加分。

---

## 面试官怎么问

- 「Android 系统启动流程？」
- 「AMS 的作用和工作原理？」
- 「App 的启动过程（从点击图标到 Activity 显示）？」

---

## 1. Android 系统启动流程

```
Bootloader → Kernel 启动 → init 进程 (PID 1)
  → Zygote 进程（fork 自 init，预加载 Java 运行时和系统类）
    → SystemServer 进程（fork 自 Zygote）
      → 启动 AMS, PMS, WMS 等系统服务
        → 启动 Launcher（桌面）

各进程的作用：
  init：解析 init.rc，启动守护进程（servicemanager、adbd 等）
  Zygote：Android 的「母进程」，预加载系统类和资源
          后续所有 App 进程都 fork 自 Zygote（Copy-On-Write 加速）
  SystemServer：Java 层的系统服务管理器（AMS, PMS, WMS 等）
```

**后端类比**：
- init ≈ Linux init / systemd（系统启动的第一个用户态进程）
- Zygote ≈ 预热的 JVM 模板（fork 快，COW 省内存）
- SystemServer ≈ Spring 的 ApplicationContext（管理所有 Bean/服务）
- AMS ≈ 进程管理器 + 调度器（调度 Activity 的生命周期）

---

## 2. AMS（ActivityManagerService）

**一句话**：AMS 是 Android 的「大总管」，管理所有 App 的四大组件生命周期。

**核心功能**：
1. **Activity 生命周期管理**：调度 Activity 的启动、暂停、销毁
2. **进程管理**：决定哪个进程应该存活、什么时候杀死（基于 OOM_ADJ）
3. **Task 和返回栈管理**：管理 Activity 在哪个 Task 里
4. **Broadcast 分发**：有序广播的队列管理

---

## 3. App 启动流程（点击图标 → Activity 显示）

```
1. Launcher 点击图标 → startActivity(Intent)
2. Launcher 通过 Binder IPC 通知 AMS
3. AMS 检查目标 Activity 的进程是否已存在
4. 如果不存在 → AMS 通知 Zygote fork 新进程
   (新进程创建后，Zygote 通过 Copy-On-Write 共享系统类)
5. 新进程启动 → ActivityThread.main()
   → Looper.prepareMainLooper()
     → 绑定到 AMS (Binder IPC)
6. AMS 通过 Binder IPC 发送「启动 Activity」指令
7. ActivityThread 收到指令 → 创建 Application → 创建 Activity
   → Activity.onCreate() → setContentView → 绘制首帧
8. 用户看到页面
```

**面试简化版**：

> 点击图标 → Launcher 通知 AMS → AMS 让 Zygote fork 新进程 → 新进程初始化 → 创建 Application → 创建 Activity → 绘制首帧。

---

## 4. OOM_ADJ（进程优先级）

系统内存不足时，按 OOM_ADJ 从高到低杀进程：

| 优先级 | ADJ 值 | 进程类型 |
|:---:|:---:|------|
| 最高 | 0 | 前台 Activity |
| ↑ | 200 | 可见进程（如 getVisibleTasks） |
| ↑ | 700 | 后台 Service |
| ↑ | 900 | 后台缓存进程（上一个 App） |
| 最低 | 906+ | 空进程 |

---

## 5. 其他核心服务一览

| 服务 | 全称 | 职责 |
|------|------|------|
| AMS | ActivityManagerService | Activity/进程管理 |
| PMS | PackageManagerService | APK 安装/卸载/权限/IntentFilter 匹配 |
| WMS | WindowManagerService | 窗口管理/动画/输入事件中转 |
| IMS | InputManagerService | 输入事件处理（触摸、按键） |
| PKMS | — | 等同于 PMS（Package 管理） |

---

## 后端类比

| Android | 后端 |
|---------|------|
| Zygote | 预热的 JVM 模板（fork 快，COW 省内存） |
| AMS | 服务编排器（Orchestrator）+ 生命周期管理器 |
| PMS | 包管理器（APT/RPM）+ 权限管理 |
| App 启动 | 微服务冷启动：创建进程 → 初始化运行时 → 加载配置 → 就绪 |
| OOM_ADJ | 服务的 eviction priority |

---

## 延伸追问

1. **为什么 App 从 Zygote fork 而不是 new 进程？**
   → fork 继承 Zygote 已预加载的系统类和资源（通过 Copy-On-Write），速度极快。new 进程需要重新加载 JVM、系统类等，要好几秒。

2. **SystemServer 里有多少个服务线程？**
   → 100+ 个系统服务（AMS、PMS、WMS、NotificationManager、BatteryService 等）。这些服务都运行在 SystemServer 进程中。

3. **Application.onCreate 和 Activity.onCreate 分别在哪个 Binder 线程？**
   → 都在主线程（ActivityThread 的 main 线程）。所以 Application.onCreate 不要太久，会阻塞所有 Activity 的启动。

4. **怎么加速 App 的启动？**
   → Application.onCreate 尽量轻（延迟初始化），spalsh 预加载主题，主 Activity 布局简化。
