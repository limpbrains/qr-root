# JavaScript → Kotlin File Mapping

This document provides the complete correspondence between JavaScript source files in qr-paulmillr and Kotlin source files in qr.

## Source Files

| JavaScript File | Kotlin File(s) | Line Range | Notes |
|-----------------|----------------|------------|-------|
| `src/decode.ts` | `PatternDetector.kt` | Lines 1-500 (approx) | Pattern detection, toBitmap, findFinder, findAlignment, detect |
| `src/decode.ts` | `BitDecoder.kt` | Lines 501-end (approx) | decodeBitmap, readInfoBits, parseInfo, segment decoding |
| `src/index.ts` | `Transform.kt` | Transform class | Perspective transformation (squareToQuadrilateral, transform) |
| `src/index.ts` | `GaloisField.kt` | GaloisField class | GF(256) finite field arithmetic, lookup tables |
| `src/index.ts` | `ReedSolomon.kt` | ReedSolomon class | RS encoder/decoder |
| `src/index.ts` | `Bitmap.kt` | Bitmap class | 2D bitmap representation, get/set, drawing primitives |
| `src/index.ts` | `QRInfo.kt` | utils.info | Constants, capacity tables, alignment patterns, version/format bits |
| `src/index.ts` | `Interleave.kt` | utils.interleave | Block interleaving/de-interleaving |
| `src/index.ts` | `Types.kt` | Type definitions | Point, Pattern, ErrorCorrection, EncodingType, exceptions |
| N/A | `Image.kt` | Kotlin-specific | Image data class (Grayscale/RGB/RGBA support) |
| N/A | `QRDecoder.kt` | Kotlin-specific | Public API entry point |

## Excluded Files (Encoding/Browser-Only)

These JavaScript files are NOT ported to Kotlin (decode-only library):

| JavaScript File | Reason for Exclusion |
|-----------------|----------------------|
| `src/index.ts` (encode functions) | Kotlin is decode-only |
| `src/dom.ts` | Browser-specific (DOM, Canvas, Camera) |

## Test Files

| JavaScript Test | Kotlin Test(s) | Notes |
|-----------------|----------------|-------|
| `test/decode.test.ts` | `VectorTest.kt` | Vector-based decoding tests |
| `test/decode.test.ts` | `ImageDecodingTest.kt` | JPEG image decoding tests |
| `test/decode.test.ts` | `QRDecoderTest.kt` | API-level tests |
| `test/bitmap.test.ts` | `BitmapTest.kt` | Bitmap operations tests |
| `test/utils.test.ts` (info) | `QRInfoTest.kt` | QR constants/tables tests |
| `test/utils.test.ts` (GF) | `GaloisFieldTest.kt` | Galois Field tests |
| `test/utils.test.ts` (RS) | `ReedSolomonTest.kt` | Reed-Solomon tests |
| `test/qr.test.ts` | N/A (mostly encoding) | Skip encoding tests, port decode integration tests if any |
| `test/encode.test.ts` | N/A | Excluded (encoding only) |
| `test/dom.test.ts` | N/A | Excluded (browser only) |

## Function/Class Mapping

### Pattern Detection (decode.ts → PatternDetector.kt)

| JavaScript Function | Kotlin Function | Notes |
|---------------------|-----------------|-------|
| `toBitmap(img: Image)` | `PatternDetector.toBitmap()` | Adaptive thresholding |
| `findFinder(b: Bitmap)` | `PatternDetector.findFinder()` | Find 3 finder patterns |
| `findAlignment(b, bl, tl, tr)` | `PatternDetector.findAlignment()` | Find alignment pattern |
| `detect(img: Image)` | `PatternDetector.detect()` | Full detection pipeline |

### Data Decoding (decode.ts → BitDecoder.kt)

| JavaScript Function | Kotlin Function | Notes |
|---------------------|-----------------|-------|
| `readInfoBits(b: Bitmap, ...)` | `BitDecoder.readInfoBits()` | Read format/version bits |
| `parseInfo(...)` | `BitDecoder.parseInfo()` | Parse ECC/version/mask |
| `decodeBitmap(b: Bitmap)` | `BitDecoder.decodeBitmap()` | Full bitmap decode |
| `decodeWithEci(bytes, eci)` | `BitDecoder.decodeWithEci()` | ECI character encoding |

### Transformation (index.ts → Transform.kt)

| JavaScript Function | Kotlin Function | Notes |
|---------------------|-----------------|-------|
| `squareToQuadrilateral(...)` | `Transform.squareToQuadrilateral()` | Perspective transform matrix |
| `transform(from, to, w, h)` | `Transform.transform()` | Apply transformation |

### Galois Field (index.ts → GaloisField.kt)

| JavaScript Property/Method | Kotlin Property/Method | Notes |
|---------------------------|------------------------|-------|
| `GF.exp(i)` | `GaloisField.exp(i)` | Exponentiation table lookup |
| `GF.log(x)` | `GaloisField.log(x)` | Logarithm table lookup |
| `GF.mul(a, b)` | `GaloisField.mul(a, b)` | Multiply in GF(256) |
| `GF.add(a, b)` | `GaloisField.add(a, b)` | Add in GF(256) (XOR) |
| `GF.inv(x)` | `GaloisField.inv(x)` | Multiplicative inverse |
| `GF.pow(base, exp)` | `GaloisField.pow(base, exp)` | Exponentiation |

### Reed-Solomon (index.ts → ReedSolomon.kt)

| JavaScript Class/Method | Kotlin Class/Method | Notes |
|------------------------|---------------------|-------|
| `new ReedSolomon(eccWords)` | `ReedSolomon(eccWords)` | Constructor |
| `rs.encode(data)` | `rs.encode(data)` | Generate error correction codes |
| `rs.decode(data)` | `rs.decode(data)` | Correct errors |

### Bitmap (index.ts → Bitmap.kt)

| JavaScript Property/Method | Kotlin Property/Method | Notes |
|---------------------------|------------------------|-------|
| `b.data[y][x]` | `b.get(x, y)` | **CRITICAL: x and y are SWAPPED!** |
| `b.data[y][x] = val` | `b.set(x, y, val)` | **CRITICAL: x and y are SWAPPED!** |
| `b.width` | `b.width` | Same |
| `b.height` | `b.height` | Same |
| `new Bitmap(w, h)` | `Bitmap(w, h)` | Constructor |
| `b.negate()` | `b.negate()` | Invert all bits (for inverted QR) |
| `b.slice(x, y, w, h)` | `b.slice(x, y, w, h)` | Extract sub-bitmap |

### Interleave (index.ts → Interleave.kt)

| JavaScript Function | Kotlin Function | Notes |
|---------------------|-----------------|-------|
| `interleave(version, ecc)` | `Interleave(version, ecc)` | Constructor |
| `interleave.deinterleave(data)` | `interleave.deinterleave(data)` | Split into blocks |
| `interleave.interleave(blocks)` | `interleave.interleave(blocks)` | Merge blocks |

### QR Info (index.ts → QRInfo.kt)

| JavaScript Property/Function | Kotlin Property/Function | Notes |
|-----------------------------|--------------------------|-------|
| `utils.info.capacity(version, ecc)` | `QRInfo.capacity(version, ecc)` | Get data capacity |
| `utils.info.sizeEncode(version)` | `QRInfo.sizeEncode(version)` | QR size from version |
| `utils.info.alignmentPattern(version)` | `QRInfo.alignmentPattern(version)` | Alignment positions |
| `utils.info.MASK_PATTERNS` | `QRInfo.MASK_PATTERNS` | 8 mask patterns |

### Types (index.ts → Types.kt)

| JavaScript Type | Kotlin Type | Notes |
|-----------------|-------------|-------|
| `Point` interface | `data class Point` | x, y coordinates |
| `Pattern` interface | `data class Pattern` | x, y, moduleSize |
| `ErrorCorrection` type | `enum class ErrorCorrection` | L, M, Q, H |
| `EncodingType` type | `enum class EncodingType` | NUMERIC, ALPHANUMERIC, BYTE |
| Error classes | Exception hierarchy | QRDecodingException, FinderNotFoundException, etc. |

## Critical Coordinate Swap Warning

**MOST COMMON SOURCE OF BUGS**: Bitmap array access order is different!

### JavaScript (row-major with [y][x])
```javascript
if (b.data[y][x]) { ... }
b.data[y][x] = true;
```

### Kotlin (column-major with get(x, y))
```kotlin
if (b.get(x, y) == true) { ... }
b.set(x, y, true)
```

**Rule**: When porting, ALWAYS swap the x and y parameters!

### Pattern to Watch For

In JavaScript, you'll often see nested loops like:
```javascript
for (let y = 0; y < height; y++) {
  for (let x = 0; x < width; x++) {
    if (b.data[y][x]) { ... }  // y first!
  }
}
```

In Kotlin, this becomes:
```kotlin
for (y in 0 until height) {
  for (x in 0 until width) {
    if (b.get(x, y) == true) { ... }  // x first!
  }
}
```

## Line Number Approximations

These are approximate line ranges for decode.ts (may shift with updates):

| Section | JavaScript Lines | Kotlin File | Function |
|---------|------------------|-------------|----------|
| Imports & types | 1-50 | Types.kt | Type definitions |
| toBitmap | 50-200 | PatternDetector.kt | Adaptive thresholding |
| findFinder | 200-500 | PatternDetector.kt | Pattern detection |
| findAlignment | 500-600 | PatternDetector.kt | Alignment detection |
| detect (main) | 600-700 | PatternDetector.kt | Detection pipeline |
| readInfoBits | 700-800 | BitDecoder.kt | Read format/version |
| parseInfo | 800-850 | BitDecoder.kt | Parse info bits |
| decodeBitmap | 850-end | BitDecoder.kt | Decode data |

**Note**: Always use `git diff` to see actual changed lines, don't rely on line numbers alone.

## Usage During Sync

When processing a change:

1. **Identify source file** from git diff (e.g., `src/decode.ts`)
2. **Look up in this mapping** to find target Kotlin file(s)
3. **Check line range** if multiple targets (PatternDetector vs BitDecoder)
4. **Apply function mapping** to find specific Kotlin function
5. **Remember coordinate swap** for Bitmap operations
6. **Port the change** to identified Kotlin location

## Kotlin-Only Enhancements to Preserve

These features exist ONLY in Kotlin - preserve them during sync:

| Feature | Location | Description |
|---------|----------|-------------|
| Grayscale input | Image.kt, PatternDetector.kt | Direct Y-plane processing (3-5x faster) |
| Triangle validation | PatternDetector.kt:384-401 | Validate finder patterns form valid triangle |
| Threshold retry | QRDecoder.kt:21 | Retry with offsets [-5, 0, 5] |
| Relaxed finder variance | PatternDetector.kt:229-246 | Progressive variance relaxation (2.0 → 2.5 → 3.0) |

When porting changes that touch these areas, ensure Kotlin enhancements are NOT removed or broken.
