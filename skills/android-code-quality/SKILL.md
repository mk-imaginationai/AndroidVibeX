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
            Timber.plant(CrashReportingTree())
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

// Usage
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
