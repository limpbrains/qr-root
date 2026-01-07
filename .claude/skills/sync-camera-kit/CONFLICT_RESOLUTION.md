# CONFLICT_RESOLUTION.md

Detailed hunk-by-hunk strategies for resolving conflicts during selective sync. Each file with fork divergence has specific extraction rules.

## Overview

This document provides concrete examples of what to apply vs skip for each Category B file. Use these strategies during Step 5 (Apply Changes Selectively) of the sync workflow.

---

## CKCamera.kt - Selective Sync Strategy

**File**: `/android/src/main/java/com/rncamerakit/CKCamera.kt`

**Complexity**: High - Mixes barcode and non-barcode logic

### Fork Signature Difference

**Upstream**:
```kotlin
private fun onBarcodeRead(barcodes: List<Barcode>, size: Size) {
    val filteredBarcodes = barcodes.filter { barcode ->
        allowedBarcodeTypes.contains(CodeFormat.fromBarcodeType(barcode.format))
    }
    // Emit event with filtered barcodes
}
```

**Fork**:
```kotlin
private fun onBarcodeRead(qrCode: String) {
    // Emit event with single QR code string
}
```

### Hunk Classification Rules

#### ✅ APPLY: Error Handling Improvements

**Example**:
```kotlin
try {
    val cameraProvider = cameraProviderFuture.get()
    bindCamera(cameraProvider)
} catch (e: ExecutionException) {
    onError(CameraErrorEvent(CameraError.CAMERA_INIT_FAILED, e.message))
} catch (e: InterruptedException) {
    onError(CameraErrorEvent(CameraError.CAMERA_INIT_FAILED, e.message))
}
```

**Rationale**: Generic error handling, no barcode logic involved.

---

#### ✅ APPLY: Camera Lifecycle Improvements

**Example**:
```kotlin
override fun onPause(owner: LifecycleOwner) {
    super.onPause(owner)
    cameraProvider?.unbindAll()
}
```

**Rationale**: Camera lifecycle, not barcode-specific.

---

#### ✅ APPLY: Focus/Zoom/Flash Features

**Example**:
```kotlin
fun setZoom(zoomRatio: Float) {
    camera?.cameraControl?.setLinearZoom(zoomRatio)
}

fun setFocusPoint(x: Float, y: Float) {
    val factory = SurfaceOrientedMeteringPointFactory(width.toFloat(), height.toFloat())
    val point = factory.createPoint(x, y)
    val action = FocusMeteringAction.Builder(point).build()
    camera?.cameraControl?.startFocusAndMetering(action)
}
```

**Rationale**: Camera control features, not barcode-specific.

---

#### ❌ SKIP: Barcode Filtering Logic

**Example**:
```kotlin
private fun filterBarcodes(barcodes: List<Barcode>): List<Barcode> {
    return barcodes.filter { barcode ->
        allowedBarcodeTypes.contains(CodeFormat.fromBarcodeType(barcode.format))
    }
}
```

**Rationale**: Relies on `List<Barcode>` and `allowedBarcodeTypes` which don't exist in fork.

---

#### ❌ SKIP: Barcode Callback Signature Changes

**Example**:
```kotlin
// Upstream change
private fun onBarcodeRead(barcodes: List<Barcode>, imageSize: Size) {
    val boundingBox = barcodes.firstOrNull()?.boundingBox
    // ...
}
```

**Rationale**: Fork uses `onBarcodeRead(qrCode: String)` - incompatible signature.

---

#### ❌ SKIP: allowedBarcodeTypes Property

**Example**:
```kotlin
private var allowedBarcodeTypes: Set<CodeFormat> = setOf(CodeFormat.QR_CODE)

fun setAllowedBarcodeTypes(types: List<String>) {
    allowedBarcodeTypes = types.mapNotNull { CodeFormat.fromString(it) }.toSet()
}
```

**Rationale**: Fork doesn't support barcode filtering (QR-only).

---

### Detection Keywords Summary

| Action | Keywords |
|--------|----------|
| **APPLY** | `cameraProviderFuture`, `onError`, `CameraErrorEvent`, `camera.cameraControl`, `setZoom`, `setFocus`, `LifecycleOwner`, `onPause`, `onResume` |
| **SKIP** | `onBarcodeRead(List<Barcode>`, `allowedBarcodeTypes`, `filterBarcodes`, `Barcode.getBoundingBox()`, `CodeFormat.fromBarcodeType` |

---

## QRCodeAnalyzer.kt - Selective Sync Strategy

**File**: `/android/src/main/java/com/rncamerakit/QRCodeAnalyzer.kt`

**Complexity**: Very High - Completely different implementation

### Fork Implementation Difference

**Upstream** (Google ML Kit):
```kotlin
import com.google.mlkit.vision.barcode.BarcodeScanning
import com.google.mlkit.vision.common.InputImage

class QRCodeAnalyzer(private val onQRCodeDetected: (List<Barcode>) -> Unit) : ImageAnalysis.Analyzer {
    private val scanner = BarcodeScanning.getClient()

    override fun analyze(image: ImageProxy) {
        val inputImage = InputImage.fromMediaImage(image.image!!, image.imageInfo.rotationDegrees)
        scanner.process(inputImage)
            .addOnSuccessListener { barcodes ->
                onQRCodeDetected(barcodes)
            }
            .addOnCompleteListener {
                image.close()
            }
    }
}
```

**Fork** (limpbrains/qr):
```kotlin
import qr.QRDecoder
import qr.QRDecodingException

class QRCodeAnalyzer(private val onQRCodeDetected: (String) -> Unit) : ImageAnalysis.Analyzer {

    override fun analyze(image: ImageProxy) {
        try {
            val grayscaleData = extractYPlane(image)
            val decoded = QRDecoder.decode(image.width, image.height, grayscaleData)
            onQRCodeDetected(decoded)
        } catch (e: QRDecodingException) {
            // No QR code found
        } finally {
            image.close()
        }
    }

    private fun extractYPlane(image: ImageProxy): ByteArray {
        // Y-plane extraction logic
    }
}
```

### Hunk Classification Rules

#### ✅ APPLY: Image Lifecycle Improvements

**Example**:
```kotlin
override fun analyze(image: ImageProxy) {
    try {
        // ... decoding logic ...
    } finally {
        image.close()  // ← Ensure image always closed
    }
}
```

**Rationale**: Generic ImageProxy lifecycle improvement, API-agnostic.

---

#### ✅ APPLY: Threading Optimizations

**Example**:
```kotlin
private val executor = Executors.newSingleThreadExecutor()

override fun analyze(image: ImageProxy) {
    executor.execute {
        try {
            // ... decoding logic ...
        } finally {
            image.close()
        }
    }
}
```

**Rationale**: Threading pattern is library-agnostic, applies to both ML Kit and QRDecoder.

---

#### ✅ APPLY: Error Handling Patterns

**Example**:
```kotlin
override fun analyze(image: ImageProxy) {
    try {
        // ... decoding logic ...
    } catch (e: Exception) {
        Log.e(TAG, "QR decoding failed", e)
    } finally {
        image.close()
    }
}
```

**Rationale**: Generic error handling, not ML Kit-specific.

---

#### ❌ SKIP: ML Kit API Usage

**Example**:
```kotlin
import com.google.mlkit.vision.barcode.BarcodeScanning
import com.google.mlkit.vision.barcode.common.Barcode

private val scanner = BarcodeScanning.getClient(
    BarcodeScannerOptions.Builder()
        .setBarcodeFormats(Barcode.FORMAT_QR_CODE)
        .build()
)
```

**Rationale**: Fork doesn't use ML Kit, uses QRDecoder instead.

---

#### ❌ SKIP: InputImage Conversion

**Example**:
```kotlin
val inputImage = InputImage.fromMediaImage(
    image.image!!,
    image.imageInfo.rotationDegrees
)
scanner.process(inputImage)
```

**Rationale**: Fork extracts Y-plane directly, doesn't use InputImage.

---

#### ❌ SKIP: Barcode List Processing

**Example**:
```kotlin
scanner.process(inputImage)
    .addOnSuccessListener { barcodes ->
        val qrCodes = barcodes.filter { it.format == Barcode.FORMAT_QR_CODE }
        onQRCodeDetected(qrCodes)
    }
```

**Rationale**: Fork callback is `(String) -> Unit`, not `(List<Barcode>) -> Unit`.

---

#### ⚠️ REVIEW: Y-Plane Extraction

**Example**:
```kotlin
private fun extractYPlane(image: ImageProxy): ByteArray {
    val yPlane = image.planes[0]
    val yBuffer = yPlane.buffer

    // NEW: Handle rowStride padding more efficiently
    if (yPlane.rowStride == image.width) {
        val data = ByteArray(yBuffer.remaining())
        yBuffer.get(data)
        return data
    }

    // Existing padding handling...
}
```

**Decision**: APPLY if optimization is generic (better rowStride handling). SKIP if specific to InputImage format conversion.

---

### Detection Keywords Summary

| Action | Keywords |
|--------|----------|
| **APPLY** | `image.close()`, `executor`, `try/catch`, `Log.e`, `finally`, generic `ImageProxy` improvements |
| **SKIP** | `BarcodeScanning`, `InputImage`, `com.google.mlkit`, `Barcode.FORMAT_`, `.addOnSuccessListener`, `scanner.process` |
| **PRESERVE** | `QRDecoder.decode`, `extractYPlane`, `QRDecodingException`, `onQRCodeDetected(String)` |

---

## CodeFormat.kt - Selective Sync Strategy

**File**: `/android/src/main/java/com/rncamerakit/CodeFormat.kt`

**Complexity**: Medium - Simple enum vs full converter

### Fork Structure Difference

**Upstream** (64 lines):
```kotlin
enum class CodeFormat(val value: String) {
    QR_CODE("qr"),
    EAN_8("ean8"),
    EAN_13("ean13"),
    // ... many more formats

    companion object {
        fun fromBarcodeType(type: Int): CodeFormat = when (type) {
            Barcode.FORMAT_QR_CODE -> QR_CODE
            Barcode.FORMAT_EAN_8 -> EAN_8
            // ...
        }

        fun toBarcodeType(format: CodeFormat): Int = when (format) {
            QR_CODE -> Barcode.FORMAT_QR_CODE
            EAN_8 -> Barcode.FORMAT_EAN_8
            // ...
        }
    }
}
```

**Fork** (17 lines):
```kotlin
enum class CodeFormat(val value: String) {
    QR_CODE("qr");

    companion object {
        fun fromString(value: String): CodeFormat? = values().find { it.value == value }
    }
}
```

### Hunk Classification Rules

#### ✅ APPLY: New Enum Values

**Example**:
```kotlin
enum class CodeFormat(val value: String) {
    QR_CODE("qr"),
    DATA_MATRIX("datamatrix"),  // ← NEW
    AZTEC("aztec"),             // ← NEW
}
```

**Rationale**: Adding enum values is harmless, even if fork doesn't use them. Future-proofs for potential multi-format support.

---

#### ✅ APPLY: Documentation

**Example**:
```kotlin
/**
 * Supported barcode formats.
 * Note: Android fork (limpbrains/qr) only supports QR_CODE.
 */
enum class CodeFormat(val value: String) {
    QR_CODE("qr"),
}
```

**Rationale**: Documentation improvements are always safe.

---

#### ❌ SKIP: ML Kit Conversion Methods

**Example**:
```kotlin
companion object {
    fun fromBarcodeType(type: Int): CodeFormat = when (type) {
        Barcode.FORMAT_QR_CODE -> QR_CODE
        Barcode.FORMAT_DATA_MATRIX -> DATA_MATRIX
        else -> throw IllegalArgumentException("Unsupported barcode type: $type")
    }
}
```

**Rationale**: Fork doesn't use ML Kit `Barcode` type, doesn't need conversions.

---

#### ❌ SKIP: ML Kit Imports

**Example**:
```kotlin
import com.google.mlkit.vision.barcode.common.Barcode
```

**Rationale**: Fork has no ML Kit dependency.

---

#### ✅ APPLY: String Conversion Helper

**Example**:
```kotlin
companion object {
    fun fromString(value: String): CodeFormat? = values().find { it.value == value }

    // NEW: Case-insensitive lookup
    fun fromStringIgnoreCase(value: String): CodeFormat? =
        values().find { it.value.equals(value, ignoreCase = true) }
}
```

**Rationale**: String helpers don't depend on ML Kit, safe to add.

---

### Extraction Example

**Upstream Diff**:
```diff
 enum class CodeFormat(val value: String) {
     QR_CODE("qr"),
+    DATA_MATRIX("datamatrix"),
+    AZTEC("aztec"),

     companion object {
+        fun fromBarcodeType(type: Int): CodeFormat = when (type) {
+            Barcode.FORMAT_QR_CODE -> QR_CODE
+            Barcode.FORMAT_DATA_MATRIX -> DATA_MATRIX
+            Barcode.FORMAT_AZTEC -> AZTEC
+        }
     }
 }
```

**What to Apply**:
```kotlin
// ✅ Apply enum value additions
DATA_MATRIX("datamatrix"),
AZTEC("aztec"),
```

**What to Skip**:
```kotlin
// ❌ Skip ML Kit conversion method
fun fromBarcodeType(type: Int): CodeFormat = ...
```

**Result in Fork**:
```kotlin
enum class CodeFormat(val value: String) {
    QR_CODE("qr"),
    DATA_MATRIX("datamatrix"),  // ← Applied
    AZTEC("aztec"),             // ← Applied

    companion object {
        fun fromString(value: String): CodeFormat? = values().find { it.value == value }
        // ← Conversion methods NOT added
    }
}
```

---

### Detection Keywords Summary

| Action | Keywords |
|--------|----------|
| **APPLY** | New enum entries, `value: String`, documentation, `fromString` helpers |
| **SKIP** | `fromBarcodeType`, `toBarcodeType`, `Barcode.FORMAT_`, `import com.google.mlkit` |

---

## build.gradle - Selective Sync Strategy

**File**: `/android/build.gradle`

**Complexity**: Medium - Dependency management

### Fork Dependency Difference

**Upstream**:
```gradle
dependencies {
    implementation 'com.google.mlkit:barcode-scanning:17.3.0'
    implementation "androidx.camera:camera-core:1.3.0"
    implementation "androidx.camera:camera-lifecycle:1.3.0"
}

repositories {
    google()
    mavenCentral()
}
```

**Fork**:
```gradle
dependencies {
    implementation 'com.github.limpbrains:qr:v0.0.1'
    implementation "androidx.camera:camera-core:1.3.0"
    implementation "androidx.camera:camera-lifecycle:1.3.0"
}

repositories {
    google()
    mavenCentral()
    maven { url 'https://jitpack.io' }
}
```

### Hunk Classification Rules

#### ✅ APPLY: Shared Dependency Updates

**Example**:
```diff
-implementation "androidx.camera:camera-core:1.3.0"
-implementation "androidx.camera:camera-lifecycle:1.3.0"
+implementation "androidx.camera:camera-core:1.4.0"
+implementation "androidx.camera:camera-lifecycle:1.4.0"
```

**Rationale**: CameraX is used by both upstream and fork, safe to update.

---

#### ✅ APPLY: Kotlin Version Bump

**Example**:
```diff
-ext.kotlin_version = '1.9.0'
+ext.kotlin_version = '1.9.10'
```

**Rationale**: Kotlin version affects both implementations.

---

#### ❌ SKIP: ML Kit Dependency

**Example**:
```diff
+implementation 'com.google.mlkit:barcode-scanning:17.3.0'
```

**Rationale**: Fork uses limpbrains/qr instead.

---

#### ❌ SKIP: Google Services

**Example**:
```diff
+implementation 'com.google.android.gms:play-services-mlkit-barcode-scanning:18.3.0'
```

**Rationale**: Fork doesn't use Google services.

---

#### ✅ PRESERVE: Fork-Specific Dependency

**Always keep**:
```gradle
implementation 'com.github.limpbrains:qr:v0.0.1'

repositories {
    maven { url 'https://jitpack.io' }
}
```

**Rationale**: Core fork dependency, never remove.

---

### Extraction Example

**Upstream Diff**:
```diff
 dependencies {
-    implementation "androidx.camera:camera-core:1.3.0"
+    implementation "androidx.camera:camera-core:1.4.0"
+    implementation 'com.google.mlkit:barcode-scanning:17.3.1'
 }
```

**What to Apply**:
```gradle
// ✅ CameraX version bump
implementation "androidx.camera:camera-core:1.4.0"
```

**What to Skip**:
```gradle
// ❌ ML Kit dependency
implementation 'com.google.mlkit:barcode-scanning:17.3.1'
```

**Result in Fork**:
```gradle
dependencies {
    implementation 'com.github.limpbrains:qr:v0.0.1'  // ← Preserved
    implementation "androidx.camera:camera-core:1.4.0" // ← Updated
}
```

---

### Detection Keywords Summary

| Action | Keywords |
|--------|----------|
| **APPLY** | `androidx.camera`, `kotlin-stdlib`, `androidx.core`, version bumps |
| **SKIP** | `google.mlkit`, `play-services-mlkit`, `barcode-scanning` |
| **PRESERVE** | `limpbrains/qr`, `jitpack.io` |

---

## package.json - Selective Sync Strategy

**File**: `/package.json`

**Complexity**: Low - Field-level extraction

### Fork Metadata Difference

**Upstream**:
```json
{
  "name": "react-native-camera-kit",
  "version": "16.2.0",
  "repository": {
    "url": "https://github.com/teslamotors/react-native-camera-kit"
  }
}
```

**Fork**:
```json
{
  "name": "@limpbrains/react-native-camera-kit-no-google",
  "version": "16.1.3",
  "repository": {
    "url": "https://github.com/limpbrains/react-native-camera-kit-no-google"
  }
}
```

### Field-Level Extraction Rules

#### ✅ APPLY: Scripts

```json
"scripts": {
  "build": "tsc",
  "lint": "eslint .",
  "test": "jest"
}
```

**Rationale**: Build scripts are fork-agnostic.

---

#### ✅ APPLY: devDependencies

```json
"devDependencies": {
  "typescript": "^5.3.0",
  "@typescript-eslint/eslint-plugin": "^6.0.0"
}
```

**Rationale**: Build tools are shared.

---

#### ✅ APPLY: peerDependencies

```json
"peerDependencies": {
  "react": "*",
  "react-native": ">=0.70.0"
}
```

**Rationale**: React Native version requirements apply to fork.

---

#### ❌ SKIP: name, version, repository

```json
{
  "name": "react-native-camera-kit",  // ← SKIP
  "version": "16.2.0",                 // ← SKIP
  "repository": { ... }                // ← SKIP
}
```

**Rationale**: Fork has independent name, version, and repository.

---

### Detection Keywords Summary

| Action | Keywords |
|--------|----------|
| **APPLY** | `scripts.*`, `devDependencies.*`, `peerDependencies.*`, `files`, `engines` |
| **SKIP** | `name`, `version`, `repository.url`, `author`, `homepage` |

---

## README.md - Selective Sync Strategy

**File**: `/README.md`

**Complexity**: Medium - Section-based merge

### Fork-Specific Sections

**Lines 1-19**: Fork badges and notice
**Lines 20-44**: Comparison table (upstream vs fork)
**Lines 45+**: API documentation (mostly shared)

### Section-Level Extraction Rules

#### ✅ APPLY: API Documentation

- Installation instructions
- Props documentation
- Method signatures
- iOS-specific sections
- TypeScript types

#### ❌ SKIP: Android Barcode Examples

- `allowedBarcodeTypes` prop examples
- Multi-format scanning code samples

#### ✅ PRESERVE: Fork Notice

```markdown
## ⚠️ Fork Notice

This is a fork of teslamotors/react-native-camera-kit that replaces Google ML Kit with limpbrains/qr for QR-only scanning on Android.

### Key Differences
...
```

**Rationale**: Essential fork context for users.

---

## General Conflict Resolution Process

### Step-by-Step Hunk Evaluation

1. **Extract diff**: `git diff <last-sync>..HEAD -- <file-path>`
2. **Split into hunks**: Each `@@ ... @@` block is a hunk
3. **For each hunk**:
   - Check skip keywords → SKIP if found
   - Check apply keywords → APPLY if found
   - If uncertain → Ask user
4. **Apply compatible hunks** using Edit tool
5. **Log decision** for each hunk in CLAUDE.md

### 3-Way Merge Template

For complex conflicts:

```
BASE (fork-point):  Original code
OURS (fork):        Fork-specific code
THEIRS (upstream):  Upstream change

DECISION: [APPLY/SKIP/MANUAL]
RATIONALE: [why]
```

### Uncertainty Protocol

If unsure about a hunk:
1. Err on side of preservation (SKIP)
2. Log as "Manual Review Required" in CLAUDE.md
3. Notify user in sync summary
4. User can manually apply later

---

## Success Validation

After all conflict resolution:

```bash
# Build must pass
yarn build

# Lint must pass
yarn lint

# Tests must pass
yarn test

# Fork integrity checks
grep "limpbrains/qr" android/build.gradle
grep "QRDecoder.decode" android/src/main/java/com/rncamerakit/QRCodeAnalyzer.kt
! grep -r "google.mlkit" android/
```

If any check fails, review selective sync decisions.
