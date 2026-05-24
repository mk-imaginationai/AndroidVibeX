---
name: android-ui-layouts
description: Use when working with Fragments, ConstraintLayout (XML), TextView, ImageView, App Shortcuts, or Navigation Components in XML-based Views. For Jetpack Compose and Material 3, use android-ui instead.
---

> **Deprecated for Compose work.** This skill covers XML Views only. For Jetpack Compose, Material 3 theming, and adaptive layouts use `android-ui`.

# Android UI & Layouts

## Concept

Android UI is built in two paradigms:
- **Jetpack Compose** — declarative, Kotlin-only, recommended for all new UI
- **XML Views** — imperative, legacy system, still valid for existing codebases

Key components: Composables (Compose) or Views (XML), Fragments (screen containers), Navigation Component (backstack management).

---

## Implementation Patterns

### Jetpack Compose — Basic screen structure

```kotlin
@Composable
fun HomeScreen(
    viewModel: HomeViewModel = hiltViewModel()
) {
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()

    when (val state = uiState) {
        is HomeUiState.Loading -> CircularProgressIndicator()
        is HomeUiState.Success -> HomeContent(items = state.items)
        is HomeUiState.Error -> ErrorMessage(message = state.message)
    }
}

@Composable
private fun HomeContent(items: List<Item>) {
    LazyColumn {
        items(items) { item ->
            ItemRow(item = item)
        }
    }
}
```

### Compose — ConstraintLayout

```kotlin
@Composable
fun ProfileCard() {
    ConstraintLayout(modifier = Modifier.fillMaxWidth()) {
        val (avatar, name, bio) = createRefs()

        Image(
            painter = painterResource(R.drawable.avatar),
            contentDescription = null,
            modifier = Modifier.constrainAs(avatar) {
                top.linkTo(parent.top, margin = 16.dp)
                start.linkTo(parent.start, margin = 16.dp)
            }
        )

        Text(
            text = "Name",
            modifier = Modifier.constrainAs(name) {
                top.linkTo(avatar.top)
                start.linkTo(avatar.end, margin = 8.dp)
            }
        )
    }
}
```

### Fragment — Compose-based Fragment

```kotlin
@AndroidEntryPoint
class HomeFragment : Fragment() {

    override fun onCreateView(
        inflater: LayoutInflater,
        container: ViewGroup?,
        savedInstanceState: Bundle?
    ): View = ComposeView(requireContext()).apply {
        setViewCompositionStrategy(ViewCompositionStrategy.DisposeOnViewTreeLifecycleDestroyed)
        setContent {
            MyAppTheme {
                HomeScreen()
            }
        }
    }
}
```

### Navigation Component — NavGraph with Compose

```kotlin
@Composable
fun AppNavGraph(navController: NavHostController) {
    NavHost(navController = navController, startDestination = "home") {
        composable("home") {
            HomeScreen(
                onNavigateToDetail = { id -> navController.navigate("detail/$id") }
            )
        }
        composable(
            route = "detail/{id}",
            arguments = listOf(navArgument("id") { type = NavType.StringType })
        ) { backStackEntry ->
            DetailScreen(id = backStackEntry.arguments?.getString("id").orEmpty())
        }
    }
}
```

### App Shortcuts — Static shortcut in shortcuts.xml

```xml
<!-- res/xml/shortcuts.xml -->
<shortcuts xmlns:android="http://schemas.android.com/apk/res/android">
    <shortcut
        android:shortcutId="compose_message"
        android:enabled="true"
        android:icon="@drawable/ic_compose"
        android:shortcutShortLabel="@string/compose_shortcut_short_label"
        android:shortcutLongLabel="@string/compose_shortcut_long_label">
        <intent
            android:action="android.intent.action.VIEW"
            android:targetPackage="com.example.app"
            android:targetClass="com.example.app.ComposeActivity" />
    </shortcut>
</shortcuts>
```

---

## Guardrails

### DO
- Use `collectAsStateWithLifecycle()` (from `lifecycle-runtime-compose`) for Flow in Compose — not `collectAsState()`.
- Keep Composables small and focused — extract if a function exceeds ~50 lines.
- Use `hiltViewModel()` in Composables (not constructor injection).
- Use `LazyColumn`/`LazyRow` for lists, never `Column` with a loop for more than ~10 items.
- Use `rememberSaveable` for state that should survive recomposition AND process death.
- Pass navigation callbacks as lambdas (`onNavigateToDetail: (String) -> Unit`) — never pass NavController into Composables directly.

### DON'T
- Don't put business logic in Composables — only UI rendering and event forwarding.
- Don't use `Fragment` for new Compose-only screens — use Compose navigation directly.
- Don't mix XML and Compose casually — only interop when migrating existing screens.
- Don't hold references to `Context` or `Activity` inside a Composable — use `LocalContext.current` when absolutely needed.
- Don't call `invalidate()`, `notifyDataSetChanged()`, or similar imperative refresh patterns in Compose.

---

## References
- [Jetpack Compose](https://developer.android.com/jetpack/compose)
- [Compose Layout Basics](https://developer.android.com/develop/ui/compose/layouts/basics)
- [ConstraintLayout in Compose](https://developer.android.com/develop/ui/compose/layouts/constraintlayout)
- [Fragments](https://developer.android.com/guide/fragments)
- [Navigation Components](https://developer.android.com/guide/navigation)
- [App Shortcuts](https://developer.android.com/guide/topics/ui/shortcuts)
