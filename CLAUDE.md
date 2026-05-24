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

---

## Vibe Coding Rules

Rules for AI-assisted rapid Android development. These guide every code generation task.

### Package Structure

Every feature lives in its own package under the app module:

```
com.example.app/
├── core/
│   ├── data/           # Base classes, Result type, network client
│   ├── ui/             # Theme, typography, shared Composables
│   └── util/           # Extensions, helpers
└── feature/
    └── <featurename>/
        ├── data/
        │   ├── <Feature>Repository.kt        # Implementation
        │   ├── <Feature>RemoteDataSource.kt
        │   ├── <Feature>LocalDataSource.kt
        │   └── model/
        │       ├── <Feature>Dto.kt           # Network model
        │       └── <Feature>Entity.kt        # DB model
        ├── domain/
        │   ├── <Feature>RepositoryInterface.kt
        │   ├── model/
        │   │   └── <Feature>.kt              # Domain model
        │   └── usecase/
        │       └── Get<Feature>UseCase.kt
        └── presentation/
            ├── <Feature>Screen.kt            # Composable root
            ├── <Feature>ViewModel.kt
            └── <Feature>UiState.kt           # Sealed class
```

### Naming Conventions

| Element | Convention | Example |
|---|---|---|
| Screen Composable | `<Name>Screen` | `ProfileScreen` |
| ViewModel | `<Name>ViewModel` | `ProfileViewModel` |
| UI State | `<Name>UiState` | `ProfileUiState` |
| Repository interface | `<Name>Repository` | `ProfileRepository` |
| Repository impl | `<Name>RepositoryImpl` | `ProfileRepositoryImpl` |
| UseCase | `<Verb><Name>UseCase` | `GetProfileUseCase` |
| Room Entity | `<Name>Entity` | `ProfileEntity` |
| Network DTO | `<Name>Dto` | `ProfileDto` |
| Room DAO | `<Name>Dao` | `ProfileDao` |
| DI Module | `<Name>Module` | `ProfileModule` |

### Feature Scaffold Checklist

When generating a new feature, create these files in order:

1. `<Feature>.kt` — domain model (pure Kotlin data class)
2. `<Feature>UiState.kt` — sealed class with `Loading`, `Success(data)`, `Error(message)`
3. `<Feature>Repository.kt` — interface in domain
4. `<Feature>Entity.kt` + `<Feature>Dto.kt` — data layer models with mapper functions
5. `<Feature>RepositoryImpl.kt` — implementation coordinating local + remote
6. `Get<Feature>UseCase.kt` — single-responsibility use case
7. `<Feature>ViewModel.kt` — exposes `StateFlow<UiState>`, calls use case
8. `<Feature>Screen.kt` — Composable that collects state, no logic
9. `<Feature>Module.kt` — Hilt bindings

Never skip steps. Never merge ViewModel and Screen into one file.

### Vibe Coding Iteration Loop

For every AI-generated feature:

1. **Scaffold** — generate all files per checklist above
2. **Build** — run `./gradlew assembleDebug` immediately; fix all errors before continuing
3. **Unit test** — write tests for ViewModel and UseCase before wiring UI
4. **Run** — launch on emulator/device, verify the happy path works visually
5. **Edge cases** — test empty state, error state, loading state
6. **Refactor** — extract if any function exceeds 40 lines

Never move to step N+1 with unresolved errors from step N.

### Prompting AI for Android Code

Structure task descriptions this way for best results:

```
Feature: <feature name>
Screen: <what the user sees>
Data: <where data comes from — API endpoint, Room table, or mock>
Actions: <what the user can do>
States: loading | success(<fields>) | error(<message>)
Dependencies: <libraries already in project>
```

Example:
```
Feature: user profile
Screen: avatar, name, email, edit button
Data: GET /api/profile (id, name, email, avatarUrl)
Actions: tap edit → navigate to EditProfileScreen
States: loading | success(Profile) | error(String)
Dependencies: Hilt, Retrofit, Coil, Compose Navigation
```

### AI Output Verification

After any AI-generated Kotlin file, verify:

- [ ] No `!!` operator used anywhere
- [ ] No `Log.*` — only `Timber.*`
- [ ] No `SharedPreferences` — DataStore only
- [ ] No `Thread` or `Handler` — Coroutines only
- [ ] `viewModelScope` used in ViewModel, not `GlobalScope`
- [ ] UI state is a sealed class, not raw booleans
- [ ] Context not held in ViewModel or Repository
- [ ] Mappers present: Entity↔Domain, Dto↔Entity
- [ ] Hilt module created for new dependencies
