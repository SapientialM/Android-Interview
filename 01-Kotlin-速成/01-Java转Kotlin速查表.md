# Java 转 Kotlin 速查表

> 每个条目 = Java 写法 vs Kotlin 写法 + 一句话解释。
> 不需要背，当成字典查，写代码时对照即可。

---

## 1. 变量声明

```java
// Java
String name = "hello";
final String name2 = "world";  // 不可变
String nullable = null;        // 可以为 null
```

```kotlin
// Kotlin
var name = "hello"       // 可变
val name2 = "world"      // 不可变（推荐默认用 val）
var nullable: String? = null  // 必须显式声明 nullable
```

**关键差异**：Kotlin 把「是否可为 null」写进了类型系统。`String` 永远不为 null，`String?` 才可能为 null。

---

## 2. 空安全

```java
// Java
String s = null;
s.length();              // NullPointerException!
if (s != null) {
    s.length();
}
```

```kotlin
// Kotlin
val s: String? = null
s?.length          // 安全调用，返回 null 而不是 NPE
s!!.length         // 强制调用，如果为 null 抛 NPE（不推荐）
s?.length ?: 0     // Elvis 操作符：如果为 null 取默认值
```

**后端类比**：`Optional.ofNullable(s).map(String::length).orElse(0)`

---

## 3. 数据类 (data class)

```java
// Java — 需要手写或 Lombok
public class User {
    private String name;
    private int age;
    // 构造方法、getter、setter、equals、hashCode、toString...几十行
}
```

```kotlin
// Kotlin — 一行搞定
data class User(val name: String, val age: Int)
// 自动生成：构造函数、getter、equals、hashCode、toString、copy、componentN
```

**后端类比**：`data class` ≈ `@Data @AllArgsConstructor` (Lombok)，但 Kotlin 是语言级别的。

---

## 4. 类与继承

```java
// Java
public class Animal {
    private String name;
    public Animal(String name) { this.name = name; }
}

public class Dog extends Animal {  // 默认可以继承
    public Dog(String name) { super(name); }
}
```

```kotlin
// Kotlin
open class Animal(val name: String)  // open 表示可以继承
class Dog(name: String) : Animal(name)  // 用 : 代替 extends，调用父类构造

// 默认所有 class 都是 final，除非加 open
```

**为什么默认 final？** 有效 Java 建议「要么为继承设计并写好文档，要么禁止继承」。Kotlin 把这个实践变成了语言特性。

---

## 5. 对象声明（单例）

```java
// Java
public class Singleton {
    private static final Singleton INSTANCE = new Singleton();
    private Singleton() {}
    public static Singleton getInstance() { return INSTANCE; }
}
```

```kotlin
// Kotlin
object Singleton
// 使用：Singleton.method()
```

**后端类比**：`object` ≈ Spring `@Component` 注入的单例 Bean（但是纯语言级别，无依赖注入）。

---

## 6. 扩展函数

```java
// Java — 写一个工具类
public class StringUtils {
    public static boolean isPhoneNumber(String s) {
        return s.matches("\\d{11}");
    }
}
// 使用：StringUtils.isPhoneNumber("13800138000")
```

```kotlin
// Kotlin — 给 String 类「扩展」新方法
fun String.isPhoneNumber(): Boolean = this.matches(Regex("\\d{11}"))
// 使用："13800138000".isPhoneNumber()
```

**后端类比**：扩展函数 ≈ 用静态工具方法，但语法糖让你看起来像原生方法。不修改原始类，只是编译期转换。

---

## 7. 高阶函数与 Lambda

```java
// Java
List<String> names = Arrays.asList("a", "bb", "ccc");
names.stream()
    .filter(s -> s.length() > 1)
    .map(String::toUpperCase)
    .collect(Collectors.toList());
```

```kotlin
// Kotlin
val names = listOf("a", "bb", "ccc")
names.filter { it.length > 1 }
     .map { it.uppercase() }
// 不需要 .stream() 和 .collect()
// it 是 lambda 的默认参数名
```

**后端类比**：Kotlin 集合操作 ≈ Java Stream API，但语法更简洁（不需要 `.stream()`、`.collect()`）。

---

## 8. 作用域函数

Kotlin 有 5 个作用域函数，初学者最容易混淆。一张表说清楚：

| 函数 | 引用对象 | 返回值 | 典型场景 |
|------|:------:|:-----:|------|
| `let` | `it` | Lambda 结果 | 安全调用 `?.let {}` |
| `apply` | `this` | 对象本身 | 对象初始化配置 |
| `run` | `this` | Lambda 结果 | 对象配置 + 计算结果 |
| `also` | `it` | 对象本身 | 附加操作（日志、验证） |
| `with` | `this` | Lambda 结果 | 对一个对象做多个操作 |

```kotlin
// let — 最常用于安全调用
val name: String? = getName()
name?.let { println(it.length) }  // 只有不为 null 才执行

// apply — 初始化对象
val user = User().apply {
    name = "Tom"
    age = 18
}

// run — 配置并返回计算结果
val message = user.run {
    "name=$name, age=$age"
}
```

**记忆口诀**：`it`（let/also）vs `this`（apply/run/with），返回自己（apply/also）vs 返回结果（let/run/with）。

---

## 9. when 表达式

```java
// Java
switch (status) {
    case LOADING: return "加载中";
    case SUCCESS: return "成功";
    case ERROR: return "错误";
    default: return "未知";
}
```

```kotlin
// Kotlin — when 是表达式，有返回值
val text = when (status) {
    Status.LOADING -> "加载中"
    Status.SUCCESS -> "成功"
    Status.ERROR -> "错误"
}
// 不需要 break，自动匹配一个即退出
// 配合 sealed class 使用时，编译器会检查是否覆盖了所有分支
```

---

## 10. 密封类 (sealed class)

```java
// Java — 用 enum 或抽象类模拟
// enum 的问题：不能有不同字段
// 抽象类的问题：编译器不检查分支覆盖
```

```kotlin
// Kotlin — sealed class 是「受限的继承」
sealed class UiState {
    object Loading : UiState()
    data class Success(val data: List<NewsItem>) : UiState()
    data class Error(val message: String) : UiState()
    object Empty : UiState()
}

// 使用 when 时，编译器会提示你要覆盖所有分支
when (state) {
    is UiState.Loading -> showLoading()
    is UiState.Success -> showContent(state.data)
    is UiState.Error -> showError(state.message)
    is UiState.Empty -> showEmpty()
}
// 少写一个分支编译器就报错 — 这就是 sealed class 的价值
```

**后端类比**：sealed class ≈ 有限状态机 (FSM)，编译器保证状态覆盖的完整性。

---

## 11. 委托 (Delegation)

```kotlin
// by lazy — 懒加载（线程安全，只在第一次访问时初始化）
val retrofit: Retrofit by lazy {
    Retrofit.Builder()
        .baseUrl(BASE_URL)
        .build()
}

// by Delegates.observable — 属性变化监听
var name: String by Delegates.observable("初始值") { _, old, new ->
    println("name 从 $old 变为 $new")
}
```

**后端类比**：`by lazy` ≈ Spring `@Lazy` 或 `double-checked locking` 单例。

---

## 12. 字符串模板

```java
// Java
String msg = "Hello, " + name + "! You have " + count + " messages.";
String.format("Hello, %s! You have %d messages.", name, count);
```

```kotlin
// Kotlin
val msg = "Hello, $name! You have ${count} messages."
// $变量名 或 ${表达式}
```

---

## 13. 构造函数简化

```kotlin
// Kotlin 主构造函数直接写在类名后面
class User(
    val name: String,       // val = 属性 + getter
    var age: Int,           // var = 属性 + getter + setter
    private var phone: String  // private 只在本类可见
)
// 这一行 = Java 中的字段声明 + 构造函数参数 + 赋值 + getter/setter
```

---

## 14. 伴生对象 (companion object)

```java
// Java — static 方法和常量
public class Config {
    public static final String BASE_URL = "https://api.example.com";
    public static Config create() { return new Config(); }
}
```

```kotlin
// Kotlin — 没有 static 关键字，用 companion object
class Config {
    companion object {
        const val BASE_URL = "https://api.example.com"
        fun create() = Config()
    }
}
// 使用：Config.BASE_URL, Config.create()
```

---

## 15. 接口与默认方法

```kotlin
interface OnClickListener {
    fun onClick()                    // 抽象方法
    fun onLongClick() = {}           // 带默认实现的方法（Java 8+ 也有）
    val clickable: Boolean           // 接口中声明属性
        get() = true                 // 属性也可以有默认 getter
}
```

---

## 16. 集合：不可变 vs 可变

```kotlin
// 默认不可变（推荐）
val list = listOf(1, 2, 3)       // 不可变 List
val map = mapOf("a" to 1)        // 不可变 Map
val set = setOf(1, 2, 3)         // 不可变 Set

// 明确声明可变
val mutableList = mutableListOf(1, 2, 3)
mutableList.add(4)

// 适用场景：
// - 函数参数用不可变类型（告诉调用者我不会改你的数据）
// - ViewModel 内部状态用可变类型，暴露给 UI 时转成不可变
val _state = MutableStateFlow(UiState.Loading)  // 内部可变
val state: StateFlow<UiState> = _state.asStateFlow()  // 暴露不可变
```

**后端类比**：这就像 Java 中返回 `Collections.unmodifiableList()` 防止外部修改。

---

## 17. 协程快速预览

```kotlin
// 协程 = 轻量级异步任务
class NewsViewModel : ViewModel() {
    fun loadNews() {
        viewModelScope.launch {                // 在 ViewModel 的作用域中启动协程
            val news = withContext(Dispatchers.IO) {  // 切换到 IO 线程
                api.getNews()                          // 网络请求（在 IO 线程）
            }
            _state.value = UiState.Success(news)       // 更新 UI（回到 Main 线程）
        }
    }
}
```

**后端类比**：
- `launch {}` ≈ `CompletableFuture.runAsync()`
- `withContext(Dispatchers.IO) {}` ≈ 在线程池上执行，完后自动切回主线程
- `viewModelScope` ≈ 与 Bean 生命周期绑定的线程池（Bean 销毁时自动取消所有任务）

详见 [02-协程核心概念](02-协程核心概念.md)。

---

## 18-20. 其他速查

| 条目 | Java | Kotlin |
|------|------|--------|
| 类型检测 | `if (x instanceof String)` | `if (x is String)` — 自动转型 |
| for 循环 | `for (int i = 0; i < n; i++)` | `for (i in 0 until n)` / `for (item in list)` |
| 默认值 | 方法重载 | `fun greet(name: String = "World")` — 默认参数 |

---

## 一分钟速记卡

1. `val` = 不可变，`var` = 可变（默认用 `val`）
2. `?` = 可为 null，`!!` = 强行非空（少用）
3. `?.` = 安全调用，`?:` = Elvis（默认值）
4. `data class` = Lombok @Data（自动生成 getter/setter/equals/hashCode）
5. `object` = 单例
6. `when` = 增强版 switch（有返回值，配合 sealed class 用）
7. `sealed class` = 有限状态机（编译器检查分支覆盖）
8. `it` = Lambda 默认参数名
9. `by lazy` = 懒加载
10. `companion object` = static 替代
