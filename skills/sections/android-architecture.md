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

### Resource<T> — Result type for offline-first UI

`Result<T>` has two states: success or failure. For offline-first UX you need a third: loading with optional stale data, and error with optional stale data. Use `Resource<T>` whenever the Repository may serve cached data alongside a failure.

```kotlin
sealed class Resource<T>(
    val data: T? = null,
    val message: String? = null
) {
    class Success<T>(data: T) : Resource<T>(data)
    class Error<T>(message: String, data: T? = null) : Resource<T>(data, message)
    class Loading<T>(data: T? = null) : Resource<T>(data)
}
```

Use `Result<T>` (`runCatching`) for simple one-off operations with no stale-data concern.
Use `Resource<T>` for repository operations that return a `Flow` and may emit cached data before or alongside errors.

---

### MVVM — Full example

```kotlin
// UI State
sealed interface HomeUiState {
    data object Loading : HomeUiState
    data class Success(val items: List<Item>) : HomeUiState
    data class Error(val message: String) : HomeUiState
}

// ViewModel — calls UseCase, never Repository directly
@HiltViewModel
class HomeViewModel @Inject constructor(
    private val getItems: GetItemsUseCase
) : ViewModel() {

    private val _uiState = MutableStateFlow<HomeUiState>(HomeUiState.Loading)
    val uiState: StateFlow<HomeUiState> = _uiState.asStateFlow()

    init {
        loadItems()
    }

    private fun loadItems() {
        viewModelScope.launch {
            getItems()
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

### Use Cases — Domain layer

Create a Use Case for every operation the ViewModel needs. The ViewModel injects Use Cases — never Repositories directly. This keeps business logic out of the ViewModel and makes it independently testable.

```kotlin
// domain/use_case/GetItemsUseCase.kt
class GetItemsUseCase @Inject constructor(
    private val repository: ItemRepository
) {
    suspend operator fun invoke(): Result<List<Item>> = repository.getItems()
}

// domain/use_case/DeleteItemUseCase.kt
class DeleteItemUseCase @Inject constructor(
    private val repository: ItemRepository
) {
    suspend operator fun invoke(id: String): Result<Unit> = repository.deleteItem(id)
}
```

Use Cases containing real logic (not just pass-through):

```kotlin
class GetFilteredItemsUseCase @Inject constructor(
    private val repository: ItemRepository
) {
    suspend operator fun invoke(query: String): Result<List<Item>> =
        repository.getItems().map { items ->
            items.filter { it.title.contains(query, ignoreCase = true) }
        }
}
```

When to skip a Use Case: never. Even a simple pass-through Use Case costs one file and pays for itself the first time a second ViewModel needs the same operation.

---

### Channel<UiEvent> — One-shot UI events

Use `Channel` for events that must be delivered exactly once: navigation, snackbars, toasts. **Never use `SharedFlow` for this** — `SharedFlow` replays the last emission when a new collector subscribes, so a navigation event fires again after rotation.

```kotlin
sealed class UiEvent {
    data class ShowSnackbar(val message: String) : UiEvent()
    data class Navigate(val route: String) : UiEvent()
    data object NavigateUp : UiEvent()
}

@HiltViewModel
class HomeViewModel @Inject constructor(
    private val getItems: GetItemsUseCase
) : ViewModel() {

    private val _uiEvent = Channel<UiEvent>()
    val uiEvent = _uiEvent.receiveAsFlow()

    fun onItemClick(id: String) {
        viewModelScope.launch {
            _uiEvent.send(UiEvent.Navigate("detail/$id"))
        }
    }

    fun onDeleteSuccess() {
        viewModelScope.launch {
            _uiEvent.send(UiEvent.ShowSnackbar("Item deleted"))
        }
    }
}

// In Composable — collect once with LaunchedEffect:
@Composable
fun HomeScreen(viewModel: HomeViewModel = hiltViewModel()) {
    val snackbarHostState = remember { SnackbarHostState() }
    val navController = LocalNavController.current

    LaunchedEffect(key1 = true) {
        viewModel.uiEvent.collect { event ->
            when (event) {
                is UiEvent.ShowSnackbar -> snackbarHostState.showSnackbar(event.message)
                is UiEvent.Navigate -> navController.navigate(event.route)
                UiEvent.NavigateUp -> navController.navigateUp()
            }
        }
    }
}
```

---

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
            // Use kotlinx.serialization (R8-safe, no reflection):
            .addConverterFactory(Json.asConverterFactory("application/json".toMediaType()))
            // See android-networking skill for full serialization setup and DTO annotations
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

### SavedStateHandle — Survive process death

```kotlin
@HiltViewModel
class FormViewModel @Inject constructor(
    private val savedStateHandle: SavedStateHandle,
    private val repository: FormRepository
) : ViewModel() {

    // Survives both rotation AND process death
    var userInput: String
        get() = savedStateHandle["user_input"] ?: ""
        set(value) { savedStateHandle["user_input"] = value }

    // StateFlow backed by SavedStateHandle — survives process death
    val formData: StateFlow<String> = savedStateHandle.getStateFlow("user_input", "")
}
```

Key distinction:
- `ViewModel` survives **rotation** (configuration change)
- `SavedStateHandle` survives **process death** (low memory, system kill)
- Use `SavedStateHandle` for: navigation arguments, user-entered data, selected items, scroll position

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
- Use `StateFlow` for UI state. Use `Channel<UiEvent>` for one-shot events (navigation, snackbar). Never `SharedFlow` for events — it replays the last emission on resubscription, causing duplicate navigation after rotation.
- Use `SharingStarted.WhileSubscribed(5_000)` to avoid resource leaks when the app goes to background.
- Use `@HiltViewModel` — never instantiate ViewModels manually with `ViewModelProvider.Factory` unless Hilt cannot be used.
- Use `Result<T>` (`runCatching`) for wrapping Repository results.
- Bind interfaces in Hilt with `@Binds` — use `@Provides` only for third-party classes.
- Use `SavedStateHandle` in ViewModels that hold data that must survive process death (navigation args, user-entered form data) — inject it via `@HiltViewModel` constructor.

### DON'T
- Don't use `LiveData` for new code — use `StateFlow`.
- Don't call `suspend` functions directly from `init {}` without `viewModelScope.launch`.
- Don't expose `MutableStateFlow` from ViewModel — always expose `StateFlow`.
- Don't put network calls or database access directly in ViewModel — always via Repository.
- Don't use `GlobalScope` — it's not lifecycle-aware and leaks.
- Don't assume ViewModel state survives process death — it doesn't. ViewModel survives rotation but is destroyed when Android kills the process for memory. Use `SavedStateHandle` for critical state.

---

## References
- [ViewModel](https://developer.android.com/topic/libraries/architecture/viewmodel)
- [LiveData](https://developer.android.com/topic/libraries/architecture/livedata)
- [Kotlin Flow](https://kotlinlang.org/docs/flow.html)
- [Dependency Injection](https://developer.android.com/training/dependency-injection)
- [Hilt](https://developer.android.com/training/dependency-injection/hilt-android)
- [Koin](https://insert-koin.io)
- [Dagger](https://dagger.dev/)
