# JavaScript → Kotlin Translation Patterns

This document provides comprehensive patterns for translating JavaScript/TypeScript code to Kotlin while preserving algorithmic correctness and maintaining Kotlin coding style.

## Type System

| JavaScript | Kotlin | Notes |
|------------|--------|-------|
| `number` | `Int` | For integers, array indices, counts |
| `number` | `Double` | For floating-point calculations, coordinates |
| `boolean` | `Boolean` | Same concept |
| `string` | `String` | Same concept |
| `Uint8Array` | `ByteArray` | 8-bit unsigned → signed bytes |
| `Uint32Array` | `IntArray` | 32-bit unsigned → signed ints |
| `number[]` | `IntArray` or `DoubleArray` | Use typed arrays for performance |
| `T[]` | `Array<T>` or `List<T>` | Generic arrays |
| `{ x: number, y: number }` record | preallocated mutable class (`Finder.Pattern`) or `DoubleArray` slots | The decoder reuses scratch objects; do not allocate per call |
| `type Union = A \| B` | `sealed class` or nullable | Context-dependent |
| `undefined` | `null` | Kotlin has no undefined |
| `T \| undefined` | `T?` | Nullable type |

## Function Definitions

| JavaScript | Kotlin |
|------------|--------|
| `function foo(x: number): number { return x * 2; }` | `fun foo(x: Int): Int = x * 2` |
| `const foo = (x: number) => x * 2` | `val foo = { x: Int -> x * 2 }` or `fun foo(x: Int) = x * 2` |
| `function foo(x?: number) { ... }` | `fun foo(x: Int? = null) { ... }` |
| `function foo(x: number = 10) { ... }` | `fun foo(x: Int = 10) { ... }` |
| `function foo(...args: number[]) { ... }` | `fun foo(vararg args: Int) { ... }` |

## Control Flow

| JavaScript | Kotlin |
|------------|--------|
| `if (cond) { ... } else { ... }` | `if (cond) { ... } else { ... }` |
| `const x = cond ? a : b` | `val x = if (cond) a else b` |
| `switch (x) { case 1: ...; break; }` | `when (x) { 1 -> ... }` |
| `for (let i = 0; i < n; i++)` | `for (i in 0 until n)` |
| `for (let i = n; i >= 0; i--)` | `for (i in n downTo 0)` |
| `for (let i = 0; i < n; i += 2)` | `for (i in 0 until n step 2)` |
| `while (cond) { ... }` | `while (cond) { ... }` |
| `do { ... } while (cond)` | `do { ... } while (cond)` |

## Arrays and Collections

| JavaScript | Kotlin |
|------------|--------|
| `new Uint8Array(n)` (pixel/luma data) | `ByteArray(n)`; read as `b.toInt() and 0xFF` |
| `new Uint8Array(n)` (small values, codewords) | `IntArray(n)`; store `v and 0xFF` if the JS store could wrap |
| `new Int16Array(n)` | `IntArray(n)`; store `(v shl 16) shr 16` when the value can leave Int16 range, NaN -> 0 |
| `new Int32Array(n)` / packed bitmap | `IntArray(n)` (JS Int32 bit ops == Kotlin Int bit ops) |
| `new Float64Array(n)` | `DoubleArray(n)` |
| typed-array read out of bounds (`undefined`) | Replicate explicitly: arithmetic -> `Double.NaN`, comparisons false, store -> 0. Kotlin throws, so guard the index and return the JS result |
| `new Array(n).fill(0)` | `IntArray(n)` or `Array(n) { 0 }` |
| `arr[i]` | `arr[i]` |
| `arr.length` | `arr.size` |
| `arr.push(x)` | `arr.add(x)` (MutableList only) |
| `arr.pop()` | `arr.removeAt(arr.size - 1)` |
| `arr.slice(start, end)` | `arr.sliceArray(start until end)` or `arr.slice(start until end)` |
| `arr.concat(other)` | `arr + other` |
| `arr.forEach(x => ...)` | `arr.forEach { x -> ... }` |
| `arr.map(x => ...)` | `arr.map { x -> ... }` |
| `arr.filter(x => ...)` | `arr.filter { x -> ... }` |
| `arr.reduce((acc, x) => ..., init)` | `arr.fold(init) { acc, x -> ... }` |
| `arr.find(x => ...)` | `arr.find { x -> ... }` |
| `arr.some(x => ...)` | `arr.any { x -> ... }` |
| `arr.every(x => ...)` | `arr.all { x -> ... }` |

## Bitwise Operations

| JavaScript | Kotlin |
|------------|--------|
| `a & b` | `a and b` |
| `a \| b` | `a or b` |
| `a ^ b` | `a xor b` |
| `~a` | `a.inv()` |
| `a << b` | `a shl b` |
| `a >> b` | `a shr b` (signed) |
| `a >>> b` | `a ushr b` (unsigned) |
| `n >>> 0` (convert to unsigned) | Double: `.toInt()` truncation (non-finite -> 0); Int: `toLong() and 0xFFFF_FFFFL` only if a Long is really needed |
| `(a & 0xFF)` (mask to byte) | `a and 0xFF` |

## Math Operations

| JavaScript | Kotlin |
|------------|--------|
| `Math.sqrt(x)` | `sqrt(x)` (import kotlin.math.sqrt) |
| `Math.floor(x)` | `floor(x)` (import kotlin.math.floor) |
| `Math.ceil(x)` | `ceil(x)` (import kotlin.math.ceil) |
| `Math.round(x)` | `Math.round(x).toInt()` (java.lang; ties toward +inf like JS). **Never** `kotlin.math.round` (ties to even). Non-finite/large input: `Homography.jsRound(x)` |
| `Math.min(a, b)` | `min(a, b)` (import kotlin.math.min) |
| `Math.max(a, b)` | `max(a, b)` (import kotlin.math.max) |
| `Math.abs(x)` | `abs(x)` (import kotlin.math.abs) |
| `Math.pow(x, y)` | `x.pow(y)` (import kotlin.math.pow) |
| `x ** 2` on doubles | `x * x` (bit-identical; V8 pow special-cases 2) |
| `Math.hypot(a, b)` | `Homography.hypot(a, b)` (V8 port; `kotlin.math.hypot`/`sqrt(a*a+b*b)` differ in the last ulp) |
| `Math.cbrt(x)` | `Homography.cbrt(x)` (V8/fdlibm port; `StrictMath.cbrt` differs on ~8% of inputs) |
| `x \| 0` on a double | `Homography.toInt32(x)` (NaN/Inf -> 0, truncate, wrap mod 2^32); `x.toInt()` only when provably finite and in Int range |
| `x >>> 0` on a double | NaN -> 0, else truncate/wrap to UInt32 (e.g. `if (v.isNaN()) 0 else v.toInt()` for 0..255 values) |
| `Infinity` (number) | `Double.POSITIVE_INFINITY`; as an option/count (`effort`, `nativeLimit`) `Int.MAX_VALUE` |
| JS truthiness of a number | `v != 0.0 && !v.isNaN()` |

## String Operations

| JavaScript | Kotlin |
|------------|--------|
| `str.length` | `str.length` |
| `str.charAt(i)` | `str[i]` |
| `str.substring(start, end)` | `str.substring(start, end)` |
| `str.substring(start)` | `str.substring(start)` |
| `str.slice(start, end)` | `str.substring(start, end)` |
| `str.split(sep)` | `str.split(sep)` |
| `str.indexOf(sub)` | `str.indexOf(sub)` |
| `str.startsWith(prefix)` | `str.startsWith(prefix)` |
| `str.endsWith(suffix)` | `str.endsWith(suffix)` |
| `str.toUpperCase()` | `str.uppercase()` |
| `str.toLowerCase()` | `str.lowercase()` |
| `'value: ' + x` | `"value: $x"` or `"value: ${x}"` |

## Object Operations

| JavaScript | Kotlin |
|------------|--------|
| `{ x: 1, y: 2 }` | write into a preallocated mutable object (`Finder.Pattern`) or scratch array slots; no per-call allocation |
| `obj.x` | `obj.x` |
| `obj['key']` | `obj["key"]` (if map) |
| `Object.keys(obj)` | `obj.keys` (if map) |
| `Object.values(obj)` | `obj.values` (if map) |

## Exceptions

| JavaScript | Kotlin |
|------------|--------|
| `throw new Error('msg')` | `throw Exception("msg")` or custom exception |
| `try { ... } catch (e) { ... }` | `try { ... } catch (e: Exception) { ... }` |
| `try { ... } finally { ... }` | `try { ... } finally { ... }` |

## Common Patterns

### Pattern 1: Error values, not exceptions, on the hot path

The v0.7.2 decoder returns failure values (`Attempt = string | Error`, `FAIL.*`) instead of
throwing inside the scan loop. Kotlin mirrors that: stage functions return `String?` (or a
Boolean) and set `QRScanner.fail` to a `Fail` tag; only the `QRDecoder` facade turns the terminal
tag into an exception. Never introduce try/catch control flow in Layer/Finder/Selection/Projection/
GridDecoder/RS/Payload.

```javascript
if (!this.timing(s, map, size)) return FAIL.timing;
```
```kotlin
if (!timing(s, map, size)) { fail = Fail.TIMING; return null }
```

### Pattern 2: Keep the JS expression shape (bit-exact doubles)

Floating-point results are compared bit-for-bit against JS dumps. Keep operand order and
grouping exactly (`a * b + c * d`, not a refactored form), do not fuse into `Math.fma`, do not
replace `/ 2` with `* 0.5` or hoist common subexpressions that change rounding. Every ported
function carries a `// decode.ts Lxxx-yyy` comment.

### Pattern 3: Null Safety

```javascript
// JavaScript - undefined checking
let x: number | undefined;
if (x !== undefined) { use(x); }
const y = x !== undefined ? x : defaultValue;
```

```kotlin
// Kotlin - null safety
var x: Int? = null
if (x != null) { use(x) }
// or
x?.let { use(it) }
// Elvis operator
val y = x ?: defaultValue
```

### Pattern 4: Error Handling

```javascript
// JavaScript
if (!condition) throw new Error('Invalid state');
```

```kotlin
// Kotlin - input/option validation only (hot-path failures are Fail tags, see Pattern 1)
require(condition) { "Invalid state" }
check(condition) { "Invalid state" }
```

### Pattern 5: Loop with Multiple Variables

```javascript
// JavaScript
for (let i = 0, j = 0; i < n; i++, j += 2) {
  process(i, j);
}
```

```kotlin
// Kotlin - use separate variable
var j = 0
for (i in 0 until n) {
  process(i, j)
  j += 2
}
```

### Pattern 6: Ternary Chains

```javascript
// JavaScript
const x = a ? 1 : b ? 2 : c ? 3 : 4;
```

```kotlin
// Kotlin - use when expression
val x = when {
  a -> 1
  b -> 2
  c -> 3
  else -> 4
}
```

### Pattern 7: Population Count

```javascript
// JavaScript
function popcnt(a) {
  let cnt = 0;
  while (a) {
    if (a & 1) cnt++;
    a >>= 1;
  }
  return cnt;
}
```

```kotlin
// Kotlin
private fun popcnt(a: Int): Int {
  var n = a
  var cnt = 0
  while (n != 0) {
    if (n and 1 != 0) cnt++
    n = n shr 1
  }
  return cnt
}
```

**Note**: Variable `a` becomes `n` because parameters are read-only in Kotlin.

### Pattern 8: Array Initialization with Function

```javascript
// JavaScript
const arr = new Array(n).fill(0).map((_, i) => i * 2);
```

```kotlin
// Kotlin
val arr = IntArray(n) { i -> i * 2 }
// or
val arr = Array(n) { i -> i * 2 }
```

### Pattern 9: String Encoding (ECI)

JS resolves an ECI number to a WHATWG `TextDecoder` label (`ECI_ENCODINGS`, decode.ts L206-233).
Kotlin maps the same ECI numbers to the Java charset that implements that WHATWG decoder
(`Payload.ECI_CHARSETS`, e.g. `shift_jis` -> `windows-31j`, `gbk` -> `GB18030`, `euc-kr` ->
`x-windows-949`), and decodes with `String(bytes, off, len, charset)` (U+FFFD for malformed input,
like TextDecoder). Unknown/unsupported ECI -> `null` (data failure), never an exception.

## Kotlin Coding Style Guidelines

These conventions are used throughout the Kotlin QR library. **ALWAYS follow these when porting code**.

### 1. Use `object` for Singletons

```kotlin
// JavaScript: module-level functions
export function decode(img) { ... }

// Kotlin: object singleton
object QRDecoder {
  fun decode(img: Image): String { ... }
}
```

### 2. Preallocated Scratch State, No Per-Call Allocation

Use preallocated scratch arrays (`IntArray`/`DoubleArray` arenas sized in the constructor) and mutable
objects such as `Finder.Pattern` that are overwritten in place; never allocate per call on the hot path
(no `data class Point`, no `kotlin.Pair` returns, no destructuring of fresh objects). Stages return
`String?` (or Boolean) and record the failure as a `Fail` tag (see Pattern 1).

### 3. Prefer `val` over `var`

```kotlin
// Immutable by default
val width = 640
val height = 480

// Only use var when necessary
var retries = 0
while (retries < 3) {
  retries++
}
```

### 4. Use `when` Expressions

```kotlin
// Instead of if-else chains
when (mode) {
  1 -> readNumeric()
  2 -> readAlphanumeric()
  4 -> readBytes()
  else -> { fail = Fail.DATA; return null }
}
```

### 5. Use Ranges

```kotlin
// Instead of C-style loops
for (i in 0 until n) { ... }        // 0 to n-1
for (i in 0..n) { ... }             // 0 to n (inclusive)
for (i in n downTo 0) { ... }       // n to 0
for (i in 0 until n step 2) { ... } // 0, 2, 4, ...
```

### 6. Use Single-Expression Functions

```kotlin
// Instead of
fun double(x: Int): Int {
  return x * 2
}

// Use
fun double(x: Int) = x * 2
```

### 7. Use KDoc for Public APIs

```kotlin
/**
 * Decode a QR code from an image.
 *
 * @param image The image containing the QR code
 * @return The decoded string content
 * @throws QRDecodingException if decoding fails
 */
fun decode(image: Image): String { ... }
```

## Common Pitfalls

### 1. Integer Division

```javascript
// JavaScript - always returns float
const avg = (a + b) / 2;  // 5 / 2 = 2.5
```

```kotlin
// Kotlin - integer division for Int / Int
val avg = (a + b) / 2  // 5 / 2 = 2 (truncated!)

// Use .toDouble() for float division
val avg = (a + b).toDouble() / 2  // 5 / 2 = 2.5
```

### 2. Unsigned Right Shift

```javascript
// JavaScript
const x = -1 >>> 1;  // Unsigned right shift
```

```kotlin
// Kotlin - use ushr
val x = -1 ushr 1  // Unsigned right shift
```

### 3. Array Initialization

```javascript
// JavaScript - creates array with undefined elements
const arr = new Array(10);  // [undefined, undefined, ...]
```

```kotlin
// Kotlin - MUST provide initializer
val arr = Array(10) { 0 }  // [0, 0, 0, ...]
// or
val arr = IntArray(10)  // [0, 0, 0, ...]
```

### 4. Loop Bounds

```javascript
// JavaScript - exclusive end
for (let i = 0; i < n; i++) { ... }
```

```kotlin
// Kotlin
for (i in 0 until n) { ... }  // Correct - until is exclusive
for (i in 0..n-1) { ... }     // Also correct
for (i in 0..n) { ... }       // WRONG - includes n!
```

### 5. ByteArray vs Uint8Array

JavaScript `Uint8Array` is unsigned (0-255), but Kotlin `ByteArray` is signed (-128 to 127).

```kotlin
// When working with bytes that might be > 127
val unsigned = byte.toInt() and 0xFF  // Convert to unsigned
```

### 6. Mutable Parameters

```javascript
// JavaScript - parameters are mutable
function foo(x) {
  x = x + 1;  // OK
  return x;
}
```

```kotlin
// Kotlin - parameters are read-only
fun foo(x: Int): Int {
  // x = x + 1  // ERROR! Parameters are val
  val y = x + 1  // Use local variable
  return y
}
```

### 7. Null vs Undefined

JavaScript has both `null` and `undefined`. Kotlin only has `null`.

```javascript
// JavaScript
if (x !== undefined) { ... }
```

```kotlin
// Kotlin
if (x != null) { ... }
```

## Imports Needed

Common imports for Kotlin QR code:

```kotlin
import kotlin.math.sqrt
import kotlin.math.floor
import kotlin.math.ceil
import kotlin.math.abs
import kotlin.math.min
import kotlin.math.max
import java.nio.charset.Charset  // For ECI encoding
```

## Quick Reference Card

| Task | JavaScript | Kotlin |
|------|------------|--------|
| Loop 0 to n-1 | `for (let i = 0; i < n; i++)` | `for (i in 0 until n)` |
| Packed bitmap bit | `layer.bitmap[y * words + (x >>> 5)] >>> (x & 31) & 1` | same with `ushr`/`and` (`Layer.bit`) |
| Round | `Math.round(x)` | `Math.round(x).toInt()` |
| `x \| 0` | `x \| 0` | `Homography.toInt32(x)` |
| Bitwise AND | `a & b` | `a and b` |
| Bitwise OR | `a \| b` | `a or b` |
| Bitwise XOR | `a ^ b` | `a xor b` |
| Right shift | `a >> b` | `a shr b` |
| Unsigned shift | `a >>> b` | `a ushr b` |
| Ternary | `a ? b : c` | `if (a) b else c` |
| Null check | `x !== undefined` | `x != null` |
| Default value | `x \|\| default` | `x ?: default` |
| String template | `'value: ' + x` | `"value: $x"` |
| Array map | `arr.map(x => ...)` | `arr.map { x -> ... }` |
| Throw error | `throw new Error(msg)` | `throw Exception(msg)` |
