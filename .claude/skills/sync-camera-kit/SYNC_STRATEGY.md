# SYNC_STRATEGY.md

High-level strategy for managing the react-native-camera-kit fork using git merge workflow.

## Philosophy

**Core Principle**: Use standard git merge from upstream, resolving conflicts to preserve fork's QR-only architecture.

The fork maintains a `patches/no-google.patch` file that captures all customizations. This patch serves as a reference during conflict resolution and is regenerated after each successful sync.

## Fork Characteristics

### Intentional Differences (Always Preserve)

1. **Android Barcode Scanner**: limpbrains/qr (QR-only) vs Google ML Kit (multi-format)
2. **Callback Signature**: `String` callback vs `List<Barcode>` callback
3. **Dependencies**: No Google ML Kit, uses JitPack for limpbrains/qr
4. **Package Metadata**: Fork name, repository URL, independent versioning

### Stay in Sync (Accept Upstream)

1. iOS implementation (no fork changes)
2. TypeScript/React layer (no fork changes)
3. Build tooling and configuration
4. Example app
5. Camera features (zoom, focus, flash)

## Sync Workflow

```
1. git fetch upstream
2. git checkout -b sync-upstream-<date>
3. git merge upstream/master
4. Resolve conflicts (use patch as reference)
5. yarn build && yarn lint && yarn test
6. git checkout master && git merge sync-branch
7. Regenerate patch
```

## Conflict Resolution Principles

### When Conflicts Occur

Conflicts typically happen in:
- `android/build.gradle` - dependency differences
- `android/.../QRCodeAnalyzer.kt` - scanner implementation
- `android/.../CKCamera.kt` - callback signatures
- `android/.../CodeFormat.kt` - enum structure
- `package.json` - metadata

### Resolution Rules

1. **Keep our scanner**: `QRDecoder.decode()` not ML Kit
2. **Keep our dependency**: `com.github.limpbrains:qr` not `google.mlkit`
3. **Keep our callback**: `onQRCodeDetected(String)` not `List<Barcode>`
4. **Accept their improvements**: Error handling, camera lifecycle, iOS changes

### Using the Patch as Reference

When resolving conflicts:
```bash
# View our customizations
cat ../patches/no-google.patch

# Search for specific file
grep -A 50 "a/android/build.gradle" ../patches/no-google.patch
```

## Sync Frequency

**Recommended**: Sync after each upstream release
**Minimum**: Quarterly

## Success Metrics

- `yarn build && yarn lint && yarn test` all pass
- No `google.mlkit` in android/ directory
- `limpbrains/qr` dependency present
- `QRDecoder.decode()` preserved
- Patch regenerated

## Rollback

If sync fails:
```bash
git checkout master
git branch -D sync-upstream-<date>
```
