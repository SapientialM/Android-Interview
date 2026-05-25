# View 事件分发与绘制流程

> 级别：⭐ 基础（Compose 时代重要性降低，但仍可能被问到）。
> 注意：Compose 项目不会用到 View 事件分发机制，面试中如果被问可以说「项目用 Compose 所以没深入，但了解基本原理」。

---

## 面试官怎么问

- 「View 事件分发机制？」
- 「View 的绘制流程？」
- 「自定义 View 需要重写哪些方法？」

---

## 1. 事件分发机制（dispatchTouchEvent → onInterceptTouchEvent → onTouchEvent）

```
Activity.dispatchTouchEvent
   → ViewGroup.dispatchTouchEvent
     → ViewGroup.onInterceptTouchEvent  (是否拦截？)
       → 不拦截 → child.dispatchTouchEvent
         → child.onTouchEvent  (消费？)
       → 拦截 → ViewGroup.onTouchEvent
       → 都不消费 → Activity.onTouchEvent
```

| 方法 | 作用 |
|------|------|
| dispatchTouchEvent | 事件分发入口 |
| onInterceptTouchEvent | ViewGroup 拦截（抢走事件） |
| onTouchEvent | 处理/消费事件 |

**经典案例**：RecyclerView 中的 Button 点击。按下时 RecyclerView 不拦截 → 事件交给 Button → Button 消费。滑动时 RecyclerView 拦截 → 事件自己处理 → 列表滚动。

---

## 2. View 绘制流程（measure → layout → draw）

```
1. measure (测量)：
   onMeasure() → 确定宽高
   MeasureSpec：模式(EXACTLY/AT_MOST/UNSPECIFIED) + 尺寸

2. layout (布局)：
   onLayout() → 确定位置（left, top, right, bottom）

3. draw (绘制)：
   onDraw() → 画出来
   顺序：背景 → 自身内容 → 子 View → 装饰（滚动条）
```

**触发入口**：`ViewRootImpl.performTraversals()`，由 VSync 信号驱动。

---

## 3. 自定义 View

```kotlin
class MyView @JvmOverloads constructor(
    context: Context,
    attrs: AttributeSet? = null
) : View(context, attrs) {

    override fun onMeasure(widthMeasureSpec: Int, heightMeasureSpec: Int) {
        // 1. 确定自己的宽高
        val width = resolveSize(desiredWidth, widthMeasureSpec)
        val height = resolveSize(desiredHeight, heightMeasureSpec)
        setMeasuredDimension(width, height)
    }

    override fun onLayout(changed: Boolean, left: Int, top: Int, right: Int, bottom: Int) {
        // 2. ViewGroup 需要放置子 View（普通 View 不需要）
    }

    override fun onDraw(canvas: Canvas) {
        // 3. 画自己的内容
        canvas.drawCircle(centerX, centerY, radius, paint)
    }
}
```

**但在 Compose 中**：自定义 View 改为自定义 Composable 函数组合，不需要继承 View 类。

---

## 后端类比

| View 概念 | 类比 |
|-----------|------|
| 事件分发 | Spring 拦截器链（HandlerInterceptor），决定谁处理请求 |
| measure/layout/draw | 模板引擎的渲染流程：计算尺寸 → 确定位置 → 渲染输出 |
| onTouchEvent | Controller 的 handler method |
| VSync | 定时任务触发渲染（类似 Cron Job 触发批处理） |

---

## 延伸追问

1. **requestLayout、invalidate、postInvalidate 的区别？**
   → requestLayout：重新 measure + layout + draw。invalidate：只重新 draw（形状没变，内容变了）。postInvalidate：非 UI 线程调用 invalidate。

2. **MeasureSpec 有三种模式？**
   → EXACTLY（match_parent 或具体值）、AT_MOST（wrap_content）、UNSPECIFIED（不限，ScrollView 内部）。

3. **为什么在 onCreate 中拿不到 View 的宽高？**
   → onCreate 时 View 还没 measure。解决方案：View.post { } -> Handler 把 Runnable 放到 measure 之后的队列中。

4. **Compose 项目还需要学这个吗？**
   → Compose 有自己的布局和绘制系统，不依赖 View 树。但面试中仍可能被问 View 事件分发。了解基本原理即可，不需要深入源码。
