---
name: sync-qr
description: This skill should be used when the user asks to "sync qr", "sync libraries", "port JS changes to Kotlin", "update Kotlin from JavaScript", or mentions synchronizing the qr-paulmillr and qr repositories. Automatically synchronizes QR code decode algorithm changes from JavaScript library to Kotlin port.
version: 1.0.0
---

# QR Library Synchronization Skill

## Overview

This skill synchronizes decode-related changes from the JavaScript QR library (qr-paulmillr) to the Kotlin port (qr). It automatically detects changes, ports decode-focused code while ignoring encoding logic, maintains Kotlin idioms, synchronizes tests, and attempts to fix test failures.

## Repository Locations

- **Root**: `/Users/limp/dev/qr-root/`
- **JS Library**: `/Users/limp/dev/qr-root/qr-paulmillr/`
- **Kotlin Library**: `/Users/limp/dev/qr-root/qr/`
- **Sync State**: `/Users/limp/dev/qr-root/CLAUDE.md`

## Sync Workflow (10 Steps)

### Step 1: Check Sync State

Read the last synced commit hash from `/Users/limp/dev/qr-root/CLAUDE.md`.

Look for section:
```markdown
## QR Sync State

Last synchronized commit: <commit-hash>
```

If no sync state exists:
- Ask user if this is the first sync
- Offer to start from current HEAD or specify a starting commit
- Default: analyze only latest commit for first sync

### Step 2: Pull Latest Changes

Update the qr-paulmillr submodule:
```bash
cd /Users/limp/dev/qr-root/qr-paulmillr
git fetch origin
git pull origin main
```

Record the new HEAD commit hash.

### Step 3: Detect Changes

If sync state exists:
```bash
cd /Users/limp/dev/qr-root/qr-paulmillr
git log --oneline <last-hash>..HEAD
```

If no changes (last-hash == HEAD):
- Report "Already up to date"
- Exit successfully

For each commit in range:
```bash
git show <commit> --stat
git show <commit> --name-status
```

### Step 4: Analyze Changes

Filter commits to identify decode-related changes.

**Include patterns** (decode-related):
- `src/decode.ts` - ALL changes (core decode file)
- `src/index.ts` - PARTIAL (Bitmap, Transform, GF, RS, Interleave, utils)
- `test/decode.test.ts` - Decode tests
- `test/bitmap.test.ts` - Bitmap tests
- `test/utils.test.ts` - Utility tests (GF, RS)
- `test/qr.test.ts` - Integration tests (decode portions)

**Exclude patterns** (encoding/browser-only):
- `src/index.ts` - Encoding functions (encode, drawTemplate, generateECC, maskScore, etc.)
- `src/dom.ts` - ALL changes (browser-specific)
- `test/encode.test.ts` - ALL changes
- `test/dom.test.ts` - ALL changes
- `README.md`, `package.json`, config files - Non-code

**Function-level filtering for index.ts**:
For changes in `src/index.ts`, extract function/class names and check:
- **Include**: Bitmap class, Transform functions, GaloisField, ReedSolomon, Interleave, utility functions
- **Exclude**: encode*, draw*, generate*, mask*, penalty* functions

Display summary to user:
```
Found 3 commits since last sync:

1. 0fbd0a4 - Add ECI support. Add inverted qr support. Speed-up (579 insertions, 317 deletions)
   - src/decode.ts: Added ECI decoding, inverted QR fallback
   - src/index.ts: Bitmap.negate() method
   - test/decode.test.ts: ECI test cases

2. <hash> - Fix pattern detection edge case
   - src/decode.ts: Improved finder pattern validation

3. <hash> - Performance optimization in GF operations
   - src/index.ts: GaloisField lookup table optimization

Changes to port:
✓ ECI encoding support (decode.ts → BitDecoder.kt)
✓ Inverted QR support (decode.ts → PatternDetector.kt, Bitmap.kt)
✓ Bitmap.negate() method (index.ts → Bitmap.kt)
✓ Pattern detection improvements (decode.ts → PatternDetector.kt)
✓ GF optimization (index.ts → GaloisField.kt)
✓ Test cases (decode.test.ts → VectorTest.kt, QRDecoderTest.kt)

Proceed with porting? (yes/no)
```

### Step 5: Port Changes

For each identified change:

1. **Read JavaScript diff**:
   ```bash
   git diff <commit>^..<commit> -- src/decode.ts
   ```

2. **Identify target Kotlin file** using [FILE_MAPPING.md](FILE_MAPPING.md):
   - `decode.ts` detection logic → `PatternDetector.kt`
   - `decode.ts` decoder → `BitDecoder.kt`
   - `index.ts` Bitmap → `Bitmap.kt`
   - `index.ts` GF → `GaloisField.kt`
   - etc.

3. **Read current Kotlin code** in target file

4. **Translate using [TRANSLATION_PATTERNS.md](TRANSLATION_PATTERNS.md)**:
   - Apply type translations (number → Int/Double, Uint8Array → ByteArray)
   - Apply syntax translations (for loops → ranges, bitwise → and/or/xor)
   - Apply Kotlin idioms (when expressions, null safety, operators)
   - Preserve Kotlin style (object, data classes, val over var)
   - Handle special cases (see TRANSLATION_PATTERNS.md)

5. **Edit Kotlin file** with translated code

6. **Preserve Kotlin-only enhancements**:
   - Grayscale input support (Image.kt, PatternDetector.kt)
   - Triangle validation (PatternDetector.kt)
   - Threshold retry (QRDecoder.kt)
   - Relaxed finder variance (PatternDetector.kt)

**Critical reminders**:
- Bitmap access: `b.data[y][x]` (JS) → `b.get(x, y)` (Kotlin) - **x and y are SWAPPED!**
- Arrays: `new Uint8Array(n)` → `ByteArray(n)`
- Bitwise: `a & b` → `a and b`, `a | b` → `a or b`, `a ^ b` → `a xor b`
- Loops: `for (let i = 0; i < n; i++)` → `for (i in 0 until n)`

### Step 6: Sync Tests

For each relevant test change:

1. **Identify Kotlin test file** using [TEST_STRATEGY.md](TEST_STRATEGY.md):
   - `decode.test.ts` → `VectorTest.kt`, `ImageDecodingTest.kt`, `QRDecoderTest.kt`
   - `bitmap.test.ts` → `BitmapTest.kt`
   - `utils.test.ts` → `GaloisFieldTest.kt`, `ReedSolomonTest.kt`, `QRInfoTest.kt`

2. **Translate test** using test framework mappings (see TEST_STRATEGY.md):
   - `it('should X')` → `@Test fun \`should X\`()`
   - `assert.strictEqual(a, b)` → `assertEquals(b, a)` (**ORDER SWAPPED!**)
   - `assert.ok(cond)` → `assertTrue(cond)`
   - `assert.throws(() => ...)` → `assertThrows<ExceptionType> { ... }`

3. **CRITICAL: Synchronize test vectors submodule**:
   Both repositories use the same test vectors from a git submodule. They MUST be at the same commit.

   ```bash
   # Check JS library's test vectors version
   cd /Users/limp/dev/qr-root/qr-paulmillr
   git submodule status test/vectors  # Note the commit hash

   # Update Kotlin test vectors to match
   cd /Users/limp/dev/qr-root/qr
   git submodule update --init --recursive
   cd test/vectors
   git checkout <commit-hash-from-js>  # Use the hash from above
   ```

   **Important**: Test vectors must be initialized and at the same commit for tests to pass.

### Step 7: Build and Test

Build the Kotlin project:
```bash
cd /Users/limp/dev/qr-root/qr
./gradlew clean build
```

If build fails:
- Review compilation errors
- Fix type mismatches, imports, syntax errors
- Common issues: missing imports, wrong types, null safety

Run tests:
```bash
./gradlew test
```

Capture and parse output.

### Step 8: Auto-Fix Test Failures

If tests fail, attempt automatic fixes (up to 3 iterations).

**Failure categories**:

1. **Off-by-One Errors** (ArrayIndexOutOfBoundsException):
   - Fix: Change `0..n` → `0 until n`, `start..end` → `start until end`

2. **Type Mismatches** (type inference errors):
   - Fix: Add `.toInt()`, `.toDouble()`, change array types

3. **Null Safety** (NullPointerException):
   - Fix: Add `?` to types, `!!` or `?.` to access, `?: defaultValue`

4. **Bitmap Access Errors** (wrong results):
   - Fix: Verify `get(x, y)` not `get(y, x)` order

5. **Encoding Issues** (garbled text):
   - Fix: Ensure ECI encoding map is complete, use correct charset

**Auto-fix process**:
1. Parse failure output (class, method, expected, actual, stack trace)
2. Categorize failure
3. Apply category-specific fix
4. Re-run tests (`./gradlew test`)
5. Check progress (failures before vs after)
6. Iterate up to 3 times
7. Report remaining failures with diagnostic info

### Step 9: Update Sync State

Edit `/Users/limp/dev/qr-root/CLAUDE.md`:

Add or update section:
```markdown
## QR Sync State

Last synchronized commit: <new-commit-hash>
Last sync date: 2026-01-07T10:30:00Z
Sync status: success
Changes ported:
- Added ECI encoding support (commit 0fbd0a4)
- Added inverted QR support (commit 0fbd0a4)
- Performance optimizations in GF operations (commit <hash>)

Test results:
- VectorTest: 9134/9281 (98.42%)
- ImageDecodingTest: 113/118 (95.76%)
```

If test failures remain:
```markdown
Sync status: partial
Notes: 3 test failures remain in ImageDecodingTest (ECI edge cases)
```

### Step 10: Commit Changes

Ask user if they want to commit:

If yes:
```bash
cd /Users/limp/dev/qr-root/qr
git add .
git commit -m "Sync from qr-paulmillr@<short-hash>: <summary>

- Added ECI encoding support
- Added inverted QR support
- Performance optimizations

Ported from: https://github.com/paulmillr/qr/commit/<hash>"
```

Display commit hash and summary.

## File Mapping Reference

See [FILE_MAPPING.md](FILE_MAPPING.md) for complete mappings.

**Quick reference**:
| JavaScript | Kotlin |
|------------|--------|
| src/decode.ts (detection) | PatternDetector.kt |
| src/decode.ts (decoder) | BitDecoder.kt |
| src/index.ts (Bitmap) | Bitmap.kt |
| src/index.ts (Transform) | Transform.kt |
| src/index.ts (GaloisField) | GaloisField.kt |
| src/index.ts (ReedSolomon) | ReedSolomon.kt |
| src/index.ts (utils.interleave) | Interleave.kt |
| src/index.ts (utils.info) | QRInfo.kt |

## Translation Strategy

See [TRANSLATION_PATTERNS.md](TRANSLATION_PATTERNS.md) for detailed patterns.

**Key principles**:
1. Preserve algorithmic correctness above all
2. Maintain Kotlin coding style (see qr/CLAUDE.md)
3. Use object singletons, data classes, operator overloading
4. Prefer `val` over `var`, use when expressions
5. Apply null safety patterns
6. Watch for Bitmap x/y swap, array index order, assertion order

## Test Strategy

See [TEST_STRATEGY.md](TEST_STRATEGY.md) for detailed guidelines.

**Key points**:
- Map test files correctly (decode.test.ts → multiple Kotlin tests)
- Translate assertions with correct order (assertEquals(expected, actual) in Kotlin)
- Ensure test vectors submodule is updated
- Maintain test pass rate ≥ baseline (98% vectors, 96% images)

## Scope Limitations

**Include (decode-only)**:
- Pattern detection algorithms
- Bitmap operations used in decoding
- Data extraction and parsing
- Error correction (GF, RS, Interleave)
- Performance optimizations in decode path
- Bug fixes in decode path
- New decode features (ECI, inverted QR, etc.)

**Exclude (encoding)**:
- QR code generation/encoding
- Encoding-only optimizations
- DOM utilities (browser-specific)
- Drawing functions, mask scoring, template generation

## Error Handling

**Merge conflicts**:
- Report to user, provide conflict details
- Don't auto-resolve - ask for guidance

**Build failures**:
- Attempt to fix common issues (imports, types)
- Escalate to user if unable to fix

**Test failures**:
- Auto-fix up to 3 iterations
- Report remaining failures with diagnostic info
- Suggest manual fixes

**Missing mappings**:
- If encounter unmapped JS file/function
- Ask user for guidance on target location

## Success Metrics

After sync, verify:
- ✓ All decode-related changes identified
- ✓ Code translated to idiomatic Kotlin
- ✓ Tests pass (≥98% vectors, ≥96% images)
- ✓ Sync state updated correctly
- ✓ Commit message is clear and traceable

## Additional Resources

- **[FILE_MAPPING.md](FILE_MAPPING.md)** - Complete JS→Kotlin file correspondence
- **[TRANSLATION_PATTERNS.md](TRANSLATION_PATTERNS.md)** - JS→Kotlin idiom translations
- **[TEST_STRATEGY.md](TEST_STRATEGY.md)** - Test synchronization guidelines
- **qr/CLAUDE.md** - Kotlin library documentation and coding style
- **qr-paulmillr/README.md** - JavaScript library documentation
