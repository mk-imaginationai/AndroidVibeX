---
name: android-performance
description: Use when optimizing app startup, reducing Compose recompositions, profiling with Perfetto or Android Studio, handling images efficiently, or catching performance regressions before they ship.
---

# Android Performance

## Concept

| Area | Primary tool | What it catches |
|---|---|---|
| **Compose recomposition** | Compose compiler metrics | Unstable types causing excess recompositions |
| **App startup** | Baseline Profiles + App Startup | Slow cold/warm start |
| **Frame drops / jank** | Perfetto + `Trace` | Long frames, slow layout, binding lag |
| **Memory** | LeakCanary + Memory Profiler | Leaks, excessive allocations, large bitmaps |
| **Main thread violations** | StrictMode | Disk/network on main thread caught at dev time |
| **Images** | Coil (Compose) / Glide (XML) | Bitmap OOM, missing caching, wrong sampling |

---

## Implementation Patterns

### Compose — Stable and Immutable types

The Compose compiler skips recomposition for a Composable if its inputs haven't changed. It can only do this if it can prove the type is **stable** — meaning it will notify Compose when its value changes (or will never change).

Data classes with only primitive/stable fields are inferred as stable automatically. Classes with mutable fields, interfaces, or external types (e.g. `List<T>`) are inferred as **unstable** and always trigger recomposition.

```kotlin
// UNSTABLE — List<T> is a mutable interface; Compose cannot guarantee it won't change
data class HomeUiState(val items: List<Item>)

// STABLE — use Kotlinx Immutable Collections or annotate explicitly
@Immutable
data class HomeUiState(val items: ImmutableList<Item>)

// Or wrap with kotlinx.collections.immutable:
// implementation("org.jetbrains.kotlinx:kotlinx-collections-immutable:0.3.7")
val stable: ImmutableList<Item> = persistentListOf(item1, item2)
```

Use `@Stable` when a class is not `data class` but you can guarantee its equality semantics:

```kotlin
@Stable
class ThemeController {
    var isDark by mutableStateOf(false)
}
```

### Compose — derivedStateOf

Use `derivedStateOf` when a computed value depends on state but changes less frequently than its inputs. Without it, every state change triggers a recomposition even if the derived result is the same.

```kotlin
// BAD — recomposes on every scroll position change, even when showButton hasn't changed
val showButton = listState.firstVisibleItemIndex > 0

// GOOD — recomposes only when the boolean result flips
val showButton by remember {
    derivedStateOf { listState.firstVisibleItemIndex > 0 }
}
```

### Compose — remember for expensive calculations

```kotlin
// BAD — recalculates on every recomposition
val sortedItems = items.sortedBy { it.title }

// GOOD — recalculates only when items changes
val sortedItems = remember(items) { items.sortedBy { it.title } }
```

### Compose — LazyColumn keys

Without stable keys, `LazyColumn` remeasures all visible items when the list changes. With keys, it animates and recycles correctly.

```kotlin
// BAD — no keys; full remeasure on any list change
LazyColumn {
    items(items) { item -> ItemRow(item) }
}

// GOOD — stable key tied to item identity
LazyColumn {
    items(items, key = { it.id }) { item ->
        ItemRow(
            item = item,
            modifier = Modifier.animateItem()
        )
    }
}
```

### Compose compiler metrics

Generate a report showing which Composables are restartable, skippable, and which types are unstable:

```bash
# build.gradle.kts (app module)
tasks.withType<KotlinCompile>().configureEach {
    compilerOptions {
        freeCompilerArgs.addAll(
            "-P", "plugin:androidx.compose.compiler.plugins.kotlin:reportsDestination=${project.buildDir}/compose_metrics",
            "-P", "plugin:androidx.compose.compiler.plugins.kotlin:metricsDestination=${project.buildDir}/compose_metrics"
        )
    }
}
# Run: ./gradlew assembleRelease
# Then inspect: build/compose_metrics/<module>-composables.txt
```

Look for `unstable` next to your state classes and `restartable skippable` next to Composables. A Composable that is `restartable` but NOT `skippable` will always recompose — fix the unstable input types.

### App startup — Baseline Profiles

Baseline Profiles tell ART to AOT-compile critical code paths at install time, reducing cold start by 30–40%.

```kotlin
// build.gradle.kts
// implementation("androidx.profileinstaller:profileinstaller:1.3.1")
// androidTestImplementation("androidx.benchmark:benchmark-macro-junit4:1.2.4")

// macrobenchmark module:
@RunWith(AndroidJUnit4::class)
class StartupBenchmark {

    @get:Rule
    val rule = MacrobenchmarkRule()

    @Test
    fun startup() = rule.measureRepeated(
        packageName = "com.example.app",
        metrics = listOf(StartupTimingMetric()),
        iterations = 5,
        startupMode = StartupMode.COLD
    ) {
        pressHome()
        startActivityAndWait()
    }
}

// Generate profile:
// ./gradlew :macrobenchmark:connectedCheck -P android.testInstrumentationRunnerArguments.androidx.benchmark.enabledRules=BaselineProfile
```

### App startup — App Startup library

Control initialization order and defer non-critical libraries:

```kotlin
// Instead of initializing in Application.onCreate():
class TimberInitializer : Initializer<Unit> {
    override fun create(context: Context) {
        if (BuildConfig.DEBUG) Timber.plant(Timber.DebugTree())
    }
    override fun dependencies(): List<Class<out Initializer<*>>> = emptyList()
}

// AndroidManifest.xml:
// <provider android:name="androidx.startup.InitializationProvider" ...>
//     <meta-data android:name="com.example.TimberInitializer"
//                android:value="androidx.startup" />
// </provider>

// Lazy initialization (defer until first use):
AppInitializer.getInstance(context)
    .initializeComponent(TimberInitializer::class.java)
```

### StrictMode — catch violations in development

```kotlin
class MyApp : Application() {
    override fun onCreate() {
        super.onCreate()
        if (BuildConfig.DEBUG) {
            StrictMode.setThreadPolicy(
                StrictMode.ThreadPolicy.Builder()
                    .detectDiskReads()
                    .detectDiskWrites()
                    .detectNetwork()
                    .penaltyLog()
                    .penaltyFlashScreen()
                    .build()
            )
            StrictMode.setVmPolicy(
                StrictMode.VmPolicy.Builder()
                    .detectLeakedSqlLiteObjects()
                    .detectLeakedClosableObjects()
                    .detectActivityLeaks()
                    .penaltyLog()
                    .build()
            )
        }
    }
}
```

`penaltyFlashScreen()` flashes the screen red on a violation — impossible to miss during manual testing.

### Perfetto — Custom trace sections

Add custom markers to Perfetto traces so you can see exactly where time is spent in your own code:

```kotlin
fun loadDashboard() {
    android.os.Trace.beginSection("loadDashboard")
    try {
        // ... work
    } finally {
        android.os.Trace.endSection()
    }
}

// Or with the AndroidX tracing library (cleaner API):
// implementation("androidx.tracing:tracing:1.2.0")
fun loadDashboard() {
    trace("loadDashboard") {
        // ... work
    }
}
```

Capture a Perfetto trace: Android Studio → App Inspection → CPU → Record → System Trace. Your custom sections appear as named bars in the timeline.

### Image loading — Coil (Compose)

```kotlin
// build.gradle.kts
// implementation("io.coil-kt.coil3:coil-compose:3.0.4")
// implementation("io.coil-kt.coil3:coil-network-okhttp:3.0.4")

// Basic usage:
AsyncImage(
    model = ImageRequest.Builder(LocalContext.current)
        .data(imageUrl)
        .crossfade(true)
        .build(),
    contentDescription = null,
    contentScale = ContentScale.Crop,
    modifier = Modifier.size(64.dp)
)

// With placeholder and error:
AsyncImage(
    model = imageUrl,
    contentDescription = null,
    placeholder = painterResource(R.drawable.placeholder),
    error = painterResource(R.drawable.error),
    contentScale = ContentScale.Crop
)

// Coil singleton setup in Hilt (optional — provides shared OkHttpClient):
@Module
@InstallIn(SingletonComponent::class)
object CoilModule {
    @Provides
    @Singleton
    fun provideImageLoader(
        @ApplicationContext context: Context,
        okHttpClient: OkHttpClient
    ): ImageLoader = ImageLoader.Builder(context)
        .okHttpClient(okHttpClient)
        .memoryCache {
            MemoryCache.Builder()
                .maxSizePercent(context, 0.25)
                .build()
        }
        .diskCache {
            DiskCache.Builder()
                .directory(context.cacheDir.resolve("image_cache"))
                .maxSizeBytes(50 * 1024 * 1024) // 50MB
                .build()
        }
        .build()
}
```

### Image loading — Glide (XML / Views)

```kotlin
// build.gradle.kts
// implementation("com.github.bumptech.glide:glide:4.16.0")

Glide.with(context)
    .load(imageUrl)
    .placeholder(R.drawable.placeholder)
    .error(R.drawable.error)
    .centerCrop()
    .into(imageView)

// Thumbnail for list performance:
Glide.with(context)
    .load(imageUrl)
    .thumbnail(0.1f)  // load 10% size first, then full
    .into(imageView)
```

---

## Guardrails

### DO
- Annotate UI state classes with `@Immutable` or use `ImmutableList` — unstable types cause silent recomposition storms.
- Use `derivedStateOf` for any value computed from state that changes less often than its source.
- Use `remember(key)` for any calculation inside a Composable that isn't trivial.
- Always provide stable `key` lambdas in `LazyColumn`/`LazyRow` — use entity IDs, never list index.
- Add `StrictMode` in `Application.onCreate()` behind `BuildConfig.DEBUG` — catch main thread violations before they become ANRs.
- Use Coil for all image loading in Compose projects — never manually decode bitmaps for remote images.
- Share the app's `OkHttpClient` with Coil's `ImageLoader` — avoids a second connection pool.
- Generate Compose compiler metrics before each release to catch new unstable types.

### DON'T
- Don't read files or query a database inside a Composable — move to ViewModel or a coroutine launched from `LaunchedEffect`.
- Don't create `remember { }` without a key for values that depend on parameters — stale values will be returned after parameter changes.
- Don't use `mutableStateListOf` or `mutableStateMapOf` at ViewModel scope — these are Compose-specific; use `StateFlow<ImmutableList<T>>` instead.
- Don't skip Baseline Profiles for production apps — cold start is the first impression.
- Don't load full-resolution images into small thumbnails — use Coil's `size()` modifier or Glide's `override()` to downsample at decode time.
- Don't ignore StrictMode logs during development — they mark the exact callsite that will eventually ANR a real user.

---

## References
- [Compose performance](https://developer.android.com/develop/ui/compose/performance)
- [Compose compiler metrics](https://github.com/androidx/androidx/blob/androidx-main/compose/compiler/design/compiler-metrics.md)
- [Baseline Profiles](https://developer.android.com/topic/performance/baselineprofiles)
- [App Startup](https://developer.android.com/topic/libraries/app-startup)
- [Macrobenchmark](https://developer.android.com/studio/profile/macrobenchmark-overview)
- [Perfetto](https://developer.android.com/studio/profile/record-traces)
- [StrictMode](https://developer.android.com/reference/android/os/StrictMode)
- [Coil](https://coil-kt.github.io/coil/)
- [Glide](https://bumptech.github.io/glide/)
