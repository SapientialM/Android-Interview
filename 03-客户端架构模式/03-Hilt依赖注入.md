# Hilt 依赖注入

> Hilt 是 Android 的依赖注入框架，基于 Dagger 但大幅简化了配置。
> **后端类比**：Hilt ≈ Spring IoC 容器（精简版），@HiltAndroidApp ≈ @SpringBootApplication, @Module ≈ @Configuration, @Provides ≈ @Bean, @Inject ≈ @Autowired。

---

## 1. 为什么需要依赖注入？

```kotlin
// ❌ 不用 DI — 手动创建，耦合严重
class NewsViewModel {
    val repository = NewsRepository(
        NewsApi(Retrofit.Builder().baseUrl("...").build())
    )
}

// ✅ 用 Hilt DI — 自动注入，解耦
@HiltViewModel
class NewsViewModel @Inject constructor(
    private val repository: NewsRepository  // Hilt 自动提供
) : ViewModel()
```

---

## 2. Hilt 核心注解速查

| 注解 | 作用 | Spring 类比 |
|------|------|-----------|
| `@HiltAndroidApp` | 标记 Application 类，触发 Hilt 代码生成 | `@SpringBootApplication` |
| `@AndroidEntryPoint` | 标记 Activity/Fragment，允许注入 | `@RestController` |
| `@HiltViewModel` | 标记 ViewModel，允许注入 | `@Component` |
| `@Inject constructor` | 告诉 Hilt 如何创建这个类的实例 | `@Autowired` 构造函数注入 |
| `@Module` | 声明一个提供依赖的模块 | `@Configuration` |
| `@InstallIn` | 指定模块安装到哪个 Component | — |
| `@Provides` | 提供实例（工厂方法） | `@Bean` |
| `@Binds` | 绑定接口到实现 | `@Service` / `@Primary` |
| `@Singleton` | 全局单例 | `@Singleton` / `@Scope("singleton")` |

---

## 3. 项目配置

```kotlin
// 1. Application 类 — Hilt 的入口
@HiltAndroidApp
class NewsApplication : Application()
// 必须在 AndroidManifest.xml 中声明

// 2. Module — 提供网络相关依赖
@Module
@InstallIn(SingletonComponent::class)
object NetworkModule {

    @Provides
    @Singleton
    fun provideOkHttpClient(): OkHttpClient {
        return OkHttpClient.Builder()
            .addInterceptor(HttpLoggingInterceptor().apply {
                level = HttpLoggingInterceptor.Level.BODY
            })
            .build()
    }

    @Provides
    @Singleton
    fun provideRetrofit(okHttpClient: OkHttpClient): Retrofit {
        return Retrofit.Builder()
            .baseUrl(BASE_URL)
            .client(okHttpClient)
            .addConverterFactory(GsonConverterFactory.create())
            .build()
    }

    @Provides
    @Singleton
    fun provideNewsApi(retrofit: Retrofit): NewsApi {
        return retrofit.create(NewsApi::class.java)
    }
}

// 3. Repository — 用 @Inject constructor
class NewsRepository @Inject constructor(
    private val newsApi: NewsApi
) {
    suspend fun getNews(): List<NewsItem> = newsApi.getNews()
}

// 4. ViewModel — 用 @HiltViewModel
@HiltViewModel
class NewsViewModel @Inject constructor(
    private val repository: NewsRepository
) : ViewModel() {
    // ...
}
```

---

## 4. Component 层级

Hilt 有一套预定义的 Component 层级，对应 Android 类的生命周期：

```
SingletonComponent     → Application 生命周期
  ├─ ViewModelComponent → ViewModel 生命周期
  ├─ ActivityComponent  → Activity 生命周期
  │   └─ FragmentComponent → Fragment 生命周期
  └─ ServiceComponent   → Service 生命周期
```

```kotlin
// @InstallIn 决定了依赖的范围
@Module
@InstallIn(SingletonComponent::class)  // 全局单例
object AppModule { }

@Module
@InstallIn(ViewModelComponent::class) // 每个 ViewModel 一份
object ViewModelModule { }
```

**Spring 类比**：SingletonComponent ≈ 全局 ApplicationContext, ActivityComponent ≈ Request Scope。

---

## 5. @Provides vs @Binds

```kotlin
// @Provides — 创建第三方库的对象（Retrofit、OkHttp 不是你写的类）
@Module
@InstallIn(SingletonComponent::class)
object AppModule {
    @Provides
    @Singleton
    fun provideRetrofit(): Retrofit {
        return Retrofit.Builder()  // 第三方对象，需要自己 new
            .baseUrl("...")
            .build()
    }
}

// @Binds — 绑定接口到实现（更高效，代码更少）
interface AuthRepository {
    suspend fun login(): Boolean
}

class AuthRepositoryImpl @Inject constructor(
    private val api: AuthApi
) : AuthRepository {
    override suspend fun login() = api.login()
}

@Module
@InstallIn(SingletonComponent::class)
abstract class RepositoryModule {
    @Binds
    @Singleton
    abstract fun bindAuthRepository(impl: AuthRepositoryImpl): AuthRepository
    // Hilt 遇到 AuthRepository 依赖时，注入 AuthRepositoryImpl
}
```

---

## 6. ViewModel 注入

```kotlin
// Hilt 为 ViewModel 提供了专门的 @HiltViewModel 注解
@HiltViewModel
class NewsViewModel @Inject constructor(
    private val repository: NewsRepository,
    private val analytics: AnalyticsService  // 多个依赖也没问题
) : ViewModel()
```

**Spring 类比**：`@HiltViewModel` ≈ `@Component` + `@Scope("session")`。构造函数参数自动从容器中获取。

---

## 7. Compose 中获取 ViewModel

```kotlin
@Composable
fun NewsScreen(
    viewModel: NewsViewModel = hiltViewModel()  // Compose 专用获取方式
) {
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()
    // ...
}

// 如果需要在 Fragment 中获取：
@AndroidEntryPoint
class NewsFragment : Fragment() {
    private val viewModel: NewsViewModel by viewModels()
}
```

---

## 面试速记

| 问题 | 回答 |
|------|------|
| Hilt 是什么？ | Android 的依赖注入框架，基于 Dagger，自动管理依赖的生命周期 |
| @Provides vs @Binds | @Provides 用于第三方类（需构造代码），@Binds 用于接口→实现绑定 |
| Hilt 的 Component 层级？ | SingletonComponent → ActivityComponent → FragmentComponent / ViewModelComponent |
| 为什么要 DI？ | 解耦、可测试（Mock 注入）、生命周期自动管理 |
