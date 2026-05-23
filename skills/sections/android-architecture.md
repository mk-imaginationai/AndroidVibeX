---
name: android-architecture
description: Use when designing app architecture, choosing between MVVM/MVP/MVC/MVI, implementing Repository pattern, setting up DI with Hilt/Koin/Dagger, or working with Flow, LiveData, or design patterns (Observer, Factory, Builder).
---

# Android Architecture & Design Patterns

## Concept

**MVVM** is the Android-recommended architecture. It separates:
- **Model** — data layer (Repository, Room, Retrofit)
- **View** — UI layer (Activity, Fragment, Composable) — observes state, emits events
- **ViewModel** — state holder — processes events, exposes UI state via Flow/StateFlow

**Repository pattern** sits between ViewModel and data sources. It decides whether to fetch from network or cache.

**Other patterns:**
- **MVI** — unidirectional data flow, single immutable state object, good for complex screens
- **MVP** — older pattern, replaced by MVVM in modern Android
- **MVC** — not idiomatic Android, avoid for new projects

---

## Implementation Patterns

### MVVM — Full example

```kotlin
// UI State
sealed interface HomeUiState {
    data object Loading : HomeUiState
    data class Success(val items: List<Item>) : HomeUiState
    data class Error(val message: String) : HomeUiState
}

// ViewModel
@HiltViewModel
class HomeViewModel @Inject constructor(
    private val repository: ItemRepository
) : ViewModel() {

    private val _uiState = MutableStateFlow<HomeUiState>(HomeUiState.Loading)
    val uiState: StateFlow<HomeUiState> = _uiState.asStateFlow()

    init {
        loadItems()
    }

    private fun loadItems() {
        viewModelScope.launch {
            repository.getItems()
                .onSuccess { items -> _uiState.value = HomeUiState.Success(items) }
                .onFailure { e -> _uiState.value = HomeUiState.Error(e.message ?: "Unknown error") }
        }
    }
}

// Repository
interface ItemRepository {
    suspend fun getItems(): Result<List<Item>>
}

class ItemRepositoryImpl @Inject constructor(
    private val api: ItemApi,
    private val dao: ItemDao
) : ItemRepository {
    override suspend fun getItems(): Result<List<Item>> = runCatching {
        val cached = dao.getAll()
        if (cached.isNotEmpty()) return@runCatching cached.map { it.toDomain() }
        val remote = api.getItems()
        dao.insertAll(remote.map { it.toEntity() })
        remote.map { it.toDomain() }
    }
}
```

### Hilt DI — Module setup

```kotlin
@Module
@InstallIn(SingletonComponent::class)
object NetworkModule {

    @Provides
    @Singleton
    fun provideOkHttpClient(): OkHttpClient =
        OkHttpClient.Builder()
            .addInterceptor(HttpLoggingInterceptor().apply {
                level = HttpLoggingInterceptor.Level.BODY
            })
            .build()

    @Provides
    @Singleton
    fun provideRetrofit(okHttpClient: OkHttpClient): Retrofit =
        Retrofit.Builder()
            .baseUrl(BuildConfig.BASE_URL)
            .client(okHttpClient)
            .addConverterFactory(GsonConverterFactory.create())
            .build()

    @Provides
    @Singleton
    fun provideItemApi(retrofit: Retrofit): ItemApi =
        retrofit.create(ItemApi::class.java)
}

@Module
@InstallIn(SingletonComponent::class)
abstract class RepositoryModule {
    @Binds
    @Singleton
    abstract fun bindItemRepository(impl: ItemRepositoryImpl): ItemRepository
}
```

### Flow — Collect in ViewModel

```kotlin
// Emit continuous updates from Room
val items: StateFlow<List<Item>> = repository.observeItems()
    .stateIn(
        scope = viewModelScope,
        started = SharingStarted.WhileSubscribed(5_000),
        initialValue = emptyList()
    )
```

### LiveData — Only use if project predates Flow adoption

```kotlin
val items: LiveData<List<Item>> = liveData {
    emit(repository.getItems().getOrDefault(emptyList()))
}
```

### Observer Pattern — via StateFlow

```kotlin
// ViewModel exposes state
val uiState: StateFlow<UiState> = _uiState.asStateFlow()

// Composable observes
val state by viewModel.uiState.collectAsStateWithLifecycle()
```

### Builder Pattern

```kotlin
data class NotificationConfig(
    val title: String,
    val body: String,
    val channelId: String,
    val deepLinkUri: Uri? = null
) {
    class Builder {
        private var title: String = ""
        private var body: String = ""
        private var channelId: String = "default"
        private var deepLinkUri: Uri? = null

        fun title(title: String) = apply { this.title = title }
        fun body(body: String) = apply { this.body = body }
        fun channelId(id: String) = apply { this.channelId = id }
        fun deepLink(uri: Uri) = apply { this.deepLinkUri = uri }
        fun build() = NotificationConfig(title, body, channelId, deepLinkUri)
    }
}
```

---

## Guardrails

### DO
- Use `StateFlow` for UI state. Use `SharedFlow` for one-shot events (navigation, snackbar).
- Use `SharingStarted.WhileSubscribed(5_000)` to avoid resource leaks when the app goes to background.
- Use `@HiltViewModel` — never instantiate ViewModels manually with `ViewModelProvider.Factory` unless Hilt cannot be used.
- Use `Result<T>` (`runCatching`) for wrapping Repository results.
- Bind interfaces in Hilt with `@Binds` — use `@Provides` only for third-party classes.

### DON'T
- Don't use `LiveData` for new code — use `StateFlow`.
- Don't call `suspend` functions directly from `init {}` without `viewModelScope.launch`.
- Don't expose `MutableStateFlow` from ViewModel — always expose `StateFlow`.
- Don't put network calls or database access directly in ViewModel — always via Repository.
- Don't use `GlobalScope` — it's not lifecycle-aware and leaks.

---

## References
- [ViewModel](https://developer.android.com/topic/libraries/architecture/viewmodel)
- [LiveData](https://developer.android.com/topic/libraries/architecture/livedata)
- [Kotlin Flow](https://kotlinlang.org/docs/flow.html)
- [Dependency Injection](https://developer.android.com/training/dependency-injection)
- [Hilt](https://developer.android.com/training/dependency-injection/hilt-android)
- [Koin](https://insert-koin.io)
- [Dagger](https://dagger.dev/)
