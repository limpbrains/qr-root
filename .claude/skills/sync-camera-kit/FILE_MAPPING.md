# FILE_MAPPING.md

Comprehensive file-by-file sync policy for react-native-camera-kit. ALL files are evaluated for selective sync - no files are completely excluded.

## Classification System

**Category A: Auto-Sync** - No fork divergence, apply entire file
**Category B: Selective Sync** - Fork divergence, hunk-level analysis required

## Category A: Auto-Sync Files

These files have no fork-specific modifications. Apply entire file changes directly.

### iOS Layer (All Swift Files)
```
/ios/**/*.swift
/ios/**/*.h
/ios/**/*.m
/ios/**/*.xcconfig
```

**Rationale**: iOS implementation unchanged in fork. Apple's AVFoundation is used in both upstream and fork.

### TypeScript/React Native Layer
```
/src/**/*.ts
/src/**/*.tsx
/src/__tests__/**/*
```

**Rationale**: JavaScript layer unchanged. React Native API is identical for both upstream and fork.

**Exception**: If upstream adds Android-specific barcode filtering props (e.g., `allowedBarcodeTypes`), sync the prop definition but document "Android: QR-only (fork limitation)" in comments.

### Build Configuration
```
/.eslintrc.js
/tsconfig.json
/jest.config.js
/.prettierrc
/.gitignore
/.npmignore
```

**Rationale**: Build tools unchanged in fork.

### Example App
```
/example/**/*
```

**Rationale**: Example app uses public API which is identical. If example uses barcode filtering, sync it but note QR-only limitation.

### Android Non-Barcode Files
```
/android/src/main/java/com/rncamerakit/events/**/*
/android/src/main/java/com/rncamerakit/CKCameraManager.kt
/android/src/main/res/**/*
/android/src/main/AndroidManifest.xml
```

**Rationale**: These files don't touch barcode scanning logic.

## Category B: Selective Sync Files

These files have fork-specific modifications. Require hunk-level analysis.

---

### B1: CKCamera.kt

**Path**: `/android/src/main/java/com/rncamerakit/CKCamera.kt`

**Fork Divergence**:
- Upstream: `onBarcodeRead(List<Barcode>, Size)` callback
- Fork: `onBarcodeRead(String)` callback (single QR code only)
- Upstream has barcode filtering by `allowedBarcodeTypes`
- Fork has no filtering (always QR)

**Selective Sync Strategy**:

**APPLY hunks that:**
- Improve error handling (`try/catch`, `onError` calls)
- Fix camera lifecycle issues
- Add focus/zoom/flash features
- Improve CameraX usage patterns
- Fix memory leaks

**SKIP hunks that:**
- Mention `onBarcodeRead(` with multiple parameters
- Use `List<Barcode>` type
- Reference `allowedBarcodeTypes` property
- Call `CodeFormat.fromBarcodeType()`
- Filter barcodes by type

**Detection Keywords**:
```
Skip: allowedBarcodeTypes, List<Barcode>, Barcode.getBoundingBox()
Apply: cameraProviderFuture, onError, image.close(), focus, zoom
```

**See**: `CONFLICT_RESOLUTION.md` for detailed examples.

---

### B2: QRCodeAnalyzer.kt

**Path**: `/android/src/main/java/com/rncamerakit/QRCodeAnalyzer.kt`

**Fork Divergence**:
- Upstream: Uses Google ML Kit `BarcodeScanner`
- Fork: Uses limpbrains/qr `QRDecoder`
- Completely different API surface

**Selective Sync Strategy**:

**APPLY hunks that:**
- Improve `ImageProxy` lifecycle management
- Optimize threading/coroutines
- Add error handling patterns
- Improve Y-plane extraction logic (if generic)

**SKIP hunks that:**
- Import `com.google.mlkit.*`
- Use `BarcodeScanning.getClient()`
- Reference `InputImage` class
- Call `.process()` on ML Kit scanner
- Use `Barcode` type from ML Kit

**Detection Keywords**:
```
Skip: BarcodeScanning, com.google.mlkit, InputImage, Barcode.FORMAT_
Apply: ImageProxy, image.close(), threading, error handling
```

**Critical Preservation**: Always preserve `QRDecoder.decode()` call and `extractYPlane()` method.

**See**: `CONFLICT_RESOLUTION.md` for detailed examples.

---

### B3: CodeFormat.kt

**Path**: `/android/src/main/java/com/rncamerakit/CodeFormat.kt`

**Fork Divergence**:
- Upstream: 64-line enum with ML Kit conversion methods
- Fork: 17-line simplified enum (string values only)

**Selective Sync Strategy**:

**APPLY hunks that:**
- Add new barcode format enum values
- Update enum string mappings
- Add documentation comments

**SKIP hunks that:**
- Add `fromBarcodeType(type: Int)` method
- Add `toBarcodeType()` method
- Import `com.google.mlkit.vision.barcode.common.Barcode`
- Reference ML Kit constants

**Detection Keywords**:
```
Skip: Barcode.FORMAT_, fromBarcodeType, toBarcodeType, com.google.mlkit
Apply: New enum entries, string values, documentation
```

**Example APPLY**:
```kotlin
DATA_MATRIX("datamatrix"),
AZTEC("aztec")
```

**Example SKIP**:
```kotlin
fun fromBarcodeType(type: Int): CodeFormat = when (type) { ... }
```

**See**: `CONFLICT_RESOLUTION.md` for detailed examples.

---

### B4: build.gradle

**Path**: `/android/build.gradle`

**Fork Divergence**:
- Upstream: Uses Google ML Kit dependency
- Fork: Uses limpbrains/qr dependency from JitPack

**Selective Sync Strategy**:

**APPLY dependency updates for:**
- `androidx.camera:camera-*` (CameraX libraries)
- `org.jetbrains.kotlin:kotlin-*` (Kotlin version)
- `androidx.core:core-ktx` (AndroidX Core)
- Shared library version bumps

**SKIP dependency changes for:**
- `com.google.mlkit:barcode-scanning`
- `com.google.android.gms:play-services-*`
- Google Maven repository additions

**PRESERVE fork-specific:**
```gradle
implementation 'com.github.limpbrains:qr:v0.0.1'

repositories {
    maven { url 'https://jitpack.io' }
}
```

**Detection Keywords**:
```
Skip: google.mlkit, play-services-mlkit-barcode
Apply: androidx.camera, kotlin-stdlib, camera-core, camera-lifecycle
Preserve: limpbrains/qr, jitpack.io
```

**See**: `CONFLICT_RESOLUTION.md` for detailed examples.

---

### B5: package.json

**Path**: `/package.json`

**Fork Divergence**:
- Upstream: `name: "react-native-camera-kit"`
- Fork: `name: "@limpbrains/react-native-camera-kit-no-google"`
- Different repository URLs
- Decoupled versioning

**Selective Sync Strategy**:

**APPLY changes to:**
- `scripts` section (build, test, lint commands)
- `devDependencies` (build tools, TypeScript, ESLint)
- `peerDependencies` (React Native version requirements)
- `files` array (published files)
- `engines` (Node version)

**PRESERVE fork-specific:**
- `name` field
- `version` field (fork has independent versioning)
- `repository.url` field
- `author` field
- `homepage` field

**Detection Keywords**:
```
Skip: name, version, repository.url, author, homepage
Apply: scripts.*, devDependencies.*, peerDependencies.*, files, engines
```

**Special Handling**: If upstream adds new dependencies that enable barcode filtering UI, sync them but document QR-only limitation.

**See**: `CONFLICT_RESOLUTION.md` for detailed examples.

---

### B6: README.md

**Path**: `/README.md`

**Fork Divergence**:
- Fork has notice about QR-only Android limitation
- Fork has comparison table (upstream vs fork)
- Fork has attribution section

**Selective Sync Strategy**:

**APPLY changes to:**
- Installation instructions
- API documentation (props, methods)
- Usage examples
- iOS-specific sections
- TypeScript type definitions

**PRESERVE fork-specific:**
- Lines 1-19: Fork notice and badges
- Lines 20-44: Comparison table and attribution
- Android barcode limitation notes

**SKIP changes to:**
- Android barcode filtering examples
- Multi-format barcode documentation

**Detection Keywords**:
```
Skip: allowedBarcodeTypes examples, multi-format scanning
Apply: API props, iOS changes, TypeScript updates
Preserve: Fork notice, comparison table, attribution
```

**Recommendation**: Section-based merge. Extract API updates, preserve fork context.

**See**: `CONFLICT_RESOLUTION.md` for detailed examples.

---

### B7: CLAUDE.md

**Path**: `/CLAUDE.md` (fork only)

**Fork Divergence**:
- Upstream: No CLAUDE.md file
- Fork: Architecture documentation + sync state

**Selective Sync Strategy**:

**NEVER sync**: This file doesn't exist in upstream.

**UPDATE sections**:
- `## Camera Kit Sync State` - Update after each sync
- Add new commits synced
- Document selective sync decisions

**PRESERVE sections**:
- `## Architecture` - Fork-specific documentation
- `## Key Differences` - Fork vs upstream comparison
- All other content

**See**: SKILL.md Step 6 for sync state format.

---

### B8: ReactNativeCameraKit.podspec

**Path**: `/ReactNativeCameraKit.podspec`

**Fork Divergence**:
- Fork has different repository URL
- Fork version may be decoupled

**Selective Sync Strategy**:

**APPLY changes to:**
- `s.dependency` lines (iOS dependencies)
- `s.ios.deployment_target`
- `s.frameworks` array
- `s.source_files` glob patterns

**PRESERVE fork-specific:**
- `s.version` (if decoupled from upstream)
- `s.homepage` URL
- `s.source.git` URL
- `s.author` information

**Detection Keywords**:
```
Skip: s.homepage, s.source.git, s.author
Apply: s.dependency, s.ios.deployment_target, s.frameworks
```

---

## Change Detection Patterns

When analyzing a file diff, classify hunks by these patterns:

### Pattern 1: Safe to Apply
- Error handling (`try/catch`, `throw`)
- Logging improvements
- Null safety checks
- Memory leak fixes
- Performance optimizations (generic)
- Documentation/comments

### Pattern 2: Needs Review (Likely Skip)
- ML Kit imports
- `Barcode` type usage
- `allowedBarcodeTypes` references
- Multi-format barcode logic
- Google service dependencies

### Pattern 3: Context-Dependent
- Camera lifecycle changes (review if barcode-related)
- Event emission (review callback signature)
- Props/state (review if barcode filtering)
- Enum additions (apply values, skip conversions)
- Dependency updates (apply shared, skip Google)

## Validation After Sync

After applying changes, validate fork integrity:

```bash
# No Google ML Kit leaked in
grep -r "google.mlkit" android/

# Fork dependency intact
grep "limpbrains/qr" android/build.gradle

# QRDecoder preserved
grep "QRDecoder.decode" android/src/main/java/com/rncamerakit/QRCodeAnalyzer.kt

# Fork metadata preserved
grep "limpbrains" package.json

# Build passes
yarn build && yarn lint && yarn test
```

## When in Doubt

If unsure whether to apply a hunk:
1. Check if it mentions ML Kit or multi-format scanning → SKIP
2. Check if it's generic improvement (error handling, etc.) → APPLY
3. If still unsure → Ask user for decision

**Philosophy**: When in doubt, preserve fork architecture. It's easier to add a missed improvement later than to fix a broken fork.
