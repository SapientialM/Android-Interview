# Kotlin 常见陷阱

> Java 转 Kotlin 最容易踩的坑。遇到报错时先来这看看。

---

## 1. 伴生对象 ≠ static

```kotlin
// ❌ 误解：companion object 就是 static
class Foo {
    companion object {
        fun bar() {}
    }
}
// Kotlin 调用：Foo.bar()
// Java 调用：Foo.Companion.bar()  ← 注意：调用方式不一样

// ✅ 如果想在 Java 中像 static 一样调用，加 @JvmStatic
class Foo {
    companion object {
        @JvmStatic
        fun bar() {}
    }
}
// Java 调用：Foo.bar()  ← 现在可以了
```

---

## 2. Lambda 和 SAM 转换

```java
// Java — 接口只有一个抽象方法时，可以用 Lambda
button.setOnClickListener(v -> doSomething());
```

```kotlin
// Kotlin — 对于 Java 接口（SAM）可以直接用 Lambda
button.setOnClickListener { doSomething() }  // ✅ OK

// ❌ 但 Kotlin 定义的接口不能直接转 Lambda！
interface Callback {
    fun onResult(data: String)
}
// fun register(callback: Callback) {}
// register { data -> println(data) }  // ❌ 编译错误！

// ✅ 方案一：用 Lambda 类型替代接口
// fun register(callback: (String) -> Unit) {}
// register { data -> println(data) }  // ✅ OK

// ✅ 方案二：接口前加 fun 关键字
fun interface Callback {
    fun onResult(data: String)
}
// register { data -> println(data) }  // ✅ OK（需要 Kotlin 1.4+）
```

---

## 3. 集合是不可变的

```kotlin
val list = listOf(1, 2, 3)
// list.add(4)  // ❌ 编译错误！listOf 返回的是不可变 List

val mutableList = mutableListOf(1, 2, 3)
mutableList.add(4)  // ✅ OK

// ⚠️ 但不可变 List 只是接口层面的不可变，底层可能被修改
val items = mutableListOf("a", "b")
val view = items as List<String>  // 转成不可变类型
// items.add("c")  // 底层被修改了，view 也会变

// 真正需要不可变时，用 toList() 拷贝一份
val safeView = items.toList()
```

---

## 4. lateinit vs lazy 用错

```kotlin
// lateinit：延迟初始化 var，用于 DI 注入
@Inject lateinit var repository: NewsRepository
// 使用前手动赋值，访问未初始化的会抛 UninitializedPropertyAccessException

// by lazy：延迟初始化 val，用于需要时才创建
val retrofit: Retrofit by lazy {
    Retrofit.Builder()
        .baseUrl(BASE_URL)
        .build()
}
// 第一次访问时执行 lambda，结果是 val（不可变）

// ✅ 正确选择：
// - Hilt/Hilt 注入的字段 → lateinit var（框架负责赋值）
// - 创建成本高但不是必需的对象 → by lazy
// - 普通数据 → 直接 val
```

---

## 5. 泛型 out/in 混淆

```kotlin
// Java 中的通配符：List<? extends Animal>（生产者，只能读）
// Kotlin 中：List<out Animal>（out = 生产者 = 只能读）

// Java 中的通配符：List<? super Animal>（消费者，只能写）
// Kotlin 中：List<in Animal>（in = 消费者 = 只能写）

// 记忆口诀：PECS → Producer-Extends, Consumer-Super
// Kotlin 版本：Producer-Out, Consumer-In

// 常见示例
fun copy(from: List<out Animal>, to: MutableList<in Animal>) {
    for (animal in from) {     // from 读（out）
        to.add(animal)          // to 写（in）
    }
}
```

---

## 6. when 不够安全

```kotlin
// ❌ when 后面跟普通类时，编译器不强制检查所有分支
val result = when (value) {
    1 -> "one"
    2 -> "two"
    // 忘记写 else，编译能过，但运行时可能抛异常
}

// ✅ 配合 sealed class 使用时，编译器会强制检查所有分支
sealed class Result {
    data class Success(val data: String) : Result()
    data class Error(val error: String) : Result()
}
val result = when (sealedResult) {
    is Result.Success -> sealedResult.data
    is Result.Error -> sealedResult.error
    // 少一个分支编译器就报错 — 这才安全
}
```

---

## 7. null 安全问题

```kotlin
// ⚠️ !! 不要习惯性使用
val name: String? = null
// name!!.length  // 如果 name 是 null，NPE！

// ✅ 教你如何优雅地处理 null
name?.length                     // 安全调用
name?.length ?: 0                // 安全调用 + 默认值
name?.let { process(it) }        // 不为 null 时才执行
requireNotNull(name)             // 前置条件检查，抛 IllegalArgumentException
checkNotNull(name)               // 状态检查，抛 IllegalStateException
```

---

## 8. 作用域函数嵌套

```kotlin
// ❌ 作用域函数嵌套导致 this 混淆
user?.apply {
    name = "Tom"          // this = user
    address?.apply {
        city = "Beijing"  // 这里的 this 是谁？（address）
    }
}

// ✅ 用 let/it 避免混淆
user?.let { user ->
    user.name = "Tom"
    user.address?.let { address ->
        address.city = "Beijing"
    }
}
// 或者用命名参数增加可读性
```

---

## 9. equals 和 ==

```kotlin
// Java 中：== 比较引用，equals 比较内容
// Kotlin 中：== 就是 equals（内容比较），=== 才是引用比较

val a = "hello"
val b = "hello"
a == b   // true — 内容相等
a === b  // true 或 false — 引用相等（取决于字符串池）

// 这是 Java 开发者最容易误会的点
// Kotlin 的 == 更安全，想做引用比较显式用 ===
```

---

## 10. 扩展函数不能重写

```kotlin
open class Parent
class Child : Parent()

fun Parent.say() = "Parent"
fun Child.say() = "Child"

val child: Parent = Child()
println(child.say())  // 输出 "Parent"，不是 "Child"！

// 扩展函数是静态分发的，没有多态
// 调用哪个版本取决于变量的声明类型，不是运行时类型
```

---

## 11. 协程相关陷阱

```kotlin
// ❌ 在非协程作用域中调用 suspend 函数
// fun foo() {
//     api.getData()  // 编译错误：suspend 函数只能在协程或另一个 suspend 函数中调用
// }

// ❌ GlobalScope 不要用（没有生命周期绑定，内存泄漏风险）
// GlobalScope.launch { }

// ✅ 始终使用有生命周期绑定的 scope
// viewModelScope.launch { }    — ViewModel 中
// lifecycleScope.launch { }    — Activity/Fragment 中
// rememberCoroutineScope()     — Composable 中

// ⚠️ 注意 launch 和 async 的异常处理方式不同
viewModelScope.launch {
    // launch 抛异常 → 冒泡到 scope（如果没有 handler 可能 crash）
}

viewModelScope.async {
    // async 抛异常 → 在 await() 时才抛
}
```

---

## 速查清单

遇到问题时按这个清单排查：

- [ ] `null` 问题 → 检查类型声明是否带 `?`
- [ ] 调用不存在的 API → 检查是否用了 `companion object` 中的方法，Java 侧需加 `@JvmStatic`
- [ ] Lambda 转换失败 → 接口是否加了 `fun interface`，或者直接用 `(Param) -> Return` 类型
- [ ] 集合修改不了 → 是否用了 `listOf` 而不是 `mutableListOf`
- [ ] 协程编译错误 → 是否在非协程作用域中调用 `suspend` 函数
- [ ] 作用域函数行为奇怪 → 检查 `this` 引用是否被覆盖
