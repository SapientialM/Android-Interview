# 网络层设计：Retrofit + OkHttp

> 面试重点 ⭐⭐⭐。Android 网络层的两大核心库：Retrofit（接口定义 + 动态代理）+ OkHttp（真正的 HTTP 客户端）。
> **后端类比**：Retrofit ≈ OpenFeign，OkHttp ≈ Apache HttpClient。两者的链式拦截器设计 ≈ Spring Security Filter Chain。

---

## 1. Retrofit 基本使用

```kotlin
// 1. 定义 API 接口（类比 Spring Feign Client 的 @FeignClient）
interface NewsApi {
    @GET("api/news")
    suspend fun getNews(
        @Query("page") page: Int,
        @Query("size") size: Int = 20
    ): ApiResponse<List<NewsDto>>

    @GET("api/news/{id}")
    suspend fun getNewsDetail(
        @Path("id") id: String
    ): ApiResponse<NewsDetailDto>
}

// 2. 创建 Retrofit 实例
val retrofit = Retrofit.Builder()
    .baseUrl("https://api.example.com/")
    .addConverterFactory(GsonConverterFactory.create())  // JSON→对象
    .client(okHttpClient)
    .build()

// 3. 创建 API 实例（动态代理）
val newsApi: NewsApi = retrofit.create(NewsApi::class.java)

// 4. 使用
val news = newsApi.getNews(page = 1)
```

**Retrofit 怎么工作的？** 编译期动态代理。你调用 `newsApi.getNews(1)`，Retrofit 内部：
1. 读取 `@GET("api/news")` → 构建 HTTP 请求
2. 读取 `@Query("page")` → 拼接 query 参数
3. 解析返回值类型 `ApiResponse<List<NewsDto>>` → 用 GsonConverterFactory 转换 JSON
4. 通过 OkHttp 发送请求 → 拿到响应 → 返回

**后端类比**：和 OpenFeign 完全一致。`@FeignClient` + `@GetMapping` + `@RequestParam` → 动态代理生成 HTTP 调用。

---

## 2. OkHttp 拦截器链 (Interceptor Chain)

```kotlin
val okHttpClient = OkHttpClient.Builder()
    .addInterceptor(LoggingInterceptor())       // 应用拦截器
    .addNetworkInterceptor(CacheInterceptor())  // 网络拦截器
    .connectTimeout(15, TimeUnit.SECONDS)
    .readTimeout(15, TimeUnit.SECONDS)
    .build()
```

**拦截器链的执行顺序**：

```
请求 ↓
  → 应用拦截器 (Request 阶段)     # 日志、统一 Header、重试
    → 网络拦截器 (Request 阶段)   # 缓存策略、桥接
      → OkHttp 核心引擎            # DNS、连接池、协议选择
    → 网络拦截器 (Response 阶段)
  → 应用拦截器 (Response 阶段)
响应 ↑
```

**后端类比**：OkHttp 拦截器链 ≈ Spring Security Filter Chain 或 Servlet Filter 链。每层只关心自己的职责，链式调用。

---

## 3. 统一错误处理

```kotlin
// 定义数据层的结果封装
sealed class Result<out T> {
    data class Success<T>(val data: T) : Result<T>()
    data class Error(val message: String, val code: Int? = null) : Result<Nothing>()
}

// Repository 中处理
class NewsRepository @Inject constructor(
    private val api: NewsApi
) {
    suspend fun getNews(page: Int): Result<List<NewsItem>> {
        return try {
            val response = api.getNews(page)
            if (response.isSuccess) {
                Result.Success(response.data.map { it.toDomain() })
            } else {
                Result.Error(response.message, response.code)
            }
        } catch (e: IOException) {
            Result.Error("网络错误，请检查网络连接")
        } catch (e: Exception) {
            Result.Error("未知错误：${e.message}")
        }
    }
}
```

---

## 4. Token 管理（Authenticator）

```kotlin
// 自动刷新 Token 的 Authenticator
class TokenAuthenticator @Inject constructor(
    private val authApi: AuthApi,
    private val tokenStore: TokenStore
) : Authenticator {
    override fun authenticate(route: Route?, response: Response): Request? {
        if (response.code == 401) {
            val newToken = runBlocking {
                refreshToken()
            }
            if (newToken != null) {
                return response.request.newBuilder()
                    .header("Authorization", "Bearer $newToken")
                    .build()
            }
        }
        return null  // 放弃刷新
    }
}
```

**后端类比**：Authenticator ≈ Spring Security 的 `AuthenticationEntryPoint` + Token 刷新逻辑。

---

## 5. 项目中网络层的标准分层

```
NetworkModule (Hilt DI)
  └─ OkHttpClient（连接池、拦截器、超时）
     └─ Retrofit（baseUrl、ConverterFactory）
        └─ NewsApi（接口定义）

Repository
  └─ 调用 NewsApi
  └─ 错误统一处理（try-catch → Result）

ViewModel
  └─ 调用 Repository
  └─ 转换数据（DTO → Domain Model）
```

---

## 面试速记

| 问题 | 回答 |
|------|------|
| Retrofit 原理 | 动态代理：接口+注解 → 自动构建 HTTP 请求 + JSON 反序列化（和 OpenFeign 一样） |
| OkHttp 拦截器链 | App Interceptor → Network Interceptor → 核心引擎（类比 Filter Chain） |
| 你的项目怎么处理网络错误？ | Repository 层 try-catch + Result 封装 → ViewModel 转换 → UiState.Error |
| Token 怎么自动刷新？ | OkHttp Authenticator 检测 401 → 刷新 → 重试原请求 |
