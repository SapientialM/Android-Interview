# 内存泄漏与 OOM

> 级别：⭐⭐ 高频。Android 面试几乎必问的一题。

---

## 面试官怎么问

- 「常见的 Android 内存泄漏场景有哪些？」
- 「Handler 为什么会造成内存泄漏？怎么解决？」
- 「LeakCanary 的原理？」
- 「OOM 有哪些常见原因？如何排查？」

---

## 回答框架

### 1. 内存泄漏的本质

**长生命周期的对象持有短生命周期对象的引用**，导致短生命周期对象无法被 GC 回收。

**后端类比**：连接池未关闭 → 连接耗尽；单例 Service 持有 Request Scope Bean → 无法回收。

### 2. 高频泄漏场景与解决方案

| 场景 | 原因 | 解决方案 |
|------|------|---------|
| **Handler** | 内部类持有 Activity，延迟消息未处理完 Activity 退出 | 静态内部类 + WeakReference，onDestroy 中 removeCallbacks |
| **单例持有 Context** | `MySingleton.getInstance(activity)`，Activity 被回收但单例还引用它 | 传 `ApplicationContext` 而不是 Activity Context |
| **匿名内部类** | Runnable/Thread/AsyncTask 是内部类，持有外部 Activity | 静态内部类 + WeakReference |
| **资源未关闭** | Cursor、InputStream、BroadcastReceiver 未注销 | try-finally 或 use{} 确保关闭 |
| **静态 View** | `static View view` — View 持有 Activity | 绝不静态持有 View |
| **WebView** | WebView 内部持有 Activity 引用，且 WebView 的销毁不是同步的 | 独立进程 + onDestroy 中 removeAllViews + destroy |

### 3. Handler 内存泄漏详解

```kotlin
// ❌ 问题代码
class MyActivity : Activity() {
    val handler = Handler(Looper.getMainLooper()) {
        updateUI()  // 隐式持有 MyActivity.this
        true
    }
    // handler.postDelayed({ ... }, 300000)  // 5分钟后执行
    // 用户 5 秒就退出了，Activity 无法被 GC
}

// ✅ 修复
class MyActivity : Activity() {
    private val handler = MyHandler(this)

    private class MyHandler(activity: MyActivity) : Handler(Looper.getMainLooper()) {
        private val weakRef = WeakReference(activity)
        override fun handleMessage(msg: Message) {
            weakRef.get()?.updateUI()  // Activity 被回收了就不处理
        }
    }

    override fun onDestroy() {
        handler.removeCallbacksAndMessages(null)  // 清空所有
        super.onDestroy()
    }
}
```

### 4. LeakCanary 原理

```
1. ObjectWatcher 用 WeakReference 包装要监控的对象
2. 对象应该被回收时，触发 GC
3. 如果 WeakReference 没被加入 ReferenceQueue → 对象没被回收
4. Dump HPROF（堆快照）
5. Hair（分析引擎）分析引用链找到最短的 GC Root 路径
6. 展示泄漏信息
```

**后端类比**：LeakCanary ≈ JVM HeapDump + MAT（Memory Analyzer Tool）的自动版。

### 5. OOM 常见原因

| 原因 | 排查方法 |
|------|---------|
| 大图加载 | Memory Profiler 查看 Bitmap 占比，ARGB_8888 图片 = 宽×高×4 字节 |
| 内存泄漏累积 | LeakCanary + Memory Profiler |
| 线程过多 | `adb shell cat /proc/<pid>/status` 看 Threads |
| 文件描述符泄漏 | `ls /proc/<pid>/fd | wc -l` |

---

## 后端类比

| Android | 后端 |
|---------|------|
| 内存泄漏 | 连接池/线程池未关闭 |
| LeakCanary | HeapDump + MAT 自动分析 |
| Bitmap OOM | 大文件上传 → 内存爆满 |
| Activity 泄漏 | 单例 Service 持有 Request Scope Bean |

---

## 延伸追问

1. **为什么用 ApplicationContext 而不是 Activity Context 可以避免泄漏？**
   → ApplicationContext 生命周期与 App 一致，不会被回收。Activity Context 在 Activity 销毁后就不该被引用了。如果单例持有 Activity Context，Activity 销毁后无法 GC。

2. **WeakReference 什么时候被回收？**
   → 只被 WeakReference 指向的对象，下一次 GC 时被回收。

3. **怎么在线上监控内存泄漏？**
   → 集成 LeakCanary 的 leakcanary-android-process 或自定义监控：定期采样 Activity 的 WeakReference→ 检查是否被回收 → 类似 LeakCanary 的机制。美团等大厂有自研框架（Matrix 等）。

4. **OOM 一定是内存泄漏造成的吗？**
   → 不一定。加载大图（瞬间峰值）也可能 OOM。这时需要用 `inSampleSize` 采样、`BitmapRegionDecoder` 分块加载。
