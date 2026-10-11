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
- `src/decode.ts` - ALL changes (core decode file; ignore the BarcodeDetector/canvas tail)
- `src/index.ts` - PARTIAL (spec tables `BYTES`/`WORDS_PER_BLOCK`/`ECC_BLOCKS`, GF `EXP`/`LOG`, `rsGenerator`/`RS_CACHE`, mask predicates, format/version bits)
- `test/decode.test.ts`, `test/decode-batch.test.ts`, `test/utils.test.ts`
- `test/misc/jpg.ts`, `test/misc/png.ts`, `test/jpg.test.ts`, `test/png.test.ts` (luma readers used by every image fixture)
- `test/vectors` submodule bumps

**Exclude patterns** (encoding/browser-only):
- `src/index.ts` - Encoding functions (encode*, draw*, penalty*, SVG/GIF output) — except when `TestEncoder.kt` mirrors them (synthetic symbols)
- `src/dom.ts`, `test/dom*.test.ts`, `test/encode.test.ts`, `test/gif.test.ts`, `test/polyfill.test.ts`, `test/decode-async.test.ts`
- `README.md`, `package.json`, config files - Non-code

**Scale check**: if `src/decode.ts` changed structurally (hundreds of lines, renamed/reordered
pipeline stages, new layer/scanner design — like the v0.7.2 rewrite), STOP the incremental flow:
tell the user it needs a written port plan (stage-by-stage, fixture-first) and do not try to port
it commit by commit.

Display summary to user:
```
Found 3 commits since last sync:

1. <hash> - Tighten alignment search radius
   - src/decode.ts L2041-2081 (findBasicAlign)
2. <hash> - Fix ECI 25 decoding
   - src/decode.ts L206-233 (ECI_ENCODINGS), test/decode.test.ts
3. <hash> - Release 0.7.3 (package.json only) — skip

Changes to port:
✓ findBasicAlign (decode.ts → Projection.kt QRScanner.findBasicAlign)
✓ ECI table (decode.ts → Payload.kt ECI_CHARSETS) + test (→ PayloadTest)
✓ Regenerate fixtures (tools/README.md)

Proceed with porting? (yes/no)
```

### Step 5: Port Changes

For each identified change:

1. **Read JavaScript diff**:
   ```bash
   git diff <old>..<new> -- src/decode.ts src/index.ts
   ```

2. **Identify target Kotlin file/function** with [FILE_MAPPING.md](FILE_MAPPING.md) (decode.ts
   line range → Layer/Finder/Selection/Projection/GridDecoder/RS/Payload/QRScanner/...). Every
   Kotlin function cites its source as `decode.ts Lxxx-yyy`; `grep -rn "decode.ts L" qr/src/main`
   finds it. Update those line references when the JS lines move.

3. **Read the current Kotlin code** next to the old and new JS.

4. **Translate line by line using [TRANSLATION_PATTERNS.md](TRANSLATION_PATTERNS.md)**: keep the
   JS expression shape and operand order (bit-exact doubles), `Math.round(x).toInt()` (never
   `kotlin.math.round`), `Homography.hypot/cbrt/toInt32`, NaN semantics for out-of-bounds typed-array
   reads, Fail tags instead of exceptions on the hot path. No defensive checks the JS lacks.

5. **Preserve Kotlin-only behaviour** (FILE_MAPPING.md "Kotlin-only"): gray/Y-plane input with
   offset/stride, the synchronous `QRScanner` API, the `QRDecoder` exception facade, ECI 2 via IBM437.

### Step 6: Sync Tests and Regenerate Fixtures

1. **Regenerate the JS parity fixtures** — this is the main test sync. From `qr/`, run the
   commands in `qr/tools/README.md` ("Regenerating everything") against `../qr-paulmillr`:
   `extract_decoded.py` (ImageVectors.kt), `dump-decode.mjs` / `--all` (decode-default/all.json),
   `dump-layers.mjs` / `--schedule` (dumps/layers, dumps/schedule), `luma-crc.mjs`
   (boofcv-luma-crc32.json), `math-fixtures.mjs`, `encoder-fixtures.mjs`. `git diff --stat src/test`
   then shows which pipeline stages changed upstream.

2. **Port hand-written JS tests** using [TEST_STRATEGY.md](TEST_STRATEGY.md):
   - `decode.test.ts` → `SyntheticTest`, `ImageDecodingTest`, `RealImageTest`, `QRScannerTest`, `QRDecoderTest`
   - `decode-batch.test.ts` → `QRScannerTest`
   - `utils.test.ts` → `QRInfoTest`, `RSTest`, `PayloadTest`
   - `test/misc/jpg.ts`, `png.ts` → `JpegLuma.kt`, `PngLuma.kt`
   - `assert.strictEqual(a, b)` → `assertEquals(b, a)` (**ORDER SWAPPED!**)

3. **Synchronize the test vectors submodule** — both repos must use the same commit:

   ```bash
   cd /Users/limp/dev/qr-root/qr-paulmillr && git submodule status test/vectors   # note the hash
   cd /Users/limp/dev/qr-root/qr && git submodule update --init --recursive
   git -C test/vectors checkout <commit-hash-from-js>
   ```

### Step 7: Build and Test

```bash
cd /Users/limp/dev/qr-root/qr
./gradlew test                    # full (~18 s)
./gradlew test -PskipExhaustive   # fast loop without the image/vector sweeps
```

**Goal: strict parity**, not a pass-rate threshold. Baseline (v0.7.2): small-vectors 9281/9281,
ImageDecodingTest 565/565 default and all mode against the regenerated JS dumps, DECODED table
311/326 = JS, LumaParityTest all CRCs, 0 failures overall (see TEST_STRATEGY.md).

Triage order when something fails:
1. `LumaParityTest` → JS image readers changed; port `jpg.ts`/`png.ts` first.
2. `LayerDumpTest` → pyramid/binarizer (`Layer.kt`).
3. `FinderDumpTest` → finder scan/refine/mandatory pick (`Finder.kt`, `Selection.kt`).
4. `ScheduleDumpTest` → retry schedule (`Selection.kt`).
5. `ImageDecodingTest` / `SyntheticTest` → projection, grid decode, RS, payload.

### Step 8a: Debug Algorithm for Test Parity Issues

When a test passes in JS but fails in Kotlin, use this systematic debugging approach:

**Algorithm**:

1. **Isolate failing test**
   - Identify the specific test case (e.g., vector index 359)
   - Create a minimal test file focusing on this case

2. **Add debug information to JS**
   ```javascript
   // In qr-paulmillr, create test/debug-case-359.test.js
   import { should } from '@paulmillr/jsbt/test.js';
   import readQR from '../src/decode.ts';

   should('debug case 359', () => {
     const img = ...; // Load the failing case
     console.log('Input image size:', img.width, img.height);
     console.log('Input image data (first 100):', img.data.slice(0, 100));

     const result = readQR(img);
     console.log('Result:', result);
     console.log('Result length:', result.length);
     console.log('Result bytes:', [...result].map(c => c.charCodeAt(0)));
   });
   ```

3. **Run JS debug test and collect output**
   ```bash
   cd /Users/limp/dev/qr-root/qr-paulmillr
   npm test -- test/debug-case-359.test.js
   ```
   Save output to a file for comparison.

4. **Create equivalent Kotlin debug test**
   ```kotlin
   // In qr/src/test/kotlin/qr/DebugCase359Test.kt
   import org.junit.jupiter.api.Test

   class DebugCase359Test {
       @Test
       fun `debug case 359`() {
           val img = ... // Load the same case
           println("Input image size: ${img.width} ${img.height}")
           println("Input image data (first 100): ${img.data.take(100)}")

           val result = QRDecoder.decode(img)
           println("Result: $result")
           println("Result length: ${result.length}")
           println("Result bytes: ${result.map { it.code }}")
       }
   }
   ```

5. **Run Kotlin debug test and collect output**
   ```bash
   cd /Users/limp/dev/qr-root/qr
   ./gradlew test --tests "DebugCase359Test"
   ```

6. **Compare debug outputs side by side**
   - Input validation: Same image dimensions and data?
   - Intermediate steps: Pattern detection, transformation, etc.
   - Output comparison: Character-by-character differences
   - Encoding: Check byte values (ECI encoding issue?)

7. **Identify differences**
   Common issues:
   - ECI encoding mismatch (wrong charset)
   - Numeric drift (operand order, rounding, hypot/cbrt, `| 0` on NaN) — see TRANSLATION_PATTERNS.md
   - Off-by-one errors in loops
   - Null handling differences
   - Integer division vs float division

8. **Fix the issue**
   Based on identified difference, apply fix to Kotlin code.

9. **Verify fix**
   ```bash
   ./gradlew test --tests "DebugCase359Test"
   ./gradlew test  # Full test suite
   ```

10. **Repeat for remaining failures**
    Continue with next failing test case.

**Example debug session**:

```
JS output:
  Input image size: 25 25
  Result: "Hello 世界"
  Result bytes: [72, 101, 108, 108, 111, 32, 228, 184, 150, 231, 149, 140]

Kotlin output:
  Input image size: 25 25
  Result: "Hello ???"
  Result bytes: [72, 101, 108, 108, 111, 32, 63, 63, 63]

→ Diagnosis: UTF-8 decoding issue (bytes 228, 184, 150 should decode to 世)
→ Fix: Check ECI encoding is set to UTF-8 (code 26)
```

### Step 8b: Auto-Fix Test Failures (General)

If tests fail, attempt automatic fixes (up to 3 iterations).

**Failure categories**:

1. **Off-by-One Errors** (ArrayIndexOutOfBoundsException):
   - Fix: Change `0..n` → `0 until n`, `start..end` → `start until end`

2. **Type Mismatches** (type inference errors):
   - Fix: Add `.toInt()`, `.toDouble()`, change array types

3. **Null Safety** (NullPointerException):
   - Fix: Add `?` to types, `!!` or `?.` to access, `?: defaultValue`

4. **Parity mismatch** (dump test or ImageDecodingTest differs):
   - Fix: diff the Kotlin function against its `decode.ts Lxxx-yyy` source token by token

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
- VectorTest: 9281/9281
- ImageDecodingTest: default 565/565, all 565/565 vs JS dumps; DECODED 311/326 = JS
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

**Quick reference** (v0.7.2 layout):
| JavaScript | Kotlin |
|------------|--------|
| decode.ts pyramid/binarizer (`scanRows.blocks/bitmap`) | Layer.kt |
| decode.ts finder scan/refine (`ratio`..`fitPattern`, `scanRows.find`) | Finder.kt |
| decode.ts triple selection, schedule (`excluded`..`pickPolarity`, `scan` schedule) | Selection.kt |
| decode.ts projection/alignment/timing/tiles | Projection.kt |
| decode.ts homography, `distance`, `snapSize` | Homography.kt |
| decode.ts `decodeGrid`/`decodeOriented`, `checkVersion` | GridDecoder.kt (RS corrector → RS.kt) |
| decode.ts `Payload`, `ECI_ENCODINGS` | Payload.kt |
| decode.ts `_QRScanner` (constructor, addImage, scan, decode) | QRScanner.kt |
| decode.ts `decodeQR` | QRDecoder.kt |
| index.ts spec tables, mask/format/version bits | QRInfo.kt |
| index.ts `EXP`/`LOG`, `rsGenerator` | GF256.kt, RS.kt |

## Translation Strategy

See [TRANSLATION_PATTERNS.md](TRANSLATION_PATTERNS.md) for detailed patterns.

**Key principles**:
1. Preserve algorithmic correctness above all
2. Maintain Kotlin coding style (see qr/CLAUDE.md)
3. Keep the JS expression shape and operand order (bit-exact doubles); every function cites `decode.ts Lxxx-yyy`
4. Preallocated scratch, typed-array-like IntArray/DoubleArray, Fail tags instead of exceptions on the hot path
5. No defensive checks the JS lacks; JS rounding/NaN/Int32 semantics via Homography helpers
6. Watch assertion order in tests (assertEquals(expected, actual))

## Test Strategy

See [TEST_STRATEGY.md](TEST_STRATEGY.md) for detailed guidelines.

**Key points**:
- Map test files correctly (decode.test.ts → multiple Kotlin tests)
- Translate assertions with correct order (assertEquals(expected, actual) in Kotlin)
- Ensure test vectors submodule is updated
- Regenerate the JS parity fixtures (qr/tools/README.md); strict parity, no pass-rate tolerance

## Scope Limitations

**Include (decode-only)**:
- Luma conversion, pyramid, binarizer, finder detection and selection
- Projection, alignment, sampling
- Grid decode, payload parsing, ECI
- Error correction (GF, RS)
- JS test image readers (test/misc/jpg.ts, png.ts)
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
