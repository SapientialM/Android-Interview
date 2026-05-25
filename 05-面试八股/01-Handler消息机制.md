# Handler 消息机制

> 级别：⭐⭐⭐ 必考。Android 面试出现率接近 100%。

---

## 面试官怎么问

- 「讲讲 Handler 的原理？」
- 「为什么主线程不会因为 Looper.loop() 卡死？」
- 「Handler 造成内存泄漏的原因和解决方案？」
- 「IdleHandler 是什么？你用过吗？」

---

## 回答框架

### 1. 一句话概括

Handler 是 Android **线程间通信**的核心机制。它让一个线程可以向另一个线程的消息队列中发送消息，并在目标线程中处理。

### 2. 四件套

```
Handler       → 发送/处理消息
Message       → 消息体（what, arg1, arg2, obj, data）
MessageQueue  → 消息队列（按时间排序的单链表）
Looper        → 消息循环（轮询器，不断从队列取消息）
```

### 3. 完整流程

```
1. Looper.prepare()
   → 创建 MessageQueue
   → 绑定到 ThreadLocal（每个线程只有一个 Looper）

2. Looper.loop()
   → 死循环：queue.next() 取消息
   → msg.target.dispatchMessage(msg)
   → handleMessage()

3. Handler.sendMessage(msg)
   → msg.target = handler
   → MessageQueue.enqueueMessage(msg)

4. Looper 取出 msg
   → msg.target.dispatchMessage(msg)
   → handler.handleMessage(msg)
```

### 4. 主线程 Looper 为什么不会卡死？

主线程的 `Looper.loop()` 确实是死循环，但：
- `MessageQueue.next()` 内部调用 `nativePollOnce()`，底层使用 **Linux epoll** 机制
- 没消息时线程**阻塞/休眠**，不消耗 CPU，不是忙等(busy waiting)
- 有消息来时 epoll 唤醒线程，继续处理

**后端类比**：这和 Netty 的 EventLoop 用 epoll 监听 IO 事件完全一致。线程没活干时阻塞在 `epoll_wait()`，有活干时才被唤醒。所以虽然是「死循环」，但实际大部分时间在睡觉。

### 5. Handler 造成的内存泄漏

**原因**：
```kotlin
// ❌ 内部类隐式持有外部 Activity 引用
class MyActivity : Activity() {
    val handler = Handler(Looper.getMainLooper()) {
        updateUI()  // 持有 MyActivity.this
        true
    }
    // handler 发了延迟消息，Activity 退出时消息还在队列
    // → Handler → Activity 引用链 → Activity 无法被 GC
}
```

**解决方案**：
```kotlin
// ✅ 方案一：静态内部类 + WeakReference
class MyActivity : Activity() {
    private val handler = MyHandler(this)

    private class MyHandler(activity: MyActivity) : Handler(Looper.getMainLooper()) {
        private val weakRef = WeakReference(activity)

        override fun handleMessage(msg: Message) {
            weakRef.get()?.updateUI()  // Activity 被回收了就安全返回
        }
    }

    override fun onDestroy() {
        super.onDestroy()
        handler.removeCallbacksAndMessages(null)  // 清空所有消息
    }
}

// ✅ 方案二：Lifecycle 感知
lifecycle.addObserver(object : DefaultLifecycleObserver {
    override fun onDestroy(owner: LifecycleOwner) {
        handler.removeCallbacksAndMessages(null)
    }
})
```

### 6. 同步屏障 (SyncBarrier)

Handler 消息分为同步消息和异步消息。同步屏障是一道「拦截所有同步消息、只放行异步消息」的屏障。

典型应用：**屏幕刷新**。VSync 信号到来时 Choreographer 发异步消息给主线程，同步屏障确保绘制优先执行（跳过其他普通消息）。

**面试提示**：能提到「同步屏障」是加分项，不需要解释太深。

---

## 后端类比

| Android | Java 后端 |
|---------|----------|
| Handler | Producer/Consumer 中的 Producer |
| MessageQueue | BlockingQueue（用 epoll 而不是锁实现阻塞） |
| Looper | EventLoop（死循环轮询队列） |
| Message | Runnable / Callable |
| HandlerThread | 自带 Looper 的后台线程 ≈ 单线程 EventLoop |

---

## 延伸追问

1. **子线程可以创建 Handler 吗？**
   → 需要先 `Looper.prepare()` 和 `Looper.loop()`。或直接用 `HandlerThread`（内部已帮你做了）。

2. **为什么 Android 推荐用 Handler 而不是直接加锁？**
   → Handler 将并发问题简化为消息队列的串行处理。生产者只管发消息，消费者专心处理，天然线程安全。加锁方案容易死锁。

3. **Message 的复用机制？**
   → `Message.obtain()` 从全局回收池取（spool，最大 50 个），`recycle()` 回收。避免频繁 new Message 造成的 GC 压力。类比：对象池 / ThreadPool 的线程复用。

4. **HandlerThread 和 IntentService 的区别？**
   → HandlerThread 是自带 Looper 的后台线程。IntentService 是 Service + HandlerThread 的封装，执行完任务自动停止。现在基本被协程替代。

5. **为什么 View.post() 能保证拿到 View 的宽高？**
   → `View.post(Runnable)` 实际通过 Handler 发送到主线程 MessageQueue。View 的绘制（measure/layout/draw）也是主线程消息。所以 View.post 的 Runnable 排在绘制之后执行，此时宽高已确定。
