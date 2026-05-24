# AndroidVibeX

Production-grade Android development skills for [Claude Code](https://claude.ai/code).

Covers Kotlin, Jetpack Compose, Material 3, Clean Architecture, Hilt, Room, Retrofit, WorkManager, CameraX, and more — with a working example app ([VibeTasks](https://github.com/your-username/VibeTasks)).

## Install

```bash
claude plugin install github:your-username/AndroidVibeX
```

## What's included

### Section skills (load automatically when relevant)

| Skill | Covers |
|---|---|
| `android-ui` | Material 3 theming, ColorScheme, Typography, Shapes, dynamic color, NavigationSuiteScaffold, adaptive layouts |
| `android-navigation` | Type-safe Navigation 2.8+ with `@Serializable` routes, NavHost, deep links, dialogs, nested graphs |
| `android-architecture` | MVVM + Repository, Clean Architecture layers, Hilt DI, Use Cases, sealed UI state |
| `android-networking` | Retrofit + OkHttp, kotlinx.serialization, `networkBoundResource`, Result wrapping |
| `android-storage` | Room (Entity, DAO, TypeConverter, Relation, migrations), DataStore |
| `android-async` | Coroutines, Flow, StateFlow, `viewModelScope`, `collectAsStateWithLifecycle` |
| `android-workmanager` | `CoroutineWorker`, `@HiltWorker`, `enqueueUniqueWork`, periodic work, Hilt integration |
| `android-camera` | CameraX, `ProcessCameraProvider`, `ImageCapture`, tap-to-focus, permission handling |
| `android-build` | AGP 9 upgrade, KSP, ProGuard/R8 rules, Baseline Profiles |
| `android-testing` | ViewModel unit tests with fakes, Room integration tests, Compose UI tests, Turbine, Roborazzi |
| `android-performance` | `@Stable`/`@Immutable`, `ImmutableList`, `LazyColumn` keys, Baseline Profiles, Macrobenchmark |
| `android-firebase` | Firestore, Auth, FCM, Crashlytics, Remote Config |
| `android-app-components` | Activity, Fragment, BroadcastReceiver, Service lifecycle |
| `android-code-quality` | Lint, Detekt, StrictMode, LeakCanary |

### Agent skills (invoke explicitly)

| Agent | Role |
|---|---|
| `android-architect` | Designs feature architecture — module structure, data flow, DI wiring |
| `android-implementer` | Executes implementation plans, follows section skills |
| `android-reviewer` | Reviews code for correctness, architecture, and skill compliance |
| `android-debugger` | Diagnoses build errors, crashes, and runtime issues |

## Usage

Once installed, invoke any agent or skill in Claude Code:

```
/android-architect design the task sync feature
/android-reviewer review the current diff
/android-implementer implement the camera capture flow
```

Section skills are loaded automatically by the implementer when relevant.

## Example project

VibeTasks demonstrates every skill in a working Android app. See `VibeTasks/` in this repo or the standalone repo.

## Rules

`CLAUDE.md` encodes project-wide Android engineering rules (Kotlin-only, MVVM, Coroutines, Hilt, Room, Timber, etc.). These load automatically for any project that includes this plugin.

## Contributing

PRs welcome. Each skill lives in `skills/sections/` or `skills/agents/`. Follow the existing format: Concept table → Implementation patterns → Guardrails (DO/DON'T) → References.
