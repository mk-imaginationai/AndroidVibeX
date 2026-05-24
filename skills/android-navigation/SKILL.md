---
name: android-navigation
description: Use when setting up navigation in an Android Compose app. Covers type-safe Navigation 2.8+ with @Serializable routes, NavHost, argument passing, deep links, back stack control, nested graphs, and dialog/bottom sheet destinations.
---

# Android Navigation

## Concept

| Concept | What it is |
|---|---|
| **Route** | `@Serializable` data object or data class — the destination identity |
| **NavHost** | The composable that renders the current destination |
| **NavController** | Holds back stack state; obtain via `rememberNavController()` |
| **Back stack** | Stack of destinations; `navigate()` pushes, `popBackStack()` pops |
| **Deep link** | URL pattern that jumps directly to a destination |

Navigation 2.8+ replaces string routes with type-safe Kotlin objects. Requires `kotlinx.serialization`.

```toml
# libs.versions.toml
navigation-compose = "2.8.0"
[libraries]
androidx-navigation-compose = { group = "androidx.navigation", name = "navigation-compose", version.ref = "navigation-compose" }
```

---

## Implementation Patterns

### Route Definitions

```kotlin
// navigation/Routes.kt — sealed hierarchy keeps all routes in one place
@Serializable data object HomeRoute
@Serializable data object SettingsRoute
@Serializable data class TaskDetailRoute(val taskId: String)
@Serializable data class EditTaskRoute(val taskId: String? = null)  // null = new task
```

### NavHost Setup

```kotlin
// navigation/AppNavHost.kt
@Composable
fun AppNavHost(
    navController: NavHostController = rememberNavController(),
    modifier: Modifier = Modifier
) {
    NavHost(
        navController = navController,
        startDestination = HomeRoute,
        modifier = modifier
    ) {
        composable<HomeRoute> {
            HomeRoute(
                onNavigateToDetail = { id -> navController.navigate(TaskDetailRoute(id)) },
                onNavigateToSettings = { navController.navigate(SettingsRoute) }
            )
        }

        composable<TaskDetailRoute> { backStackEntry ->
            val route = backStackEntry.toRoute<TaskDetailRoute>()
            TaskDetailRoute(
                taskId = route.taskId,
                onBack = { navController.popBackStack() }
            )
        }

        composable<EditTaskRoute> { backStackEntry ->
            val route = backStackEntry.toRoute<EditTaskRoute>()
            EditTaskRoute(
                taskId = route.taskId,  // null = create mode
                onSaved = { navController.popBackStack() },
                onBack = { navController.popBackStack() }
            )
        }

        composable<SettingsRoute> {
            SettingsRoute(onBack = { navController.popBackStack() })
        }
    }
}
```

> Each `composable<Route>` block is a **Route composable** — it owns the ViewModel and passes state/callbacks down to a stateless screen composable. See `android-architecture` for the Route/Screen split pattern.

### Passing the NavController

Never pass `NavController` into screen composables. Pass lambda callbacks instead:

```kotlin
// WRONG — untestable, couples composable to nav graph
@Composable
fun HomeScreen(navController: NavController) {
    Button(onClick = { navController.navigate(SettingsRoute) }) { ... }
}

// CORRECT — stateless, testable
@Composable
fun HomeScreen(onNavigateToSettings: () -> Unit) {
    Button(onClick = onNavigateToSettings) { ... }
}
```

### Back Stack Control

```kotlin
// Pop back to a specific destination, keeping it in the stack
navController.popBackStack(route = HomeRoute, inclusive = false)

// Navigate and clear entire back stack (e.g., after login)
navController.navigate(HomeRoute) {
    popUpTo(navController.graph.startDestinationId) { inclusive = true }
}

// Bottom nav tab selection — avoid duplicate entries, restore saved state
navController.navigate(tab.route) {
    popUpTo(navController.graph.startDestinationId) { saveState = true }
    launchSingleTop = true
    restoreState = true
}
```

### Dialog Destinations

```kotlin
// Define a dialog route
@Serializable data class ConfirmDeleteRoute(val taskId: String)

// Register in NavHost
dialog<ConfirmDeleteRoute> { backStackEntry ->
    val route = backStackEntry.toRoute<ConfirmDeleteRoute>()
    ConfirmDeleteDialog(
        taskId = route.taskId,
        onConfirm = {
            viewModel.deleteTask(route.taskId)
            navController.popBackStack()
        },
        onDismiss = { navController.popBackStack() }
    )
}

// Navigate to it like any destination
navController.navigate(ConfirmDeleteRoute(taskId = "123"))
```

### Nested Navigation Graphs

```kotlin
@Serializable data object OnboardingGraphRoute

fun NavGraphBuilder.onboardingGraph(navController: NavHostController) {
    navigation<OnboardingGraphRoute>(startDestination = WelcomeRoute) {
        composable<WelcomeRoute> {
            WelcomeRoute(onNext = { navController.navigate(PermissionsRoute) })
        }
        composable<PermissionsRoute> {
            PermissionsRoute(
                onComplete = {
                    navController.navigate(HomeRoute) {
                        popUpTo(OnboardingGraphRoute) { inclusive = true }
                    }
                }
            )
        }
    }
}

// In AppNavHost:
NavHost(navController, startDestination = OnboardingGraphRoute) {
    onboardingGraph(navController)
    composable<HomeRoute> { ... }
}
```

### Deep Links

```kotlin
composable<TaskDetailRoute>(
    deepLinks = listOf(
        navDeepLink<TaskDetailRoute>(basePath = "https://myapp.com/task")
    )
) { backStackEntry ->
    val route = backStackEntry.toRoute<TaskDetailRoute>()
    TaskDetailRoute(taskId = route.taskId, onBack = { navController.popBackStack() })
}

// AndroidManifest.xml — declare the intent filter on MainActivity:
// <intent-filter>
//   <action android:name="android.intent.action.VIEW" />
//   <category android:name="android.intent.category.DEFAULT" />
//   <category android:name="android.intent.category.BROWSABLE" />
//   <data android:scheme="https" android:host="myapp.com" />
// </intent-filter>
```

### Bottom Navigation Highlight (current route detection)

```kotlin
@Composable
fun AppBottomBar(navController: NavController, tabs: List<BottomTab>) {
    val navBackStackEntry by navController.currentBackStackEntryAsState()
    val currentDestination = navBackStackEntry?.destination

    NavigationBar {
        tabs.forEach { tab ->
            NavigationBarItem(
                selected = currentDestination?.hierarchy?.any {
                    it.hasRoute(tab.route::class)
                } == true,
                onClick = {
                    navController.navigate(tab.route) {
                        popUpTo(navController.graph.startDestinationId) { saveState = true }
                        launchSingleTop = true
                        restoreState = true
                    }
                },
                icon = { Icon(tab.icon, contentDescription = tab.label) },
                label = { Text(tab.label) }
            )
        }
    }
}
```

### Hilt ViewModels in Navigation

```kotlin
composable<TaskDetailRoute> { backStackEntry ->
    val route = backStackEntry.toRoute<TaskDetailRoute>()
    val viewModel: TaskDetailViewModel = hiltViewModel()
    // ViewModel is scoped to this back stack entry — destroyed when popped
    TaskDetailScreen(
        uiState = viewModel.uiState.collectAsStateWithLifecycle().value,
        onAction = viewModel::onAction
    )
}
```

### Returning a Result to the Previous Screen

```kotlin
// In the destination screen's Route composable — write to previous back stack entry:
val previousEntry = navController.previousBackStackEntry
previousEntry?.savedStateHandle?.set("selected_item", selectedItem)
navController.popBackStack()

// In the caller's Route composable — observe:
val savedStateHandle = navController.currentBackStackEntry?.savedStateHandle
val result = savedStateHandle
    ?.getStateFlow<Item?>("selected_item", null)
    ?.collectAsStateWithLifecycle()
```

---

## Guardrails

### DO
- Define all routes in a single `Routes.kt` — keeps the navigation contract in one place.
- Use `toRoute<T>()` to extract typed arguments from `NavBackStackEntry`.
- Scope ViewModels to the back stack entry via `hiltViewModel()` inside Route composables — they are destroyed when the destination is popped.
- Use `launchSingleTop = true` for bottom nav tabs to prevent duplicate back stack entries.
- Use `dialog<Route>` for confirmation/picker dialogs instead of `showDialog` boolean state in the parent.

### DON'T
- Don't pass `NavController` into screen composables — always pass lambda callbacks.
- Don't call `navigate()` from a ViewModel — emit a `UiEvent` via `Channel` and let the Route composable respond.
- Don't use string-based routes for new code — use `@Serializable` type-safe routes.
- Don't use `popUpTo` with an integer ID in new code — use the route type (`popUpTo<HomeRoute>`).
- Don't navigate inside `LaunchedEffect(Unit)` as a substitute for ViewModel-driven one-shot events — it re-fires on recomposition.

---

## Navigation 3 (next-generation library)

Navigation 3 is a separate, newer library (`androidx.navigation:navigation3-*`) with a different API. Use it for new projects targeting adaptive/multi-pane layouts. Navigation 2.8 remains stable and is fine for most apps.

**Key differences from Nav 2:**

| | Navigation 2.8 | Navigation 3 |
|---|---|---|
| Back stack | managed internally | `List<Any>` as Compose state |
| Host | `NavHost` | `NavDisplay` |
| Entry | `composable<Route>` | `NavEntry` |
| Multi-pane | manual | `ListDetailSceneStrategy` / `SupportingPaneSceneStrategy` |

```kotlin
// Navigation 3 — basic setup
val backStack = remember { mutableStateListOf<Any>(HomeRoute) }

NavDisplay(
    backStack = backStack,
    onBack = { if (backStack.size > 1) backStack.removeLast() },
    entryProvider = entryProvider {
        entry<HomeRoute> {
            HomeScreen(onNavigate = { backStack.add(DetailRoute(it)) })
        }
        entry<DetailRoute> { entry ->
            DetailScreen(id = entry.key.id, onBack = { backStack.removeLast() })
        }
    }
)

// List-detail multi-pane (Navigation 3 + material3-adaptive)
val strategy = rememberListDetailSceneStrategy<Any>()
NavDisplay(
    backStack = backStack,
    sceneStrategies = listOf(strategy),
    entryProvider = entryProvider {
        entry<HomeRoute>(metadata = ListDetailSceneStrategy.listPane(
            detailPlaceholder = { Text("Select an item") }
        )) { HomeScreen(...) }

        entry<DetailRoute>(metadata = ListDetailSceneStrategy.detailPane()) {
            DetailScreen(...)
        }
    }
)
```

Navigation 3 also uses `NavigationSuiteScaffold` for adaptive nav bar/rail — replace `Scaffold + NavigationBar`:

```kotlin
NavigationSuiteScaffold(
    navigationSuiteItems = {
        navItems.forEachIndexed { index, item ->
            item(
                selected = selectedIndex == index,
                onClick = { selectedIndex = index },
                icon = { Icon(item.icon, item.label) },
                label = { Text(item.label) }
            )
        }
    }
) {
    // screen content — NavigationSuiteScaffold auto-switches between
    // NavigationBar (compact) and NavigationRail (medium/expanded)
}
```

---

## References
- [Navigation Compose](https://developer.android.com/jetpack/compose/navigation)
- [Type-safe navigation (2.8+)](https://developer.android.com/guide/navigation/design/type-safety)
- [Navigation 3](https://developer.android.com/guide/navigation/navigation-3)
- [Deep links](https://developer.android.com/guide/navigation/design/deep-link)
- [Nested graphs](https://developer.android.com/guide/navigation/design/nested-graphs)
