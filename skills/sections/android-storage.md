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
            // Add .addMigrations(MIGRATION_X_Y) for each schema version change
            // Never use .fallbackToDestructiveMigration() in production — it silently deletes user data
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
