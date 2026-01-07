# Test Synchronization Strategy

This document describes how to synchronize test cases between the JavaScript QR library (qr-paulmillr) and the Kotlin port (qr).

## Test File Correspondence

| JavaScript Test File | Kotlin Test File(s) | Description |
|---------------------|---------------------|-------------|
| `test/decode.test.ts` | `VectorTest.kt` | Vector-based integration tests using ASCII-art QR codes |
| `test/decode.test.ts` | `ImageDecodingTest.kt` | JPEG image decoding tests (BoofCV dataset) |
| `test/decode.test.ts` | `QRDecoderTest.kt` | Public API tests, format detection |
| `test/bitmap.test.ts` | `BitmapTest.kt` | Bitmap class operations |
| `test/utils.test.ts` | `QRInfoTest.kt` | QR constants, version info, alignment patterns |
| `test/utils.test.ts` | `GaloisFieldTest.kt` | GF(256) arithmetic tests |
| `test/utils.test.ts` | `ReedSolomonTest.kt` | Reed-Solomon error correction tests |
| `test/qr.test.ts` | N/A (mostly encoding) | Skip encoding tests |
| `test/encode.test.ts` | N/A | Excluded (encoding only) |
| `test/dom.test.ts` | N/A | Excluded (browser only) |

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

### Pattern 4: Bitmap Test

```javascript
// JavaScript
it('should get and set bitmap values', () => {
  const b = new Bitmap(10, 10);
  b.data[5][3] = true;
  assert.strictEqual(b.data[5][3], true);
});
```

```kotlin
// Kotlin - remember x/y swap!
@Test
fun `should get and set bitmap values`() {
  val b = Bitmap(10, 10)
  b.set(3, 5, true)  // x=3, y=5 (swapped!)
  assertEquals(true, b.get(3, 5))
}
```

**CRITICAL**: `b.data[y][x]` becomes `b.get(x, y)` - x and y are swapped!

## Test Coverage Expectations

After synchronizing tests, Kotlin should maintain similar pass rates:

| Test Suite | Baseline Pass Rate | Acceptable Range |
|------------|-------------------|------------------|
| VectorTest (small-vectors) | 98.42% (9134/9281) | ≥ 98% |
| ImageDecodingTest (boofcv) | 95.76% (113/118) | ≥ 95% |

**If pass rate drops below acceptable range**:
- Investigate failures before committing
- Check for porting errors (x/y swap, off-by-one, type mismatches)
- Verify test data is up to date

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

### Fix Category 5: Bitmap Coordinate Errors

**Symptom**:
```
Expected: "Hello world"
Actual: "Garbled text" or wrong output
```

**Cause**: Bitmap get/set calls with swapped x/y

**Fix**: Search for `b.get(` and `b.set(` calls, verify order:
- Should be `get(x, y)` not `get(y, x)`
- Check surrounding loops to determine correct order

### Fix Category 6: Character Encoding

**Symptom**:
```
Expected: "日本語"
Actual: "æ—¥æœ¬èª??"
```

**Cause**: Missing or incorrect ECI encoding support

**Fix**:
- Ensure `decodeWithEci()` function is ported
- Add missing encodings to `eciToEncoding` map
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
   ✓ VectorTest: 9134/9281 (98.42%) - maintained
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
- [ ] Run tests and verify pass rate ≥ baseline
- [ ] Apply auto-fixes for common failures
- [ ] Report any remaining issues with diagnostic info
