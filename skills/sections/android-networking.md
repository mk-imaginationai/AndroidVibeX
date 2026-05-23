---
name: android-networking
description: Use when setting up Retrofit, OkHttp, or Apollo Android for GraphQL. Covers API interface definition, response handling, interceptors, and network error management.
---

# Android Networking

## Concept

| Library | Use for |
|---|---|
| **Retrofit** | REST APIs — type-safe HTTP client |
| **OkHttp** | HTTP engine powering Retrofit, interceptors, caching |
| **Apollo Android** | GraphQL APIs |

Always wrap network calls in `Result<T>`. Never let raw exceptions propagate to the ViewModel.

---

## Implementation Patterns

### Retrofit — API interface

```kotlin
interface ItemApi {
    @GET("items")
    suspend fun getItems(): List<ItemDto>

    @GET("items/{id}")
    suspend fun getItem(@Path("id") id: String): ItemDto

    @POST("items")
    suspend fun createItem(@Body request: CreateItemRequest): ItemDto

    @DELETE("items/{id}")
    suspend fun deleteItem(@Path("id") id: String): Response<Unit>
}
```

### OkHttp — Auth interceptor

```kotlin
class AuthInterceptor @Inject constructor(
    private val tokenProvider: TokenProvider
) : Interceptor {
    override fun intercept(chain: Interceptor.Chain): Response {
        val token = tokenProvider.getToken()
        val request = if (token != null) {
            chain.request().newBuilder()
                .header("Authorization", "Bearer $token")
                .build()
        } else {
            chain.request()
        }
        return chain.proceed(request)
    }
}
```

### Full Retrofit + OkHttp module with logging

```kotlin
@Module
@InstallIn(SingletonComponent::class)
object NetworkModule {

    @Provides
    @Singleton
    fun provideLoggingInterceptor(): HttpLoggingInterceptor =
        HttpLoggingInterceptor().apply {
            level = if (BuildConfig.DEBUG)
                HttpLoggingInterceptor.Level.BODY
            else
                HttpLoggingInterceptor.Level.NONE
        }

    @Provides
    @Singleton
    fun provideOkHttpClient(
        loggingInterceptor: HttpLoggingInterceptor,
        authInterceptor: AuthInterceptor
    ): OkHttpClient =
        OkHttpClient.Builder()
            .addInterceptor(authInterceptor)
            .addInterceptor(loggingInterceptor)
            .connectTimeout(30, TimeUnit.SECONDS)
            .readTimeout(30, TimeUnit.SECONDS)
            .build()

    @Provides
    @Singleton
    fun provideRetrofit(client: OkHttpClient): Retrofit =
        Retrofit.Builder()
            .baseUrl(BuildConfig.BASE_URL)
            .client(client)
            // Prefer kotlinx.serialization (reflection-free, R8-safe, idiomatic Kotlin):
            // .addConverterFactory(Json.asConverterFactory("application/json".toMediaType()))
            // Requires: implementation("org.jetbrains.kotlinx:kotlinx-serialization-json:1.7.3")
            //           implementation("com.jakewharton.retrofit:retrofit2-kotlinx-serialization-converter:1.0.0")
            // And annotate DTOs with @Serializable instead of relying on Gson reflection.
            //
            // Gson (simpler setup, but reflection-based — add R8 keep rules for DTO fields):
            .addConverterFactory(GsonConverterFactory.create())
            .build()

    @Provides
    @Singleton
    fun provideItemApi(retrofit: Retrofit): ItemApi =
        retrofit.create(ItemApi::class.java)
}
```

### Repository — Safe network call wrapper

```kotlin
class ItemRepositoryImpl @Inject constructor(private val api: ItemApi) : ItemRepository {

    override suspend fun getItems(): Result<List<Item>> = runCatching {
        api.getItems().map { it.toDomain() }
    }

    override suspend fun deleteItem(id: String): Result<Unit> = runCatching {
        val response = api.deleteItem(id)
        if (!response.isSuccessful) {
            error("Delete failed: ${response.code()} ${response.message()}")
        }
    }
}
```

### Apollo Android — GraphQL query

```kotlin
// In build.gradle.kts:
// apollo { service("service") { packageName.set("com.example.graphql") } }

// Query (auto-generated from .graphql files):
val response = apolloClient.query(GetItemsQuery()).execute()
if (response.hasErrors()) {
    error(response.errors?.first()?.message ?: "GraphQL error")
}
val items = response.data?.items?.map { it.toDomain() } ?: emptyList()
```

### kotlinx.serialization — R8-safe DTO pattern (recommended)

Gson uses reflection and breaks under R8 minification without keep rules. Prefer `kotlinx.serialization`:

```kotlin
// build.gradle.kts
// plugins { kotlin("plugin.serialization") }
// implementation("org.jetbrains.kotlinx:kotlinx-serialization-json:1.7.3")

@Serializable
data class ItemDto(
    @SerialName("id") val id: String,
    @SerialName("title") val title: String,
    @SerialName("created_at") val createdAt: Long
)

// Retrofit converter:
val json = Json { ignoreUnknownKeys = true }
Retrofit.Builder()
    .addConverterFactory(json.asConverterFactory("application/json".toMediaType()))
```

**Why**: No reflection → R8-safe by default. Compile-time serialization errors. Kotlin-idiomatic.
If using Gson, add to `proguard-rules.pro`:
```
-keepclassmembers class com.example.** { @com.google.gson.annotations.SerializedName <fields>; }
```

---

## Guardrails

### DO
- Always use `suspend` functions in Retrofit interfaces — never `Call<T>` for new code.
- Log only in DEBUG builds — use `HttpLoggingInterceptor.Level.NONE` in release.
- Always wrap API calls in `runCatching {}` in the Repository.
- Use `Response<T>` from Retrofit when you need to inspect HTTP status codes explicitly.
- Define a `NetworkResult<T>` sealed class if the project needs differentiated error types (auth error vs network error vs server error).

### DON'T
- Don't call Retrofit from the main thread — `suspend` functions must be called from a coroutine.
- Don't expose `Retrofit`, `OkHttpClient`, or `ApiService` directly outside the Repository.
- Don't hardcode base URLs — use `BuildConfig.BASE_URL` from `buildConfigField`.
- Don't log request bodies in release builds — they may contain auth tokens or PII.
- Don't retry failed requests silently — surface errors to the user or log them.

---

## References
- [Retrofit](https://square.github.io/retrofit/)
- [OkHttp](https://square.github.io/okhttp/)
- [Apollo Android](https://www.apollographql.com/docs/kotlin/)
- [Network connectivity](https://developer.android.com/guide/topics/connectivity)
