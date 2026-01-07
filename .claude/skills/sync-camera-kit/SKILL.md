---
name: sync-camera-kit
description: Synchronizes changes from Tesla's react-native-camera-kit to no-google fork. Selectively syncs iOS and TypeScript improvements while preserving QR-only Android implementation. Use when user requests to "sync camera kit", "sync react-native-camera-kit", or mentions updating from Tesla upstream.
version: 1.0.0
---

# sync-camera-kit

Selectively synchronize changes from Tesla's react-native-camera-kit to the no-google fork, extracting valuable improvements while preserving the QR-only Android barcode implementation.

## Overview

This skill manages the selective synchronization of changes from the upstream Tesla repository to the limpbrains fork. Unlike `sync-qr` which translates algorithms between languages, this skill performs intelligent hunk-level extraction to benefit from upstream improvements while maintaining the fork's QR-only architecture.

**Key Principle**: ALL files are evaluated for selective sync - no files are completely excluded from upstream improvements.

## Repository Context

**Upstream (Tesla):**
- Path: `/Users/limp/dev/qr-root/react-native-camera-kit-tesla`
- Android barcode: Google ML Kit (closed source, multi-format)
- Branch: `master`

**Fork (No-Google):**
- Path: `/Users/limp/dev/qr-root/react-native-camera-kit-no-google`
- Android barcode: limpbrains/qr (open source, QR-only)
- Fork point: `5a709e0`
- Documentation: `CLAUDE.md` (architecture + sync state)

## File Classification

### Category A: Auto-Sync (No Fork Divergence)

Apply entire file changes directly:

- `/ios/**/*` - All iOS Swift files
- `/src/**/*.ts`, `/src/**/*.tsx` - TypeScript React Native layer
- `/src/__tests__/**/*` - Jest tests
- `/.eslintrc.js`, `/tsconfig.json`, `/jest.config.js` - Build configs
- `/example/**/*` - Example app
- `/android/src/main/java/com/rncamerakit/events/**/*` - Event classes
- `/android/src/*/java/com/rncamerakit/CKCameraManager.kt` - Camera manager

### Category B: Selective Sync (Fork Divergence - Hunk-Level Analysis)

Extract compatible hunks, skip ML Kit-specific code:

- **CKCamera.kt** - Extract non-barcode improvements, preserve String callback
- **QRCodeAnalyzer.kt** - Extract camera lifecycle improvements, preserve QRDecoder usage
- **CodeFormat.kt** - Extract new format enums, preserve simplified structure
- **build.gradle** - Extract shared dependency updates, preserve limpbrains/qr
- **package.json** - Extract scripts/devDependencies, preserve name/version/repo
- **README.md** - Extract API updates, preserve fork notice
- **CLAUDE.md** - Update sync state section only
- **ReactNativeCameraKit.podspec** - Extract dependency changes, preserve fork URLs

See `CONFLICT_RESOLUTION.md` for detailed hunk classification strategies per file.

## 10-Step Sync Workflow

### Step 1: Check Sync State

Read sync state from fork's `CLAUDE.md`:

```bash
grep -A 10 "## Camera Kit Sync State" /Users/limp/dev/qr-root/react-native-camera-kit-no-google/CLAUDE.md
```

Extract:
- `Last synchronized upstream commit`: Starting point for diff
- `Fork version`: Current fork version

If section missing, default to fork point `5a709e0`.

### Step 2: Pull Latest Changes

Update upstream repository:

```bash
cd /Users/limp/dev/qr-root/react-native-camera-kit-tesla
git fetch origin
git pull origin master
```

### Step 3: Detect Changes

Get commit range and file changes:

```bash
git log --oneline <last-sync-hash>..HEAD
git log --name-status --format="%H %s" <last-sync-hash>..HEAD
```

If no changes: Exit with "Already up to date".

### Step 4: Analyze Changes

Classify changed files by category:

**For each changed file:**
1. Determine if Category A or Category B
2. If Category A: Mark for auto-sync
3. If Category B: Mark for selective sync with specific strategy

Display analysis to user with recommendations.

### Step 5: Apply Changes Selectively

**Category A (Auto-Sync):**
```bash
git diff <last-sync>..HEAD -- <file-path>
```
Apply entire diff using Edit tool.

**Category B (Selective Sync):**

For each file, use file-specific strategy (see `CONFLICT_RESOLUTION.md`):

1. Extract diff hunks
2. Classify hunks:
   - **Apply**: Generic improvements (error handling, lifecycle, etc.)
   - **Skip**: ML Kit dependencies, barcode filtering, Google-specific APIs
3. Apply compatible hunks only
4. Log extraction decisions

**Example hunk classification keywords:**

Skip if contains:
- `google.mlkit`, `BarcodeScanner`, `Barcode.FORMAT_*`
- `onBarcodeRead(List<Barcode>)`, `allowedBarcodeTypes`
- `fromBarcodeType()`, `toBarcodeType()`

Apply if contains:
- Error handling patterns (`try/catch`, `onError`)
- Camera lifecycle (`image.close()`, `cameraProviderFuture`)
- Focus, zoom, flash improvements
- New enum values (without ML Kit conversions)
- Shared dependency version bumps

### Step 6: Update CLAUDE.md

Add comprehensive sync report to fork's `CLAUDE.md`:

```markdown
## Camera Kit Sync State

**Last synchronized upstream commit**: <new-hash>
**Upstream version**: <upstream-version>
**Fork version**: <fork-version>
**Last sync date**: <ISO-8601>
**Sync status**: success | partial | conflict
**Fork point**: 5a709e0

### Changes Synced (commits <old-hash>..<new-hash>)

**iOS Improvements**: [list]
**TypeScript Layer**: [list]
**Android (Non-Barcode)**: [list]

### Changes Skipped (Android Barcode Conflicts)

**File.kt** (commits <hash>, <hash>):
- Upstream: [what changed]
- Fork: [why incompatible]
- Action: [what was applied vs skipped]

### Selective Sync Summary

**QRCodeAnalyzer.kt**: [hunks applied vs skipped]
**CodeFormat.kt**: [hunks applied vs skipped]
**build.gradle**: [dependencies updated vs preserved]

### Notes

Fork maintains QR-only Android barcode scanning using limpbrains/qr. All files are selectively synced - no files are completely excluded from upstream improvements.
```

### Step 7: Run Tests

Build and validate:

```bash
cd /Users/limp/dev/qr-root/react-native-camera-kit-no-google
yarn build    # TypeScript compilation
yarn lint     # ESLint
yarn test     # Jest (minimal tests)
```

**Critical validation checks:**

```bash
# Ensure no Google ML Kit leaked in
grep -r "google.mlkit" android/build.gradle  # Should be empty

# Ensure fork dependency intact
grep "limpbrains/qr" android/build.gradle    # Should find it

# Ensure QRDecoder preserved
grep "QRDecoder.decode" android/src/main/java/com/rncamerakit/QRCodeAnalyzer.kt  # Should find it
```

### Step 8: Report Summary

Display comprehensive summary:

```
Sync Summary
============
Upstream: teslamotors/react-native-camera-kit
Range: <old-hash>..<new-hash> (N commits)

Files Auto-Synced (Category A): N files
Files Selectively Synced (Category B): N files
  - Hunks Applied: N hunks
  - Hunks Skipped: N hunks (incompatible with QR-only fork)

Build Status: ✓/✗
Lint Status: ✓/✗
Test Status: ✓/✗

Key Selective Sync Decisions:
- QRCodeAnalyzer.kt: [extraction summary]
- CodeFormat.kt: [extraction summary]
- build.gradle: [extraction summary]
```

### Step 9: Commit Changes

Ask user for confirmation, then commit:

```bash
git add .
git commit -m "Sync from teslamotors/react-native-camera-kit@<hash>

Selectively synced upstream changes while preserving QR-only Android implementation:
- [Category A auto-synced files]
- [Category B selectively synced files with hunk counts]

Upstream range: <old-hash>..<new-hash> (N commits)

Extraction decisions:
- QRCodeAnalyzer.kt: Applied X hunks, skipped Y hunks (ML Kit incompatible)
- CodeFormat.kt: Applied X hunks, skipped Y hunks (conversion methods)
- build.gradle: Updated shared deps, preserved limpbrains/qr

🤖 Generated with Claude Code
Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>"
```

### Step 10: Final Status

Report completion status to user with:
- Total files synced
- Total hunks applied vs skipped
- Build/lint/test results
- Next manual actions if needed

## Success Criteria

A successful sync:

1. ✓ All Category A files synced (entire files)
2. ✓ All Category B files selectively synced (hunk-level extraction)
3. ✓ `yarn build` passes
4. ✓ `yarn lint` passes
5. ✓ `yarn test` passes
6. ✓ No Google ML Kit in build.gradle
7. ✓ QRCodeAnalyzer.kt preserves QRDecoder.decode() usage
8. ✓ CodeFormat.kt preserves simplified enum structure
9. ✓ build.gradle preserves limpbrains/qr dependency
10. ✓ CLAUDE.md sync state updated with extraction details
11. ✓ Clear commit message with upstream range and selective sync summary

## Rollback Strategy

If sync breaks fork:

```bash
cd /Users/limp/dev/qr-root/react-native-camera-kit-no-google
git reset --hard HEAD~1
# Review conflict, adjust FILE_MAPPING.md rules
```

## Related Documentation

- `FILE_MAPPING.md` - Detailed file classification and patterns
- `CONFLICT_RESOLUTION.md` - Hunk-level strategies per file
- `SYNC_STRATEGY.md` - High-level sync philosophy

## Key Differences from sync-qr

| Aspect | sync-qr | sync-camera-kit |
|--------|---------|-----------------|
| **Task** | Algorithm port | Selective file sync |
| **Translation** | Heavy (JS→Kotlin) | None (hunk extraction) |
| **Scope** | Decode-only | ALL files (selective) |
| **Philosophy** | Port algorithm | Extract improvements |
| **Complexity** | High (code translation) | Medium (hunk classification) |
