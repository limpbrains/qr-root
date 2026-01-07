# SYNC_STRATEGY.md

High-level strategy for managing the react-native-camera-kit fork and staying synchronized with upstream improvements.

## Philosophy

**Core Principle**: Extract maximum value from upstream while preserving fork architecture.

Unlike traditional forks that drift away from upstream over time, this fork maintains an active synchronization strategy. We selectively extract improvements from every upstream commit, ensuring the fork benefits from Tesla's continued development while maintaining the QR-only Android implementation.

**No files are off-limits** - every upstream change is evaluated for potential value, even in files with significant divergence.

---

## Fork Management Model

### The "Selective Extraction" Model

```
Upstream (Tesla)          Fork (No-Google)
     |                          |
     |------ Sync Process ----->|
     |                          |
   commit                   selective
   changes                  extraction
     |                          |
     v                          v
  All files              Category A: Full sync
  updated               Category B: Hunk extraction
                               |
                               v
                         QR-only Android
                         + All upstream
                           improvements
```

**NOT a traditional fork**: We don't maintain a separate branch that diverges permanently.

**Instead**: We perform continuous selective merges, extracting value from every upstream commit.

---

## Divergence Points

### Intentional Divergence

These divergences are **by design** and will persist:

1. **Android Barcode Scanner**: limpbrains/qr (QR-only, open source) vs Google ML Kit (multi-format, closed source)
2. **Android Barcode API**: String callback vs List<Barcode> callback
3. **Package Metadata**: Fork name, repository URL, versioning
4. **Documentation**: Fork notice, comparison table, limitations

### Unintentional Divergence (To Be Minimized)

These divergences are **technical debt** to be eliminated:

1. Missed upstream bug fixes
2. Missed upstream performance improvements
3. Missed upstream iOS enhancements
4. Missed upstream TypeScript/API improvements

**Goal**: Keep unintentional divergence near zero through regular syncing.

---

## Sync Frequency

### Recommended Schedule

**Ideal**: Sync after every upstream release (v16.x.x tags)
**Minimum**: Sync at least quarterly

### Triggers for Immediate Sync

Sync immediately if upstream:
- Fixes critical security vulnerability
- Fixes critical camera crash
- Adds iOS features
- Updates React Native peer dependencies
- Updates CameraX dependencies (Android)

### Triggers to Skip Sync

Can delay sync if upstream:
- Only adds new barcode formats (no benefit to QR-only fork)
- Only changes ML Kit-specific configuration
- Only updates Google-specific documentation

---

## State Tracking Philosophy

### Why Track Sync State?

1. **Transparency**: Users know what upstream improvements are included
2. **Reproducibility**: Can trace any fork behavior to specific upstream commits
3. **Debugging**: Can compare fork vs upstream at specific commit ranges
4. **Compliance**: Clear attribution of upstream contributions

### What to Track

Store in `/Users/limp/dev/qr-root/react-native-camera-kit-no-google/CLAUDE.md`:

#### Mandatory Fields
- **Last synchronized upstream commit**: Git hash
- **Upstream version**: Semantic version (e.g., v16.2.0)
- **Fork version**: Independent fork version
- **Last sync date**: ISO-8601 timestamp
- **Sync status**: success | partial | conflict
- **Fork point**: Original fork commit (5a709e0)

#### Mandatory Sections
- **Changes Synced**: What was fully applied (Category A files)
- **Changes Skipped**: What hunks were skipped and why (Category B files)
- **Selective Sync Summary**: Detailed hunk counts per file

#### Optional Sections
- **Manual Actions Required**: TODOs for user
- **Known Divergence**: Intentional differences
- **Upstream Incompatibilities**: Features that can't be ported

---

## Conflict Resolution Strategy

### Hunk-Level Decision Tree

```
For each diff hunk:
│
├─ Contains ML Kit import/usage?
│  └─ YES → SKIP (log reason)
│
├─ Contains Barcode type/multi-format logic?
│  └─ YES → SKIP (log reason)
│
├─ Contains allowedBarcodeTypes?
│  └─ YES → SKIP (log reason)
│
├─ Generic improvement (error handling, lifecycle)?
│  └─ YES → APPLY (log decision)
│
├─ Shared dependency update (CameraX, Kotlin)?
│  └─ YES → APPLY (log decision)
│
└─ Uncertain?
   └─ Default: SKIP (log as manual review required)
      User can manually apply later
```

### When to Ask User

Automatic decision insufficient if:
1. Hunk mixes barcode and non-barcode logic (can't cleanly separate)
2. Major architectural change (e.g., CameraX 2.0 migration)
3. Breaking change to public API
4. New prop/feature with unclear QR-only implications

**Default**: When in doubt, preserve fork architecture (SKIP). It's easier to add a missed improvement later than to fix a broken fork.

---

## Version Management

### Fork Versioning Strategy

**Fork version is decoupled from upstream version.**

**Rationale**: Fork and upstream have different release cadences. Fork may release multiple times between upstream releases (e.g., to update limpbrains/qr dependency).

**Scheme**: Fork uses independent semantic versioning (e.g., v16.1.3) while tracking upstream version separately.

### When to Bump Fork Version

**Patch** (16.1.X):
- Bug fixes in fork-specific code
- limpbrains/qr dependency updates
- Documentation fixes

**Minor** (16.X.0):
- Synced upstream improvements
- New iOS features from upstream
- New props/API from upstream (if applicable to fork)

**Major** (X.0.0):
- Breaking changes to fork's public API
- React Native peer dependency major version bump
- Major upstream sync that changes behavior

---

## Testing Strategy

### Pre-Sync Validation

Before starting sync:
```bash
cd /Users/limp/dev/qr-root/react-native-camera-kit-no-google
yarn build && yarn lint && yarn test
```

**All must pass** - ensures fork is in good state before changes.

### Post-Sync Validation

After applying changes:

#### 1. Build Tests
```bash
yarn build    # TypeScript compilation
yarn lint     # ESLint
yarn test     # Jest
```

#### 2. Fork Integrity Checks
```bash
# No Google ML Kit leaked in
grep -r "google.mlkit" android/ && echo "FAIL: ML Kit found" || echo "PASS"

# Fork dependency intact
grep "limpbrains/qr" android/build.gradle || echo "FAIL: qr dependency missing"

# QRDecoder usage preserved
grep "QRDecoder.decode" android/src/main/java/com/rncamerakit/QRCodeAnalyzer.kt || echo "FAIL: QRDecoder missing"
```

#### 3. Manual Testing (Recommended)

**iOS**:
- Build example app: `cd example/ios && pod install && cd .. && npx react-native run-ios`
- Test camera preview
- Test QR code scanning

**Android**:
- Build example app: `npx react-native run-android`
- Test camera preview
- Test QR code scanning with limpbrains/qr

### Regression Prevention

If post-sync tests fail:
1. Review selective sync decisions
2. Identify which hunk caused failure
3. Revert that specific hunk
4. Update FILE_MAPPING.md to prevent similar failures
5. Re-run tests

---

## Long-Term Maintenance

### Quarterly Fork Health Checkup

Every 3 months:

1. **Review divergence**:
   ```bash
   cd /Users/limp/dev/qr-root/react-native-camera-kit-tesla
   git log --oneline <last-sync>..HEAD
   ```

2. **Check for security issues**:
   ```bash
   cd /Users/limp/dev/qr-root/react-native-camera-kit-no-google
   yarn audit
   ```

3. **Update dependencies**:
   - Update limpbrains/qr if new version available
   - Update React Native peer dependencies if needed

4. **Sync from upstream** (follow 10-step workflow)

### When to Consider Rebasing Fork

Consider full rebase (instead of selective sync) if:
- Upstream has major architectural refactor (e.g., CameraX 2.0)
- Fork has accumulated significant tech debt
- Easier to re-apply QR-only changes than to continue selective sync

**Trigger**: If selective sync takes >4 hours per release, evaluate rebase cost.

---

## Rollback Strategy

### If Sync Breaks Fork

#### Immediate Rollback
```bash
cd /Users/limp/dev/qr-root/react-native-camera-kit-no-google
git reset --hard HEAD~1
```

#### Analyze Failure
```bash
git log -1 --stat  # See what changed
yarn build         # See specific error
```

#### Fix and Retry
1. Update FILE_MAPPING.md with new skip rule
2. Update CONFLICT_RESOLUTION.md with hunk strategy
3. Re-run sync with corrected rules

### If Upstream Diverges Too Much

If selective sync becomes unmaintainable:

**Option 1**: Fork freeze
- Stop syncing from upstream
- Maintain fork independently
- Only backport critical security fixes

**Option 2**: Architectural alignment
- Modify fork to use adapter pattern
- Create abstraction layer over barcode scanner
- Allows swapping ML Kit <-> limpbrains/qr
- Reduces divergence, enables easier syncing

---

## Success Metrics

### Healthy Fork Indicators

✅ **Sync Frequency**: At least quarterly syncs
✅ **Sync Duration**: <2 hours per upstream release
✅ **Hunk Success Rate**: >80% of non-barcode hunks applied
✅ **Build Success**: yarn build && yarn lint && yarn test all pass
✅ **Fork Integrity**: All validation checks pass

### Unhealthy Fork Indicators

❌ **Sync Lag**: >6 months behind upstream
❌ **Sync Duration**: >4 hours per upstream release
❌ **Hunk Success Rate**: <50% of non-barcode hunks applied
❌ **Build Failures**: Frequent post-sync build failures
❌ **Missed Improvements**: iOS/TypeScript improvements not synced

If fork becomes unhealthy, consider:
1. Updating FILE_MAPPING.md rules (too conservative?)
2. Improving CONFLICT_RESOLUTION.md strategies
3. Allocating more time to sync process
4. Re-evaluating fork maintenance strategy

---

## Communication Strategy

### Documenting Sync in CLAUDE.md

Each sync must update fork's CLAUDE.md with:

1. **Executive summary**: What was synced, what was skipped
2. **Detailed breakdown**: File-by-file hunk decisions
3. **Rationale**: Why certain hunks were skipped
4. **Testing results**: Build/lint/test status
5. **Known limitations**: What upstream features fork doesn't support

**Goal**: Any developer can read CLAUDE.md and understand fork's relationship to upstream.

### Communicating to Users (README.md)

Fork's README.md should clearly state:

1. **Fork purpose**: Why this fork exists (QR-only, no Google dependencies)
2. **Sync status**: Last upstream version synced
3. **Limitations**: What upstream features are missing (multi-format scanning)
4. **Trade-offs**: What users gain (open source, privacy) vs lose (multi-format)

---

## Comparison: sync-qr vs sync-camera-kit

Understanding the difference helps apply the right sync strategy:

| Aspect | sync-qr | sync-camera-kit |
|--------|---------|-----------------|
| **Upstream → Fork** | paulmillr/qr (JS) → limpbrains/qr (Kotlin) | teslamotors/react-native-camera-kit → limpbrains fork |
| **Sync Type** | Algorithm translation | File/hunk extraction |
| **Frequency** | Ad-hoc (when upstream improves decoder) | Regular (per upstream release) |
| **Complexity** | High (manual code porting) | Medium (hunk classification) |
| **Automation Potential** | Low (requires understanding algorithm) | High (keyword-based hunk filtering) |
| **State Tracking** | Root CLAUDE.md | Fork's CLAUDE.md |
| **Testing** | 9k test vectors | Build + lint + manual |

**Key Insight**: sync-camera-kit is more automatable because it extracts code, not translates it. This allows for regular, sustainable syncing.

---

## Future Improvements

### Potential Automation

**Hunk Classification Automation**:
- Script to automatically classify hunks by keywords
- Generates "suggested APPLY/SKIP" for review
- Reduces manual effort from 2 hours → 30 minutes per sync

**Implementation**:
```bash
# Pseudo-code
for file in changed_files:
    diff = git diff $last_sync..HEAD -- $file
    for hunk in diff.hunks:
        if contains(hunk, SKIP_KEYWORDS):
            log("SKIP: $hunk (reason: ML Kit)")
        elif contains(hunk, APPLY_KEYWORDS):
            log("APPLY: $hunk (reason: generic improvement)")
        else:
            log("REVIEW: $hunk")
```

### Long-Term Vision

**Goal**: Maintain fork indefinitely with <1 hour sync time per upstream release.

**Path**:
1. Improve FILE_MAPPING.md heuristics over time
2. Build automation scripts for hunk classification
3. Establish monthly sync cadence
4. Keep fork <1 release behind upstream

**Success**: Fork becomes a sustainable, long-term maintained alternative to upstream.
