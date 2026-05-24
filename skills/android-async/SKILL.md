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
suspend fun loadDashboard(): Result<DashboardData> = runCatching {
    coroutineScope {
        val userDeferred = async { userRepository.getCurrentUser().getOrThrow() }
        val statsDeferred = async { statsRepository.getStats().getOrThrow() }
        DashboardData(
            user = userDeferred.await(),
            stats = statsDeferred.await()
        )
    }
}
```

The outer `runCatching` catches any exception thrown by `getOrThrow()` inside the `coroutineScope`, so the Repository boundary contract (no raw exceptions reaching the ViewModel) is preserved.

### WorkManager — One-time background task

```kotlin
class SyncWorker(
    context: Context,
    params: WorkerParameters
) : CoroutineWorker(context, params) {

    override suspend fun doWork(): Result {
        return try {
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
