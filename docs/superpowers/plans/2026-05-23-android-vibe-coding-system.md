# Android Vibe Coding System Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a three-layer skill system (CLAUDE.md + 9 section skills + 4 agent roles) that encodes senior Android engineering knowledge so AI agents can write correct, idiomatic Android code without guidance.

**Architecture:** A CLAUDE.md file sets non-negotiable global rules auto-loaded by Claude Code. Nine section skill files each cover one roadmap domain with concept + implementation patterns + guardrails. Four agent role skills activate specialized personas (architect, implementer, debugger, reviewer). All files are self-contained and reusable across any Android project.

**Tech Stack:** Kotlin, Jetpack Compose, MVVM, Hilt, Coroutines, Room, Retrofit, Firebase — all modern Android stack.

---

## File Map

| File | Type | Purpose |
|---|---|---|
| `CLAUDE.md` | Rules | Global Android rules, auto-loaded in any project |
| `skills/sections/android-app-components.md` | Skill | Activity, Services, Intents, Lifecycle |
| `skills/sections/android-ui-layouts.md` | Skill | Compose, Fragments, ConstraintLayout, Navigation |
| `skills/sections/android-architecture.md` | Skill | MVVM, Repository, DI, Flow, LiveData |
| `skills/sections/android-storage.md` | Skill | Room, DataStore, SharedPrefs, File System |
| `skills/sections/android-networking.md` | Skill | Retrofit, OkHttp, Apollo GraphQL |
| `skills/sections/android-async.md` | Skill | Coroutines, WorkManager, threading rules |
| `skills/sections/android-firebase.md` | Skill | Auth, Crashlytics, FCM, Firestore, Maps |
| `skills/sections/android-code-quality.md` | Skill | Ktlint, Detekt, Timber, LeakCanary, Chucker |
| `skills/sections/android-testing.md` | Skill | JUnit, Espresso, test strategy |
| `skills/agents/android-architect.md` | Agent | App structure, module boundaries, pattern selection |
| `skills/agents/android-implementer.md` | Agent | Feature code, following established patterns |
| `skills/agents/android-debugger.md` | Agent | Crashes, leaks, ANRs, perf issues |
| `skills/agents/android-reviewer.md` | Agent | Code review against Android best practices |

---

## Task 1: CLAUDE.md — Global Android Rules

**Files:**
- Create: `CLAUDE.md`

- [ ] **Step 1: Create CLAUDE.md**

```markdown
# Android Project Rules

These rules apply to every task in this project. They encode senior Android engineering standards. Follow them unconditionally unless the user explicitly overrides a specific rule.

## Language
- Always Kotlin. Never Java for new code.
- Use Kotlin idioms: data classes, sealed classes, extension functions, scope functions (`let`, `apply`, `run`, `also`).
- Prefer `val` over `var`. Immutability by default.
- Never use `!!`. Use `?.let`, `?: return`, or `requireNotNull()` with a message.

## Architecture
- Default: MVVM + Repository pattern.
- ViewModel holds UI state. Never put business logic in Activity/Fragment/Composable.
- Repository is the single source of truth. It abstracts data sources (remote + local).
- Use sealed classes or sealed interfaces for UI state (`Loading`, `Success`, `Error`).
- One ViewModel per screen. Never share ViewModels across unrelated screens.

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
- XML Views are accepted in existing projects. Do not mix Compose and XML without justification.
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
```

- [ ] **Step 2: Verify**

```bash
grep -c "^##" /Users/kashif/Documents/AndroidVibeX/CLAUDE.md
```
Expected: `9` (9 section headers)

- [ ] **Step 3: Commit**

```bash
cd /Users/kashif/Documents/AndroidVibeX && git init && git add CLAUDE.md && git commit -m "feat: add global Android rules CLAUDE.md"
```

---

## Task 2: App Components Skill

**Files:**
- Create: `skills/sections/android-app-components.md`

- [ ] **Step 1: Create the skill file**

```markdown
---
name: android-app-components
description: Use when working with Activity, Fragment, Services, Content Providers, Broadcast Receivers, Intents, Intent Filters, or Activity Lifecycle and state changes.
---

# Android App Components

## Concept

Android apps are built from four component types. Each has a distinct role and lifecycle managed by the Android OS.

| Component | Purpose | When to use |
|---|---|---|
| **Activity** | Single screen with UI | Every user-facing screen |
| **Service** | Background work without UI | Long-running tasks (music, downloads) |
| **Content Provider** | Shared data between apps | Exposing data to other apps (e.g. contacts, media) |
| **Broadcast Receiver** | React to system/app events | Battery low, network change, custom app events |

**Intent** is the messaging object that connects components — both within an app (explicit) and between apps (implicit).

---

## Implementation Patterns

### Activity — Lifecycle-aware setup

```kotlin
@AndroidEntryPoint
class MainActivity : AppCompatActivity() {

    private val viewModel: MainViewModel by viewModels()

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContent {
            MyAppTheme {
                MainScreen(viewModel = viewModel)
            }
        }
    }
}
```

### Activity Lifecycle — Key callbacks

```
onCreate  → onStart → onResume  [VISIBLE + FOCUSED]
onPause → onStop → onDestroy    [HIDDEN]
onPause → onStop → onCreate     [RECREATED, e.g. rotation]
```

**State changes to handle:**
- `onSaveInstanceState` — save transient UI state (scroll position, user input)
- `onRestoreInstanceState` — restore it
- ViewModel survives rotation automatically — use it for data, not savedInstanceState

```kotlin
override fun onSaveInstanceState(outState: Bundle) {
    super.onSaveInstanceState(outState)
    outState.putString("key", value)
}
```

### Explicit Intent — Navigate to another Activity

```kotlin
val intent = Intent(this, DetailActivity::class.java).apply {
    putExtra("item_id", itemId)
}
startActivity(intent)
```

### Implicit Intent — Open a URL

```kotlin
val intent = Intent(Intent.ACTION_VIEW, Uri.parse("https://example.com"))
if (intent.resolveActivity(packageManager) != null) {
    startActivity(intent)
}
```

### Intent Filter — Declare in AndroidManifest.xml

```xml
<activity android:name=".ShareActivity">
    <intent-filter>
        <action android:name="android.intent.action.SEND" />
        <category android:name="android.intent.category.DEFAULT" />
        <data android:mimeType="text/plain" />
    </intent-filter>
</activity>
```

### Service — Background work with coroutines

```kotlin
class SyncService : Service() {

    private val scope = CoroutineScope(SupervisorJob() + Dispatchers.IO)

    override fun onStartCommand(intent: Intent?, flags: Int, startId: Int): Int {
        scope.launch {
            doSyncWork()
            stopSelf()
        }
        return START_NOT_STICKY
    }

    override fun onDestroy() {
        scope.cancel()
        super.onDestroy()
    }

    override fun onBind(intent: Intent?): IBinder? = null
}
```

### Broadcast Receiver — React to network changes

```kotlin
class NetworkReceiver : BroadcastReceiver() {
    override fun onReceive(context: Context, intent: Intent) {
        if (intent.action == ConnectivityManager.CONNECTIVITY_ACTION) {
            // handle network change
        }
    }
}
```

Register dynamically (preferred over manifest for runtime receivers):
```kotlin
val receiver = NetworkReceiver()
val filter = IntentFilter(ConnectivityManager.CONNECTIVITY_ACTION)
registerReceiver(receiver, filter)
// unregister in onDestroy
unregisterReceiver(receiver)
```

---

## Guardrails

### DO
- Use `ViewModel` to survive configuration changes — never store UI data in Activity fields.
- Always check `intent.resolveActivity(packageManager) != null` before firing implicit intents.
- Use `lifecycleScope` for coroutines launched from Activity/Fragment.
- Declare only the permissions you need in the manifest.
- Prefer `registerReceiver()` at runtime over manifest registration for dynamic events.
- Use `foreground services` with a notification for user-visible long-running work.

### DON'T
- Don't put business logic in `Activity` or `Fragment` — delegate to `ViewModel`.
- Don't start a `Service` for short background tasks — use `WorkManager` or coroutines.
- Don't ignore `onStop`/`onDestroy` — release resources, cancel jobs, unregister receivers.
- Don't use `startActivityForResult` — use the Activity Result API (`registerForActivityResult`).
- Don't pass large objects via Intent extras — use a shared ViewModel or Repository.
- Don't call `finish()` in `onCreate` without showing UI — handle this in the ViewModel state.

---

## References
- [App Fundamentals](https://developer.android.com/guide/components/fundamentals)
- [Activity](https://developer.android.com/reference/android/app/Activity)
- [Activity Lifecycle](https://developer.android.com/guide/components/activities/activity-lifecycle)
- [Services](https://developer.android.com/guide/components/services)
- [Content Providers](https://developer.android.com/guide/topics/providers/content-providers)
- [BroadcastReceiver](https://developer.android.com/reference/android/content/BroadcastReceiver)
- [Intents & Intent Filters](https://developer.android.com/guide/components/intents-filters)
```

- [ ] **Step 2: Verify structure**

```bash
grep "^## " /Users/kashif/Documents/AndroidVibeX/skills/sections/android-app-components.md
```
Expected output:
```
## Concept
## Implementation Patterns
## Guardrails
## References
```

- [ ] **Step 3: Commit**

```bash
git add skills/sections/android-app-components.md && git commit -m "feat: add android-app-components skill"
```

---

## Task 3: UI & Layouts Skill

**Files:**
- Create: `skills/sections/android-ui-layouts.md`

- [ ] **Step 1: Create the skill file**

```markdown
---
name: android-ui-layouts
description: Use when working with Jetpack Compose, Fragments, ConstraintLayout (XML or Compose), TextView, ImageView, App Shortcuts, or Navigation Components.
---

# Android UI & Layouts

## Concept

Android UI is built in two paradigms:
- **Jetpack Compose** — declarative, Kotlin-only, recommended for all new UI
- **XML Views** — imperative, legacy system, still valid for existing codebases

Key components: Composables (Compose) or Views (XML), Fragments (screen containers), Navigation Component (backstack management).

---

## Implementation Patterns

### Jetpack Compose — Basic screen structure

```kotlin
@Composable
fun HomeScreen(
    viewModel: HomeViewModel = hiltViewModel()
) {
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()

    when (val state = uiState) {
        is HomeUiState.Loading -> CircularProgressIndicator()
        is HomeUiState.Success -> HomeContent(items = state.items)
        is HomeUiState.Error -> ErrorMessage(message = state.message)
    }
}

@Composable
private fun HomeContent(items: List<Item>) {
    LazyColumn {
        items(items) { item ->
            ItemRow(item = item)
        }
    }
}
```

### Compose — ConstraintLayout

```kotlin
@Composable
fun ProfileCard() {
    ConstraintLayout(modifier = Modifier.fillMaxWidth()) {
        val (avatar, name, bio) = createRefs()

        Image(
            painter = painterResource(R.drawable.avatar),
            contentDescription = null,
            modifier = Modifier.constrainAs(avatar) {
                top.linkTo(parent.top, margin = 16.dp)
                start.linkTo(parent.start, margin = 16.dp)
            }
        )

        Text(
            text = "Name",
            modifier = Modifier.constrainAs(name) {
                top.linkTo(avatar.top)
                start.linkTo(avatar.end, margin = 8.dp)
            }
        )
    }
}
```

### Fragment — Compose-based Fragment

```kotlin
@AndroidEntryPoint
class HomeFragment : Fragment() {

    override fun onCreateView(
        inflater: LayoutInflater,
        container: ViewGroup?,
        savedInstanceState: Bundle?
    ): View = ComposeView(requireContext()).apply {
        setViewCompositionStrategy(ViewCompositionStrategy.DisposeOnViewTreeLifecycleDestroyed)
        setContent {
            MyAppTheme {
                HomeScreen()
            }
        }
    }
}
```

### Navigation Component — NavGraph with Compose

```kotlin
@Composable
fun AppNavGraph(navController: NavHostController) {
    NavHost(navController = navController, startDestination = "home") {
        composable("home") {
            HomeScreen(
                onNavigateToDetail = { id -> navController.navigate("detail/$id") }
            )
        }
        composable(
            route = "detail/{id}",
            arguments = listOf(navArgument("id") { type = NavType.StringType })
        ) { backStackEntry ->
            DetailScreen(id = backStackEntry.arguments?.getString("id").orEmpty())
        }
    }
}
```

### App Shortcuts — Static shortcut in shortcuts.xml

```xml
<!-- res/xml/shortcuts.xml -->
<shortcuts xmlns:android="http://schemas.android.com/apk/res/android">
    <shortcut
        android:shortcutId="compose_message"
        android:enabled="true"
        android:icon="@drawable/ic_compose"
        android:shortcutShortLabel="@string/compose_shortcut_short_label"
        android:shortcutLongLabel="@string/compose_shortcut_long_label">
        <intent
            android:action="android.intent.action.VIEW"
            android:targetPackage="com.example.app"
            android:targetClass="com.example.app.ComposeActivity" />
    </shortcut>
</shortcuts>
```

---

## Guardrails

### DO
- Use `collectAsStateWithLifecycle()` (from `lifecycle-runtime-compose`) for Flow in Compose — not `collectAsState()`.
- Keep Composables small and focused — extract if a function exceeds ~50 lines.
- Use `hiltViewModel()` in Composables (not constructor injection).
- Use `LazyColumn`/`LazyRow` for lists, never `Column` with a loop for more than ~10 items.
- Use `rememberSaveable` for state that should survive recomposition AND process death.
- Pass navigation callbacks as lambdas (`onNavigateToDetail: (String) -> Unit`) — never pass NavController into Composables directly.

### DON'T
- Don't put business logic in Composables — only UI rendering and event forwarding.
- Don't use `Fragment` for new Compose-only screens — use Compose navigation directly.
- Don't mix XML and Compose casually — only interop when migrating existing screens.
- Don't hold references to `Context` or `Activity` inside a Composable — use `LocalContext.current` when absolutely needed.
- Don't call `invalidate()`, `notifyDataSetChanged()`, or similar imperative refresh patterns in Compose.

---

## References
- [Jetpack Compose](https://developer.android.com/jetpack/compose)
- [Compose Layout Basics](https://developer.android.com/develop/ui/compose/layouts/basics)
- [ConstraintLayout in Compose](https://developer.android.com/develop/ui/compose/layouts/constraintlayout)
- [Fragments](https://developer.android.com/guide/fragments)
- [Navigation Components](https://developer.android.com/guide/navigation)
- [App Shortcuts](https://developer.android.com/guide/topics/ui/shortcuts)
```

- [ ] **Step 2: Verify**

```bash
grep "^## " /Users/kashif/Documents/AndroidVibeX/skills/sections/android-ui-layouts.md
```
Expected: `## Concept`, `## Implementation Patterns`, `## Guardrails`, `## References`

- [ ] **Step 3: Commit**

```bash
git add skills/sections/android-ui-layouts.md && git commit -m "feat: add android-ui-layouts skill"
```

---

## Task 4: Architecture & Design Patterns Skill

**Files:**
- Create: `skills/sections/android-architecture.md`

- [ ] **Step 1: Create the skill file**

```markdown
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
```

- [ ] **Step 2: Verify**

```bash
grep "^## \|^### " /Users/kashif/Documents/AndroidVibeX/skills/sections/android-architecture.md | head -20
```

- [ ] **Step 3: Commit**

```bash
git add skills/sections/android-architecture.md && git commit -m "feat: add android-architecture skill"
```

---

## Task 5: Storage Skill

**Files:**
- Create: `skills/sections/android-storage.md`

- [ ] **Step 1: Create the skill file**

```markdown
---
name: android-storage
description: Use when working with Room Database, DataStore, SharedPreferences, or the Android File System. Covers local persistence patterns and migration strategies.
---

# Android Storage

## Concept

| Solution | Use for | When to choose |
|---|---|---|
| **Room** | Structured relational data | Any app with a list of entities, offline support |
| **DataStore (Preferences)** | Simple key-value pairs | User settings, feature flags, small state |
| **DataStore (Proto)** | Typed key-value with schema | Complex settings that need type safety |
| **File System** | Blobs — images, audio, documents | Media files, large downloads, exports |
| **SharedPreferences** | Legacy key-value | Existing code only — do not use for new features |

---

## Implementation Patterns

### Room — Full setup

```kotlin
// Entity
@Entity(tableName = "items")
data class ItemEntity(
    @PrimaryKey val id: String,
    val title: String,
    val createdAt: Long
)

// DAO
@Dao
interface ItemDao {
    @Query("SELECT * FROM items ORDER BY createdAt DESC")
    fun observeAll(): Flow<List<ItemEntity>>

    @Query("SELECT * FROM items")
    suspend fun getAll(): List<ItemEntity>

    @Insert(onConflict = OnConflictStrategy.REPLACE)
    suspend fun insertAll(items: List<ItemEntity>)

    @Delete
    suspend fun delete(item: ItemEntity)
}

// Database
@Database(entities = [ItemEntity::class], version = 1, exportSchema = true)
abstract class AppDatabase : RoomDatabase() {
    abstract fun itemDao(): ItemDao
}

// Hilt provision
@Module
@InstallIn(SingletonComponent::class)
object DatabaseModule {
    @Provides
    @Singleton
    fun provideDatabase(@ApplicationContext context: Context): AppDatabase =
        Room.databaseBuilder(context, AppDatabase::class.java, "app.db")
            .fallbackToDestructiveMigration()
            .build()

    @Provides
    fun provideItemDao(db: AppDatabase): ItemDao = db.itemDao()
}
```

### Room — Migration

```kotlin
val MIGRATION_1_2 = object : Migration(1, 2) {
    override fun migrate(database: SupportSQLiteDatabase) {
        database.execSQL("ALTER TABLE items ADD COLUMN description TEXT NOT NULL DEFAULT ''")
    }
}

// In DatabaseModule:
Room.databaseBuilder(context, AppDatabase::class.java, "app.db")
    .addMigrations(MIGRATION_1_2)
    .build()
```

### DataStore — Preferences

```kotlin
// Keys
object PrefsKeys {
    val IS_ONBOARDED = booleanPreferencesKey("is_onboarded")
    val THEME = stringPreferencesKey("theme")
}

// DataStore injection
@Module
@InstallIn(SingletonComponent::class)
object DataStoreModule {
    @Provides
    @Singleton
    fun provideDataStore(@ApplicationContext context: Context): DataStore<Preferences> =
        PreferenceDataStoreFactory.create(
            produceFile = { context.preferencesDataStoreFile("user_prefs") }
        )
}

// Repository usage
class UserPrefsRepository @Inject constructor(
    private val dataStore: DataStore<Preferences>
) {
    val isOnboarded: Flow<Boolean> = dataStore.data
        .map { prefs -> prefs[PrefsKeys.IS_ONBOARDED] ?: false }

    suspend fun setOnboarded(value: Boolean) {
        dataStore.edit { prefs -> prefs[PrefsKeys.IS_ONBOARDED] = value }
    }
}
```

### File System — Save to app-specific internal storage

```kotlin
suspend fun saveFile(context: Context, fileName: String, content: ByteArray) {
    withContext(Dispatchers.IO) {
        val file = File(context.filesDir, fileName)
        file.writeBytes(content)
    }
}

suspend fun readFile(context: Context, fileName: String): ByteArray? {
    return withContext(Dispatchers.IO) {
        val file = File(context.filesDir, fileName)
        if (file.exists()) file.readBytes() else null
    }
}
```

---

## Guardrails

### DO
- Enable `exportSchema = true` in `@Database` and commit schema JSON to version control.
- Use `Flow<T>` from Room DAOs for reactive UI updates.
- Always provide migrations between Room versions — never ship `fallbackToDestructiveMigration()` in production without user warning.
- Use `Dispatchers.IO` for all Room and file operations.
- Use DataStore in a Repository — never access it directly from ViewModel.

### DON'T
- Don't use `SharedPreferences` for new features — use `DataStore`.
- Don't run Room queries on the main thread — Room enforces this but make sure `.allowMainThreadQueries()` is never set.
- Don't store passwords, tokens, or PII in plain DataStore — use `EncryptedSharedPreferences` or Android Keystore.
- Don't store large files in Room — store the file path, keep the blob on disk.
- Don't use external storage for sensitive files — use `context.filesDir` (internal, private).

---

## References
- [Storage overview](https://developer.android.com/guide/topics/data/data-storage)
- [Room Database](https://developer.android.com/training/data-storage/room)
- [DataStore](https://developer.android.com/topic/libraries/architecture/datastore)
- [Shared Preferences](https://developer.android.com/training/data-storage/shared-preferences)
- [File System](https://developer.android.com/training/data-storage/)
```

- [ ] **Step 2: Verify**

```bash
grep "^## " /Users/kashif/Documents/AndroidVibeX/skills/sections/android-storage.md
```

- [ ] **Step 3: Commit**

```bash
git add skills/sections/android-storage.md && git commit -m "feat: add android-storage skill"
```

---

## Task 6: Networking Skill

**Files:**
- Create: `skills/sections/android-networking.md`

- [ ] **Step 1: Create the skill file**

```markdown
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
        val request = chain.request().newBuilder()
            .header("Authorization", "Bearer $token")
            .build()
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
```

- [ ] **Step 2: Verify**

```bash
grep "^## " /Users/kashif/Documents/AndroidVibeX/skills/sections/android-networking.md
```

- [ ] **Step 3: Commit**

```bash
git add skills/sections/android-networking.md && git commit -m "feat: add android-networking skill"
```

---

## Task 7: Async & Concurrency Skill

**Files:**
- Create: `skills/sections/android-async.md`

- [ ] **Step 1: Create the skill file**

```markdown
---
name: android-async
description: Use when working with Kotlin Coroutines, WorkManager, background tasks, or threading. Covers dispatcher selection, coroutine scope lifecycle, and WorkManager constraints.
---

# Android Async & Concurrency

## Concept

| Tool | Use for |
|---|---|
| **Coroutines** | All async work — network, database, computation |
| **WorkManager** | Deferrable, guaranteed background work that must survive process death |
| **Thread** | Never use directly — use coroutines |
| **AsyncTask** | Deprecated — remove if you see it |

The dispatcher determines which thread a coroutine runs on:
- `Dispatchers.Main` — UI updates
- `Dispatchers.IO` — Network, disk, database
- `Dispatchers.Default` — CPU-intensive work (sorting, parsing large datasets)

---

## Implementation Patterns

### Coroutines — ViewModel scope

```kotlin
@HiltViewModel
class UploadViewModel @Inject constructor(
    private val repository: UploadRepository
) : ViewModel() {

    private val _uploadState = MutableStateFlow<UploadState>(UploadState.Idle)
    val uploadState: StateFlow<UploadState> = _uploadState.asStateFlow()

    fun upload(file: File) {
        viewModelScope.launch {
            _uploadState.value = UploadState.Loading
            repository.uploadFile(file)
                .onSuccess { _uploadState.value = UploadState.Success }
                .onFailure { e -> _uploadState.value = UploadState.Error(e.message ?: "Upload failed") }
        }
    }
}
```

### Coroutines — Dispatcher switching in Repository

```kotlin
class FileRepository @Inject constructor(
    @ApplicationContext private val context: Context
) {
    suspend fun saveFile(name: String, data: ByteArray): Result<Unit> =
        withContext(Dispatchers.IO) {
            runCatching {
                File(context.filesDir, name).writeBytes(data)
            }
        }
}
```

### Coroutines — Parallel execution

```kotlin
suspend fun loadDashboard(): DashboardData = coroutineScope {
    val userDeferred = async { userRepository.getCurrentUser() }
    val statsDeferred = async { statsRepository.getStats() }
    DashboardData(
        user = userDeferred.await().getOrThrow(),
        stats = statsDeferred.await().getOrThrow()
    )
}
```

### WorkManager — One-time background task

```kotlin
class SyncWorker(
    context: Context,
    params: WorkerParameters
) : CoroutineWorker(context, params) {

    override suspend fun doWork(): Result {
        return try {
            // injected via HiltWorker
            syncRepository.sync()
            Result.success()
        } catch (e: Exception) {
            Timber.e(e, "Sync failed")
            if (runAttemptCount < 3) Result.retry() else Result.failure()
        }
    }
}

// Schedule it
val syncRequest = OneTimeWorkRequestBuilder<SyncWorker>()
    .setConstraints(
        Constraints.Builder()
            .setRequiredNetworkType(NetworkType.CONNECTED)
            .build()
    )
    .setBackoffCriteria(BackoffPolicy.EXPONENTIAL, 15, TimeUnit.MINUTES)
    .build()

WorkManager.getInstance(context).enqueueUniqueWork(
    "sync",
    ExistingWorkPolicy.KEEP,
    syncRequest
)
```

### WorkManager — Periodic task

```kotlin
val periodicSync = PeriodicWorkRequestBuilder<SyncWorker>(1, TimeUnit.HOURS)
    .setConstraints(
        Constraints.Builder()
            .setRequiredNetworkType(NetworkType.CONNECTED)
            .setRequiresBatteryNotLow(true)
            .build()
    )
    .build()

WorkManager.getInstance(context).enqueueUniquePeriodicWork(
    "periodic_sync",
    ExistingPeriodicWorkPolicy.KEEP,
    periodicSync
)
```

---

## Guardrails

### DO
- Use `viewModelScope` in ViewModels, `lifecycleScope` in Activity/Fragment. Always tie coroutines to a lifecycle.
- Use `withContext(Dispatchers.IO)` at the Repository level — not in the ViewModel.
- Use `SupervisorJob()` in custom scopes so one child failure doesn't cancel siblings.
- Use `CoroutineWorker` (not plain `Worker`) for WorkManager tasks so you get suspend functions.
- Inject workers with `@HiltWorker` + `AssistedInject`.

### DON'T
- Don't use `GlobalScope` — it's never lifecycle-aware and leaks.
- Don't use `Thread`, `AsyncTask`, `Handler.post`, or `Executors` — use coroutines.
- Don't block inside a coroutine — replace blocking calls with their suspend equivalents.
- Don't create `CoroutineScope(Job())` inside a `ViewModel` or `Fragment` manually — use the provided scopes.
- Don't use WorkManager for immediate short tasks — use coroutines in `viewModelScope`.

---

## References
- [Coroutines overview](https://kotlinlang.org/docs/coroutines-overview.html)
- [WorkManager](https://developer.android.com/topic/libraries/architecture/workmanager)
- [Background tasks guide](https://developer.android.com/guide/background)
- [Processes and threads](https://developer.android.com/guide/components/processes-and-threads)
```

- [ ] **Step 2: Verify**

```bash
grep "^## " /Users/kashif/Documents/AndroidVibeX/skills/sections/android-async.md
```

- [ ] **Step 3: Commit**

```bash
git add skills/sections/android-async.md && git commit -m "feat: add android-async skill"
```

---

## Task 8: Firebase & Google Services Skill

**Files:**
- Create: `skills/sections/android-firebase.md`

- [ ] **Step 1: Create the skill file**

```markdown
---
name: android-firebase
description: Use when integrating Firebase Authentication, Crashlytics, Remote Config, Cloud Messaging (FCM), Firestore, AdMob, Google Play Services, or Google Maps.
---

# Firebase & Google Services

## Concept

| Service | Purpose |
|---|---|
| **Firebase Auth** | User authentication — email, Google, phone, anonymous |
| **Crashlytics** | Crash reporting and non-fatal error tracking |
| **Remote Config** | Server-driven feature flags and config values |
| **FCM** | Push notifications |
| **Firestore** | NoSQL cloud database with real-time sync |
| **AdMob** | In-app advertising |
| **Google Maps** | Map rendering, geocoding, directions |

Always add `google-services.json` to `app/` (never commit secrets in it to public repos).

---

## Implementation Patterns

### Firebase Auth — Sign in with Google

```kotlin
class AuthRepository @Inject constructor(
    private val firebaseAuth: FirebaseAuth,
    private val googleSignInClient: GoogleSignInClient
) {
    val currentUser: FirebaseUser? get() = firebaseAuth.currentUser

    suspend fun signInWithGoogle(idToken: String): Result<FirebaseUser> = runCatching {
        val credential = GoogleAuthProvider.getCredential(idToken, null)
        val result = firebaseAuth.signInWithCredential(credential).await()
        result.user ?: error("Auth succeeded but user is null")
    }

    suspend fun signOut() {
        firebaseAuth.signOut()
        googleSignInClient.signOut().await()
    }
}
```

### Crashlytics — Log non-fatals and custom keys

```kotlin
// Initialize automatically via Firebase SDK (no explicit init needed)

// Log non-fatal exceptions
fun logError(tag: String, e: Throwable) {
    Timber.e(e, tag)
    FirebaseCrashlytics.getInstance().recordException(e)
}

// Set user context
fun setUser(userId: String) {
    FirebaseCrashlytics.getInstance().setUserId(userId)
}

// Custom keys for debugging context
fun setScreenContext(screenName: String) {
    FirebaseCrashlytics.getInstance().setCustomKey("screen", screenName)
}
```

### Remote Config — Fetch and activate

```kotlin
class RemoteConfigRepository @Inject constructor(
    private val remoteConfig: FirebaseRemoteConfig
) {
    suspend fun fetchAndActivate(): Boolean = remoteConfig.fetchAndActivate().await()

    fun isFeatureEnabled(key: String): Boolean = remoteConfig.getBoolean(key)

    fun getString(key: String): String = remoteConfig.getString(key)
}

// In Hilt module:
@Provides
@Singleton
fun provideRemoteConfig(): FirebaseRemoteConfig =
    FirebaseRemoteConfig.getInstance().apply {
        setDefaultsAsync(R.xml.remote_config_defaults)
        setConfigSettingsAsync(remoteConfigSettings {
            minimumFetchIntervalInSeconds = if (BuildConfig.DEBUG) 0 else 3600
        })
    }
```

### FCM — Handle incoming messages

```kotlin
class MyFirebaseMessagingService : FirebaseMessagingService() {

    override fun onNewToken(token: String) {
        // Send token to your backend
        CoroutineScope(SupervisorJob() + Dispatchers.IO).launch {
            tokenRepository.updateToken(token)
        }
    }

    override fun onMessageReceived(message: RemoteMessage) {
        message.notification?.let { notification ->
            showNotification(
                title = notification.title ?: return,
                body = notification.body ?: return
            )
        }
    }

    private fun showNotification(title: String, body: String) {
        val notificationManager = getSystemService(NotificationManager::class.java)
        val notification = NotificationCompat.Builder(this, CHANNEL_ID)
            .setContentTitle(title)
            .setContentText(body)
            .setSmallIcon(R.drawable.ic_notification)
            .setAutoCancel(true)
            .build()
        notificationManager.notify(System.currentTimeMillis().toInt(), notification)
    }
}
```

### Firestore — CRUD with coroutines

```kotlin
class FirestoreItemRepository @Inject constructor(
    private val firestore: FirebaseFirestore
) {
    private val collection = firestore.collection("items")

    suspend fun getItem(id: String): Result<Item> = runCatching {
        val snapshot = collection.document(id).get().await()
        snapshot.toObject(ItemDto::class.java)?.toDomain()
            ?: error("Item not found: $id")
    }

    fun observeItems(): Flow<List<Item>> = callbackFlow {
        val listener = collection.addSnapshotListener { snapshot, error ->
            if (error != null) { close(error); return@addSnapshotListener }
            val items = snapshot?.documents
                ?.mapNotNull { it.toObject(ItemDto::class.java)?.toDomain() }
                ?: emptyList()
            trySend(items)
        }
        awaitClose { listener.remove() }
    }

    suspend fun saveItem(item: Item): Result<Unit> = runCatching {
        collection.document(item.id).set(item.toDto()).await()
    }
}
```

---

## Guardrails

### DO
- Initialize Firebase in `Application.onCreate()` — `FirebaseApp.initializeApp(this)` (auto if using google-services plugin).
- Use Hilt to provide `FirebaseAuth`, `FirebaseFirestore`, `FirebaseRemoteConfig` — never access `.getInstance()` outside a module.
- Always await Firebase Tasks with `.await()` (from `kotlinx-coroutines-play-services`).
- Set Remote Config defaults in XML (`res/xml/remote_config_defaults.xml`) — never hardcode.
- Send FCM token to your backend on every `onNewToken` call — tokens rotate.

### DON'T
- Don't commit `google-services.json` with production API keys to public repos.
- Don't call Firebase APIs on the main thread without `await()`.
- Don't use Firestore as a relational database — design for document-oriented access patterns.
- Don't ignore Crashlytics in release builds — it's your primary crash visibility tool.
- Don't use `remoteConfig.fetch()` + `remoteConfig.activateFetched()` separately — use `fetchAndActivate()`.

---

## References
- [Firebase Auth](https://firebase.google.com/docs/auth/android/start)
- [Crashlytics](https://firebase.google.com/docs/crashlytics/get-started?platform=android)
- [Remote Config](https://firebase.google.com/docs/remote-config/get-started?platform=android)
- [FCM](https://firebase.google.com/docs/cloud-messaging/android/client)
- [Firestore](https://firebase.google.com/docs/firestore)
- [Google Maps SDK](https://developers.google.com/maps/documentation/android-sdk/overview)
- [Google Play Services](https://developer.android.com/google/play-services)
```

- [ ] **Step 2: Verify**

```bash
grep "^## \|^### " /Users/kashif/Documents/AndroidVibeX/skills/sections/android-firebase.md | head -20
```

- [ ] **Step 3: Commit**

```bash
git add skills/sections/android-firebase.md && git commit -m "feat: add android-firebase skill"
```

---

## Task 9: Code Quality & Debugging Skill

**Files:**
- Create: `skills/sections/android-code-quality.md`

- [ ] **Step 1: Create the skill file**

```markdown
---
name: android-code-quality
description: Use when setting up or using Ktlint, Detekt, Timber, LeakCanary, Chucker, or Jetpack Benchmark. Covers logging, static analysis, memory leak detection, and network inspection.
---

# Android Code Quality & Debugging

## Concept

| Tool | Purpose |
|---|---|
| **Ktlint** | Kotlin code style enforcement (formatting) |
| **Detekt** | Static analysis — code smells, complexity, bugs |
| **Timber** | Structured logging with automatic tag |
| **LeakCanary** | Memory leak detection in debug builds |
| **Chucker** | In-app HTTP traffic inspector |
| **Jetpack Benchmark** | Measure code performance with reliable results |

---

## Implementation Patterns

### Timber — Setup and usage

```kotlin
// In Application.onCreate()
class MyApp : Application() {
    override fun onCreate() {
        super.onCreate()
        if (BuildConfig.DEBUG) {
            Timber.plant(Timber.DebugTree())
        } else {
            Timber.plant(CrashReportingTree()) // custom tree to Crashlytics
        }
    }
}

// Custom Crashlytics tree
class CrashReportingTree : Timber.Tree() {
    override fun log(priority: Int, tag: String?, message: String, t: Throwable?) {
        if (priority >= Log.WARN) {
            FirebaseCrashlytics.getInstance().log("$tag: $message")
            t?.let { FirebaseCrashlytics.getInstance().recordException(it) }
        }
    }
}

// Usage — never use Log.* directly
Timber.d("User logged in: %s", userId)
Timber.e(exception, "Failed to load items")
Timber.w("Unexpected state: %s", state)
```

### Ktlint — Gradle setup

```kotlin
// build.gradle.kts (root)
plugins {
    id("org.jlleitschuh.gradle.ktlint") version "12.1.0"
}

ktlint {
    version.set("1.2.1")
    android.set(true)
    outputToConsole.set(true)
    filter {
        exclude("**/generated/**")
        include("**/kotlin/**")
    }
}

// Run: ./gradlew ktlintCheck
// Auto-fix: ./gradlew ktlintFormat
```

### Detekt — Gradle setup

```kotlin
// build.gradle.kts (root)
plugins {
    id("io.gitlab.arturbosch.detekt") version "1.23.6"
}

detekt {
    buildUponDefaultConfig = true
    config.setFrom("$rootDir/config/detekt.yml")
}

// config/detekt.yml (key rules)
// complexity:
//   LongMethod:
//     threshold: 40
//   LongParameterList:
//     threshold: 6
// style:
//   MagicNumber:
//     active: true
```

### LeakCanary — Zero-config setup

```kotlin
// build.gradle.kts (app module)
dependencies {
    debugImplementation("com.squareup.leakcanary:leakcanary-android:2.14")
}
// No other setup needed — LeakCanary auto-installs via ContentProvider
```

### Chucker — HTTP inspector

```kotlin
// build.gradle.kts (app module)
dependencies {
    debugImplementation("com.github.chuckerteam.chucker:library:4.0.0")
    releaseImplementation("com.github.chuckerteam.chucker:library-no-op:4.0.0")
}

// Add to OkHttpClient:
OkHttpClient.Builder()
    .addInterceptor(
        ChuckerInterceptor.Builder(context)
            .collector(ChuckerCollector(context))
            .maxContentLength(250_000L)
            .redactHeaders("Authorization", "Cookie")
            .alwaysReadResponseBody(true)
            .build()
    )
    .build()
```

### Jetpack Benchmark — Micro benchmark

```kotlin
// In benchmarkTest source set
@RunWith(AndroidJUnit4::class)
class SortingBenchmark {

    @get:Rule
    val benchmarkRule = BenchmarkRule()

    @Test
    fun sortItems() {
        val items = List(1000) { Item(id = it.toString(), title = "Item $it") }
        benchmarkRule.measureRepeated {
            items.sortedBy { it.title }
        }
    }
}
// Run: ./gradlew :benchmark:connectedCheck
```

---

## Guardrails

### DO
- Plant Timber in `Application.onCreate()` — only `DebugTree` in debug, `CrashReportingTree` in release.
- Use `releaseImplementation("...library-no-op")` for Chucker — never ship the full library in release.
- Run `ktlintCheck` and `detekt` in CI — block merges on failures.
- Enable Detekt's `buildUponDefaultConfig = true` so you get all default rules.
- Use LeakCanary only in debug — it's already debug-only by default, don't add to release dependencies.

### DON'T
- Never use `Log.d`, `Log.e`, etc. directly — always use `Timber`.
- Don't disable Detekt rules globally — suppress specific findings with `@Suppress("RuleName")` and a comment explaining why.
- Don't check Chucker interceptor into release builds — use the no-op artifact.
- Don't ignore LeakCanary alerts in debug builds — fix leaks before they accumulate.
- Don't run benchmarks on emulators — always on a physical device with a `release` build variant.

---

## References
- [Linting](https://developer.android.com/studio/write/lint)
- [Ktlint](https://ktlint.github.io/)
- [Detekt](https://detekt.dev/)
- [Timber](https://github.com/JakeWharton/timber)
- [LeakCanary](https://square.github.io/leakcanary/)
- [Chucker](https://github.com/ChuckerTeam/chucker)
- [Jetpack Benchmark](https://developer.android.com/studio/profile/benchmark)
```

- [ ] **Step 2: Verify**

```bash
grep "^## " /Users/kashif/Documents/AndroidVibeX/skills/sections/android-code-quality.md
```

- [ ] **Step 3: Commit**

```bash
git add skills/sections/android-code-quality.md && git commit -m "feat: add android-code-quality skill"
```

---

## Task 10: Testing Skill

**Files:**
- Create: `skills/sections/android-testing.md`

- [ ] **Step 1: Create the skill file**

```markdown
---
name: android-testing
description: Use when writing JUnit unit tests, Espresso UI tests, or setting up a testing strategy. Covers ViewModel testing with TestCoroutineDispatcher, Repository testing with fakes, and UI testing with Espresso/Compose.
---

# Android Testing

## Concept

| Test type | Tool | What to test | Speed |
|---|---|---|---|
| **Unit** | JUnit 5 + MockK | ViewModel, Repository, UseCases | Fast (JVM) |
| **Integration** | JUnit + Room in-memory | Repository + DAO | Medium |
| **UI** | Espresso / Compose Testing | User flows end-to-end | Slow (device) |

Test structure: `Given → When → Then` (or `Arrange → Act → Assert`).

---

## Implementation Patterns

### ViewModel unit test with fake repository

```kotlin
@OptIn(ExperimentalCoroutinesApi::class)
class HomeViewModelTest {

    @get:Rule
    val mainDispatcherRule = MainDispatcherRule()

    private val fakeRepository = FakeItemRepository()
    private lateinit var viewModel: HomeViewModel

    @BeforeEach
    fun setup() {
        viewModel = HomeViewModel(fakeRepository)
    }

    @Test
    fun `uiState is Success when repository returns items`() = runTest {
        val items = listOf(Item(id = "1", title = "Test"))
        fakeRepository.setItems(items)

        viewModel.loadItems()

        val state = viewModel.uiState.value
        assertIs<HomeUiState.Success>(state)
        assertEquals(items, state.items)
    }

    @Test
    fun `uiState is Error when repository throws`() = runTest {
        fakeRepository.setShouldThrow(true)

        viewModel.loadItems()

        assertIs<HomeUiState.Error>(viewModel.uiState.value)
    }
}

// MainDispatcherRule — replaces Main dispatcher with test dispatcher
class MainDispatcherRule(
    val dispatcher: TestCoroutineDispatcher = TestCoroutineDispatcher()
) : TestWatcher() {
    override fun starting(description: Description) = Dispatchers.setMain(dispatcher)
    override fun finished(description: Description) = Dispatchers.resetMain()
}
```

### Fake Repository

```kotlin
class FakeItemRepository : ItemRepository {
    private var items: List<Item> = emptyList()
    private var shouldThrow = false

    fun setItems(items: List<Item>) { this.items = items }
    fun setShouldThrow(value: Boolean) { shouldThrow = value }

    override suspend fun getItems(): Result<List<Item>> = runCatching {
        if (shouldThrow) error("Forced failure")
        items
    }
}
```

### Room integration test (in-memory)

```kotlin
@RunWith(AndroidJUnit4::class)
class ItemDaoTest {

    private lateinit var db: AppDatabase
    private lateinit var dao: ItemDao

    @Before
    fun setup() {
        db = Room.inMemoryDatabaseBuilder(
            ApplicationProvider.getApplicationContext(),
            AppDatabase::class.java
        ).build()
        dao = db.itemDao()
    }

    @After
    fun teardown() { db.close() }

    @Test
    fun insertAndRetrieve() = runTest {
        val entity = ItemEntity(id = "1", title = "Test", createdAt = 0L)
        dao.insertAll(listOf(entity))

        val result = dao.getAll()
        assertEquals(1, result.size)
        assertEquals("Test", result.first().title)
    }
}
```

### Espresso UI test

```kotlin
@RunWith(AndroidJUnit4::class)
@HiltAndroidTest
class HomeScreenTest {

    @get:Rule(order = 0)
    val hiltRule = HiltAndroidRule(this)

    @get:Rule(order = 1)
    val activityRule = ActivityScenarioRule(MainActivity::class.java)

    @Before
    fun setup() { hiltRule.inject() }

    @Test
    fun itemsDisplayedOnSuccess() {
        onView(withId(R.id.recycler_view))
            .check(matches(isDisplayed()))

        onView(withText("Test Item"))
            .check(matches(isDisplayed()))
    }

    @Test
    fun clickingItemNavigatesToDetail() {
        onView(withText("Test Item")).perform(click())

        onView(withId(R.id.detail_title))
            .check(matches(isDisplayed()))
    }
}
```

### Compose UI test

```kotlin
@RunWith(AndroidJUnit4::class)
class HomeScreenComposeTest {

    @get:Rule
    val composeTestRule = createComposeRule()

    @Test
    fun showsLoadingInitially() {
        composeTestRule.setContent {
            HomeScreen(viewModel = FakeHomeViewModel(HomeUiState.Loading))
        }
        composeTestRule.onNodeWithTag("loading_indicator").assertIsDisplayed()
    }

    @Test
    fun showsItemsOnSuccess() {
        val items = listOf(Item(id = "1", title = "Test Item"))
        composeTestRule.setContent {
            HomeScreen(viewModel = FakeHomeViewModel(HomeUiState.Success(items)))
        }
        composeTestRule.onNodeWithText("Test Item").assertIsDisplayed()
    }
}
```

---

## Guardrails

### DO
- Use fake implementations (not mocks) for Repository in ViewModel tests — fakes are more reliable.
- Use `runTest` from `kotlinx-coroutines-test` for all coroutine-based tests.
- Use `MainDispatcherRule` in every ViewModel test that touches `Dispatchers.Main`.
- Test ViewModels in isolation — inject fakes, not real implementations.
- Use `Room.inMemoryDatabaseBuilder` for DAO tests — never a real on-disk database.

### DON'T
- Don't write tests that rely on specific timing — use `runTest` and `advanceUntilIdle()`.
- Don't use `Thread.sleep()` in tests — use `runTest` coroutine control.
- Don't test implementation details (private methods, internal state) — test observable behavior.
- Don't skip the `@After` teardown — always close in-memory databases.
- Don't share ViewModel instances between tests — recreate in `@BeforeEach`.

---

## References
- [Testing overview](https://developer.android.com/training/testing)
- [JUnit local tests](https://developer.android.com/training/testing/local-tests)
- [Espresso](https://developer.android.com/training/testing/espresso)
- [Compose testing](https://developer.android.com/develop/ui/compose/testing)
```

- [ ] **Step 2: Verify**

```bash
grep "^## " /Users/kashif/Documents/AndroidVibeX/skills/sections/android-testing.md
```

- [ ] **Step 3: Commit**

```bash
git add skills/sections/android-testing.md && git commit -m "feat: add android-testing skill"
```

---

## Task 11: Android Architect Agent

**Files:**
- Create: `skills/agents/android-architect.md`

- [ ] **Step 1: Create the agent skill**

```markdown
---
name: android-architect
description: Activate when designing app structure, choosing architecture patterns, defining module boundaries, or planning a new feature before implementation begins. The architect decides HOW before anyone codes WHAT.
---

# Android Architect Agent

You are a senior Android architect. When this skill is active, your job is to design before anyone writes a line of code.

## Responsibilities

- Analyze requirements and define the feature/app structure
- Choose the right architecture pattern (MVVM by default; justify any deviation)
- Define module boundaries and dependency direction
- Identify which section skills are needed (reference them by name)
- Produce a clear implementation blueprint for the `android-implementer` to execute

## Decision Framework

### Architecture selection
- **MVVM** — default for all screens with UI state
- **MVI** — use when the screen has complex, multi-step state transitions (wizards, onboarding flows)
- **Repository pattern** — always, between ViewModel and data sources
- **Single Activity** — always, with Navigation Component managing screens

### Module strategy
For small/medium apps: single `:app` module is fine.
For large apps, split by feature:
```
:app
:feature:home
:feature:profile
:core:network
:core:database
:core:ui
```

### When to create a UseCase
Create a UseCase when:
- Multiple ViewModels need the same business logic
- A ViewModel function exceeds ~15 lines of non-UI logic
- Business rules need to be tested independently

Skip UseCases for simple pass-through operations (ViewModel → Repository with no logic).

## Output format

When acting as architect, always produce:

1. **Component map** — list every class/file that will be created
2. **Dependency graph** — what depends on what (text or ASCII diagram)
3. **Data flow** — how data moves from source to UI (arrows)
4. **Section skills needed** — list which android-* skills the implementer should load
5. **Open questions** — anything that needs a decision before coding starts

## Example output

```
## Architecture Decision: Home Feed Feature

**Pattern:** MVVM + Repository

**Components:**
- `HomeViewModel` — holds `HomeUiState`, calls `GetItemsUseCase`
- `GetItemsUseCase` — orchestrates cache-first fetch
- `ItemRepository` (interface) + `ItemRepositoryImpl`
- `ItemApi` — Retrofit interface
- `ItemDao` — Room DAO
- `HomeScreen` — Composable, observes ViewModel state

**Data flow:**
HomeScreen → HomeViewModel.loadItems() → GetItemsUseCase → ItemRepository
ItemRepository → ItemDao (cache hit) OR ItemApi (miss) → map to domain → emit Result<List<Item>>
HomeViewModel → uiState: StateFlow<HomeUiState>
HomeScreen ← collectAsStateWithLifecycle()

**Section skills needed:** android-architecture, android-networking, android-storage, android-async

**Open questions:**
- Pagination required? (affects DAO query + ViewModel state shape)
- Should items be editable offline? (affects conflict resolution strategy)
```

## Guardrails

- Never design for hypothetical future requirements — YAGNI.
- Never let the View layer own business logic.
- Always establish data flow direction before anyone writes code.
- If a feature needs >3 section skills, consider whether it should be split into sub-features.
```

- [ ] **Step 2: Verify**

```bash
grep "^## \|^### " /Users/kashif/Documents/AndroidVibeX/skills/agents/android-architect.md | head -20
```

- [ ] **Step 3: Commit**

```bash
git add skills/agents/android-architect.md && git commit -m "feat: add android-architect agent skill"
```

---

## Task 12: Android Implementer Agent

**Files:**
- Create: `skills/agents/android-implementer.md`

- [ ] **Step 1: Create the agent skill**

```markdown
---
name: android-implementer
description: Activate when writing feature code for an Android app. The implementer executes the architect's blueprint, follows established patterns, and writes production-quality Kotlin code with tests.
---

# Android Implementer Agent

You are a senior Android developer executing an implementation plan. When this skill is active, you write code — not plans, not designs, not discussions. You execute.

## Responsibilities

- Implement features following the architect's blueprint
- Write clean, idiomatic Kotlin following the project's CLAUDE.md rules
- Write tests alongside implementation (unit tests at minimum)
- Follow the section skill patterns exactly — do not invent new patterns

## Before Writing Code

Load the relevant section skills for the task:
- Working with screens? → `android-ui-layouts`
- Making network calls? → `android-networking`
- Persisting data? → `android-storage`
- Background work? → `android-async`
- Wiring DI? → `android-architecture`

## Implementation Order (always follow this sequence)

1. **Domain model** — data classes, sealed interfaces (no Android dependencies)
2. **Data layer** — DAO, API interface, DTO, mappers
3. **Repository** — interface + implementation
4. **ViewModel** — UI state, events, business logic orchestration
5. **UI** — Composable screen, wired to ViewModel
6. **DI module** — bind everything together
7. **Tests** — unit tests for ViewModel + Repository, UI test for the happy path

## Code Standards

- File per class. One class = one responsibility.
- Function max ~40 lines. Extract if longer.
- No magic numbers — use named constants.
- Sealed class for every UI state that has >1 variant.
- `Result<T>` wrapping at every Repository boundary.
- `Timber.d/e/w` for all logging — never `Log.*`.

## When you hit ambiguity

- Missing architecture decision → stop and invoke `android-architect`
- Missing implementation pattern → check the relevant section skill
- Missing library knowledge → use `android docs search <keyword>` via the android CLI

## Guardrails

- Never skip writing the unit test for a ViewModel or Repository.
- Never add `@SuppressWarnings` or `@Suppress` without a comment explaining why.
- Never implement more than what the task requires — YAGNI.
- Always run the build and tests before marking a task complete.
- If a class is growing beyond its responsibility, split it — don't add more methods.
```

- [ ] **Step 2: Verify**

```bash
grep "^## \|^### " /Users/kashif/Documents/AndroidVibeX/skills/agents/android-implementer.md
```

- [ ] **Step 3: Commit**

```bash
git add skills/agents/android-implementer.md && git commit -m "feat: add android-implementer agent skill"
```

---

## Task 13: Android Debugger Agent

**Files:**
- Create: `skills/agents/android-debugger.md`

- [ ] **Step 1: Create the agent skill**

```markdown
---
name: android-debugger
description: Activate when investigating a crash, memory leak, ANR, performance issue, or unexpected behavior in an Android app. The debugger diagnoses before fixing.
---

# Android Debugger Agent

You are a senior Android engineer debugging a production or development issue. When this skill is active, you diagnose first — never guess and patch.

## Responsibilities

- Identify root cause of crashes, leaks, ANRs, janks, and logic bugs
- Use available tooling systematically before concluding
- Propose a precise fix, not a workaround
- Prevent regression by identifying what test would have caught this

## Diagnostic Playbook

### Crash (NullPointerException / IllegalStateException)

1. Read the full stack trace — identify the exact line, not just the exception type.
2. Check: is this a lifecycle issue? (accessing a destroyed Fragment, dereferenced ViewModel)
3. Check: is this a null that should have been handled? (add `?.let` or `requireNotNull`)
4. Check: is this a threading issue? (UI operation on background thread)
5. Fix the root cause. Add a test that would reproduce the crash.

### Memory Leak (LeakCanary alert)

1. Read the leak trace — identify the leaking object and the GC root holding it.
2. Common causes:
   - Context/Activity held in a static field or long-lived object
   - Anonymous inner class capturing `this`
   - BroadcastReceiver not unregistered in `onDestroy`
   - Coroutine scope not cancelled
3. Fix: use `WeakReference`, lifecycle-aware components, or proper cleanup.

### ANR (Application Not Responding)

1. Check: is there IO/network on the main thread? → Move to `Dispatchers.IO`.
2. Check: is there a long computation on the main thread? → Move to `Dispatchers.Default`.
3. Check: is there a deadlock? (two coroutines blocking each other)
4. Use `android layout` CLI command to inspect if the UI is blocked.

### Jank / Dropped frames

1. Check: is `LazyColumn` rendering items with expensive operations in the item composable?
2. Check: is `collectAsState()` used instead of `collectAsStateWithLifecycle()`? (causes extra recompositions)
3. Check: are large objects being created inside a Composable without `remember`?
4. Use Perfetto trace skill if available to profile the exact frame.

### Logic Bug (unexpected behavior, wrong data)

1. Add `Timber.d` logging at the data flow boundaries (API response, DAO result, ViewModel state).
2. Check the data mapping — `toDomain()` and `toEntity()` are frequent bug sites.
3. Check coroutine cancellation — was a scope cancelled before a result was emitted?
4. Write a unit test that reproduces the wrong behavior before fixing it.

## Tools to use

| Symptom | Tool |
|---|---|
| Crash | Logcat, Crashlytics stack trace |
| Memory leak | LeakCanary, Android Studio Memory Profiler |
| Network issue | Chucker (debug), OkHttp logs |
| Slow UI | `android layout` CLI, Perfetto |
| Wrong data | Timber logs at each layer, unit test |

## Output format

When reporting a diagnosis, always produce:

1. **Root cause** — one sentence
2. **Evidence** — what in the logs/trace/code confirms it
3. **Fix** — exact code change
4. **Regression test** — what test to add

## Guardrails

- Never patch symptoms — find and fix the root cause.
- Never add `try { } catch (e: Exception) { }` to hide a crash — fix what's crashing.
- Always identify what test would have caught the bug before it shipped.
- Never assume the bug is in the framework — it's almost always in our code.
```

- [ ] **Step 2: Verify**

```bash
grep "^## \|^### " /Users/kashif/Documents/AndroidVibeX/skills/agents/android-debugger.md
```

- [ ] **Step 3: Commit**

```bash
git add skills/agents/android-debugger.md && git commit -m "feat: add android-debugger agent skill"
```

---

## Task 14: Android Reviewer Agent

**Files:**
- Create: `skills/agents/android-reviewer.md`

- [ ] **Step 1: Create the agent skill**

```markdown
---
name: android-reviewer
description: Activate when reviewing Android code — a PR, a diff, a file, or a vibe coder's output. The reviewer checks correctness, idioms, architecture compliance, and test coverage.
---

# Android Reviewer Agent

You are a senior Android engineer doing a code review. When this skill is active, you review with precision — not vague comments, but specific findings with line-level detail and concrete fixes.

## Responsibilities

- Verify the code follows CLAUDE.md rules
- Catch architecture violations (logic in View, missing Repository boundary, etc.)
- Catch threading and lifecycle mistakes
- Assess test coverage quality
- Flag security issues (plaintext credentials, missing permission checks, exported components)

## Review Checklist

Run through this checklist for every review:

### Architecture
- [ ] ViewModel has no Android framework imports except lifecycle? (no Context, no Activity)
- [ ] Repository returns `Result<T>` or sealed class — never throws raw exceptions to ViewModel?
- [ ] No business logic in Activity, Fragment, or Composable?
- [ ] No direct database/network calls from ViewModel?

### Threading
- [ ] No IO/network on `Dispatchers.Main`?
- [ ] `viewModelScope` / `lifecycleScope` used — never `GlobalScope`?
- [ ] No `Thread.sleep()` or blocking calls inside coroutines?

### UI
- [ ] `collectAsStateWithLifecycle()` used — not `collectAsState()`?
- [ ] NavController not passed into Composables?
- [ ] No logic beyond rendering and event forwarding in Composables?

### Lifecycle
- [ ] BroadcastReceivers unregistered in `onStop`/`onDestroy`?
- [ ] Coroutine scopes cancelled on destroy?
- [ ] No Context reference in ViewModel or Repository?

### Testing
- [ ] ViewModel has unit tests?
- [ ] Tests use fakes, not mocks?
- [ ] `MainDispatcherRule` present in ViewModel tests?

### Security
- [ ] No hardcoded API keys or secrets in code?
- [ ] Sensitive data not logged (no `Timber.d("token: $token")`)?
- [ ] Exported activities/services require appropriate permissions?

## Output format

For each finding, always write:

```
**[SEVERITY] File:LineNumber — Finding title**
Issue: What is wrong and why it matters.
Fix:
```kotlin
// exact replacement code
```
```

Severity levels:
- **BLOCKER** — will crash, leak, or expose data in production
- **MAJOR** — architecture violation, missing test, threading bug
- **MINOR** — style issue, suboptimal pattern, missing log

## Guardrails

- Every finding must include a concrete fix — never comment-only feedback.
- Prioritize BLOCKERs first. Stop listing MAJORs if there are >5 BLOCKERs.
- Do not request changes that aren't in CLAUDE.md or these review criteria — reviewer taste is not a blocker.
- Approve when: no BLOCKERs, no MAJORs, MINORs are acknowledged.
```

- [ ] **Step 2: Verify**

```bash
grep "^## \|^### " /Users/kashif/Documents/AndroidVibeX/skills/agents/android-reviewer.md
```

- [ ] **Step 3: Commit**

```bash
git add skills/agents/android-reviewer.md && git commit -m "feat: add android-reviewer agent skill"
```

---

## Self-Review Notes

- All 9 section skills follow the same template (Concept / Implementation Patterns / Guardrails / References)
- All 4 agent skills define: responsibilities, decision framework, output format, guardrails
- CLAUDE.md covers all 10 rule categories from the spec
- No TBD or TODO in any file
- Types and method names are consistent across skills (e.g. `ItemRepository.getItems()` returns `Result<List<Item>>` throughout)
- No skill duplicates what `android-cli` covers (tooling, SDK, emulators)
- Every section links back to the original roadmap references
