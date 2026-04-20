# FILE_MAPPING.md

Files in the `patches/no-google.patch` and their purpose.

## Patch Contents

The patch captures all differences between `upstream/master` and our fork's `master`. These files may cause merge conflicts when syncing.

### Core Android Changes (QR-Only Implementation)

| File | Our Customization |
|------|-------------------|
| `android/build.gradle` | Uses `com.github.limpbrains:qr` instead of `google.mlkit` |
| `android/.../QRCodeAnalyzer.kt` | Uses `QRDecoder.decode()` instead of ML Kit scanner |
| `android/.../CKCamera.kt` | Callback `onQRCodeDetected(String)` instead of `List<Barcode>` |
| `android/.../CodeFormat.kt` | Simplified enum without ML Kit conversion methods |
| `android/.../CKCameraManager.kt` | Adapted for String callback |

### Metadata Files

| File | Our Customization |
|------|-------------------|
| `package.json` | Fork name, version, repository |
| `README.md` | Fork notice and comparison table |
| `CLAUDE.md` | Architecture docs and sync state |
| `ReactNativeCameraKit.podspec` | Fork repository URL |

### Files with Minor Differences

| File | Our Customization |
|------|-------------------|
| `.gitignore` | Fork-specific ignores |
| `example/**` | Minor adjustments |
| `ios/**` | May have sync state differences |
| `src/**` | May have sync state differences |

## Conflict Likelihood

**High conflict probability:**
- `android/build.gradle` - Different dependencies
- `android/.../QRCodeAnalyzer.kt` - Completely different implementation
- `android/.../CKCamera.kt` - Different callback signature

**Medium conflict probability:**
- `android/.../CodeFormat.kt` - Different enum structure
- `package.json` - Metadata differences

**Low conflict probability:**
- iOS files - We accept upstream
- TypeScript files - We accept upstream
- Build configs - We accept upstream

## Regenerating the Patch

After each successful sync:
```bash
cd react-native-camera-kit-no-google
git diff upstream/master..master > ../patches/no-google.patch
```

This keeps the patch current with any upstream changes we've incorporated.
