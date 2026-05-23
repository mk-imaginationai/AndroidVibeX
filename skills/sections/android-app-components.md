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
