---
name: android-build
description: Use when upgrading Android Gradle Plugin (AGP), migrating from kapt to KSP, managing version catalogs, or configuring ProGuard/R8 rules. Covers AGP 9 upgrade steps, built-in Kotlin migration, and release build rule patterns.
---

# Android Build

## AGP 9 Upgrade

AGP 9 has multiple breaking changes from AGP 8. Run **AGP Upgrade Assistant** in Android Studio first, then apply the steps below.

### Step 1 — Update minimum versions

```toml
# libs.versions.toml
agp   = "9.0.0"
kotlin = "2.3.0"
ksp   = "2.3.6-1.0.31"   # must be 2.3.6+ for AGP 9
hilt  = "2.59.2"           # must be 2.59.2+ for AGP 9
```

### Step 2 — Migrate to built-in Kotlin

AGP 9 ships Kotlin as a built-in dependency. Remove the standalone version override:

```kotlin
// settings.gradle.kts — remove explicit version, AGP resolves it
// id("org.jetbrains.kotlin.android") version "x.x.x"  ← delete version

// app/build.gradle.kts — alias stays, version comes from AGP
plugins {
    alias(libs.plugins.android.application)
    alias(libs.plugins.kotlin.android)
}
```

### Step 3 — Migrate kapt → KSP

```toml
# libs.versions.toml
[plugins]
ksp = { id = "com.google.devtools.ksp", version.ref = "ksp" }
# remove: kotlin-kapt
```

```kotlin
// app/build.gradle.kts
plugins {
    alias(libs.plugins.ksp)
    // REMOVE: alias(libs.plugins.kotlin.kapt)
}

dependencies {
    ksp(libs.hilt.compiler)   // was: kapt(libs.hilt.compiler)
    ksp(libs.room.compiler)   // was: kapt(libs.room.compiler)
}
```

### Step 4 — New DSL: jvmTarget

```kotlin
// REMOVED in AGP 9
// android { kotlinOptions { jvmTarget = "11" } }

// CORRECT
kotlin {
    compilerOptions {
        jvmTarget.set(org.jetbrains.kotlin.gradle.dsl.JvmTarget.JVM_11)
    }
}
```

### Step 5 — Clean up gradle.properties

```properties
# DELETE if present — obsolete in AGP 9:
# android.builtInKotlin=true
# android.newDsl=true
# android.uniquePackageNames=true
# android.enableAppCompileTimeRClass=true
```

### Verification

```bash
./gradlew help               # confirms task graph resolves
./gradlew build --dry-run    # dry-run build — no clean needed
```

---

## Version Catalog Canonical Structure

```toml
# libs.versions.toml
[versions]
agp              = "9.0.0"
kotlin           = "2.3.0"
compose-bom      = "2024.12.01"
hilt             = "2.59.2"
room             = "2.7.0"
ksp              = "2.3.6-1.0.31"
retrofit         = "2.11.0"
okhttp           = "4.12.0"
navigation       = "2.8.9"
work             = "2.9.0"

[libraries]
# Compose BOM — omit version on individual compose libs
androidx-compose-bom      = { group = "androidx.compose", name = "compose-bom", version.ref = "compose-bom" }
hilt-android              = { group = "com.google.dagger", name = "hilt-android", version.ref = "hilt" }
hilt-compiler             = { group = "com.google.dagger", name = "hilt-android-compiler", version.ref = "hilt" }
room-runtime              = { group = "androidx.room", name = "room-runtime", version.ref = "room" }
room-ktx                  = { group = "androidx.room", name = "room-ktx", version.ref = "room" }
room-compiler             = { group = "androidx.room", name = "room-compiler", version.ref = "room" }

[plugins]
android-application  = { id = "com.android.application", version.ref = "agp" }
kotlin-android       = { id = "org.jetbrains.kotlin.android", version.ref = "kotlin" }
kotlin-compose       = { id = "org.jetbrains.kotlin.plugin.compose", version.ref = "kotlin" }
kotlin-serialization = { id = "org.jetbrains.kotlin.plugin.serialization", version.ref = "kotlin" }
hilt                 = { id = "com.google.dagger.hilt.android", version.ref = "hilt" }
ksp                  = { id = "com.google.devtools.ksp", version.ref = "ksp" }
```

---

## ProGuard / R8 Rules

R8 is enabled in release builds by default. Missing rules → "works in debug, crashes in release."

```proguard
# proguard-rules.pro

# kotlinx.serialization
-keepattributes *Annotation*, InnerClasses
-dontnote kotlinx.serialization.AnnotationsKt
-keep @kotlinx.serialization.Serializable class * { *; }
-keepclassmembers class kotlinx.serialization.json.** { *** Companion; }

# Retrofit + OkHttp
-keepattributes Signature, Exceptions
-keep class retrofit2.** { *; }
-keep interface retrofit2.** { *; }
-dontwarn okhttp3.**
-dontwarn okio.**

# Hilt (mostly auto-handled by hilt-android-gradle-plugin)
-keep class dagger.hilt.** { *; }

# Room — keep entity classes
-keep class * extends androidx.room.RoomDatabase { *; }
-keepclassmembers @androidx.room.Entity class * { *; }

# API response models (adjust package to your own)
-keep class com.example.myapp.data.model.** { *; }

# Timber
-dontwarn org.jetbrains.annotations.**
```

Verify R8 output: test the **release APK** on a device. R8 failures appear as `ClassNotFoundException` or `NoSuchMethodException` at runtime.

---

## Guardrails

### DO
- Run AGP Upgrade Assistant in Android Studio before manual edits.
- Always test the release APK after changing proguard rules.
- Use `ksp()` for all annotation processors — `kapt` is legacy.
- Pin `compose-bom` in `libs.versions.toml`; omit version on individual Compose libs.

### DON'T
- Don't run `./gradlew clean` as part of build verification — adds time without value.
- Don't use `kapt` for new projects.
- Don't add broad `-keep class com.example.**` rules — bloats APK and defeats R8 optimization.
- Don't leave `android.builtInKotlin` / `android.newDsl` flags in `gradle.properties` after AGP 9 migration.

---

## References
- [AGP 9 release notes](https://developer.android.com/build/releases/gradle-plugin)
- [Migrate to KSP](https://developer.android.com/build/migrate-to-ksp)
- [Version Catalog](https://docs.gradle.org/current/userguide/platforms.html)
- [R8 / ProGuard rules](https://developer.android.com/build/shrink-code)
