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

### Foreground Service — user-visible long-running work

Declare in `AndroidManifest.xml`:
```xml
<service
    android:name=".SyncService"
    android:foregroundServiceType="dataSync"
    android:exported="false" />
```

Required permissions:
```xml
<uses-permission android:name="android.permission.FOREGROUND_SERVICE" />
<uses-permission android:name="android.permission.FOREGROUND_SERVICE_DATA_SYNC" />
```

Start with type parameter (required API 34+):
```kotlin
// Create notification channel (required API 26+) — do this in Application.onCreate()
if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
    val channel = NotificationChannel(
        CHANNEL_ID,
        "Sync Service",
        NotificationManager.IMPORTANCE_LOW
    )
    getSystemService(NotificationManager::class.java).createNotificationChannel(channel)
}

// Request POST_NOTIFICATIONS permission at runtime (required API 33+) before starting service
// ActivityCompat.requestPermissions(activity, arrayOf(Manifest.permission.POST_NOTIFICATIONS), RC)

val notification = NotificationCompat.Builder(this, CHANNEL_ID)
    .setContentTitle("Syncing")
    .setSmallIcon(R.drawable.ic_sync)
    .setOngoing(true)
    .build()

// Must call startForeground() within 5 seconds of onStartCommand() — call it first, before launching coroutines
startForeground(NOTIFICATION_ID, notification, ServiceInfo.FOREGROUND_SERVICE_TYPE_DATA_SYNC)
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

Modern network monitoring (API 21+) uses NetworkCallback. If using BroadcastReceiver (legacy), API 34+ requires exported flag:
```kotlin
// Modern approach (API 21+): NetworkCallback instead of BroadcastReceiver
val connectivityManager = getSystemService(ConnectivityManager::class.java)
val networkCallback = object : ConnectivityManager.NetworkCallback() {
    override fun onAvailable(network: Network) { /* network connected */ }
    override fun onLost(network: Network) { /* network lost */ }
}
connectivityManager.registerDefaultNetworkCallback(networkCallback)
// unregister in onDestroy
connectivityManager.unregisterNetworkCallback(networkCallback)

// If you still need BroadcastReceiver for legacy reasons (API < 21):
val receiver = NetworkReceiver()
val filter = IntentFilter(ConnectivityManager.CONNECTIVITY_ACTION)
// API 34+ requires explicit exported flag for dynamic receivers
ContextCompat.registerReceiver(this, receiver, filter, ContextCompat.RECEIVER_NOT_EXPORTED)
// unregister in onDestroy
unregisterReceiver(receiver)
```

### Fragment — Lifecycle-aware setup with Compose

```kotlin
@AndroidEntryPoint
class HomeFragment : Fragment() {

    private val viewModel: HomeViewModel by viewModels()

    override fun onCreateView(
        inflater: LayoutInflater,
        container: ViewGroup?,
        savedInstanceState: Bundle?
    ): View = ComposeView(requireContext()).apply {
        setViewCompositionStrategy(ViewCompositionStrategy.DisposeOnViewTreeLifecycleDestroyed)
        setContent {
            MyAppTheme {
                HomeScreen(viewModel = viewModel)
            }
        }
    }
}
```

Key Fragment lifecycle note: use `viewLifecycleOwner` (not `this`) when observing in Fragments — the View has a shorter lifecycle than the Fragment itself.

### Content Provider — Expose data to other apps

```kotlin
class ItemsProvider : ContentProvider() {

    override fun query(
        uri: Uri, projection: Array<String>?, selection: String?,
        selectionArgs: Array<String>?, sortOrder: String?
    ): Cursor? {
        requireNotNull(context).checkCallingPermission(READ_PERMISSION).let { result ->
            if (result != PackageManager.PERMISSION_GRANTED) return null
        }
        // query and return cursor
        return null
    }

    override fun getType(uri: Uri): String = "vnd.android.cursor.dir/vnd.com.example.items"
    override fun insert(uri: Uri, values: ContentValues?): Uri? = null
    override fun delete(uri: Uri, selection: String?, selectionArgs: Array<String>?): Int = 0
    override fun update(uri: Uri, values: ContentValues?, selection: String?, selectionArgs: Array<String>?): Int = 0
    override fun onCreate(): Boolean = true
}
```

Declare in manifest with permissions:
```xml
<provider
    android:name=".ItemsProvider"
    android:authorities="com.example.app.provider"
    android:exported="true"
    android:readPermission="com.example.app.READ_ITEMS" />
```

---

## Guardrails

### DO
- Use `ViewModel` to survive configuration changes — never store UI data in Activity fields.
- Always check `intent.resolveActivity(packageManager) != null` before firing implicit intents. Prefer explicit intents where possible — implicit intents can be intercepted by other apps.
- Use `lifecycleScope` for coroutines launched from Activity/Fragment.
- Declare only the permissions you need in the manifest.
- Prefer `registerReceiver()` at runtime over manifest registration for dynamic events.
- Use `foreground services` with a notification for user-visible long-running work.

### DON'T
- Don't put business logic in `Activity` or `Fragment` — delegate to `ViewModel`.
- Don't start a `Service` for short background tasks — use `WorkManager` or coroutines.
- Don't ignore `onStop`/`onDestroy` — release resources, cancel jobs, unregister receivers. In a Service, always cancel the `CoroutineScope` in `onDestroy()` — see the Service example above.
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
