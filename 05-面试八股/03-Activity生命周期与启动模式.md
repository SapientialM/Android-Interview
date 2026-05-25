# Activity 生命周期与启动模式

> 级别：⭐⭐ 高频。Android 面试基础题。

---

## 面试官怎么问

- 「Activity 的生命周期？」
- 「A 启动 B 后，A 和 B 的生命周期回调顺序？」
- 「四种启动模式的区别和应用场景？」
- 「onSaveInstanceState 和 onRestoreInstanceState 的调用时机？」

---

## 回答框架

### 1. 完整生命周期

```
onCreate → onStart → onResume → (Running)
                                       ↓
                                  onPause → onStop → onDestroy

onStop → onRestart → onStart（重新回到前台）
```

| 回调 | 说明 | 常见操作 |
|------|------|---------|
| onCreate | Activity 被创建 | setContentView, 初始化 ViewModel |
| onStart | 即将可见 | 注册 BroadcastReceiver |
| onResume | 已经可见且可交互 | 开始动画、相机预览 |
| onPause | 部分不可见（另一 Activity 覆盖） | 暂停动画、保存草稿 |
| onStop | 完全不可见 | 释放资源 |
| onDestroy | Activity 被销毁 | 清理所有资源 |
| onRestart | onStop → onStart 之间 | 重新加载数据 |

### 2. A 启动 B 的回调顺序（面试区分度最高的题）

```
A (Standard) 启动 B (Standard)：

A.onPause() → B.onCreate() → B.onStart() → B.onResume() → A.onStop()
```

**记忆口诀**：A 的 Pause 先走，B 全创建，A 最后停。

为什么这个顺序？A 先暂停（释放资源）→ B 创建（更快显示）→ 等 B 显示好了 A 再完全停止。这是为了让新 Activity 尽快可见。

**B 返回 A 的顺序**：

```
B.onPause() → A.onRestart() → A.onStart() → A.onResume() → B.onStop() → B.onDestroy()
```

**对话框弹出不会触发 A.onStop()**，因为 A 仍然部分可见。

### 3. 四种启动模式

| 模式 | 行为 | 典型场景 |
|------|------|------|
| **Standard** | 每次新建实例，不检查栈 | 默认，适合大多数页面 |
| **SingleTop** | 栈顶复用（调 onNewIntent），不新建 | 通知点击跳转到已打开的页面 |
| **SingleTask** | 栈内复用（清掉上面的），调 onNewIntent | 主页/首页（全局只有一个） |
| **SingleInstance** | 独占一个新栈 | 极少用，如系统电话、闹钟 |

**SingleTask 的特殊行为**：如果目标 Activity 在其他 Task 中，会把整个 Task 切到前台。

### 4. onSaveInstanceState vs ViewModel

| 对比 | onSaveInstanceState | ViewModel |
|------|-------------------|-----------|
| 存活场景 | 进程被杀（系统回收内存） | 屏幕旋转、配置变更 |
| 数据格式 | Bundle（序列化，有大小限制） | 任意对象（内存中） |
| 用途 | 保存少量临时状态（如输入框文字） | 保存页面业务数据 |
| 触发条件 | 系统随时可能调（进入后台前） | Activity finish() 才清理 |

**结论**：两者配合使用。
- ViewModel 存业务数据（进程死亡后重建时重新加载）
- onSaveInstanceState 存轻量的临时状态（滚动位置、输入内容）

---

## 后端类比

| Activity 概念 | Spring 对应 |
|-------------|-----------|
| 生命周期回调 | Servlet/Filter 生命周期（init → service → destroy） |
| 启动模式 | Bean 的 Scope（Singleton=SingleTask, Prototype=Standard） |
| Intent 跳转 | 路由转发（Controller 返回 redirect） |
| onSaveInstanceState | @SessionAttributes 暂存少量数据 |
| ViewModel | @SessionScope Bean |

---

## 延伸追问

1. **Activity 重建时（如旋转），ViewModel 怎么存活？**
   → ViewModel 存在 ViewModelStore 中，用 `setRetainInstance` 机制。Activity 重建时 ViewModelStore 保留，新 Activity 从中取回同一个 ViewModel。

2. **SingleTask 和 taskAffinity 的关系？**
   → taskAffinity 决定 Activity 进入哪个 Task。默认 TaskAffinity 是包名。SingleTask 会查找有相同 TaskAffinity 的 Task 并进入。

3. **onNewIntent 什么时候被调用？**
   → SingleTop（栈顶复用）或 SingleTask（栈内复用）时，新的 Intent 通过 onNewIntent 传给已有 Activity，而不是重新创建。

4. **Activity 的 onStop 一定会被调用吗？**
   → 不一定。如果系统内存极低，可能直接 kill 进程，onStop/onDestroy 都不回调。

5. **Fragment 的生命周期和 Activity 的关系？**
   → Fragment 有自己的生命周期（onAttach → onCreateView → onDestroyView → onDetach），受父 Activity 生命周期影响。Activity onStop → Fragment onStop。
