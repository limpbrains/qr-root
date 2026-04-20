---
name: sync-camera-kit
description: Synchronizes changes from Tesla's react-native-camera-kit to no-google fork. Uses git merge with patch-based conflict resolution. Use when user requests to "sync camera kit", "sync react-native-camera-kit", or mentions updating from Tesla upstream.
version: 2.0.0
---

# sync-camera-kit

Synchronize changes from Tesla's react-native-camera-kit to the no-google fork using git merge and patch-based conflict resolution.

## Overview

This skill manages synchronization from the upstream Tesla repository to the limpbrains fork. It uses a standard git merge workflow with a patch file that captures all no-google customizations for reference during conflict resolution.

## Repository Context

**Upstream (Tesla):**
- Remote: `upstream` -> `https://github.com/teslamotors/react-native-camera-kit.git`
- Android barcode: Google ML Kit (closed source, multi-format)
- Branch: `master`

**Fork (No-Google):**
- Path: `/Users/limp/dev/qr-root/react-native-camera-kit-no-google`
- Remote: `origin` -> `git@github.com:limpbrains/react-native-camera-kit-no-google.git`
- Android barcode: limpbrains/qr (open source, QR-only)
- Documentation: `CLAUDE.md` (architecture + sync state)

**Patch File:**
- Path: `/Users/limp/dev/qr-root/patches/no-google.patch`
- Contains: All customizations that differ from upstream
- Updated: After each successful sync

## 8-Step Sync Workflow

### Step 1: Check Current State

```bash
cd /Users/limp/dev/qr-root/react-native-camera-kit-no-google
git status
git remote -v  # Verify upstream remote exists
```

If upstream remote missing:
```bash
git remote add upstream https://github.com/teslamotors/react-native-camera-kit.git
```

### Step 2: Fetch Upstream

```bash
git fetch upstream
```

Check what's new:
```bash
git log master..upstream/master --oneline
```

If no new commits: Exit with "Already up to date".

### Step 3: Create Sync Branch

```bash
git checkout -b sync-upstream-$(date +%Y-%m-%d)
```

### Step 4: Merge Upstream

```bash
git merge upstream/master
```

This will either:
- **Clean merge**: No conflicts, proceed to Step 5
- **Conflicts**: Resolve using guidance below

### Step 5: Resolve Conflicts

Reference `patches/no-google.patch` to understand our customizations.

**Key files that typically conflict:**

| File | Our Customization |
|------|-------------------|
| `android/build.gradle` | Uses `com.github.limpbrains:qr` instead of Google ML Kit |
| `android/.../QRCodeAnalyzer.kt` | Uses `QRDecoder.decode()` instead of ML Kit scanner |
| `android/.../CKCamera.kt` | String callback `onQRCodeDetected(String)` instead of `List<Barcode>` |
| `android/.../CodeFormat.kt` | Simplified enum without ML Kit conversion methods |
| `package.json` | Different name/version/repository |
| `CLAUDE.md` | Fork-specific documentation |

**Conflict resolution rules:**

1. **Keep our dependency**: `implementation 'com.github.limpbrains:qr:...'`
2. **Reject ML Kit**: Any `google.mlkit`, `BarcodeScanner`, `Barcode.FORMAT_*`
3. **Keep our callback signature**: `onQRCodeDetected(decodedValue: String)`
4. **Accept upstream improvements**: Camera lifecycle, error handling, iOS changes, TypeScript layer

After resolving:
```bash
git add .
git commit -m "Resolve merge conflicts, preserve no-google customizations"
```

### Step 6: Verify & Test

```bash
yarn build    # TypeScript compilation
yarn lint     # ESLint
yarn test     # Jest tests
```

**Critical validation:**
```bash
# No Google ML Kit leaked in
grep -r "google.mlkit" android/  # Should be empty

# Fork dependency intact
grep "limpbrains/qr" android/build.gradle  # Should find it

# QRDecoder preserved
grep "QRDecoder.decode" android/src/main/java/com/rncamerakit/QRCodeAnalyzer.kt
```

### Step 7: Merge to Master & Regenerate Patch

```bash
git checkout master
git merge sync-upstream-$(date +%Y-%m-%d)

# Regenerate patch to capture any upstream changes we want to keep
git diff upstream/master..master > /Users/limp/dev/qr-root/patches/no-google.patch
```

### Step 8: Update CLAUDE.md & Commit

Update the sync state section in `CLAUDE.md`:

```markdown
## Camera Kit Sync State

**Last synchronized upstream commit**: <new-hash>
**Last sync date**: <ISO-8601>
**Sync status**: success
```

Commit in the parent qr-root repo if patch changed:
```bash
cd /Users/limp/dev/qr-root
git add patches/no-google.patch
git commit -m "Update no-google.patch after camera-kit sync"
```

## Success Criteria

A successful sync:

1. `git merge upstream/master` completed (with or without conflicts)
2. All conflicts resolved preserving no-google customizations
3. `yarn build` passes
4. `yarn lint` passes
5. `yarn test` passes
6. No `google.mlkit` in android/ directory
7. `limpbrains/qr` dependency present in build.gradle
8. `QRDecoder.decode()` present in QRCodeAnalyzer.kt
9. `patches/no-google.patch` regenerated
10. CLAUDE.md sync state updated

## Rollback Strategy

If sync breaks the fork:

```bash
cd /Users/limp/dev/qr-root/react-native-camera-kit-no-google
git checkout master
git branch -D sync-upstream-<date>  # Delete failed sync branch
```

## Quick Reference

```bash
# Full sync in one go (if no conflicts expected)
cd /Users/limp/dev/qr-root/react-native-camera-kit-no-google
git fetch upstream
git checkout -b sync-upstream-$(date +%Y-%m-%d)
git merge upstream/master
yarn build && yarn lint && yarn test
git checkout master && git merge sync-upstream-$(date +%Y-%m-%d)
git diff upstream/master..master > ../patches/no-google.patch
```
