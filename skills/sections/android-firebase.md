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
fun logError(tag: String, e: Throwable) {
    Timber.e(e, tag)
    FirebaseCrashlytics.getInstance().recordException(e)
}

fun setUser(userId: String) {
    FirebaseCrashlytics.getInstance().setUserId(userId)
}

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
