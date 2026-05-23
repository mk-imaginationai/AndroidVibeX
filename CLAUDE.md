# Android Project Rules

These rules apply to every task in this project. They encode senior Android engineering standards. Follow them unconditionally unless the user explicitly overrides a specific rule.

## Language
- Always Kotlin. Never Java for new code.
- Use Kotlin idioms: data classes, sealed classes, extension functions, scope functions. Prefer `let` for null-safe chaining, `apply` for object initialization/mutation, `run` for scoped computation, `also` for side effects.
- Prefer `val` over `var`. Immutability by default.
- Never use `!!`. Use `?.let`, `?: return`, or `requireNotNull()` with a message.

## Architecture
- Default: MVVM + Repository pattern.
- ViewModel holds UI state. Never put business logic in Activity/Fragment/Composable.
- Repository is the single source of truth. It abstracts data sources (remote + local).
- Use sealed classes or sealed interfaces for UI state (`Loading`, `Success`, `Error`).
- One primary ViewModel per screen for UI state. Shared/utility ViewModels for cross-screen concerns are acceptable if scoped appropriately.

## Threading
- All async work uses Kotlin Coroutines. Never use `Thread`, `AsyncTask`, `Handler`, or RxJava.
- IO operations on `Dispatchers.IO`. CPU-intensive work on `Dispatchers.Default`. UI updates on `Dispatchers.Main`.
- Never block the main thread — no network, database, or heavy computation on `Dispatchers.Main`.
- Use `viewModelScope` in ViewModels. Use `lifecycleScope` in Activity/Fragment. Never create bare `CoroutineScope` without a lifecycle owner.

## Dependency Injection
- Hilt is the default DI framework. Use `@HiltViewModel`, `@Inject`, `@Module`, `@Provides`.
- Koin is accepted if the project already uses it — do not migrate unless asked.
- Never manually construct dependencies for non-trivial dependency graphs.

## UI
- Jetpack Compose is preferred for all new UI.
- XML Views are accepted in existing projects. Do not mix Compose and XML within the same screen without justification; screen-level interop during migration is acceptable.
- Never call UI methods from a background thread.
- Always use `collectAsStateWithLifecycle()` (not `collectAsState()`) for Flow in Compose.

## Storage
- Room for all structured local data. Never raw SQLite.
- DataStore (Proto or Preferences) for key-value storage. Never `SharedPreferences` for new code.
- Never store sensitive data in plain SharedPreferences or files — use EncryptedSharedPreferences or Keystore.

## Networking
- Retrofit + OkHttp for all HTTP. Never `HttpURLConnection` or Volley.
- Always define a sealed `Result<T>` / `NetworkResult<T>` type. Never expose raw exceptions to the ViewModel.
- Use `suspend` functions in Retrofit interfaces. Never use `Call<T>` unless in Java interop.

## Error Handling
- Never swallow exceptions silently with empty `catch` blocks.
- Log errors with `Timber.e(exception, "message")`. Never use `Log.*` directly.
- Return `Result<T>` or sealed classes from repositories, not raw exceptions.

## Lifecycle
- Never hold a Context reference in a ViewModel or Repository.
- Always use lifecycle-aware observers. Never manually manage lifecycle subscriptions.
- Handle `onStop`/`onDestroy` cleanup — cancel jobs, unregister receivers, release resources.

## Code Quality
- No magic numbers or strings — use constants or string resources.
- No God classes or functions over 40 lines. Extract if growing beyond that.
- All new code must have unit tests for business logic (ViewModel, Repository, UseCases).
