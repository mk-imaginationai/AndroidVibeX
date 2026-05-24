---
name: android-workmanager
description: Use when scheduling deferrable background work that must survive process death. Covers CoroutineWorker, OneTimeWorkRequest, PeriodicWorkRequest, chaining, constraints, Hilt injection with @HiltWorker, and observing work status.
---

# Android WorkManager

## Concept

| Class | Use for |
|---|---|
| **CoroutineWorker** | All new workers — suspending, cancellable, Hilt-injectable |
| **OneTimeWorkRequest** | Work that runs once (sync, upload, cleanup) |
| **PeriodicWorkRequest** | Repeating work — minimum 15-minute interval |
| **WorkChain** | Sequential or parallel pipelines of workers |
| **Constraints** | Preconditions — network type, battery, storage, charging |

**Use WorkManager when:**
- Work must survive app process death or device reboot
- Work needs constraints (Wi-Fi only, battery not low, etc.)
- Work needs automatic retry / backoff
- Work is deferrable (not exact to the millisecond)

**Don't use WorkManager for:**
- Exact alarm scheduling → use `AlarmManager`
- Immediate work while the user is in the app → use coroutines in `viewModelScope`
- Long-running foreground tasks with a visible notification → use a Foreground Service directly

```toml
# libs.versions.toml
work-runtime = "2.9.0"
hilt-work = "1.2.0"
[libraries]
work-runtime-ktx       = { group = "androidx.work", name = "work-runtime-ktx", version.ref = "work-runtime" }
hilt-work              = { group = "androidx.hilt", name = "hilt-work", version.ref = "hilt-work" }
hilt-work-compiler     = { group = "androidx.hilt", name = "hilt-compiler", version.ref = "hilt-work" }
```

---

## Implementation Patterns

### CoroutineWorker with Hilt

```kotlin
// worker/SyncWorker.kt
@HiltWorker
class SyncWorker @AssistedInject constructor(
    @Assisted context: Context,
    @Assisted params: WorkerParameters,
    private val repository: TaskRepository   // injected by Hilt — no manual construction
) : CoroutineWorker(context, params) {

    override suspend fun doWork(): Result {
        return try {
            val taskId = inputData.getString(KEY_TASK_ID)
                ?: return Result.failure(workDataOf(KEY_ERROR to "Missing task ID"))

            repository.syncTask(taskId)

            Result.success(workDataOf(KEY_SYNCED_AT to System.currentTimeMillis()))
        } catch (e: Exception) {
            Timber.e(e, "SyncWorker failed on attempt $runAttemptCount")
            // Retry up to 3 times, then fail permanently
            if (runAttemptCount < 3) Result.retry() else Result.failure()
        }
    }

    companion object {
        const val KEY_TASK_ID = "task_id"
        const val KEY_ERROR = "error"
        const val KEY_SYNCED_AT = "synced_at"
    }
}
```

### Register HiltWorkerFactory

```kotlin
// di/WorkManagerModule.kt
@Module
@InstallIn(SingletonComponent::class)
object WorkManagerModule {
    @Provides
    @Singleton
    fun provideWorkManager(@ApplicationContext context: Context): WorkManager =
        WorkManager.getInstance(context)
}

// MyApp.kt — wire HiltWorkerFactory as the custom Configuration.Provider
@HiltAndroidApp
class MyApp : Application(), Configuration.Provider {

    @Inject lateinit var workerFactory: HiltWorkerFactory

    override val workManagerConfiguration: Configuration
        get() = Configuration.Builder()
            .setWorkerFactory(workerFactory)
            .build()
}
```

Disable the default `WorkManagerInitializer` in `AndroidManifest.xml` — it conflicts with a custom `Configuration.Provider`:

```xml
<provider
    android:name="androidx.startup.InitializationProvider"
    android:authorities="${applicationId}.androidx-startup"
    android:exported="false"
    tools:node="merge">
    <meta-data
        android:name="androidx.work.WorkManagerInitializer"
        android:value="androidx.startup"
        tools:node="remove" />
</provider>
```

### OneTimeWorkRequest

```kotlin
fun enqueueSync(workManager: WorkManager, taskId: String) {
    val constraints = Constraints.Builder()
        .setRequiredNetworkType(NetworkType.CONNECTED)
        .build()

    val request = OneTimeWorkRequestBuilder<SyncWorker>()
        .setConstraints(constraints)
        .setInputData(workDataOf(SyncWorker.KEY_TASK_ID to taskId))
        .setBackoffCriteria(
            BackoffPolicy.EXPONENTIAL,
            WorkRequest.MIN_BACKOFF_MILLIS,
            TimeUnit.MILLISECONDS
        )
        .addTag(TAG_SYNC)
        .build()

    workManager.enqueueUniqueWork(
        "sync_$taskId",               // unique name prevents duplicate runs
        ExistingWorkPolicy.KEEP,      // KEEP = skip if already running; REPLACE = cancel and restart
        request
    )
}

const val TAG_SYNC = "sync"
```

### PeriodicWorkRequest

```kotlin
fun schedulePeriodicSync(workManager: WorkManager) {
    val constraints = Constraints.Builder()
        .setRequiredNetworkType(NetworkType.UNMETERED)  // Wi-Fi only
        .setRequiresBatteryNotLow(true)
        .build()

    val request = PeriodicWorkRequestBuilder<SyncWorker>(
        repeatInterval = 6,
        repeatIntervalTimeUnit = TimeUnit.HOURS,
        flexTimeInterval = 30,               // run within the last 30 min of each 6h window
        flexTimeIntervalUnit = TimeUnit.MINUTES
    )
        .setConstraints(constraints)
        .addTag(TAG_PERIODIC_SYNC)
        .build()

    workManager.enqueueUniquePeriodicWork(
        "periodic_sync",
        ExistingPeriodicWorkPolicy.UPDATE,  // UPDATE = apply new params; KEEP = ignore if exists
        request
    )
}

const val TAG_PERIODIC_SYNC = "periodic_sync"
```

### Chaining Work

```kotlin
// Sequential chain: compress → upload → notify
fun enqueueUploadPipeline(workManager: WorkManager, fileUri: String) {
    val compressRequest = OneTimeWorkRequestBuilder<CompressWorker>()
        .setInputData(workDataOf("uri" to fileUri))
        .build()

    val uploadRequest = OneTimeWorkRequestBuilder<UploadWorker>()
        .setConstraints(
            Constraints.Builder().setRequiredNetworkType(NetworkType.CONNECTED).build()
        )
        .build()

    val notifyRequest = OneTimeWorkRequestBuilder<NotifyWorker>().build()

    workManager
        .beginUniqueWork("upload_pipeline", ExistingWorkPolicy.REPLACE, compressRequest)
        .then(uploadRequest)   // uploadRequest receives compressRequest's outputData as inputData
        .then(notifyRequest)
        .enqueue()
}

// Parallel fan-out then merge
workManager
    .beginWith(listOf(workerA, workerB))  // A and B run concurrently
    .then(mergeWorker)                    // mergeWorker starts after both complete
    .enqueue()
```

### Observing Work Status

```kotlin
class SyncViewModel @Inject constructor(
    private val workManager: WorkManager
) : ViewModel() {

    val syncState: Flow<WorkInfo?> =
        workManager.getWorkInfosForUniqueWorkFlow("periodic_sync")
            .map { it.firstOrNull() }
}

@Composable
fun SyncStatusBadge(viewModel: SyncViewModel = hiltViewModel()) {
    val workInfo by viewModel.syncState.collectAsStateWithLifecycle(initialValue = null)

    when (workInfo?.state) {
        WorkInfo.State.RUNNING   -> CircularProgressIndicator()
        WorkInfo.State.SUCCEEDED -> Icon(Icons.Default.Check, contentDescription = "Synced")
        WorkInfo.State.FAILED    -> Icon(Icons.Default.Error, contentDescription = "Sync failed")
        else -> {}
    }
}
```

### Expedited Work (user-initiated, must start immediately)

```kotlin
val request = OneTimeWorkRequestBuilder<SendMessageWorker>()
    .setExpedited(OutOfQuotaPolicy.RUN_AS_NON_EXPEDITED_WORK_REQUEST)
    .setInputData(workDataOf("message_id" to messageId))
    .build()

workManager.enqueue(request)

// Worker must override getForegroundInfo() for pre-API-31 devices:
class SendMessageWorker(...) : CoroutineWorker(...) {
    override suspend fun getForegroundInfo(): ForegroundInfo =
        ForegroundInfo(NOTIFICATION_ID, buildProgressNotification())

    override suspend fun doWork(): Result { /* ... */ }
}
```

### Cancellation

```kotlin
workManager.cancelUniqueWork("sync_$taskId")      // by unique name
workManager.cancelAllWorkByTag(TAG_SYNC)           // by tag
```

---

## Testing

```kotlin
// testImplementation("androidx.work:work-testing:2.9.0")

@RunWith(AndroidJUnit4::class)
class SyncWorkerTest {

    private lateinit var workManager: WorkManager

    @Before
    fun setup() {
        val context = ApplicationProvider.getApplicationContext<Context>()
        val config = Configuration.Builder()
            .setExecutor(SynchronousExecutor())   // synchronous — no background threads
            .build()
        WorkManagerTestInitHelper.initializeTestWorkManager(context, config)
        workManager = WorkManager.getInstance(context)
    }

    @Test
    fun syncWorker_succeeds_with_valid_input() {
        val request = OneTimeWorkRequestBuilder<SyncWorker>()
            .setInputData(workDataOf(SyncWorker.KEY_TASK_ID to "task_1"))
            .build()

        workManager.enqueue(request).result.get()

        val workInfo = workManager.getWorkInfoById(request.id).get()
        assertEquals(WorkInfo.State.SUCCEEDED, workInfo.state)
    }

    @Test
    fun periodicWork_runs_after_simulated_interval() {
        val request = PeriodicWorkRequestBuilder<SyncWorker>(1, TimeUnit.HOURS).build()
        workManager.enqueue(request)

        val testDriver = WorkManagerTestInitHelper.getTestDriver(context)!!
        testDriver.setPeriodDelayMet(request.id)   // simulate the interval elapsing

        val workInfo = workManager.getWorkInfoById(request.id).get()
        assertEquals(WorkInfo.State.ENQUEUED, workInfo.state)
    }
}
```

---

## Guardrails

### DO
- Always use `CoroutineWorker` — it handles coroutine cancellation correctly and works with `suspend` functions.
- Always use `enqueueUniqueWork` / `enqueueUniquePeriodicWork` — prevents accidental duplicate workers.
- Use `BackoffPolicy.EXPONENTIAL` for network-dependent workers.
- Inject dependencies via `@HiltWorker` + `@AssistedInject` — never construct them manually in `doWork()`.
- Return `Result.retry()` for transient failures (network timeout); `Result.failure()` for permanent ones (invalid data).
- Check `isStopped` inside long loops in `doWork()` — WorkManager can cancel the worker at any time.
- Observe work state via `getWorkInfosForUniqueWorkFlow` in ViewModel — expose it as a `StateFlow`.

### DON'T
- Don't use WorkManager for exact-time alarms — use `AlarmManager`.
- Don't use WorkManager for immediate foreground work — use coroutines in `viewModelScope`.
- Don't pass data payloads larger than 10 KB via `workDataOf` — store large objects in Room/files and pass only the ID.
- Don't forget to remove `WorkManagerInitializer` from the manifest when using a custom `Configuration.Provider` — both initializers will conflict and crash at startup.
- Don't use `ExistingPeriodicWorkPolicy.REPLACE` casually — it resets the interval timer and can cause the work to never run if repeatedly replaced.

---

## References
- [WorkManager overview](https://developer.android.com/topic/libraries/architecture/workmanager)
- [CoroutineWorker](https://developer.android.com/reference/kotlin/androidx/work/CoroutineWorker)
- [Constraints](https://developer.android.com/reference/androidx/work/Constraints)
- [Testing WorkManager](https://developer.android.com/topic/libraries/architecture/workmanager/how-to/testing)
- [Hilt with WorkManager](https://developer.android.com/training/dependency-injection/hilt-jetpack#workmanager)
