---
name: android-ui
description: Use when building UI with Jetpack Compose and Material 3. Covers MaterialTheme DSL (ColorScheme, Typography, Shapes), dynamic color, key M3 components, adaptive layouts with WindowSizeClass, and surface/elevation patterns.
---

# Android UI — Material 3 & Jetpack Compose

## Concept

| Pillar | What it controls |
|---|---|
| **ColorScheme** | All role-based colors (`primary`, `surface`, `error`, etc.) — light and dark |
| **Typography** | Text styles (`displayLarge` → `labelSmall`) with custom fonts |
| **Shapes** | Corner radii across 5 size buckets (`extraSmall` → `extraLarge`) |
| **Dynamic Color** | Wallpaper-derived palette on Android 12+ (`Build.VERSION_CODES.S`) |
| **WindowSizeClass** | Adaptive breakpoints — `Compact`, `Medium`, `Expanded` |

---

## Implementation Patterns

### Full MaterialTheme Setup

```kotlin
// ui/theme/Color.kt — brand seeds from Material Theme Builder
val BrandPrimary = Color(0xFF6750A4)
val BrandSecondary = Color(0xFF625B71)
val BrandTertiary = Color(0xFF7D5260)

val LightColorScheme = lightColorScheme(
    primary = BrandPrimary,
    secondary = BrandSecondary,
    tertiary = BrandTertiary,
    // override other roles as needed; unset roles get sensible M3 defaults
)

val DarkColorScheme = darkColorScheme(
    primary = Color(0xFFD0BCFF),
    secondary = Color(0xFFCCC2DC),
    tertiary = Color(0xFFEFB8C8),
)

// ui/theme/Type.kt
val AppTypography = Typography(
    // Override only the styles you need; others keep M3 defaults
    headlineLarge = TextStyle(
        fontFamily = FontFamily.Default,
        fontWeight = FontWeight.SemiBold,
        fontSize = 32.sp,
        lineHeight = 40.sp,
    ),
    bodyMedium = TextStyle(
        fontFamily = FontFamily.Default,
        fontWeight = FontWeight.Normal,
        fontSize = 14.sp,
        lineHeight = 20.sp,
    ),
)

// ui/theme/Shape.kt
val AppShapes = Shapes(
    extraSmall = RoundedCornerShape(4.dp),
    small = RoundedCornerShape(8.dp),
    medium = RoundedCornerShape(12.dp),
    large = RoundedCornerShape(16.dp),
    extraLarge = RoundedCornerShape(28.dp),
)

// ui/theme/AppTheme.kt
@Composable
fun AppTheme(
    darkTheme: Boolean = isSystemInDarkTheme(),
    dynamicColor: Boolean = true,   // disable to always use brand palette
    content: @Composable () -> Unit
) {
    val colorScheme = when {
        dynamicColor && Build.VERSION.SDK_INT >= Build.VERSION_CODES.S -> {
            val context = LocalContext.current
            if (darkTheme) dynamicDarkColorScheme(context)
            else dynamicLightColorScheme(context)
        }
        darkTheme -> DarkColorScheme
        else -> LightColorScheme
    }

    MaterialTheme(
        colorScheme = colorScheme,
        typography = AppTypography,
        shapes = AppShapes,
        content = content,
    )
}
```

### Custom Font Integration

```kotlin
// res/font/ — add font files (e.g., inter_regular.ttf, inter_bold.ttf)

// ui/theme/Type.kt
private val Inter = FontFamily(
    Font(R.font.inter_regular, FontWeight.Normal),
    Font(R.font.inter_medium, FontWeight.Medium),
    Font(R.font.inter_bold, FontWeight.Bold),
)

val AppTypography = Typography(
    bodyLarge = TextStyle(fontFamily = Inter, fontWeight = FontWeight.Normal, fontSize = 16.sp),
    labelSmall = TextStyle(fontFamily = Inter, fontWeight = FontWeight.Medium, fontSize = 11.sp),
    // override remaining styles as needed
)
```

### Reading Theme Tokens

```kotlin
// Access from any Composable via MaterialTheme.*
val primary = MaterialTheme.colorScheme.primary
val headlineMedium = MaterialTheme.typography.headlineMedium
val cardShape = MaterialTheme.shapes.medium

// contentColorFor() returns the on-color for a given container color
Surface(color = MaterialTheme.colorScheme.primaryContainer) {
    Text(
        text = "Hello",
        color = contentColorFor(MaterialTheme.colorScheme.primaryContainer) // onPrimaryContainer
    )
}
```

### Surface & Tonal Elevation

M3 surfaces change tint with elevation (not shadow depth). Use `Surface` or `Card`, not raw `Box + background`.

```kotlin
// Tonal elevation — adds primary color tint proportional to elevation
Surface(
    modifier = Modifier.fillMaxWidth(),
    tonalElevation = 2.dp,   // slight primary tint in light theme
    shadowElevation = 0.dp,  // physical shadow; usually 0 in M3
    shape = MaterialTheme.shapes.medium,
) {
    content()
}

// Card variants
ElevatedCard(modifier = Modifier.padding(16.dp)) { /* ... */ }
FilledCard(modifier = Modifier.padding(16.dp)) { /* ... */ }
OutlinedCard(modifier = Modifier.padding(16.dp)) { /* ... */ }
```

### Top App Bar

```kotlin
@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun MyScreen(onBack: () -> Unit) {
    val scrollBehavior = TopAppBarDefaults.pinnedScrollBehavior()

    Scaffold(
        modifier = Modifier.nestedScroll(scrollBehavior.nestedScrollConnection),
        topBar = {
            TopAppBar(
                title = { Text("Tasks") },
                navigationIcon = {
                    IconButton(onClick = onBack) {
                        Icon(Icons.AutoMirrored.Filled.ArrowBack, contentDescription = "Back")
                    }
                },
                actions = {
                    IconButton(onClick = { /* filter */ }) {
                        Icon(Icons.Default.FilterList, contentDescription = "Filter")
                    }
                },
                scrollBehavior = scrollBehavior,
            )
        }
    ) { innerPadding ->
        // Use innerPadding to avoid content hiding behind bars
        LazyColumn(contentPadding = innerPadding) { /* ... */ }
    }
}
```

Use `LargeTopAppBar` / `MediumTopAppBar` with `exitUntilCollapsedScrollBehavior()` for collapsible headers.

### Navigation Bar vs Navigation Rail (Adaptive)

```kotlin
@Composable
fun AdaptiveNavigation(
    windowSizeClass: WindowSizeClass,
    selectedIndex: Int,
    onSelect: (Int) -> Unit,
    content: @Composable () -> Unit
) {
    val useRail = windowSizeClass.widthSizeClass != WindowWidthSizeClass.Compact

    if (useRail) {
        Row {
            NavigationRail {
                navItems.forEachIndexed { index, item ->
                    NavigationRailItem(
                        selected = selectedIndex == index,
                        onClick = { onSelect(index) },
                        icon = { Icon(item.icon, contentDescription = item.label) },
                        label = { Text(item.label) },
                    )
                }
            }
            Box(modifier = Modifier.weight(1f)) { content() }
        }
    } else {
        Column {
            Box(modifier = Modifier.weight(1f)) { content() }
            NavigationBar {
                navItems.forEachIndexed { index, item ->
                    NavigationBarItem(
                        selected = selectedIndex == index,
                        onClick = { onSelect(index) },
                        icon = { Icon(item.icon, contentDescription = item.label) },
                        label = { Text(item.label) },
                    )
                }
            }
        }
    }
}
```

Obtain `WindowSizeClass` in your Activity:

```kotlin
// build.gradle.kts — included with compose-bom 2024+
// implementation("androidx.compose.material3:material3-window-size-class")

class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContent {
            val windowSizeClass = calculateWindowSizeClass(this)
            AppTheme {
                AdaptiveNavigation(windowSizeClass = windowSizeClass, ...) { /* ... */ }
            }
        }
    }
}
```

### Modal Bottom Sheet

```kotlin
@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun FilterSheet(
    onDismiss: () -> Unit,
    onApply: (FilterOptions) -> Unit
) {
    val sheetState = rememberModalBottomSheetState(skipPartiallyExpanded = true)

    ModalBottomSheet(
        onDismissRequest = onDismiss,
        sheetState = sheetState,
    ) {
        Column(
            modifier = Modifier
                .padding(horizontal = 24.dp)
                .padding(bottom = 32.dp)
        ) {
            Text("Filter", style = MaterialTheme.typography.titleLarge)
            Spacer(Modifier.height(16.dp))
            // filter content
            Button(onClick = { onApply(options) }, modifier = Modifier.fillMaxWidth()) {
                Text("Apply")
            }
        }
    }
}
```

### Button Variants

```kotlin
// Filled — primary action
Button(onClick = { }) { Text("Save") }

// Tonal — secondary action alongside a filled button
FilledTonalButton(onClick = { }) { Text("Save Draft") }

// Outlined — medium-emphasis
OutlinedButton(onClick = { }) { Text("Cancel") }

// Text — low-emphasis; typical in dialogs
TextButton(onClick = { }) { Text("Dismiss") }

// Elevated — use sparingly when surface needs to stand out
ElevatedButton(onClick = { }) { Text("Promote") }

// Icon buttons
IconButton(onClick = { }) { Icon(Icons.Default.Share, contentDescription = "Share") }
FilledIconButton(onClick = { }) { Icon(Icons.Default.Add, contentDescription = "Add") }
```

### Text Fields

```kotlin
// Filled (default) — use inside forms on surface backgrounds
var value by remember { mutableStateOf("") }
TextField(
    value = value,
    onValueChange = { value = it },
    label = { Text("Title") },
    supportingText = { if (value.isBlank()) Text("Required") },
    isError = value.isBlank(),
    singleLine = true,
    keyboardOptions = KeyboardOptions(imeAction = ImeAction.Next),
    modifier = Modifier.fillMaxWidth(),
)

// Outlined — use when the field needs to stand out against a non-surface background
OutlinedTextField(
    value = value,
    onValueChange = { value = it },
    label = { Text("Search") },
    leadingIcon = { Icon(Icons.Default.Search, contentDescription = null) },
    trailingIcon = {
        if (value.isNotEmpty()) {
            IconButton(onClick = { value = "" }) {
                Icon(Icons.Default.Clear, contentDescription = "Clear")
            }
        }
    },
    modifier = Modifier.fillMaxWidth(),
)
```

---

## Guardrails

### DO
- Wrap your app in `AppTheme` in `MainActivity` — never call `MaterialTheme(...)` directly outside a theme composable.
- Use `MaterialTheme.colorScheme.*` tokens everywhere — no hardcoded `Color(0xFF...)` in composables.
- Provide both `LightColorScheme` and `DarkColorScheme`; test both in `@Preview(uiMode = Configuration.UI_MODE_NIGHT_YES)`.
- Use `contentColorFor(containerColor)` to derive text/icon color on any colored surface.
- Use `Scaffold` for screens with a `TopAppBar`, `BottomBar`, or `FloatingActionButton` — it wires up `innerPadding` and `WindowInsets` correctly.
- Generate brand color palettes with [Material Theme Builder](https://m3.material.io/theme-builder) to ensure accessible contrast ratios.
- Use `skipPartiallyExpanded = true` on `ModalBottomSheet` unless you explicitly want a half-expanded state.

### DON'T
- Don't hardcode colors (`Color.Blue`, `Color.White`) in composables — always use color scheme roles.
- Don't use `shadowElevation` as the primary depth signal in M3 — use `tonalElevation` for the surface tint effect.
- Don't pass `NavController` into composables — pass lambda callbacks; `NavController` in a composable makes it untestable.
- Don't use `NavigationBar` on `Expanded` windows — switch to `NavigationRail` or `NavigationDrawer`.
- Don't use `mutableStateOf` in a `ViewModel` — use `MutableStateFlow` and collect with `collectAsStateWithLifecycle()` in Compose.
- Don't ignore `innerPadding` from `Scaffold` — passing it to `contentPadding` (LazyColumn) or `padding` (Column) prevents content hiding behind system bars.

---

## References
- [Material 3 for Android](https://m3.material.io/develop/android/jetpack-compose)
- [Material Theme Builder](https://m3.material.io/theme-builder)
- [ColorScheme reference](https://developer.android.com/reference/kotlin/androidx/compose/material3/ColorScheme)
- [Typography reference](https://developer.android.com/reference/kotlin/androidx/compose/material3/Typography)
- [WindowSizeClass](https://developer.android.com/guide/topics/large-screens/support-different-screen-sizes)
- [Compose Material 3 components](https://developer.android.com/jetpack/compose/designsystems/material3)
