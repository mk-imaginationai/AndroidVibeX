---
name: android-testing
description: Use when writing JUnit unit tests, Espresso UI tests, or setting up a testing strategy. Covers ViewModel testing with StandardTestDispatcher, Repository testing with fakes, and UI testing with Espresso/Compose.
---

# Android Testing

## Concept

| Test type | Tool | What to test | Speed |
|---|---|---|---|
| **Unit** | JUnit 4 + MockK | ViewModel, Repository, UseCases | Fast (JVM) |
| **Integration** | JUnit + Room in-memory | Repository + DAO | Medium |
| **UI** | Espresso / Compose Testing | User flows end-to-end | Slow (device) |

Test structure: `Given → When → Then` (or `Arrange → Act → Assert`).

---

## Implementation Patterns

### ViewModel unit test with fake repository

```kotlin
@OptIn(ExperimentalCoroutinesApi::class)
class HomeViewModelTest {

    @get:Rule
    val mainDispatcherRule = MainDispatcherRule()

    private val fakeRepository = FakeItemRepository()
    private lateinit var viewModel: HomeViewModel

    @Before
    fun setup() {
        viewModel = HomeViewModel(fakeRepository)
    }

    @Test
    fun `uiState is Success when repository returns items`() = runTest {
        val items = listOf(Item(id = "1", title = "Test"))
        fakeRepository.setItems(items)

        viewModel.loadItems()

        val state = viewModel.uiState.value
        assertIs<HomeUiState.Success>(state)
        assertEquals(items, state.items)
    }

    @Test
    fun `uiState is Error when repository throws`() = runTest {
        fakeRepository.setShouldThrow(true)

        viewModel.loadItems()

        assertIs<HomeUiState.Error>(viewModel.uiState.value)
    }
}

// MainDispatcherRule — replaces Main dispatcher with test dispatcher
// Uses StandardTestDispatcher (coroutines-test 1.6+); TestCoroutineDispatcher is deprecated
class MainDispatcherRule(
    val dispatcher: TestDispatcher = StandardTestDispatcher()
) : TestWatcher() {
    override fun starting(description: Description) = Dispatchers.setMain(dispatcher)
    override fun finished(description: Description) = Dispatchers.resetMain()
}
```

### Fake Repository

```kotlin
class FakeItemRepository : ItemRepository {
    private var items: List<Item> = emptyList()
    private var shouldThrow = false

    fun setItems(items: List<Item>) { this.items = items }
    fun setShouldThrow(value: Boolean) { shouldThrow = value }

    override suspend fun getItems(): Result<List<Item>> = runCatching {
        if (shouldThrow) error("Forced failure")
        items
    }
}
```

### Room integration test (in-memory)

```kotlin
@RunWith(AndroidJUnit4::class)
class ItemDaoTest {

    private lateinit var db: AppDatabase
    private lateinit var dao: ItemDao

    @Before
    fun setup() {
        db = Room.inMemoryDatabaseBuilder(
            ApplicationProvider.getApplicationContext(),
            AppDatabase::class.java
        ).build()
        dao = db.itemDao()
    }

    @After
    fun teardown() { db.close() }

    @Test
    fun insertAndRetrieve() = runTest {
        val entity = ItemEntity(id = "1", title = "Test", createdAt = 0L)
        dao.insertAll(listOf(entity))

        val result = dao.getAll()
        assertEquals(1, result.size)
        assertEquals("Test", result.first().title)
    }
}
```

### Espresso UI test

```kotlin
@RunWith(AndroidJUnit4::class)
@HiltAndroidTest
class HomeScreenTest {

    @get:Rule(order = 0)
    val hiltRule = HiltAndroidRule(this)

    @get:Rule(order = 1)
    val activityRule = ActivityScenarioRule(MainActivity::class.java)

    @Before
    fun setup() { hiltRule.inject() }

    @Test
    fun itemsDisplayedOnSuccess() {
        onView(withId(R.id.recycler_view))
            .check(matches(isDisplayed()))

        onView(withText("Test Item"))
            .check(matches(isDisplayed()))
    }

    @Test
    fun clickingItemNavigatesToDetail() {
        onView(withText("Test Item")).perform(click())

        onView(withId(R.id.detail_title))
            .check(matches(isDisplayed()))
    }
}
```

### Compose UI test

```kotlin
@RunWith(AndroidJUnit4::class)
class HomeScreenComposeTest {

    @get:Rule
    val composeTestRule = createComposeRule()

    @Test
    fun showsLoadingInitially() {
        composeTestRule.setContent {
            HomeScreen(viewModel = FakeHomeViewModel(HomeUiState.Loading))
        }
        composeTestRule.onNodeWithTag("loading_indicator").assertIsDisplayed()
    }

    @Test
    fun showsItemsOnSuccess() {
        val items = listOf(Item(id = "1", title = "Test Item"))
        composeTestRule.setContent {
            HomeScreen(viewModel = FakeHomeViewModel(HomeUiState.Success(items)))
        }
        composeTestRule.onNodeWithText("Test Item").assertIsDisplayed()
    }
}
```

---

## Guardrails

### DO
- Use fake implementations (not mocks) for Repository in ViewModel tests — fakes are more reliable.
- Use `runTest` from `kotlinx-coroutines-test` for all coroutine-based tests.
- Use `MainDispatcherRule` in every ViewModel test that touches `Dispatchers.Main`.
- Test ViewModels in isolation — inject fakes, not real implementations.
- Use `Room.inMemoryDatabaseBuilder` for DAO tests — never a real on-disk database.

### DON'T
- Don't write tests that rely on specific timing — use `runTest` and `advanceUntilIdle()`.
- Don't use `Thread.sleep()` in tests — use `runTest` coroutine control.
- Don't test implementation details (private methods, internal state) — test observable behavior.
- Don't skip the `@After` teardown — always close in-memory databases.
- Don't share ViewModel instances between tests — recreate in `@Before`.

---

## References
- [Testing overview](https://developer.android.com/training/testing)
- [JUnit local tests](https://developer.android.com/training/testing/local-tests)
- [Espresso](https://developer.android.com/training/testing/espresso)
- [Compose testing](https://developer.android.com/develop/ui/compose/testing)
