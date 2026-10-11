# Test Synchronization Strategy

This document describes how to synchronize test cases between the JavaScript QR library (qr-paulmillr) and the Kotlin port (qr).

## Test File Correspondence

| JavaScript test | Kotlin test(s) | Description |
|---|---|---|
| `test/decode.test.ts` (DECODED table, image sweeps) | `ImageDecodingTest`, `RealImageTest` | Strict per-image parity with JS dumps; generated `ImageVectors.kt` |
| `test/decode.test.ts` (synthetic symbols, options, errors) | `SyntheticTest`, `QRScannerTest`, `QRDecoderTest` | TestEncoder-built symbols, option validation, facade exceptions |
| `test/decode-batch.test.ts` | `QRScannerTest` | Scanner reuse, all mode, batch order |
| `test/utils.test.ts` | `QRInfoTest`, `RSTest`, `PayloadTest` | Spec tables, RS corrector, segment parser |
| `test/jpg.test.ts`, `test/png.test.ts` (+ `test/misc/jpg.ts`, `png.ts`) | `LumaParityTest` (+ `JpegLuma`, `PngLuma`) | Test-only image readers, luma CRC guard |
| qr-code-vectors `small-vectors.json.gz` | `VectorTest` | All 9281 must decode |
| JS stage dumps (`qr/tools/js-dump`) | `LayerDumpTest`, `FinderDumpTest`, `ScheduleDumpTest`, `HomographyTest`, `TestEncoderTest` | Stage-level bit-exact parity |
| `test/decode-async.test.ts`, `dom*.test.ts`, `encode.test.ts`, `gif.test.ts`, `polyfill.test.ts`, `qr.test.ts` | — | Not ported (async, browser, encoding) |

## Parity Workflow (primary)

The Kotlin decoder is checked against **fixtures dumped from the JS sources**, not just against
hand-ported assertions. After bumping `qr-paulmillr`:

1. Regenerate every fixture with the commands in `qr/tools/README.md` (node >= 22,
   `--experimental-strip-types`; `JS=../qr-paulmillr`). This covers `ImageVectors.kt`
   (`extract_decoded.py`), `dumps/decode-default.json` + `decode-all.json` (`dump-decode.mjs`),
   `dumps/layers` + `dumps/schedule` (`dump-layers.mjs`, `--schedule`), `boofcv-luma-crc32.json`
   (`luma-crc.mjs`), V8 math and encoder fixtures.
2. `git -C qr diff --stat src/test` shows which stages changed upstream; port the matching JS
   ranges (FILE_MAPPING.md) until `./gradlew test` is green again.
3. **Luma CRC guard first**: if `LumaParityTest` fails, the JS test image readers changed — port
   `test/misc/jpg.ts` / `png.ts` into `JpegLuma`/`PngLuma` before touching the decoder (every other
   image fixture depends on identical luma).
4. **Strict table**: `ImageDecodingTest` fails on any mismatch (text or fail tag, default and all
   mode, 565 images; DECODED first-ok/all-set table). Its `KNOWN_DIVERGENCES` hook must stay empty
   unless the user accepts a deviation (document it in qr/CLAUDE.md "Accepted Deviations").
5. Hand-written JS tests (decode.test.ts, decode-batch.test.ts, utils.test.ts) are ported with the
   assertion patterns below.

`./gradlew test -PskipExhaustive` skips the slow image/vector sweeps for quick iteration;
`-PonlyExhaustive` runs only them.

## Test Framework Differences

### JavaScript (Node.js Test Runner)

```javascript
import { describe, it } from 'node:test';
import assert from 'node:assert/strict';

describe('QR Decoder', () => {
  describe('pattern detection', () => {
    it('should find finder patterns', () => {
      const result = decode(img);
      assert.strictEqual(result, 'Hello world');
    });
  });
});
```

### Kotlin (JUnit 5)

```kotlin
import org.junit.jupiter.api.*
import org.junit.jupiter.api.Assertions.*

class QRDecoderTest {
  @Nested
  inner class PatternDetection {
    @Test
    fun `should find finder patterns`() {
      val result = QRDecoder.decode(img)
      assertEquals("Hello world", result)
    }
  }
}
```

## Assertion Translation

### Basic Assertions

| JavaScript | Kotlin | Notes |
|------------|--------|-------|
| `assert.strictEqual(actual, expected)` | `assertEquals(expected, actual)` | **ORDER SWAPPED!** |
| `assert.notStrictEqual(actual, notExpected)` | `assertNotEquals(notExpected, actual)` | **ORDER SWAPPED!** |
| `assert.ok(condition)` | `assertTrue(condition)` | |
| `assert.ok(condition, 'message')` | `assertTrue(condition, "message")` | |
| `assert.equal(actual, expected)` | `assertEquals(expected, actual)` | **ORDER SWAPPED!** |
| `assert.deepStrictEqual(actual, expected)` | `assertEquals(expected, actual)` | Works for data classes |

**CRITICAL**: JUnit has `assertEquals(expected, actual)`, but JavaScript assert has `assert.strictEqual(actual, expected)`. The parameter order is SWAPPED!

### Exception Assertions

| JavaScript | Kotlin |
|------------|--------|
| `assert.throws(() => decode(img))` | `assertThrows<QRDecodingException> { decode(img) }` |
| `assert.throws(() => decode(img), /pattern/)` | `val ex = assertThrows<QRDecodingException> { decode(img) }`<br>`assertTrue(ex.message!!.contains("pattern"))` |
| `assert.doesNotThrow(() => decode(img))` | `assertDoesNotThrow { decode(img) }` |

### Collection Assertions

| JavaScript | Kotlin |
|------------|--------|
| `assert.deepStrictEqual(arr, [1, 2, 3])` | `assertEquals(listOf(1, 2, 3), arr.toList())` |
| `assert.strictEqual(arr.length, 3)` | `assertEquals(3, arr.size)` |

## Test Structure Translation

### Test Suites (describe → class)

```javascript
// JavaScript
describe('QR Decoder', () => {
  describe('ECI encoding', () => {
    it('should decode UTF-8', () => { ... });
    it('should decode ISO-8859-1', () => { ... });
  });
});
```

```kotlin
// Kotlin
class QRDecoderTest {
  @Nested
  inner class EciEncoding {
    @Test
    fun `should decode UTF-8`() { ... }

    @Test
    fun `should decode ISO-8859-1`() { ... }
  }
}
```

### Setup and Teardown

```javascript
// JavaScript
beforeEach(() => {
  // setup
});

afterEach(() => {
  // cleanup
});
```

```kotlin
// Kotlin
@BeforeEach
fun setup() {
  // setup
}

@AfterEach
fun teardown() {
  // cleanup
}
```

### Parameterized Tests

```javascript
// JavaScript
[1, 2, 3, 4, 5].forEach(version => {
  it(`should work for version ${version}`, () => {
    const result = decodeVersion(version);
    assert.ok(result);
  });
});
```

```kotlin
// Kotlin - Option 1: @ParameterizedTest
@ParameterizedTest
@ValueSource(ints = [1, 2, 3, 4, 5])
fun `should work for version {0}`(version: Int) {
  val result = decodeVersion(version)
  assertTrue(result)
}

// Kotlin - Option 2: Manual loop
@Test
fun `should work for all versions`() {
  for (version in 1..5) {
    val result = decodeVersion(version)
    assertTrue(result, "Failed for version $version")
  }
}
```

## Test Data Synchronization

### Test Vectors (Git Submodule)

Both JavaScript and Kotlin repositories use the same test vectors from a git submodule:

**Location**: `test/vectors/` (submodule: `paulmillr/qr-code-vectors`)

**Files**:
- `small-vectors.json.gz` - Compressed ASCII-art QR codes
- `boofcv-v3/` - JPEG images for real-world testing

**Update vectors**:
```bash
cd /Users/limp/dev/qr-root/qr
git submodule update --remote test/vectors
```

### Loading Test Data

```javascript
// JavaScript - uses streaming JSON parser
import { jsonGZ } from './utils.ts';
const vectors = await jsonGZ('./vectors/small-vectors.json.gz');
```

```kotlin
// Kotlin - uses streaming JSON parser
val vectors = loadVectors("test/vectors/small-vectors.json.gz")
```

Both use streaming parsers to avoid loading 100MB+ file into memory.

## Common Test Patterns

### Pattern 1: Simple Decode Test

```javascript
// JavaScript
it('should decode simple QR', () => {
  const img = loadImage('test.jpg');
  const result = decode(img);
  assert.strictEqual(result, 'Hello world');
});
```

```kotlin
// Kotlin
@Test
fun `should decode simple QR`() {
  val img = loadImage("test.jpg")
  val result = QRDecoder.decode(img)
  assertEquals("Hello world", result)
}
```

**Note**: assertEquals has expected first!

### Pattern 2: Exception Test

```javascript
// JavaScript
it('should throw for invalid image', () => {
  assert.throws(() => {
    decode(emptyImage);
  }, /ImageTooSmallException/);
});
```

```kotlin
// Kotlin
@Test
fun `should throw for invalid image`() {
  val exception = assertThrows<ImageTooSmallException> {
    QRDecoder.decode(emptyImage)
  }
  assertTrue(exception.message!!.contains("too small"))
}
```

### Pattern 3: Vector Test Loop

```javascript
// JavaScript
describe('vector tests', () => {
  vectors.forEach(({ input, expected }) => {
    it(`should decode: ${expected.substring(0, 20)}`, () => {
      const result = decode(input);
      assert.strictEqual(result, expected);
    });
  });
});
```

```kotlin
// Kotlin - use @TestFactory for dynamic tests
@TestFactory
fun `vector tests`() = vectors.map { (input, expected) ->
  DynamicTest.dynamicTest("should decode: ${expected.take(20)}") {
    val result = QRDecoder.decode(input)
    assertEquals(expected, result)
  }
}

// Or use single test with loop
@Test
fun `vector tests`() {
  var passed = 0
  var failed = 0

  vectors.forEach { (input, expected) ->
    try {
      val result = QRDecoder.decode(input)
      assertEquals(expected, result)
      passed++
    } catch (e: Exception) {
      failed++
    }
  }

  println("Passed: $passed/${vectors.size}")
  assertTrue(passed >= vectors.size * 0.98, "Pass rate too low: $passed/${vectors.size}")
}
```

## Test Coverage Expectations (baseline, v0.7.2, 2026-10-11)

| Suite | Baseline | Acceptable |
|---|---|---|
| VectorTest (small-vectors) | 9281/9281 | 9281/9281 |
| ImageDecodingTest default mode vs `decode-default.json` | 565/565 | 565/565 |
| ImageDecodingTest all mode vs `decode-all.json` | 565/565 | 565/565 |
| DECODED table (536 detection images) | default 311 = JS 311; all 326 = JS 326 | equal to JS |
| LumaParityTest | 539/539 CRCs | all |
| `./gradlew test` | 748 tests, 0 failures (~18 s) | 0 failures |

Any drop is a porting bug (or an upstream change whose fixtures were not regenerated).

## Auto-Fix Test Failures

When tests fail after porting, attempt automatic fixes in this order:

### Fix Category 1: Assertion Order

**Symptom**:
```
Expected: <actual value>
Actual: <expected value>
```

**Cause**: assertEquals parameters swapped

**Fix**: Swap parameters in assertEquals call

### Fix Category 2: Off-by-One Errors

**Symptom**:
```
ArrayIndexOutOfBoundsException: Index 10 out of bounds for length 10
```

**Cause**: Loop bound using `0..n` instead of `0 until n`

**Fix**: Change inclusive range to exclusive:
- `for (i in 0..n)` → `for (i in 0 until n)`
- `arr.slice(start..end)` → `arr.slice(start until end)`

### Fix Category 3: Type Mismatches

**Symptom**:
```
Type mismatch: inferred type is Double but Int was expected
```

**Fix**: Add type conversion:
- Add `.toInt()` or `.toDouble()`
- Change array type (IntArray vs DoubleArray)

### Fix Category 4: Null Safety Errors

**Symptom**:
```
NullPointerException
```

**Fix**:
- Add `?` to nullable types
- Add `!!` for non-null assertion (if truly non-null)
- Add `?.` for safe navigation
- Add `?: defaultValue` for default

### Fix Category 5: Numeric Drift (parity fixtures)

**Symptom**: a `LayerDumpTest`/`FinderDumpTest`/`ScheduleDumpTest` value differs in the last digits,
or ImageDecodingTest differs on a handful of images.

**Cause**: an expression whose rounding differs from JS: reordered operands, `kotlin.math.round`,
`Math.hypot`/`cbrt` from java.lang, `x.toInt()` where JS uses `| 0` on NaN/Infinity, a missing
out-of-bounds `undefined` -> NaN replication. See TRANSLATION_PATTERNS.md.

**Fix**: diff the Kotlin function against its `decode.ts Lxxx-yyy` source token by token.

### Fix Category 6: Character Encoding

**Symptom**:
```
Expected: "日本語"
Actual: "æ—¥æœ¬èª??"
```

**Cause**: Missing or incorrect ECI encoding support

**Fix**:
- Check the ECI number is mapped in `Payload.ECI_CHARSETS`
- Add missing encodings to `Payload.ECI_CHARSETS`
- Verify charset names are correct

## Auto-Fix Process

1. **Run tests**:
   ```bash
   cd /Users/limp/dev/qr-root/qr
   ./gradlew test
   ```

2. **Parse output**: Extract failure details (test name, expected, actual, stack trace)

3. **Categorize failures**: Match symptoms to fix categories

4. **Apply fixes**: Edit Kotlin test files

5. **Re-run tests**: `./gradlew test`

6. **Check progress**:
   - Count failures before and after
   - If failures decreased: continue
   - If failures same: try different strategy
   - If failures increased: revert changes

7. **Iterate**: Repeat up to 3 times

8. **Report results**:
   ```
   Test Synchronization Results:

   ✓ Ported 15 new test cases
   ✓ VectorTest: 9281/9281 - maintained
   ✗ ImageDecodingTest: 110/118 (93.22%) - 3 regressions

   Remaining failures:
   1. testEciShiftJis: Expected '日本語', got 'æ—¥æœ¬èª?'
      → Add ECI code 20 (shift-jis) to encoding map

   2. testRotatedQR: ArrayIndexOutOfBoundsException at line 45
      → Likely off-by-one in loop bounds
   ```

## Testing After Porting Code Changes

When you port a code change (not a test change), still run tests:

1. **Port code change** from JS to Kotlin

2. **Run existing tests**:
   ```bash
   ./gradlew test
   ```

3. **Check for regressions**:
   - If all tests pass: ✓ Success
   - If new failures appear: Port likely has bugs

4. **Fix regressions** using auto-fix strategies

5. **Verify fix preserves correctness**: Review ported code carefully

## Imports Needed for Kotlin Tests

```kotlin
import org.junit.jupiter.api.*
import org.junit.jupiter.api.Assertions.*
import org.junit.jupiter.api.DynamicTest
```

For parameterized tests:
```kotlin
import org.junit.jupiter.params.ParameterizedTest
import org.junit.jupiter.params.provider.ValueSource
import org.junit.jupiter.params.provider.CsvSource
```

## Test Naming Conventions

**Kotlin uses backtick syntax** for descriptive test names:

```kotlin
@Test
fun `should decode QR with ECI encoding`() { ... }

@Test
fun `should throw FinderNotFoundException when no patterns found`() { ... }
```

This allows spaces and special characters in test names, making them more readable.

## Summary Checklist

When porting tests:

- [ ] Map JavaScript test file to correct Kotlin test file(s)
- [ ] Translate test structure (describe → class, it → @Test)
- [ ] Translate assertions with **correct parameter order** (expected first!)
- [ ] Handle exception tests with assertThrows<Type> { }
- [ ] Update test vectors submodule if needed
- [ ] Regenerate fixtures (qr/tools/README.md) and run tests: every parity suite at baseline
- [ ] Apply auto-fixes for common failures
- [ ] Report any remaining issues with diagnostic info
