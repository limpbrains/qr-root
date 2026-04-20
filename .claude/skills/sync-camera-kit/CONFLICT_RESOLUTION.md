# CONFLICT_RESOLUTION.md

Guide for resolving merge conflicts during sync.

## General Approach

When `git merge upstream/master` produces conflicts:

1. Use `patches/no-google.patch` as reference for our customizations
2. Keep our QR-only implementation, accept upstream improvements elsewhere
3. Test after resolution

## File-by-File Conflict Resolution

### android/build.gradle

**Our version** (preserve):
```gradle
implementation 'com.github.limpbrains:qr:v0.0.2'

repositories {
    maven { url 'https://jitpack.io' }
}
```

**Their version** (reject):
```gradle
implementation 'com.google.mlkit:barcode-scanning:...'
```

**Accept from upstream**: CameraX version bumps, Kotlin version bumps, other shared dependencies.

---

### android/.../QRCodeAnalyzer.kt

**Our version** (preserve entirely):
```kotlin
import qr.QRDecoder

class QRCodeAnalyzer(private val onQRCodeDetected: (String) -> Unit) {
    override fun analyze(image: ImageProxy) {
        val decoded = QRDecoder.decode(width, height, grayscaleData)
        onQRCodeDetected(decoded)
    }
}
```

**Their version** (reject):
```kotlin
import com.google.mlkit.vision.barcode.BarcodeScanning

class QRCodeAnalyzer(private val onQRCodeDetected: (List<Barcode>) -> Unit) {
    // ML Kit implementation
}
```

**Resolution**: Keep our entire implementation. Accept only generic improvements like better error handling or image lifecycle management if they're separable.

---

### android/.../CKCamera.kt

**Our callback** (preserve):
```kotlin
private fun onBarcodeRead(qrCode: String) {
    // Single string callback
}
```

**Their callback** (reject):
```kotlin
private fun onBarcodeRead(barcodes: List<Barcode>, size: Size) {
    // Multi-barcode callback with filtering
}
```

**Accept from upstream**: Camera lifecycle improvements, focus/zoom features, error handling - anything not related to barcode callback.

---

### android/.../CodeFormat.kt

**Our version** (preserve structure):
```kotlin
enum class CodeFormat(val value: String) {
    QR_CODE("qr");

    companion object {
        fun fromString(value: String): CodeFormat? = values().find { it.value == value }
    }
}
```

**Their version** (reject ML Kit methods):
```kotlin
companion object {
    fun fromBarcodeType(type: Int): CodeFormat = when (type) {
        Barcode.FORMAT_QR_CODE -> QR_CODE
        // ...
    }
}
```

**Accept from upstream**: New enum values (harmless even if unused).

---

### package.json

**Preserve our fields**:
- `name`
- `version`
- `repository`

**Accept their fields**:
- `scripts`
- `devDependencies`
- `peerDependencies`

---

## Using the Patch

View our customizations during conflict resolution:
```bash
# Full patch
cat ../patches/no-google.patch

# Specific file
grep -A 100 "diff --git a/android/build.gradle" ../patches/no-google.patch
```

## Validation After Resolution

```bash
# Must pass
yarn build && yarn lint && yarn test

# Must be empty (no ML Kit)
grep -r "google.mlkit" android/

# Must find our dependency
grep "limpbrains/qr" android/build.gradle

# Must find our decoder
grep "QRDecoder.decode" android/src/main/java/com/rncamerakit/QRCodeAnalyzer.kt
```

## If Resolution Fails

```bash
# Abort and retry
git merge --abort

# Or reset to before merge
git reset --hard HEAD
```
