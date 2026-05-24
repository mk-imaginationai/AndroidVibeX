---
name: android-camera
description: Use when adding camera functionality. Covers CameraX lifecycle binding, Compose CameraXViewfinder, image capture, tap-to-focus, and camera switching. Never use legacy Camera1 or raw Camera2.
---

# Android Camera (CameraX)

## Concept

| Component | Role |
|---|---|
| `ProcessCameraProvider` | Binds use cases to `LifecycleOwner` — no manual open/close |
| `Preview` | Live viewfinder feed |
| `ImageCapture` | Take still photos |
| `CameraXViewfinder` | Compose viewfinder (preferred over `PreviewView` in `AndroidView`) |

```toml
camerax = "1.5.0"   # 1.5.0+ for Compose extensions
[libraries]
camera-core      = { group = "androidx.camera", name = "camera-core",      version.ref = "camerax" }
camera-camera2   = { group = "androidx.camera", name = "camera-camera2",   version.ref = "camerax" }
camera-lifecycle = { group = "androidx.camera", name = "camera-lifecycle",  version.ref = "camerax" }
camera-view      = { group = "androidx.camera", name = "camera-view",       version.ref = "camerax" }
camera-compose   = { group = "androidx.camera", name = "camera-compose",    version.ref = "camerax" }
```

---

## Implementation Patterns

### Full Camera Screen

```kotlin
@Composable
fun CameraScreen(
    onImageCaptured: (Bitmap) -> Unit,
    onError: (String) -> Unit
) {
    val context = LocalContext.current
    val lifecycleOwner = LocalLifecycleOwner.current
    var surfaceRequest by remember { mutableStateOf<SurfaceRequest?>(null) }
    var imageCapture by remember { mutableStateOf<ImageCapture?>(null) }
    var cameraControl by remember { mutableStateOf<CameraControl?>(null) }
    var lensFacing by remember { mutableIntStateOf(CameraSelector.LENS_FACING_BACK) }
    val coordinateTransformer = remember { MutableCoordinateTransformer() }
    val cameraExecutor = remember { Executors.newSingleThreadExecutor() }

    LaunchedEffect(lensFacing) {
        val cameraProvider = ProcessCameraProvider.getInstance(context).await()

        val preview = Preview.Builder().build().apply {
            setSurfaceProvider { request -> surfaceRequest = request }
        }
        val capture = ImageCapture.Builder()
            .setCaptureMode(ImageCapture.CAPTURE_MODE_MINIMIZE_LATENCY)
            .build().also { imageCapture = it }

        cameraProvider.unbindAll()
        val camera = cameraProvider.bindToLifecycle(
            lifecycleOwner,
            CameraSelector.Builder().requireLensFacing(lensFacing).build(),
            preview,
            capture
        )
        cameraControl = camera.cameraControl
    }

    DisposableEffect(Unit) { onDispose { cameraExecutor.shutdown() } }

    Box(Modifier.fillMaxSize()) {
        surfaceRequest?.let { request ->
            CameraXViewfinder(
                surfaceRequest = request,
                coordinateTransformer = coordinateTransformer,
                modifier = Modifier
                    .fillMaxSize()
                    .pointerInput(cameraControl) {
                        detectTapGestures { offset ->
                            val surfaceCoords = with(coordinateTransformer) { offset.transform() }
                            val factory = SurfaceOrientedMeteringPointFactory(
                                request.resolution.width.toFloat(),
                                request.resolution.height.toFloat()
                            )
                            val point = factory.createPoint(surfaceCoords.x, surfaceCoords.y)
                            cameraControl?.startFocusAndMetering(
                                FocusMeteringAction.Builder(point, FocusMeteringAction.FLAG_AF).build()
                            )
                        }
                    }
            )
        }

        Row(
            modifier = Modifier.align(Alignment.BottomCenter).padding(24.dp),
            horizontalArrangement = Arrangement.spacedBy(32.dp)
        ) {
            IconButton(onClick = {
                lensFacing = if (lensFacing == CameraSelector.LENS_FACING_BACK)
                    CameraSelector.LENS_FACING_FRONT else CameraSelector.LENS_FACING_BACK
            }) { Icon(Icons.Default.FlipCameraAndroid, "Flip camera") }

            IconButton(onClick = {
                imageCapture?.takePicture(cameraExecutor,
                    object : ImageCapture.OnImageCapturedCallback() {
                        override fun onCaptureSuccess(image: ImageProxy) {
                            val matrix = Matrix().apply {
                                postRotate(image.imageInfo.rotationDegrees.toFloat())
                                if (lensFacing == CameraSelector.LENS_FACING_FRONT)
                                    postScale(-1f, 1f)
                            }
                            val rotated = Bitmap.createBitmap(
                                image.toBitmap(), 0, 0,
                                image.width, image.height, matrix, true
                            )
                            image.close()  // MUST close — locks pipeline if forgotten
                            onImageCaptured(rotated)
                        }
                        override fun onError(e: ImageCaptureException) {
                            onError(e.message ?: "Capture failed")
                        }
                    })
            }) { Icon(Icons.Default.Camera, "Take photo") }
        }
    }
}
```

### Permission Handling

```kotlin
// AndroidManifest.xml
// <uses-permission android:name="android.permission.CAMERA" />
// <uses-feature android:name="android.hardware.camera" android:required="false" />

val permissionState = rememberPermissionState(Manifest.permission.CAMERA)
LaunchedEffect(Unit) {
    if (!permissionState.status.isGranted) permissionState.launchPermissionRequest()
}
when {
    permissionState.status.isGranted -> CameraScreen(...)
    permissionState.status.shouldShowRationale -> CameraRationaleScreen(...)
    else -> CameraPermissionDeniedScreen()
}
```

---

## Guardrails

### DO
- Bind camera to `LifecycleOwner` via `ProcessCameraProvider` — handles open/close automatically.
- Use `CameraXViewfinder` in Compose — not `PreviewView` wrapped in `AndroidView`.
- Always call `image.close()` in `OnImageCapturedCallback` — forgetting locks the capture pipeline.
- Use `MeteringPointFactory` for tap-to-focus — never calculate coordinates manually.

### DON'T
- Don't use Camera1 (`android.hardware.Camera`) or raw Camera2 for new code.
- Don't wrap `PreviewView` in `AndroidView` for Compose — causes resizing bugs.
- Don't manage camera lifecycle in `onResume`/`onPause`.
- Don't mark `android.hardware.camera` as `required="true"` unless camera is mandatory.

---

## References
- [CameraX overview](https://developer.android.com/media/camera/camerax)
- [CameraXViewfinder](https://developer.android.com/reference/kotlin/androidx/camera/compose/package-summary)
- [ImageCapture](https://developer.android.com/reference/kotlin/androidx/camera/core/ImageCapture)
